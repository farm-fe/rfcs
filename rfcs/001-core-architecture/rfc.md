- Name: Farm Core Architecture Design
- Start Date: 2022-07-18
- RFC PR: [farm-fe/rfcs#3](https://github.com/farm-fe/rfcs/pull/3)

# Summary
This is an RFC to design how to implement a super fast web compiler with Typescript and Rust. The new designed compiler should inherit all advantages of existing tools like webpack and vite, but avoid their disadvantages and extremely faster.

# Motivation
As the web project scales, building performance has been the major bottleneck, a web project compilation using webpack may cost 10min or more, a hmr update may cost 10s or more, heavily reduced the efficiency.

So some tools like vite came out, but vite is using native esm and unbundled in dev mode, the huge numbers of module requests becomes the new bottleneck, it may crash the network panel when there are thousands of module requests.

And vite is so fast as it uses esbuild which is written in go, which takes performance advantages of the native. But esm is not available for legacy browsers and esbuild is not strong enough to be used in production for now, so vite uses rollup as bundler in production to solve compatibility issue and esbuild stabilization issue, which brings new problems, for example, the dev and prod's behavior maybe greatly different, and rollup is greatly slower than esbuild as it's written in Typescript.

Actually we can take advantages of both webpack and vite, and avoid all of their disadvantages. Webpack is slow, we can use system level language (Rust) to greatly improve building performance; Vite is unbundled which means the caching can be finer than webpack, but it has problems like inconsistency(dev and prod) and huge requests(may slow down resource loading even crash browser), we can use some partial bundling strategy to reduce the request numbers without losing cache granularity.

As we discussed above, Farm is a web building tool aim to be faster(both building performance and resources loading performance) and more consistent, take advantages of existing tools and discard their disadvantages. Farm team will focus on web project compiling at present, which means our inputs are mainly web assets html, js/jsx/ts/tsx, css/scss/less, png/svg/... and so on, and every design we made will be browser first. Though universal bundler(bundle everything together and output various format) is not our first goal currently, you can achieve whatever you want by plugins.

Our goal is to design a real modern web compiler which is super fast, stable, consistent, compatible, and modern web tech first. What we want is a real next generation building tool.

> Note: this RFC mainly covers the architecture, the details of each part will be split to a separate RFC.

# Reference-level explanation
This section covers the technical design details of Farm.

## 1. Design Philosophy
* **Performance first**: Everything will be written in rust as long as we can, only several parts  which is not the performance bottleneck will be written in JS
* **Rollup style plugin system**: Easy to create your own plugins and easy to migrate your plugins/projects from rollup/vite/webpack. 
* **first class citizen support of all web assets**: We won't need to transform everything to Javascript any more, we treat anything as first class citizen, in the farm core basic asset like `html`, `js/jsx/ts/tsx`, `css/scss`, `png/svg/...` will be support by default, you can using plugins to support more assets.
* **browser first**: Farm's production aims to run in the browser/nodejs(only for SSR), we won't be a universal bundler and only try our best to improve web performance and efficiency.
* **Unbundled first**: We only partial bundling modules together when the module numbers of size reach our limits, when partial bundling modules we will use a powerful partial bundling strategy to control the resources request numbers without losing cache granularity.
* **Consistence first**: We will make sure the development and production exactly the same by default, what you see in development will be the same as what you got in production.
* **Compatibility**: Farm will work with both legacy and modern browser.

## 2. Terms
The definition of terms Farm used:
* **Module**: Basic compilation unit, it may be a file or a virtual module, for example, all kinds of web asset like `js, ts, jsx, tsx, css, scss, png, svg...`, or virtual modules implemented by plugins.
* **Resource**: A `Resource` presents a `js/css/html/png/svg/map/..` file and may contain may original modules.
* **ResourcePot**: A `ResourcePot` may generate one or many `Resource`. The `ResourcePot` comes from many partial bundled `Module`s, for example many `js` modules bundled to one `ResourcePot`, and one `ResourcePot` generates a `.js` file and a `.js.map` file.
* **ModuleGroup**: All static imported modules from an entry will be in the same `ModuleGroup`.
* **ModuleGraph**: Dependency graph of all resolved modules
* **ResourcePotGraph**: Dependency graph of the `ResourcePot`.
* **ModulePot**: A `ModulePot` is a group of modules that will always be together, which means the modules in the same `ModulePot` will always in the same `ResourcePot`.


## 3. Architecture
Farm core contains two parts:
* Typescript is responsible for config/plugins handling, DevServer and FileWatcher(for HMR)
* Rust core is responsible for the compilation details, including module resolving/loading/parsing and resource optimizing/generating.

See detailed graph below:

![Farm Architecture](./resources/farm-architecture.png)

The details of each part will be designed in following sections.

## 4. Compilation Context
Compilation Context contains all shared info across the compilation, this section covers the details of CompilationContext.

CompilationContext can be accessed in the plugins through hook params:
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

The definition of `CompilationContext` is:
```rust
/// Shared context through the whole compilation.
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
`meta` is shared data through the compilation, for example, SourceMap of Swc. Plugins can also custom data and insert it into `meta.custom`.

Other data structures like module_graph or resource_graph are constructed during the compilation lifecycle of the Farm core.

The details of each field of `CompilationContext` will be introduced in a separate RFC, for example, `ModuleGraph` and `ModuleGroupMap` are related to `partial bundling algorithm` and `CacheManager` is related to `cache system`.

## 5. Compilation Flow And Plugin Hooks
We divide the compilation flow into two stages(which we borrowed from rollup) - Build Stage and Generate Stage, and the compilation flow is all about hooks, see the graph below for details:

![plugin-hooks](./resources/plugin-system.png)

There three kinds of hooks (the same as rollup):
* `first`: The hooks execute in serial, and return immediately when a hook returns `non-null` value. (The `null` means `null` and undefined in js, `None` in rust).
* `serial`: The hooks execute in serial, and every hook's result will pass to the next hook, using the last hook's result as final result.
* `parallel`: The hooks execute in parallel in a thread pool, and should be isolate.

### 5.1 Build Stage
The goal of `Build Stage` is to build a `ModuleGraph`.

Starting from the user configured compilation entry, resolving, loading, transforming and parsing the entry module, then analyze its dependencies and do the same operation for the dependencies again util all related modules handled.

Each module's building flow as follow.
```txt
./index.html -> resolve -> load -> transform -> parse -> moduleParsed -> analyzeDeps ----> resolve deps recursively
```

Each module will build in a separate thread in a thread pool, and after `analyzeDeps` we back to resolve again for each dependency.

#### 5.1.1 Resolve
The resolve hook is charge of Resolving a module, return its id and related properties. See the hook definition, parameter and result below:

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
  /// the start location to resolve `specifier`, being [None] if resolving a entry or resolving a hmr update.
  pub importer: Option<ModuleId>,
  /// for example, [ResolveKind::Import] for static import (`import a from './a'`)
  pub kind: ResolveKind,
}

#[derive(Debug, Default, Serialize, Deserialize)]
#[serde(rename = "camelCase", default)]
pub struct PluginResolveHookResult {
  /// resolved path, normally a resolved path.
  pub resolved_path: String,
  /// whether this module should be external, if true, the module won't present in the final result
  pub external: bool,
  /// whether this module has side effects, affects tree shaking
  pub side_effects: bool,
  /// the query parsed from specifier, for example, query should be `{ inline: true }` if specifier is `./a.png?inline`
  /// if you custom plugins, your plugin should be responsible for parsing query
  /// if you just want a normal query parsing like the example above, [farmfe_toolkit::resolve::parse_query] should be helpful
  pub query: HashMap<String, String>,
}
```

#### 5.1.2 Load
Loading the module's content based on `id`, return **String** based content. If load binary content like images, it should be encoded to String first (usually base64). We force to use String because of the serialization, the value should be easy to pass to the js plugins and rust plugin, so we force it be serializable.

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
  /// the type of the module, for example [ModuleType::Js] stands for a normal javascript file,
  /// usually end with `.js` extension
  pub module_type: ModuleType,
}
```

#### 5.1.3 Transform
Transforming the module's content base on `loaded source content`, the transforming is String in and String out. If you want to share Farm's internal ast, you can use `SWC plugins`

* **`Hook Kind`**: `serial`

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
  /// source content after load or transformed result of previous plugin
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
  /// transformed source content, will be passed to next plugin.
  pub content: String,
  /// you can change the module type after transform.
  pub module_type: Option<ModuleType>,
  /// transformed source map, all plugins' transformed source map will be stored as a source map chain.
  pub source_map: Option<String>,
}
```

