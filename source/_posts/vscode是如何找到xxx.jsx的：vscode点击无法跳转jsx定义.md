---
title: vscode是如何找到xxx.jsx的：vscode点击无法跳转jsx定义
date: 2020.07.13 20:16
categories: 原创
---


## 问题描述
在新接手的项目中，存在vscode点击无法跳转到对应文件的情况，具体如下：
相对路径引入的xxx.jsx文件无法自动跳转，但是.js文件可以

针对这个问题，做了一个非常简单的demo来进行演示和问题的复现：[github](https://github.com/yuzai/vscode_gotodefinitions_test)

在该项目的入口文件src/test.js中就存在这样的问题

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1gzlimstdloj20my0d4q3t.jpg)

## 解决方案
经过对vscode源码debug，基本了解了大致流程。得到的正确解法如下：

### 解法1:
删除项目目录下的jsconfig.js，可以实现jsx，js文件的自动跳转

### 解法2:
jsconfig.json中配置如下：

```js
{
    "compilerOptions": {
      "jsx": "preserve", // 共有三种枚举： react, react-native, preserve，实测均可
   },
    "exclude": ["node_modules"]
}
```

### 改进解法2:
一般来讲，jsconfig.json的存在都是配合webpack alias进行快速跳转，故可以参考如下配置：

```js
{
    "compilerOptions": {
      "jsx": "react-native",
      "baseUrl": "./",
      "paths": {
        "@/*": ["src/*"]
      }
    },
    "exclude": ["node_modules"]
}
```
增加了baseUrl和paths，此时代码中可以进行如下引入：

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1gzlijttm3gj20my0d4q3t.jpg)

## 原理探究

### vscode源码本地启动
此部分可以参考vscode的[contribute文档](https://github.com/Microsoft/vscode/wiki/How-to-Contribute)。文档写的比较详尽，此处补两个点：
1. yarn配置淘宝源，等待时间会短很多
2. 一部分资源需要从github拉取，一般公司的命令行都会配置成gitlab的名称，可以局部修改成github的姓名邮箱。不然有部分资源会403。

### vscode相关代码定位
vscode源码主要分为两部分：src, extensions。由于本身对源码不熟悉，而且也只是为了定位问题，故没有仔细查看调用链路，最终得出以下结论：
寻找jsx文件是由内置扩展: typescript-language-features提供的，具体代码可以见[definitions.ts](https://github.com/microsoft/vscode/blob/master/extensions/typescript-language-features/src/features/definitions.ts#L31)。如果想编写支持其他文件的扩展，只需要注册registerDefinitionProvider这样的方法即可，可以参考[jump-definition](https://github.com/sxei/vscode-plugin-demo/blob/master/src/jump-to-definition.js)。
此处再往下追的话，会是一份typescript经过打包后的代码，故没有深究。看到这里的时候，其实可以推测进入js文件之后，左下角会出现这一个状态：

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1gzlik8js49j207302jq2s.jpg)

等该内置扩展加载之后，才能使用点击跳转的功能。这里可以参考该扩展的package.json:

![image.png](https://tva1.sinaimg.cn/large/e6c9d24ely1gzlikl08ctj20bw098dgx.jpg)

当检测到当前文件语言是js/jsx的时候，该扩展会被激活，从而出现左下角的加载状态。

## 小结
这个问题解决方案是非常简单的，不过之前在网上查了很久也没有得出结论（因为我的项目下有个jsconfig.json，即便什么配置都没有也还是找不到jsx文件），迫不得已去本地debug vscode的源代码。
不过还好，最终拿到了结果，顺利解决问题。希望能够让遇到类似问题的同学少走弯路。
