---
title: 设计一款赏心悦目的 React 状态管理库
date: 2023-03-22 15:47
categories: 原创
---

本文的出发点是记录笔者在设计一款状态管理库时的心路历程和思考。起初笔者仅仅是想做一些粗浅的封装，但是实际思考后，非常希望能够开发一款用起来好用，让自己觉得赏心悦目的状态管理库。

<!-- more -->

(提前说结论：笔者最终的方案，基于 zustand，内置了计算属性、suspense 的支持，并开箱即用 react ssr 的最新方案：renderToPipeableStream。且保留了中间件的功能，能够承接现有 zustand 周边中间件的能力。)

本文主要包含以下几个部分：

1. 状态管理库的本质，要解决的问题，应该提供哪些开箱即用的能力
2. 现有各状态管理库在基础能力上的区别及优缺
3. 为什么选择 zustand 作为底层实现，详解 zustand，分析其优势与不足
4. 一个现代成熟的状态管理库还应该具备哪些能力
5. 为 zustand 弥补不足：增加计算属性、suspense 支持，提供 react renderToPipeableStream 的开箱即用能力

## 状态管理的本质

简单来说，状态管理的本质便是能够从组件树的任何位置读取和操作状态。

对于 react、vue 来讲，组件间状态的共享，便是状态管理的最核心的地方。

但是在实际的业务开发中，往往需要共享的不仅仅是状态，还有操作状态的全局方法（同步/异步），以及派生/计算状态的共享。

在 vuex 中，提供了 state 来定义基础状态，getter 来定义派生/计算状态，mutation & action 来定义操作全局状态的方法。

在 redux 中，提供了 state 定义基础状态，reducer 来提供操作状态的方法。（ps：从这里可以看出，其实 redux 没有提供简单的定义派生状态的地方，虽然有方案，但是确实不如 vuex 的 getter 来的显而易见）。

在 recoil/jotai 中，提供了 atom 定义基础状态，selector 来定义派生状态，但是缺少明确的定义操作状态的方法的地方（也有方法，但是不那么显而易见）。

抛开渲染优化、组件更新心智模型、原子化/中心化不谈，单从开发者的诉求来看，笔者认为，状态管理的本质，便是给组件树中的任何组件提供：

1. 基础状态的读取
2. 派生/计算状态的读取，下文统称 派生状态
3. 操作基础状态的方法(同步/异步)，下文统称 action。

在此之上，才是原子化、更新机制、渲染优化，以及数据请求，store 拆分 等解决方案的讨论。

## 各状态库的对比

各个状态库诞生的时间和背景都不同，不存在优劣，只存在是否适合当下的背景以及能够解决当时的问题。

在展开之前，可以简单看下各个库的发家史：

react 在 2013 年发布第一个版本，这之后 2015 年左右出现了 react-redux，当时非常新潮的 flux 单项数据流架构(当时似乎也没人念叨 reducer 难写，模版代码太多，能解决问题就不错了)。

而 vue 在 2014年发布第一个版本，这之后 2016年左右出现了 vuex，从 API 上看，还有 redux 的影子: mutation。但是也做了改进，异步 action 的支持及 派生状态 getter 的天然支持。

这中间，在 vue 问世之后，2015年，可以看到 mobx 项目的开启，给 react 项目也加入 reactive。不过笔者对这个库不了解，不做过多说明。

而在这之后很长一段时间，其实并没有什么数据状态库的问世，更多可能是围绕 redux 进行各个中间件的开发（猜的）。

直到 2019 年 hooks 的问世，才带来了一些新的状态管理库，比如 2019年的 zustand, 再到 2020 年的 recoil、jotai、2021 年的 valtio。

这中间比较有趣的也是 vue 的 hooks，2020年追随 react 的脚步（仅仅是思路，其实现及功能完全不同）出现 vue3，而后衍生出 pinia。

而目前的 npm trends 来看，react 的库 还是 redux 一骑绝尘，不过 zustand 势头也比较猛，后起之秀也在逐步渗透中。

有点偏题了，回归正题，接下来将从状态管理的本质入手，对几个典型的库进行简单对比分析。

### vuex & pinia

