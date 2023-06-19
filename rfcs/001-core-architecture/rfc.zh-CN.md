- 主题: Farm 核心架构设计
- 开始时间: 2022-07-18
- RFC PR: [farm-fe/rfcs#3](https://github.com/farm-fe/rfcs/pull/3)

# 总结

这个RFC设计了使用Typescript和Rust实现极速的Web编译器。该编译器设计继承现有工具（如webpack和vite）的优点，同时避免它们的缺点并且将会更加高效。

# 动机

随着 web 项目的复杂度以及规模的不断增加, 构建性能成为了 web 开发中的一个重要的瓶颈, 使用 webpack 构建的 web 项目可能需要十分钟甚至更长时间, 并且 (HMR) 的更新的时间可能花费数十秒甚至更长时间, 降低了开发效率, 这对于开发体验来说是不可接受的。

然后就出现了类似 vite 的工具来解决这个问题, vite 采用原生 ESM 和 unbundled 解决开发模式的问题, 这样就可以避免 webpack 的构建过程中的大量的 IO 操作。但是缺点是原生 ESM 模块以及动态加载模块, 导致大量开发模块的请求成为了新的瓶颈, 当你有数千个模块的请求时, 可能会导致网络面板崩溃

Vite 的速度非常快, 因为使用了 Go 编写的 Esbuild, 并利用本地性能。然而，对于旧版浏览器来说ESM不可用，而且esbuild还不稳定，不能在生产中使用。因此，在生产环境中vite使用Rollup作为打包工具来解决兼容性和稳定性问题。这引入了新的问题，例如开发和生产环境之间行为差异显著，并且与esbuild相比由于Rollup是用Typescript编写的导致性能较慢。

我们可以充分利用webpack和vite，同时避免它们的缺点。Webpack构建速度较慢，但是我们可以使用系统级语言（Rust）来显著提高构建性能。Vite 的 unbundled, 这意味着缓存比webpack更细粒度，但它存在一些问题，如不一致性（在开发和生产之间）和大量请求（可能会减慢资源加载甚至导致浏览器崩溃）。我们可以采用部分打包策略来减少请求数量而不失去缓存粒度。

综上所述, Farm是一个旨在更快（无论是构建性能还是资源加载性能）和更一致的Web构建工具，利用现有工具并丢弃它们的缺点。 Farm团队最初将专注于Web项目编译，这意味着我们的输入主要是Web资产，例如HTML、JS/JSX/TS/TSX、CSS/SCSS/LESS、PNG/SVG等。我们做出的每个设计决策都将优先考虑浏览器兼容性。尽管通用打包程序（将所有内容捆绑在一起并输出各种格式）目前不是我们的主要目标，但您可以通过插件实现任何想要达到的效果。

我们的目标是设计出一个真正现代化的 web 编译器, 它将会是快速, 稳定, 构建产物一致, 以及兼容性良好, 优先考虑现代 web 技术, 我们希望创建出一个能够满足开发者需求的下一代构建工具


> 注意：本RFC主要涵盖核心架构设计, 后续每个部分的详细信息将拆分为单独的RFC。

# 参考

该RFC涵盖 Farm 的核心架构设计 

## 1. 设计理念

* **性能优先**: 我们的目标是所有都用 Rust 编写, 只有极少数或者不是性能瓶颈的部分才会使用 js。
* **Rollup 风格的插件系统**: 以类似 Rollup 的方式设计插件系统, 使得插件可以更加灵活并且更加简便的使用, 并且易于从 Rollup/Vite/Webpack 迁移您的插件/项目。
* **对所有 web 资源提供支持**: 我们不在需要把所有内容都转换到 javascript, 我们将把任何东西视为一等公民, 在 Farm 中, 默认支持基本资源(如 `html`、`js/jsx/ts/tsx`、`css/scss`、`png/svg/...`)，您可以使用插件来支持更多资源。
* **浏览器优先**: Farm 的目标是在浏览器或者 nodejs 中运行(仅用于 SSR), 我们不会成为通用打包工具, 而是专注提高在 web 的性能和效率。
* **首选非捆绑的方式**: 当模块数量或大小达到限制时，我们只会构建部分模块。当部分模块构建时，我们将使用强大的局部捆绑策略控制资源请求数量而不失去缓存粒度
* **始终保持一致性**: 开发环境和生产环境产物完全相同。您在开发中看到的内容将与生产中获得的内容相同。
* **兼容性**: Farm 将适用于传统和现代浏览器。

## 2. 术语

Farm使用的术语定义如下:

* **模块(Module)**：基本编译单元，可以是文件或虚拟模块。例如各种Web资源，如`js、ts、jsx、tsx、css、scss、png、svg...`，或由插件实现的虚拟模块。

* **资源(Resource)**：表示一个 `js/css/html/png/svg/map/..` 文件，并可能包含多个原始模块。

* **资源池(ResourcePot)**：可以生成一个或多个 `Resource`。 `ResourcePot` 可能是一些部分打包的 `Module`, 例如将许多 `js` 模块捆绑到一个 `ResourcePot`, 并且一个 ResourcePot 会生成 `.js` 文件和 `.js.map` 文件。

* **模块组(ModuleGroup)**: 所有从入口静态导入的模块都在同一 ModuleGroup 中。

* **模块依赖图(ModuleGraph)**：所有已解析模块的依赖关系图。

* **资源池关系依赖图(ResourcePotGraph)**：'ResourcePot' 的依赖关系图.

* **模块池(Module Pot)**: 'Module Pot' 是一组总是在一起的 module, 这意味着相同 'Module Pot' 中的 modules 将始终位于相同 'Resource Pot'.

## 3. 架构

Farm 核心由两部分组成：

* Typescript 负责处理配置/插件、开发服务器 和文件监视器（用于 HMR）。

* Rust core 负责编译核心部分细节，包括模块解析/加载/解析和资源优化/生成。

请参见下面的详细图表：

![Farm 架构](./resources/farm-architecture.png)

每个部分的详细信息将在以下章节中设计。


## 4. 编译上下文

编译上下文包含了整个编译过程中的所有共享信息。本小节将详细介绍编译上下文。

插件可以通过钩子访问 CompilationContext。

```rust
struct MyPlugin {}

impl Plugin for MyPlugin {
  fn name(&self) -> String {
    String::from("MyPlugin")
  }
  // access CompilationContext via parameter
  fn resolve(&self, param: &PluginResolveHookParam, context: &Arc<CompilationContext>) -> Result<Option<PluginResolveHookResult>> {
    // ..
  }
}
```

`CompilationContext` 的定义如下:
```rust
/// Shared context throughout the whole compilation.
pub struct CompilationContext {
  pub config: Config,
  pub module_graph: RwLock<ModuleGraph>,
  pub module_group_map: ModuleGroupMap,
  pub plugin_driver: PluginDriver,
  pub resource_graph: ResourceGraph,
  pub cache_manager: CacheManager,
  pub meta: ContextMetaData,
}
```

`meta`是编译过程中共享的数据，例如 swc 的 SourceMap。插件还可以添加自定义数据并将其插入到`meta.custom`中。

其他数据结构，如`module_graph`或者`resource_graph`, 是在Farm核心的编译生命周期内构建的。

关于 `CompilationContext` 的每个字段的详细信息将在单独的RFC中介绍。例如， `ModuleGraph` 和 `ModuleGroupMap` 与 `partial bundling algorithm` 关联,  `CacheManager` 与缓存系统关联。


## 5. 编译流程和插件钩子

我们将编译流程分为了两个阶段 (借鉴于 Rollup) - 构建阶段和生成构建产物阶段, 所有编辑流程都是基于 hooks 完成的, 详见下图

![plugin-hooks](./resources/plugin-system.png)

有三种类型的钩子（与Rollup相同）：

* `first`: 这些钩子按顺序执行，并在某个钩子返回一个“非空”值时立即返回。( null 表示JS中的 null 和 undefined，“None” 表示Rust)

* `serial`: 这些钩子按顺序执行，每个钩子的结果将传递给下一个钩子，使用最后一个hook结果作为最终结果。

* `parallel`: 这些 hook 在线程池中并行执行，并应该被隔离。

接下来我们来介绍一下这两个阶段

### 5.1 构建阶段

构建阶段的目标是去构建出一套模块依赖图。
从入口配置文件开始解析, 加载, 转换并且解析入口模块, 然后分析依赖, 并且对所有依赖模块进行相同的操作, 直到所有模块都被解析完毕。
每个模块的构建流程如下:

```txt

./index.html -> 解析 -> 加载 -> 转换 -> 语法分析 -> 解析模块 -> 分析依赖关系 ----> 递归地解决依赖关系

```
每个模块由线程池中的单独线程构建, 在分析依赖关系之后, 我们将再次返回解析其他依赖项

#### 5.1.1 Resolve

resolve 钩子负责模块解析, 返回当前模块的 id 以及其他的相关属性, 请参阅以下钩子定义, 参数 结果如下

* **`Hook Kind`**: `first`

```rust
fn resolve(
  &self,
  _param: &PluginResolveHookParam,
  _context: &Arc<CompilationContext>,
  _hook_context: &PluginHookContext,
) -> Result<Option<PluginResolveHookResult>> {
  // ...
}

/// Parameter of the resolve hook
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename = "camelCase")]
pub struct PluginResolveHookParam {
  /// the source would like to resolve, for example, './index'
  pub source: String,
  /// the start location to resolve `specifier`, being [None] if resolving an entry or resolving an HMR update.
  pub importer: Option<ModuleId>,
  /// for example, [ResolveKind::Import] for static import (`import a from './a'`)
  pub kind: ResolveKind,
}

#[derive(Debug, Default, Serialize, Deserialize)]
#[serde(rename = "camelCase", default)]
pub struct PluginResolveHookResult {
  /// resolved path, normally a resolved path.
  pub resolved_path: String,
  /// whether this module should be external, if true, the module won't be present in the final result
  pub external: bool,
  /// whether this module has side effects, affects tree shaking
  pub side_effects: bool,
  /// the query parsed from specifier, for example, query should be `{ inline: true }` if specifier is `./a.png?inline`
  /// if you have custom plugins, your plugin should be responsible for parsing the query
  /// if you just want a normal query parsing like the example above, [farmfe_toolkit::resolve::parse_query] should be helpful
  pub query: HashMap<String, String>,
}
```

#### 5.1.2 Load

根据 `id` 加载模块的内容, 返回基于 **String** 的内容, 如果加载类似像图片这样的二进制内容, 则应该首先将其编码为 **base64** 字符串, 我们强制使用字符串, 因为使用序列化可以更好的把值传递到 JS 插件和 Rust 插件中。

* **`Hook Kind`**: `first`

```rust
fn load(
  &self,
  _param: &PluginLoadHookParam,
  _context: &Arc<CompilationContext>,
  _hook_context: &PluginHookContext,
) -> Result<Option<PluginLoadHookResult>> {
  // ..
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename = "camelCase")]
pub struct PluginLoadHookParam<'a> {
  /// the resolved path from resolve hook
  pub resolved_path: &'a str,
  /// the query map
  pub query: HashMap<String, String>,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename = "camelCase")]
pub struct PluginLoadHookResult {
  /// the source content of the module
  pub content: String,
  /// the type of the module, for example [ModuleType::Js] stands for a normal JavaScript file,
  /// usually ending with `.js` extension
  pub module_type: ModuleType,
}
```

#### 5.1.3 Transform

根据 `加载的资源` 转换的模块内容, 转换为字符串输入和字符串输出, 如果您想要共享 Farm 中的内部 `AST` 可以是用 `SWC` 插件

```rust
fn transform(
  &self,
  _param: &PluginTransformHookParam,
  _context: &Arc<CompilationContext>,
) -> Result<Option<PluginTransformHookResult>> {
  // ..
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename = "camelCase")]
pub struct PluginTransformHookParam<'a> {
  /// source content after load or transformed result of the previous plugin
  pub content: String,
  /// module type after load
  pub module_type: ModuleType,
  /// resolved path from resolve hook
  pub resolved_path: &'a str,
  /// query from resolve hook
  pub query: HashMap<String, String>,
}

#[derive(Debug, Default, Serialize, Deserialize)]
#[serde(rename = "camelCase", default)]
pub struct PluginTransformHookResult {
  /// transformed source content, will be passed to the next plugin.
  pub content: String,
  /// you can change the module type after transform.
  pub module_type: Option<ModuleType>,
  /// transformed source map, all plugins' transformed source map will be stored as a source map chain.
  pub source_map: Option<String>,
}
```

#### 5.1.4 Parse

将模块的内容解析为内部“Module”实例，将源代码转换为AST等

* **`Hook Kind`**: `first`

```rust
fn parse(
  &self,
  _param: &PluginParseHookParam,
  _context: &Arc<CompilationContext>,
  _hook_context: &PluginHookContext,
) -> Result<Option<ModuleMetaData>> {
  // ..
}

#[derive(Debug)]
pub struct PluginParseHookParam {
  /// module id
  pub module_id: ModuleId,
  /// resolved path
  pub resolved_path: String,
  /// resolved query
  pub query: HashMap<String, String>,
  pub module_type: ModuleType,
  /// source content(after transform)
  pub content: String,
}
```

#### 5.1.5 Process Module

流程模块的作用是处理和转换模块, 可能会更改模块的任何属性，例如，转换解析后的AST(使用SWC转换器和SWC插件)。

* **`Hook Kind`**: `serial`
```rust
fn process_module(
  &self,
  _module: &mut PluginProcessModuleHookParam,
  _context: &Arc<CompilationContext>,
) -> Result<Option<()>> {
  // ..
}

pub struct PluginProcessModuleHookParam<'a> {
  pub module_id: &'a ModuleId,
  pub module_type: &'a ModuleType,
  pub meta: &'a mut ModuleMetaData,
}
```

#### 5.1.6 Analyze Deps

分析每个模块之间的依赖关系, 例如，`import a from './a'`, 转换之后的结果应该是 `{ specifier: './a', kind: ResolveKind::Import }`

* **`Hook Kind`**: `serial`
```rust
fn analyze_deps(
  &self,
  _param: &mut PluginAnalyzeDepsHookParam,
  _context: &Arc<CompilationContext>,
) -> Result<Option<()>> {
  // ..
}

pub struct PluginAnalyzeDepsHookParam<'a> {
  pub module: &'a Module,
  /// analyzed deps from previous plugins, if you want to analyze more deps, you must push new entries to it.
  pub deps: Vec<PluginAnalyzeDepsHookResultEntry>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename = "camelCase")]
pub struct PluginAnalyzeDepsHookResultEntry {
  /// the source would like to resolve, for example, './index'
  pub source: String,
  /// the start location to resolve `specifier`, being [None] if resolving an entry or resolving an HMR update.
  pub importer: Option<ModuleId>,
  /// for example, [ResolveKind::Import] for static import (`import a from './a'`)
  pub kind: ResolveKind,
}
```

### 5.2 生成阶段

生成阶段的目标是尽可能高效地生成可部署资源（如js、html、css、wasm）等其他资源。

流程如下: 

```plain
ModuleGraph generated in build stage ---> optimize_module_graph -> analyze_module_graph -> partial bundle module -> process_resource_graph ---> for each resource in parallel ---> render_resource -> optimize_resource -> generate_resource_file -> write_resource_file
```

每类钩子代码如下:

```rust
/// Some optimization of the module graph should be performed here, for example, tree shaking, scope hoisting
  fn optimize_module_graph(
    &self,
    _module_graph: &RwLock<ModuleGraph>,
    _context: &Arc<CompilationContext>,
  ) -> Result<Option<()>> {
    Ok(None)
  }

  /// Analyze module group based on module graph
  fn analyze_module_graph(
    &self,
    _module_graph: &RwLock<ModuleGraph>,
    _context: &Arc<CompilationContext>,
  ) -> Result<Option<ModuleGroupMap>> {
    Ok(None)
  }

  /// Partial Bundling modules of the module group map to [crate::resource::resource_graph::ResourceGraph]
  fn partial_bundle_modules(
    &self,
    _module_group: &ModuleGroupMap,
    _context: &Arc<CompilationContext>,
  ) -> Result<Option<ResourceGraph>> {
    Ok(None)
  }

  /// process resource graph before render and generating each resource
  fn process_resource_graph(
    &self,
    _resource_graph: &RwLock<ResourceGraph>,
    _context: &Arc<CompilationContext>,
  ) -> Result<Option<()>> {
    Ok(None)
  }

  /// Render the [Resource] in [ResourceGraph].
  /// May merge the module's ast in the same resource to a single ast and transform the output format to custom module system and ESM
  fn render_resource(
    &self,
    _resource: &mut Resource,
    _context: &Arc<CompilationContext>,
  ) -> Result<Option<()>> {
    Ok(None)
  }

  /// Optimize the final resource, for example, minimize every resource in the resource graph
  fn optimize_resource(
    &self,
    _resource: &mut Resource,
    _context: &Arc<CompilationContext>,
  ) -> Result<Option<()>> {
    Ok(None)
  }

  /// Generate resources based on the [ResourceGraph]
  /// This hook is executed in serial and should update the content inside ResourceGraph
  fn generate_resource(
    &self,
    _resource_graph: &mut Resource,
    _context: &Arc<CompilationContext>,
  ) -> Result<Option<()>> {
    Ok(None)
  }

  /// Write the final output [Resource] to disk or not
  fn write_resource_file(
    &self,
    _resource: &Resource,
    _context: &Arc<CompilationContext>,
  ) -> Result<Option<()>> {
    Ok(None)
  }
```