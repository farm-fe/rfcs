---
author: "[@roland-reed](https://github.com/roland-reed)"
issue: "https://github.com/farm-fe/farm/issues/9"
pr: "https://github.com/farm-fe/rfcs/pull/1"
state: open
start_date: "2022-07-18"
---

# Farm 运行时模块管理

## 概述

该 RFC 描述了一个运行时模块管理系统, 作为 Farm 在浏览器以及 Node 中加载, 注册, 初始化模块

## 动机

由于在浏览器中分发时需要对模块进行打包, 捆绑在一起。 因此必须要将源码中的模块进行合并然后分发成不同的 Chunks, 因为并非所有版本的浏览器或者 NodeJs 都提供支持原生模块分发的功能, 所以我们需要模拟出来一套模块系统

## 概述

该 RFC 包含:

- 什么是模块和资源
- 如何标准化 ECMAScript 模块和 CommonJs 模块
- 如何加载和初始化模块
- 模块系统插件的工作原理
- 支持现代浏览器/ NodeJs 和传统浏览器之间的不同策略

学术文档:

- **ECMAScript modules**: modules that satisfy the [ECMAScript Specification](https://tc39.es/ecma262/#sec-modules)
- **CommonJS modules**: modules that satisfy the [CommonJS Specification](https://wiki.commonjs.org/wiki/Modules/1.1)
- **Resource**: files that contain any type of modules created, transformed, and merged by Farm
- **Module specifications**: descriptions of how modules are managed; common specifications include ECMAScript modules, CommonJS, AMD, and UMD
- **Legacy browsers**: browsers that do not support ECMAScript modules

## 详细设计

## 模块系统

模块系统是一个包含多个方法的对象，这些方法可用于创建、删除或更新模块以及模块的缓存。

## 解析模块系统

获取整个模块系统必须要通过在全局对象 (`globalThis`) 中公开的函数来获取模块系统实例

该函数的默认名称是 `__acquire_farm_module_system__`。函数的名称可以通过名为 `__farm_module_system_acquisition_name__` 的全局变量进行配置。例如，如果 globalThis.`__farm_module_system_acquisition_name__` 为 foo，那么模块系统获取函数的名称就是 foo。

模块系统获取函数使用单例模式来暴露 API。对于一个获取函数名称，无论你调用该函数多少次，只会返回一个实例。

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

#### 模块实例

在调用模块初始化函数之前，必须创建一个模块实例。每个模块实例具有以下属性：

- **id**: 一个表示模块 ID 的字符串
- **exports**: 一个对象，模块在其中导出其绑定
- **meta**: 一个类似于 import.meta 的对象
- **loaded**: 一个布尔值，指示该模块是否已加载
- **error**: 如果在初始化此模块期间出现错误，则为一个 Error 实例；如果初始化成功，则为 null。

模块的定义：

```ts
interface Module {
  id: string;
  exports: any;
  meta: Record<string, any>;
  loaded: boolean;
  error?: Error;
}
```

#### 初始化模块

一个模块将被转换为一个同步或异步模块初始化函数，其函数体包含模块内容。当一个模块被 import 或 require 时，模块初始化函数将被调用。

模块初始化函数接受三个参数：

- **`module`**: 模块实例
- **`exports`**: 模块的导出对象
- **`__farm_require__`**: require 函数，用于引入其他模块

模块初始化函数没有返回值。在函数执行后，exports 对象应该包含模块的所有导出，如果在初始化过程中出现问题，则会抛出错误。

模块初始化函数的定义如下：

```ts
type ModuleInitialization = (module: Module, exports: Record<string, any>, __farm_require__: (request: string) => Module);
```

以下是一个示例：

```js
// foo.js
export const foo = 1;

// transformed runtime module
const modules = {
  "foo.js": /* runtime module starts here */ function (
    module,
    exports,
    __farm_require__
  ) {
    exports.foo = 1;
  } /* runtime module ends here */,
};
```

#### 模块系统架构

模块系统由三个部分组成：

- **模块初始化函数映射**: an object containing initialization functions of all modules
- **已缓存模块映射**: 一个包含所有已缓存模块的对象
- **已加载资源映射**: 一个包含已加载资源的 ID 的对象（仅适用于旧版浏览器）

```ts
// 伪代码如下：
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

#### 模块系统 API

以下是模块系统所暴露的一些 API：

```ts
interface ModuleSystem {
  require(id: string): Module;
  register(id: string, moduleInitialization: ModuleInitialization): void;
  update(id: string, moduleInitialization: ModuleInitialization): void;
  delete(id: string): boolean;
  deleteCache(id: string): boolean;
}
```

#### 模块系统 API 别名

为了减小生产体积大小，每个 API 都有一个短名称的别名：

- `require` → `r`
- `register` → `g`
- `update` → `u`
- `delete` → `d`
- `deleteCache` → `c`

#### 模块状态

模块具有四种状态：**未注册**、**已注册**、**已初始化** 和 **失败**：

- **未注册**：模块尚未注册；此状态只能转变为已注册状态。
- **已注册**：模块已注册；此状态只能转变为已初始化或失败状态。
- **已初始化**：模块已初始化；此状态为最终状态，不得更改。
- **失败**：模块由于语法或运行时错误而初始化失败；此状态为最终状态，不得更改。

```plain
                          initialized
                         ↗
unregistered → registered
                         ↘
                          failed
```

### 插件系统

由于模块系统是模拟的，一些原本由宿主提供的特性必须由运行时模块系统来实现，例如 import.meta 对象。热模块替换（Hot Module Replacement，又称 HMR）依赖于 import.meta.hot 或 module.hot 来正常工作。为了使诸如 HMR 这样的特性能够正常工作，需要引入一个插件系统。

运行时模块插件可以在模块初始化函数调用之前或之后访问模块实例，允许插件修改模块实例的属性，如 module.meta 和 module.exports。

#### 生命周期钩子

运行时模块插件可以通过钩子（hooks）拦截模块活动。有五种钩子：

- `function bootstrap(ms: ModuleSystem): void`: 在模块系统启动时调用；此钩子仅会被调用一次。
- `function create(module: Module): Module`: 在创建新的模块实例时调用；此钩子仅会被调用一次。
- `function initialize(module: Module): Module`: 在调用模块初始化函数时调用；此钩子将在每次调用模块初始化函数时被调用。
- `function error(module: Module, error: Error): void`: 在模块初始化函数调用失败时调用；此钩子将在每次在模块初始化函数执行过程中抛出错误时被调用。
- `function cacheRead(importer: Module | null, imported: Module): Module | null`: 在读取模块缓存时调用；此钩子将在每次读取模块缓存时被调用。

以下是模块的生命周期：

```plain
                                                                           hook:error
                                                                         ↗
hook:bootstrap → module instance created → hook:create → hook:initialize → hook:cacheSave
                            ↑                                      ↓                ↑
                            |                        nested modules required        |
                            |                                    ↓    ↓             |
                            -------------------------------------      --------------
```

#### Hook 注册

Like [tapable](https://github.com/webpack/tapable), 插件通过 tap 函数进行注册，例如：

```js
globalThis.__acquire_farm_module_system__().hooks.bootstrap.tap(fn);
globalThis.__acquire_farm_module_system__().hooks.create.tap(fn);
```

#### Hook 类型

尽管所有钩子都是同步的，但每个钩子的函数会按照它们被注册的顺序依次调用。仍然存在两种类型的钩子：**基本钩子** 和 **瀑布钩子**。

- **Basic hook**: 依次调用每个函数，按照它们被注册的顺序
- **Waterfall hook**: 依次调用每个函数，但每个函数的调用都使用上一个调用的返回值，除了第一个调用之外

```js
// pseudo code
basicHooks.forEach((fn) => fn(initialArg));

waterfallHooks.reduce((acc, fn) => fn(acc), initialArg);
```

#### 插件注册

插件必须在入口点模块加载之前注册，否则插件将无法访问每个模块的活动。为了避免这种情况，插件注册方法会在入口点模块初始化后抛出错误。

```js
// pseudo code
globalThis.__acquire_farm_module_system__ = function () {
  /* module system setup */
};
// register plugins
globalThis.__acquire_farm_module_system__().hooks.bootstrap.tap(plugin); // works
// load entry point module
loadEntryPointModule();
// register plugins after entry point module has been initialized
globalThis.__acquire_farm_module_system__().hooks.bootstrap.tap(plugin); // error will be thrown
// Error: cannot register plugins after entry point module has been initialized
```

### 模块规范

Farm 使用 `CommonJS` 模块规范来管理模块。ECMAScript 模块会被转换为类似 `CommonJS` 的模块，其中的 `import`、`import()` 和 `export` 分别被替换为 `require()`、`Promise.resolve(require())` 和 `exports[exportName]`。

> 注意：下面的 require 将会被 Farm 实际的运行时模块管理的 require 实现所替代。

#### `import` 和 `export` 语句

以下是 ECMAScript 模块的 import/export 语句与 CommonJS 模块的 require/exports[exportName] 之间的关系：

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

> 查看: [ECMAScript Modules](https://tc39.es/ecma262/#sec-modules)

#### `import()` 调用

import() 调用将会被转换为 Promise.resolve(require('IDENTIFIER'))。require('IDENTIFIER') 的结果应该是一个即时的模块（如果所需的模块已注册），或者是一个解析为模块的 Promise（如果所需的模块尚未注册）。

#### 顶层 await 导出

顶层 await 导出是在 ES2022 中引入的，它允许在模块的顶层使用 await。这使得模块可以像异步函数一样运行，允许在模块中使用 await 来等待 promises。

由于这个特性是在 ES2022 中引入的，旧版浏览器不支持它，但可以通过一些技术进行模拟。

对于普通模块，相应的转换后的初始化函数是同步函数，而对于使用顶层 await 的模块，相应的转换后的初始化函数是异步函数。请看下面的示例。

同步模块的源代码：

```js
// foo.sync.js

export const foo = 1;
```

转换后的模块初始化函数：

```js
{
  'foo.sync.js': function(module, exports, __farm_require__) {
    exports.foo = 1;
  }
}
```

使用顶层 await 的模块的源代码：

```js
// bar.async.js

// returns { code: 0, message: 'Hello world!' }
export const bar = await await fetch("https://foo.bar").json();
```

转换后的模块初始化函数：

```js
{                // ↓ notice the async here
  'bar.async.js': async function(module, exports, __farm_require__) {
    exports.foo = await (await fetch('https://foo.bar').json());
  }
}
```

使用顶层 await 导入模块的模块也必须被转换为异步模块。请看下面的示例。

导入使用了顶层 await 的模块的模块的源代码：

```js
// foo.async.js

// returns { code: 0, message: 'Hello from foo!' }
export const foo = await await fetch("https://foo.com").json();

// bar.async.js

// returns { code: 0, message: 'Hello from bar!' }
export const bar = await await fetch("https://bar.com").json();

// baz.sync.js

// returns { code: 0, message: 'Hello from baz!' }
export const bar = await await fetch("https://baz.com").json();

// main.js
import { bar } from "./bar.async.js";
import { baz } from "./baz.sync.js";
import { foo } from "./foo.async.js";

console.log(bar); // { code: 0, message: 'Hello from bar!' }
console.log(typeof bar); // object
console.log(baz); // { code: 0, message: 'Hello from baz!' }
console.log(typeof baz); // object
console.log(foo); // { code: 0, message: 'Hello from foo!' }
console.log(typeof foo); // object
```

转换后的模块初始化函数：

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

import.meta 是一个由宿主环境提供的元属性（meta-property），其中包含一组由宿主环境确定的属性。

在支持 ECMAScript 模块的浏览器和具有本机 ECMAScript 模块支持的 Node.js 中，存在一个 import.meta.url 属性，表示模块的 URL（对于浏览器）或文件路径（对于 Node.js）。

> 详见 [proposal](https://github.com/tc39/proposal-import-meta).

在编译时，import.meta 将被替换为 module.meta。module.meta 是一个以 null 为原型创建的对象。默认情况下，module.meta 只有一个属性：url。

如果资源是一个 ECMAScript 模块，import.meta.url 将被赋值给 module.meta.url。如果资源不是 ECMAScript 模块，在模块注册期间，document.currentScript.src 将被赋值给 module.meta.url。

ECMAScript 模块的示例：

源代码：

```js
// main.js
export {}; // indicate this is a module

console.log(import.meta.url);
```

转换后的资源：

```js
{
  'main.js': function(module, exports) {
    // this line is injected by the compiler
    module.meta.url = import.meta.url; // use import.meta.url directly
    console.log(module.meta.url);
  }
}
```

非 ECMAScript 模块的示例：

源代码：

```js
// main.js
export {}; // indicate this is a module

console.log(import.meta.url);
```

转换后的资源：

```js
(function (modules, url) {
  Object.keys(modules).forEach(function (key) {
    __acquire_farm_module_system__().register(key, modules[key]);
  });
})(
  {
    "main.js": function (module, exports, require) {
      module.meta.url = url; // use url argument provided while initializing the resource script
      console.log(module.meta.url);
    },
  },
  document.currentScript.src
);
```

> 关于 `document.currentScript.src` 的详细介绍 详见 [Document.currentScript](https://developer.mozilla.org/en-US/docs/Web/API/Document/currentScript).

### Resource

**Resources** 是包含代码的文件，可以包含许多模块。一个脚本资源可能有许多模块初始化函数，而一个样式资源可能只包含 CSS 代码。

#### Resource 类型

**Resources** 分为两种类型：初始资源和异步资源。初始资源对于入口模块的运行至关重要，而异步资源根据某些条件异步加载。

初始资源将使用 `<script>` 标签在 HTML 中加载，而异步资源将使用 import() 调用（用于现代浏览器和 Node.js）或动态添加的 `<script>` 元素（用于旧版浏览器）加载。

#### Resource 种类

资源有两种种类： **script** and **style**。

脚本资源的一个示例：

```js
(function (modules, url) {
  Object.keys(modules).forEach(function (id) {
    __acquire_farm_module_system__().register(id, modules[id]);
  });
})(
  {
    "foo.js": function (module, exports, __farm_require__) {
      exports.foo = 1;
    },
    "bar.js": function (module, exports, __farm_require__) {
      exports.bar = 2;
    },
  },
  document.currentScript.src
);
```

样式资源的一个示例：

```css
body {
  background: #fff;
}
```

##### Resource 加载

每个模块将被打包成一个资源。动态导入的模块将在一个资源中加载，其资源名称在编译期间确定。例如，如果一个脚本模块 foo.js 被动态导入，并且它被打包到资源 resource.js 中，导入的代码可能如下所示：

```js
// source code
import foo from "./foo";

// compiled
loadResource("resource.js").then(() => require("./foo"));
```

对于样式资源，它们将会像这样导入：

```js
// source code
import "style.css";

// compiled
loadResource("resource.css").then(() => console.log("style loaded"));
```

其中，`loadResource`函数会创建一个`link`元素，并将其插入到文档的`head`中。

#### Resource 状态

资源有四种状态：**已卸载**、**正在加载**、**已加载**、**失败**：

-**已卸载**：资源尚未加载，此状态**只能**转入**正在加载**状态。 -**正在加载**：资源正在加载；该状态**只能**变为**已加载**或**失败**状态。 -**已加载**：资源加载成功，此状态为最终状态，**不得**更改。 -**失败**：由于网络错误或其他原因导致资源加载失败；此状态为最终状态，**不得**更改

```plain
                     loaded
                   ↗
unloaded → loading
                   ↘
                     failed
```

#### Resource 加载策略

可能会有许多资源，每个资源包含一个或多个模块，一个资源可以导入一个或多个其他模块，这些模块位于其他资源中。对于现代浏览器和 Node.js 来说，使用主机提供的模块系统来加载资源比手动管理这些资源要稳定可靠得多。然而，对于旧版浏览器来说，会采用备用的模块系统。

一个应用的加载分为以下步骤：

1. Farm 生成一个包含初始资源和引导代码的 HTML 文件。
2. 浏览器解析 HTML 后，将加载初始资源。
   1. 如果初始资源成功加载，其中包含的模块将被注册。
   2. 如果初始资源无法成功加载，将抛出错误。
3. 在所有初始资源加载并且模块被注册之后，引导代码将加载入口模块。
4. 在入口模块执行期间，会加载更多的模块。对于每个模块：
   1. 如果模块之前已经被初始化过，则返回缓存的模块引用。
   2. 如果模块之前已经被初始化过，但是以错误结束，会抛出错误。
   3. 如果模块已经被注册但尚未初始化，尝试初始化该模块。
      1. 如果模块初始化成功，将其保存在缓存中，并返回模块引用。
      2. 如果模块初始化不成功，会抛出错误，并将错误保存在缓存中。
   4. 如果模块尚未被注册，则找出包含该模块的资源，加载该资源，在资源加载完成后，尝试初始化该模块。

#### 例子

假设在一个应用程序中有四个资源：`foo.js`、`bar.js`、`baz.js`和`ops.js`，每个资源都包含一个或多个模块：

- foo.js
  - module_a.js
  - main.js (imports module_a.js and module_b.js)
- bar.js
  - module_b.js (dynamically imports module_c.js)
- baz.js
  - module_c.js (imports module_a.js and dynamically imports module_d.js)
- ops.js
  - module_d.js

关系图如下所示：

```plain
         main.js
           ↙ ↘︎
module_a.js   module_b.js
    ↑              ↓ (dynamic import)
     --------- module_c.js
                   ↓ (dynamic import)
              module_d.js
```

##### 现代浏览器和 Node.js 中的情况如下：

HTML 文件将包含`foo.js`和`bar.js`，因为它们是初始资源，而`baz.js`和`ops.js`将在使用时动态加载。初始资源的加载顺序是经过拓扑排序的，在这个示例中，首先加载`bar.js`，然后稍后加载`foo.js`。

```html
<html>
  <head>
    <script>
      globalThis.__acquire_farm_module_system__ = function () {
        /* module system setup */
      };
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
import("baz").then(() =>
  console.log(__acquire_farm_module_system__().require("module_c.js"))
);

// foo.js
// register module_a.js and main.js in this resource
// module main.js
const { a } = __acquire_farm_module_system__().require("module_a.js");
const { b } = __acquire_farm_module_system__().require("module_b.js");

console.log(a);
console.log(b);

// baz.js
// module_a.js has been loaded in main.js
const { a } = __acquire_farm_module_system__().require("a.js");

// ops is loaded as it contains module_d.js
import("ops.js").then(() =>
  console.log(__acquire_farm_module_system__().require("module_d.js"))
);
```

##### 旧版浏览器

在旧版浏览器中，加载初始资源与现代浏览器几乎相同，但仍存在两个区别。第一个区别是，资源不再是模块，因此缺少属性`type="module"`。第二个区别是，使用动态添加的`<script>`元素来异步加载动态导入的资源。

```js
// baz.js
// instead of using code below:
import("ops.js").then(() => {
  /* after module is loaded */
});

// using code like this (simplified version):
function __farm_dynamic_import__(resourceName) {
  return new Promise(function (resolve, reject) {
    var s = document.createElement("script");

    s.src = "ops.js";
    s.onload = function () {
      // after module is loaded
      resolve(__acquire_farm_module_system__().require("module_d.js"));
    };
    s.onerror = function (error) {
      reject(error);
    };
    document.body.appendChild(s);
  });
}

__farm_dynamic_import__("ops.js").then((m) => {
  /* after module is loaded */
});
```
