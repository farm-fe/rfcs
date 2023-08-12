---
author: "[@roland-reed](https://github.com/roland-reed)"
issue: "https://github.com/farm-fe/farm/issues/9"
pr: "https://github.com/farm-fe/rfcs/pull/1"
state: open
start_date: "2022-07-18"
---

# Farm 运行时模块管理

## 概述

该 RFC 描述了描述了一个运行时模块管理系统, 作为 Farm 在浏览器, Node 中加载, 注册, 初始化模块

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
