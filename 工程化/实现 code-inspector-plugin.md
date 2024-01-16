# *code-inspector-plugin*	



## 一、前言

在日常开发中，开发人员经常需要去调整页面。不幸的是，要调整的部分刚好不是自己写的部分，或者是自己写的但是时间太久，记不清在什么位置。对于一些大型项目，由于文件数量多、层级深、代码行数多，定位源码往往会花费大量的时间和精力。

不知道是不是很多人会这么去找源文件：

1. 打开 F12，光标选中要修改部分的某个元素。
   ![image-20240115143311269](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115143311269.png)

2. 获取选中元素的一些信息，比如 `class`、`id`、文字内容等等，然后去编辑器中搜索关键字。
   ![image-20240115143340128](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115143340128.png)

当搜索结果只有一项时，我们应该感到庆幸，因为毫无疑问这里就是源代码的位置。

但是下面这种情况应该如何应对？

![image-20240115143417408](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115143417408.png)

![image-20240115143438234](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115143438234.png)

有人可能会说，页面元素的 `id` 是唯一的，那只要有 `id`就能找到唯一的位置。包括我们平时在查的时候，也会尽量去找有唯一代表性的内容，这种思考方式没有什么大问题。但是，我们要知道的是，不管是 `id`、`class`、文本还是页面上其它乱七八糟的东西，在源码里面都只是字符串的形式。只要是字符串就有重复的可能性，谁能保证 `id` 对应的字符串就不会被 `class`使用，甚至我们都无法保证某个 `id` 是唯一的。

回想刚才通过人为的方式定位源码的两步骤里面，哪一步是最麻烦的？

毫无疑问，肯定是第二步。那能不能省略第二步，或者说让程序自动执行第二步，并且更加精确地定位到源码？答案是肯定的。

