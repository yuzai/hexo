---
title: 面试官：用过 redux 是吧？要不手写一个？
date: 2023-06-06 18:40
categories: 原创
---

前一段时间一直在面试，因为做过状态管理相关的事情，所以在聊到 redux 时，提到了一行代码实现 redux，所以就有了后续的问题：

能一行代码实现下 redux 吗？ 
能再实现下 redux 的中间件吗？
能说说怎么支持异步吗？
能让普通函数也享受下中间件吗？

由于 redux 也确实是比较久远但是经典的库了，事后我趁此机会也算是好好回顾了一下 redux，也发现了其中的一些优秀设计，故有了本篇文章。

<!--more-->

注：本文不会过多介绍 redux 的使用和 api，而是围绕这次面试的问题，一步步进行完善给出 redux实现, 中间件的实现。并在此基础上进行扩展，用 redux 的中间件思路对普通函数包装，使其洋葱模型也可用在普通函数中。

## 一行代码实现下 redux

关于 redux，现代的论调更多的是：一行代码就能实现的库，redux 却写出了非常玄学的文档。

抛开论调不讨论，真的能一行代码实现吗？

能，但是功能不全。

```js
const createStore = (reducer, state) => ({
    getState: () => state,
    dispatch: (action) => state = reducer(state, action),
})
```

上述就是网上流传的一行代码加入了换行之后的样子。

本质上提供了一个获取 state 的方法，getState，以及修改 state 的方法，dispatch。并在 dispatch 中，调用 reducer 并使用返回值更新 state。

可以说，麻雀虽小，但是确实五脏俱全了。

但是从功能上讲，欠缺了订阅这一必不可少的环节，毕竟要触发 react 或者其他框架更新的话，订阅是必要环节。

不过实现一个订阅也是非常简单的，只是一行代码的话，确实不行，加入了订阅之后的代码如下：

```js
const createStore = (reducer, state) => {
    const listeners = new Set();
    return {
        getState: () => state,
        dispatch: (action) => {
            const preState = state;
            state = reducer(state, action);
            listeners.forEach((listener) => listener(preState, state));
        },
        subscribe: (listener) => {
            listeners.add(listener);
            return () => listeners.delete(listener);
        }
    }
};
```

这里借助了 set 非常轻便的实现了订阅。

## 再实现下中间件？

中间件的思想还是很优秀的，洋葱模型的加入，使得用户可以非常方便的在调用 dispatch 函数前后插入很多的通用代码，这在状态管理中是非常实在的，比如 打日志，持久化缓存等功能上。

### 常规写法，复写 store.dispatch

如果抛开 redux 的实现，其实很多库包括我们日常的代码，都会有类似的实现，就是函数的复写。

比如 vconsole，就会对 全局的 console 进行复写，伪代码如下：

```js
function vConsole() {
    const originConsoleLog = window.console.log;
    window.console.log = (...args) => {
        /* 执行前的钩子 */
        // do something
        // 执行原本 console 的逻辑
        const res = originConsoleLog(...args);
        /* 结束后的钩子 */
        // do something
        
        // 原封不动返回原函数的返回值
        return res;
    }
}
```

一个非常常见的方法，参考这个方法，想要在 dispatch 前后打印日志，可以非常简单的写出如下代码：

```js
const loggerMiddware = (store) => {
    const originDispatch = store.dispatch;
    store.dispatch = (...args) => {
        console.log('before dispatch', store.getState());
        const res = originDispatch(...args);
        console.log('after dispatch', store.getState());
        return res;
    }
}

const store = createStore(() => ({ a: 2 }), { a: 1 });

loggerMiddware(store);

store.dispatch('test');

// 输出如下：
// before dispatch: { a: 1 }
// after dispatch: { a: 2 }
```

与此同时可以再多个打印时间的中间件：

```js
const timeMiddware = (store) => {
    const originDispatch = store.dispatch;
    store.dispatch = (...args) => {
        console.log('before dispatch', new Date().getTime());
        const res = originDispatch(...args);
        console.log('after dispatch', new Date().getTime());
        return res;
    }
}

const store = createStore(() => ({ a: 2 }), { a: 1 });

loggerMiddware(store);

timeMiddware(store);

store.dispatch('test');

// 输出如下：
// before dispatch: 时间戳
// before dispatch: { a: 1 }
// after dispatch: { a: 2 }
// after dispatch: 时间戳
```

