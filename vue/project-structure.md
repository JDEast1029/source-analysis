## Vue项目的目录结构
下面的结构是基于`Vue3`的源码生成的，Vue3的多包架构是基于`pnpm`和它的`workspace`实现的。
```js
├── BACKERS.md
├── CHANGELOG.md
├── LICENSE
├── README.md
├── SECURITY.md
├── netlify.toml
├── package.json
├── packages
│   ├── compiler-core
│   ├── compiler-dom
│   ├── compiler-sfc
│   ├── compiler-ssr
│   ├── dts-test
│   ├── global.d.ts
│   ├── reactivity // 响应式
│   ├── reactivity-transform
│   ├── runtime-core
│   ├── runtime-dom
│   ├── runtime-test
│   ├── server-renderer
│   ├── sfc-playground
│   ├── shared
│   ├── size-check
│   ├── template-explorer
│   ├── vue
│   └── vue-compat
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
├── rollup.config.js
├── rollup.dts.config.js
├── scripts
│   ├── aliases.js
│   ├── build.js
│   ├── const-enum.js
│   ├── dev.js
│   ├── pre-dev-sfc.js
│   ├── preinstall.js
│   ├── release.js
│   ├── setupVitest.ts
│   ├── utils.js
│   └── verifyCommit.js
├── tsconfig.build.json
├── tsconfig.json
├── vitest.config.ts
├── vitest.e2e.config.ts
└── vitest.unit.config.ts
```

### 写作的顺序
1. [createApp.md](./createApp.md)
2. [简化版渲染](./render/index.md)
3. [生命周期](./render/lifecycle.md)
4. [调度器](./render/scheduler.md)

### 学习源码的目的
1. `template`是怎么渲染到页面上的
2. `Vue`的编译原理
3. 了解Vue的响应机制
4. 虚拟Dom的设计：diff算法，更新机制等等
5. 生命周期的设计

