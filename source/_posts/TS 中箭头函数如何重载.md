---
title: TS 中箭头函数如何重载
date: 2023-05-24 15:56
categories: 原创
---

这个问题来自于我在网上搜索的时候，基本上清一色的翻译官网的函数重载的章节的内容，对于我想要的箭头函数的重载没有太多帮助。包括官网，其实也没有非常明确的说明箭头函数该如何重载。

<!--more-->

这里先直接上结论，在 ts 中，可以借助 [Call Signatures](https://www.typescriptlang.org/docs/handbook/2/functions.html#call-signatures) 这个特性来实现箭头函数的重载。原本是用来给函数声明增加静态属性的，但是却可以用来完成箭头函数的类型声明。

```ts
type Test = {
  (s: string): string;
  (s: number): number;
  (s: string, b: number): number;
};

type getState<T> = {
  (): T;
  <K extends keyof T>(key: K): T[K];
};
```

本质就是定一个新的类型，键值就用括号包，里面就是不同的入参，函数的返回值就是键值即可。

但是不出意外的话，意外就会发生，在实操的时候，往往这么写非常不 ok。

## 实操

实战中这么写，类型声明是好写的，但是函数的实现，其实并不好写，以这样一个函数为例：

```ts
const deal = (a: number | string, b: number | string): number | string => {
  if (typeof a === 'number' && typeof b === 'number') {
    return a + b;
  }
  if (typeof a === 'string' || typeof b === 'string') {
    return String(a) + String(b);
  }

  return 0;
};
```

当入参都是 number 时，返回 number，入参有一个是 string 时，返回 string。

那么对于这样一个函数，其实存在几个重载的情况：

```ts
type Deal1 = {
  (a: number, b: number): number;
  (a: number, b: string): string;
  (a: string, b: number): string;
  (a: string, b: string): string;
};
```

要想把上述类型赋值给 deal 函数，会出现返回值匹配不上的问题。

```ts
// ts 会提示类型错误：
// Type '(a: number | string, b: number | string) => number | string' is not assignable to type 'Deal1'.
// Type 'string | number' is not assignable to type 'number'.
const deal: Deal1 = (
  a: number | string,
  b: number | string,
): number | string => {
  if (typeof a === 'number' && typeof b === 'number') {
    return a + b;
  }
  if (typeof a === 'string' || typeof b === 'string') {
    return String(a) + String(b);
  }

  return 0;
};
```

从现象来看，ts 对于这样的写法在做检查的时候，会将当前函数对重载的几个类型都进行检查，看看类型上是否能够赋值，相当于：

```ts
// 伪代码，理解意思就行
check1: (a: number, b: number) => number = (a: number | string, b: number | string) => number | string);
check2: (a: number, b: string) => string = (a: number | string, b: number | string) => number | string);
check3: (a: string, b: number) => string = (a: number | string, b: number | string) => number | string);
check4: (a: string, b: string) => string = (a: number | string, b: number | string) => number | string);
```

对于入参，由于是逆变位置，所以 `number = number | string` 能够赋值，所以参数的类型能够通过校验，而返回值属于顺变位置，所以 `number = number | string` 是不能通过类型校验的。

想要将返回值赋值成功，返回值必须是 `number & string` 或者 any，前者就是 never 了，此时会发现虽然赋值通过了 deal 的校验，但是函数的实现中，就会报返回值错误的问题。如果改成 any，那么虽然不会报错，但是在函数中就缺失了对返回值的检查。如下：

```ts
// 返回值改为 number & string，
// 赋值处能够避免类型错误
const deal: Deal1 = (
  a: number | string,
  b: number | string,
): number & string => {
  if (typeof a === 'number' && typeof b === 'number') {
    // 此时返回值是 never，此处会报类型错误
    return a + b;
  }
  if (typeof a === 'string' || typeof b === 'string') {
    // 此时返回值是 never，此处会报类型错误
    return String(a) + String(b);
  }

  // 此时返回值是 never，此处会报类型错误
  return 0;
};

// 返回值改为 any，
// 赋值处能够避免类型错误
const deal: Deal1 = (a: number | string, b: number | string): any => {
  if (typeof a === 'number' && typeof b === 'number') {
    return a + b;
  }
  if (typeof a === 'string' || typeof b === 'string') {
    return String(a) + String(b);
  }

  // 此处写任何类型都不会抛错，缺失了原本的期望的校验
  return undefined;
};
```

那么目前看，想要用这种方式实现箭头函数的重载，就只能将返回值设定为 any，这样，虽然在用户使用的时候能够进行非常好的类型提示，但是开发者本身不能再借助 ts 完成对这个函数的返回值的校验。

此时还有一种写法，就是 `as`，写法如下：

```ts
const deal = ((a: string | number, b: number | string): number | string => {
  if (typeof a === 'number' && typeof b === 'number') {
    return a + b;
  }
  if (typeof a === 'string' || typeof b === 'string') {
    return String(a) + String(b);
  }

  return 0;
}) as Deal1;
```

这样的写法，即能够满足函数本身的返回值校验 (可以把 0 改成其他类型试试)，同时，又具备了 Deal1 的重载的类型声明:

```ts
// case1: number
const case1 = deal(1, 2);

// case2: number
const case2 = deal('1', 2);

// case3: never，入参类型错误
const case3 = deal({}, 2);
```

## 最佳实践

附上完整的代码：

```ts
// 重载类型声明
type Deal1 = {
  (a: number, b: number): number;
  (a: number, b: string): string;
  (a: string, b: number): string;
  (a: string, b: string): string;
};

const deal = ((
  // 此时入参，返回值的类型可自行限制
  // 无需挂念 Deal1 中的定义
  a: string | number,
  b: number | string,
): number | string => {
  if (typeof a === 'number' && typeof b === 'number') {
    return a + b;
  }
  if (typeof a === 'string' || typeof b === 'string') {
    return String(a) + String(b);
  }
  // 这样写，原函数具备校验的能力
  return 0;
  // 通过 as 指定类型
}) as Deal1;

const case1 = deal(1, 2);

const case2 = deal('1', 2);

const case3 = deal({}, 2);
```

核心就是原函数类型写法一致，重载的类型通过 as 进行赋值，这样就兼顾了类型提示和原函数的类型校验。

## 其他方法

由于 type 和 interface 的用法在此处并无歧义，所以换成 interface 也是 ok。

另一种就是函数的交叉，也是实现重载的一种方案，如下：

```ts
type Deal3 = ((a: number, b: number) => number) &
  ((a: number, b: string) => string) &
  ((a: string, b: number) => string) &
  ((a: string, b: string) => string);
```

## 总结

本文介绍了 TS 实现箭头函数重载的几种方案，借助了 [Call Signatures](https://www.typescriptlang.org/docs/handbook/2/functions.html#call-signatures) 的特性。

同时给出了这样写，在实战中可能会遇到的类型不匹配的问题及解决方案。希望能够对遇到同样问题的同学有所帮助吧。

最后，我写了一份 [TS 类型体操的全题解](https://blog.maxiaobo.com.cn/type-challenge/dist/)，也拉了个小群讨论学习 TS，感兴趣的小伙伴可以 [联系我](https://blog.maxiaobo.com.cn/type-challenge/dist/Contactme.html) 一起交流学习呀。