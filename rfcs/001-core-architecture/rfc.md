- Feature Name: Farm Core Architecture Design
- Start Date: 2022-07-18
- RFC PR: [farm-fe/rfcs#0000](https://github.com/farm-fe/rfcs/pull/0000)

# Summary
This is an RFC to design how to implement a super fast web compiler using the rust language. The new designed compiler should inherit all advantages of existing tools like webpack and vite, but avoid their disadvantages and extremely faster.

# Motivation
As the web project scales, building performance has been the major bottleneck, a web project compilation using webpack may cost 10min or more, a hmr update may cost 5s or more, heavily reduced the efficiency.

So some tools like vite came out, but vite is using native esm and unbundled, the compatibility and the huge numbers of module requests becomes the new bottleneck. And vite is so fast as it uses esbuild which is written in go, which takes performance advantages of the native. But because esbuild can not be used in production for now, vite uses rollup as bundler in production to solve compatibility issue and esbuild stabilization issue, makes the dev and prod greatly different.

Actually we can take advantages of both webpack and vite, and avoid all of their disadvantages. Webpack is slow, we can use system level language (Rust) to greatly improve building performance; Vite is unbundled which means the caching can be finer than webpack, but it has problems like inconsistency(dev and prod) and huge requests(may slow down resource loading even crash browser), we can use some module merging strategy to reduce the request numbers without losing cache granularity.

As we discussed above, Farm is a web building tool aim to be faster(both building performance and resources loading performance) and more consistent, take advantages of existing tools and discard their disadvantages. But Farm is not aim to be a universal bundler, we just focus on web project compiling, which means our inputs are mainly web assets html, js/jsx/ts/tsx, css/scss/less, png/svg/... and so on, and every design we made will be browser first. Though universal bundler is not our first goal, you can achieve whatever you want by plugins.

# Guide-level explanation
Farm will be divided into two parts:
- **Compilation core implemented by Rust**: All the compilation flow like resolving, loading, transforming, parsing and dependencies analyzing are charged by Rust compilation core. The Rust core is not visible to users, it is used privately by Farm's npm packages.
- **CLI and Node API implemented by Typescript**: For the users, they only need to install Farm's npm packages which are written in typescript, then all functionalities are available. The npm packages encapsulate the Rust core and provide CLI and Node API for users.

All performance sensitive actions will be implemented in Rust, and others will be implemented in typescript. Using typescript because farm can easily share js community, for example dev-server middleware. And the users should only care about the npm packages.

Farm npm packages is designed to provide two kind of usages: CLI or Node Api, CLI is provided by the farm team to use farm easily; Node Api is provided for the advanced developers who want to build tools on top of Farm.

## 1. CLI Usage
Two kind of official cli provided: `create-farm-app` for creating a farm project with official templates and `@farmfe/cli` for starting or building a farm project.

### 1.1 Using `create-farm-app` to create a project
```bash
npx create-farm-app # create a farm project using official templates
npm start # start the project using farm
npm run build # build the project using farm
```

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

then run `npm start` at the project root and visit `http://localhost:7896`.

## Node Api Usage
Farm's Api export internal compiler, middleware, watcher, config handler and server, the developer can build their own tools based on these functionalities.

To use farm's api, first install the `@farmfe/core` npm package:
```bash
npm i -D @farmfe/core # install required package into the project
# or `yarn add --dev @farmfe/core` if you use yarn
# or `pnpm add -D @farmfe/core` if you use pnpm
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.