---
title: React Hooks 心智负担：消个抖都消不利索了
date: 2022-05-27 18:04
categories: 原创
---

React Hooks已经有三年多的时间了，团队中也基本上已经很少看到class组件的身影，也很少再看到用不用hooks的争论，笔者认为，hooks优势确实明显，但是随之而来的心智负担也确实存在。就比如，按钮消个抖，非常常见的业务诉求，但是实际写起来，确实并不容易。

网上也确实有一些优秀的 hooks 库已经进行了实现，但是一些分析介绍的文章，或多或少有点问题。故写此文记录下，写的不好的地方也欢迎交流指出。

<!--more-->

## 业务背景

消抖在日常的开发中遇到的场景非常常见，比如一个输入搜索的逻辑，页面中一个输入框，根据用户输入，从远程拉取结果返回。

这个场景下，如果不对用户输入进行消抖，那么请求的频率会非常频繁。故需要对输入行为进行消抖，来达到减少请求频次的效果。

未经消抖的简单的代码如下：

```jsx
import React, { useCallback, useState } from 'react';

const fakeFetch = (v) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(JSON.stringify(v));
    }, 500);
  });
};

export default function App() {
  const [value, setValue] = useState('');
  const [res, setRes] = useState('');
  const [fakeFetchCount, setFakeFetchCount] = useState(0);

  const getRes = useCallback((v) => {
    setFakeFetchCount((pre) => pre + 1);
    fakeFetch({
        value: v,
    }).then((res) => setRes(res));
  }, []);

  const onChange = useCallback((e) => {
    setValue(e.target.value);
    getRes(e.target.value);
  }, []);

  return (
    <div>
      <input value={value} onChange={onChange} />
      <div>{res}</div>
      <div>fetch次数: {fakeFetchCount}</div>
    </div>
  );
}
```

效果如图：

