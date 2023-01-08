---
title: 可视化埋点（四）：如何维持 Xpath 稳定性
date: 2023-01-06 10:36
categories: 原创
---

## 概述

在之前的文章中提到一些 xpath 实战的边界场景，评论区小伙伴比较感兴趣，故写此篇文章进行记录。本文主要将 xpath 的稳定性分为三个环节（是足够涵盖之前文章提出的一些边界场景的）：

1. xpath 生成的边界场景：div[1] 与 div，svg 元素，id 的识别
2. 实际业务中，dom 结构不稳定的场景分析：逻辑与表达式，三元表达式，动态弹窗，动态组件
3. xpath 的特殊场景：列表标记。

<!--more-->

## xpath 简介

在正文开始前，还是先对 xpath 进行一个简单的介绍，毕竟即便是工作几年的前端，我想不是特别注意的话，多半是没有听说过 xpath。

xpath 历史渊源什么的，不在此处梳理，在我们现代的 web 开发体系下，用简单的一句话来讲，就是指 元素 在 整个文档流中 当下  所处的位置。

举个例子，html 中有如下 dom 结构：

```html
<html>
    <body>
        <div>
            <ul>
                <li>1</li>
                <li>2</li>
                <li>3</li>
            </ul>
        </div>
        <div>
            <p>一个p</p>
        </div>
    </body>
</html>
```

在上述的 结构下，可以用一个字符串标识 内容为 3，tag 为 li 的元素：```/html/body/div[1]/ul/li[3]```。这一字符串可以通过 chrome 下选中元素，右键复制 -> 复制完整 xpath(此处完整 xpath 和 xpath 的区别，笔者本身也没有做过更仔细的研究，一个非常明显的区别就在于完整 xpath 不会对路径中有 id 属性的元素做处理)。不过这一生成方法浏览器并没有暴露出来，需要使用者自行实现。

一个非常简易的获取元素 xpath 的代码如下：

```js
function getXpath(ele) {
    let cur = ele;
    const path = [];
	while (cur.nodeType === Node.ELEMENT_NODE) {
		const currentTag = cur.nodeName.toLowerCase();
		const nth = findIndex(cur, currentTag);
		path.push(`${(cur.tagName).toLowerCase()}${(nth === 1 ? '' : `[${nth}]`)}`);
		cur = cur.parentNode;
	}
	return `/${path.reverse().join('/')}`;
}

// 其中 findIndex 代码如下：
function findIndex(ele, currentTag) {
    let nth = 0;
    while (ele) {
        if (ele.nodeName.toLowerCase() === currentTag) nth += 1;
        ele = ele.previousElementSibling;
    }
    return nth;
}
```

