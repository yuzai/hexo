---
title: 前端面试：循环绑定click事件
date: 2016-10-15 21:27:26
tags:
- javascript
- 前端面试题
categories: 学习笔记
---
总结了一道昨天面试明略数据前端的一道js面试题。
<!--more-->
### 题目重述
下面是一段有问题的javascript代码，请输出结果，并找出其中的问题，对源代码进行修改使之达到想要的目的
```html
<html>
<head>
    <meta charset="utf-8">
    <script>
      function init(){
        var pElement = document.getElementsByTagName('p');
        for(var i=0;i<pElement.length;i++)
        {
          pElement[i].onclick = function(){
            alert(i+1);
          }
        }
      }
    </script>
</head>
<body onload = 'init()'>
<p>段落1</p>
<p>段落2</p>
<p>段落3</p>
<p>段落4</p>
</body>
</html>
```
这个代码很简单，最后不论按任何一个按键，都是输出5。因为onclick函数在执行的时候，i始终是4。当时也想到了应该是5，但是之前确实没有解决过这个问题，当时也没有想出来解决方案。太菜了，真的是！
### 分析
笔试结束想了半天，其实很简单。给每个元素加个id或者属性，值就是他们的段落序号。
```js
pElement
[<p>​段落1​</p>​, <p>​段落2​</p>​, <p>​段落3​</p>​, <p>​段落4​</p>​]
typeof(pElement)
"object"
typeof(pElement[0])
"object"
typeof(pElement[1])
"object"
```
今天想到的时候突然发现，我都不知道`pElement=document.getElementsByTagName('p');`获取的元素pElement的类型是什么，索性查了一下，发现全部是Object,里面的元素pElement[i]也是object，也就是说，其实，我是可以给pElement[i]添加任何属性的，我完全可以添加一个属性，pElement[i].index = i;然后再函数中，使用this.index就可以完满的解决问题了。
### 解决方案
```html
<html>
<head>
    <meta charset="utf-8">
	<script>
	  function init(){
	    var pElement = document.getElementsByTagName('p');
	    //console.log(typeof(pElement[0]));//object
	    for(var i=0;i<pElement.length;i++)
	    {
	      pElement[i].index = i;
	      pElement[i].onclick = function(){
	        alert(this.index+1);
	      }
	    }
	  }
	</script>
</head>
<body onload = 'init()'>
<p>段落1</p>
<p>段落2</p>
<p>段落3</p>
<p>段落4</p>
</body>
</html>
```
其实最后就是给每个pElement[i]添加了一个index属性，在onclick函数中访问this.index就可以圆满的解决问题。但是想到这里就不禁想到了给pElement[i]添加属性，会不会对原本的p元素的DOM结果产生影响？
### 最后的分析
给每个p元素都增加了一个属性Index,并不会对原本的html文件有什么影响，用控制台查看源码，p元素还是原来的简简单单的p元素，没有任何改变。在控制台试了一下，结果终于让我找到了DOM中node节点的属性。
```js
for(var i in pElement[0]){console.log(i)}
```
输出了一串一串的东西，有index,有style，有onclick,addListener......容我之前没有系统学习，吓我一跳。这么多原本就有的属性，这下我就懂了，为什么在js中可以通过`pElement[i].style.color = 'blue'`有用，其实就是通过给对象增加一个color属性，又或者本来就存在这个属性（又跑去控制台验证了一下：`for(var i in pElement[0].style){console.log(i)}`,果然又有一堆属性，已经有了color属性）。**但是如果使用`pElement[i].style = {'color':'blue','background':'red'};`并不能改变原来的style，具体的原因有待现在还是不太懂，需要后续进行分析。**

### 12.21补充
**最近突然看到了网上一些闭包解决循环绑定问题的方法，突然感觉之前的方法，理论上可行，但是修改了dom元素的属性，有可能某一天，dom的基本属性中增加了该属性，我的代码就不能用了，不得不说，闭包解决这个问题还是很机智的，很好的利用了函数保留上下文的特点。很好用。**
```js
<html>
<head>
    <meta charset="utf-8">
    <script>
      function init(){
        var pElement = document.getElementsByTagName('p');
        //console.log(typeof(pElement[0]));//object
        for(var i=0;i<pElement.length;i++)
        {
          pElement[i].index = i;
          pElement[i].onclick = (function(i){
            return function(){alert(i+1)}
          })(i);
        }
      }
    </script>
</head>
<body onload = 'init()'>
<p>段落1</p>
<p>段落2</p>
<p>段落3</p>
<p>段落4</p>
</body>
</html>
```
将一个匿名函数赋值给了pElement[i].onclick，因为这个函数还没有执行，所以内存会开辟一块来存储该匿名函数的上下文，在本例中，就是匿名函数外层的那个函数，保留了他的参数，当onclick触发时，输出的就是当时保留的上下文中的i,也就是对应的段落编号。
### 总结
水平还太低了，原生Js都还是没怎么搞清楚，欢迎留言，欢迎指正！
**12.21补充，之前的方案不好，闭包解决才是最好的方法。**