#### 5.1.4 Parse
Parsing the module's content to a internal `Module` instance, parsing source code to ast and so on.

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
Process and transform the module, may change any property of the module, for example, transform the parsed ast (using swc transformer and swc plugins).

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
Analyzing the dependencies of each module. For example, for `import a from './a'`, the result should be `{ specifier: './a', kind: ResolveKind::Import }`

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
  /// analyzed deps from previous plugins, if you want to analyzer more deps, you must push new entries to it.
  pub deps: Vec<PluginAnalyzeDepsHookResultEntry>,
}

#[derive(Debug, PartialEq, Eq)]
pub struct PluginAnalyzeDepsHookResultEntry {
  pub source: String,
  pub kind: ResolveKind,
}
```

### 5.2 Generate Stage
The goal of generate stage is generating deployable resources (js, html, css, wasm and so on).

The generation flow as follow:
```plain
ModuleGraph generated in build stage ---> optimize_module_graph -> analyze_module_graph -> partial bundle module -> process_resource_graph ---> for each resource in parallel ---> render_resource -> optimize_resource -> generate_resource_file -> write_resource_file
```

The detailed hooks as below:
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

## 6. Plugin System
We have designed all hooks in last section, this section we'll discuss how to register, load and execute Farm's plugin.

Farm plans to support two kinds of plugins:
* **Rust Plugin**: Written in Rust and distributed as dynamic library, provide the best performance and can implement all compilation hooks. Which is the recommended way to write plugins.
* **JS Plugin**: Written in Javascript(Typescript) and distributed as NodeJs executable script file. It will slow down the compilation process and can only implement limited hooks. Farm support Js Plugins because Farm wants to share existing community tools written in Js/Ts, as many web tools (for example, less) do not have a Rust version for now.

### 6.1 Rust Plugin
Rust plugins are our first goal cause it's fast and powerful, but when we encounter some ability that Rust ecosystem do not provide, Js Plugins would be the fallback.

Using `dynamic library`(`.dylib` on macos, `.dll` on windows and `.so` on linux) to distribute Rust Plugin because it is the most performant way, we investigate many methods but none of them are suitable:
* **abi_stable**: A Rust library support FFI safe calling, but we have to use its primitive std types(like `RString` to replace std `String`), and these types are not rkyv serializable (Serialize is necessary for supporting Persist Cache). We will have a hard work if we have to support both.
* **wasm**: This is what SWC currently used, the advantage of wasm plugin is that it's portable and performant. But wasm runtime can not shared memory with Rust, need to serialize and copy to the wasm runtime and copy back to Rust and deserialize after plugin execution. The hooks may be called thousands of times and it is really inefficient. Also wasm plugins do not support parallel well, it is hard to make sure the CompilationContext consistent between threads, especially we need to serialize/deserialize it.

So we decided to choose plain `dynamic library` to support Rust Plugin. We loaded the dynamic library and execute it as a internal plugin, thus we can avoid many problems above:
* **best performant**: the dynamic library plugin is execute as the same fast as internal plugin, as we can share memory and fully parallelize.
* **single type**: we do not need to provide extra types for dynamic library plugin.

But there are problems we need to resolve:
* **cross platform distribute**: the dynamic library plugin is not portable, it needs to be built separately for different platforms, when built on one machine, it needs cross-compile.
* **shared type memory layout**: we shared the same type as internal plugins for dynamic library plugins, but if the types changed, the plugins needs to rebuild too, as their memory layout changed, if the plugin is not rebuilt, a `segment fault error` thrown.

We solve the problem above as follow:
* **provide portable cross-compile tool**: we'll provide a independent tool for building and publish plugin, the plugin author only needs to use something like `farm plugin build`, `farm plugin plugin`, then Farm will handle all the details.
* **minimize the shared type and makes sure it's stable**: we only expose the necessary types to plugin, and the mutable part will be something like `Custom(Box<dyn Any>)`, so the types will rarely change once it's stable. And we also record the version of current types, if we really need to change the types, we'll bump the type schema version and inform users to upgrade their legacy plugins.

### 6.2 Js Plugin
We only plan to provide limited js plugin support, which means the js plugin can only implement `build_start`, `resolve`, `load` and `transform`, `build_end`, `finish` hook. Because data transform between rust and js is expensive, if we send all data like ModuleGraph to the js side, it will greatly slow slow down the compilation, which violates our goal.

And the js plugin should specify the `filters` field too, to specify which module it is willing to process, we add this limitation for performance too.

A example Js plugin:
```js
export default {
  name: 'js-plugin', // plugin name
  priority: 10, // priority, controls the execute order of plugins
  resolve: {
    // specify the target modules this plugin wants to process
    filters: {
      importers: [],
      specifiers: ['from_js_plugin'],
    },
    // hook callback
    executor: async (param, context) => {
      console.log(param, context);

      if (!param.caller) {
        const resolved = await context.resolve({
          ...param,
          specifier: './from_js_plugin',
          caller: 'js-plugin',
        });
        console.log('call internal resolve in js', resolved);
        resolved.id += '.js-plugin';
        return resolved;
      }
    },
  },
},
```

## 7. Cache System
> We only introduce our goal of the cache system here, we will design the details in a separate RFC.

The goal of `Cache System` is to provide a universal cache across the whole compilation flow, which means:
* It covers HMR, the HMR process only update some necessary module in the ModuleGraph and final resources, other module will remain unchanged as they hit cache.
* It covers `Compilation flow Cache`, this means a module only be processed once.
* It covers `Disk Cache`, this means the cache can be serialized to disk and restore from disk, providing similar ability like webpack5's persist cache.

With the cache system, Farm can provide extremely fast experience for HMR, and hot start with cache.

The cache ability is designed in `CacheManager`, we will cover details in a separate RFC.

## 8. Partial Bundling And Resource Generation
> We only introduce our goal of the Partial Bundling strategy here, we will design the details in a separate RFC.

Farm use `Partial Bundling` instead of `Bundling` cause we are `unbundled` first, Farm only merges modules together when the request numbers of module exceeds the request number limit. We have done a lot of tests and the result shows that 20-30 requests will be the most performant in modern browsers. When there are thousands of modules, we will merge the thousands of modules into 20-30 resources using our optimized strategy.

For the Partial Bundling strategy, Farm will make sure that:
* All shared module and dynamic module wound be in a separate resource as possible. This is similar to code split in traditional bundlers but we will control more precisely what are the modules in each resource.
* Related modules would be in the same resource as possible, for example, the modules under the same directory, they are more related as developer usually put all related assets together.

And also, the developer can config which modules should be merged together. We will cover the details in a separate RFC.

## 9. User Interface Design
Farm is designed to be divided into two parts:
- **Compilation core implemented by Rust**: All the compilation flow like resolving, loading, transforming, parsing, dependencies analyzing and code optimization/generation are charged by Rust. The Rust part is not visible to users, it is used privately by Farm's npm packages. Only a few apis exported from the core for Rust plugin authors.
- **CLI and Node API implemented by Typescript**: For the users, they only need to install Farm's npm packages which are written in typescript, then all functionalities are available. The npm packages encapsulate the Rust core and provide CLI and Node API for users. And all plugins are distributed as npm packages too.

All performance sensitive actions will be implemented in Rust, and others will be implemented in typescript. Using typescript because farm want to easily share js community, for example dev-server middleware and transformer(less, markdown and so on). And the users should only care about the npm packages.

Farm npm packages are designed to provide two kinds of usages: CLI or Node Api, CLI is provided by the farm team to use farm easily; Node Api is provided for the advanced developers who want to build tools on top of Farm.

## 1. CLI Usage
Two kind of official cli provided: `create-farm-app` for creating a farm project with official templates and `@farmfe/cli` for starting or building a farm project.

### 1.1 Using `create-farm-app` to create a project

```bash
npx create-farm-app # create a farm project using official templates
npm start # start the project using farm
npm run build # build the project using farm
```
`create-farm-app` will create a react based project at first, more templates and feature selection will be added in the feature.

#### 1.2 Using `@farmfe/cli` to start/build a project
First you need to create a project by yourself, then install the farm project as below:
```bash
npm i -D @farmfe/core @farmfe/cli # install required package into the project
# or `yarn add --dev @farmfe/core @farmfe/cli` if you use yarn
# or `pnpm add -D @farmfe/core @farmfe/cli` if you use pnpm
```

then create a `farm.config.ts` in the project root(where you call npm start), the config looks like:
```ts
import { defineConfig } from '@farmfe/core';

