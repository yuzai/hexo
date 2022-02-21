---
title: chrome 94预计cookie策略再加码，怎么放心大胆的升级？ 
date: 2021-08-16 15:25
categories: 原创
---

chrome 94之后，将禁用命令行参数对SameSiteByDefaultCookies提供，届时，本地localhost下，直接访问测试环境接口将成为问题。本文先介绍这一策略的历史及曾经的解决方案，并会针对此问题，提出一个chrome 94之后的解决方案作为参考。让大家尽量少走弯路，放心大胆的升级chrome。
<!--more-->

## 背景

（想直接拿解决方案的直接翻到解决方案一节即可）

相信前一段时间（2021.5 - 7月份，适逢chrome91发布），有不少前端开发同学和我一样。一觉起来，昨天还能正常调试的页面，今天一直在反复报未登录，而自己昨天晚上下班的时候还好好好的，怎么睡一觉就不行了呢？

调bug真理有二：

1. 计算机是不会出错的，如果结果不对，一定是代码错了
2. 排除一切不可能，剩下的不管多么难以置信，一定就是真相

相信聪明的小伙伴肯定很快就能定位到问题，立马就祭出safair，可以正常使用。那么毫无疑问，chrome更新带来的(相信chrome作为前端开发的必备，开启自动更新是很多同学的选择)。而无法登陆，无非就是cookie不会携带呗，老问题了。

顺带一提：

在我们当前团队目前的项目中，不是所有项目都会出现这个cookie问题。仔细看代码，有些项目的接口请求，始终是发送到本地域名的，而请求测试环境，是通过webpackDevServer中配置proxy实现接口转发，由于在浏览器看来始终是发送到localhost的，接口域名是localhost，本地域名也是localhost，所以不存在cookie不携带导致无法登录的问题。

而一些h5项目，是直接发送到测试环境域名的，由于localhost和接口环境域名不同源，从而出现了cookie无法携带，从而无法登录的问题。



## 同源策略的历史及对应的解决方案

当页面内发起请求时，会默认携带该域名下的cookie，而cookie同源策略，是指，除非当前域名和请求域名，是同源，才会默认携带cookie，这就导致，localhost直接请求测试环境的借口，比如api.baidu.com，此时不会携带cookie，从而导致接口301，登录失效的问题。

这一策略从chrome 80开始存在，而chrome 91升级，chrome 94(预计2021.9月份发布)预计再升级。

在chrome 91以前，可以通过在chrome 浏览器中打开chrome://flags/，设置SameSiteByDefaultCookie为disable即可。

而chrome 91之后，删除了该配置，需要手动配置启动参数来关闭该限制，通过增加启动参数--args --disable-features=SameSiteByDefaultCookies。即可实现cookie的跨域传递。自动添加启动参数可以参考[文章](https://www.jianshu.com/p/a8cc2c04ee7c)。

而chrome 94预计删除启动参数的支持，那么届时该怎么解决呢？

## 几个解决方案

chrome 94之后，当chrome禁用了跨域cookie的命令行操作之后，笔者实践过的有以下几个解决方案：

### 借助webpack-dev-server实现接口转发

在发起接口请求时，请求的链接始终是localhost:yourport/your/api/name，在webpack or easy中配置proxy把api接口转发到需要的域名即可。

当然，这样也会有一些缺点：

1. 每次切换proxy需要修改webpack配置，需要重启(有些脚手架会自动重启，但是也会自动打开一个新页面而非热更新，不是那么丝滑，当然也有些脚手架会热更新)。
2. proxy配置转发时，往往是按照正则去匹配路径，而有些场景，我们有些接口需要到测试域名，而有些接口需要到mock域名，就需要做一些单独的处理。在合代码、开发过程中往往多有不便。

### 借助charles实现域名转发（推荐）

cookie不携带的根源在于本地localhost和域名和测试环境的域名不同，导致跨域从而cookie不携带，那么只需要通过代理工具将与测试环境同域的域名代理到本地，即可解决该问题。

由于笔者自己比较熟悉charles，此处以charles为例。

step1: 安装charles，然后安装证书（不安装证书无法进行https信息的查看），google or baidu即可，此处不做赘述。

step2: 点击tools -> Map Remote -> add, 进行如下配置（假设端口是3000），并保存，并勾选enable map remote即可。

![image-20210815234502383](https://tva1.sinaimg.cn/large/008i3skNly1gthwlwdfkuj60i50hdt9o02.jpg)

上述配置，便将x x x.163.com下的请求，转发到了127.0.0.1:3000端口下，浏览器直接打开xxx.163.com即可返回本地3000端口下的页面。此处，163.com仅仅是笔者自己的域名，实际配置时，保证想要访问的接口的二级域名相同即可。由于cookie的跨域判断只到二级域名。另外还要注意http 和https，两者不同，也是会被判定为跨域，不会携带cookie。

## WebIDE or CloudIDE

 一般来讲，公司自己搭建的WEBIDE，都基本与想要访问的接口同域，那么在开发时，由于本身就是同域，自然不存在跨域后cookie无法携带导致登录不上的问题。

当然，webide的建设也需要一定的积累。感兴趣的小伙伴私信加入我们团队（云音乐直播团队），一起建设webide喽。



## 其他方案

上述几个方案都是可行的，笔者全部进行过实操验证。接下来还有几个方案（但是未经笔者验证）：

ngnix、whistle、fiddler、mitmproxy等转发工具，可以选择转发域名，转发接口，原理上均可行

charles转发接口，上述是转发域名，使得cookie同域，理论上转发接口也没毛病，不过我们的接口环境变化多端，配置起来比较麻烦，不如域名来的实在
服务端做修改，chrome允许跨域cookie携带，但是cookie需要满足两个条件，cookie的samesite属性设置为none, secure设置为true，网站协议为https。所以理论上本地需要改成 https://localhost 同时服务端配合，完成跨域cookie的携带（个人觉得服务端不会做这个修改，跨域cookie携带，让csrf攻击来的更轻松）

## 最后

祝大家升级愉快喽！笔者本人属于网易云音乐直播团队，氛围好，技术紧跟潮流，也欢迎私信来撩哈。

