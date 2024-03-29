---
title: js快速排序实现及稳定性分析
date: 2017-03-10 15:05:26
tags:
- javascript
categories: 学习笔记
---

最近面试遇到了一个问题：快速排序是稳定的吗？。我当时想了想稳定的定义：相同元素在排序前和排序后的顺序不会发生改变，就成为稳定的。否则就是不稳定的。在简单形式化一下，如果Ai = Aj, Ai原来在位置前，排序后Ai还是要在Aj位置前。
那么快速排序是否稳定？教科书上的答案是不稳定。。。我还是打算自己探索一下。
<!--more-->

## 快速排序的原理
对于一个数组，从中随机选择一个数字（一般选取第一个），然后把整个数组中小于它的元素放在左侧，大于它的元素放在右侧，然后递归执行。

## 快速排序js实现1
按照上面的原理，快速排序也没有那么难嘛，我每次新建2个数组，left，和right，然后遍历原数组，从而将小于它的push进left，大于它的push进right，然后再进行递归即可。代码如下：

```js
function quick(arr){
  if(arr.length<=1){
    return arr;
  }
  var left = [];
  var right = [];
  var base = arr[0];
  for(var i=1;i<arr.length;i++)
  {
   // 判决条件
    if(arr[i]>base){
      right.push(arr[i]);
    }else {
      left.push(arr[i])
    }
  }
  return quick(left).concat(base,quick(right));
}
// console.log(quick([3,2,0,1]));
```

照着上面的写法实现的排序，是有可能发生相同元素的改变的，比如[1,2,1,0]，第一次遍历之后，left = [1,0]，right = [2],base = 1.从而新组成的数组就是1,0,1,2.原本处于第一个位置的1跑到了第三个1的右侧，顺序发生了改变，从而是不稳定的。事实上，如果我将上述代码的判决条件换成>=，这样，第一次排序之后，left = [0] base = 1,right = [2,1]，从而就变成稳定的了。
所以，快速排序到底是稳定还是不稳定的？我也不是很确定，看了教课书之后，我认为上述的算法严格来讲算是快速排序的一个变种，在快速排序的过程中新建了一些辅助数组，对空间的占用率更高。下面介绍实现手段2，这个是严格按照快排的定义来的。

## 快速排序js实现2
快速排序的实现，其实不用新建一些辅助数组，只需要在原数组中进行操作就可以实现，当然，js中可以先复制一份出来，以免改变原数组。关于真正的快排的实现，在这里我就不赘述了，[相关的文章](http://blog.csdn.net/morewindows/article/details/6684558)解释的很清楚，核心的思想就是在原数组上进行交换，在不新建数组的情况下实现左小右大的排序。代码如下：

```js
function quick_sort2(arr){
  var _arr = arr.slice();//复制一份，以免影响之前的arr
  return quick_sort(_arr,0,_arr.length-1);//进行排序
}
function quick_sort(arr,i,j){
  if((j-i)<=1)//如果数组长度小于1，不用排序
  {
    return arr;
  }
  var left = i;
  var right = j;
  var base = left;
  var center = arr[left];
  while(left<right){
   //从右向左扫描是否存在比基数小的数字
    while(left<right && arr[right]>=center){
      right--;
    }
    if(left<right)
    {
      //将小于基数的数字放置到左侧
      arr[left] = arr[right];
      left++;
    }
   //从左向右扫描是否存在比基数大的数字
    while(left<right && arr[left]<center){
      left++;
    }
    if(left<right){
       //将大于基数的数字放置到右侧
      arr[right] = arr[left];
      right--;
    }
  }
  //更新基数
  base = left;
  arr[base] = center;
  quick_sort(arr,i,base-1);//递归对左侧进行排序
  quick_sort(arr,(base+1),j);//递归对右侧进行排序
  return arr;
}
```

上述代码就是严格按照最经典的快速排序写成的代码，这个算法没有新建数组，全部都是在复制出来的arr上进行排序，能够很好的节省空间，但是在排序过程中，有可能会导致相同元素的顺序发生改变，从而是不稳定的。教课书上的写法就是这样，所以是不稳定的。

## 小结
相比之下，第一种算法更加清晰易懂，但是其实第一种算法新建了很多辅助数组，消耗的内存比较多，而第二种算法，没有新建数组（除了最开始的复制一份），没有新建新的数组，消耗的内存少，但是在操作的过程中，因为左右的交替扫描，虽然基数和相同元素的位置不会发生改变（主要保证>=即可），但是别的非基数的相同元素很有可能发生位置颠倒的情况，所以说这种方式的快速排序是不稳定的。

## 稳定性的好处
关于这一点，如果数组的元素是纯数字，那么顺序真心没有什么意义，但是如果是一个对象，假设是学生，如果希望先按照学号排个序，然后再按照成绩排个序，如果第二次的排序是稳定排序算法，那么对于相同成绩的学生，其学号必定是按照之前的次序，而如果采用非稳定的排序算法，相同成绩的学生的学号有可能发生改变，这个时候就需要对相同成绩的学生进行重新按照学号排序。也就是说：
排序算法如果是稳定的，那么从一个键(学号)上排序，然后再从另一个键上(成绩)排序，第一个键排序的结果可以为第二个键排序所用。

## 参考文章
1. [白话快速排序](http://blog.csdn.net/morewindows/article/details/6684558)
2. [排序算法稳定性](http://baike.baidu.com/link?url=ARs3uxoNnDIriK636qMlzMcbFcxwC4hdlXQZorYRo4Q4JTnaaeG1uTBNEQ1LGqlIGNj0xBFa2acDk5bZ5I8SXGjX7Z6BGVp2OfJlPc-emP5quSBoFHvIkzokVHIJhPgRk5RqGtQCglEUdUZdF58fJCjIOjvoiEpbXZgJZC30foi)