export default defineConfig({
  // shared config
  root: process.cwd(),
  // compilation config
  compilation: {
    input: {
      index: './index.html'
    },
    output: {
      // details will be described later...
    },
    // details will be described later...
  },
  // dev server config
  server: {
    port: 7896
  }
});
```

then run `farm start` at the project root and visit `http://localhost:7896`.

Farm CLI will provide following commands:
* **start**: start a farm project in dev mode, enable hmr and dev server by default.
* **build**: build a farm project in prod mode.
* **watch**: similar to start except `watch` won't launch dev server.
* **preview**: launch a server to preview the built resources.


## 2. Node Api Usage
Farm's Api export internal compiler, middleware, watcher, config handler and server, the developer can build their own tools based on these functionalities.

To use farm's api, first install the `@farmfe/core` npm package:
```bash
npm i -D @farmfe/core # install required package into the project
# or `yarn add --dev @farmfe/core` if you use yarn
# or `pnpm add -D @farmfe/core` if you use pnpm
```

Example usage:

* Using start/build function directly:
```ts
import { start, build, watch } from '@farmfe/core';

// Start a project with a dev server and file watcher. First build the project in dev mode, then the dev server serves compiled resources and the file watcher watches file changes and trigger re-compile
start({ configPath: './custom.config.js' });

// First build the project in develop mode, then watch a project with a file watcher
watch({ configPath: './custom.config.js' });

// Only build a project in production mode
build({ configPath: './custom.config.js' });
```

* Using internal compiler
```ts
// Note that the internal compiler only provide interface to compile or update a project, do not contain functions like Dev Server or HMR, you should use DevServer and Watcher API separately
import { Compiler, resolveUserConfig } from '@farmfe/core';

const userConfig = resolveUserConfig('./custom.config.js');
const compiler = new Compiler(userConfig);
// async compile
await compiler.compile();
// sync compile
compiler.compileSync();
// async update
await compiler.update(['./index.ts']);
// sync update
compiler.updateSync(['./index.ts']);
```

* Using dev server and watcher, you can use both of them, or use only one of them and custom another
```ts
// DevServer should cooperate with compiler, it will read resources from compiler and serve it.
// Note that DevServer also contains hmr and web socket or lazy compilation middleware if hmr or lazyCompilation is enabled but does not contains file watcher.
import { Compiler, DevServer, FileWatcher } from '@farmfe/core'

// ...
const compiler = new Compiler(config);
const devServer = new DevServer(compiler, config);
const fileWatcher = new FileWatcher(compiler, config);

// compiling
await compiler.compile();
// watching file system and trigger compiler.update()
await fileWatcher.watch();
// listening the specified port
await devServer.listen();
```