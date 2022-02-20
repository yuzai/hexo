---
title: es6笔记
date: 2016-11-15 21:27:26
tags:
- javascript
- es6
categories: 学习笔记
---
es6的学习笔记
<!--more-->
## 3.1 对象字面量

--------------------------------------------------------------------------------

> 三个改进的地方：1.属性值缩写，可计算属性名称，方法定义

### 3.1.1 属性值缩写

当变量已经定义后，再定义对象的时候可以直接写变量名称，就会给该对象增加一个属性，属性名称为变量名，属性的值就是对应的变量的值
```js
var store = {};
var s=[1,2,3,4];
var api={s,getItem,setItem,clear};

function getItem(key){
  return key in store?store[key]:null;
}
function setItem(key,value){
  store[key]=value;
}
function clear(){
  store = {};
}
console.log(api.getItem);
console.log(api.setItem);
console.log(api.clear);
console.log(api.s);
```
### 3.1.2 可计算属性名

定义对象时，用方括号括住的属性名，最终的属性名应该是以该属性名命名的变量的值。**不能和第一个特性结合使用**

```js
var expertise = 'journalism'
var person = {
  name: 'Sharon',
  age: 27,
  [expertise]: {
    years: 5,
    interests: ['international', 'politics', 'internet']
  }
}
console.log(person[expertise].interests);
```

### 3.1.3 方法定义

对象中的函数可以直接定义。不必通过function。setter和getter和以前的用法相同，别的函数可以直接采用函数名+参数+定义的形式定义对象的方法。

```js
var s=4;
var person={
  get fule(){
    return s;
  },
  set fule(value){
    s=value;
  },
  add(){
    return s+1;
  }
}
console.log(person.fule);//4
console.log(person.add());//5
person.fule=10;
console.log(person.fule);//10
```

## 3.2 箭头函数

```js
var example = function (parameters) {
  // function body
}
var example = (parameters) => {
  // function body
}
```

箭头函数看起来和匿名函数很像，但是实际上有很大的区别：他们没有函数名，他们被绑定了词法范围。

### 3.2.1 词法范围

```js
var timer={
  second:0,
  start(){
    setInterval(()=>{
      console.log(this);//object timer
      this.second++;
    },1000);
  }
}
timer.start();
setTimeout(function(){console.log(timer.second)},3700);//3
```

```js
var timer={
  second:0,
  start(){
    setInterval(function(){
      console.log(this);//window
      this.second++;
    },1000);
  }
}
timer.start();
setTimeout(function(){console.log(timer.second)},3700);//0
```

```js
var timer={
  second:0,
  start(){
    var that=this;
    setInterval(function(){
      console.log(that);//window
      that.second++;
    },1000);
  }
}
timer.start();
setTimeout(function(){console.log(timer.second)},3700);//3
```

通过使用箭头函数，可以保证调用对象的方法时this一直绑定在方法的词法范围内，也就是this一直指向对象（不太完善，大部分情况下）。而如果使用匿名函数，词法范围由使用该匿名函数的此法范围决定，一般是window

### 3.2.2 箭头函数的简写

```js
var example = (parameters) => {
  // function body
}
var double = value => {//只有一个参数
  return value * 2
}
var double = (value) => value * 2
var double = value => value * 2
```

### 3.2.3 优点和使用场景
十分适合用在词法绑定的匿名函数上。
```js
[1, 2, 3, 4]
  .map(value => value * 2)
  .filter(value => value > 2)
  .forEach(value => console.log(value))
// <- 4
// <- 6
// <- 8
```
**注意事项**
当箭头函数的隐含返回值（没有用return）时，必须在对象外围加上()不然和函数的定义是一致的。

```js
var objectFactory = () => ({ modular: 'es6' })

[1, 2, 3].map(value => { number: value })
// <- [undefined, undefined, undefined]

[1, 2, 3].map(value => { number: value, verified: true })
// <- SyntaxError

[1, 2, 3].map(value => ({ number: value, verified: true }))
/* <- [
  { number: 1, verified: true },
  { number: 2, verified: true },
  { number: 3, verified: true }]
*/
```

## 3.3 assign destructuring
最简单的特性，方便了把对象的属性值赋值给变量
### 3.3.1 解构对象

```js
var pseudonym = character.pseudonym//es5
var {pseudonym} = character //es6

var alias = character.pseudonym//es5
var { pseudonym: alias } = character//es6

var person = { scientist: true }
var type = 'scientist'
var { [type]: value } = person //es6
console.log(value)
// <- true
value = person[type] //es5
```

### 3.3.2 解构数组
跟解构对象一样，只是采用方括号。毕竟操作的对象是数组

