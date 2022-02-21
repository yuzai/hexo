---
title: 极简webpack插件：彻底解决vscode点击无法跳转问题
date: 2020-11-09 10:51
categories: 原创
---

在实际开发过程中，经常会用到webpack的别名，来缩短我们在项目中的各种相对路径的引入，方便开发也方便阅读。

但是vscode在不做配置的情况下，本身并不能识别别名，故无法自动跳转，一定程度上降低了使用的便捷性。要解决也很简单，在项目根目录下建立jsconfig.json，配置baseUrl和path即可，但是也还是会有一些问题其他的细节问题，配置起来也比较机械化，在需要配置多个alias时，会显得有些重复化。

<!-- more -->

作为一个技术打工人，当然是不允许重复化、机械化的操作产生，一切能够通过工具解决的事情绝不手动解决。打工人，不在重复中爆发，就在机械化操作中衰落。

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1gzliisiectj20go0bsgmd.jpg)

[alias-jsconfig-webpack-plugin](https://www.npmjs.com/package/alias-jsconfig-webpack-plugin)，通过在webpack中增加插件，根据alias自动生成jsconfig.json（也可以选择tsconfig.json）。用法也非常简单，跟常规的webpack插件一样：

```js
// install
// npm i --save-dev alias-jsconfig-webpack-plugin
const Alias = require('alias-jsconfig-webpack-plugin');
 
// ...plugins
export default {
    // your other config
    // ...
    plugins: [
        // your other plugins
        new Alias({
            language: 'js', // or 'ts'
            jsx: true, // default to true,
            indentation: 4, // default to 4, the indentation of jsconfig.json file
        }),
    ]
}
```

源代码也非常简单，不过80行，核心思路就是读取webpack配置中的alias，判断jsconfig.json文件，如果不存在则按照默认配置新建并将alias注入，存在则对比替换并新增path配置。从而完成jsconfig.json根据alias自动配置。代码在[这里](https://github.com/yuzai/alias-jsconfig-webpack-plugin/blob/main/webpack-plugin/index.js)。

最后，欢迎试用，欢迎issues。

npm包链接：https://www.npmjs.com/package/alias-jsconfig-webpack-plugin
