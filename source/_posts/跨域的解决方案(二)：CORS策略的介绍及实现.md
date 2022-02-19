---
title: 跨域的解决方案(二)：CORS策略的介绍及实现
date: 2017-02-01 18:32:26
tags:
- javascript
- 跨域
categories: 总结
---

最近遇到了跨域问题，有很多种解决方案，本文主要对其中的CORS进行演示，本文使用node编写服务器端代码进行验证。在解决该问题的时候，顺带对cookie的操作进行了一定的说明。
<!--more-->

### CORS简介
CORS是一个W3C标准，全称是"跨域资源共享"（Cross-origin resource sharing）。
它允许浏览器向跨源服务器，发出ajax请求，从而克服了AJAX只能[同源](http://www.jianshu.com/p/3704f51d1a6b)使用的限制。
CORS依赖于服务器端的设定，只要在服务器端进行了设置，就可以实现相应的资源访问。

### CORS简单服务器端实现
之前写过一篇文章[原生javascript封装ajax](http://www.jianshu.com/p/4e1d2ee63da7)，在该文章中，用面向对象的方法简单封装了一个ajax通信类，同时建立了一个本地的服务器来进行验证，（本文的代码就基于上述代码,仓库在[这里](https://github.com/yuzai/ajax-),服务器，客户端的代码都在里面，也有一些测试的说明。），事实上，在我进行验证的时候就遇到了跨域的问题，因为本地运行的页面和本地运行的服务器之间，不符合同源的策略，是不能进行ajax通信的，我当时采用的解决方案就是通过CORS方法在服务器端添加了Access-Control-Allow-字段实现了跨域通信。下面总结一下我在实际中的解决过程。

#### 问题的出现
本地的页面，直接打开html文件，而用node建立了一个localhost的服务器，在html中使用直接使用ajax向服务器端发送请求，服务器端没有做CORS处理，此时浏览器会如下报错：

![problem.png](http://upload-images.jianshu.io/upload_images/3967512-24b8c9ce36f26832.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

大致的意思就是服务器端没有发现Access-Control-Allow-Origin字段，而本地的html的origin默认是null(如果是在服务器上运行，举个例子，运行在`http://blog.xiaoboma.com/ajax-/`，则origin是`http://blog.xiaoboma.com`,只包含协议和域名，不包含后面的url路由等路径)，因为不匹配，所以不允许进行通信，ajax请求失败。值得注意的是，此时的通信失败不能通过xhr.status判断，需要通过xhr.onerror事件进行相应的处理。

#### 解决方案
知道了问题，只需要在服务器端的header中增加Access-Control-Allow-Origin即可解决该问题，关键代码如下：

```js
res.writeHead(200,{'content-Type':'text/plain',"Access-Control-Allow-Origin":"*"});
```

此时，ajax请求就可以正常发送，观察调试工具（我使用的chrome）中的network一项，就可以看到发送的数据以及收到的返回信息。

![ajax.png](http://upload-images.jianshu.io/upload_images/3967512-b8edfc0342019c99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这张图片主中的responseHeader和origin只要匹配，就可以进行ajax请求。\*表示任何origin都可以进行访问，事实上，在这里如果在服务器端将其设置成Access-Control-Allow-Origin":null也是可以进行ajax通信的。

#### CORS小结
可以看出来，CORS的实现主要是服务器端的设定，origin也是浏览器自动添加，人为是不能修改的，所以只要在服务器端设置了Access-Control-Allow-Origin字段，就可以实现相应网页的跨域资源访问，客户端的ajax通信代码不需要做任何的改动。

### cookie的设定与发送
在设置Access-Control-Allow-Origin的时候不可避免的碰到了'Access-Control-Allow-Credentials'字段，按捺不住好奇，就搜索了这个字段，该字段只能设置为true,当设置了true之后，允许客户端通过ajax发送cookie，否则不接受客户端发送的cookie,客户端需要设置`xhr.withCredentials = true;`
在服务端的res.writeHead添加如下字段：

```js
var time = new Date();
      var time = time.getTime() + 24*60*60*1000;//按ms计算，一天
      var time2 = new Date(time);
      var timeObj = time2.toGMTString();
      res.writeHead(200,{'content-Type':'text/plain',"Access-Control-Allow-Origin":null,
      	'Set-Cookie':'myCookie="type=ninja",language="javascript";Expires = '+timeObj,'Access-Control-Allow-Credentials':true});
      res.end(util.inspect(url.parse(req.url, true)));//处理get请求，并将结果传递给客户端
```

通过Access-Control-Allow-Credentials：true允许客户端发送cookie至服务器端，通过Set-Cookie设置cookie内容和过期时间（如果不设置过期时间，默认页面关闭）。这种状态下，Access-Control-Allow-Origin不能为'\*'，否则会报错，毕竟，不能让每个网站都直接向服务器发送cookie,必须是经过指定的域名才能，这里因为是本地，直接设置为null。此时，在index.html中发送post请求，可以看到cookie信息。

![cookie.png](http://upload-images.jianshu.io/upload_images/3967512-cf03d57ba0e3cc6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### cookie设置小结
要想通过客户端发送cookie给服务器端，必须有服务器端支持。
1. Access-Control-Allow-Credentials必须为true且只能为true
2. Access-Control-Allow-Origin必须指定且不能为'\*'
3. 客户端需要指定xhr.withCredentials = true;

*需要注意的是，如果要发送Cookie，Access-Control-Allow-Origin就不能设为星号，必须指定明确的、与请求网页一致的域名。同时，Cookie依然遵循同源政策，只有用服务器域名设置的Cookie才会上传，其他域名的Cookie并不会上传，且（跨源）原网页代码中的document.cookie也无法读取服务器域名下的Cookie。*

### 参考文章
1. [跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)
2. [node.js 操作Cookies](http://blog.csdn.net/leironghao/article/details/8555243)
