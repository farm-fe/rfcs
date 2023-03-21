---
author: '[@roland-reed](https://github.com/roland-reed)'
issue: 'https://github.com/farm-fe/farm/issues/9'
pr: 'https://github.com/farm-fe/rfcs/pull/1'
state: open
start_date: '2022-07-18'
---

# Farm's runtime module management

## Summary

This RFC describes a runtime module management system to load, register, and initialize modules in browsers and Node.js for Farm.

## Motivation

Since modules need to be bundled when distributed in browsers, modules from source code have to be merged into a limited number of chunks. Additionally, native module features are not available in all versions of browsers and Node.js, so an emulated module system is required.

## Overview

This RFC contains:

- What are modules and resources
- How to normalize ECMAScript modules and CommonJS modules
- How modules are loaded and initialized
- How module system plugins work
- Different strategies to support modern browsers/Node.js and legacy browsers

Terms:

- **ECMAScript modules**: modules that satisfy the [ECMAScript Specification](https://tc39.es/ecma262/#sec-modules)
- **CommonJS modules**: modules that satisfy the [CommonJS Specification](https://wiki.commonjs.org/wiki/Modules/1.1)
- **Resource**: files that contain any type of modules created, transformed, and merged by Farm
- **Module specifications**: descriptions of how modules are managed; common specifications include ECMAScript modules, CommonJS, AMD, and UMD
- **Legacy browsers**: browsers that do not support ECMAScript modules

## Detailed Design

### Module system

The module system is an object containing several methods that can be used to create, delete, or update modules or cache of modules.

#### Module system acquisition

A module system instance **MUST** be acquired via a function exposed in the global object (`globalThis`).

The default name of this function is `__acquire_farm_module_system__`. The name of the function can be configured via a global variable named `__farm_module_system_acquisition_name__`. For example, if `globalThis.__farm_module_system_acquisition_name__` is `foo`, then the name of the module system acquisition function is `foo`.

The module system acquisition function uses a singleton to expose the API. For one acquisition function name, only one instance will be returned, regardless of how many times this function is called.

```js
globalThis.__acquire_farm_module_system__ = (function () {
  function ModuleSystem() {
    // module system implementation
  }

  var moduleSystem = new ModuleSystem();

  return function () {
    return moduleSystem;
  };
})();
```

#### Module instance

Before calling the module initialization function, a module instance **MUST** be created. Every module instance has the following properties:

- **id**: a string representing the id of the module
- **exports**: an object where the module exports its bindings
- **meta**: an object that works like `import.meta`
- **loaded**: a boolean value indicating whether this module is loaded
- **error**: an Error instance if there is an error during the initialization of this module, or `null` if the initialization succeeds

The definition of modules:

```ts
interface Module {
  id: string;
  exports: any;
  meta: Record<string, any>;
  loaded: boolean;
  error?: Error;
}
```

#### Module initialization function

A module will be transformed into a **sync** or an **async** module initialization function, with its body containing the module content. When a module is `import`ed/`require`d, the module initialization function will be called.

Module initialization functions take three arguments:

- **`module`**: the module instance
- **`exports`**: the exports object of the module
- **`__farm_require__`**: the require function, used to require other modules

The module initialization function has no return value. After the function is executed, the `exports` object should contain all exported bindings of the module, or an error will be thrown if something goes wrong during initialization.

The definition of module initialization functions is:

```ts
type ModuleInitialization = (module: Module, exports: Record<string, any>, __farm_require__: (request: string) => Module);
```

Below is an example:

```js
// foo.js
export const foo = 1;

// transformed runtime module
const modules = {
  'foo.js': /* runtime module starts here */ function(module, exports, __farm_require__) {
    exports.foo = 1;
  } /* runtime module ends here */
}
```

#### Module system architecture

The module system consists of three parts:

- **module initialization functions map**: an object containing initialization functions of all modules
- **cached modules map**: an object containing all cached modules
- **loaded resources map**: an object containing all resources' ids that have been loaded (only for legacy browsers)

```ts
// pseudo code
class ModuleSystem {
  readonly moduleMap: Record<string, ModuleInitialization>;
  readonly cacheMap: Record<string, Module>;
  readonly loadedResourceMap: Record<string, boolean>;

  require(id: string): Module {}
  register(id: string, moduleInitialization: ModuleInitialization): void {}
  update(id: string, moduleInitialization: ModuleInitialization): void {}
  delete(id: string): boolean {}
  deleteCache(id: string): boolean {}
}
```

#### Module system API

Here are the APIs exposed by the module system:

```ts
interface ModuleSystem {
  require(id: string): Module;
  register(id: string, moduleInitialization: ModuleInitialization): void;
  update(id: string, moduleInitialization: ModuleInitialization): void;
  delete(id: string): boolean;
  deleteCache(id: string): boolean;
}
```

#### Module system API alias

For a smaller production size, every API has a short-name alias:

- `require` → `r`
- `register` → `g`
- `update` → `u`
- `delete` → `d`
- `deleteCache` → `c`

#### Module states

Modules have four states: **unregistered**, **registered**, **initialized**, and **failed**:

- **unregistered**: the module is not registered yet; this state can **ONLY** turn into **registered** state
- **registered**: the module is registered; this state can **ONLY** turn into **initialized** or **failed** state
- **initialized**: the module is initialized; this state is final and **MUST NOT** be changed
- **failed**: the module failed to initialize due to syntax or runtime errors; this state is final and **MUST NOT** be changed

```plain
                          initialized
                         ↗
unregistered → registered
                         ↘
                          failed
```

### Plugin system

Since the module system is simulated, some features that are originally provided by the host have to be implemented by the runtime module system, for example, the `import.meta` object. Hot Module Replacement (aka HMR) relies on `import.meta.hot` or `module.hot` to work properly. To make features like HMR work, a plugin system is needed.

A runtime module plugin can access the module instance before or after the module initialization function is called, allowing a plugin to modify the properties of the module instance, such as `module.meta` and `module.exports`.

#### Hook lifecycle

A runtime module plugin can intercept module activities via hooks. There are five hooks:

- `function bootstrap(ms: ModuleSystem): void`: invoked when the module system is bootstrapped; this hook will only be invoked once
- `function create(module: Module): Module`: invoked when new module instances are created; this hook will only be invoked once
- `function initialize(module: Module): Module`: invoked when module initialization functions are called; this hook will be invoked every time a module initialization function is called
- `function error(module: Module, error: Error): void`: invoked when module initialization function calls fail; this hook will be invoked every time an error is thrown during module initialization function execution
- `function cacheRead(importer: Module | null, imported: Module): Module | null`: invoked when module caches are read; this hook will be invoked every time a module cache is read

Below is a module's lifecycle:

```plain
                                                                           hook:error
                                                                         ↗
hook:bootstrap → module instance created → hook:create → hook:initialize → hook:cacheSave
                            ↑                                      ↓                ↑
                            |                        nested modules required        |
                            |                                    ↓    ↓             |
                            -------------------------------------      --------------
```

#### Hook registration

Like [tapable](https://github.com/webpack/tapable), a plugin is registered with a `tap` function, for example:

```js
globalThis.__acquire_farm_module_system__().hooks.bootstrap.tap(fn);
globalThis.__acquire_farm_module_system__().hooks.create.tap(fn);
```

#### Hook type

Although all hooks are synchronous, every hook's function is called in the order they are tapped. There are still two types of hooks: **basic** and **waterfall**.

- **Basic hook**: calls every function one by one, in the order they are tapped
- **Waterfall hook**: calls every function one by one, but every function is called with the return value of the last call, except for the first call

```js
// pseudo code
basicHooks.forEach(fn => fn(initialArg));

waterfallHooks.reduce((acc, fn) => fn(acc), initialArg);
```

#### Plugin registration

Plugins must be registered before the entry point module is loaded, or a plugin will not be able to access every module's activities. To avoid that, the plugin register method will throw an error after the entry point module has been initialized.

```js
// pseudo code
globalThis.__acquire_farm_module_system__ = function() { /* module system setup */ }
// register plugins
globalThis.__acquire_farm_module_system__().hooks.bootstrap.tap(plugin); // works
// load entry point module
loadEntryPointModule();
// register plugins after entry point module has been initialized
globalThis.__acquire_farm_module_system__().hooks.bootstrap.tap(plugin); // error will be thrown
// Error: cannot register plugins after entry point module has been initialized
```

### Module specification

Farm uses the CommonJS module specification to manage modules. ECMAScript modules will be transformed into CommonJS-like modules, with `import`, `import()` and `export` replaced by `require()`, `Promise.resolve(require())`, and `exports[exportName]` respectively.

> Note: `require` below will be replaced with Farm's actual require implementation of runtime module management.

#### `import` and `export` statements

The following is the relationship between ECMAScript modules' `import`/`export` statements and CommonJS modules' `require`/`exports[exportName]`:

- _ImportDeclaration_:
  - `import` _ImportClause_ _FromClause_
    - _ImportClause_:
      - _ImportedDefaultBinding_:
        - `import foo from 'IDENTIFIER'` -> `const { foo } = require('IDENTIFIER').default`
      - _NameSpaceImport_:
        - `import * as ns from 'IDENTIFIER'` -> `const ns = require('IDENTIFIER')`
      - _NamedImports_:
        - `import { foo, bar } from 'IDENTIFIER'` -> `const { foo, bar } = require('IDENTIFIER')`
      - _ImportedDefaultBinding_, _NameSpaceImport_:
        - `import foo, * as ns from 'IDENTIFIER'` -> `const ns = require('IDENTIFIER'); const foo = ns.default`
      - _ImportedDefaultBinding_, _NamedImports_:
        - `import foo, { foo, bar, baz as z } from 'IDENTIFIER'` -> `const { default: foo, foo, bar, baz: z } = require('IDENTIFIER')`
  - `import` _ModuleSpecifier_:
    - `import 'IDENTIFIER'` -> `require('IDENTIFIER')`
- _ExportDeclaration_:
  - `export` _ExportFromClause_ _FromClause_
    - `export * from 'IDENTIFIER';`
      - for every export identifier `ex` from 'IDENTIFIER', `module.exports[ex] = require('IDENTIFIER')[ex]`
    - `export * as ns from 'IDENTIFIER';` - > `exports.ns = require('IDENTIFIER')`
    - `export { foo, bar, baz as z} from 'IDENTIFIER';` -> `exports.foo = require('IDENTIFIER').foo; exports.bar = require('IDENTIFIER').bar; exports.z = require('IDENTIFIER').baz`
  - `export` _NamedExports_:
    - `export { foo, bar, baz as z };` -> `exports.foo = foo; exports.bar = bar; exports.z = baz`
  - `export` _VariableStatement_:
    - `export var baz = 1` -> `export.baz = 1`
  - `export` _Declaration_:
    - _HoistableDeclaration_:
      - `export function foo() {}` -> `exports.foo = function foo() {}`
      - `export function foo* () {}` -> `exports.foo = function foo* () {}`
      - `export async function foo() {}` -> `exports.foo = async function foo() {}`
      - `export async function foo* () {}` -> `exports.foo = async function foo* () {}`
    - _ClassDeclaration_:
      - `export class foo {}` -> `exports.foo= class foo {}`
    - _LexicalDeclaration_:
      - `export let foo = 1` -> `exports.foo = 1`
      - `export let foo, bar, baz = 1` -> `exports.foo = undefined; exports.bar = undefined; exports.baz = 1`
      - `export const bar = 1` -> `exports.bar = 1`
      - `export const foo = 1, bar = 1` -> `exports.foo = 1; exports.bar = 1`
  - `export default` _HoistableDeclaration_:
    - `export default function foo() {}` -> `exports.default= function foo() {}`
    - `export default function foo* () {}` -> `exports.default= function foo* () {}`
    - `export default async function foo() {}` -> `exports.default= async function foo() {}`
    - `export default async function foo* () {}` -> `exports.default= async function foo* () {}`
  - `export default` _ClassDeclaration_:
    - `export default class foo {}` -> `exports.default= class foo {}`
  - `export default [lookahead ∉ { function, async [no [LineTerminator](https://tc39.es/ecma262/#prod-LineTerminator) here] function, class }]` _AssignmentExpression_:
    - _ConditionalExpression_:
      - `export default foo ? bar : baz` -> `exports.default = foo ? bar : baz`
    - _ArrowFunction_:
      - `export default () => 1` -> `exports.default = () => 1`
    - _AsyncArrowFunction_:
      - `export default async () => 1` -> `exports.default = async () => 1`
    - _LeftHandSideExpression_ = _AssignmentExpression_:
      - `export default a = 1` (where `a` is a variable declared with `var` or `let`) -> `exports.default = a = 1`
    - _LeftHandSideExpression_ = _AssignmentOperator_ _AssignmentExpression_:
      - `export default a += 1` (where `a` is a variable declared with `var` or `let`, _AssignmentExpression_: one of `*= /= %= += -= <<= >>= >>>= &= ^= |= **=`) -> `exports.default = a += 1`
    - _LeftHandSideExpression_ `&&=` _AssignmentExpression_:
      - `export default a &&= 1` (where `a` is a variable declared with `var` or `let`) -> `exports.default = a &&= 1`
    - _LeftHandSideExpression_ `||=`_AssignmentExpression_:
      - `export default a ||= 1` (where `a` is a variable declared with `var` or `let`) -> `exports.default = a ||= 1`
    - _LeftHandSideExpression_ `??=`_AssignmentExpression_:
      - `export default a ??= 1` (where `a` is a variable declared with `var` or `let`) -> `exports.default = a ??= 1`

> See also: [ECMAScript Modules](https://tc39.es/ecma262/#sec-modules)

#### `import()` calls

`import()` calls will be transformed to `Promise.resolve(require('IDENTIFIER'))`. The result of `require('IDENTIFIER')` should be an instant module (if the required module has been registered) or a promise that resolves to a module (if the required module has not been registered).

#### Top-level await export

Top-level await export is introduced in ES2022, which allows the usage of `await` at the top level of modules. This enables modules to act like async functions, allowing you to `await` promises in modules.

Since this feature is introduced in ES2022, legacy browsers do not support it, but it can be simulated with some techniques.

For normal modules, the corresponding transformed initialization functions are sync functions, while for modules using top-level await, the corresponding transformed initialization functions are async functions. See the example below.

Source code of sync modules:

```js
// foo.sync.js

export const foo = 1;
```

Transformed module initialization function:

```js
{
  'foo.sync.js': function(module, exports, __farm_require__) {
    exports.foo = 1;
  }
}
```

Source code of modules using top-level await:

```js
// bar.async.js

// returns { code: 0, message: 'Hello world!' }
export const bar = await (await fetch('https://foo.bar').json());
```

Transformed module initialization function:

```js
{                // ↓ notice the async here
  'bar.async.js': async function(module, exports, __farm_require__) {
    exports.foo = await (await fetch('https://foo.bar').json());
  }
}
```

Modules importing modules using top-level await have to be transformed to async modules as well. See the example below.

Source code of modules importing modules that used top-level await:

```js
// foo.async.js

// returns { code: 0, message: 'Hello from foo!' }
export const foo = await (await fetch('https://foo.com').json());

// bar.async.js

// returns { code: 0, message: 'Hello from bar!' }
export const bar = await (await fetch('https://bar.com').json());

// baz.sync.js

// returns { code: 0, message: 'Hello from baz!' }
export const bar = await (await fetch('https://baz.com').json());

// main.js
import { bar } from './bar.async.js';
import { baz } from './baz.sync.js';
import { foo } from './foo.async.js';

console.log(bar); // { code: 0, message: 'Hello from bar!' }
console.log(typeof bar); // object
console.log(baz); // { code: 0, message: 'Hello from baz!' }
console.log(typeof baz); // object
console.log(foo); // { code: 0, message: 'Hello from foo!' }
console.log(typeof foo); // object
```

Transformed module initialization function:

```js
{           // ↓ notice the async here
  'main.js': async function(module, exports, __farm_require__) {
    const [
      { bar },
      { baz },
      { foo }
    //    ↓ notice the await here
    ] = await Promise.all([
      __farm_require__('bar.async.js'),
      __farm_require__('baz.sync.js'),
      __farm_require__('foo.async.js')
    ]);

    console.log(bar); // { code: 0, message: 'Hello from bar!' }
    console.log(typeof bar); // object
    console.log(baz); // { code: 0, message: 'Hello from baz!' }
    console.log(typeof baz); // object
    console.log(foo); // { code: 0, message: 'Hello from foo!' }
    console.log(typeof foo); // object
  }
}
```

#### `import.meta`

`import.meta` is a host-provided meta-property containing a set of properties determined by the host environment.

In browsers that support ECMAScript modules and Node.js with native ECMAScript module support, there is an `import.meta.url` property indicating the URL (for browsers) or file path (for Node.js) of the module.

> For details, see the [proposal](https://github.com/tc39/proposal-import-meta).

In compile time, `import.meta` will be replaced with `module.meta`.  `module.meta` is an object created with a `null` prototype. By default, `module.meta` has only one property: `url`.

If the resource is an ECMAScript module, `import.meta.url` will be assigned to `module.meta.url`. If the resource is not an ECMAScript module, `document.currentScript.src` will be assigned to `module.meta.url` during module registration.

Example of ECMAScript modules:

Source code:

```js
// main.js
export {}; // indicate this is a module

console.log(import.meta.url);
```

Transformed resource:

```js
{
  'main.js': function(module, exports) {
    // this line is injected by the compiler
    module.meta.url = import.meta.url; // use import.meta.url directly
    console.log(module.meta.url);
  }
}
```

Example of non-ECMAScript modules:

Source code:

```js
// main.js
export {}; // indicate this is a module

console.log(import.meta.url);
```

Transformed resource:

```js
(function (modules, url) {
  Object.keys(modules).forEach(function (key) {
    __acquire_farm_module_system__().register(key, modules[key]);
  });
})({
  'main.js': function (module, exports, require) {
    module.meta.url = url; // use url argument provided while initializing the resource script
    console.log(module.meta.url);
  }
}, document.currentScript.src)
```

> For details about `document.currentScript.src`, please see [Document.currentScript](https://developer.mozilla.org/en-US/docs/Web/API/Document/currentScript).

### Resource

**Resources** are files that contain code, which may include many modules. A script resource has many module initialization functions, and a style resource may only contain CSS code.

#### Resource type

There are two types of resources: **initial resources** and **async resources**. **Initial resources** are essential for the entry module to run, while **async resources** are loaded asynchronously based on certain conditions.

**Initial resources** will be loaded in HTML with `<script>` tags, while **async resources** will be loaded with `import()` calls (for modern browsers and Node.js) or dynamically added `<script>` elements (for legacy browsers).

#### Resource kind

There are two kinds of resources: **script** and **style**.

An example of a script resource:

```js
(function (modules, url) {
  Object.keys(modules).forEach(function (id) {
    __acquire_farm_module_system__().register(id, modules[id]);
  });
})({
  'foo.js': function (module, exports, __farm_require__) {
    exports.foo = 1;
  },
  'bar.js': function (module, exports, __farm_require__) {
    exports.bar = 2;
  }
}, document.currentScript.src)
```

An example of a style resource:

```css
body {
  background: #fff;
}
```

##### Resource loading

Every module will be bundled into a resource. Dynamically imported modules will be loaded in a resource whose resource name is determined during compilation. For example, if a script module `foo.js` is dynamically imported, and it's bundled into resource `resource.js`, the importing code may look like this:

```js
// source code
import foo from './foo';

// compiled
loadResource('resource.js').then(() => require('./foo'));
```

For style resources, they will be imported like this:

```js
// source code
import 'style.css';

// compiled
loadResource('resource.css').then(() => console.log('style loaded'));
```

where function `loadResource` will create a `link` element and insert it into the `head` of the document.

#### Resource states

Resources have four states: **unloaded**, **loading**, **loaded**, and **failed**:

- **unloaded**: the resource is not loaded yet; this state can **ONLY** turn into **loading** state
- **loading**: the resource is loading; this state can **ONLY** turn into **loaded** or **failed** state
- **loaded**: the resource is loaded successfully; this state is the final state and **MUST NOT** be changed
- **failed**: the resource failed to load due to network error or other reasons; this state is the final state and **MUST NOT** be changed

```plain
                     loaded
                   ↗
unloaded → loading
                   ↘
                     failed
```

#### Resource loading strategies

There may be many resources, each resource contains one or more modules, a resource may import one or more other modules, which are located in other resources. For modern browsers and Node.js, using the host-provided module system to load resources is much more stable and reliable than manually managing them. However, for older browsers, a fallback module system is used.

An application is loaded in the following steps:

1. Farm generates an HTML file that contains initial resources and bootstrap code
2. After the browser parses the HTML, initial resources will be loaded
   1. If initial resources are loaded successfully, modules contained in these resources will be registered
   2. If the initial resource cannot be loaded successfully, an error will be thrown
3. After all initial resources are loaded and modules are registered, the bootstrap code will load the entry point module
4. During the entry point module execution, more modules will be loaded. For every module:
   1. If the module has been initialized before, return the cached module reference
   2. If the module has been initialized before but ended with an error, throw the error
   3. If the module has been registered but not initialized, try to initialize the module
      1. If module initialization is successful, save it in cache and return the module reference
      2. If module initialization is not successful, throw the error and save the error in cache
   4. If the module has not been registered, find out which resource contains this module, load the resource, and after the resource is loaded, try to initialize the module


#### Example

Say there are four resources in an application: `foo.js`, `bar.js`, `baz.js` and `ops.js`, every resource contains one or more modules:

- foo.js
  - module_a.js
  - main.js (imports module_a.js and module_b.js)
- bar.js
  - module_b.js (dynamically imports module_c.js)
- baz.js
  - module_c.js (imports module_a.js and dynamically imports module_d.js)
- ops.js
  - module_d.js

Relationship in diagram:

```plain
         main.js
           ↙ ↘︎
module_a.js   module_b.js
    ↑              ↓ (dynamic import)
     --------- module_c.js
                   ↓ (dynamic import)
              module_d.js
```

##### Modern browsers and Node.js

HTML file will contain `foo.js` and `bar.js`, as they are **initial resources**, `baz.js` and `ops.js` will be dynamically loaded when used. The order of initial resources is sorted topologically, in this example, load `bar.js` first, then load `foo.js` later.

```html
<html>
  <head>
    <script>
      globalThis.__acquire_farm_module_system__ = function() { /* module system setup */ }
    </script>
    <script src="bar.js" type="module"></script>
    <script src="foo.js" type="module"></script>
  </head>
</html>
```

```js
// bar.js
// register module_b.js in this resource
// baz is loaded as it contains module_c.js
import('baz').then(() => console.log(__acquire_farm_module_system__().require('module_c.js')));

// foo.js
// register module_a.js and main.js in this resource
// module main.js
const { a } = __acquire_farm_module_system__().require('module_a.js');
const { b } = __acquire_farm_module_system__().require('module_b.js');

console.log(a);
console.log(b);

// baz.js
// module_a.js has been loaded in main.js
const { a } = __acquire_farm_module_system__().require('a.js');

// ops is loaded as it contains module_d.js
import('ops.js').then(() => console.log(__acquire_farm_module_system__().require('module_d.js')));
```

##### Legacy browsers

It's almost identical to load initial resources with modern browsers in legacy browsers, but there are still two differences, one is that resources are not modules any more, so the attribute `type="module"` is absent, second is that dynamically added `<script>` elements are used to asynchronously load dynamic imported resources.

```js
// baz.js
// instead of using code below:
import('ops.js').then(() => { /* after module is loaded */ });

// using code like this (simplified version):
function __farm_dynamic_import__(resourceName) {
  return new Promise(function(resolve, reject) {
    var s = document.createElement('script');

    s.src = 'ops.js';
    s.onload = function() {
      // after module is loaded
      resolve(__acquire_farm_module_system__().require('module_d.js'));
    }
    s.onerror = function(error) {
      reject(error);
    }
    document.body.appendChild(s);
  });
}

__farm_dynamic_import__('ops.js').then(m => { /* after module is loaded */ });
```
