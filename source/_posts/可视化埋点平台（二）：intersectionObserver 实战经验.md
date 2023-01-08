---
title: 可视化埋点（二）：intersectionObserver 实战经验
date: 2022-07-31 17:50
categories: 原创
---

最近一直在开发可视化埋点系统，其中元素的曝光埋点，就是借助了 [intersectionObserver](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API) 这个原生 api。也是网上推荐度比较高的方案，同时 2022 年，该 api [兼容性](https://caniuse.com/?search=intersection)也已经很高，同时也有 [polyfill](https://www.npmjs.com/package/intersection-observer)，基本上使用无虞。

intersectionObserver 本身 api 非常简单，但是在实际使用的过程中，由于可视化埋点的一些特殊性以及对埋点准确性的要求，还是遇到了一些 dom 变更后的边缘场景，本文便是对这些边缘场景的一个记录及实现背后的一些考虑。

<!--more-->

通常来讲，元素的埋点是开发者主动在元素中直接埋点，而笔者正在开发的可视化埋点，在数据收集侧的工作，是异步从后台获取用户需要的元素 xpath，然后再通过 xpath 寻找到元素，调用 intersectionObserver 的 observe 方法进行监听元素的曝光。所以在进行监听的时机上，都是在元素挂载到 dom 后进行元素的监听，那么初次监听，是否会触发其回调，便关系到曝光埋点的准确性。

同时，可视化埋点也监听页面 dom 的变更，当变更后，需要解除旧元素的监听，同时增加对新元素的监听。那么如果多次监听同一元素，是否会多次触发回调，也影响到曝光的准确性。

为了快速上线第一版，笔者最初的设计方案为：

1. 根据服务端返回的 xpath，寻找到对应的 dom 元素，对元素进行 observe
2. 监听 dom 变更事件，当 dom 发生改变后，重新根据 xpath 寻找 dom 元素，对元素进行再次 observe

在该方案中，存在几个触发时机可能导致的问题：

1. 当监听发生在元素渲染到页面后，首次监听的同时，是否会触发回调（影响到初次曝光的准确性）
2. 多次监听同一 dom 元素，是否会多次触发回调（影响到 dom 变更，多次监听同一元素后，曝光的准确性）

经过实际测验，得到如下结论：

1. 当初次监听元素时，会立即触发一次回调
2. 多次监听同一元素，并不会多次触发回调

在上述逻辑成立的情况下，笔者最初的方案其实是可以正常工作，对于初次曝光，虽然发生在元素渲染到 dom 之后，但是由于会立即触发一次，故初次曝光能够正常上报。而当 dom 发生变更后，消失的元素，虽然没有调用 unobserve，但是由于该元素消失了，并不会影响后续曝光埋点的上报，所以并没有带来大的问题，而 dom 变更后，元素如果依然存在，虽然再次进行了监听，但是多次监听并不影响同一元素，所以其实也没有问题。

对于第一版，上线后也确实能够正常工作，但是对于没有 unobserve 这一点，由于 js 的垃圾回收机制，必须是没有引用后才会销毁，而没有 unobserve，那么内部必然会维护一份监听的元素的列表，保留了已经从 dom 中移除的元素的引用，从而造成内存泄漏。

故需要做一些策略来避免该问题（不然代码也会被吐槽），思路如下：

维护一份 xpath -> 元素的映射，当 dom 发生变更时，遍历所有 xpath 寻找对应的元素，

如果元素同映射中一致，那么表示该元素没有发生变更，此时可以直接忽略，什么都不做。

而如果元素发生变化，那么调用 unobserve 取消旧元素的监听，同时对新元素进行监听即可。

完整的伪代码如下：

```js
// 工具函数命名很清晰，含义不赘述
import { getEleByXpath, getXpathByEle, debounce } from 'utils'; 

class track {
    constructor() {
        /* 
        {
            xpath: '',
            id: ''id
        }
        */
        this.config = {} // 存储远程服务端返回的埋点 xpath 信息
        /*
            [xpath]: el,
        */
        this.map = {} // 维护的 xpath -> el 映射
        this.observer = null; // intersectionObserver 实例
    }
    // 远端获取需要曝光点的元素
    getConfig() {
        return fetch('xxx').then(res => {
            this.config = res;
            this.config.forEach(item => {
                // 初始化 xpath -> el 映射
                this.map[item.xpath] = null;
            })
        });
    }
    // 监听 dom 变更
    addDomtreeMutatorObserver() {
        // 不关心属性变化
        const config = { attributes: false, childList: true, subtree: true };
        // 监听dom变更，消个抖
        const observer = new MutationObserver(debounce(this.observe.bind(this), 200));
        observer.observe(document.body, config);
    }
    // 监听元素曝光
    observe() {
        // 此处可以加个 requestIdleCallback 来增强性能
        this.config.forEach((item) => {
            const targetEl = getEleByXpath(item.xpath);

            // 新旧元素不一致才需要取消旧元素监听，增加新元素监听
            if (targetEl !== this.map[item.xpath]) {
                // 元素存在，就监听
                if (targetEl) {
                    this.observer.observe(targetEl);
                }
                // 取消旧元素的监听
                if (this.map[shadow.xpath]) {
                    this.observer.unobserve(this.map[shadow.xpath]);
                }
                // 更新map中的el
                this.map[shadow.xpath] = targetEl;
            }
            // 一致，则什么都不做
        });
    }
    // 创建 intersectionObserver
    initObserver() {
        const callback = (entries) => {
            entries.forEach((entry) => {
                if (entry.intersectionRatio > 0.2) {
                    const targetXpath = getXpath(entry.target);
                    for (let i = 0; i < this.config.length; i++) {
                        if (this.config[i].xpath === targetXpath) {
                            // xpath一致
                            // 发送曝光埋点
                        }
                    }
                }
            });
        };

        const observer = new IntersectionObserver(callback, {
            threshold: 0.2,
        });

        this.observer = observer;
    }
    
    init() {
        this.getConfig().then(() => {
            // 初始化 intersectionObserver
            this.initObserver();
            // 监听元素
            this.observe();
            // 监听 dom 元素变更
            this.addDomtreeMutatorObserver();
        });
    }
}

```

以上便是精简后的伪代码，其核心的优化点便是维护 xpath -> el 的 map（说实话，一开始笔者其实就想到了 map，但是当时想的是拿 dom 引用作为 key，拿对象做 key 显然是行不通的，就短暂放弃，先上线再说。后来想到用 xpath 做key，才有了这篇文章）。

其实本文的初衷是记录一下 intersectionObserver 的一些边缘情况及结论（其实还对 intersectionObserver 做了其他一些比较傻的测试），但是在文章编写的过程中发现干巴巴的分析，显得实在是无趣，故夹杂了实际的可视化埋点曝光采集的业务场景，希望没有写的很乱。

同时，当笔者彻底理清思路后，发现这优化其实不值一提，只是笔者最初在设计可视化埋点方案时，由于对这些 api 的不熟悉，导致的一些迷迷糊糊的摸爬滚打经验，还望懂的同学别笑。


