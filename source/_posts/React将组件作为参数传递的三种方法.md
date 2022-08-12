---
title: React 将组件作为参数进行传递的三种方法
date: 2022-07-17 17:50
categories: 原创
---

在日常的开发中，开发通用组件的机会其实并不多，尤其是在各种组件库已经遍地都是的情况下。而作为一个通用组件库的使用者，经常会看到把 React 组件作为参数传递下去的场景，每当这个时候，其实或多或少都会有一些疑问，比如：有些组件传递下去的是组件名，而有些组件传递下去的是一个箭头函数返回一个组件，而有些直接传递一个 jsx 创建好的元素，这些传递方案的适用场景如何，有什么不同，是否会导致组件的 memo 失效，是否会引发组件的不必要渲染?

本文是笔者在阅读了 [antd](https://ant.design/components/overview-cn/)、[mui](https://mui.com/material-ui/react-button/#buttons-with-icons-and-label), [react-select](https://react-select.com/components) 的 api 之后，结合自己日常业务中使用的组件 api 格式，对传递一个组件作为 React 组件参数的方式的思考和总结，如果有写的不到位的，欢迎补充和指点。

<!--more-->

大体来讲，传递组件的方式，分为三种：

1. 传递 jsx 创建好的元素
2. 传递组件本身
3. 传递返回 jsx 创建好的元素的函数

下文也主要展开介绍这三种方式并结合实际场景对比这三种方案。

### 方式一：直接传递 jsx 创建好的元素

在 antd 的组件 api 中，最常见的方式便是这个方法，以 button 为例，有一个 icon 参数便是允许使用者传递一个经过 jsx 创建好的元素。简化后的示例如下：

```js
function DownloadOutlined() {
    return /* icon 的实现*/;
} 

function Button({ icon, children }) {
    return <button>
        {icon}
        {children}
    </button>
}

function App() {
    return <Button icon={<DownloadOutlined />}>test</Button>
}
```

可以看出来，icon 直接传递了一个 jsx 创建好的组件，从而满足了用户自定义 icon 的需求。

相比于通过字符串枚举内置 icon, 给了用户更大的定制空间。

### 方式二：直接传递组件本身

这一用法在 antd 中很少出现，在 [react-select](https://react-select.com/components) 中比较常见。

这里为了方便还是以 Button 为例，修改下上文的 Button 组件，将其参数改为传递 ```DownloadOutlined``` 而非经过 jsx 创建好的元素 ```<DownloadOutlined />```

```js
function DownloadOutlined() {
    return /* icon 的实现*/;
} 

function Button({ icon: Icon, children }) {
    return <button>
    // 渲染方式进行了改变
    <Icon />
    {children}
    </Button>
}

function App() {
    return <Button icon={DownloadOutlined}>test</Button>
}
```

通过直接传递组件本身的方式，也可将其传递给子组件进行渲染，当然，子组件渲染的地方也改成了 ```<Icon />``` 而非上文的 ```{icon}```。ps: 上文中由于 jsx 语法要求，将 icon 变量名改成了首字母大写的 Icon。

### 方式三：传递一个返回组件的函数

这一用法用 Button 示例改写如下：

```js
function DownloadOutlined() {
    return /* icon 的实现*/;
} 

function Button({ icon, children }) {
    return <button>
    // 渲染方式进行了改变
    {icon()}
    {children}
    </Button>
}

function App() {
    return <Button icon={() => <DownloadOutlined />}>test</Button>
}
```

在这一例子中，由于传递的是个函数，那么返回值在渲染时，改成执行函数即可。

### 三种方案的对比

上文中分别介绍了这三种方案的实现方法，从结果来看，三种方案都能满足传递组件作为组件参数的场景。

但是在实际的场景中，往往不会这么简单，往往有更多需要考虑的情况。

情况一： 考虑是否存在不必要的渲染?

三种方案下，当父组件发生渲染时，Button 组件是否会发生不必要的渲染。示例如下：

```js
import React, { useState } from 'react';

function DownloadOutlined() {
  return <span>icon</span>;
}

const Button1 = React.memo(({ icon, children }) => {
  console.log('button1 render');

  return (
    <button>
      {icon}
      {children}
    </button>
  );
});

const Button2 = React.memo(({ icon: Icon, children }) => {
  console.log('button2 render');

  return (
    <button>
      <Icon />
      {children}
    </button>
  );
});

const Button3 = React.memo(({ icon, children }) => {
  console.log('button3 render');
  return (
    <button>
      {icon()}
      {children}
    </button>
  );
});

export default function App() {
  const [count, setCount] = useState(0);
  console.log('App render');

  return (
    <>
      <Button1 icon={<DownloadOutlined />}>button1</Button1>
      <Button2 icon={DownloadOutlined}>button2</Button2>
      <Button3 icon={() => <DownloadOutlined />}>button3</Button3>
      <button onClick={() => setCount((pre) => pre + 1)}>render</button>
    </>
  );
}
```

在该示例中，点击 render button，此时，期望的最小渲染应该是仅仅渲染 app 组件即可，Button1 - Button3 由于并未依赖 count 的变化，同时 Button1 - Button3 都通过 React.memo 进行包裹，期望的是组件不进行渲染。

实际输出如下：

![img](https://tva1.sinaimg.cn/large/e6c9d24egy1h4a8l7sj5uj204g01zglf.jpg)

可以看出，Button1 和 Button3 均进行了渲染，这是由于这两种方案下，icon的参数发生了变化，对于 Button1, ```<DownloadOutlined />```, 本质是 ```React.createElement(DownloadOutlined)```, 此时将会返回一个新的引用，就导致了 Button1 参数的改变，从而使得其会重新渲染。而对于 Button3，就更加明显，每次渲染后返回的箭头函数都是新的，自然也会引发渲染。而只有方案二，由于返回的始终是组件的引用，故不会重新渲染。

要避免（虽然实际中，99%的场景都不需要避免，也不会有性能问题）这种情况，可以通过加 memo 解决。改动点如下：

```js
export default function App() {
  const [count, setCount] = useState(0);
  console.log('App render');

  const button1Icon = useMemo(() => {
      return <DownloadOutlined />;
  }, []);

  const button3Icon = useCallback(() => {
      return () => <DownloadOutlined />;
  }, []);

  return (
    <>
      <Button1 icon={butto1Icon}>button1</Button1>
      <Button2 icon={DownloadOutlined}>button2</Button2>
      <Button3 icon={button3Icon}>button3</Button3>
      <button onClick={() => setCount((pre) => pre + 1)}>render</button>
    </>
  );
}
```

通过 useMemo, useCallback包裹后，即可实现 Button1, Button3 组件参数的不变，从而避免了多余的渲染。相比之下，目前看，直接传递组件本身的方案写法似乎更为简单。

实际的场景中，Icon 组件往往不会如此简单，往往会有一些参数来控制其比如颜色、点击行为以及大小等等，此时，要将这些参数传递给 Icon 组件，这也是笔者想要讨论的：

情况二：需要传递来自父组件(App)的参数的情况。

在现有的基础上, 以传递 size 到 Icon 组件为例，改造如下：

```js
import React, { useState, useMemo, useCallback } from 'react';

// 增加 size 参数, 控制 icon 大小
function DownloadOutlined({ size }) {
  return <span style={{ fontSize: `${size}px` }}>icon</span>;
}

// 无需修改
const Button1 = React.memo(({ icon, children }) => {
  console.log('button1 render');

  return (
    <button>
      {icon}
      {children}
    </button>
  );
});

// 增加 iconProps，来传递给 Icon 组件
const Button2 = React.memo(({ icon: Icon, children, iconProps = {} }) => {
  console.log('button2 render');

  return (
    <button>
      <Icon {...iconProps} />
      {children}
    </button>
  );
});

// 无需修改
const Button3 = React.memo(({ icon, children }) => {
  console.log('button3 render');
  return (
    <button>
      {icon()}
      {children}
    </button>
  );
});

export default function App() {
  const [count, setCount] = useState(0);
  const [size, setSize] = useState(12);
  console.log('App render');
  
  // 增加size依赖
  const button1Icon = useMemo(() => {
    return <DownloadOutlined size={size} />;
  }, [size]);
  
  // 增加size依赖
  const button3Icon = useCallback(() => {
    return <DownloadOutlined size={size} />;
  }, [size]);

  return (
    <>
      <Button1 icon={button1Icon}>button1</Button1>
      <Button2 icon={DownloadOutlined} iconProps={{ size }}>
        button2
      </Button2>
      <Button3 icon={button3Icon}>button3</Button3>
      <button onClick={() => setCount((pre) => pre + 1)}>render</button>
      <button onClick={() => setSize((pre) => pre + 1)}>addSize</button>
    </>
  );
}
```

通过上述改动，可以发现，当需要从 App 组件中，向 Icon 传递参数时，Button1 和 Button3 组件本身不需要做任何改动，仅仅需要修改 Icon jsx创建时的参数即可，而 Button2 的 Icon 由于渲染发生在内部，故需要额外传递 iconProps 作为参数传递给 Icon。与此同时，render按钮点击时，由于 iconProps 是个引用类型，导致触发了 Button2 的额外渲染，当然可以通过 useMemo 来控制，此处不再赘述。

接下来看情况三，当子组件(Button1 - button3)需要传递它自身内部的状态到 Icon 组件中时，需要做什么改动。

设想一个虚构的需求， Button1 - Button3 组件内部维护了一个状态，count，也就是每个组件点击的次数，而 ```DownloadOutlined``` 也接收一个参数，count, 随着 count 的变化，他的颜色会从 ```rbg(0, 0, 0)``` 变化为 ```rgb(count, 0, 0)```。

DownloadOutlined 改动如下:

```js
// 增加 count 参数，控制 icon 颜色
function DownloadOutlined({ size = 12, count = 0 }) {
  console.log(count);
  return (
    <span style={{ fontSize: `${size}px`, color: `rgb(${count}, 0, 0)` }}>
      icon
    </span>
  );
}
```

Button2 的改造(Button1放在最后)如下:

```js
const Button2 = React.memo(({ icon: Icon, children, iconProps = {} }) => {
  console.log('button2 render');
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(pre => pre + 40)}>
      {/* 将count参数注入即可 */}
      <Icon {...iconProps} count={count} />
      {children}
    </button>
  );
});
```

Button3的改造如下：

```js
const Button3 = React.memo(({ icon, children }) => {
  console.log('button3 render');
  const [count, setCount] = useState(0);

  return (
    // 此处为了放大颜色的改变，点击一次加 40
    <button onClick={() => setCount(pre => pre + 40)}>
      {/* 将 count 作为参数传递给 icon 函数 */}
      {icon({count})}
      {children}
    </button>
  );
});
```

相应的，App 组件传入也需要做改动

```js
export default function App() {
  /* 省略 */
  
  const button3Icon = useCallback((props) => {
    // 接收参数并将其传递给icon组件
    return <DownloadOutlined size={size} {...props} />;
  }, [size]);

  /* 省略 */
}
```

而对于 button1, 由于 icon 渲染的时机，是在 App 组件中，而在 App 组件中，获取 Button1 组件内部的状态并不方便(可以通过 ref， 但是略显麻烦)。此时可以借助 ```React.cloneElement``` api来新建一个 Icon 组件并将子组件参数注入，改造如下：

```js
const Button1 = React.memo(({ icon, children }) => {
  console.log('button1 render');
  const [count, setCount] = useState(0);
  // 借助 cloneElement 向icon 注入参数
  const newIcon = React.cloneElement(icon, {
    count,
  });

  return (
    <button onClick={() => setCount((pre) => pre + 40)}>
      {newIcon}
      {children}
    </button>
  );
});
```

从这个例子可以看出，如果传入的组件(icon)，需要获取即将传入组件(Button1, Button2, Button3)内部的组件，那么直接传递 jsx 创建好的元素，并不方便，因为在父组件(App)中获取子组件(Button1)内部的状态并不方便，而直接传递组件本身，和传递返回 jsx 创建元素的函数，前者由于元素真正的创建，就是发生在子组件内部，故可以方便的获取子组件状态，而后者由于是函数式的创建，通过简单的参数传递，即可将内部参数传入 icon 中，从而方便的实现响应的需求。

### 总结

本文先简单介绍了三种将组件作为参数传递的方案：

1. 传递 jsx 创建好的元素: ```icon = {<Icon />}```
2. 传递组件本身: ```icon={Icon}```
3. 传递返回 jsx 创建好的元素的函数: ```icon={() => <Icon />}```

接下来，从三个角度对其进行分析：

1. 是否存在不必要的渲染
2. Icon 组件需要接收来自父组件的参数
3. Icon 组件需要接收来自子组件的参数

其中，三种方案，在不做 useMemo, useCallback 这样的缓存情况下，直接传递组件本身，由于引用不变，可以直接避免非必要渲染，但是当需要接收来自父组件的参数时，需要开辟额外的字段 iconProps 来接收父组件的参数，在不做缓存的情况下，由于参数的对象引用每次都会更新从而也存在不必要渲染的情况。当然，这种不必要的渲染，在绝大部分场景下，并不会存在性能问题。

考虑了来自父组件的传参后，除了方案二直接传递组件本身的方案需要对子组件增加 iconProps 之外，其余两个方案由于 jsx 创建组件元素的写法本身就在父组件中，只需稍作改动即可将参数携带入 Icon 组件中。

而当需要接收来自子组件的参数场景下，方案一显得略有不足，jsx 的创建在父组件已经创建好，子组件中需要注入额外的参数相对麻烦(使用 cloneElement 实现参数注入)。而方案三由于函数的执行时机是在子组件内部，可以很方便的将参数通过函数传参带入 Icon 组件，可以很方便的满足需求。

从实际开发组件的场景来看，被作为参数传递的组件需要使用子组件内部参数的，一般通过方案三传递函数的方案来设计，而不需要子组件内部参数的，方案一二三均可，实际的开销几乎没有差异，只能说方案一写法较为简单，也是 antd 的 api 中最常见的用法。而方案三，多见于需要子组件内部状态的情况，比如 antd 的面包屑 [itemRender](https://ant.design/components/breadcrumb-cn/#Breadcrumb)，Form.list的 [children](https://ant.design/components/form-cn/#Form.List) 的渲染，通过函数注入参数给被作为参数传递的组件方便灵活的进行渲染。

最后，由于笔者之前写过一段时间vue，不免还是想到了 vue 中 slot 的写法，说实话，还是回去翻了下文档，其实就是方案一和方案三的合集，由于slot本身是在父组件渲染的，所以直接具备父组件的作用域，能够访问父组件的状态，需要注入父组件参数的，直接在插槽的组件中使用即可，而作用域插槽便是提供子组件的作用域，使插槽中的组件可以获取到子组件的参数。

最后的最后，觉得有帮助的，还请不要吝啬你的赞哈，也欢迎交流。毕竟笔者绝大部分的时间都在写业务。
