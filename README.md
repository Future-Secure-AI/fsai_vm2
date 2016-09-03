# vm2 [![NPM Version][npm-image]][npm-url] [![Package Quality][quality-image]][quality-url] [![Travis CI][travis-image]][travis-url]

vm2 is a sandbox that can run untrusted code with whitelisted Node's built-in modules. Securely!

**Features**

* Runs untrusted code securely in a single process with your code side by side
* Full control over sandbox's console output
* Sandbox has limited access to process's methods
* Sandbox can require modules (builtin and external)
* You can limit access to certain (or all) builtin modules
* You can securely call methods and exchange data and callback between sandboxes
* Is immune to `while (true) {}` (VM only, see docs)
* Is immune to all known methods of attacks
* Transpilers support

**How does it work**

* It uses internal VM module to create secure context
* It uses [Proxies](https://developer.mozilla.org/cs/docs/Web/JavaScript/Reference/Global_Objects/Proxy) to prevent escaping the sandbox
* It overrides builtin require to control access to modules

## Installation

**IMPORTANT**: Requires Node.js 6 or newer.

    npm install vm2

## Quick Example

```javascript
const {VM} = require('vm2');
const vm = new VM();

vm.run(`process.exit()`); // TypeError: process.exit is not a function
```

```javascript
const {NodeVM} = require('vm2');
const vm = new NodeVM({
	require: {
		external: true
	}
});

vm.run(`
	var request = require('request');
	request('http://www.google.com', function (error, response, body) {
		console.error(error);
		if (!error && response.statusCode == 200) {
			console.log(body) // Show the HTML for the Google homepage.
		}
	})
`, 'vm.js');
```

## Documentation

* [VM](#vm)
* [NodeVM](#nodevm)
* [Cross-sandbox relationships](#cross-sandbox-relationships)
* [CLI](#cli)
* [2.x to 3.x changes](https://github.com/patriksimek/vm2/wiki/2.x-to-3.x-changes)
* [1.x and 2.x docs](https://github.com/patriksimek/vm2/wiki/1.x-and-2.x-docs)
* [Contributing](https://github.com/patriksimek/vm2/wiki/Contributing)

## VM

VM is a simple sandbox, without `require` feature, to synchronously run an untrusted code. Only JavaScript built-in objects + Buffer are available.

**Options:**

* `timeout` - Script timeout in milliseconds.
* `sandbox` - VM's global object.
* `compiler` - `javascript` (default) or `coffeescript` or custom compiler function.

**IMPORTANT**: Timeout is only effective on code you run through `run`. Timeout is NOT effective on any method returned by VM.

```javascript
const {VM} = require('vm2');

const vm = new VM({
    timeout: 1000,
    sandbox: {}
});

vm.run("process.exit()"); // throws ReferenceError: process is not defined
```

You can also retrieve values from VM.

```javascript
let number = vm.run("1337"); // returns 1337
```

**TIP**: See tests for more usage examples.

## NodeVM

Unlike `VM`, `NodeVM` lets you require modules same way like in regular Node's context.

**Options:**

* `console` - `inherit` to enable console, `redirect` to redirect to events, `off` to disable console (default: `inherit`).
* `sandbox` - VM's global object.
* `compiler` - `javascript` (default) or `coffeescript` or custom compiler function.
* `require` - `true` or object to enable `require` method (default: `false`).
* `require.external` - `true` to enable `require` of external modules (default: `false`).
* `require.builtin` - Array of allowed builtin modules (default: none).
* `require.root` - Restricted path where local modules can be required (default: every path).
* `require.mock` - Collection of mock modules (both external or builtin).
* `require.context` - `host` (default) to require modules in host and proxy them to sandbox. `sandbox` to load, compile and require modules in sandbox. Builtin modules except `events` always required in host and proxied to sandbox.
* `require.import` - Array of modules to be loaded into NodeVM on start.
* `nesting` - `true` to enable VMs nesting (default: `false`).
* `wrapper` - `commonjs` (default) to wrap script into CommonJS wrapper, `none` to retrieve value returned by the script.

**IMPORTANT**: Timeout is not effective for NodeVM so it is not immune to `while (true) {}` or similar evil.

**REMEMBER**: The more modules you allow, the more fragile your sandbox becomes.

```javascript
const {NodeVM} = require('vm2');

const vm = new NodeVM({
	console: 'inherit',
    sandbox: {},
    require: {
        external: true,
        builtin: ['fs', 'path'],
        root: "./",
        mock: {
	        fs: {
		        readFileSync() { return 'Nice try!'; }
	        }
        }
    }
});

let functionInSandbox = vm.run("module.exports = function(who) { console.log('hello '+ who); }");
functionInSandbox('world');
```

When `wrapper` is set to `none`, `NodeVM` behaves more like `VM` for synchronous code.

```javascript
assert.ok(vm.run('return true') === true);
```

**TIP**: See tests for more usage examples.

### Loading modules by relative path

To load modules by relative path, you must pass full path of the script you're running as a second argument of vm's `run` method. Filename then also shows up in any stack traces produced from the script.

```javascript
vm.run("require('foobar')", "/data/myvmscript.js");
```

## Cross-sandbox relationships

```javascript
const assert = require('assert');
const {VM} = require('vm2');

let sandbox = {
	object: new Object(),
	func: new Function(),
	buffer: new Buffer([0x01, 0x05])
}

let vm = new VM({sandbox});

assert.ok(vm.run(`object`) === sandbox.object);
assert.ok(vm.run(`object instanceof Object`));
assert.ok(vm.run(`object`) instanceof Object);
assert.ok(vm.run(`object.__proto__ === Object.prototype`));
assert.ok(vm.run(`object`).__proto__ === Object.prototype);

assert.ok(vm.run(`func`) === sandbox.func);
assert.ok(vm.run(`func instanceof Function`));
assert.ok(vm.run(`func`) instanceof Function);
assert.ok(vm.run(`func.__proto__ === Function.prototype`));
assert.ok(vm.run(`func`).__proto__ === Function.prototype);

assert.ok(vm.run(`new func() instanceof func`));
assert.ok(vm.run(`new func()`) instanceof sandbox.func);
assert.ok(vm.run(`new func().__proto__ === func.prototype`));
assert.ok(vm.run(`new func()`).__proto__ === sandbox.func.prototype);

assert.ok(vm.run(`buffer`) === sandbox.buffer);
assert.ok(vm.run(`buffer instanceof Buffer`));
assert.ok(vm.run(`buffer`) instanceof Buffer);
assert.ok(vm.run(`buffer.__proto__ === Buffer.prototype`));
assert.ok(vm.run(`buffer`).__proto__ === Buffer.prototype);
assert.ok(vm.run(`buffer.slice(0, 1) instanceof Buffer`));
assert.ok(vm.run(`buffer.slice(0, 1)`) instanceof Buffer);
```

## CLI

Before you can use vm2 in command line, install it globally with `npm install vm2 -g`.

```
$ vm2 ./script.js
```

## Known Issues

* It is not possible to define class that extends proxied class.

## License

Copyright (c) 2014-2016 Patrik Simek

The MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

[npm-image]: https://img.shields.io/npm/v/vm2.svg?style=flat-square
[npm-url]: https://www.npmjs.com/package/vm2
[quality-image]: http://npm.packagequality.com/shield/vm2.svg?style=flat-square
[quality-url]: http://packagequality.com/#?package=vm2
[travis-image]: https://img.shields.io/travis/patriksimek/vm2/master.svg?style=flat-square&label=unit
[travis-url]: https://travis-ci.org/patriksimek/vm2
