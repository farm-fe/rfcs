- Name: Farm Partial Bundling
- Start Date: 2023-08-30
- RFC PR: [farm-fe/rfcs#4](https://github.com/farm-fe/rfcs/pull/4)

# Summary
This RFC designs Farm's Partial Bundling strategy, including:
1. What is `Partial Bundling`
2. The Motivation of `Partial Bundling`
3. Internal details of `Partial Bundling`

# What's Partial Bundling
`Partial Bundling` is a strategy that Farm uses to bundle modules, similar to what other bundlers do but the goal of Farm's `Partial Bundling` is different.

Unlike other bundlers, Farm will not trying to bundle everything together and then split them out using something like `splitChunk`, on the opposite, Farm will bundle projects into several output files directly. For example, if there are hundreds of modules when we need to launch a html page, Farm will trying to bundle them into 20-30 output files directly. For each output file Farm will do `Partial Bundling`.

Farm's goal of Partial Bundling is to:
1. **Reduce request numbers**: Make hundreds or thousands of module requests reduce to 20-30 requests, which would make resource loading faster.
2. **Increase cache hit rate**: When a modules changed, makes sure that only a few output files are affected, so more cache can be used for a online project.

For traditional bundlers, we may have a hard time to configure complex `splitChunks` or `manualChunks` to achieve the goal above, but in Farm, it is supported natively through `Partial Bundling`.

# Motivation
There are two main methods of handling modules in web build tools now: Bundling or native ESM. But they both have drawbacks:
* For bundling, bundlers aim to bundle everything together and then split them out for optimization, but splitting is often hard to configure and is hard to balance resources loading performance and cache hit rate manually. 
* For native esm, every module can be compiled, cached separately, but the load performance are heavily affected when there are hundreds of modules.

So I was always thinking that if there is a strategy to avoid these two extremes - maybe we can do partial bundling? we can just bundle the project into several limited, size balanced resources directly and automatically. I named this thinking `Module Merging` - Find a balance between bundle and unbundled, only bundles a few related modules to improve loading performance instead of losing cache granularity.

> I renamed `Module Merging` to `Partial Bundling` later because I think `Partial Bundling` can expresses more accurately what I am thinking.

But some of my friends thought that is what `splitChunks` of webpack does, but I don't think so, they are similar but their basic ideas is different:
* Farm does not always bundle, only if concurrent module requests exceed Farm's threshold, Farm does partial bundling only when necessary.
* Farm does bundling only for performance reason, if one day hundreds concurrent requests are acceptable, then Farm will not bundle any more.

And this fundamental difference also greatly affects the chooses when implement bundling. The detailed designs will be described in following sections.

# Design Philosophy
To achieve our two goals(reduce request numbers and increase cache hit rate), some general optimize principles should be respected:
1. The request numbers should be limited and the size of each output resource should be close to.
2. When changing a few modules, the impact on output resources should be as small as possible.

Based on above principles, following rules are designed:

1. **Mutable and immutable modules should always be in different output files**: For example, if we changed our business code, we would not expect that modules under `node_modules` are affected.
2. **Shared modules should be in isolate output files as long as they can**: For modules shared between multiple entries or dynamic imported entries, they should be in separated output file, so we won't loading unnecessary modules when loading these files.
3. **The max concurrent requests for a resource loading should be between 20-30 by default**: After a lots of tests, we found that 20-30 concurrent requests have best performance when loading resources concurrently.
4. **Related modules should be in the same output file as long as they can**: For example, modules under the same package should be together, if the package is updated, then only a few output files are affected.
    * 4.1 Modules under the same package should be in the same output file.
    * 4.2 Closer module in the dependency tree should be more likely in the same output file.
5. **Output files should be of similar size and min resource size should be greater than 20KB by default**: Avoid buckets effect and make concurrent resources loading faster.
6. **When above rules are conflicts, more requests are preferred**: For example, if `minResourceSize` is violated due to `mutable/immutable modules`, `minResourceSize` will be ignored, because one of farm's philosophy is `Unbundled First`, we prefer more requests.

# Reference-level explanation
This section explains the technical part of Partial Bundling.

## Related Terms That Farm Defines
* **Module**: Basic compilation Unit for Farm, it can be a file loaded for disk or a virtual module generated by plugins.
* **ModuleGraph**: Graph of modules, if a module `imports` another module, then a dependency edge would be created in the graph.
* **ModuleGroup**: All necessary modules for initial execution or dynamic request. For example, all statically imported modules start from `index.ts` or all statically imported modules start from a dynamically imported modules. We'll illustrate it in following sections.
* **ModuleGroupGraph**: Graph of module groups, if any module of module group dynamically imports another module group's entry modules, then a dependency edge would be created in the graph. 
* **ModuleBucket**: Modules which are in the Module Groups. A module can be in many Module Groups, if all the ModuleGroups of two modules are the same, then these two modules are in the same ModuleBucket.
* **ModulePot**: Modules in the same Module Pot would always be in the same output resource. For example, modules in the same package will be in the same ModulePot if there are many packages.
* **ResourcePot**: A Resource Pot consist of one or more Module Pot, and a Resource Pot should produce a `output resource` and a optional `.map` resource

## Partial Bundling Process
![General Bundle Process](./resources/general-process.png)

The partial bundling process is split into 4 steps:
1. Create `ModuleGroupGraph` base on module graph that we created in `Build Stage`.
2. Create `ModuleBuckets` from `ModuleGroupGraph`
3. Create `ModulePots` for each `ModuleBucket` that satisfy our rules fined above.
4. Merge `ModulePots` into `ResourcePot`

we discuss the details of each step in following sections. 

Assume that we already have following `ModuleGraph`:

<img src="./resources/ModuleGraph.png" width="200px" />

## Generate Module Groups
ModuleGroups need to be generated first like illustration below:
![ModuleGroup](./resources/ModuleGroup.png)


## Generate Module Buckets

## Generate Resource Pots

