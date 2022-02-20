---
title: javascript垃圾收集和内存泄漏
date: 2017-01-26 22:10:21
tags: javascript
categories: 学习笔记
---
这几天在写代码的时候一直想到一个问题，内存的问题，这个问题解决不了，始终不能安安心心写代码（我这里又又又定义了一个变量，闭包能访问到它，但是我在闭包函数中并没有访问他，闭包一直存在，不占内存吗？之前也一直有一个问题，不解决也是寝食难安，见[事件处理函数中this的指向以及函数上下文的继承](http://www.jianshu.com/p/46a752b117fb)），以前写c,c++的时候，似乎每次全局作用域有数组我不用了，我就手动给它删除了（局部变量出了作用域就自行销毁），而写javascript的时候意识到，虽然避免使用了全局变量，可是因为闭包的存在难道不会导致内存的泄漏（只要闭包函数存在，就始终拥有对外部函数的作用域的访问权限）？
<!--more-->
之前第一次看《javascript高级程序设计》的时候，记得有一节是说垃圾回收机制，当时连对象，应用，原型什么的都没有搞清楚，那章基本上也就跳过了，现在回想起来，似乎我很少在js中对其内存进行管理，除了我不想要某个属性了，会delete一下，别的都没有进行管理，那么Js不需要内存管理吗？不会发生内存泄漏？

### js垃圾回收机制
Js有自己的垃圾回收机制，会帮助开发者管理内存。回收机制会查找应用无法到达的内存。这其实跟我们的初衷是有一些细微但是很重要的差别，该机制是寻找无法到达的内存，而我们想要的其实是寻找我不会再使用的内存。一块内存（一个变量，引用），只有开发者我们自己才知道会不会再用到，而检测机制，只能检测到程序中别的地方不会再调用的内存（也正常，要是检测机制能检测到那个变量我不用了才奇怪了）。所以，如果我们不使用某个变量了，让程序别的地方都引用不到它，垃圾回收机制就能够发现并处理它，而内存泄漏的根本原因就是我们不打算继续使用的内存还存在着引用，从程序别的地方可以访问到它，就不会对其进行释放。
综上可以这么说：**js的回收机制的关键是理解可到达的概念。从跟(window)出发，能够到达的变量都会留在内存中，只有无法到达的节点(变量，函数)才会被回收机制回收。从而完成内存的释放。而引起内存泄漏的根本原因就是存在不想要的引用，使得不需要的内存能够从根节点到达，从而无法释放该内存。**

### js回收机制示例
##### 普通对象回收机制
如下代码：

```js
function Menu(title) {
  this.title = title
  this.elem = document.getElementById('id')
}

var menu = new Menu('My Menu')

document.body.innerHTML = ''  // (1)

menu = new Menu('His menu') // (2)
```

在第一步执行完之后，此时由于document.body中改成了空，那么此时menu中this.elem是否还存在？根据Js垃圾回收机制，虽然document.body中无法访问到elem了，但是还能够从menu.elem中访问到该元素，存在指向该内存(document.getElementById('id'))的引用(menu.elem)，所以该元素占用的内存并没有被真正的释放。当执行第二步之后，menu.elem指向空，此时，没有引用指向该元素，该内存被真正释放。
##### 循环引用收集机制
先上代码：

```js
function setHandler() {

  var elem = document.getElementById('id')

  elem.onclick = function() {
    // ...
  }

}
```

上述代码，属于一个经典的闭包，虽然，elem.onclick中没有任何代码，但是依然在内存中保存了elem变量（闭包的特性），所以在这个例子中，存在如下的循环引用：

![循环引用.jpg](http://upload-images.jianshu.io/upload_images/3967512-2231f6c978f663b6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
elem变量的onclick指向的函数，内部保存了一个指向serHandler的作用域的引用，而该作用域中又存在该elem的引用，所以出现了循环引用的情况。这个onclick函数一般会随着elem的消失而被释放（在现代浏览器中，什么情况下不会清除见下文）。所以，在现代浏览器中，一般通过下列代码即可释放该内存。

```js
function cleanUp() {
  var elem = document.getElementById('id')
  elem.parentNode.removeChild(elem)
}
```

在现代浏览器中，移除elem节点就可以移除绑定的对应事件。（IE<8中不能，原因见下文。不得不说，以后我再也不用担心移除元素对应事件没有解除的问题了，毕竟2017年了嘛）。

### js常见内存泄漏
##### IE<8 DOM-JS 泄漏
在上面实例中，使用cleanUp函数清除元素节点并不能清除onclick函数，因为清除了dom元素，但是因为在elem.onclick的作用域中还存在对该元素的引用，所以并没有对其进行回收（IE老版本中使用的DOM对象采用的是引用计数的回收机制，在这里不进行深入讨论，引用计数的回收机制不行，我也就不深入了）。解决方法如下：
![解决方案.png](http://upload-images.jianshu.io/upload_images/3967512-7324eb4c9cef0c41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这样处理之后也就不存在循环引用的问题，如果需要在onclick函数中使用elem,直接使用this（事件触发函数中this指向元素本身）即可。因为onclick中没有保存对elem的引用，所以删除元素的时候不存在因为循环引用的关系无法删除elem。

##### IE<9 ajax内存泄漏
```js
var xhr = new XMLHttpRequest() // or ActiveX in older IE

xhr.open('GET', '/server.url', true)

xhr.onreadystatechange = function() {
  if(xhr.readyState == 4 && xhr.status == 200) {           
    // ...
  }
}

xhr.send(null)
```
xhr是一个被浏览器跟踪的对象，当xhr请求结束后，该引用会被浏览器回收，xhr就不能被到达，从而进行内存回收，但是IE<9不会做这个处理。是因为其实这里存在一个循环引用：
![ajax循环引用.png](http://upload-images.jianshu.io/upload_images/3967512-0ab87d2d2741ae72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图中，浏览器对xhr有一个内部的引用，在xhr回调函数处理结束之后会删除这个内部引用，但是因为在onreadystatechange函数中存在对xhr的引用，因为IE采用引用计数的回收机制，并不会对xhr进行清除，从而导致内存泄漏。解决方案如下图：
![解决方案.png](http://upload-images.jianshu.io/upload_images/3967512-c3a8ffc6da47080c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图中，在处理完xhr.send(null)之后增加xhr=null就可以使循环打破，从而在xhr回调函数处理完毕之后对xhr进行回收。
修正代码如下：
```js
var xhr = new XMLHttpRequest()

xhr.open('GET', 'jquery.js', true)

xhr.onreadystatechange = function() {
  if(this.readyState == 4 && this.status == 200) {           
    document.getElementById('test').innerHTML++
  }
}

xhr.send(null)
   xhr = null
```
在现代浏览器中，其实上述这两个问题均不需要进行考虑，主要是IE残留的对DOM的回收机制的问题。
##### setInterval
定时函数的作用域一直保存在内存中，造成内存泄漏的原因在于，定时函数里面的函数如果本身不实现任何功能，但是有没有clear，就会一直占用内存。这个只需要在不需要定时函数之后进行clear即可。
##### 闭包
闭包函数对外部函数的所有变量均有引用，所以只要闭包函数还存在，对这些变量的引用就一直存在，即便闭包函数中并没有对其进行访问，所以就造成了内存的泄漏，这一点其实也很好解决，在l临时使用的变量在使用之后令其=null即可。如下：
```js
function f() {
  var data = "Large piece of data, probably received from server"

  /* do something using data */

  function inner() {
    // ...
  }
 data = null;
  return inner
}
```


### 总结
这几天其实在网上看了很多文章，几乎每篇文章关于内存泄漏的书法都不太一致，有些文章说了一些循环引用的问题，但是没有指定是所有的浏览器不能使用还是只有IE,有些文章对js的回收机制并没有给出很好的解释。
我尽可能的总结了一些我认为比较正确的结论，由于这种泄漏的问题不好进行直接的验证，所以文章中可能会出现错误，如果有误，希望能进行指出！

### 参考文章
1. [4 Types of Memory Leaks in JavaScript and How to Get Rid Of Them](https://auth0.com/blog/four-types-of-leaks-in-your-javascript-code-and-how-to-get-rid-of-them/)
2. [Memory leaks](http://javascript.info/tutorial/memory-leaks)
3. [Memory leak patterns in JavaScript](http://www.ibm.com/developerworks/library/wa-memleak/)