这里就要介绍一下[`code-inspector-plugin`](https://github.com/zh-lx/code-inspector/blob/main/docs/README-ZH.md) 这个插件，它是一个提效插件，开发人员只需点击页面元素即可自动定位到编辑器中源码的位置。并且，它支持多种 IDE、 web 框架以及编译器。

![image-20240115150318489](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115150318489.png)



## 二、原理

抛开这个插件不谈，如果我们要实现这个功能，应该怎么去做？要解决哪些问题？这些问题的解决方案就是实现原理。

需求很简单：点击页面元素，自动定位到编辑器中源代码的位置。

![image-20240115152927032](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115152927032.png)

这个需求涉及到的需要解决的问题点：

- 如何自动打开编辑器并定位到源码位置
- 点击元素如何通知编辑器
- 如何知道元素对应的源代码位置
- 如何绑定元素和源代码位置的关系
- 如何不污染页面



### 1. 如何自动打开编辑器并定位到源码位置

最常用的打开编辑器的方式有两种。第一种，直接双击编辑器客户端启动。第二种，在终端执行命令启动，比如在 mac 系统中，可以通过 `code .`的方式启动编辑器。

![image-20240115161911293](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115161911293.png)

我们可以把命令中的 `.`理解为 `./`， 此时编辑器打开的就是终端命令执行所在的当前文件目录。

我们也可以先打开某个项目目录，然后在该目录下执行终端命令 `code .`，这样就会打开当前项目目录。

既然通过 `code .` 的方式可以让编辑器打开当前目录，那么能不能把 `.`换成具体的目录地址，然后打开对应目录？显然也是可以的。

![image-20240115161900826](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115161900826.png)

打开具体文件和打开目录是同一个道理。

![image-20240115161849308](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115161849308.png)

只不过在打开具体文件时并不会打开其所在的整个项目目录，只会打开单独的文件。如果编辑器已经打开了该文件对应的项目，则会跳转到对应的编辑器中去打开这个文件。

现在文件已经能打开了，那有办法让光标定位到文件中具体的位置吗？显然也是可以的，`code -g` 或 `code --goto` 是一个特殊的选项，它允许你打开一个文件并直接跳转到指定的行和列。

![image-20240115162639275](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115162639275.png)

那么自动打开编辑器并定位到源代码位置就非常简单了，只需要通过 `node` 启动一个子进程去执行终端命令 `code -g 文件路径:行:列` 即可。

```ts
import child_process from 'child_process';
child_process.spawn('code', ['-g', 文件名 + ':' + 行 + ':' + 列], { stdio: 'inherit' });
```



### 2. 点击元素如何通知编辑器

我们知道，元素是在网页端，而编辑器是在客户端，网页端是无法操作本地程序的，那它们之间应该如何产生联系？

答案是：本地 `http`服务。

通过 `node`进程执行终端命令是在后台，而元素的点击操作在前端，它们之间唯一的通信方式就接口请求。点击页面元素时可以发送一个本地 `localhost`请求，本地 `http` 服务在接收到请求之后再通过子进程执行终端命令。

![image-20240115164631577](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115164631577.png)



### 3. 如何知道元素对应的源代码位置

这部分如果有学习过编译相关的知识的话应该能知道如何获取代码的位置信息。比如，在 `Vue3`中有模板编译阶段，在编译阶段会将 `template` 模版字符串转换成 `render`函数。

编译涉及到三个阶段：解析（`parse`）、转换（`transform`）、生成（`generate`）。

其中 `parse`阶段会将模版字符串解析成 `AST`抽象语法树，语法树内部每一个节点都包含相关的位置信息。

![image-20240115172145785](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240115172145785.png)

下面是一个线上解析字符串[生成 `AST`的网站](https://astexplorer.net/)，里面可以选择各种语法的解析。

![image-20240116143411724](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240116143411724.png)

`transform` 阶段则会根据生成的 `AST` 做进一步代码转换优化处理。`generate` 阶段则是最终生成 `render`函数的过程。

在这三个阶段中，我们能获取到源代码位置信息的阶段就是 `transform` 阶段 ，在这个阶段我们可以利用 `AST`获取源代码对应的位置信息。



### 4. 如何绑定元素和源代码位置的关系

将元素和位置信息绑定意味着需要建立两者的映射关系，一个比较简单的实现方式就是将位置信息作为某个属性值绑定到元素上。比如可以定义个额外的元素属性 `code-inspector-path`，值是对应的位置信息字符串。

既然要进行绑定，那就需要处理源代码字符串。作为开发者，我们不可能在每个源文件中的每个元素身上手动去加属性和位置信息。最合理的方式还是借助编译阶段，将属性注入到代码字符串中。

上面也提到了，编译阶段中的 `transform`阶段是可以进行代码转换的，而代码的位置信息也是在这个阶段获取的，因此，我们同样可以在这个阶段进行绑定操作。



### 5. 如何不污染页面

如果在每个元素上额外绑定一个点击事件用作源代码定位，势必会影响到页面正常开发，这种方式是侵入性的。

要想不侵入已有元素，那就需要额外提供一个可以触发该功能的元素或者组件。这里可以考虑使用遮罩的方式，当我们将鼠标悬浮在某个元素上时可以将遮罩显示出来，然后点击遮罩触发定位功能。

不过这里还要思考以下几个点：

- 遮罩显示时机：在不需要定位源码的情况下，即使将鼠标放到元素上也不应该让遮罩显示出来。这里可以考虑结合键盘事件，当按下某个特殊的键位后再将鼠标移动到元素上才能显示遮罩，或者添加一个额外的开关元素，开启开关后，鼠标放到元素上才有遮罩效果。
- 遮罩样式和逻辑的隔离：由于遮罩或者开关元素是需要额外注入的，它本身并不属于页面开发需求。因此，这个元素的功能和样式不能影响原页面，这里可以考虑使用 `web component` 来实现。
- 元素注入：这里也可以像属性注入一样，在构建阶段将创建遮罩元素的代码字符串注入到编译后的文件中，当页面请求到文件后会执行注册元素的逻辑。



最重要的一点，该功能仅支持开发环境，不能参与到生成环境中。



## 三、简单实现

要编写插件的话首先得确定项目所使用的构建工具，不同的构建工具有不同的插件编写方式，这里以 `vite`为例进行 `vite`插件的编写。



### 1. 定义插件基本结构

`vite`插件一般是一个工厂函数，因为可能需要传入额外参数。工厂函数需要返回插件配置对象，对象中包含插件名 `name`以及各种钩子函数。上面提到，我们需要在 `transform`阶段对代码字符串进行处理，因此需要定义 `transform`钩子。

```typescript
// CodeInspectorPlugin.ts
export const MyCodeInspectorPlugin = () => {
  return {
    name: 'vite-code-inspector-plugin', // 插件名称
    enforce: 'pre', // 插件执行顺序，pre 表示优先执行
    async transform(code, id) { // 转换阶段的钩子函数
      // 转换逻辑
    }
  }
}
```



### 2. 实现元素属性以及位置信息注入

`transform`钩子接收两个参数：

- *code*：当前处理的文件内容，也就是代码字符串
- *id*：文件路径

这里需要对自身项目中的 `vue`文件进行处理，其它文件直接跳过。

```typescript
// CodeInspectorPlugin.ts
import addContent from './addContent';

const CodeInspectorPlugin = () => {
  return {
    name: 'vite-code-inspector-plugin',
    transform(code, id) {
      // 排除 node_modules 中的文件
      if (id.match('node_modules')) return code;
      const isVue = /\.vue$/.test(id);
      // 处理 vue 文件
      if (isVue) {
        // 转换逻辑
        code = addContent(code, id);
      }
      return code;
    }
  }
}
```

完善注入功能 `addContent`：

```typescript
// addContent.ts
import { parse, transform } from '@vue/compiler-dom';
import MagicString from 'magic-string';

const attrName = 'code-inspector-path';

// 代码注入
export default (code, id) => {
  // 不需要处理的内置元素
  const escapeTags = [
    'style',
    'script',
    'template',
  ];
  const s = new MagicString(code);
  // 生成 ast
  const ast = parse(code, {
    comments: true // 将注释也添加到 ast 中
  });
  transform(ast, {
    nodeTransforms: [(node) => {
      // node.type === 1 表示元素节点
      if (node.type === 1 && !node.loc.source.includes(attrName) && !escapeTags.includes(node.tag)) {
        // 获取需要注入的位置
        const insertPosition = node.loc.start.offset + node.tag.length + 1;
        // 需要注入的位置信息
        const content = ` ${attrName}="${id}:${node.loc.start.line}:${node.loc.start.column}:${node.tag}"`;
        // 借助 MagicString 在指定位置插入字符串
        s.prependLeft(insertPosition, content);
      }
    }]
  });
  return s.toString();
}
```

这里需要注意插件执行的优先级，`vite` 内置了很多核心插件。默认情况下，`vite`会优先执行核心插件。

这里就涉及到一个插件处理顺序问题：

1. 执行核心插件，编译源文件 —> 在编译后的代码中注入属性字符串
2. 在源代码中注入属性字符串 —> 执行核心插件，编译源文件

这里抛出一个思考题：现在默认以第一种方式执行，但是它是有问题的，为什么？



要想改变插件执行的顺序就要使用 [`enforce`](https://cn.vitejs.dev/guide/using-plugins.html#enforcing-plugin-ordering) 这个属性，`pre` 表示在核心插件之前执行当前插件，

```typescript
// CodeInspectorPlugin.ts
import addContent from './addContent';

const CodeInspectorPlugin = () => {
  return {
    name: 'vite-code-inspector-plugin',
    enforce: 'pre', // 优先执行
    transform(code, id) {
      // 排除 node_modules 中的文件
      if (id.match('node_modules')) return code;
      const isVue = /\.vue$/.test(id);
      // 处理 vue 文件
      if (isVue) {
        // 转换逻辑
        code = addContent(code, id);
      }
      return code;
    }
  }
}
```

至此，属性的注入就完成了，可以看一下页面效果。

![image-20240116162813774](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240116162813774.png)



### 3. 开启本地服务

这里只需要借助 `node` 中的 `http`模块。

```typescript
// httpServer.ts
import http from 'http';

export const startServer = () => {
  const server = http.createServer((req, res) => {
    // 跨域处理
    res.writeHead(200, {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': '*',
      'Access-Control-Allow-Headers': '*',
      'Access-Control-Allow-Private-Network': 'true',
    });
    res.end('ok');
  });
  server.listen('4399');
};
```

```typescript
// CodeInspectorPlugin.ts
import addContent from './addContent';
import startServer from './httpServer';

const server = {
  start: false
}

export const CodeInspectorPlugin = () => {
  return {
    name: 'vite-code-inspector-plugin',
    enforce: 'pre',
    async transform(code, id) {
      if (!server.start) {
        // 启动服务
        await startServer();
        server.start = true;
      }
      // 排除 node_modules 中的文件
      if (id.match('node_modules')) return code;
      const isVue = /\.vue$/.test(id);
      if (isVue) {
        // 转换逻辑
        code = addContent(code, id)
      }
      return code;
    }
  }
}
```



### 4. 实现遮罩组件

遮罩组件利用 `web component`来编写，这里推荐一个库 `lit`。

`lit` 是一个用于构建 Web 应用程序的 JavaScript 库。它提供了一种简单、高效的方式来创建和更新 DOM。`lit` 的主要特点是它的轻量级和高性能，以及它的模板语法，这使得创建复杂的 UI 变得更加简单。

`LitElement` 是 `lit` 库的一部分，它是一个基础类，用于创建 Web 组件。`LitElement` 使得创建自定义元素变得更加简单，它提供了一种声明式的方式来定义和渲染 DOM，以及响应属性和状态的变化。使用 `LitElement`，你可以创建可重用、易于维护的 Web 组件。

```typescript
// code-inspector-component.ts
import { LitElement, css, html } from 'lit';
import { state } from 'lit/decorators.js';
import { styleMap } from 'lit/directives/style-map.js';

export class CodeInspectorComponent extends LitElement {
  @state()
  element = { name: '', line: 0, column: 0, path: '' }; // 选中节点信息
  // @state 声明的属性会自动触发更新
  @state()
  show = false; // 是否显示遮罩
  @state()
  position = {
    top: 0,
    right: 0,
    bottom: 0,
    left: 0,
    padding: {
      top: 0,
      right: 0,
      bottom: 0,
      left: 0,
    },
    border: {
      top: 0,
      right: 0,
      bottom: 0,
      left: 0,
    },
    margin: {
      top: 0,
      right: 0,
      bottom: 0,
      left: 0,
    },
  }; // 弹窗位置

  getDomPropertyValue = (target: HTMLElement, property: string) => {
    const computedStyle = window.getComputedStyle(target);
    return Number(computedStyle.getPropertyValue(property).replace('px', ''));
  };
  renderMask(node) {
    // 获取节点位置
    const { top, right, bottom, left } = node.getBoundingClientRect();
    // 生成遮罩位置信息
    this.position = {
      top,
      right,
      bottom,
      left,
      border: {
        top: this.getDomPropertyValue(node, 'border-top-width'),
        right: this.getDomPropertyValue(node, 'border-right-width'),
        bottom: this.getDomPropertyValue(node, 'border-bottom-width'),
        left: this.getDomPropertyValue(node, 'border-left-width'),
      },
      padding: {
        top: this.getDomPropertyValue(node, 'padding-top'),
        right: this.getDomPropertyValue(node, 'padding-right'),
        bottom: this.getDomPropertyValue(node, 'padding-bottom'),
        left: this.getDomPropertyValue(node, 'padding-left'),
      },
      margin: {
        top: this.getDomPropertyValue(node, 'margin-top'),
        right: this.getDomPropertyValue(node, 'margin-right'),
        bottom: this.getDomPropertyValue(node, 'margin-bottom'),
        left: this.getDomPropertyValue(node, 'margin-left'),
      },
    };
    // 获取元素信息
    const paths = node.getAttribute(PATH_NAME) || '';
    const pathsInfo = paths.split(':');
    const name = pathsInfo[pathsInfo.length - 1];
    const column = Number(pathsInfo[pathsInfo.length - 2]);
    const line = Number(pathsInfo[pathsInfo.length - 3]);
    const path = pathsInfo.slice(0, pathsInfo.length - 3).join(':');
    this.element = { name, path, line, column };
    this.show = true;
  }
  removeMask() {
    this.show = false;
  }
  handleMouseMove = (e) => {
    // 按住了 alt 键显示遮罩
    if (e.altKey) {
      const nodePath = composedPath(e);
      let targetNode;
      // 寻找第一个有 data-insp-path 属性的元素
      for (let i = 0; i < nodePath.length; i++) {
        const node = nodePath[i];
        if (node.hasAttribute && node.hasAttribute(PATH_NAME)) {
          targetNode = node;
          break;
        }
      }
      // 元素找到了就渲染遮罩
      if (targetNode) {
        this.renderMask(targetNode);
      }
    }
  }
  handleKeyUp = (e: any) => {
    if (!e.altKey) {
      this.removeMask();
    }
  };
  handleMouseClick = (e) => {
    if (this.show && e.altKey) {
      // 唤起客户端
      const url = `http://localhost:4399/?file=${this.element.path}&line=${this.element.line}&column=${this.element.column}`;
      const xhr = new XMLHttpRequest();
      xhr.open('GET', url, true);
      xhr.send();
    }
  };
  /**
   * 
   * firstUpdated 是 LitElement 的生命周期钩子函数之一。它在元素首次更新完成后被调用，用于执行一些初始化操作或与外部资源进行交互。
   * 在 firstUpdated 中，您可以访问元素的属性和子元素，并执行任何必要的操作，例如初始化状态、绑定事件监听器或发送网络请求。这个钩子函数通常用于在元素首次渲染完成后执行一些副作用操作。
   * 需要注意的是，firstUpdated 只会在元素首次更新完成后被调用一次，之后的更新不会再触发该钩子函数。如果您需要在每次更新后执行操作，可以考虑使用 updated 钩子函数。
   */
  protected firstUpdated(): void {
    window.addEventListener('mousemove', this.handleMouseMove);
    document.addEventListener('keyup', this.handleKeyUp);
    document.addEventListener('mouseleave', this.removeMask);
    window.addEventListener('click', this.handleMouseClick, true);
  }
  render() {
    const containerPosition = {
      display: this.show ? 'block' : 'none',
      top: `${this.position.top - this.position.margin.top}px`,
      left: `${this.position.left - this.position.margin.left}px`,
      height: `${this.position.bottom -
        this.position.top +
        this.position.margin.bottom +
        this.position.margin.top
        }px`,
      width: `${this.position.right -
        this.position.left +
        this.position.margin.right +
        this.position.margin.left
        }px`,
    };
    const marginPosition = {
      borderTopWidth: `${this.position.margin.top}px`,
      borderRightWidth: `${this.position.margin.right}px`,
      borderBottomWidth: `${this.position.margin.bottom}px`,
      borderLeftWidth: `${this.position.margin.left}px`,
    };
    const borderPosition = {
      borderTopWidth: `${this.position.border.top}px`,
      borderRightWidth: `${this.position.border.right}px`,
      borderBottomWidth: `${this.position.border.bottom}px`,
      borderLeftWidth: `${this.position.border.left}px`,
    };
    const paddingPosition = {
      borderTopWidth: `${this.position.padding.top}px`,
      borderRightWidth: `${this.position.padding.right}px`,
      borderBottomWidth: `${this.position.padding.bottom}px`,
      borderLeftWidth: `${this.position.padding.left}px`,
    };
    return html`
    <div
      class="code-inspector-container"
      id="code-inspector-container"
      style=${styleMap(containerPosition)}
    >
      <div class="margin-overlay" style=${styleMap(marginPosition)}>
        <div class="border-overlay" style=${styleMap(borderPosition)}>
          <div class="padding-overlay" style=${styleMap(paddingPosition)}>
            <div class="content-overlay"></div>
          </div>
        </div>
      </div>
    </div>
  `;
  }

  static styles = css`
  .code-inspector-container {
    position: fixed;
    pointer-events: none;
    z-index: 999999;
    font-family: 'PingFang SC';
    .margin-overlay {
      position: absolute;
      inset: 0;
      border-style: solid;
      border-color: rgba(255, 155, 0, 0.3);
      .border-overlay {
        position: absolute;
        inset: 0;
        border-style: solid;
        border-color: rgba(255, 200, 50, 0.3);
        .padding-overlay {
          position: absolute;
          inset: 0;
          border-style: solid;
          border-color: rgba(77, 200, 0, 0.3);
          .content-overlay {
            position: absolute;
            inset: 0;
            background: rgba(120, 170, 210, 0.7);
          }
        }
      }
    }
  }
  .element-info {
    position: absolute;
  }
  .element-info-content {
    max-width: 100%;
    font-size: 12px;
    color: #000;
    background-color: #fff;
    word-break: break-all;
    box-shadow: 0 0 10px rgba(0, 0, 0, 0.25);
    box-sizing: border-box;
    padding: 4px 8px;
    border-radius: 4px;
  }
  .element-info-top {
    top: -4px;
    transform: translateY(-100%);
  }
  .element-info-bottom {
    top: calc(100% + 4px);
  }
  .element-info-top-inner {
    top: 4px;
  }
  .element-info-bottom-inner {
    bottom: 4px;
  }
  .element-info-left {
    left: 0;
    display: flex;
    justify-content: flex-start;
  }
  .element-info-right {
    right: 0;
    display: flex;
    justify-content: flex-end;
  }
  .element-name .element-title {
    color: coral;
    font-weight: bold;
  }
  .element-name .element-tip {
    color: #006aff;
  }
  .path-line {
    color: #333;
    line-height: 12px;
    margin-top: 4px;
  }
  .inspector-switch {
    position: fixed;
    z-index: 9999999;
    top: 16px;
    left: 50%;
    font-size: 22px;
    transform: translateX(-50%);
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: rgba(255, 255, 255, 0.8);
    color: #555;
    height: 32px;
    width: 32px;
    border-radius: 50%;
    box-shadow: 0px 1px 2px -2px rgba(0, 0, 0, 0.2),
      0px 3px 6px 0px rgba(0, 0, 0, 0.16),
      0px 5px 12px 4px rgba(0, 0, 0, 0.12);
    cursor: pointer;
  }
  .active-inspector-switch {
    color: #006aff;
  }
  .move-inspector-switch {
    cursor: move;
  }
`;
}

