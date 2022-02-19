---
title: 原生js封装一个下拉加载的小组件
date: 2017-04-06 21:38:30
tags:
- javascript
categories: 创作
---

之前在面试的时候，有些公司直接问我写过什么组件，而当时的我的回答是：[封装ajax](http://www.jianshu.com/p/4e1d2ee63da7)，封装事件处理（兼容性），[前端路由](http://www.jianshu.com/p/5a5813648d87)，这些方法组件，而并没有一个比如下拉加载，轮播，加载中，拖拽这样的配合css以及js的这种组件。当时最后的结果就是现场思考：一个轮播组件，该有哪些方法，应该暴露什么方法，暴露什么属性。所以最近静下心来，也刚好在自己做的一个前后端分离的[小项目](http://blog.xiaoboma.com/dazhequan2/)中用到了下拉加载，就顺带把这个给抽离出来。记录一下写组件的过程，希望能够加深自己的感悟。
<!--more-->

### 下拉加载的实现原理
下拉加载的原理其实并不复杂，实现方法也有很多种，这里我采用设置margin-top为负值，监听touchmove或者mousemove事件来检测手指或者鼠标的位移，从而移动margin-top，在达到下拉阈值之后，touchend或者mouseup事件进行回调函数的处理。从而完成重新加载。原理很简单，但是在我实现的过程中，还是碰到了不少问题。下面一一说明。

### 具体的实现
按照需求驱动型的思路，我要新建一个上拉加载类，这个类应该至少具有两个方法：
1. start：通过这个方法，上拉加载启动。
2. remove：通过这个方法，移除上拉加载相关事件。
同时，需要有一些参数：
1. content：在哪个元素上下拉进行加载
2. callback: 当下拉加载触发的时候要进行什么样的回调
3. ptr： 显示下拉的loading的元素
根据上述的说法，大致的结构应该是这样：

```js
var callback
var pullreload = function(options){
    this.content = options.content;//目标下拉元素
    callback = options.callback || (default自己设计一个默认的回调)//下拉触发后的回调
   this.ptr = options.ptr；
    this.start = function(){}//绑定事件
    this.remove = function(){} //清除绑定
}
```

可以看到，上述callback定义在外部，这是因为touchened,mouseup这些事件中需要对其进行操作，定义在内部，一方面是事件回调不好进行访问，另一方面，用户能够通过this.callback进行访问，这个可以有，但是我并不希望用户能够通过这样的方式修改callback，将其放在外部，除了内部函数，没有别人可以访问到，能够私有这个变量，达到私有的效果。事实上，个人感觉，用原生js写组件的话，**有些方法需要放在外部，而有些方法需要放在属性，有些方法放在prototype上。这些取决于实际的应用，不希望实例访问，当然是放在外部合适，而希望每一个实例都拥有，放在属性上更合适，而公用的方法，适合放在原型上。最后，外部通过立即执行函数包裹或者webpack等工具进行打包，都可以避免外部私有变量污染全局。**
除了上述变量，还定义了一些私有变量以及方法，最终的代码框架如下：

```js
//检测是否正在拉动,start设为true,end设为false，防止移动二次滑动产生干扰
var isDragging = false;
//是否达到阈值，下拉到一定程度之后，为true,否则为false
var isThresholdReached = false;
//记录下拉初始鼠标位置
var popStart = 0;
//设置初始阈值
var threshold = 20;
//记录上一次end回调是否执行完毕
var isend = true;
//记录当前是否处于最顶端
var isTop = true;

function getheight(event) {
   //通过event获取高度，因为pc端和移动端不同，单独写一个方法来解决兼容问题
};
function movestart(event){
  //记录初始位置，判断是否位于顶端
}
function moving(event){
  //检测滑动方向，进行相应的位移
}
function moveend(){
  //检测是否达到下拉阈值，达到则进行回调，否则恢复初始状态
}
var callback;
var pullReload = function(options){
  this.content = document.getElementById(options.content);
  this.ptr = document.getElementById('ptr');
  callback = options.callback || function(){};
  this.start = function(){
    //绑定事件
  }
  this.remove = function(){
    //移除事件
  }
  return this;
}
export default pullReload;
```
源码见[这里](https://github.com/yuzai/dazhequan2/tree/master/src/method/components/pullreload),里面是js和css分开写的，最后通过webpack进行一个打包从而完成组件化。

### 碰到的问题
1. content加载好之后，start中应该进行所有事件(touchmove)的绑定还是当touchstart或者mousedown之后进行事件的绑定？
    这里的话，有几个考虑，一方面，touchmove触发频率特别快，一直绑定给的话，对资源浪费很大，另一方面，touchmove并不需要实时绑定，因为当touchstart之后，如果此时并不处于最顶端，那么touchmove是完全不需要触发的，基于上述两点，我对touchmove,touchend事件的绑定在touchstart中当检测到用户处于最顶端的时候进行绑定。
2. event.preventDefault(),主要在touchmove事件中存在问题，因为touchmove事件的默认行为是滚动，如果不进行event.preventDefault()事件，那么在最顶部的时候，用户下拉造成滚动，那么此时滚动条会发生一些奇怪的行为，进而导致检测到的用户触摸点的位置发生偏差，从而导致阈值的不正确触发。而如果执行event.preventDefault()的话，又会带来另外一个问题，当用户处于最顶端的时候，上拉，因为event,preventDefault()又带来另一个问题，拉不动了，这个时候用户无法上拉。很尴尬，迫于无奈，我最终的解决方案是在touchmove中进行一个上拉下拉的检测，上拉就不禁止默认，下拉禁止默认行为。从而解决了这个问题。
3. touchmove,touchend事件绑定在content元素上还是document上？
这里主要是因为用户有可能在电脑上一滑，滑出了content的范围，就会导致touchmove,touchend没有触发，进而带来一系列的问题，所以稳妥起见，我绑定在了document上，当然，因为只有在最顶端的时候才会绑定个，所以性能上的损耗其实并不大，并不会影响用户非下拉状态下的体验。

### 小结
组件的考虑，其实还是在暴露的方法、属性，以及私有的方法，属性上的取舍，根据具体的情况进行取舍，才能上组件更加通用，可靠度更高。个人认为：不希望实例访问，当然是放在外部进行私有更合适，而希望每一个实例都拥有，放在属性上更合适，而公用的需要暴露的方法，适合放在原型上。
下一步的计划希望能将这个小小的组件放到npm上，之前也一直没有试过，用来体验一把npm install还是很好玩的。