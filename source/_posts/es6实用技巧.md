---
title: es6实用技巧
date: 2017-01-30 23:04:43
tags:
- es6
categories: 总结
---

一些实际使用中经常会碰到的es6的语法,慢慢更新.
(2017.01.30更新)
<!--more-->

### object属性名缩写
当在定义object的时候，如果对属性值采用变量赋值的方法，当变量名和属性名相同时，可以只写一次。
```js
const name = 'xiaobo'
const person={
  name,
  age:16
};
console.log(person.name);//xiaobo
```

### 判断一个变量是否是数组
这个方法属于对Array类型的一个扩展

```js
  var obj ={
    name:'xiaobo',
    age:24
  }
  console.log(Array.isArray(obj));//false
  console.log(Array.isArray([1,2,3,[1,2,3]]);//true
```
