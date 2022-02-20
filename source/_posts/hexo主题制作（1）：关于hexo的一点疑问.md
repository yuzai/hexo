---
title: hexo主题制作（1）：关于hexo的一点疑问
date: 2016-09-28 09:06:04
tags: hexo主题制作
categories: 原创
---
hexo变量的作用域
问题：page变量在archive.ejs和post.ejs中的含义不同，不好在post页面访问整个博客的所有文章列表
<!--more-->
根据手册，archive.ejs中的page表示的是所有的文章，也就是page.posts的元素就是每一篇文章。而在post.ejs中，也就是访问:year/:month...某篇文章时，page变量就是archive.ejs中的page.posts中对应于该文章的变量。
我希望实现的功能是在post.ejs中，也就是展示每篇文章的页面时，左侧显示文章，右侧显示toc,同时toc下面显示我发表过的文章。但是在post.ejs中并不能访问archive中page.posts变量，在post.ejs中，page就表示当前页面，page(in post.ejs)=page.post[index] (in archive.ejs)。所以不能访问到整个的博客文章变量，不好将其显示出来。
### 思考
变量是内部定义好的，框架这样安排是有它的道理的，这样，在post中可以很方便的处理当前文章的内容，而在archive中，可以直接对所有的文章进行相应的处理。在性能上应该有不少提升，很想知道内部的实现原理。不过要看源码了。暂时还是把自己的博客主题做好，不容易啊。
### 暂时的解决方案
hexo提供的辅助函数（helpers）可以解决这个问题
` <%- list_archives([options]) %> `
  可以显示当前的列表，具体的参数设置可以查看手册
  `<%- list_posts([options]) %>`可以显示所有的文章，具体有个格式，索然可以通过更改options获得自己想要的格式，不过获取这个格式之后，可以自己添加js,css来实现想要的渲染效果。