拿到 xpath 之后，想要通过 xpath 反查元素，可以通过浏览器原生的方法，完整的使用介绍见[这篇文章](https://developer.mozilla.org/en-US/docs/Web/XPath/Introduction_to_using_XPath_in_JavaScript)。

简单来讲，用如下代码即可通过 xpath 反查到元素：

```js
function getEleByXpath(xpath) {
    const doc = document;
    const result = doc.evaluate(xpath, doc);
    const item = result.iterateNext();
    return item;
}
```

## 生成 xpath 的边界场景

简单介绍 xpath 之后，便是本文要讨论的主题，首先是 xpath 的一些边界场景，本章将会从 div[1]、svg 元素 以及 id 的处理来分析实战中的经验及改进方法。

### div[1] 与 div

在上文的例子中，可以看到对于 ```/html/body/div[1]/ul/li[3]``` 中的 ul 元素，浏览器直接生成的 ul 元素，并没有 ul[1]，而是直接 ul，这在当前的 dom 结构下，确实并不影响，但是在实际场景中，如果由于用户的操作，导致当下的 dom 结构变化成如下情形：

```html
<html>
    <body>
        <div>
            <ul>
                <li>1</li>
                <li>2</li>
                <!--原本的 li 3 消失-->
                <!--<li>3</li>-->
            </ul>
            <!--新增了 ul，并且也有 3个 li 元素-->
            <ul>
                <li>1</li>
                <li>2</li>
                <li>3</li>
            </ul>
        </div>
        <div>
            <p>一个p</p>
        </div>
    </body>
</html>
```

此时，执行 ```getEleByXpath('/html/body/div[1]/ul/li[3]')```，由于 ul 没有指明位置，当第一个 ul 下不存在 li[3] 的时候，浏览器会尝试寻找 ul[2]/li[3]，此时命中，便会将原本第一个 ul/li[3] 判定成 第二个 ul/li[3]。

这在可视化埋点场景下，就是用户在录入埋点的时候，由于 dom 结构中仅有一个 ul/li[3]，录入了该 xpath，但是在实际场景中由于用户的交互，导致原本的 ul/li[3] 消失，新增了 ul[2]/li[3]。而浏览器的判断，对于 没有指明 ul[x] 中 x 的情况，会向下继续寻找符合条件的元素，此时就会发生误判。

故在实际使用时，生成的 xpath，必须显式的指明，当前元素在 dom 结构中，同类型元素的第几个，而非之前代码中，如果是位于第一个便省略的处理。

修改后的 getXpath 如下：

```js
function getXpath(ele) {
    let cur = ele;
    const path = [];
	while (cur.nodeType === Node.ELEMENT_NODE) {
		const currentTag = cur.nodeName.toLowerCase();
		const nth = findIndex(cur, currentTag);
		// 此处不再判断是否是第一个元素，全部追加 nth
		path.push(`${(cur.tagName).toLowerCase()}[${nth}]`);
		cur = cur.parentNode;
	}
	return `/${path.reverse().join('/')}`;
}

// 其中 findIndex 代码如下：
// 省略 findIndex 的代码
```

通过显式指明 不同 tag 类型位于当前同类型的第几个，可以有效避免省略 [1] 带来的误判，此为 xpath 中的第一个踩坑点。

### svg 元素

对于 svg 元素，假设有如下 dom 结构。

```html
<html>
    <body>
        <h1>My first SVG</h1>
        <svg width="100" height="100">
            <circle cx="50" cy="50" r="40" stroke="green" stroke-width="4" fill="yellow" />
        </svg>
    </body>
</html>

```

通过上文的 getXpath 方法，获取 circle 元素，可以得到如下 xpath：```/html[1]/body[1]/svg[1]/circle[1]```。

此时通过上文介绍的 ```getEleByXpath('/html[1]/body[1]/svg[1]/circle[1]')```，会发现的到的是空值，这是因为浏览器不支持 svg 元素的查找。

该方法对于 svg 元素，仅支持如下形式的查找：```html[1]/body[1]/*[name()='svg'][1]/*[name()='circle'][1]```。

故，对于 svg 标签，想要浏览器能够查找到，只需要将 svg 替换为 ```*[name()='svg']``` 即可。修改 xpath 获取代码如下：

```js
const svgTags = ['svg', 'path', 'g', 'image', 'text', 'line', 'rect', 'polygon', 'circle', 'ellipse'];

function getTagName(ele) {
    const tag = ele.tagName.toLowerCase();
    if (svgTags.indexOf(tag) !== -1) {
        // 如果是 svg 元素，替换为 name()='xxx' 的形式
        return `*[name()='${tag}']`;
    }
    return tag;
};

function getXpath(ele) {
    let cur = ele;
    const path = [];
	while (cur.nodeType === Node.ELEMENT_NODE) {
		const currentTag = cur.nodeName.toLowerCase();
		const nth = findIndex(cur, currentTag);
		// 替换为 gettagName(cur) 获取元素的 tag
		path.push(`${(getTagName(cur))}[${nth}]`);
		cur = cur.parentNode;
	}
	return `/${path.reverse().join('/')}`;
}
```

通过这样的方法，就可以实现对 svg 元素的识别。虽说如今回过头来看，改动其实很细微，但是在实际的落地过程中，这些问题都是当作 bug 来处理，而且彼时还要兼容旧版本的 xpath，实际的痛是真的痛。

### id 的识别

在浏览器自动生成的完整 xpath 中，对带有 id 的元素不会做特殊的处理，但是如果选择的是复制 xpath，会对有 id 属性标识的元素有特殊的处理。试想如下 dom 结构：

```html
<html>
    <body id="body">
        <h1 id="h1">My first SVG</h1>
        <div id="h1">div h1</div>
    </body>
</html>

```

此时选中 h1 并复制其 xpath 和完整 xpath，可以得到如下两个 xpath:

```//*[@id="h1"]``` 和 ```/html/body/h1``` (可以看出来，浏览器获取的 h1 是没有带序号的，这个问题已经在上文讨论过，故此处不再赘述)。

在这里，可以看到 xpath 的一个新的写法， ```/*[@id="xxx"]```，由于 id 在 html 中，确实是一个比较特殊的存在，虽然在现代 web 开发中，id 越来越不常用，但是也阻止不了大家对同一个 id 只有一个的共识。

那么对于可视化埋点，如果加入 id 的判定，是否能够让 xpath 更加稳定，或者说减少误判呢？

我的理解是能，对于业务中，比较常见的 tab 场景，就可以在每一个 tab 中加入 id，如果在 xpath 中能够对 id 进行处理，那么这几个 tab 即便再相似，也不会出现误判的情形，关于 tab 场景，会在后文讨论，此处仅仅讨论 xpath 如何支持 id。

首先，浏览器默认生成的，基本上都是 ```/*[@id="xxx"]```，此处由于 id 基本只有唯一一个，所以省略了 tag 的标识，但是实际场景中， id 的唯一性保证其实是由使用者保证的，并不可靠，故我在此处加入了 tag 的标识，对于上述结构中的 h1，我选择用 ```/h1[@id="h1"]``` 进行替换。而经验证，浏览器的 getEleByXpath 是能够识别带 tag 的场景。

同时，考虑到 id 有了之后，其实再往上的 dom 结构显得并不重要，故对于在 路径中遇到了包含 id 属性的 dom 节点，将不会再向上搜集结构，对于上文中的 h1, 他的完整 xpath 将会是 ```//h1[@id="h1"]```。用 / 替代了上层的 dom 结构。

同时，我们一般业务中使用的 svg，多半都是从某些软件中直接导出的，这些软件的导出，会默认存在一些 id，同时由于 svg 元素在曝光点 intersectionObserver api 中并不生效，故对于包含了 svg 的 xpath，遇到了包含 id 属性的 dom 节点，不会停止向上搜集，同时在实际记录曝光的时候，会根据 xpath 选择最近的非 svg 元素的 dom 节点作为曝光的依据。

至此，对于 id 的处理，意义主要在于提升埋点的准确性，需要做的改造有：

1. 对于不是 svg 元素的场景，遇到了有 id 属性的 dom 节点，停止向上搜查，用 ```/``` 替代上层的 dom，并在 id 中保留该节点 tag 以便增强稳定性

2. 对于有 svg 元素的场景，保留所有的 dom 层级，对于遇到 id 属性的场景，保留 ```/tag[@id="xxx"]``` 的形式。

改写后的完整代码如下：

```js
export function getXpath(ele: HTMLElement) {
    if (!isDOM(ele)) {
        return null;
    }
    let cur: HTMLElement | null = ele;
    // 是否追加过 id 
    let hasAddedId = false;
    // 判断当前节点是否是 svg 元素
    // 默认认为一般不会有人在 svg 中嵌套 普通 dom 节点
    const hasSvgEle = svgTags.indexOf(cur.tagName.toLowerCase()) !== -1;

    const path = [];
    while (cur && cur.nodeType === Node.ELEMENT_NODE) {
        const currentTag = cur.nodeName.toLowerCase();
        // 查找位置
        const nth = findIndex(cur, currentTag);
        let idMark = '';
        if (!hasAddedId && cur.hasAttribute('id')) {
            // 仅保留最近一层的id
            idMark = `[@id="${cur.id}"]`;
            // 标记加过 id 了
            hasAddedId = true;
        }
        let nthmark = '';
        if (idMark) {
            // 有了 id ，可以忽略位置
            nthmark = '';
            // ignoreTags = ['html', 'body']
        } else if (ignoreTags.indexOf(currentTag) === -1) {
            nthmark = `[${nth}]`;
        } else {
            // 对于body, html 忽略 nth
            nthmark = '';
        }
        path.push(`${getTagName(cur)}${nthmark}${idMark}`);
        // svg元素会一直往上冒，由于 intersectionObser 对 svg 无效，所以需要获取完整的 xpath 来寻找最近的非 svg 元素
        if (idMark && !hasSvgEle) {
            path.push('');
            break;
        }
        cur = cur.parentNode as HTMLElement;
    }
    return `/${path.reverse().join('/')}`;
}
```

此部分完整代码可以参照[这个文件](https://github.com/yuzai/dom-inspector-pro/blob/main/src/dom.ts)。

至此，关于 xpath 的几个边界生成规则，便讲述完成了，接下来将会是由于代码的书写导致的 dom 结构不稳定从而导致的 xpath 不稳定的场景。

## dom 结构不稳定导致的 xpath 不稳

除了在生成 xpath 时的边界场景，还有一个导致 xpath 不稳定的因素就在于我们日常代码中非常常见的一些组件的写法，此节便是对这些场景的分析。

同时为了解决这些问题，开发了一款 babel 插件对这些场景进行语法转换。目前代码并未开源，后文称之为 shadow-babel-plugin

### 逻辑与 && 表达式

在 react 代码中，经常会写下如下代码：

```jsx
function App() {
    return (
        <div>
            {a && <Comp1 />}
            {b && <Comp2 />}
            {c && <Comp3 />}
        </div>
    )
}
```

此时，当用户在 a,b 均为 false 时，对 comp3 录入了埋点，那么当实际上线后，a or b 发生了改变，此时 comp3 的 xpath 便有可能不会命中，此时便会导致 comp3 埋点的误判。

shadow-babel-plugin 针对该场景做了优化，上述代码会转换成以下代码：

```jsx
function App() {
    return (
        <div>
            {a ? <Comp1 /> : <div data-from-shadow="true" style={{"display": "none"}} />}
            {b ? <Comp2 /> : <div data-from-shadow="true" style={{"display": "none"}} />}
            {c ? <Comp3 /> : <div data-from-shadow="true" style={{"display": "none"}} />}
        </div>
    )
}
```

当 a,b 为 false 时，此时会保留一个 display 为 none 的 空的 div 来维持原 dom 结构，从而一定程度上保证了 相邻元素的 xpath 的稳定性。

但是此时还是存在误判的场景，比如 a 为 false， b 为 true，那么此时 dom 结构是 ```<div /><Comp2 />```，当 a 为 true，b 也为 true 时，此时 dom 结构是 ```<Comp1 /><Comp2 />```，如果 comp1 的最外层不是 div，那么就会造成 comp2 的结构误判，故组件最好最外层以 div 包裹，才能最大程度的避免误判。

同时，也会对普通的 && 表达式进行过滤，如果不涉及到 jsx，则不会转换。

与此同时，由于代码结构中多了一层 div，虽然绝大部分场景不会影响，但是 css 选择器 ，如 ```nth-child``` 等，包括父组件对第 x 的元素的特殊处理，还是会有影响。不过好在 babel 插件在开发时便引入，能够在开发阶段就发现问题。

### 条件 ? 表达式

在 react 代码中，三元表达式的代码也非常常见，如下：

```jsx
function App() {
    return (
        <div>
            {a ? <Comp1 /> : <Comp2 />}
            {c ? <Comp3 /> : <Comp4 />}
        </div>
    )
}
```

此时，当策划在埋点时，如果 a 为 false， c 为 false，那么此时页面内只有 comp2 和 comp4，当上线后，在某个状态下，a 为 true, c 为 true，那么此时页面内 只有 comp1 和 comp3，如果 comp1 和 comp2 结构类似，很容易造成 两者的误判。

shadow-babel-plugin 针对该场景做了优化，上述代码会转换成以下代码：

```jsx
function App() {
    return (
        <div>
            { a ? <Comp1 /> : <div data-from-shadow="true" style={{"display": "none"}} /> }
            { a ? <div data-from-shadow="true" style={{"display": "none"}} /> : <Comp2 /> }
            { c ? <Comp3 /> : <div data-from-shadow="true" style={{"display": "none"}} /> }
            { c ? <div data-from-shadow="true" style={{"display": "none"}} /> : <Comp4 /> }
        </div>
    )
}
```

此时，不论 a 是否为 false, 只会是 ```<Comp1 /><div />``` or ```<div /><Comp2 />```，从 结构上，就 不会出现 comp1 和 comp2 的误判。comp3 和 comp4 同理。

但是此时还是存在误判的场景，比如 a 为 false， c 为 true，那么此时 dom 结构是 ```<div /><Comp2 /><comp3 /><div />```(此时假设 comp2 comp3 最外层都是 div, 那么 comp3 的 xpath 为 div[3])，当 a 为 true，c 也为 true 时，此时 dom 结构是 ```<Comp1 /><div /><Comp3 /><div />```，如果 comp1 的最外层不是 div，那么可能就会造成 comp3 的结构误判（假设 comp3 顶层是 div, 那么 xpath 便是 div[2]），故组件最好最外层以 div 包裹，才能最大程度的避免误判。

同时，该插件也会对普通 的 三元表达式进行过滤，如果不涉及到 jsx，则不会转换。

### 动态 modal 场景

在项目中最常见的便是动态 modal 的场景了，此场景下，一般会在 body 下新挂一个 div 作为 mask，同时会根据参数的不同对 div 进行保留，此时 modal 元素的 xpath 便会非常雷同。

比如页面中通过类似 ```Modal.create``` 方法或者静态 modal(其最终也会在 body 上挂载 div 实现)，创建了多个 modal，此时存在两个问题：

1. modal 挂载在 body 下的 mask div 的顺序不同，顺序不同，就导致了同一个 modal 的 xpath 依赖弹窗打开的时机，这会造成非常大的误判

2. modal 挂载在 body 下的 div，结构非常类似，往往内部都是一张背景图 + 一些按钮，而策划运营需要的就是整个 modal 的曝光，圈选时便会指向最外侧的 div，此时误判会非常严重。

针对这一场景，shadow-babel-plugin 增加了 id 作为唯一的判别标准。

完整的 xpath ，会从元素开始，一直收集到最顶层作为 xpath，而增加了 id 后，元素在碰到第一个 带有 id 的元素便会停止，忽略更顶层的结构。

在 modal 场景下，为每一个 modal 动态追加 id 属性，从而保障了弹窗的 xpath 稳定性。

在这一场景中，id 属性来自于代码执行的堆栈位置，故此 id 受代码修改的影响很大，尽可能的保证策划和运营在埋点后减少对代码的调整，或在代码调整后，及时查看对埋点的影响并及时重新选择埋点。

id 的生成是通过静态分析注入了运行时的代码，通过查找代码执行时堆栈所在的代码位置来作为唯一标记，生成代码如下：

```js
(function() {
    const a = new Error().stack.split('\\n');
    const max = a.length <= 4 ? a.length : 4;
    let id = 'shadow-modal-id';
    for (let i = 1; i < max; i++) {
        const b = a[i].split(':');
        id += '-';
        id += b[b.length - 1].replace(')', '');
        id += '-';
        id += b[b.length - 1];
    }
    return id;
}())
```

通过查找函数的执行堆栈位于代码中的位置，来作为唯一的 id 依据，通过这个方法，可以保证动态 modal 场景下，有唯一的 id 做支撑。但是这个方法也强依赖于代码，故需要在代码调整后，及时查看修改埋点。

### 动态组件

jsx 语法本身的灵活性，导致了有些场景下， dom 结构确实变化很大，很难进行区分。比如 动态组件场景，在 h5 中，最常见的便是 tab 场景，示例如下：

```jsx
function App() {
    // 动态组件1
    const Comp = useMemo(() => {
        switch (tabIndex) {
            case 1: return Comp1;
            case 2: return Comp2;
            case 3: return Comp3;
        }
    }, [tabIndex]);

    return (
        <div>
            <Comp />
        </div>
    );
}
```

动态组件还有一些其他的写法，比如：

```jsx
const map = {
    1: Comp1,
    2: Comp2,
    3: Comp3,
}

const Comp = map[type];
```

在这种动态组件写法下，dom 结构的重复性极高，很容易出现 comp1 ~ comp3 被重复命中的场景，同时，还会对其后面的兄弟节点产生影响，这一问题静态分析几乎没有意义，故需要使用者自行追加稳定性 id，比如：

```jsx
// 为 comp1 追加 id 
function Comp1() {
    return (
        <div id="xxxx">
        </div>
    )
}
```

而为了避免对兄弟节点造成干扰，需要保证 comp1 ~ comp3 最外层的 dom 原生 tag 保持一致，才可以避免对后面的兄弟节点带来 xpath 的干扰。

### 小结

这一章节，分析了 逻辑与、三元表达式、动态 modal 以及动态组件的场景。

对于 逻辑与 以及 三元表达式的场景，开发了 babel 插件进行节点替换，能够避免很多 xpath 重复的场景。

但是节点的插入可能会会对原逻辑造成一定的影响，不过好在这些问题并不常遇到，即便遇到，也是开发时便能够发现，问题不算很大。

不过由于插入的节点是 div，为了避免对兄弟节点的影响，组件最外层最好通过 div 来包裹，以减少对兄弟组件的影响。

而对于 动态 modal 的场景，仅仅只能通过自动追加 id 的形式实现，但是 id 的生成强依赖于在代码中的位置，故需要开发者修改代码后及时查看，或自行增加 id 来解决。

至于动态组件的场景，静态分析实在没有好的解决方案，只能开发者自行添加 id 以区分。

## 列表 xpath

对于实际的使用过程中，还存在列表 xpath 如何生成的情况，关于列表元素的判定，在 [可视化埋点平台（三）：元素圈选器实现](https://juejin.cn/post/7184367763123601469)，文章中有提及，而对于 xpath 的处理相对比较简单，只需要模糊 列表 tag 的顺序即可，简单来说，对于如下 dom 结构：

```html
<html>
    <body>
        <div>
            <ul>
                <li>
                    <div>111</div>
                </li>
                <li>
                    <div>111</div>
                </li>
                <li>
                    <div>111</div>
                </li>
            </ul>
        </div>
    </body>
</html>
```

其中，ul 便是列表元素，将其之下的第一级子元素的位置进行模糊即可，生成的 xpath 如下：```html/body/div[1]/ul[1]/li[*]/div[1]```，关键点就在于 ```li[*]```，通过指定 ```*``` ，从而模糊 li 的位置，这样，只需要不断迭代即可获取到所有的子元素。获取元素们的代码如下：

```js
function getElesByXpath(xpath, max = 200) {
    const doc = document;
    const result = doc.evaluate(xpath, doc);
    let item = result;
    const eles = [];
    let count = 0;
    while (item && count < max) {
        item = result.iterateNext();
        if (item) {
            eles.push(item);
        }
        count++;
    }
    return eles;
}
```

实际使用时，可以通过 max 控制选择的元素数量。在做点击匹配时，可以直接比对 xpath 中的 ```*``` 与目标元素的 xpath 即可，并不需要全部获取到列表元素，此处的获取列表元素仅仅用于录入埋点后的反向验证。

## 还有什么

元素圈选浮层也会对 dom 结构造成影响，此处参考 vconsole 将浮层挂载在 html 的最后一个节点下，也就是不在 body 内，足以消除其对 dom 节点的影响。

在解决了 xpath 的稳定性后，本身平台还需要具备一些其他的能力，比如通信层的 iframe, 仅仅支持 iframe 在实际的场景中是不够的，因为 iframe 中往往存在登陆，以及 展示的 dom 结构同 app 内完全不同的情况。

故可视化埋点平台还需要支持直接在手机上打开的场景，这一部分通过 websocket 实现，将原本由 iframe 通信转发处理的信息，转而使用 websocket 进行转发，同时圈选器也支持移动端，即可实现该功能。

除此之外，目前可视化平台对于埋点自定义数据的携带，还存在功能上的缺陷，目前还没有想到非常好的完全不需要人工干预的方案，好在目前策划并没有非常纠结于这一点，毕竟大部分的自定义数据其实都可以通过从库中对数据处理来拿到，有想法的小伙伴可以评论区交流。

另，由于埋点脚本在获取初始埋点信息的时候，需要发送请求，这一步可以在构建的时候通过 webpack-plugin 将项目的埋点信息进行注入，来达到减少请求的优化。

## 最后

至此，整个可视化埋点平台的建设及相关文章将会告一段落。

回头来看，整个可视化埋点系统链路还是比较长，功能点上也比较碎，涉及到 iframe 通信、ws 通信，元素圈选，webpack 插件，babel 插件的开发等等。

从产品角度来讲，个人认为目前在功能上已经覆盖了日常的场景，比如扫码手机圈选、列表圈选，数据离线等功能。大部分的过程都实现了无开发参与的目标。但是自定义埋点数据，目前还是一大痛点，并不能实现无人工下的自定义数据携带。

从技术角度来讲，整个链路涉及一些生僻 dom api，babel 替换节点，webpack 注入 js 等等，其中，元素的圈选逻辑已[独立](https://github.com/yuzai/dom-inspector-pro)，而整个项目的组件间通信，是通过自行定义的消息中心完成 iframe、ws 消息的收发与 react 组件的通信。整体来讲，技术上的细节还是挺多的。

最后，希望本文能够带给有相同需求的同学带来一点帮助吧。

ps：之前有评论区的同学提到，可视化埋点 由于 xpath 不稳定的一些缺陷，可以和低代码平台结合，由低代码平台生成自动生成 id，目测能够大幅提升 xpath 的稳定性，可能是良配，笔者待后续有机会可以和团队做的低代码平台合作看看。
