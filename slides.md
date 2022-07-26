---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
highlighter: shiki
titleTemplate: '%s'
---

# monorepo 2.0

---

# 总览

- ko 升级至 v6.x 版本
- 集成 ko-lint-config
- CI 改善

---

# ko v6.x

- 放弃了之前读取 webpack 相关配置并在内部合并之后使用的做法，改为暴露常用的配置并在内部进行整理合并以及后续的调用
- 集成 eslint,stylelint,prettier 相关 cli,只负责读取相关配置并进行调用,且支持多线程调用，cli 增加`--concurrency`配置即可开启
- 支持 css modules（目前仅对 sass 文件启用，且需要以`.module.sass`或,`.module.scss`为文件后缀）

---

# `ko.config.js`配置变更

部分配置:

```ts
export type IOptions = {
  externals?: Record<string, string>; //excluding dependencies from the output bundles
  plugins?: any[]; // ko internal plugins, you can define your own plugin of ko.
  htmlTemplate?: string; //output html file template
  // dev, or serve configs
  serve: {
    proxy?: Record<string, any>; // proxy of dev server
    host: string; // host of dev server
    port: number; // port of dev server
    staticPath?: string; // static path that will be watch of dev server
  };
  // experimental features
  experiment?: {
    speedUp?: boolean; // enable speed up configs of dev & build actions
    minimizer?: boolean; // enable minimizer via esbuild in build action
    enableCssModule?: boolean; //enable css module
  };
  lints?: Record<IKeys, Omit<IOpts, 'write'>>; // lint configs
};
```

[Configuration](https://dtstack.github.io/ko/docs/current/configuration)

---

# 选型

1. 前置 VS 懒加载
2. 底层语言是前端基建的未来
3. dev & build 输出产物应共用一套逻辑

---

# 性能对比(ko dev)

硬件信息:

- CPU: Apple M1
- Memory: 8 GB LPDDR4

  | version | withCache | benchmark(ms) |
  | ------- | --------- | ------------- |
  | 5.x     | false     | 27078         |
  | 6.x     | false     | 16642         |
  | 5.x     | true      | 17965         |
  | 6.x     | true      | 4036          |

---

# Formatter & Linters

- Formatter 只关注代码风格，例如 `tabWidth` 等，典型的代表是 **prettier**
- Linter 则将关注点放在可能的 bugs 并提供检测方法，Script 相关的典型代表为 **eslint**, Style 相关的典型代表为 **stylelint**

ko 提供相应的命令进行调用:

```bash
# Run lint via prettier on files:
pnpm exec ko prettier file1.js file2.ts
# Run lint via prettier on multiple files via glob syntax and try to fix problems
pnpm exec ko prettier "src/**" --write
# Run lint via eslint on files:
pnpm exec ko eslint file1.js file2.ts
# Run lint via eslint on multiple files via glob syntax on concurrency mode
pnpm exec ko eslint "src/**" --concurrency
# Run lint via stylelint on files:
pnpm exec ko stylelint file1.css file2.less
# Run lint via stylelint on multiple files via glob syntax with custom config
pnpm exec ko stylelint "src/**" --configPath="/path/to/custom/config"
```

---

# 性能对比(ko eslint)

<br />

普通模式下的 log 为:

```txt
exec cmd: pnpm exec ko eslint '**/*.{ts,tsx,js,jsx}' --write
exec eslint with 704 files cost 31.71s
```

多线程模式下的 log 为:

```txt
exec cmd: pnpm exec ko eslint '**/*.{ts,tsx,js,jsx}' --write --concurrency
Using Multithreading...
exec eslint with 704 files cost 23.60s
```

---

# [ko-lint-config](https://github.com/DTStack/ko/tree/master/packages/ko-lint-config) @liuyi

- 定义了 prettier,eslint,stylelint 相关规则
- eslint 规则 遵循[Code Style Guide](https://github.com/DTStack/Code-Style-Guide)
- 可自行升级，不受 ko 相关版本迭代升级影响

# 全局配置 VS 自定义配置

1. 很多人希望更简洁的配置，但是少部分人则需要更多的细节配置
2. 全局配置应当有较强的约束性和简洁性
3. 自定义配置需要更高的自由度

---

# CI

包含以下 CI:

- git pre-commit hooks(lint-staged)
  - 增加了 eslint 和 stylelint 的 fix
  - 对不同文件进行了分别处理
- npm scripts

  - 通过 turbo 定义相关 pipeline
  - 增加`pnpm lint`用于检测，增加`pnpm lint:fix`用于自动修复

- gitlab ci
  - 根据 merge 源分支名称（CI_MERGE_REQUEST_SOURCE_BRANCH_NAME）来匹配范围 @qiming
  - 增加了 eslint, prettier, stylelint, tsc, test 五项内容的检查

统一 ci 调用到`scripts/ci`目录,相关逻辑可在此处修改

---