在 vue 中，由于其组件更新模式固定，所以到目前为止，比较流行的状态管理也仅有 [vuex](https://vuex.vuejs.org/) 及 [pinia](https://pinia.vuejs.org/)。

这两者在其 store 的定义中，就完成了上述基本能力的提供：基础状态、衍生状态、action 的定义。

当然在 vuex 中，还存在一些当时看很好的设计模式，现在看，太冗余的设定：mutation。

同时 vuex 的 store 拆分走的是 module 的模式，而 pinia 走的是多 store 实例的模式。具体的对比可以见 [文章](https://www.vuemastery.com/blog/advantages-of-pinia-vs-vuex/)。

整体来讲，二者都提供了非常显而易见的状态管理的基础能力。能够很好的满足开发者的需求（ps: 怪不得没见 vue 的开发者选状态管理库）。

### redux/zustand

在 react 中，由于 react 更新机制的问题，涌现出很多不同的更新模型的管理库。其中 redux/zustand 保留了 react 最原始的更新机制：不可变性，每次更新都是一份新的 state，这也导致渲染的优化完全依赖开发者定义的 selector。

回归原始，从基础能力上来讲，redux/zustand 均提供了 基础状态、action 的共享，这一点在 store 被定义的时候便已经明确。

不同的是，在 redux 中，将操作基础状态的方法用 reducer 包裹起来。而在 zustand 中，action 和基础状态的定义一致，只是前者通过 function 的形式定义在 store 中。而对于异步 action 的实现，redux 需要配合其中间件使用，而 zustand 由于将 store 的 get, set 直接暴露出来，所以在 action 中，并不在意同步还是异步，调用 set，便会触发更新。从这个角度来讲，zustand 更胜一筹。

从使用者的体验来讲，zustand 概念少，使用更加轻便。

但是二者均没有非常明显的提供 派生状态 的共享。虽然在 react 中，实现状态派生非常简单，只需要在组件中获取全局状态，然后直接计算即可。但是这样带来的问题就是派生状态的计算比较频繁，且如果多个组件都使用的话，需要一套代码写不少次。

对于这个问题，两者均提供了借助第三方库进行解决的方案，[redux](https://redux.js.org/usage/deriving-data-selectors), [zustand](https://github.com/dai-shi/proxy-memoize/tree/61ffff7562fdfbae4859886aa5e81a5a50eae78e#usage-with-zustand)。

但是总体来说，不如 vuex、pinia 中在 store 中直接定义来的快捷简便。

同时，两者底层均是 store 改变，通知所有使用的组件，在 selector 中比较变化来判断组件是否渲染。虽然写 selector 的实际开发体验并不差，因为本身基础状态的获取就需要写 selector。但是其渲染的优化依赖 selector，如果开发者选择不写，那么 store 中一丁点的变动，都会导致组件渲染。

最后，zustand 默认脱离了 provider、context，在组件树外使用起来更加便捷，一些动态的 modal 元素，由于脱离组件树，不能在 redux 中直接获取全局状态，而 zustand 可以，与此同时，一些 im 消息，也可以很好的直接在组件外操作 store。当然还有 store 的拆分等，这一点同 vuex、pinia 一致，zustand 可以通过创建多个 store 来隔离， redux 可能需要配合其他工具使用。

### recoil/jotai

这两者相比于其他库，最大的区别在于原子性。所以单独拿出来。

不过先抛开原子性不谈，还是谈谈其作为状态管理的本质，基础能力的提供。

在 recoil/jotai 中，均提供了 atom 作为基础状态的管理，也提供了 selector(jotai 没有这个概念，通过 read-atom 的方式实现) 这样的方案进行派生状态的管理。

但是这两者，没有提供非常显而易见的 action 的共享。这就导致很多新同学在使用的时候会觉得不适，一些通用的 action 往往需要自定义 hooks 去实现，或者 selector 别扭的拼好（不是很好的模式，对于接口请求还好，交互触发类的行为就非常别扭了）。

但是这两个库提供了 suspense 的能力，如果定义异步的派生状态，那么配合 react 的 suspense，可以很方便的管理 loading 态。这一点确实非常好用，毕竟如果自己写，每个异步请求都做 loading 态 error 态，管理的话，其实至少需要维护 3 个变量： error, data, loading。笔者依稀记得以前写异步的时候，一个请求就需要至少搭配 loading 态， error 信息，虽然也是模版套路，但是确实有代码冗余。

### valtio

如果说， recoil/jotai 都还是维持了 react 不可变性的理念，那么 valtio 直接打破了这一理念，所以对于习惯了 react 思维模式的开发者手上，会存在一定的心智模型的切换。

还是回归状态管理的本质，valtio 提供了基础状态的共享，派生属性也天然支持，而 操作状态的方法也支持，可以看出，其天然满足了一个状态库的基本诉求。

但是由于其不可变性的改变，在开发时需要进行心智模型的切换，特别是局部状态和全局状态同时使用时，会比较麻烦。虽然可变性操作复杂状态确实方便，但是架不住局部状态的存在，来回切换很容易出现 bug。

### 小结

下面是一张总表，涵盖了大部分能力的比较

|  库      | 基础状态   | 派生状态 | action支持      | 更新 API   | 渲染优化   | suspense | context   | 多store|
|  :----:  | :----:     | :----:   | :----:          | :----:     | :----:     | :----:   | :----:    | :---:  |
| redux    | selector   | 无内置   | 默认仅同步      | 不可变     | 选择器     | 不支持   | 依赖      | 无内置 |
| zustand  | selector   | 无内置   | 支持            | 不可变     | 选择器     | 不支持   | 可选      | 多实例 |
| recoil/jotai | atom   | selector/readatom | 不显而易见 | 不可变 | 自动       | 支持     | 依赖/可选 | 无中心 |
| valtio  | useSnapShot | get      | 支持            | 可变       | proxy 自动 | 支持     | 可选      | 多实例 |
| vuex/pinia  | state   | getter   | mutation/action | 可变       |   ---      | -----    | ---       | module/多实例|

ps: 上述无内置表示需要和第三方库结合。不显而易见便是官方文档没有注明最佳实践，需要使用者自行体会。

另：实际笔者真正项目中使用过的库有: vuex/zustand/recoil，其他的库仅仅是参考了官方文档和部分源码，所描述的可能有所纰漏，有问题可以指出交流哈。

当然除了这些库，其实开发者实现的库非常多。比较有趣的有 [resso](https://github.com/nanxiaobei/resso)，[flooks](https://github.com/nanxiaobei/flooks) 等等，其思路比较有趣。

## 状态库选择

### 笔者的抉择

从 [npm trends](https://npmtrends.com/jotai-vs-mobx-vs-recoil-vs-redux-vs-valtio-vs-zustand) 上看，redux 一骑绝尘，差了几个量级但是确实比其他几个库领先的就是 zustand 了，大众用脚投票是一方面，从整体的诉求上来讲，可能也是 zustand 被广泛使用的原因。

从 api 上来讲，zustand 轻便，上手成本基本为 0。

而 redux，上手成本太高，引入 redux 不说，要想正常使用，还需要引入一堆中间件，对于历史项目还行，新项目从头再整，每一次都是比较头疼的体验。

笔者在做决定的时候，还是回归到业务本质，我们开发人员，在实际业务中，需要状态管理的本质就 3个，基础状态、派生状态、action 的共享。

在 action 共享中，zustand 随便支持异步，胜过 redux。

而在 派生状态 中，两者均默认不支持，需要配合第三方库使用，如果想要开箱即用，那么都需要改造。

除了基础能力外，zustand 默认不依赖 provider 的特点也非常友好，这意味着，可以在一些组件树外挂载的组件，方便的使用全局数据。当然 redux 也可以通过 store 实例自行操作，只是稍微看起来奇怪而已。

同时，zustand 数据的隔离可以通过多 store 隔离，使用上更加便捷。

### 简介 zustand 原理

其实类 redux 的状态库，原理都是一致的，useStore 时增加监听器，set 时触发所有监听器，通过 selector 判断是否进行组件渲染。废话就不说了，毕竟 zustand 代码去除类型声明也就不到 100行。再次简化后的代码也就 40 来行，如下：

```js
import { useState, useEffect } from 'react';

const create = (createStore) => {
  // 存放状态
  let state = {};
  // 存放监听器
  const listeners = new Set();
  
  // 订阅改变
  const subscribe = (listener) => {
    listeners.add(listener);
    return () => listeners.delete(listener);
  };
  
  // 获取状态
  const getState = () => state;
  
  // 设置状态
  const setState = (partial) => {
    let nextState = partial;
    if (typeof partial === 'function') {
      nextState = partial(state);
    }
    const previousState = state;

    state = Object.assign({}, state, nextState);
    
    // 触发所有监听器
    listeners.forEach((listener) => listener(state, previousState));
  };

  const destroy = () => listeners.clear();

  const api = { setState, getState, destroy, subscribe };

  state = createStore(setState, getState, api);

  const useStore = (selector, equalityFn = Object.is) => {
    // 为了方便使用了 forceUpdate，实际内部使用了 react 提供的 use-sync-external-store 用来解决 react18 并发的撕裂问题
    const [, forceUpdate] = useState(0);
    useEffect(
      () =>
        subscribe((curState, nextState) => {
          // selector 判断更改前后是否一致，不一致触发更新。
          if (!equalityFn(selector(curState), selector(nextState))) {
            forceUpdate((pre) => pre + 1);
          }
        }),
      []
    );

    return selector(state);
  };
  
  Object.assign(useStore, api);

  return useStore;
};

export default create;
```

其核心就是内部维护一个 listeners 数组，每次 useStore 就进行监听，每次 setState 就是触发所有的 listener。listener 中通过 selector 进行比较，不一致强制渲染组件。

当然，虽然现在说起来这么简单，但是其实笔者一开始看到没有 Provider 的时候还是一惊(内心 os：牛啊 ，怎么做的，赶紧让我看看代码压压惊？)。

这里就不得不提 redux 为什么要使用 provider 了（其实也可以轻易不需要），众周所知，provider 的 value 一变化，所有使用到 context 的都会更新（这也是大部分开发者不自己使用 useContext 来做状态管理的原因，要处理重复渲染），但是 redux 中，其实 provider 的 value 就是一个 store 单例，每次 dispatch 并不会引起单例变化，而是触发在 useSelector 时订阅的监听器，本质同 zustand 是一样的，只是 useSelector 是从 context 上获取到的 state 而已。所以 redux 依赖 context，仅仅也只是从 context 上获取 state，而不是借助 context 触发渲染，所以要去除 provider 也很简单，只是需要动动 api 而已，但这就是使用者众多的库不能动的地方了。

当然在 ssr 场景下，由于不同请求间需要隔离 store，就需要使用 context 进行隔离。

### zustand 的不足

zustand 的 api 非常简洁，create, useStore 基本上就能够覆盖日常使用。也无需考虑 reducer, 同步异步 action，context，组件树内组件树外等。而且 set 也采用的是不可变的写法，同日常的 setState 用起来高度相似。可以说在简洁程度上好到爆，非常匹配他的 slogan: Bear necessities for state management in React。仅包含了 react 状态管理的必须品。

当然，配合其官方的中间件: persist、devtools，使用起来会更加称心如意。

但是细数到实践，其实会发现有一些不好使的默认设定：

#### 写一个和 setState 一样的功能，需要 n 行代码。

思考如下场景，就是代码中原本的一个 ```[count, setCount] = useState(0)```。想要把 count 和 setCount 都共享到 store 中，需要写上如下代码：

```js
// 定义全局的 count, setCount
const useStore = create((set, get) =>({
    count: 0,
    setCount: (f) => {
        if (typeof f === 'function') {
            set({
                count: f(get().count),
            })
        } else {
            set({
                count: f,
            })
        }
    }
}));

// 组件中使用 
const App = () => {
    const [count, setCount] = useStore((state) => [state.count, state.setCount]);
    
    return (
    /* some render */
    )
}
```

可以看出来，为了把 setCount 共享到 store 中，需要进行入参是否是函数的判断，虽说并不麻烦，而且也能抽象成高阶函数包裹下，但是确实代码多了一些。

与此同时，selector: ```(state) => [state.count, state.setCount]``` 的写法，也会导致组件会在 state 每次有变更的时候，进行渲染，这是因为 selector 的默认比较函数是值比较，可以通过加入 shallow 浅比较来避免，不麻烦，就是不好做到开箱即用。

#### 状态获取不够简洁

在 create 函数中，状态的获取就使用 get 就能拿到所有状态，但是使用起来就是不够简洁。开发者往往习惯通过以下的方式获取：

```js
const useStore = create((set, get) => ({
    count: 0,
    reportCount: () => {
        // 原本 zustand 可以通过如下方式取值
        const count = get().count;
        const { count } = get();
        // 但是可能更习惯这样的方式
        // 通过 string 获取
        const count = get('count');
        // 通过 selector 获取
        const count = get((state) => state.count);
    }
}))
```

#### 对 js 代码的类型提示不够友好

zustand 为了类型提示做了很多优化，因为他本身的类型声明中，存在[鸡生蛋、蛋生鸡](https://github.com/pmndrs/zustand/blob/main/docs/guides/typescript.md#basic-usage)的问题，可以说是为了 typescript 的类型提示，做了很多工作。

但是带来的副作用就是，js 里面的类型提示没了。

其本质，就是如下的类型推断失效了：

```ts
// create 抽象一下，就是如下定义
type Create = <T>(createFn: (get: () => T) => T) => T

// 再简化一下，核心在于此处
// createFn: (get: () => T) => T
```

简单来说，就是因为这个 T 的推断，预期是希望根据传入的 createFn 的返回值来自动推断 T，但是由于在 get 中，又需要返回 T，导致 TS 无法正确从 createFn 的返回值中推断 T。说起来就是 createFn 执行后产生 T，但是 createFn 执行中你又用了 T，那这就是经典的死循环了，目前没解。

所以 zustand 在 ts 中对这个问题进行了处理，但是副作用就是对于 js 代码，你写 selector，编辑器都无法给你提供 state.xxx 的提示。

ps： 这里，细心的小伙伴其实不难发现，之所以 js 没有类型提示，就是因为创建 store 的代码中：  ```create((set, get) => ({}))``` 里面，get, set的 类型中存在对泛型的引用，要想在 js 下，有好的类型提示，就是不通过 set, get，而是通过返回的 ```useStore.getState, useStore.setState``` 操作状态，即可有类型提示。这里经过笔者思考，zustand 之所以这样设计，一方面是使用体验，如果代码中通过  ```useStore.getState, useStore.setState``` 去获取和更新状态，多半觉得有点奇怪，看起来像鸡生蛋蛋生鸡（虽然实际并不奇怪，也并不是蛋鸡问题）。另一方面就是中间件的设计需要访问到 set, get，其实不过就是修改 set, get, api 函数来完成中间件的能力。这里有些偏题，也有点悖论，不过就是取舍了，不再过多介绍，有兴趣交流的同学可以私信或者评论区交流哈。

#### 计算属性的支持不够直白

前文已经提过，zustand 没有提供非常直接的写计算属性的方式，虽然在 react中，计算属性不过就是直接在组件中计算一下，但是对于状态库来讲，这是个不优雅的做法，尤其当计算属性的计算比较重的时候(虽然不多，但是得有)，以及这个计算属性在多处都用到的场景下。

zustand 推荐的做法比较隐秘，在 proxy-memorize 中有提及：[proxy-memorize with zustand](https://github.com/dai-shi/proxy-memoize/tree/61ffff7562fdfbae4859886aa5e81a5a50eae78e#usage-with-zustand)。

本质是通过 proxy 监听计算属性依赖的 state 值，当 setState 时，通过比较新的 state 中，依赖的属性是否有变化来决定 selector 是否执行，从而减少了不必要的执行和多次写同一份代码。但是整体不够直白。

#### suspense 没有内置, 新一代 ssr 方案不好支持

在 react 18的场景下，suspense 的特性可谓是给开发者提供了非常便利的管理 loading 态的处理逻辑，笔者依稀还记得以前写类 redux 的时候，每一个 fetch 配套 3 个属性： data, error，status 的场景。

而在其他的状态库中，recoil/jotai/valtio 均出厂带了 suspense 的支持，而且 suspense 在 新一代的 SSR 方案：[renderToPipeableStream](https://beta.reactjs.org/reference/react-dom/server/renderToPipeableStream#) 中也扮演了极其重要的角色。不同于以往的 renderToString 注水方案，新的 stream 方案，对状态管理库的配合提出了新的要求。

#### devtools 中间件配合起来使用不适

zustand 的 set 有第二个参数，表示是否对全部状态进行覆盖，这一点使用下来笔者发现不仅没有使用场景，反而容易犯错误。

而为了配合 devtools 给 set 的行为加上名称，就需要 set(xxx, false, '一些描述')，可以说是非常不便了。

## 成熟的状态管理库应该具备的能力

前文描述了数据状态管理的本质，能够在组件树中任何位置访问到：

1. 基础状态的读取
2. 派生状态的读取
3. 操作基础状态的公共方法的获取

本章将结合上一节对 zustand 劣势分析的讨论 成熟的状态管理库还应该具备哪些能力：

1. 数据持久化。这一点相信开发的同学非常受用。开发中经常需要在本地缓存一些设置等，此时状态库提供一个配置能解决的事情，相信即便给一个 useLocalstorage 这样的 hook 都不想用。

2. devtools。这一点不多说，谁还在用 console 看日志（相信很多小伙伴都是，笔者其实也是，但是复杂之后 console 出来一堆真的难顶，更别提使用 valtio 的小伙伴了，打印出来个 proxy 怎么看？ps: valtio 未经考证仅做调侃）。

3. SSR 的支持。这一点是由于服务端渲染时需要隔离多个请求的 store 的干扰，所以在以往的 renderToString 场景下，只需要提供一个 context 即可。但是在推出 renderToPipeableStream 后，笔者认为，还应该提供 内置的 suspense 以及配套的分批次注水能力。

4. 完善的 ts 支持，同时在 js 中也有非常好的代码提示(毕竟有些上线一次就不改的项目也不少，给这些项目写 ts 就十分浪费了)。

5. 其他，暂时想不到了，笔者的业务场景下，能够想到的就是这四条，有想法的小伙伴可以提示下我哈。

## sustand，这就是笔者最后设计出来的产物

针对 zustand 在实际使用过程中存在的一些不便，以及一个成熟的状态库应该具备的能力的缺失两个方面入手，笔者对 zustand 进行了较大的改造，最终产出了 [sustand](https://github.com/yuzai/sustand)

### 使用中的不便的改造

#### 状态获取不够简洁

如上文所述，在 zustand 中获取状态，只能通过 ```get()``` 拿到所有状态后解构，而实际中往往更习惯 ```get('key')``` 或者 ```get((state) => state.key)``` 的形式，这一点支持起来也非常方便，写一个简单的中间件即可，如下：

```js
const getMiddware = (fn) => (set, get, api) => {
    const originGetState = api.getState;
    api.getState = (param) => {
        if (typeof param === 'function') {
            return param(originGetState());
        }
        if (typeof param === 'string') {
            return originGetState()[param];
        }
        return originGetState();
    }
    return fn(set, api.getState, api);
}
```

如此，便可以解决上述的状态获取不够简洁的问题。

#### devtools 中间件配合起来使用不适

这是因为 zustand 的 set 的第二个参数表示是否覆盖整个 store 导致的，从而 devtools 中想要显示 message，就必须借助第三个参数。而实践下来，覆盖 store 的操作实在是没有应用场景，故可以开发一个 set 中间件修改 zustand 的默认行为，把参数调整下顺序即可（在本库中，直接强制 false，因为笔者实在没有看到覆盖整个 store 的场景）。

```js
const setMiddleware = (func) => (_, get, api) => {
    const originSetState = api.setState;

    // eslint-disable-next-line
    api.setState = (partial, desc: string) => {
        return originSetState(partial, false, desc);
    };

    const states = func(api.setState, get, api);

    return states;
};
```

如此，在和 devtools 搭配使用时，只需要传递第二个参数作为 message 信息，即可在 redux-devtools 的面板中展示相关的信息，从而解决使用上的不适。

#### js 的类型支持

前文已经分析过，在 zustand 中，虽然为 ts 类型做了非常好的支持，但是这会导致使用 js 的情况下，类型推断失败，而其根源就在于 set 和 get 中对泛型的过早获取，要解决也非常简单，有两种方案：

第一种方案：针对 js 的场景下，修改 get 和 set 的类型声明，去除对泛型的引用，如此，编辑器便可以直接根据 create 函数返回的类型进行推断，从而 useStore 中可以获取 state 的推断。当然，代价就是在 store 定义时，set 和 get 中的类型推断失效

第二种方案，便是在 create 函数中，不使用 set 和 get ，遇到需要使用的地方，通过 useStore.get 和 useStore.set 进行，此时，可以获得完整的类型体验，示例如下：

```js
import { create } from 'zustand';

const useStore = create(() => ({
    a: 1,
    b: 2,
    c: () => {
        const res = useStore.getState();
        useStore.setState((state) => {})
    }
}));

useStore((state) => state.a);
```

通过上述的方式，即可在 js 下获得非常好的类型提示。

在本方案中，均进行了处理，一方面，通过重载使得在 js 场景下, set, get 没有提前使用泛型，另一方面，返回值不仅仅是 useStore，把 store 也做了返回，可能使用起来不会那么奇怪，毕竟 use 开头，是 hooks 的标志。改动点大致如下：

```ts
/** 为 js 准备的类型声明 */
// 在 set 和 get 中没有获取泛型 t
export type StateCreatorJs<T> = (set: SetState<any>, get: GetState<any>, api: StoreApi<any>) => T;

export type StateCreator<T extends {}, S = T> =
(set: SetState<Convert<S & T>>, get: GetState<Convert<S & T>>, api: StoreApi<Convert<S & T>>) => S;

/** 创建 store */
export type Create = {
    // 重载，传递参数时，使用 js 的类型
    <T extends {}>(create: StateCreatorJs<T>, opts?: CreateOptions<any>): {
        useStore: UseStore<T>,
        useStoreSuspense: UseStoreSuspense<T>,
        useStoreLoadable: UseStoreLoadable<T>,
        store: StoreApi<Convert<T>>
    },
    // 无参数时，使用 ts 的类型声明
    <T extends {}>():
    (create: StateCreator<T>, opts?: CreateOptions<Convert<T>>) => {
        useStore: UseStore<T>,
        useStoreSuspense: UseStoreSuspense<T>,
        useStoreLoadable: UseStoreLoadable<T>,
        store: StoreApi<Convert<T>>
    },
};
```

#### 更加贴近 useState 的用法

zustand 的简便及易扩展性是有目共睹的，但是其扩展性也仅限于对 store 的 getState, setSgtate 等 api 的修改，想要修改 useStore 的行为，单靠中间件是做不到的，需要覆盖整个 create 函数。

想要做到 useState 一样的体验，可以通过提供 useStore('key') 的方式，返回 [state, setState] 的方法。

修改伪代码如下：

```js
import { create as zustandCreate } from 'zustand';
import shallow from 'zustand/shallow';

const create = (fn) => {
    const lazySetActions = {};
    // 省去上文几个中间件的定义
    const useZustandStore = zustandCreate(setMiddware(getMiddware(fn)));
    
    const store = {
        getState: useZustandStore.getState,
        setState: useZustandStore.setState,
        sunscribe: useZustandStore.subscribe,
    }

    const useStore = (f, equalityFn) => {
        let fn = f;
        let equality = equalityFn || shallow;
        if (typeof f === 'string') {
            const state = store.getState(f);
            if (!lazySetActions[f]) {
                lazySetActions[f] = (v) => {
                    if (typeof v === 'function') {
                        store.setState({
                            [f]: v(state),
                        });
                    } else {
                        store.setState({
                            [f]: v,
                        })
                    }
                }
            }
            const setState = lazySetActions[f];
            fn = () => [state, setState];
            // 修改比较函数为浅比较第一个参数
            equality = equalityFn || (a, b) => shallow(a[0], b[0]);
        }
        return useZustandStore(fn, equality);
    }
    
    return {
        useStore,
        store,
    }
}
```

如此这样，即可在代码中通过 useStore('key') 得方式获取 state 和 setState 的方法。其内部维护了一个惰性创建的 setState 方法的合集。

#### 默认的比较函数替换为浅比较

在笔者实践得过程中，发现 ```useStore(state => [state.a, state.b])``` 这样的写法最为常见，这样写，在默认的 zustand 情况下，任何 state 的改变，都会触发当前组件的渲染，因为 ```state => [state.a, state.b]``` 这样返回的始终是一个新数组，故笔者默认将比较函数替换为了浅比较，来减少实践中冗余的代码。

### 缺失能力的弥补

#### 衍生状态/计算属性

在 react 开发中，不论是 redux, 或者 zustand，往往都会在 store 中定义一些状态，这里举个最近业务中的例子，假设定义了两个属性，users 和 currentUser，这两个属性分别来自两个接口。伪代码如下：

```js
const { useStore } = create((set) => ({
    users: [],
    currentUser: {},
    getUsers: () => {
        fetch().then(res => set({
            users: res,
        }))
    },
    getCurrentUser: () => {
        fetch().then(res => set({
            currentUser: res,
        }))
    },
}));
```

最初，我们在一个组件中，需要判断当前用户 currentUser 是否在 users 中，我们在该组件中可以很轻易的写出如下代码:

```js
const App = () => {
    const [users, currentUser] = useStore((state) => [state.users, state.currentUser]);
    
    const joined = useMemo(() => {
        return users.filter((user) => user.userId === currentUser.userId).length > 0;
    }, [users, currentUser]);
    
    // 一些渲染逻辑
}
```

后来，我们在另一个组件中也需要这个 joined 属性，那么此时，当然可以直接复制粘贴过去，但是，如果用到的组件不止这一两处呢？
当然，我们也可以把 joined 属性写在 store 中，但是由于 getUsers 和 getCurrentUsers 会改变，所以代码需要写成如下样子：

```js
const { useStore } = create((set) => ({
    users: [],
    currentUser: {},
    joined: false,
    getUsers: () => {
        fetch().then(res => set((state) => ({
            users: res,
            joined: res.filter(user => user.userId === state.currentUser.userId).length > 0,
        })))
    },
    getCurrentUser: () => {
        fetch().then(res => set((state) => ({
            currentUser: res,
            joined: state.users.filter(user => user.userId === res.userId).length > 0,
        })))
    },
}));
```

可以看出来，每一个更改 currentUser 和 users 的地方，都需要更新 joined，当然，现在的代码也不是不能接受，但是如果修改 currentUser 和 users 的地方很多呢？每处都这么写显然是不合适的。

在没有用状态管理之前，我们可以通过 useMemo 计算一次，然后当作 props 进行传递，这样，虽然传递麻烦，但是计算只有一次。而状态管理用了之后，虽然不需要传递了，但是如果把该状态定义在全局，那么就需要在依赖的状态变更时，同时更新该状态，或者把状态定义在组件内，就需要各个组件都计算一遍。

当然，上述代码还有一种写法，就是在 selector 中做派生状态，伪代码如下：

```js
const App = () => {
    const joined = useStore((state) => {
        const { users, currentUser } = state;
        return users.filter((user) => user.userId === currentUser.userId).length > 0;
    });
    
    // 一些渲染逻辑
}
```

在 zustand 中，userStore 中传入的函数成为 selector，可以发现，selector 其实是个纯函数，那么多个组件使用的话，其实是可以把 selector 提取出来的单独放在一个文件中的，多个组件使用无非就是多次使用这个 selector 即可。

这样做，确实能够提升代码的维护性，但是性能上是不靠谱的，selector 会被执行多次。

为了避免这个问题，zustand 给出了 [proxy-memoize](https://github.com/dai-shi/proxy-memoize) 的写法。大致如下：

```js
import { memoize } from 'proxy-memoize';

const joinedSelector = memorize((state) => {
    const { users, currentUser } = state;
    return users.filter((user) => user.userId === currentUser.userId).length > 0;
});

const App = () => {
    const joined = useStore(joinedSelector);
    // 一些渲染逻辑
}
```

其内部是通过 proxy 收集用到的 state 属性，下次执行时，这些属性如果没有变化，那么就返回上一次的状态。

通过这种方案，能够达到一个比较理想的效果，计算代码只写一次，真正的计算逻辑也只执行一次，但是状态的比较还是被执行了多次, memorize 本身也有开销。

笔者最终开发的 sustand 也是基于这个方案做了封装加改进，最终的写法如下：

```js
import create, { compute } from 'zustand-with-suspense';

const { useStore } = create(() => ({
    counta: 1,
    countb: 2,
    sumAB: compute((state) => state.a + state.b),
}));

const App = () => {
    const sumAB = useStore('sumAB');
    // 也支持如下获取：
    // const sumAB = useStore((state) => state.sumAB);
}
```

相较于 proxy-memoize 的方案，笔者做了如下的改进：

在 selector 的定义上，隐去了 memorize 的依赖，同时计算属性 selector 的执行时机是 state 发生改变之后，直接计算所有的衍生属性，useStore 的地方仅仅是获取该值而已。此方案带来的好处就是，每一个衍生属性的 selector 仅执行一次，减少了 memorize 的开销，同时在代码上也更加工整，不必再额外的抽离 selector。

#### suspense 属性的支持

这个特性几乎可以说是提升开发体验必备的特性了，笔者最初也是为了支持 suspense 的特性才决定进行本库的开发，先看看笔者最终的改造结果：

```js
import create, { suspense } from 'zustand-with-suspense';

const { useStore } = create(() => {
    suspenseValue: suspense((args) => {
        return fetch('xxx') or anyPromsie,
    })
});

const App = () => {
    const { data, status, refresh } = useStore('suspenseValue');
}
```

以上便是一个最简洁的 suspense 属性的用法，当然，App 组件外需要自行包裹 Suspense, errorBoundory 来处理 loading 态和 error 态。

内部的实现思路如下：

1. 定义时搜集哪些 key 是 suspense 属性
2. useStore 时判断是否命中 suspense 属性
3. 如果是 suspense 属性，则执行 action，并根据 promise 得状态抛出 promise 或者 error，或者返回数据。

当然，说起来容易，但是开发过类似库的同学应该明白，里面还有不少门道，比如：

1. 不同的入参，是不是应该缓存一下结果？
2. refresh 的时候，是不是可以提供下 loadable 的场景？
3. 是不是应该提供一个 selector，在数据变更时，自动同步下属性？

嗯，这些，都考虑进去了，具体可以通过使用文档查看。当然，相关代码不多，但是要考虑得边界条件比较多，不过不是本文的重点，不再赘述。

#### renderToPipeableStream 的开箱支持

终于到了最后，但是也是最重要的一个特性，对 react 18 [renderToPipeableStream](https://react.dev/reference/react-dom/server/renderToPipeableStream#rendertopipeablestream) 的开箱支持。

这一 ssr 方案的优势在于分批次注水，用户体验好了。而开发也不需要把接口请求放到统一的地方了，只需要抛出 promise 就行了。不过问题就来了，分批次注水，那我的全局 store 怎么办？

这就对状态管理提出了更高的要求：支持分批次注水。

当然，原理还是一样，只不过以前是一把梭把状态全部注入 store，现在只不过是一点一点注入。这里面得问题就在于：

以前一把梭注入，开发者可以很方便的写出注水代码：

```js
const useStore = create(() => ({
    a: 1,
    ...(client ? window.__somedata__from__server__ : {}),
}))
```

其注水的过程都由开发者控制，其本质上状态管理库并无感知。

而分批次注水，单靠开发者就显得无能无力，需要库配合支持，本库的方案如下:

```jsx
const App = () => {
    const { data, refresh, loadScript } = useStore('suspenseValue');
    
    return (
        <div>
            {loadScript}
        </div>
    )
};
```

开发者仅仅需要把 loadScript 渲染出来即可。

loadScript 的实现大致如下：

```js
const loadScript = createElement('div', {
            dangerouslySetInnerHTML: {
                __html: `<script>
                    window.__ssrstreamingdata__ = window.__ssrstreamingdata__ || {};
                    window.__ssrstreamingdata__[JSON.stringify(${cacheKey})] = ${JSON.stringify({
    data,
    status,
    fromServer: true,
})}
                </script>`
            }
        });
```

本质就是插入一段 script ，在这段 script 中，向window 上注入 suspense 数据，而在 useStore 时，只需根据这段数据的有无，来进行请求的发送与否即可。（当然，边界场景也很多，不赘述）。

## 结尾

总算是到了最后，为了这个状态库，最初的方案真的是苦思冥想，最后的实现时也是各种边界条件要考虑，这才产出了这样的一个令我觉得满意的状态管理库。

在修复了笔者在使用 zustand 的各种不便的基础上，增加了衍生属性、suspense、renderToPipableStream 的开箱支持。其中 renderToPipableStream ️应该是头一家(if 不是，请轻喷)，心满意足了已经。

当然，这篇文章也是前前后后快2个月，期间写着写着，就发现了一些不足的地方，最终才产出了这个状态管理库。

希望能够给大家带来一点帮助吧。欢迎一起讨论哈。

最后，能看到这里的也是很不容易了，库的代码也放在了 [github](https://github.com/yuzai/sustand)，有比较详细的使用方法，欢迎探讨交流小星星哈。
