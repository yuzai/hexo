---
title: arrow的疑问
date: 2016-11-23 16:25:53
tags:
- 疑问
- es6
categories: 总结
---
学会使用es6的箭头函数之后，在几乎所有需要用到匿名函数的地方都改为了箭头函数，this绑定外部函数的this无往不利，但是最近遇到了一个问题。
<!--more-->

对象的方法定义时，箭头函数的this值应该是什么？

```js
//code1
var s={
}
s.fun=()=>{console.log(this)} //严格模式下undefined，浏览器控制台输出window
//s.fun=function(){console.log(this)}//[object:s]
s.fun();
//code2
var s={
  fun:()=>console.log(this)//严格模式下undefined，浏览器控制台输出window
  //fun:function(){console.log(this)}//[object:s]
}
s.fun();
```
上述代码表示，在定义对象的方法时，使用箭头函数，this没有绑定(严格模式下undefined，非严格模式时是window).
翻看了箭头函数的说明，对this的说明是，this会继承外部的this，也就是 **如果在函数内部定义了箭头函数，箭头函数的this值是等于该函数的this值，而在定义对象的时候，外部的函数并不存在，它的外部就是全局，所以this会指向window或者undefined。**

> 知识欠缺太多了，还需继续努力

欢迎批评指正
