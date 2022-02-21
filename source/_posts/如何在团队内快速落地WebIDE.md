---
title: 如何在团队内快速落地WebIDE
date: 2021-09-16 15:16
categories: 原创
---

在云音乐团队内部的WebIDE终于落地了。目前功能上覆盖前端正常的业务开发已无问题，期间也踩了不少坑，故写此篇文章，给想要在团队落地WebIDE做一份参考。当然，一些很细节的问题的处理，一篇文章是不好涵盖的，本文更多的是设计的方案及落地的方向的参考。

<!-- more -->

## WebIDE的意义

其实在发起WebIDE这个项目之初，很多人，上级、同事、部门领导，都会对项目发起质疑：意义在哪？能带来什么业务价值？

说实话这个问题，最初包括现在我也觉得很难回答，新人上手更方便？远程办公？代码更安全？统计开发人员的写代码时长？私有插件域？

但是这些，似乎并不是业务开发中的一个特别痛的点。当然，在小程序，faas或者云服务的业务确实能带来一些业务价值，毕竟，这些服务本身有一部分就是靠WebIDE吃饭的。但是排除了这几个业务场景，对于大部分普通的业务团队，其实WebIDE的意义，业务上的价值说实话并不大。

但是衡量一个项目的价值，并不能仅从业务的角度去思考。WebIDE的其他价值，我个人从团队、研发效率、公司层面梳理了几个点（都是做的差不多之后才凑出来的）：

1. 激发团队活力，提升新人留存率。本身作为一项不太普及的技术，其实或多或少能够激发团队的思考，增加团队活力。而作为一项未来可能成为前端基建的技术，对于新人的留存是有帮助的。基建不完善，那么新人招来了，内心必定会吐槽，流失也就理所当然。
2. 切换项目更方便。对于multirepo的前端团队，相信不少同学经常会在3-4个项目间切换，项目启动关闭一次较麻烦不说，对电脑的电量，内存都是不小的负荷。而webide将这一部分负荷转移到远程容器，省去了本地的负荷和每次启动打开的不便(finder -> terminal -> run dev -> open vscode)，说实话，个人感觉开发起来更方便。毕竟启动项目就是打开一个网页，而关闭项目也无需担心进度未保存。
3. 团队配置化的统一。统一化插件、ide配置，代码移交更顺畅
4. 远程办公。疫情之下，不多说，不必合适的设备即可写代码，便利性极大提升
5. 代码更加安全。稍加限制，即可防止代码被拷贝，同时从公司层面，衡量程序员的工作时间会更加方便（ps：我是不会把这个功能做进去的，程序员使用IDE的时长对衡量程序员的价值没有多少参考意义）
6. demo编写方便。开源的公开的工具很多，但是往往需要私有包，这一点外部开源项目基本无能为力
7. 研发平台的整合，闭合研发链路。团队大了，各个平台层出不穷，开发过程中往往会伴随多个平台的切换，将IDE嵌入网页，同时整合各个研发平台，避免反复切换平台带来的繁琐与不便
8. 私有插件。当然避免团队插件发布共有域的方案也有，但是直接搭建私有插件市场，更加方便直观。

个人认为，讨论一项还未落地的技术的意义，不见得是一个有意义的讨论。WebIDE的意义有多大，更多的取决于团队能把，或者说想把这个事做的多大。所以本文不在此赘述，对IDE感兴趣？干就完事了。

## 业内主流产品、技术方案对比

虽然WebIDE本身还未流行，但是在各大云厂商，各大国内大厂均已有落地产品，谈不上一个比较新的技术，而其实现方案，也基本明确并且成熟，此处挑几个进行简单介绍。

### vscode流派

