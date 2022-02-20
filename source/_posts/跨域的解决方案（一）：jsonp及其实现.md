---
title: 跨域的解决方案（一）：jsonp及其实现
date: 2017-01-31 13:32:26
tags:
- javascript
categories: 学习笔记
---
最近遇到了跨域问题，有很多种解决方案，本文主要对其中的jsonp进行演示，本文使用node编写服务器端代码进行测试。
<!--more-->

### jsonp的来历
ajax中的资源访问受到浏览器安全的['同源'限制](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html),存在跨域问题。
但是script标签的引入并不具有'同源'限制,不存在跨域问题，利用这个特性进行数据的获取，就是jsonp。
JSONP(JSON with Padding)是一个非官方的协议，它允许在服务器端集成Script tags返回至客户端，通过javascript callback的形式实现跨域访问（这仅仅是JSONP简单的实现形式）。
简单的一句话：jsonp是一个协议，由于同源策略的限制，XmlHttpRequest只允许请求当前源（域名、协议、端口）的资源，为了实现跨域请求，可以通过script标签实现跨域请求，然后在服务端输出JSON数据并执行回调函数，从而解决了跨域的数据请求。

### jsonp的实现
从上文可知，jsonp的实现需要客户端代码和服务器端代码共同努力。整体的思路如下图：
![Paste_Image.png](http://upload-images.jianshu.io/upload_images/3967512-4a0c232f37c44707.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

script标签不受跨域资源请求的影响，可以通过增加callback参数将回调函数传递，与服务器端配合从而在资源加载之后执行该回调函数,客户端关键部分写法如下：

```html
<!DOCTYPE html>
<body>
    <h1>测试jsonp</h1>
    <p>用node搭建能够解析jsonp请求的服务器，前端通过script发送jsonp请求，通过回调处理服务器返回的数据</p>
    <script>
        function test(data){
            alert(data.name);
        };
    </script>
    <!-- 将test作为callback参数传递 -->
    <script src="http://localhost:3000/jsonp?callback=test"></script>
</body>
```

在服务器端，需要对script的src进行url解析，将data作为参数放入回调函数中，最后通过res.end(callback(data))中将要执行的函数放入客户端的script中，在客户端对该函数进行执行。
服务器端的关键代码如下：

```js
//解析url
var urlPath = url.parse(req.url).pathname;
console.log(urlPath);
//获取从客户端传递的参数
var qs = querystring.parse(req.url.split('?')[1]);
//约定的url的名称为jsonp
if(urlPath === '/jsonp' && qs.callback){
    res.writeHead(200,{'Content-Type':'application/json;charset=utf-8'});
    var data = {
        "name": "Monkey"
    };
    data = JSON.stringify(data);
    var callback = qs.callback+'('+data+');';
    //在end中返回callback(data)，写入script中，进而客户端进行该函数的执行
    res.end(callback);
}
```

仓库在[这里](https://github.com/yuzai/-/tree/master/js_cors/jsonp)(仓库中的代码相对复杂一点，不过更加便于演示，具体看仓库中的说明)这里要说明的一点就是，url中"?callback=test"的形式其实是不必要的，只需要客户端和服务器端协商一致即可。（这里我在写的时候，其实有一点疑惑：就是为什么把callback(data)传递回去就会在客户端执行？其实是因为客户端是通过script请求的，获取的内容会放在<script></script>中，而res.end(data)中的data就会放到script中，浏览器会自动调用其内部代码执行，从而完成回调函数的执行）。

### 总结
jsonp需要服务器端和客户端配合使用，以script标签无'同源'限制的便利来实现跨域资源的共享。
优点：解决了跨域资源共享
缺点：代码麻烦，需要服务器端和客户端约定好，同时客户端传递数据只能通过url的形式，有长度，安全等限制。
jsonp原理其实很简单，学习难度主要在于与服务器端的配合，这篇文章是我学习了node之后的一些想法，如有问题，还请各位指点！

### 参考文章
1. [借助node实战JSONP跨域](http://www.cnblogs.com/giggle/p/5496596.html)
2. [前端跨域整理
](https://gold.xitu.io/post/5815f4abbf22ec006893b431)
3. [浏览器同源政策及其规避方法](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)
