---
title: imgmaker-解放双手高效生成占位图
date: 2021-04-13 20:52:26
tags:
- javascript
- chrome扩展
- vue
categories: 原创
---

我开发了一款chrome插件--imgmaker（插件地址及仓库地址见文末），可以根据指定的尺寸、指定的关键字，生成想要的图片。可以在占位图的时候，输入想要的关键字，比如妹子、美女，动漫等等，生成一张赏心悦目的图片放上去，解决问题的同时没准还能提高工具效率（毕竟，好看的meizi图片 = 第一生产力）
<!--more-->

## 背景

在前端的日常开发中，经常会碰到这样的情景：
1. 在写一个h5页面，视觉稿此处有个375*200的背景图，但是切图又没给，只能先随便找个图或者直接用背景色替代，往往背景图被拉伸，或者背景色很难看
2. 在写后台的时候，需要提供图片上传的功能，这里一般是需要限制图片上传尺寸的，而开发手头并没有合适尺寸的图片，不得不打开裁剪工具，低效又浪费时间。

在这样的困扰下，我开发了一款chrome插件--imgmaker（插件地址及仓库地址见文末），可以根据指定的尺寸、指定的关键字，生成想要的图片。可以在占位图的时候，输入想要的关键字，比如妹子、美女，动漫等等，生成一张赏心悦目的图片放上去，解决问题的同时没准还能提高工具效率（毕竟，好看的meizi图片 = 第一生产力）

## 最终效果

![image.png](https://pic4.zhimg.com/v2-b0ca900628103f5a353d7ec20defbb2b_b.jpg)

图片不够，可以看看视频效果：[v.douyin.com/eMhKmFj/](https://v.douyin.com/eMhKmFj/)

修改宽高、关键字以及输出类型，都会重新生成一张图片。图片内容是通过爬取百度图片的接口获得，每次都不同，更具新鲜感！同时点击save， 可以将图片保存到本地。

## 技术实现

### chrome插件

chrome插件开发时，需要书写manifest.json，如果想要使用vue、react或者比较新的js语法的话，还需要配置一些自动化的工作流，同时热更新也同正常的热更新有所区别。经过调研后，直接使用现成的模版进行修改开发，还比较好用，拿到手即可着手开发，非常方方便。[模版地址](https://github.com/mubaidr/vue-chrome-extension-boilerplate)

### 图片源获取

如果在插件中内置几张图片的话，会显得非常单调，故通过爬取百度图片的接口，来根据关键字，宽高获取图片，同时，chrome插件中允许配置跨域接口的地址，可以解决掉跨域的问题

### 图片的生成及下载

采用常规的canvas的drawImg以及toBlob进行绘制和下载。其中需要注意的是cover和contain的绘制，还是花了点时间。以画布大小作为canvas的宽高，contain模式时，只需要等比例对图片绘制区域进行缩放，而cover时，需要将短比例的边铺满，长比例的边裁剪，才能达到该效果。

## 结束语

最后，欢迎大家试用，希望能够带来工作效率的提升，奉上[chrome商店地址](https://chrome.google.com/webstore/detail/imgmaker/gcnibpcodipjdbobdnfihkljfdojhgdb?hl=zh-CN)(需梯子)及[github地址](https://github.com/yuzai/imgmaker)（可以自行打包安装）

## 碎碎念

工作之余开发，欢迎issues，欢迎聊天[提需求](http://blog.maxiaobo.com.cn/rs/wechat.jpg)