// 如果没有注册过 code-inspector-component 元素则注册
if (!customElements.get('code-inspector-component')) {
  customElements.define('code-inspector-component', CodeInspectorComponent);
}
```



### 5. 实现遮罩组件的注入

```typescript
// inject-component.ts
import fs from 'fs';
import path from 'path';

const getWebComponentCode = () => {
  // 读取编译后的 webcomponent 文件
  const jsCode = fs.readFileSync(path.resolve(__dirname, 'dist/code-inspector-component.umd.cjs'), 'utf-8');
  // 返回注入的代码字符串
  return `if (!document.body.querySelector('code-inspector-component')) {
    const script = document.createElement('script');
    script.setAttribute('type', 'text/javascript');
    script.textContent = ${jsCode};
    const inspector = document.createElement('code-inspector-component');
    document.body.append(inspector);
  }`
};

export const injectComponent = (code, id, record) => {
  // 已经注入过的话就不再注入
  if (record.inject) return true;
  // 获取需要注入的代码
  const injectCode = getWebComponentCode();
  code = `${code}${injectCode}`;
  record.inject = true;
  return code;
}
```

```typescript
// CodeInspectorPlugin.ts
import addContent from './addContent';
import startServer from './httpServer';
import { injectComponent } from './inject-component';

const server = {
  start: false,
  inject: false
}

