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

For traditional bundlers, we may have a hard time to configure complex `splitChunk` to achieve the goal above, but in Farm, it is supported natively through `Partial Bundling`.

# Motivation

# Design Philosophy
To achieve our two goals(reduce request numbers and increase cache hit rate), some general optimize principles should be respected:
1. The request numbers should be limited and the size of each output resource should be close to.
2. When changing a few modules, the impact on output resources should be as small as possible.

Based on above principles, following rules are designed:

1. **Mutable and immutable modules should always be in different output files**: For example, if we changed our business code, we would not expect that modules under `node_modules` are affected.
2. **Shared modules should be in isolate output files as long as they can**: For modules shared between multiple entries or dynamic imported entries, they should be in separated output file, so we won't loading unnecessary modules when loading these files.
3. **The max concurrent requests for a resource loading should be between 20-30**: After a lots of tests, we found that 20-30 concurrent requests have best performance when loading resources concurrently.
4. **Related modules should be in the same output file as long as they can**: For example, modules under the same package should be together, if the package is updated, then only a few output files are affected.
    * 4.1 Modules under the same package should be in the same output file.
    * 4.2 Closer module in the dependency tree should be more likely in the same output file.
5. **Output files should be of similar size**: Avoid buckets effect and make concurrent resources loading faster.

# Reference-level explanation
This section explains the technical part of Partial Bundling.

## Terms
* Module
* ModuleGraph
* ModuleGroup
* ModuleGroupGraph
* ModuleBucket
* 

## Partial Bundling Process
Ge

## Generate Module Groups

## Generate Module Buckets

## Generate Resource Pots