比较流行的方案，代表有[coder](https://coder.com/), [code-server](https://github.com/cdr/code-server)，[vsonline](https://github.com/features/codespaces)。

这一类IDE产品，基于vscode进行web端的一些修改，同时贴合github，实现个人项目的webide开发。

但是缺点很统一：收费且团队难以接入使用（团队登陆检验，gitlab接入），代码安全也难以保证

### Theia

[theia](https://theia-ide.org/)比较流行的方案。一部分功能复用vscode，但是设计初衷便是为了扩展，所以私有化部署，定制化功能极其方便。

目前有的产品有：

[gitpod](https://gitpod.io/)，阿里系ide（自研kaitian不是基于theia），华为云ide，google云ide。

在扩展能力上，远超vscode，同时具备全开源，社区活跃度较高，接入改造成本低，不收费的优点

本人在团队内落地的IDE也是基于该框架。

###  stackblitz

[stackblitz](https://stackblitz.com/)相信大家都不陌生，非常方便的一款写demo的IDE，本来个人认为他是与[codesandbox](https://codesandbox.io/)一个流派的，但是他推出了[web container](https://blog.stackblitz.com/posts/introducing-webcontainers/)的概念，提供了非容器化方案的纯前端node环境，可以说非常有价值。

目前该方案不开源，目前猜测较多的是service worker实现请求拦截，webassembly提供了node的环境，从而实现不需要远程机器，全部都跑在浏览器的一个IDE。

整体来讲，其技术方案的优点在于不消耗远程资源，但是缺点，一个是不开源，一个是毕竟是模拟的node环境，在各个系统的一些层面可能会有所缺失（猜测）。

### 其他

还有一些其他的方案：

1. 基于前端构建的[codesandbox](https://codesandbox.io/)，这个网上分析的文章比较多，此处不赘述，其缺点在于没有完整的命令行，跑跑基于webpack的前端页面还行，node代码不能跑起来的，更多的是拿来写ui demo的一个工具。
2. 阿里的kaitian，纯自研的一款IDE，尝试取代THEIA的一款纯自研的IDE，目前未开源
3. coding团队的webide，[coding](https://coding.net/)，曾经开过源，目前不明

### 整体对比

1. vscode流派：付费，接入不便，扩展不易
2. theia流派：业内流行的开源方案，成熟落地产品多，扩展非常容易
3. Stackblitz: 技术方案独特，纯前端的WebIDE，但是目前看除了写demo快，团队接入和扩展都没有提供方案
4. codesandbox: 前端构建的，更多用来写demo
5. kaitian, codeing等自研，不开源，具体不太清楚

### 接入方案选择

对于团队需要的WebIDE，个人认为满足日常的开发是必须的。所以易扩展，一定是排在第一位的，这一点非theia莫属。而开源、不收费、稳定性好、社区活跃也是theia的长处。长期来看，theia的接入成本低、扩展性好，社区活跃方便后期维护，是一个非常棒的方案。本人在云音乐团队落地的也是最终采取的这一方案。



## 如何在团队内快速落地

说了这么多，其实到这一章才算正式开始落地。

WebIDE作为一项成熟的技术，这几年才逐渐的出现在较多的场合，这里面也是有技术方面的一些原因。目前各大WebIDE的方案，基本都是基于容器的方案，这一方案，需要团队有比较完善的容器化的管理及动态部署的能力。

所以在落地前，容器化部署的能力是需要团队所必须具备的，当然，使用外部商业化的也可。

除此之外，一些基本的基础设施也需要比较完善，比如gitlab、成员管理、前后端项目部署等基建产品。

具备上述能力后，才可以满足快速落地的要求。

### 整体结构

笔者认为，WebIDE的落地可以充分发挥互联网小步迭代，快速上线的迭代方式。先搭小汽车，再做摩托车，最后再到小汽车。一步一步对功能进行完善。

在笔者看来，一个自行车版WebIDE所需要具备的基础能力有：

2. 团队成员管理系统 or oa接入
3. 团队gitlab接入
4. 私有包安装的能力
5. IDE管理，新建，关闭，删除等基本功能
6. 支持前端项目运行

出于满足上述能力的需求，笔者最初（后来也只是新增了一些独立的工程）将项目划分为4个工程：

1. Ide-studio: 前端管理后台，负责IDE的新建，关闭，后期可拓展出插件市场等功能
2. Ide-studio-backend: 纯nodejs后台，一方面负责管理后台的接口，另一方面接入团队成员管理系统，后期可用来拓展私有插件以及共有插件等功能，同时处理和容器化部署平台的协作。
3. Ide-extension: 用于开发theia扩展，定制团队特有功能，比如接入团队成员管理系统，记录活跃时间，集成团队内部研发平台等
4. Ide-docker: 用于存放ide的docker相关文件。由于theia本身是为拓展而生，只需要一个package.json即可生成ide相关，所以单独将docker镜像的构建伶出来

### 落地方案

#### theia容器构建

theia本身已有容器[启动方案](https://github.com/theia-ide/theia-apps#theia-docker)，已经提供了dockerfile文件，可以直接使用该docker进行部署，笔者团队使用的是基于[theia-docker](https://github.com/theia-ide/theia-apps/tree/master/theia-docker)进行的修改。

官方dockerfile如下，笔者直接在上面增加了自己的注释：

```dockerfile
# 选择容器底层node依赖
ARG NODE_VERSION=12.18.3
FROM node:${NODE_VERSION}-alpine
# 增加theia install时需要的必要系统依赖
RUN apk add --no-cache make pkgconfig gcc g++ python libx11-dev libxkbfile-dev libsecret-dev
ARG version=latest
WORKDIR /home/theia
ADD $version.package.json ./package.json
# 这个GITHUB_TOKEN，需要在容器构建时指定(由于部分npm包安装需要从github拉取，使用github_token才可以顺利安装)
ARG GITHUB_TOKEN
# 安装，同时清理掉全局缓存
RUN yarn --pure-lockfile && \
    NODE_OPTIONS="--max_old_space_size=4096" yarn theia build && \
    yarn theia download:plugins && \
    yarn --production && \
    yarn autoclean --init && \
    echo *.ts >> .yarnclean && \
    echo *.ts.map >> .yarnclean && \
    echo *.spec.* >> .yarnclean && \
    yarn autoclean --force && \
    yarn cache clean

# 重新选择底层系统依赖，目的是保留容器系统极简，去掉了theia安装过程中需要的系统依赖，从而减小镜像提及
FROM node:${NODE_VERSION}-alpine
# See : https://github.com/theia-ide/theia-apps/issues/34
RUN addgroup theia && \
    adduser -G theia -s /bin/sh -D theia;
RUN chmod g+rw /home && \
    mkdir -p /home/project && \
    chown -R theia:theia /home/theia && \
    chown -R theia:theia /home/project;
# 增加git bash等基础系统依赖
RUN apk add --no-cache git openssh bash libsecret
ENV HOME /home/theia
WORKDIR /home/theia
COPY --from=0 --chown=theia:theia /home/theia /home/theia
EXPOSE 3000
ENV SHELL=/bin/bash \
    THEIA_DEFAULT_PLUGINS=local-dir:/home/theia/plugins
ENV USE_LOCAL_GIT true
USER theia
# run theia服务，默认运行在3000端口
ENTRYPOINT [ "node", "/home/theia/src-gen/backend/main.js", "/home/project", "--hostname=0.0.0.0" ]
```

整体来讲，这是一个比较标准的前端工程运行文件，对docker没有基础的搜一搜看懂也不难。

不过，其中几个实际踩的坑需要注意一下：

1. github_token需要指定，否则会出现部分npm包无法下载的问题。使用自己的github_token即可，生成方法见[官方文档](https://docs.github.com/cn/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token)
2. 构建镜像的时候注意关闭自己电脑的代理(比如charles等)，也会造成npm包安装问题

当然，这是theia官方的镜像，实际实现自行车版本的WebIDE的时候，还需要执行如下操作：

1. 注入git公私钥，此处建议使用单个仓库的私钥，使权限最小化
2. 自动clone目标代码
3. 注入npm 私有域，从而使用者无需再进行配置
4. 执行用户自定义的自动化脚本

这些操作可以自己写一个node脚本，在最后同theia服务的启动一起执行一遍即可。

而在实际的落地及使用过程中，还遇到一些实际的问题：

1. git公私钥如何获取，此处在IDE容器内确实不好获取，存在鉴权等问题，故笔者是在容器创建时(此时是有用户身份信息的)，将相关变量注入到容器的环境变量中，容器内直接拿环境变量的数据构建公私钥即可
2. root权限，如dockerfile中所示，theia出于容器的安全考虑，创建了一个工作区，用户的身份都是theia，而系统内的一些依赖其实是不够的，比如gcc, g++等部分npm包安装所需要的系统依赖都没有，另外也没有一些必要的系统依赖比如，curl。所以可以给用户开放root权限
3. 命令行问题，默认使用的是linux的bash，是没有git、高亮等支持，体验非常差，所以可以把oh-my-zsh集成进去

为了解决上述问题，笔者在原有的dockerfile中增加了如下命令

```dockerfile
# 增加系统依赖，以便npm安装可以顺利进行
RUN apk add --no-cache make pkgconfig gcc g++ python curl

# 安装zsh，美化命令行
RUN apk add --no-cache git openssh bash zsh libsecret-dev

# 增加root身份，以便用户可以自行增加全局系统依赖
RUN chmod 4755 /bin/busybox
RUN echo 'root:danger' | chpasswd

# 修改最终的启动命令
ENTRYPOINT ["/bin/sh", "/home/cloud-theia/start.sh"]

# 在start.sh中，执行git,npm全局配置的脚本，并且启动theia自身的服务
```

至此，容器的功能基本完善。可以直接本地构建一个跑起来看看。

#### IDE管理页面搭建

这一部分是比较常规的前端页面，此处大家应该耳熟能详，使用一些中后台模版搭建即可。

主要具备的功能有：

1. IDE的管理：增删查改
2. 插件的管理：私有/公有插件，插件的上传及管理
3. 基本的团队成员登陆/注销

其中，插件管理可以使用theia本身推荐的[openvsx](https://open-vsx.org/)，本身全开源，也推荐私有化部署后使用。

笔者当时由于IDE管理系统想和插件管理系统直接无缝切换（懒），只是对接口通过node后台直接转发到openvsx，从而获取其数据，简单快捷的实现了公有插件的获取/下载。

#### IDE后台服务搭建

笔者基于egg.js搭建了IDE的后台，主要承载的能力有：

1. IDE管理，比较常规的后台接口
2. 插件管理，私有插件的自存储，公有插件进行转发到openvsx
3. 处理网易轻舟云容器部署、域名转发等服务的交互（网易轻舟，个性化等用下来体验还不错）
4. 处理IDE容器内，插件安装/搜索时需要的一些服务处理

5. 基本的成员登陆/注销

整体来讲，代码基本跟平时的node服务无差。不再赘述

#### theia扩展

这一部分，应该是大部分没有接触过theia的同学比较难以处理的地方。虽然说theia本身提供了非常方便的扩展能力，但是该方便，还是建立在你对theia的构建有一定的了解之上。

官网介绍的不多，可以看[这篇文章](https://theia-ide.org/docs/authoring_extensions/)，跑起来问题不大，剩下的用到了查看源码即可。

但是接入自己的团队，其实需要扩展的地方不多，笔者目前也仅仅改动了以下几条：

1. 团队成员登陆/注销（纯前端扩展）
2. 团队logo、成员信息展示、相关帮助链接的跳转

这一部分都还没有涉及到theia的jsonrpc，后端扩展能力。但是自行车版足以。

上个图吧，其实在扩展能力上，只是用到了一些皮毛：

![image-20210916151858921](https://tva1.sinaimg.cn/large/008i3skNly1guiht5jz3zj61up0u0wjn02.jpg)

笔者在实际使用过程中，还有以下几个环境变量的改动：

1. VSX_REGISTRY_URL，这是插件市场的插件源改动，默认是指向openvsx，要实现私有插件，直接改变其到自己的私有市场并实现相关接口即可
2. THEIA_WEBVIEW_EXTERNAL_ENDPOINT，webview的web版方案，默认是指向了${uuid}.minibrowser.${hostname}，由于在实际落地场景中，这样修改域名的写法，https泛域名证书也无法使用，必须一个ide申请一个，而webview借助了 service worker，如果不使用https就无法工作，所以笔者这边直接改成了hostname，其利害及缘由可以参考[此文章](https://blog.mattbierner.com/vscode-webview-web-learnings/)
3. THEIA_MINI_BROWSER_HOST_PATTERN，同上，也是改成了hostname，来完成图片文件的预览

#### 其他

文档站建设，模版建设等一些常规的操作，不再赘述

### 总结

整体来讲，上述都搭建好之后，一个团队可用的webide就做好了。

目前笔者团队的webide，已经能够满足日常的开发使用，后续会进一步增加扩展，作为各个研发平台的承载点，解决目前开发过程中的不同平台反复切换的问题。

当然，实际落地中，还会有很多琐碎的细节问题，大家可以一起交流哈。



## 最后

webide作为一项成熟的技术，团队落地整个流程下来，笔者认为回头去看，其实没有什么特别大的坑点以及技术上的难点。

原本题目准备起：前端小白如何在团队内落地webide，但是后来想想，这就有点误人子弟了，虽然前端相关的代码确实没写几行，但是对其他的一些知识需要一定的储备，比如容器化、node、运维等知识。

最后，笔者在建立了完备的webide之后，自己的业务都已经在webide上进行开发，但是团队的大部分成员还是持保守态度。后续会继续提升开发体验，让越来越多的成员接纳这个套了老技术的新IDE。

