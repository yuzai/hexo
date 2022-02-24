---
title: 如何打造轻量级 WebIDE
date: 2022-02-22 22:53
categories: 原创
---

之前在团队中基于 [theia](https://theia-ide.org/)，搭建了一套 CloudIDE（其相关介绍见此[文章](https://zhuanlan.zhihu.com/p/411030285)），其本质是将 IDE 的前端运行在浏览器 or 本地 electron，而文件系统，多语言服务等，运行在远端容器侧，中间通过 rpc 进行通信从而完成整个 IDE 的云端化。

而目前团队在低代码平台的建设中，发现还是需要一些简单的 IDE 场景，如果采用容器化的方案，那么每一个 IDE 可能就需要一个容器，不论是启动速度，还是资源的占用，都不太合适。

团队急需一款轻量级的 IDE 来嵌入低代码平台中，故近期着手开发了一款基于 monaco-editor 的轻量级且不依赖后端容器的 IDE。

<!-- more -->

对于依赖容器化技术，能够提供完整终端能力的 IDE，在本文中，称之为 CloudIDE，而仅仅依赖浏览器能力的 IDE，本文称之为 WebIDE。本文想要分享的 IDE 属于后者。

得益于 [monaco-editor](https://microsoft.github.io/monaco-editor/) 的强大，使用 monaco-editor 去搭建一个简单的 WebIDE 非常容易，但是要把多文件支持、[ESLint](https://eslint.org/)、[Prettier](https://prettier.io/)、代码补全等功能加进去，并不是一件容易的事情。

本文意在分享我在建设 WebIDE 中学到的一些经验，以及一些解决方案，希望能够帮助到有同样需求的同学。同时，这不是一篇手把手的文章，仅仅是介绍一些决策的思路及示例代码。完整的代码见 [github](https://github.com/yuzai/base-editor)，同时也搭建了一个 [demo](https://blog.maxiaobo.com.cn/base-editor/build/index.html) 可以体验(demo 依赖不少静态文件，部署在 github pages 上，访问速度过慢可能无法正常加载，可以clone后run dev查看，移动端也建议通过chrome打开)，也提供了一个 [npm 组件](https://www.npmjs.com/package/yuzai-base-editor)可以当作 react 组件直接使用。

## 引入monaco-editor

引入 monaco-editor 的方式主要是两种，amd 或者 esm。

两者接入方式都比较容易，我均有尝试。

相对来讲，起初更偏向于 esm 方式，但是由于 [issue](https://github.com/microsoft/monaco-editor/issues/2671) 问题，导致打包后，在当前项目中可以正常使用，但是当把它作为 npm 包发布后，他人使用时，打包会出错。

故最终采取第一种方式，通过动态插入 script 标签来引入 monaco-editor，项目中通过定时器轮 window.monaco 是否存在来判断 monaco-editor 是否加载完成，如未完成，提供一个 loading 进行等待。

## 多文件支持

monaco-editor 的官方例子中，基本都是单文件的处理，不过多文件处理也非常简单，本文在此处仅做简单的介绍。

多文件处理主要涉及到的就是 [monaco.editor.create](https://microsoft.github.io/monaco-editor/api/modules/monaco.editor.html#create) 以及 [monaco.editor.createModel](https://microsoft.github.io/monaco-editor/api/modules/monaco.editor.html#createModel) 两个api。

其中，createModel 就是多文件处理的核心 api。根据文件路径创建不同的 model，在需要切换时，通过调用 [editor.setModel](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.IStandaloneCodeEditor.html#setModel) 即可实现多文件的切换

创建多文件并切换的一般的伪代码如下：

```js
const files = {
    '/test.js': 'xxx',
    '/app/test.js': 'xxx2',
}

const editor = monaco.editor.create(domNode, {
    ...options,
    model: null, // 此处model设为null，是阻止默认创建的空model
});

Object.keys(files).forEach((path) =>
    monaco.editor.createModel(
        files[path],
        'javascript',
        new monaco.Uri().with({ path })
    )
);

function openFile(path) {
    const model = monaco.editor.getModels().find(model => model.uri.path === path);
    editor.setModel(model);
}

openFile('/test.js');
```

通过再编写一定的 ui 代码，可以非常轻易的实现多文件的切换。

### 保留切换之前状态

通过上述方法，可以实现多文件切换，但是在文件切换前后，会发现鼠标滚动的位置，文字的选中态均发生丢失的问题。

此时可以通过创建一个 map 来存储不同文件在切换前的状态，核心代码如下：

```js
const editorStatus = new Map();
const preFilePath = '';

const editor = monaco.editor.create(domNode, {
    ...options,
    model: null,
});

function openFile(path) {
    const model = monaco.editor
        .getModels()
        .find(model => model.uri.path === path);
        
    if (path !== preFilePath) {
        // 储存上一个path的编辑器的状态
        editorStatus.set(preFilePath, editor.saveViewState());
    }
    // 切换到新的model
    editor.setModel(model);
    const editorState = editorStates.get(path);
    if (editorState) {
        // 恢复编辑器的状态
        editor.restoreViewState(editorState);
    }
    // 聚焦编辑器
    editor.focus();
    preFilePath = path;
}
```

核心便是借助editor实例的 [saveViewState](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.IStandaloneCodeEditor.html#saveViewState) 方法实现编辑器状态的存储，通过 [restoreViewState](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.IStandaloneCodeEditor.html#restoreViewState) 方法进行恢复。

### go to definition

monaco-editor 作为一款优秀的编辑器，其本身是能够感知到其他model的存在，并进行相关代码补全的提示。但是我们最常用的 cmd + 点击，默认是不能够跳转的。虽然 hover 上去能看到相关信息，但是默认不能跳转。

这一条也算是比较常见的问题了，详细的原因及解决方案可以查看此 [issue](https://github.com/microsoft/monaco-editor/issues/2000)。

简单来说，库本身没有实现这个打开，是因为如果允许跳转，那么用户没有很明显的方法可以再跳转回去。

实际中，可以通过覆盖 openCodeEditor 的方式来解决，在没有找到跳转结果的情况下，自己实现 model 切换

```js
    const editorService = editor._codeEditorService;
    const openEditorBase = editorService.openCodeEditor.bind(editorService);
    editorService.openCodeEditor = async (input, source) =>  {
        const result = await openEditorBase(input, source);
        if (result === null) {
            const fullPath = input.resource.path;
            // 跳转到对应的model
            source.setModel(monaco.editor.getModel(input.resource));
            // 此处还可以自行添加文件选中态等处理
        
            // 设置选中区以及聚焦的行数
            source.setSelection(input.options.selection);
            source.revealLine(input.options.selection.startLineNumber);
        }
        return result; // always return the base result
    };
```

### 受控

在实际编写 react 组件中，往往还需要对文件内容进行受控的操作，这就需要编辑器在内容变化时通知外界，同时也允许外界直接修改文本内容。

先说内容变化的监听，monaco-editor 的每个 model 都提供了 [onDidChangeContent](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.ITextModel.html#onDidChangeContent) 这样的方法来监听文件改变，可以继续改造我们的 openFile 函数。

```js

let listener = null;

function openFile(path) {
    const model = monaco.editor
        .getModels()
        .find(model => model.uri.path === path);
        
    if (path !== preFilePath) {
        // 储存上一个path的编辑器的状态
        editorStatus.set(preFilePath, editor.saveViewState());
    }
    // 切换到新的model
    editor.setModel(model);
    const editorState = editorStates.get(path);
    if (editorState) {
        // 恢复编辑器的状态
        editor.restoreViewState(editorState);
    }
    // 聚焦编辑器
    editor.focus();
    preFilePath = path;
    
    if (listener) {
        // 取消上一次的监听
        listener.dispose();
    }
    
    // 监听文件的变更
    listener = model.onDidChangeContent(() => {
        const v = model.getValue();
        if (props.onChange) {
            props.onChange({
                value: v,
                path,
            })
        }
    })
}
```

解决了内部改动对外界的通知，外界想要直接修改文件的值，可以直接通过 [model.setValue](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.ITextModel.html#setValue) 进行修改，但是这样直接操作，就会丢失编辑器 undo 的堆栈，想要保留 undo，可以通过 [model.pushEditOperations](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.ITextModel.html#pushEditOperations) 来实现替换，具体代码如下：

```js
function updateModel(path, value) {
    let model = monaco.editor.getModels().find(model => model.uri.path === path);
    
    if (model && model.getValue() !== value) {
        // 通过该方法，可以实现undo堆栈的保留
        model.pushEditOperations(
            [],
            [
                {
                    range: model.getFullModelRange(),
                    text: value
                }
            ],
            () => {},
        )
    }
}
```

### 小结

通过上述的 monaco-editor 提供的 api，基本就可以完成整个多文件的支持。

当然，具体到实现还有挺多的工作，文件树列表，顶部 tab，未保存态，文件的导航等等。不过这部分属于我们大部分前端的日常工作，工作量虽然不小但是实现起来并不复杂，此处不再赘述。

## ESLint支持

monaco-editor 本身是有语法分析的，但是自带的仅仅只有语法错误的检查，并没有代码风格的检查，当然，也不应该有代码风格的检查。

作为一名现代的前端开发程序员，基本上每个项目都会有 ESLint 的配置，虽然 WebIDE 是一个精简版的，但是 ESLint 还是必不可少。

ESLint 的原理，是遍历语法树然后检验，其核心的 Linter，是不依赖 node 环境的，并且官方也进行了单独的打包输出，具体可以通过 clone[官方代码](https://github.com/ESLint/ESLint) 后，执行 npm run webpack 拿到核心的打包后的 ESLint.js。其本质是对 [linter.js](https://github.com/ESLint/ESLint/blob/main/lib/linter/linter.js) 文件的打包。

同时官方也基于该打包产物，提供了 ESLint 的官方 [demo](https://ESLint.org/demo)。

该 linter 的使用方法如下：

```js
import { Linter } from 'path/to/bundled/ESLint.js';

const linter = new Linter();

// 定义新增的规则，比如react/hooks, react特殊的一些规则
// linter中已经定义了包含了ESLint的所有基本规则，此处更多的是一些插件的规则的定义。
linter.defineRule(ruleName, ruleImpl)；

linter.verify(text, {
    rules: {
        'some rules you want': 'off or warn',
    },
    settings: {},
    parserOptions: {},
    env: {},
})
```

如果只使用上述 linter 提供的方法，存在几个问题：

1. 规则太多，一一编写太累且不一定符合团队规范
2. 一些插件的规则无法使用，比如 react 项目强依赖的 ESLint-plugin-react, react-hooks的规则。

故还需要进行一些针对性的定制。

在日常的 react 项目中，基本上团队都是基于 [ESLint-config-airbnb](https://www.npmjs.com/package/ESLint-config-airbnb) 规则配置好大部分的 rules，然后再对部分规则根据团队进行适配。

通过阅读 ESLint-config-airbnd 的代码，其做了两部分的工作：

1. 对 ESLint 的自带的大部分规则进行了配置
2. 对 ESLint 的插件，ESLint-plugin-react, ESLint-plugin-react-hooks 的规则，也进行了配置。

而 ESLint-plugin-react, ESLint-plugin-react-hooks，核心是新增了一些针对 react 及 hooks 的规则。

那么其实解决方案如下：
1. 使用打包后的 ESLint.js 导出的 linter 类
2. 借助其 defineRule 的方法，对 react, react/hooks 的规则进行增加
3. 合并 airbnb 的规则，作为各种规则的 config 合集备用
4. 调用 linter.verify 方法，配合3生成的 airbnb 规则，即可实现完整的 ESLint 验证。

通过上述方法，可以生成一个满足日常使用的 linter 及满足 react 项目使用的 ruleConfig。这一部分由于相对独立，我将其单独放在了一个 github 仓库 [yuzai/ESLint-browser](https://github.com/yuzai/ESLint-browser)，可以酌情参考使用，也可根据团队现状修改使用。

下一步就是调用的时机，在每次代码变更时，频繁同步执行ESLint的verify可能会带来ui的卡顿，在此，我采取方案是：

1. 通过 webworker 执行 linter.verify
2. 在 [model.onDidChangeContent](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.ITextModel.html#onDidChangeContent) 中通知 worker 进行执行。并通过消抖来减少执行频率
3. 通过 [model.getVersionId](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.ITextModel.html#getVersionId)，拿到当前 id，来避免延迟过久导致结果对不上的问题

主进程核心的代码如下：

```js
// 监听ESLint web worker 的返回
worker.onmessage = function (event) {
    const { markers, version } = event.data;
    const model = editor.getModel();
    // 判断当前model的versionId与请求时是否一致
    if (model && model.getVersionId() === version) {
        window.monaco.editor.setModelMarkers(model, 'ESLint', markers);
    }
};

let timer = null;
// model内容变更时通知ESLint worker
model.onDidChangeContent(() => {
    if (timer) clearTimeout(timer);
    timer = setTimeout(() => {
        timer = null;
        worker.postMessage({
            code: model.getValue(),
            // 发起通知时携带versionId
            version: model.getVersionId(),
            path,
        });
    }, 500);
});
```

worker 内核心代码如下：

```js
// 引入ESLint，内部结构如下：
/*
{
    esLinter, // 已经实例化，并且补充了react, react/hooks规则定义的实例
    // 合并了airbnb-config的规则配置
    config: {
        rules,
        parserOptions: {
            ecmaVersion: 'latest',
            sourceType: 'module',
            ecmaFeatures: {
                jsx: true
            }
        },
        env: {
            browser: true
        },
    }
}
*/
importScripts('path/to/bundled/ESLint/and/ESLint-airbnbconfig.js');

// 更详细的config, 参考ESLint linter源码中关于config的定义: https://github.com/ESLint/ESLint/blob/main/lib/linter/linter.js#L1441
const config = {
    ...self.linter.config,
    rules: {
        ...self.linter.config.rules,
        // 可以自定义覆盖原本的rules
    },
    settings: {},
}

// monaco的定义可以参考：https://microsoft.github.io/monaco-editor/api/enums/monaco.MarkerSeverity.html
const severityMap = {
    2: 8, // 2 for ESLint is error
    1: 4, // 1 for ESLint is warning
}

self.addEventListener('message', function (e) {
    const { code, version, path } = e.data;
    const extName = getExtName(path);
    // 对于非js, jsx代码，不做校验
    if (['js', 'jsx'].indexOf(extName) === -1) {
        self.postMessage({ markers: [], version });
        return;
    }
    const errs = self.linter.esLinter.verify(code, config);
    const markers = errs.map(err => ({
        code: {
            value: err.ruleId,
            target: ruleDefines.get(err.ruleId).meta.docs.url,
        },
        startLineNumber: err.line,
        endLineNumber: err.endLine,
        startColumn: err.column,
        endColumn: err.endColumn,
        message: err.message,
        // 设置错误的等级，此处ESLint与monaco的存在差异，做一层映射
        severity: severityMap[err.severity],
        source: 'ESLint',
    }));
    // 发回主进程
    self.postMessage({ markers, version });
});
```

主进程监听文本变化，消抖后传递给 worker 进行 linter，同时携带 versionId 作为返回的比对标记，linter 验证后将 markers 返回给主进程，主进程设置 markers。

以上，便是整个 ESLint 的完整流程。

当然，由于时间关系，目前只处理了 js，jsx，并未对ts,tsx文件进行处理。支持 ts 需要调用 linter 的 defineParser 修改语法树的解析器，相对来讲稍微麻烦，目前还未做尝试，后续有动静会在 github 仓库 [yuzai/ESLint-browser](https://github.com/yuzai/ESLint-browser) 进行修改同步。

## Prettier支持

相比于 ESLint, Prettier 官方支持浏览器，其用法见此[官方页面](https://Prettier.io/docs/en/browser.html), 支持 amd, commonjs, es modules 的用法，非常方便。

其使用方法的核心就是调用不同的 parser，去解析不同的文件，在我当前的场景下，使用到了以下几个 parser:

1. babel: 处理 js
2. html: 处理 html
3. postcss: 用来处理 css, less, scss
4. typescript: 处理 ts

其区别可以参考[官方文档](https://Prettier.io/docs/en/options.html#parser)，此处不赘述。一个非常简单的使用代码如下：

```js
const text = Prettier.format(model.getValue(), {
    // 指定文件路径
    filepath: model.uri.path,
    // parser集合
    plugins: PrettierPlugins,
    // 更多的options见：https://Prettier.io/docs/en/options.html
    singleQuote: true,
    tabWidth: 4,
});
```

在上述配置中，有一个配置需要注意：filepath。

该配置是用来来告知 Prettier 当前是哪种文件，需要调用什么解析器进行处理。在当前WebIDE场景下，将文件路径传递即可，当然，也可以自行根据文件后缀计算后使用 parser 字段指定用哪个解析器。

在和 monaco-editor 结合时，需要监听 cmd + s 快捷键来实现保存时，便进行格式化代码。

考虑到 monaco-editor 本身也提供了格式化的指令，可以通过⇧ + ⌥ + F进行格式化。

故相比于 cmd + s 时，执行自定义的函数，不如直接覆盖掉自带的格式化指令，在 cmd + s 时直接执行指令来完成格式化来的优雅。

覆盖主要通过 [languages.registerDocumentFormattingEditProvider](https://microsoft.github.io/monaco-editor/api/modules/monaco.languages.html#registerDocumentFormattingEditProvider) 方法，具体用法如下：

```js
function provideDocumentFormattingEdits(model: any) {
    const p = window.require('Prettier');
    const text = p.Prettier.format(model.getValue(), {
        filepath: model.uri.path,
        plugins: p.PrettierPlugins,
        singleQuote: true,
        tabWidth: 4,
    });

    return [
        {
            range: model.getFullModelRange(),
            text,
        },
    ];
}

monaco.languages.registerDocumentFormattingEditProvider('javascript', {
    provideDocumentFormattingEdits
});
monaco.languages.registerDocumentFormattingEditProvider('css', {
    provideDocumentFormattingEdits
});
monaco.languages.registerDocumentFormattingEditProvider('less', {
    provideDocumentFormattingEdits
});
```

上述代码中 window.require，是 amd 的方式，由于本文在选择引入 monaco-editor 时，采用了 amd 的方式，所以此处 Prettier 也顺带采用了 amd 的方式，并从 cdn 引入来减少包的体积，具体代码如下：

```js
window.define('Prettier', [
        'https://unpkg.com/Prettier@2.5.1/standalone.js',
        'https://unpkg.com/Prettier@2.5.1/parser-babel.js',
        'https://unpkg.com/Prettier@2.5.1/parser-html.js',
        'https://unpkg.com/Prettier@2.5.1/parser-postcss.js',
        'https://unpkg.com/Prettier@2.5.1/parser-typescript.js'
    ], (Prettier: any, ...args: any[]) => {
    const PrettierPlugins = {
        babel: args[0],
        html: args[1],
        postcss: args[2],
        typescript: args[3],
    }
    return {
        Prettier,
        PrettierPlugins,
    }
});
```

在完成 Prettier 的引入，提供格式化的 provider 之后，此时，执行⇧ + ⌥ + F即可实现格式化，最后一步便是在用户 cmd + s 时执行该指令即可，使用 [editor.getAction](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.editor.IStandaloneCodeEditor.html#getAction) 方法即可，伪代码如下：

```js
// editor为create方法创建的editor实例
editor.getAction('editor.action.formatDocument').run()
```

至此，整个 Prettier 的流程便已完成，整理如下：

1. amd 方式引入
2. monaco.languages.registerDocumentFormattingEditProvider 修改 monaco 默认的格式化代码方法
3. editor.getAction('editor.action.formatDocument').run() 执行格式化

## 代码补全

monaco-editor 本身已经具备了常见的代码补全，比如 window 变量，dom，css 属性等。但是并未提供 node_modules 中的代码补全，比如最常见的 react，没有提示，体验会差很多。

经过调研，monaco-editor 可以提供代码提示的入口至少有两个 api：

1. [registerCompletionItemProvider](https://microsoft.github.io/monaco-editor/api/modules/monaco.languages.html#registerCompletionItemProvider)，需要自定义触发规则及内容
2. [addExtraLib](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.languages.typescript.LanguageServiceDefaults.html#addExtraLib)，通过添加 index.d.ts，使得在自动输入的时候，提供由 index.d.ts 解析出来的变量进行自动补全。

第一种方案网上的文章较多，但是对于实际的需求，导入 react, react-dom，如果采用此种方案，就需要自行完成对 index.d.ts 的解析，同时输出类型定义方案，在实际使用时非常繁琐，不利于后期维护。

第二种方案比较隐蔽，也是偶然发现的，经过验证，stackbliz 就是用的这种方案。但是 stackbliz 只支持 ts 的跳转及代码补全。

经过测试，只需要同时在 ts 中的 javascriptDefaults 及 typescriptDefaults 中使用 addExtraLib 即可实现代码补全。

体验及成本远远优于方案一。

方案二的问题在于未知第三方包的解析，目前看，stackbliz 也仅仅只是对直系 npm 依赖进行了 .d.ts 的解析。相关依赖并无后续进行。实际也能理解，在不经过二次解析 .d.ts 的情况下，是不会对二次引入的依赖进行解析。故当前版本也不做 index.d.ts 的解析，仅提供直接依赖的代码补全及跳转。不过 ts 本身提供了 [types分析](https://www.npmjs.com/package/@typescript/ata) 的能力，后期接入会在 github 中同步。

故最终使用方案二，内置 react, react-dom 的类型定义，暂不做二次依赖的包解析。相关伪代码如下：

```js
window.monaco.languages.typescript.javascriptDefaults.addExtraLib(
    'content of react/index.d.ts',
    'music:/node_modules/@types/react/index.d.ts'
);
```

同时，通过 [addExtraLib](https://microsoft.github.io/monaco-editor/api/interfaces/monaco.languages.typescript.LanguageServiceDefaults.html#addExtraLib) 增加的 .d.ts 的定义，本身也会自动创建一个 model，借助前文所描述的 openCodeEditor 的覆盖方案，可以顺带实现 cmd + click 打开 index.d.ts 的需求，体验更佳。

## 主题替换

此处由于 monaco-editor 同 vscode 使用的解析器不同，导致无法直接使用 vscode 自带的主题，当然也有办法，具体可以参考[手把手教你实现在 Monaco Editor 中使用 VSCode 主题](https://segmentfault.com/a/1190000040746839)文章，可以直接使用 vscode 主题，我采取的也是这篇文章的方案，本身已经很详细了，我就不在这里做重复劳动了。

## 预览沙箱

这一部分由于公司内有[基于 codesandbox 的沙箱方案](https://zhuanlan.zhihu.com/p/264990261)，故实际在公司内部落地时，本文所述的 WebIDE 仅仅作为一个代码编辑与展示的方案，实际预览时走的是基于 codesandbox 的沙箱渲染方案。

除此之外，得益于浏览器对 modules 的天然支持，也尝试过不打包，直接借助浏览器的 modules 支持，通过 service worker 中对 jsx, less文件处理后做预览的方案。该方案应对简单场景可以直接使用，但是实际场景中，存在对 node_modules 的文件需要特殊处理的情况，没有做更深入的尝试。

这一部分我也没有做更多深入尝试，故不做赘述。

## 最后

本文详细介绍了基于 monaco-editor 打造一款轻量级的 WebIDE 的必备环节。

整体来讲，monaco-editor 本身的能力比较完善，借助其基础 api 再加上适当的 ui 代码，可以非常快速的构建出一款可用的 WebIDE。但是要做好，并不是那么容易。

本文在介绍了相关 api 的基础上，对多文件的细节处理、ESLint 浏览器化方案、Prettier 与 monaco 的贴合及代码补全的支持上做了更为详细的介绍。希望能够对有同样需求的同学起到帮助的作用。

最后，[源代码](https://github.com/yuzai/base-editor)奉上，觉得不错的话，帮忙点个赞 or 小星星就更好啦。

## 参考文章

1. [Building a code editor with Monaco
](https://blog.expo.dev/building-a-code-editor-with-monaco-f84b3a06deaf)

2. [手把手教你实现在 Monaco Editor 中使用 VSCode 主题](https://segmentfault.com/a/1190000040746839)