export const CodeInspectorPlugin = () => {
  return {
    name: 'vite-code-inspector-plugin',
    enforce: 'pre',
    async transform(code, id) {
      if (!server.start) {
        // 启动服务
        await startServer();
        server.start = true;
      }

      if (!server.inject) {
        // 注入 web component 元素
        code = await injectComponent(code, id);
        server.inject = true;
      };

      // 排除 node_modules 中的文件
      if (id.match('node_modules')) return code;
      const isVue = /\.vue$/.test(id);
      if (isVue) {
        // 转换逻辑
        code = addContent(code, id)
      }
      return code;
    }
  }
}
```

注入方式就是在入口文件最后添加一段 script 脚本字符串即可，脚本字符串执行的逻辑就是注册并创建 `web component`遮罩组件。

此时，在页面上按下 `options`键并将鼠标移到元素上时已经可以显示遮罩了。

![image-20240116171123875](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240116171123875.png)



### 6. 点击遮罩唤起编辑器并定位到源码

点击遮罩元素时需要发起本地 `localhost` 请求，这段代码在上面遮罩组件的实现中已经添加，这里单独再拎出来。

```typescript
// code-inspector-component.ts

export class CodeInspectorComponent extends LitElement {
  // 省略其代码
  handleMouseClick = (e) => {
    if (this.show && e.altKey) {
      // 唤起客户端
      const url = `http://localhost:4399/?file=${this.element.path}&line=${this.element.line}&column=${this.element.column}`;
      const xhr = new XMLHttpRequest();
      xhr.open('GET', url, true);
      xhr.send();
    }
  };
  protected firstUpdated(): void {
    window.addEventListener('click', this.handleMouseClick, true);
  }
}
```

在本地服务部分需要根据 `url`获取位置信息。

```typescript
// httpServer.ts
import http from 'http';

export default () => {
  const server = http.createServer((req, res) => {
    // 跨域处理
    res.writeHead(200, {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': '*',
      'Access-Control-Allow-Headers': '*',
      'Access-Control-Allow-Private-Network': 'true',
    });
    const params = new URLSearchParams(req?.url?.slice(1));
    const file = params.get('file') as string;
    const line = Number(params.get('line'));
    const column = Number(params.get('column'));
    // 唤起客户端并定位源码
    launchEditor(file, line, column);
    res.end('ok');
  });
  server.listen('4399');
};
```

最后就是创建子进程，执行终端命令定位源码。

```typescript
// httpServer.ts
import fs from 'fs';
import child_process from 'child_process';

const launchEditor = (file, line, column) => {
  // 文件不存在，直接返回
  if (!fs.existsSync(file)) return;
  // 执行终端命令
  child_process.spawn('code', ['-g', file + ':' + line + ':' + column], { stdio: 'inherit' });
};
```

