---
title: 100行代码盘一个react数据流管理工具
date: 2021-10-27 18:04
categories: 原创
---

## 前言

在我们团队目前的业务中，随着移动端h5页面复杂度增加以及需要长期维护的项目增多，初期选择多页无数据管理的一些弊端就暴露出来。页面内多个轮询，消息监听、各种优化交互的loading态、倒计时，导致一个h5中，往往有几十个核心的状态，组件嵌套的层级也较多。代码往往也是开头十几个useState的定义，而这些state通过组件一层层传递下去，让后来的维护者，可以说是如履薄冰。

而在组内推行数据流管理时，大家往往会觉得，redux太重，而mobx、基于rxjs的状态流管理等工具又有学习成本，就很难整。

但是从维护角度来看，一次性的活动页面也就算了，多次反复修改的页面，后期维护成本非常高。

所以，花了点时间，用100行代码写了一个类redux的、生产环境完全可用的数据流管理工具，暂且称为 tinysm, 也就是tiny state manage的缩写。本文便是介绍其实现过程及用法。

<!-- more -->

## 目标

预期的使用方法类似redux， 暴露的api有：

```js
export {
    createStore, // 创建store, 传入state, reducer
    ContextProvider, // react的contex provider组件
    useSelector, // 获取state中的部分数据
    useDispatch, // 获取触发reducer的方法
    shadowEqual, // 浅比较方法
}

```

使用方法的伪代码如下：

```jsx
// app.js
import { ContextProvider, createStore } from 'tinysm';
import Todo from './Todo';
import Counter from './Counter';

// 定义初始state
const initialState = {
  count: 1,
  todos: []
};

// 定义更新state的reducer
const reducer = (state, action) => {
  switch(action.type) {
    case 'ADD_COUNT': return {
      ...state,
      count: state.count + 1,
    }
    case 'ADD_TODO': return {
      ...state,
      todos: [...todos, action.payload],
    }
    default: return {
      ...state,
    }
  }
};

const store = createStore(initialState, reducer);

export const App = () => (
  <ContextProvider value={store}>
      <>
    		<Counter />
    		<Todo />
  		</>
  </ContextProvider>
)

// Counter.js
import { useSelect, useDispatch } from 'tinysm';

export const Counter = React.memo(() => {
  const count = useSelect(state => state.count);
  const dispatch = useDispatch();
  const handleClick = useCallback(() => {
    dispatch({
      type: 'ADD_COUNT',
    })
  }, [dispatch]);
  
  return (
    <div>
      <p>{count}</p>
      <button onclick={handleClick}>add count</button>
    </div>
  )
});

// Todo.js
import { useSelect, useDispatch } from 'tinysm';

export const Todo = React.memo(() => {
  const todos = useSelect(state => state.todos);
  const dispatch = useDispatch();
  const handleClick = useCallback(() => {
    dispatch({
      type: 'ADD_TODO',
      payload: 'test'
    })
  }, [dispatch]);
  
  return (
    <div>
      {
        todos.map((v, index) => (
          <li key={index}>{v}</li>
        ))
      }
      <button onclick={handleClick}>add todo test</button>
    </div>
  )
});
```

上述代码中，使用者定义好初始state， reducer，通过`createStore`创建store, 顶层父组件通过`ContextProvider`包裹，将store注入context，子组件通过`useSelector`获取store中的数据，通过`useDispatch`获取触发reducer的方法。

通过这样的方式，开发者一方面不需要将状态及操作状态的方法通过props一层层传递，另一方面，state及操作state的方法集中管理在一起，非常便于后期维护和修改。

## 具体实现

本人在探索过程中，实现了几个版本，也在网上查了不少文章以及redux的代码，下述的这几个版本间是有递进关系的，按照顺序来看更容易理清思路。

### v0-极简版

按照react正常的开发思路，父组件的state以及修改state的方法通过props传递给子组件，而孙组件可能也会需要这些state以及操作这些state的方法，如此往复，就导致一个参数一层一层被传递。对于嵌套层级较深的情况，很麻烦不说，传递过程中还容易出错。

react本身为这种情况提供了方案，那就是context，核心就三个api:

1. createContext
2. Context.Provider
3. useContext

