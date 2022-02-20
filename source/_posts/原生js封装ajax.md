---
title: 原生javascript封装ajax
date: 2017-01-24 22:12:26
tags:
- javascript
categories: 原创
---
最近重新看《javascript高级程序设计》，突然看到了ajax，想起来之前学习的各种坑，又想着结合最近学习的模块化编程、面向对象式编程，所以用原生的js采用面向对象的设计思路对ajax进行了一个封装，同时，想起之前学习ajax的最大困难：没有服务器端代码，不好测试，所以这次用原生node写了一个简单的服务器，用于处理ajax的测试。
<!--more-->

综上，本文主要提供了以下几个点：
1. 采用面向对象的方法用原生js封装了一个ajax类，更便捷的实现ajax通信
2. 提供了一个Node写的服务器端的代码，可以用来测试ajax
3. 提供了一个demo，服务器端以及浏览器端的代码来测试ajax类

仓库在[这里](https://github.com/yuzai/ajax-),服务器，客户端的代码都在里面，也有一些测试的说明。

### ajax实现原理
ajax是一项伟大的技术，其很好的解决了传统浏览器一言不合就重新发送整个页面，速度慢，用户体验差的问题。它是一个获取资源的手段，可以在不进行整体刷新的情况下进行局部dom修改，速度快，用户体验好。实现的原理主要是基于一个类，XMLHttpRequest(IE7以下不支持，提供了另外的类进行实现，不在本文讨论范围中，都2017年了。。)。
大概的原理分成下面几步：
1. 新建XMLHttpRequest对象，var xhr = new XMLHttpRequest().
2. 通信需要有一个回调函数，就是接收到对方的回信之后需要有一个处理的函数，这个函数可以通过xhr.onreadystatechange = callback；来进行回调。对方的回信存储在xhr.responseText。当通信状态xhr.readyState === 4的时候，xhr.responseText就是完整的接收到的数据。
3. 收到回信，进行相应的dom操作，实现页面的局部刷新。

可以看到，其实一个ajax的请求，是很简单简洁的，只要开发者提供，method（get还是post）,data(要传递的数据，ajax本身是用来通信的嘛)，url(通信的对象)，callback(收到回信后的处理)就可以完成一次通信，别的地方是重复性的劳动，不必每次都繁琐的写一遍。所以我建了一个类，提供send方法，实现信息的发送（ajax重复性的工作），上述四个数据存储在每个实例中，只要实例调用send方法，就会进行一次ajax通信。大概的代码思路是这样：
```js
function Ajax(obj){
  //根据obj对method,data,url等进行初始化
};
Ajax.prototype.send = function(){
  var xhr = new XMLHttpRequest();//新建ajax请求，不兼容IE7以下
  xhr.onreadystatechange = function(){//注册回调函数
    if(xhr.readyState === 4)
      callback(xhr.responseText);
  }
  if(method === 'get'){//如果是get方法，需要把data中的数据转化作为url传递给服务器
    xhr.open(method,url,true);
    xhr.send(null);
  }else if(method === 'post'){//如果是post，需要在头中添加content-type说明
      xhr.open(method,url,true);
      xhr.setRequestHeader('Content-Type','application/x-www-form-urlencoded');
      xhr.send(JSON.stringify(data));//发送的数据需要转化成JSON格式
  }else {
    console.log('不识别的方法:'+method);
    return fasle;
  }
}
```

源码点击[这里](https://github.com/yuzai/ajax-/blob/master/ajax2.js)
有了这个类，要实现一条ajax请求，只需执行下面代码即可:

```js
var ajax = new Ajax({
  method:'get',//设置ajax方法
  url:'http://localhost:3000',//设置通讯地址
  callback:function(res){//设置回调函数
     alert(res)
  },
  data: data//需要传递的数据
})
ajax.send();
```

详细的关于这个类的使用方法，可以参考[这里](https://github.com/yuzai/ajax-)
有了ajax，要测试，怎么办？当然可以找网上一些提供接口的网站进行ajax通信，比如聚合数据，有一些Key需要自己申请，相比之下，自己写一个服务器端的代码来处理请求，反而显得更简单一点。

### 服务器端处理ajax请求
node.js出来之后，javascript又能打前端，又能干后端，也把我拉进了坑里。在这里，主要是为了进行测试，所以越简单越好（我服务器端的代码水平还不够）。这里我主要参考了菜鸟教程中关于处理客户端请求的代码：原理及介绍点[这里](http://www.runoob.com/nodejs/node-js-get-post.html)
```js
var http = require('http');
var url = require('url');
var util = require('util');

http.createServer(function(req,res){
  console.log(req.method);
  if(req.method==='GET'){//处理get请求
      res.writeHead(200,{'content-Type':'text/plain',"Access-Control-Allow-Origin":"*"});
      res.end(util.inspect(url.parse(req.url, true)));//处理get请求，并将结果传递给客户端
  }
  else{//处理post请求
      // 定义了一个post变量，用于暂存请求体的信息
        var post = '';

        // 通过req的data事件监听函数，每当接受到请求体的数据，就累加到post变量中
        req.on('data', function(chunk){
            post += chunk;
        });

        // 在end事件触发后然后向客户端返回。
        req.on('end', function(){
            post = JSON.parse(post);
            console.log(post);
            res.writeHead(200, {'Content-Type': 'text/html',"Access-Control-Allow-Origin":"*"});
            res.end(util.inspect(post));
        });
    }
}).listen(3000);
console.log('server is listening on "3000"');
```
get请求的处理比较简单，因为get方法将信息都加在url中，服务器端对url进行解析就可以得到该请求的内容，我这里的处理是将其原封不动的返回给客户端。
post的请求相对麻烦，需要注册两个时间，on,end。一个是处理数据块，因为Post请求一般数据比较多，分多次传输，我们要的其实只是end之后的数据，在这里我也是直接将其信息几乎原封不动的返回。

### 联合测试
可以直接clone[这里](https://github.com/yuzai/ajax-)
里面的example下面有两个文件，一个是客户端的代码，一个是服务端的代码，服务端代码需要node server.js即可实现监听，浏览器端直接打开index.html即可。

### 总结
第一次发这种文章，也是发现写一篇文章不容易，在写的过程中也是有不少收获，我也深知，我还小，知识还有很多的漏洞，如果有误，严厉指出即可，谢谢！
