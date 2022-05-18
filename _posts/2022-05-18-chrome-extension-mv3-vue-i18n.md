---
layout: post
title: Chrome Extension Manifest V3 Vue-i18n 集成解决方案
tags: [Chrome Extension, Manifest V3, Vue-i18n, EvalError]
author: Jackin
---

## 背景

最近在做一款 Chrome 扩展插件，由于 Google 提示了 Manifest V2 支持时间轴，将在2023年（企业用户是2023年6月）不再支持V2上架和更新，因此刚开始做的时候就确定了使用 **Manifest V3** 开发。

## 问题

采用 `Vue-CLi` +` TypeScript` + `Vue 3` 来搭建了 Chrome Extension 开发工程。**Manifest V3** 的 CSP （Content-Security-Policy）不再允许使用 `unsafe-eaval` 配置。但是在集成 `Vue-i18n` 组件实现插件多语言时，编译后加载插件报错：

```raw
Uncaught EvalError: Refused to evaluate a string as JavaScript because 'unsafe-eval' is not an allowed source of script in the following Content Security Policy directive: "script-src 'self'
```



经过一番搜索和查找，发现了问题所在 [参考链接](https://github.com/intlify/vue-i18n-next/issues/543#issuecomment-864348217)

> As for the CSP problem, if you are using bundler, you can solve it by using the runtime-only module, as described in the docs.
>
> [vue-i18n bundle doc](https://vue-i18n.intlify.dev/ja/guide/advanced/optimization.html#improve-performance-and-reduce-bundle-size-with-runtime-build-only)
>
> **Currently, Vue applications are generally developed using bundler.**

Github上也有其他开发者遇见[类似问题](https://github.com/intlify/vue-i18n-next/issues/381)。

## 原因

`Vue-i18n` 为 Bundler 提供了以下两个内置的 ES 模块：

- message compiler + runtime: **`vue-i18n.esm-bundler.js`**

- runtime only: **`vue-i18n.runtime.esm-bundler.js`**

Bundler 默认采用 **``vue-i18n.esm-bundler.js``** 进行 messages 的编译和打包，`EvalError` 就是在这个 `esm-bundler` 内部抛出的。因此我们要切换到 **`vue-i18n.runtime.esm-bundler.js`** 来实现 messages 的编译打包，另外根据官方文档，采用 `runtime.esm-bundler` 还有另外一个好处，可以减少打包后的 size 。

> If you want to reduce the bundle size further, you can configure the bundler to use `vue-i18n.runtime.esm-bundler.js`, which is runtime only.
>
> NOTE
>
> In the [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) environment, `vue-i18n.esm-bundler.js` does not work with compiler due to policy, so you need to use `vue-i18n.runtime.esm-bundler.js` in that environment as well.

## 解决前奏

很简单，根据官方文档，基于 `webpack` 打包的工程，在 `vue.config.js` 打包配置文件里面，将 `vue-i18n` 替换成 `vue-i18n.runtime.esm-bundler.js` 即可

- `webpack` 独立配置

``` js
module.exports = {
  // ...
  resolve: {
    alias: {
      'vue-i18n': 'vue-i18n/dist/vue-i18n.runtime.esm-bundler.js'
    }
  },
  // ...
}
```

- 基于 `Vue-CLi` 的 `webpack` 配置

```js
webpackConfig.resolve.alias.set(
  "vue-i18n",
  "vue-i18n/dist/vue-i18n.runtime.esm-bundler.js"
);
```

配置好然后我们再来编译一下 Chrome 插件，运行后的确是没有再抛出 **`Uncaught EvalError`** 异常了，有进展。

## 解决后序

然而，问题并没有得到彻底解决。我们定义在 `messages.json` 的 key 一直都被 `esm-bunlder` 提示找不到，因此也无法正确加载当前 locale 对应的语言配置。

```raw
[intlify] The message format compilation is not supported in this build. Because message compiler isn't included. You need to pre-compilation all message format. So translate function return 'key'.
```

查看 `Vue-i18n` 的文档后发现，其实官方已经给出了解决方案

> NOTE
>
> If you use `vue-i18n.runtime.esm-bundler.js`, you will need to precompile all locale messages, and you can do that with `.json` (`.json5`) or `.yaml`, i18n custom blocks to manage i18n resources. Therefore, you can be going to pre-compile all locale messages with bundler and the following loader / plugin.
>
> - [`@intlify/vue-i18n-loader`](https://github.com/intlify/bundle-tools/tree/main/packages/vue-i18n-loader)
> - [`@intlify/rollup-plugin-vue-i18n`](https://github.com/intlify/bundle-tools/tree/main/packages/rollup-plugin-vue-i18n)
> - [`@intlify/vite-plugin-vue-i18n`](https://github.com/intlify/bundle-tools/tree/main/packages/vite-plugin-vue-i18n)

那么根据我们搭建的工程，需要引入 `vue-i18n-loader` 来给 `messages.json` 进行预编译，修改 `vue.config.js`  的 `webpack` 配置：

```js
webpackConfig.module
      .rule("i18n")
      .test(/\.(json5?|ya?ml)$/)
      .include.add(path.resolve(__dirname, "./src/locales"))
      .end()
      .type("javascript/auto")
      .use("i18n")
      .loader("@intlify/vue-i18n-loader")
      .end();
```

然后再来编译一下 Chrome 插件，重新加载运行后，终于看到了对应的 locale 配置，所有问题迎刃而解。