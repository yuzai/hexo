---
title: ajax本地通信
date: 2016-11-22 17:31:57
tags:
- js
- ajax
- 随笔
categories: 学习笔记
---
最近在学习es6的promise，已经看了第2遍了，这次看过感觉有点懂了，去阮一峰的es6教程上又看了一下，有两个例子，一个是异步加载图片，一个是ajax。对异步加载图片还是不太懂，但是ajax我总会吧，就尝试用promise+ajax试下笔，结果。。。。
发个文章，记录今天苦逼的一天。
<!--more-->
### ajax通信代码
ajax用原生的Js来实现的话，其实核心是利用XMLHttpRequest对象，可以进行get和post通信，这不是今天的问题，问题是用谷歌浏览器的时候报错。源代码如下：回头一定研究一下如何让静态网页代码高亮。
```html
<!DOCTYPE html>
<html>

<head>
    <meta charset='utf-8'>
    <script>
        function loadXml() {
            const xmlhttp= new XMLHttpRequest();
            xmlhttp.onreadystatechange = function() {
                console.log(xmlhttp.readyState);
                console.log(xmlhttp.status);
                if (xmlhttp.readyState == 4 && xmlhttp.status == 200) {
                    document.getElementById("myDiv").innerHTML = xmlhttp.responseText;
                }
            }
            xmlhttp.open("GET", "flex.html", true);
            xmlhttp.send();
        }
    </script>
</head>

<body>
  <div id='myDiv'><h2>test on ajax</h2></div>
  <button type='button' onclick='loadXml()'>修改内容</button>
</body>

</html>
```
上述代码定义了一个XMLHttpRequest对象，通过open和send方法实现了对本地flex.html的通信，获取其中的信息并写入到DOM的myDiv中去，当用chrome的时候，时始终提示Load出错，没有权限，但是用firefox就没有问题，所以去网上搜索了一下，其实是chrome为了安全，禁止了网页与系统文件的ajax，所以只需要在chrome的快捷方式上加入--allow-file-access-from-files选项就可以了。如下图：

![图一](http://blog.xiaoboma.com/设置.jpg)

要吐槽的地方其实是，我改了很多遍，浏览器重启了很多遍，都没有成功，最后发现竟然是因为我在另一个桌面上一直开着chrome，（win10多桌面），导致了一直失败，最后，修改了这里，终于没报错了，然而还是没有效果，所以把xmlhttp.status的值输出了一下，发现输出是0，去网上搜了一下，这个是因为本地通信，所以是0.但是在firefox中依旧是200。

### promise+ajax
吐槽完，还是得把该做的工作做完，用promise+ajax来实现更高端的效果。promise其实是回调函数的更优雅的写法，它的写法更加易懂并且简单。
```js
function loadXml() {
    const p = new Promise((resolve, reject) => {
        const xmlhttp = new XMLHttpRequest();
        xmlhttp.onreadystatechange = handler;
        xmlhttp.open("GET", "flex.html", true);
        xmlhttp.send();

        function handler() {
            if (this.readyState !== 4) {
                return;
            }
            if (this.status === 200) {
                resolve(this.response);
            } else {
                reject(new Error(this.status));
            }
        }
    });
    p.
    then(data => document.getElementById("myDiv").innerHTML = data).
    catch(data => document.getElementById("myDiv").innerHTML = data);
}
```
上面代码通过新建一个Promise,内部使用xmlhttp.onreadystatechange来触发不同的resolver,通过绑定then和catch来完成不同的功能。

### 总结
本地ajax通信，只需要在chrome进行简单的快捷方式修改就可以了，Promise类似回调，但是写起来更加方便，尤其是当级联的时候（本程序没有体现）。