此时就能达到洋葱模型的表现。依次从外(后调用的中间件)至内(先调用的中间件)执行，执行到原始的 store.dispatch，再从内(先调用的中间件)至外(后调用的中间件)执行。

这里由于复写的存在，先被调用的中间件，会后执行。可以整一个 applyMiddware 的方法来隐藏这一顺序问题，如下：

```js
const applyMiddware = (store, middwares) => {
    // 逆序执行，从而隐藏执行顺序的问题
    middwares.slice().reverse().forEach((middware) => middware(store));
}

const store = createStore(() => ({ a: 2 }), { a: 1 });

// 先传入的中间件，先执行
applyMiddware(store, [timeMiddware, loggerMiddware])
```

再优化一点，可以把 applyMiddware 这个过程放在 createStore 中去，不过这里不是核心，不再展开。

## 去除复写的不纯

上一部分的实现，其实基本和 [zustand](https://github.com/pmndrs/zustand/tree/main) 的中间件系统一样了，都是通过复写 api 来实现中间件的效果。

这样写没什么问题，但是就是不符合纯函数的优雅，loggerMiddware、timeMiddware，都不是纯函数，因为他们修改了入参中的 dispatch 属性，在 redux 的哲学中，这里很不纯。

想要去除 store.dispatch 的显式复写，只能将这个过程对用户隐藏，因为最终都是要修改的。

可以在 applyMiddware 中进行复写的操作：

```js
const applyMiddware = (store, middwares) => {
    // 逆序执行，从而隐藏执行顺序的问题
    const mids = middwares.slice().reverse();
    let dispatch = store.dispatch;
    // 关键
    mids.forEach((middware) => dispatch = middware(store)(dispatch));
  
    store.dispatch = dispatch;
}
```

本质就是将 dispatch 的赋值过程，不暴露在中间件的定义，而是写在了 applyMiddware 函数中，如此写来，就需要对原来中间件的写法做出修改：

```js
const loggerMiddware = (store) => (dispatch) => (...args) => {
    console.log('before dispatch', store.getState());
    const res = dispatch(...args);
    console.log('after dispatch', store.getState());
    return res;
}

const timeMiddware = (store) => (dispatch) => (...args) => {
    console.log('before dispatch', new Date().getTime());
    const res = dispatch(...args);
    console.log('after dispatch', new Date().getTime());
    return res;
}

const store = createStore(() => ({ a: 2 }), { a: 1 });

// 先传入的中间件，先执行
applyMiddware(store, [timeMiddware, loggerMiddware])
```

这里中间件的写法其实和 redux 的写法就一致了，只是内部实现还不完全相同，留作下一小结讲解。这里要说的是，不知道看到这么连续几次的箭头函数的定义，有没有把一些同学转晕，我第一次看的时候还是很头疼的，怎么这么多次箭头函数的连续定义。

在没有消除 dispatch 的复写时，我们的中间件写起来还是非常简单，而为了消除复写，突然多了几层箭头函数的连续定义，不免觉得头大。这里写在一起对比一下：

```js
// 复写模式下中间件的写法
const loggerMiddware = (store) => {
    const originDispatch = store.dispatch;
    // 关键注释1: 第二个箭头函数
    store.dispatch = (...args) => {
        console.log('before dispatch', store.getState());
        const res = originDispatch(...args);
        console.log('after dispatch', store.getState());
        return res;
    }
}

// 剥离了复写逻辑的写法
// 关键注释2: 三个箭头函数连续定义，但是实质上只比上述多了一层
const loggerMiddware2 = (store) => (dispatch) => (...args) => {
    console.log('before dispatch', store.getState());
    const res = dispatch(...args);
    console.log('after dispatch', store.getState());
    return res;
}
```

仔细分析下后者的写法，其实比原来的模式，看起来多了两层箭头函数，第一层是 `(dispatch) => xxx`，这一层对应 appplyMiddware 中的 `middware(store)(dispatch)`，下一层是 `(...args) => xxx`，对应 appplyMiddware 中的 `dispatch = middware(store)(dispatch)`。但是说到底，这两个箭头函数，是把原本的 dispatch 作为入参，复写的结果作为返回值，在外部通过 `dispatch = middware(store)(dispatch)`，隐藏了原本的复写逻辑。本质上其实只是多了一层箭头函数。第二层的箭头函数，在原本的复写模式下也有，只是看起来没有那么明显罢了。

而之所以看起来难懂，还有一点就是这里巧妙的借助了箭头函数返回值的特性，所以看起来非常简单，实际上，抛除这一特性后，代码和复写模式下差不太多：

```js
// 抛开箭头函数的写法，可能看起来跟复写版本类似一些。
const loggerMiddware2 = function (store) {
    function rewrite(dispatch) {
        return function (...args) {
            console.log('before dispatch', store.getState());
            const res = dispatch(...args);
            console.log('after dispatch', store.getState());
            return res;
        }
    }
    return rewrite;
}
```

这里不得不说，思路还是比较巧妙的，借助 applyMiddware 隐藏了复写的过程，又通过箭头函数的联写，简化了代码的实现，站在使用侧的角度来说，看起来确实神清气爽一些。

### 真正 redux 的实现

上述写法上已经和 redux 中间件一致了，但是在 applyMiddware 的实现上，还存在一些偏差，虽然结果一样，但是 redux 的实现更为巧妙。代码如下：

```js
// 借助 reduce 实现 fn1, fn2, fn3 => (...args) => fn1(fn2(fn3(...args))) 的效果
const compose = (fns) => fns.reduce((a, b) => (args) => a(b(args)));

const applyMiddware = (store, middwares) => {
    //  执行一次注入 store
    const fns = middwares.map((middware) => middware(store));
    
    // 使用 compose 组合后，赋值修改真正的 dispatch
    store.dispatch = compose(fns)(store.dispatch);
};
```

核心就是 compose 函数，这里确实不好理解，这里面的一个关键点就是 reduce 的返回值是个函数，这个函数本身并不会执行，只有当 `compose(fns)(store.dispatch)` 时，才会真正执行。

而 compose 的功能，就是将入参中的一系列函数: `fn1, fn2, fn3, fn4` 转为 `(...args) => fn1(fn2(fn3(fn4(...args))))`。

如此，当执行这个函数时，就会是从 fn4 开始执行，其入参就是 store.dispatch, 返回值就是修改后的 dispatch，接下来依次执行 fn3, fn2, fn1，从而生成了最终的 store.dispatch，当调用 store.dispatch 时，就会形成 fn1 -> fn2 -> fn3 -> fn4 -> store.dispatch -> fn4 -> fn3 -> fn2 -> fn1 的洋葱效果。

虽说结果和前一节的效果一样，但是这一节的 compose 函数可谓是精华中的精华，相比之下省去了前一节需要的 reverse 操作。

至此，基本上就是整个 redux 的精华实现了。

## 能再支持下异步吗？

redux 最初让人迷惑比较多的地方就是异步(最初)，因为 dispatch 是同步的，而往往业务中往往是发起请求前，dispatch loading态，请求结束后 dispatch 结束态/错误态。

不过想要支持异步其实也很简单，因为本身异步其实和 redux 是无关的，只需要用户自己写函数，然后在不同的时机去触发同步的 dispatch 即可。

```js
// 省略 store 的创建
// const store = createStore();
const asyncAction = () => {
    store.dispatch({ action: 'startAcyns' });
    fetch('some api').then(() => {
        store.dispatch({ action: 'acynsResolve' });
    }).catch(() => {
        store.dispatch({ action: 'acynsError' });
    })
}
```

这样写，可以用，只是看起来手动调用 store.dispatch, store.getState 可能并不优雅。

可以通过中间件来解决，这也是 redux-thunk 的方案，dispatch 一个函数，将 store.dispatch, store.getState 作为入参提供给函数，从而实现更优雅的异步操作。

```js
const thunkMiddware = (store) => (dispatch) => (action) => {
    if (typeof action === 'function') {
        return action(store.dispatch, store.getState);
    }
    
    return dispatch(action);
}

const asyncAction = (dispatch, getState) => {
    dispatch({ action: 'startAcyns' });
     new Promise(resolve => setTimeout(() => { resolve(1) }, 1000)).then(() => {
        dispatch({ action: 'acynsResolve' });
    }).catch(() => {
        dispatch({ action: 'acynsError' });
    })
}

store.dispatch(asyncAction);
```

## 能让普通函数也用上中间件吗？

redux 这个洋葱模型还是很实用的，如果想给普通函数绑定上这样按照洋葱模型使用的中间件，可以吗？

通过之前的分析，redux 中间件的本质就是在不断的修改 store.dispatch，那么其实只需要将其替换为目标的函数，即可实现。完整的代码如下：

```js
// 给普通函数增加中间件
const createFuncWithMiddware = (func, middwares) => {
  const compose = (fns) => fns.reduce((a, b) => (args) => a(b(args)));

  const applyMiddware = (store, middwares) => {
    const fns = middwares.map((middware) => middware(store));
    store.dispatch = compose(fns)(store.dispatch);
  };
  
  
  // 按照 store 的模式直接创建一个
  const store = {
    getState: () => {},
    // dispatch 指定为目标函数
    dispatch: func,
  };

  applyMiddware(store, middwares);

  return store.dispatch;
};
```

用法如下：

```js
// 省略timeMiddware, loggerMiddware 的定义
function normalfunction(a, b) {
  console.log(a, b);
  return a + b;
}

const newFunction = createFuncWithMiddware(
  normalfunction,
  [timeMiddware, loggerMiddware],
);

newFunction(1, 2);
```

此时，普通函数也会按照洋葱模型执行中间件的一个逻辑。

不过其中的 store 的创建不太有必要，函数本身并不需要 getState 等方法，基于此，可以对上述代码再进行简化，省略掉整体的 store。

```js
const compose = (fns) => fns.reduce((a, b) => (args) => a(b(args)));

// 省去了 store.dispatch 相关逻辑
const applyMiddware = (func, middwares) => compose(middwares)(func);

// 省去了 store => xxx 的逻辑，少了一层箭头函数的联写
const loggerMiddware = (dispatch) => (...args) => {
    console.log('before dispatch', args);
    const res = dispatch(...args);
    console.log('after dispatch', args);
    return res;
};

function normalfunction(a, b) {
  console.log(a, b);
  return a + b;
}

const newFunction = applyMiddware(normalfunction, [loggerMiddware]);

newFunction(1, 2);

// 输出
// before dispatch [1, 2]
// 1 2
// after dispatch [1, 2]
```

但是，对于普通函数来讲，也存在异步的场景，如果不加以处理，loggerMiddware 便会失去其原本的作用。如下：

```js
async function normalfunction(a, b) {
  const res = await new Promise((resolve) => {
    setTimeout(() => {
      resolve(a + b);
    }, 1000);
  });
  console.log(a, b);
  return res;
}

// 此时会输出
// before dispatch [1, 2]
// after dispatch [1, 2]
// 1s 后，输出 1 2
```

这是由于中间件中没有支持异步，也很简单，但是异步具有传导性，所有的中间件都必须改为支持异步的写法：

```js
const loggerMiddware =
  // 外层无需异步，这里就应该是同步修改原函数的行为
  (dispatch) =>
  // 此处需要支持异步
  async (...args) => {
    console.log('before dispatch', args);
    const res = await dispatch(...args);
    console.log('after dispatch', args);
    return res;
  };
```

这样修改之后，就能够满足正常的异步需求了。

至此，这一轮面试题，我认为才算走到了终点。当然，还可以继续发散，比如 koa 的中间件，express 的中间件，这些又和 redux 的中间件有何区别？不过面试就这么点时间，这么多怕是问不完啦。

## 结束

本文来自一道面试题。虽说是一道题，但是 redux 背后的中间件思想还是非常实用的一个模式。

本文在实现 redux 核心的基础上，对中间件的实现进行了比较深入的探讨，从写法和原因上都进行了不同程度的讨论，这块代码虽然短，但是确实不太容易理解。

与此同时，本文进行了发散，在借助 redux 的基础上，对任意函数套用洋葱模型来将中间件应用在每个函数上，而且代码真的就一两行，可以说是本文的精华了。

最后，面试结果，自然是通过啦，都写到这个份上了，也算是真真真超常发挥了。
