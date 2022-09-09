- Name: Partial Bundling
- Start Date: 2022-09-09
- RFC PR: 

# Summary
This RFC will cover following:
* what is `partialing bundling` and why we design `partial bundling`
* the default `partial bundling` strategy
* the implemention details of `partial bundling`

# Motivation
By now the front end community mainly has 2 methods to handle modules bundling while compiling: `bundling or not`. The typical tools of these 2 methods are webpack and vite, webpack bundles everything together and split it if necessary while vite uses native esm and unbundled in dev mode.

But these two methods both have their disadvantages:
* webpack bundles everything together means that if a module changes the whole bundle will change and need to regenerate, this is not good for caching and HMR performance, though we can use splitChunks to optimize but splitChunks is really complex to config and it's hard to ensure that the resources loading performance is really improved.
* vite chooses another way that uses native esm and unbundled(with node modules pre-compiled using esbuild) in dev mode, and only compile the module when it is requested. This is really fast for small project, but then the project scales, there may be thousands of module which there are thousands of requests, which will crash the browser. And because native esm is not supported by many old browsers, and hundreds or thousands of requests are not acceptable when deploy to production, vite uses rollup to bundle modules when build in prod mode, but this make the dev and prod differ greatly.

We want to solve the above problems and design a more powerful module bundling strategy in Farm. So there is `partial bundling` come out: **unbundled first but partial bundle some modules when the concurrent resource requests exceed the limit**. `partial bundling` can solve above problems:
* only related modules will be bundled together, if a module changed, only a few related production resources changed.
* after `partial bundling`, the requests number won't exceed the limit (the default limit is 25, we tested that the 25 concurrent requests are the best), and the dev mode and prod prod can use the same strategy to ensure consistence.

We will design a powerful strategy to control when and how to do `partial bundling`, and how `partial bundling` cooperates with HMR and Persist Caching.

# Reference-level explanation
## 1. Terms
The following are terms we will use in the rest of this RFC:
* **`Partial Bundling`**: Bundling a set of related modules together, the result of `partial bundling` is `ResourcePot` (a `ResourcePot` contains executable resource css, js and so on).
* **`ModuleBucket`**: The unit of `partial bundling`, a `ModuleBucket` may contain one or more modules, by default module within the same packages belong to the same `ModuleBucket`.
* **`ModuleGroup`**: A group of modules which is needed from a specified entry. For example, if A is an js entry, then all non-dynamic depenencies of A are in the same `ModuleGroup`, the dynamic dependencies (imported using `import()`) will be in other `ModuleGroup`, as dynamic dependencies are not necessary for first execution.
* **`ResourcePot`**: A abstract resource after partial bundled modules. A `ResourcePot` may generate one or more resources.


## 2. Partial Bundling Flow
```text
Analyze Module Group -> For Each Module Group -> Analyze Module Bucket -> Merge Module Bucket -> Output ResourcePot
```

## 3. ModuleBucket Analyzing