```js
var left = 5
var right = 7
[left, right] = [right, left]
```

### 3.3.3 解构函数

```js
function sumOf (a = 1, b = 2, c = 3) {
  return a + b + c
}
console.log(sumOf(undefined, undefined, 4))
// <- 1 + 2 + 4 = 7

function carFactory ({ brand = 'Volkswagen', year = 1999 } = {}) {
  console.log(brand)
  console.log(year)
}
carFactory()
// <- 'Volkswagen'
// <- 1999
```

### 3.3.4 使用场景

```js
function getCoordinates () {
  return { x: 10, y: 22, z: -1, type: '3d' }
}
var { x, y } = getCoordinates()

function splitDate (date) {
  var rdate = /(\d+).(\d+).(\d+)/
  return rdate.exec(date)
}
var [, year, month, day] = splitDate('2015-11-06')
```

## 3.4 rest参数和扩散运算操作符

```js
function print () {
  var list = Array.prototype.slice.call(arguments)
  console.log(list)
}
print('a', 'b', 'c')
// <- ['a', 'b', 'c']
```

在es5中，获取参数比较繁琐，在es6中，使用rest参数可以很好的获取函数的参数
### 3.4.1 rest 参数

```js
function print (...list) {
  console.log(list)
}
print('a', 'b', 'c')
// <- ['a', 'b', 'c']

function print (first, ...list) {
  console.log(first)
  console.log(list)
}
print('a', 'b', 'c')
// <- 'a'
// <- [b', 'c']
```

通过3个点来获取参数到list，...前面的参数可以按照原来的方式进行赋值，后面的多余参数会直接赋值给list
箭头函数必须加括号如果使用...参数，即便只有一个参数。否则会报错。

### 3.4.2 spread操作符
符号和...操作符一样，可以将数组或者类数组的元素返回到一个数组中。

```js
var a=[1,2,3,...[2,3],...[3,4],5];
console.log(a);//1,2,3,2,3,3,4,5
```

## 3.5 模板字面量
可以使用\`\`来定义字符串，这样在字符串内部使用单引号双引号都可以。另外一个好处是可以在其中插入js表达式

```js
var text = `This is my first template literal`
```

### 3.5.1 字符串插值

```js
var s=`This a template literal ${ `with another ${ 'one' } embedded inside it` }`
// <- 'This a template literal with another one embedded inside it'
```

### 3.5.2 多行模板字面量

可以方便的定义多行字符串

```js
var escaped =
'The first line\n\
A second line\n\
Then a third line'

var concatenated =
'The first line\n' +
'A second line\n' +
'Then a third line'

var joined = [
'The first line',
'A second line',
'Then a third line'
].join('\n')

//es6 with template literal
var multiline =
`The first line
A second line
Then a third line`
```

###　3.5.3 Tagged Templates
不是很懂，有待下一次研究

## 3.6 let 和 const
js变量提升以及作用域很让人困惑，没有块级作用域
### 3.6.1 let声明和块级作用域
let变量具有块级作用域

```js
let topmost = {}
{
  let inner = {}
  {
    let innermost = {}
  }
  // attempts to access innermost here would throw
}
// attempts to access inner here would throw
// attempts to access innermost here would throw
```

最常用的地方就是for循环

```js
for (let i = 0; i < 2; i++) {
  console.log(i)
  // <- 0
  // <- 1
}
console.log(i)
// <- i is not defined
```

### 3.6.2 临时死区 TDZ
进入一个块级作用域，如果在let变量定义之前使用了该变量名，会抛出异常，如果代码在let定义之前企图访问用let定义的变量，都会抛出异常，因为let的变量有临时死区，也就是let变量定义之前，试图访问let变量都会报错。但是在函数中访问并不会报错，除非在定义之前执行了该函数

```js
{
  name = 'Barbara Penner'
  // <- ReferenceError: name is not defined
  let name = 'Stephen Hawking'
}
```

### 3.6.3 const声明
const变量和let变量都具有tdz，也就是在定义之前试图访问均会抛出错误。
const变量定义的时候必须要初始化，而且不能被显式赋值。在非严格模式下不会报错，但是该赋值语句不会起作用。但是只是指向的对象不能更改，对象本身可以被更改。

```js
const people = ['Tesla', 'Musk']
people = []
console.log(people)
// <- ['Tesla', 'Musk']

const people = ['Tesla', 'Musk']
people.push('Berners-Lee')
console.log(people)
// <- ['Tesla', 'Musk', 'Berners-Lee']

const people = ['Tesla', 'Musk']
var humans = people
humans = 'evil'
console.log(humans)
// <- 'evil'
```

### 3.6.4 let和const的优点