关于这三个api此处不做介绍，具体查看[官网文档Context章节](https://react.docschina.org/docs/context.html)即可。

对上述三个api进行包装简化，很容易写出一个非常简单的代码如下：

```jsx
// tinysm.js

import { createContext, useContext } from 'react';

const Context = createContext();

const ContextProvider = Context.Provider;

const useSelector = (selector) => {
  const store = useContext(Context);

  return selector(store.state);
};

const useDispatch = () => {
  const store = useContext(Context);

  return store.dispatch;
};

export { ContextProvider, useSelector, useDispatch };

```

在app.js中借助`useReducer`将数据注入ContextProvider中，如下：

```jsx
import React, { useReducer } from 'react';
import { ContextProvider } from './tinysm';
import Todo from './Todo';
import Counter from './Counter';
import './style.css';

const initialState = {
  todos: [1, 2, 3, 4],
  count: 0,
};

const reducer = (state, action) => {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [...state.todos, action.payload],
      };
    case 'ADD_COUNT':
      return {
        ...state,
        count: state.count + 1,
      };
    default:
      return {
        ...state,
      };
  }
};

export default function App() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <ContextProvider
      value={{
        state,
        dispatch,
      }}
    >
      <Todo />
      <Counter />
    </ContextProvider>
  );
}

```

如此，在Todo, Counter中即可通过useSelector和useDipatch获取全局的state和修改state的方法。

完整的demo在[这里](https://stackblitz.com/edit/tinysmv0?file=src%2FApp.js)。

在这个版本中，没有实现createStore，而是在app.js中通过useReducer作为其替代品。是因为useContext的机制会导致组件不更新，官方文档指出:

`任何使用了useContext的组件，在Context.Provider的value发生改变时，就会触发更新`

试想，如果Context.Provider中传入了createStore的返回值，那么value其实永远不会主动发生改变，就会导致子组件dispatch失效。即便在app.js中通过解构获取store，但是由于dispatch并没有触发组件渲染，解构也就没有执行，context.provider的value值自然也不会更新，就带来了子组件的不更新。这一问题，可以看看接下来的v1版本。

除此之外，这个版本最大的问题就是不必要渲染，也是因为useContext的机制，只要value值变化，所有使用了useContext的组件都会更新。例如在[demo](https://stackblitz.com/edit/tinysmv0?file=src%2FApp.js)中，count组件触发dispatch后，todo组件也触发了渲染，虽说只是一次重复的操作，由于diff的存在，并不会真的触发dom的反复操作，但是不必要的渲染总归不是个好的开始。

### v1-不更新版

```jsx
// tinysm.js
import { createContext, useContext } from 'react';

// 创建context
const Context = createContext();
// 构建provoder
const ContextProvider = Context.Provider;

// 创建store
function createStore(initialState, reducer) {
  // 定义store
  const store = {
    state: initialState,
  }
  
  // 增加dispatch方法，本质就是把state, action传递给reducer执行，而后更新state
  store.dispatch = function(action) {
    this.state = reducer(this.state, action);
  }
  
  // 绑定this
  store.dispatch = store.dispatch.bind(store);
  
  return store;
}

const useSelector = (selector) => {
  const store = useContext(Context);
  return selector(store.state);
}

const useDispatch = () => {
  const store = useContext(Context);
  return store.dispatch;
}

```

在app.js中，可以按照目标中提到的方式进行代码的编写:

```jsx
import React from 'react';
import { ContextProvider, createStore } from './tinysm';
import Todo from './Todo';
import Counter from './Counter';

import './style.css';

const reducer = (state, action) => {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [...state.todos, action.payload],
      };
    case 'ADD_COUNT':
      return {
        ...state,
        count: state.count + 1,
      };
    default:
      return {
        ...state,
      };
  }
};

const initialState = {
  todos: [1, 2, 3, 4],
  count: 0,
};

const store = createStore(initialState, reducer);

export default function App() {
  return (
    <ContextProvider value={store}>
      <Todo />
      <Counter />
    </ContextProvider>
  );
}

```

通过该代码实现的tinysm demo可以查看[demo](https://stackblitz.com/edit/tinysmv1?file=src/App.js)。

相信眼尖的同学，通过代码就能发现，这个dispatch是不能触发更新的，虽然改变了store.state的值，但是页面并不会更新。如果说v0版本是带来了很多不必要的更新，那么v1版本，就是全部不更新。

究其原因，在于dispatch执行之后，没有触发app中store的改变，可能有同学会说，把app.js中`<ContextProvider value={{...store}}>`，这样解构之后，value在app每次渲染的时候，就会更新了。但是问题是，并没有任何人触发app的重新渲染，解构也没用。

当然，这里是可以在dispatch后重新执行`ReactDOM.render(<App />, document.getElementById("root"));`但是这样做的话，也会带来所有使用到useContext的子组件的更新，还是没有解决不必要渲染的问题。还不如人v1版本呢，至少代码短且清晰。

### v2-自动更新版

上述v0版的核心在于借助react Context的特性，来实现参数的跨组件传递，但是由于react的useContext的特点，在ContextProvider的value值改变的时候，触发所有用到useContext组件的不必要渲染。

而v1版本，始终保持value值不变，但是也导致dispatch无法触发更新。

那么我们想要的效果，肯定是按需渲染。使用了state.todos的组件，就在state.todos变化的时候更新，使用了state.count的组件，就在state.count变化的时候更新。

好了，怎么做呢？

本人也是翻看了不少文章，查了redux的源代码发现的，其核心就是：

1. 保持ContextProvider的value值不变，因为如果value变化，那么不必要渲染就必然存在
2. 创建事件中心EventCenter
3. 在useSelector中利用useEffect往事件中心中订阅通知，当收到通知，对比从selector中拿到的state中的数据是否发生改变，如果改变，forceRender，否则，不做任何事情
4. 在dispatch操作完成后，通知所有的事件订阅者

好了，不废话了，上代码：

```jsx
// tinysm.js
import { useContext, createContext, useRef, useEffect, useReducer } from 'react';

const Context = createContext();
const ContextProvider = Context.Provider;

// 非常常见的发布订阅模式
const EventCenter = {
  listensers: [],
  subscribe: function (func) {
    this.listensers.push(func);
  },
  unsubscribe: () => {}, // 一一比较删除即可，不赘述
  notify: function() {
    this.listensers.forEach(v => v()); // 监听器一一执行
  }
}

function createStore(initialState, reducer) {
  const store = {
    state: initialState,
  }
  
  store.dispatch = function(action) {
    this.state = reducer(this.state, action);
    // 通知监听器执行
    EventCenter.notify();
  }
  
  store.dispatch = store.dispatch.bind(store);
  
  return store;
}

const useSelector = (selector, equalFunc = (a, b) => a === b) => {
  const store = useContext(Context);
  // 获取用户需要的state
  const state = selector(store.state);
  // 通过ref存储上一次的state值
  const preState = useRef(state);
  // 触发组件更新
  const [, forceRender] = useReducer(s => s + 1, 0);

  const checkForUpdate = () => {
    // 获取最新的state值
    const curState = selector(store.state);
		
    // 比较上一次的值和当前最新值是否相同
    if (equalFunc(curState, preState.current)) {
        return;
    }
		// 不相同，更新上一次的state值，并触发组件渲染
    preState.current = curState;
    forceRender();
  };

  useEffect(() => {
    // 订阅
    EventCenter.subscribe(checkForUpdate);
    return () => {
       // 取消订阅
       EventCenter.unsubscribe(checkForUpdate);
    }
    // eslint-disable-next-line
  }, [store]);
  return state;
};

const useDispatch = () => {
    const store = useContext(Context);

    return store.dispatch;
};

export {
	ContextProvider,
  createStore,
  useSelector,
  useDispatch,
}
```

上述代码的核心，有三个点

1. useSelector中在useEffect中订阅事件
2. createStore中，dispatch方法在执行完reducer后，发起事件通知所有订阅者执行回调
3. 在回调中，useSelector根据用户传入的selector和equalFunc，判断用户使用到的值在dispatch前后是否发生改变，如果改变，就出发子组件更新，否则不做任何事情。

通过该代码实现的tinysm demo可以查看[demo](https://stackblitz.com/edit/tinysmv2?file=src/App.js)

至此，一个功能比较完备的tinysm即完成了，稍微整理下，它

1. 利用react 的context的特性，实现了参数跨组件传递
2. 保持context.provider组件的value值不变，从而避免不必要渲染
3. 借助发布订阅模式，利用useSelector比较前后组件依赖的state的变化来实现组件按需渲染

嗯，这就是v2版本的所有内容。

但是这个版本是否是功能比较完备了呢？

异步，相信很多同学会说到异步，业务代码中，绝大部分的数据都是通过异步api从后台拉取，等取到数据后，dispatch action，而这一拉取过程，往往不是只有顶层组件需要做的事情，子组件中如果也需要这样的异步操作，那么此时有两个选择，代码重写一遍，or 通过props一层一层传递下去。

显然这两种方案都不优雅。更好的方案就是store支持把异步action传递进来，子组件通过dispatch去触发即可。

### v3-支持异步版

支持异步的初衷，其实是想把异步action放到context中，不需要一层一层传递，从而代码不论是可维护性还是复用性都大大提升。

而所谓的异步action，无非就是执行时，延时执行reducer，那么做一层抽象，一个action需要的参数有：

1. store中当前的state, 可能会根据当前的state做不同的异步操作
2. 触发reducer的方法dispatch，此处为避免误会，效仿vuex改为commit
3. 异步行为需要的参数payload，可能会根据当前的参数执行不同的异步操作

定义一个简单的action:

```jsx
const actions = {
  getATodo: (state, commit, payload) => {
    const fetchSomeTodoFromRemote = (payload) => {
      // your logic to fetch data according to payload
      return new Promise(resolve => {
        setTimeout(() => {
          resolve('remote todo');
        }, 3000);
      })
    }
    fetchSomeTodoFromRemote(payload).then((data) => {
      commit({
        type: 'ADD_TODO',
        payload: data,
      })
    })
  }
}
```

在tinysm中增加如下代码：

```jsx
// 增加参数actions
function createStore(initialState, reducer, actions) {
    const store = {
        state: initialState,
    }
    store.dispatch = function(action) {
        // dispatch优先从action中获取 
        if (this.actions[action.type]) {
            const act = this.actions[action.type];
            if (typeof act === 'function') {
                // commit本质就是触发reducer的方法，也就是dispatch
                act(this.state, this.dispatch, action.payload);
                // act执行后不需要手动触发事件通知，因为在action中
                // 用户自行编写代码选择何时触发reducer，届时便会触发事件通知 
            } else {
                console.error('action is not a function');
            }
        } else {
            // 如果actions中没有找到，就执行reducer
            this.state = reducer(this.state, action);
            EventCenter.notify();
        }
    }
    store.dispatch = store.dispatch.bind(store);
    return store;
}
```

通过该代码实现的tinysm demo可以查看[demo](https://stackblitz.com/edit/tinysmv3?file=src/tinysm.js)。

到了这个版本，功能上基本都实现了，支持异步，支持按需更新子组件，能够覆盖移动端h5中数据管理的场景。在代码中再不不会一层一层传递state和setState了。而且数据集中在一起，可维护性高了非常多。

## 最后

本文相关的代码，除了上述提到的demo，github上也有一份[地址](https://github.com/yuzai/tinysm)，npm也发了个小包[地址](https://www.npmjs.com/package/hooks-tinysm)。

另外，因为去除了很多redux的功能，比如middleware, class支持，modules，以及devtool的支持，所以并不太适合比较大型的web单页应用。尤其是不支持modules，如果是大单页的话，页面之间的state会夹杂在一起，需要靠个人去维护。但是对于移动端多页场景非常适合，我们当前业务面临的问题是，一个h5交互逻辑较为复杂，状态数较多，导致组件拆的很多，嵌套层级深，带来了一些维护上的问题。

同时，因为去除了很多redux的功能，代码非常简洁，100行不到即可实现完整的功能，不乐意装包的话，可以直接[代码](https://github.com/yuzai/tinysm/blob/master/src/tinysm/tinysm.js)考过去直接用，当然，考过去的另一个好处就是有bug可以随时改，哈哈。

其实，写这篇文章的起点，是某天突然好奇redux的useSelector是如何实现的，查阅了不少代码及文章，才有了这篇文章。相信本文对这一点应该解释得非常透彻了。

不过react的数据状态管理方案很多，mobx,  Recoil，以及redux的中间件等等，不在本文的讨论范畴（目前也不熟悉）。

最后，希望本文能够或多或少给耐心阅读到这里的你一点收获，就心满意足啦。



