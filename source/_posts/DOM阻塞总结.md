---
title: DOM阻塞总结
date: 2017-02-21 22:10:26
tags:
- javascript
- DOM
categories: 学习笔记
---

要找实习了，要准备面试了。拿出之前积攒的一些问题点一个一个进行深入的研究，就碰到了这个，[一个微信面试题引发的血案 --[译] 什么阻塞了 DOM？](https://gold.xitu.io/post/587f4afb61ff4b00651b3c18)。浏览器是怎么样加载css,js,图片的？这个问题卡在这里，当我在写html的时候，引入一个css,script标签，不禁会想到，这个东西会不会阻塞我页面的展示，放在这个位置行吗？困扰了我很久，狠下心来一定要把这个问题搞清楚，不然写代码迷迷糊糊的。
<!--more-->

## 浏览器解析html机制
这个问题水比较深，我看了不少文章，解释的都还行，大体都是dom树的构建，css的渲染，但是并没有解决我的疑惑，在我困惑的一些细节上真心不敢恭维。其实，作为一名前端开发者（。。。好吧，自称前端开发者），我关注的点其实很明确：
1. 我引入的css会被异步下载还是同步下载，css下载之后是怎么解析的，解析是并行的还是先建一个数据结构存储样式，最后dom树弄完了才进行全部的套用？
2. 我引入的script是同步下载还是异步下载，script执行的时候会不会暂停Dom树的构建，我的DOMContentLoaded（对应jq中的$().ready）还有onload事件到底是什么时候触发？
3. 我引入的图片呢？这些应该是异步的吧，没有必要让dom解析去等待吧？

我对浏览器做的工作的一些理解用一句话来说，就 **是下载我的资源，解析，生成一个dom树，一个css规则树，然后合成render tree，然后画图，展示到用户面前。DOM Tree 》Render Tree》layout》paint**

下面是一些详细的说明，基本能解决上述的问题：
1. 浏览器下载HTML文件并开始解析DOM
2. 遇到link[rel=stylesheet]的时候，将其加入资源下载队列，继续解析dom（css没有阻塞dom解析）
3. 当遇到script的时候，之后有三种情况：
    1. 如果之前的stylesheet没有下载解析完毕，阻塞dom，并行下载js，等待下载解析(关于解析这块我没有弄懂，解析应该是构成css规则树，但是是否参考dom树来完成还是单独完成，这一块没有弄明白)完毕再执行代码。（此时会阻塞dom的解析,css阻塞了js，进而阻塞了dom解析）
    2. 如果没有未下载完成的css，下载script,下载完毕之后立即执行代码。（这中间的过程包括下载和执行都会阻塞dom解析,js阻塞了dom）
    3. script有defer,async标签，下载好之后立即执行（两者下载均不会阻塞dom的解析，defer会在DOMContentLoaded事件之前按照顺序执行，async下载完毕直接执行，两者的执行都会阻塞dom解析，不过defer执行的时候dom的解析就只剩下执行这个script了），执行完之后继续解析dom。关于defer和async，下一节的截图。
4. 整个dom解析完成，触发DOMContentLoaded。
5. css下载完毕（有可能在4之前，如果在4之前则进行等待dom解析，如果没有下载完毕，即便dom树已经构建完成，chrome是不会展示页面的，因为render tree没有构建，无法paint），渲染，展示页面（这个就是一般访问国外网站很久都是一片空白的原因，css阻塞了渲染）
6. 等待图片等别的类似资源加载完毕，触发onload事件

## 关于defer和async
defer async两者都是使得js的下载不会阻塞dom解析。之所以普通的script的下载会阻塞dom解析，是因为script有可能会改变dom，如果在下载的时候还去解析dom，那下载之后再执行script又改变dom就是一种浪费了，所以浏览器在下载script的时候就阻塞了dom解析。关于defer和async，我觉得下面两幅图就够了。

![deferasync区别.jpg](http://upload-images.jianshu.io/upload_images/3967512-8c57e8c6ad5bca11.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于执行的顺序，这个图说的很明白：

![righttime.jpg](http://upload-images.jianshu.io/upload_images/3967512-df26d394fc05f6dc.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图没有说的是defer会在DOMContentLoaded事件之前按照顺序执行（多个defer定义的script标签），别的都比较明了了，我就不多说了

## 总结
1. css是并行下载，决定了css规则树，render tree的构建，这个如果没有构建好，即便dom解析完毕，也是空白。它并不会直接阻塞dom解析，但是html5规定，如果script标签（非defer,async）前面的css没有下载解析完成，就会等待其完成再执行（可以提前下载）。所以css有可能会间接阻塞dom解析。
2. js阻塞dom解析，下载以及执行的时候也会停止dom解析，当所有的js加载完毕，dom解析完毕，触发DOMContentLoaded事件，也就是jquery的$().ready事件
3. js可以通过设置defer,async来异步加载。
4. 当所有的资源，比如图片,audio等加载完毕会触发window.onload事件

初次对浏览器的加载深入研究，说的不对的地方还请指出！

##　参考文章
1. [兼容所有浏览器的DOM载入事件](http://harttle.com/2016/05/14/binding-document-ready-event.html)
2. [css载入与DOMContentLoaded事件延迟](http://harttle.com/2016/05/15/stylesheet-delay-domcontentloaded.html)
3. [异步脚本载入提高页面性能](http://harttle.com/2016/05/18/async-javascript-loading.html)
4. [css/js对DOM渲染的影响](http://www.tuicool.com/articles/7v2IJ37)
5. [浏览器的渲染原理简介](http://coolshell.cn/articles/9666.html)
6. [javascript的装载与执行](http://coolshell.cn/articles/9749.html)
