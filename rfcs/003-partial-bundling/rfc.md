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

Unlike other bundlers, Farm will not trying to bundle everything together and then split them out using something like `splitChunk`, on the opposite, Farm will bundle projects into several output files directly. For example, if there are hundreds of modules when we need to launch a html page, Farm will trying to bundle them into 20-30 output files. For each output file Farm will do `Partial Bundling`.

Farm's goal of Partial Bundling is to:
1. **Reduce request numbers**: Make hundreds or thousands of module requests reduce to 20-30 requests, which would make resource loading faster.
2. **Increase cache hit rate**: When a modules changed, makes sure that only a few output files are affected, so more cache can be used for a online project.

For traditional bundlers, we may have a hard time to configure complex `splitChunk` to achieve the goal above, but in Farm, it is supported natively through `Partial Bundling`.

# Motivation


# Reference-level explanation