![img](https://tva1.sinaimg.cn/large/e6c9d24ely1h2jb1szrb1g205h02mt9a.gif)

上例中，当用户输入，那么便会进行请求拉取远程接口的返回。这里使用了一个mock的fetch，定时500ms后返回结果。

可以看出来，每输入一个字母，就会发起一次请求，这在实际的业务中，显然是给服务器带来了不必要的压力。

## 消抖方法

### 传统的消抖

正常的消抖比较简单，其本质是在执行时，设定一个延时为 waitTime 的定时函数，每次执行，都先清除上一次的定时器，再新起一个定时器，如果触发频率较快，那么上一次设定好的定时器，会一直被清除，直到该函数不再被触发，从而执行上一次设定好的定时器函数，利用这样的方法，达到消除抖动的目的。

常见的实现如下：

```js
function debounce(func, waitTime) {
  let timer = null;

  return function (...args) {
    if (timer) {
      clearTimeout(timer);
    }
    timer = setTimeout(() => {
      func.apply(this, args);
    }, waitTime);
  };
}
```

### hooks 写法

#### 直接消抖

借助上文的 debounce 函数，很容易在组件中写下如下的消抖方案：

```jsx
// version 1： 直接消抖的代码
import React, { useCallback, useState, useEffect } from 'react';

function debounce(func, waitTime) {
  let timer = null;

  return function (...args) {
    if (timer) {
      clearTimeout(timer);
    }
    timer = setTimeout(() => {
      func.apply(this, args);
    }, waitTime);
  };
}

const fakeFetch = (v) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(JSON.stringify(v));
    }, 500);
  });
};

export default function App() {
  const [value, setValue] = useState('');
  const [res, setRes] = useState('');
  const [fakeFetchCount, setFakeFetchCount] = useState(0);
  
  // 用debounce函数包裹原函数
  const getRes = useCallback(
    debounce((v) => {
      setFakeFetchCount((pre) => pre + 1);
      fakeFetch({
        value: v,
      }).then((res) => {
        setRes(res);
      });
    }, 500),
    []
  );

  const onChange = useCallback(
    (e) => {
      setValue(e.target.value);
      getRes(e.target.value);
    },
    [getRes]
  );

  return (
    <div>
      <input value={value} onChange={onChange} />
      <div>{res}</div>
      <div>fetch次数: {fakeFetchCount}</div>
    </div>
  );
}
```

关键修改点就是将原函数通过 debounce 函数包裹即可，其效果如下图：

![img](https://tva1.sinaimg.cn/large/e6c9d24ely1h2jb9uvd2ug207x031q3l.gif)

从结果上来看，这样写其实就已经达到了想要的效果。

但是仔细分析下代码：

```jsx
const getRes = useCallback(
    debounce((v) => {
      setFakeFetchCount((pre) => pre + 1);
      fakeFetch({
        value: v,
      }).then((res) => {
        setRes(res);
      });
    }, 500),
    []
);
```

组件每次在渲染的时候，debounce 函数每次都会执行一次，生成一个新的经过消抖的函数，但是由于 useCallback 中的依赖为空，所以 getRes 每次都会使用第一次生成的消抖后的函数，从而在表现上是符合预期的。

#### 加入了依赖项

每次都执行一次 debounce 在性能上的损耗可以忽略不计，但是如果 useCallback 有依赖，那么每次 getRes 函数都会是新的消抖后的函数，抛开函数更新后引发的子组件渲染问题不谈，由于消抖函数中存在一个定时器，getRes 函数更新后，旧的消抖后的函数中的定时器无人清除，会被执行，从而缺失了消抖的效果。

带到实例中，假设服务端的接口，除了需要用户的输入，还需要用户在页面停留的时长作为参数，修改代码如下：

```jsx
// version2: 加入了停留时长依赖的消抖
import React, { useCallback, useState, useEffect } from 'react';

function debounce(func, waitTime) {
  let timer = null;

  return function (...args) {
    if (timer) {
      clearTimeout(timer);
    }
    timer = setTimeout(() => {
      func.apply(this, args);
    }, waitTime);
  };
}

const fakeFetch = (v) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(JSON.stringify(v));
    }, 500);
  });
};

export default function App() {
  const [value, setValue] = useState('');
  const [res, setRes] = useState('');
  const [fakeFetchCount, setFakeFetchCount] = useState(0);
  // 增加用户停留时间状态
  const [duration, setDuration] = useState(0);
  
  // 用户停留时长更新
  useEffect(() => {
    setInterval(() => {
      setDuration((pre) => pre + 1);
    }, 1000);
  }, []);

  const getRes = useCallback(
    debounce((v) => {
      setFakeFetchCount((pre) => pre + 1);
      fakeFetch({
        value: v,
        // 请求时携带duration参数
        duration,
      }).then((res) => {
        setRes(res);
      });
    }, 500),
    // 增加对用户停留时长状态的依赖
    [duration]
  );

  const onChange = useCallback(
    (e) => {
      setValue(e.target.value);
      getRes(e.target.value);
    },
    [getRes]
  );

  return (
    <div>
      <input value={value} onChange={onChange} />
      <div>{res}</div>
      <div>fetch次数: {fakeFetchCount}</div>
    </div>
  );
}
```

核心的修改在于，原来 useCallback 中的依赖为空，现在依赖了用户停留时长 duration，效果如下：

![version3](https://tva1.sinaimg.cn/large/e6c9d24ely1h2jc30phs5g20em0310ve.gif)

当一直快速输入时，fetch 次数还一直在增加，这个其实就是由于 duration 这个状态每隔 1s 更新一次，此时 getRes 函数由于依赖了 duration，那么 getRes 函数也会变成新的消抖后的函数，从而导致上一个的 getRes 函数设置的定时器无人清除，从而导致 fetch 的执行。

简单整理下，就是**直接对原函数使用 debounce 消抖，在普通情况下，是能够达到预期的。**

**但是当原函数对外部有依赖，同时组件在消抖函数执行的期间，触发了该依赖的修改，那么此时旧的消抖函数的定时器会无人清理，导致消抖函数还是会被执行，从而达不到预期的效果。**

当然，在上述这个例子中，其实 duration 可以用 useRef 来避免 getRes 函数对它的依赖，但是如果依赖的参数很多呢？此时每一个依赖项都改成 ref, 甚至有些依赖项的修改还需要触发组件的渲染，那么就需要为该状态同时保留 state 和 ref两个变量，即便可以通过自定义hook来封装这层逻辑，代码可读性也会变差。

#### 维持同一个 timer

其实通过上述分析，出现问题的核心在于：

debounce 函数在每次组件渲染后，都会新生成一个函数，此时如果 useCallback 中有依赖，那么上一次的消抖函数生成的定时器可能就没有清理，导致依旧执行带来预期不一致。

了解了问题，解决方案其实也呼之欲出，**核心就在于维持一个 timer，只要 timer 是同一个，那么不管多少次更新，都能保证上一次 timer 的清除**，代码如下：

```jsx
// version 4： 维持同一个 timer
import React, { useCallback, useState, useEffect, useRef } from 'react';

const fakeFetch = (v) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(JSON.stringify(v));
    }, 500);
  });
};

export default function App() {
  const [value, setValue] = useState('');
  const [res, setRes] = useState('');
  const [fakeFetchCount, setFakeFetchCount] = useState(0);
  const [duration, setDuration] = useState(0);

  useEffect(() => {
    setInterval(() => {
      setDuration((pre) => pre + 1);
    }, 1000);
  }, []);
  
  // 用 ref 维持同一个timer
  const timerRef = useRef(null);

  const getRes = useCallback(
    (v) => {
      if (timerRef.current) {
        clearTimeout(timerRef.current);
      }
      timerRef.current = setTimeout(() => {
        setFakeFetchCount((pre) => pre + 1);
        fakeFetch({
          value: v,
          duration,
        }).then((res) => {
          setRes(res);
        });
      }, 500);
    },
    [duration]
  );

  const onChange = useCallback(
    (e) => {
      setValue(e.target.value);
      getRes(e.target.value);
    },
    [getRes]
  );

  return (
    <div>
      <input value={value} onChange={onChange} />
      <div>{res}</div>
      <div>停留时长：{duration}s</div>
      <div>fetch次数: {fakeFetchCount}</div>
    </div>
  );
}
```

其效果如下图：

![img](https://tva1.sinaimg.cn/large/e6c9d24ely1h2jd4jhzmrg20em03dabg.gif)

看起来好像是符合预期的，确实是当用户停止输入后，延时一段时间才进行 fetch。

但是这里其实还隐藏了一个 bug, 由于时延时执行的，虽然 useCallback 加入了 duration 作为依赖，当用户最后一次输入结束后，会开启一个定时器函数，该函数延时一定时间后发出 fetch 请求。

如果 duration 在用户最后一次输入结束后发生了改变，那么定时器使用的其实是上一次的 duration，严格来讲，与预期是不符的。可以在上述例子中，将延时改成 5000ms，效果会比较明显，如下图：

![img](https://tva1.sinaimg.cn/large/e6c9d24ely1h2jdbj3ye6g20em03djt1.gif)

在上图中，将消抖函数延时的长度改成了5s，当用户第一次停止输入时，是5s左右，当定时器真正执行时，大约是 10s 左右，而实际传递给服务器的时间是停止输入时的 duration, 5s。当然，在这个场景下，可能是符合业务预期的，但是如果还依赖其他参数，其他参数也这延时的 5s 发生了改变，但是请求时是旧的值，很有可能会带来线上bug。

#### 维持同一个函数

在维持了同一个 timer 后，确实消除了上一个 timer 未清除带来的频繁执行问题，但是由于消抖函数是延迟 xxxms 执行的，那么此时如果依赖项发生变化，但是用户又没有新的行为触发生成新的消抖函数，那么就会访问到旧的依赖项，从而带来潜在的bug。

**对于延迟带来的旧依赖项问题，其实在 hooks 中非常常见，一般通过 ref 即可解决，但是在这个问题中，如果把消抖函数依赖的参数 ref 化，一方面参数数量不确定，另一方面也不好把逻辑抽离出来，换个思路，可以把函数 ref 化**。代码看起来会更容易理解一些，如下：

```jsx
// version5: 维持同一个timer, func
import React, { useCallback, useState, useEffect, useRef } from 'react';

const fakeFetch = (v) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(JSON.stringify(v));
    }, 500);
  });
};

export default function App() {
  const [value, setValue] = useState('');
  const [res, setRes] = useState('');
  const [fakeFetchCount, setFakeFetchCount] = useState(0);
  const [duration, setDuration] = useState(0);

  useEffect(() => {
    setInterval(() => {
      setDuration((pre) => pre + 1);
    }, 1000);
  }, []);

  const timerRef = useRef(null);
  
  // ref 维持函数
  const funcRef = useRef(null);

  // 原函数定义
  const getRes = useCallback(
    (v) => {
      setFakeFetchCount((pre) => pre + 1);
      fakeFetch({
        value: v,
        duration,
      }).then((res) => {
        setRes(res);
      });
    },
    [duration]
  );
  
  // 始终跟原函数保持一致
  funcRef.current = getRes;
  
  // 消抖逻辑
  const getResDebounced = useCallback((...args) => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
    timerRef.current = setTimeout(() => {
      // 定时器中通过访问ref，避免闭包带来的访问旧值的问题
      funcRef.current.apply(null, args);
    }, 5000);
  }, []);

  const onChange = useCallback(
    (e) => {
      setValue(e.target.value);
      getResDebounced(e.target.value);
    },
    [getResDebounced]
  );

  return (
    <div>
      <input value={value} onChange={onChange} />
      <div>{res}</div>
      <div>停留时长：{duration}s</div>
      <div>fetch次数: {fakeFetchCount}</div>
    </div>
  );
}
```

核心在于：
1. 新建funcRef，同步 getRes 函数，始终保持最新
2. 新建 getRefDebounced 函数，该函数中使用 funcRef.current 作为定时器中执行的函数。

通过这样的方式，延时执行时，始终执行最新的函数，也就是使用了最新参数的 getRes 函数，从而解决 hooks 中，延时执行带来的用到旧依赖项的 bug（通常称为 hooks 闭包问题）。

#### 封装自定义hook

上述代码已经比较清晰了，但是每一次消抖都定义这么多玩意，真的吃不消，hooks 的一大好处就是逻辑的封装，那么自然是要封装一个自定义的消抖 hooks 了。如下：

```jsx
function useDebounceFn(fn, delay) {
  const timerRef = useRef(null);
  const fnRef = useRef(fn);
  fnRef.current = fn;

  const fnDebounced = useCallback(
    function (...args) {
      if (timerRef.current) {
        clearTimeout(timerRef.current);
      }
      timerRef.current = setTimeout(() => {
        fnRef.current.apply(this, args);
      }, delay);
    },
    [delay]
  );

  return fnDebounced;
}
```

通过 ref 维持同一个 timer, 再通过 fnRef, 始终保持跟外部的 fn 同步，最后返回 fnDebounced。

这里有几个细节：

1. fnRef.current = fn, 要不要用 useEffect 包裹，这里我的观点是不需要，useEffect会做一次比较，其实有比较的这个功夫，赋值也不耽误事，实际套个循环（hooks并不推荐，此处仅做实验），1000次左右的话，直接赋值会在0.1ms左右，而useEffect会在1ms左右，差别也不算大。套不套 useEffect 影响真的不大。

2. fnDebounced中的 useCallback 中的函数，一定要使用 普通函数 而非 箭头函数。因为一旦使用箭头函数，那么 this 其实外面就改变不了了。虽然在函数组件中，this 变得越来越不常用，但是可以通过普通函数保留这个 this， 以便外界需要的时候自行使用。

3. 依赖项 deps 放在哪里？不需要放 deps 的地方，全部交由外部传递进来的fn处理，如果需要根据 deps 变化而更新消抖函数，那么直接在外部的 fn 中处理即可。

完整的示例代码如下：

```jsx
// version5: 维持同一个timer, func
import React, { useCallback, useState, useEffect, useRef } from 'react';

const fakeFetch = (v) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(JSON.stringify(v));
    }, 500);
  });
};

function useDebounceFn(fn, delay) {
  const timerRef = useRef(null);
  const fnRef = useRef(fn);

  fnRef.current = fn;

  const fnDebounced = useCallback(
    function (...args) {
      if (timerRef.current) {
        clearTimeout(timerRef.current);
      }
      timerRef.current = setTimeout(() => {
        fnRef.current.apply(this, args);
      }, delay);
    },
    [delay]
  );

  return fnDebounced;
}

export default function App() {
  const [value, setValue] = useState('');
  const [res, setRes] = useState('');
  const [fakeFetchCount, setFakeFetchCount] = useState(0);
  const [duration, setDuration] = useState(0);

  useEffect(() => {
    setInterval(() => {
      setDuration((pre) => pre + 1);
    }, 1000);
  }, []);

  const getRes = useCallback(
    (v) => {
      setFakeFetchCount((pre) => pre + 1);
      fakeFetch({
        value: v,
        duration,
      }).then((res) => {
        setRes(res);
      });
    },
    [duration]
  );

  const getResDebounced = useDebounceFn(getRes, 500);

  const onChange = useCallback(
    (e) => {
      setValue(e.target.value);
      getResDebounced(e.target.value);
    },
    [getResDebounced]
  );

  return (
    <div>
      <input value={value} onChange={onChange} />
      <div>{res}</div>
      <div>停留时长：{duration}s</div>
      <div>fetch次数: {fakeFetchCount}</div>
    </div>
  );
}
```

效果如下：

![img](https://tva1.sinaimg.cn/large/e6c9d24ely1h2jgp1ooexg20em03djs1.gif)

改成 5000ms 延迟后，在调用时，也会走当下的 duration 而非上一次的值。效果如下：

![img](https://tva1.sinaimg.cn/large/e6c9d24ely1h2jgqhk9isg20em03d0tb.gif)

至此，一通分析下，终于得到了一个还算满意无 bug 的消抖自定义hook。

当然这个版本下，也还有可以优化的地方，就是当组件卸载后，对 timer 的消除，来避免组件卸载后回调的执行，代码如下：

```jsx
function useDebounceFn(fn, delay) {
  const timerRef = useRef(null);
  const fnRef = useRef(fn);

  fnRef.current = fn;
  
  
  // 增加组件卸载时，timer 的清除
  useEffect(() => {
    return () => {
      if (timerRef.current) {
        clearTimeout(timerRef.current);
      }
    };
  }, []);

  const fnDebounced = useCallback(
    function (...args) {
      if (timerRef.current) {
        clearTimeout(timerRef.current);
      }
      timerRef.current = setTimeout(() => {
        fnRef.current.apply(this, args);
      }, delay);
    },
    [delay]
  );

  return fnDebounced;
}
```

## 总结

本文通过一个常见的根据用户输入发请求的消抖业务场景入手，从最基础的写法，到 hooks 写法，分析了其中的 bug 及存在的问题，并最终得出了自定义 hooks 的写法。

整体而言，hooks 让 react 组件变的十分纯粹，每次的渲染，不过就是将函数重新执行一遍，相比于 class 组件的生命周期，函数式的组件确实纯粹了许多。

但是由于异步操作以及依赖项的存在，还是导致了不少的闭包问题，也让消抖等一些常见的操作一不小心就掉坑里了。

不过梳理其常见的心智负担之后，其实 hooks 还是又方便又纯粹，而消抖自定义 hooks 也充分展现了其逻辑易复用的好处。

最后，都看到这了，点个赞吧. ╮(╯_╰)╭ ·
