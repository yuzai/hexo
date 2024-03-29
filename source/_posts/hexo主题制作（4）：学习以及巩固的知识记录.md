---
title: hexo主题制作（4）：学习以及巩固的知识记录
date: 2016-10-06 11:22:25
tags:
- hexo主题制作
- javascript
- css
categories: 原创
---

记录我在hexo制作主题时学到的知识点
<!--more-->
### 学习笔记
> 知识不在乎大小，在乎积累，在乎延伸

### hexo的功能
刚接触hexo的时候，我觉得hexo并没有完成什么功能，我想要做的还得自己写ejs模板来实现，
index.ejs,archive.ejs,tag.ejs,category.ejs,post.ejs,都是自己写的。但是在使用过程中，我就不断的感叹hexo的强大了。
#### 内部的路由
首先就是路由了。比如index.ejs做好之后，访问'/'就是访问该界面，而archives就是访问对应的'/archive'页面，至于文章，每一篇文章都有一个自己的界面，只要在post.ejs中写好，就可与实现/2016/9/15/title就是访问那一篇文章的页面。它内部建立的路由，只要了解，然后自己去写对应的ejs，就可以轻松的实现想要的功能。
#### 内部变量以及函数
 内部提供不同的变量，page在Index中是所有的文章，而在tag.ejs中，就是对应tag所对应的文章，也就是说，自己不需要选择，直接在tag中使用page变量，就可以实现只显示当前tag对应的文章。虽然从所有文章中查找不是什么难事，但是提供这样的变量，大大简化了开发者需要的工作。开发者要做的，就只是设计页面，当然，还需要参考手册。这里有一份一直在看的手册，有点过时，但是很大部分内容没变，毕竟，有时候并不能访问hexo的主页。附上链接[hexo手册](http://wiki.jikexueyuan.com/project/hexo-document/)。可以下载，这样搜索的时候就很方便了。称赞一下[极客学院](http://www.jikexueyuan.com/)。这个手册做的很好，做成pdf也不容易。
内部提供的函数也很好用，list_archive()s，list_categories()等等，博客想有的数据都有了，而且可以通过修改css格式的方法来改变显示的样式，真的很方便。
#### 实时更新config文件可配置建立服务器
  很多功能hexo都帮忙实现了，开发者只需要安安心心做ejs模板就可以完成博客的搭建，当然，也可以直接选择已经有的模板，改一改立马就能用了，不过我还是喜欢自己做主题（模板），随心所欲。也刚好巩固一下自己的前端知识。

---
### 用到的css知识点
整体来说，css没有用到什么特别有花样的，只是巩固了一下自己前一段所学的css知识。w3school。真的相当基础
1. bootstrap的导航条、栅格系统、按钮（毕竟好看啊）、well
2. 原生css: media,position(relative,absolute,fixed),float,clearfloat。

---
### javascript
整体来说，这次进步比较大的地方是javascript，很多知识，都是看过了就忘了，只有真正使用的时候才能记得更深刻。
1. 原生选择器，getElementsByClassName().这个函数我写了n遍，一开始是大小写不对，接下来是element **s**。
2. 添加事件监听器，documen.addEventListener('scroll')，因为页面需要监测滑动，右侧的侧边栏需要在一定的位置固定下来，所以做了一个滑动的监听器。
3. setTimeout(func,time[,aug])，在之前学的，都没有讲参数如何引入，最后是找了手册才翻到引入参数。第一个参数是函数名称，不带()，可以传递匿名函数。看到这个函数，才真切的感受到，以前学的真的都只是基础的用法。
4. jquery，不得不说，jquery很好用。这次使用到了addClass(),css(aug),css({aug:aug}),animate(),animate很强大，不过对color这种属性并没有作用，应该是对于别的属性，渐变都很好做，但是对于颜色，至少我这个版本的jquery并不能实现颜色渐变。

---
### 总结
到现在，基本上hexo的界面做的差不多了，基本上我想要的功能也都实现了，这次的总结，主要就是：
1. hexo很强大，nodejs就是我下一步的目标
2. css,js必须要实际用，用起来，理解才会更加透彻
