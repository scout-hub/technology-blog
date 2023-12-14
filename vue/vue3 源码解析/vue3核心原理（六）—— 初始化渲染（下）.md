# vue3核心原理（六）—— 初始化渲染（下）

### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。

这一章节将介绍其它几种节点类型的初始化渲染流程。



### 一、Fragment 片段节点

在 Vue2.x 版本中，一个 Vue 组件的模板只能有一个根节点。如果我们尝试在一个组件的模板中使用多个根节点，Vue 将会抛出一个错误。但是在 Vue 3.x 中，这个限制已经被移除，我们可以在一个组件的模板中使用多个根节点。

```vue
<!-- vue2.x -->
<template>
	<div></div>
</template>

<!-- 报错 -->
<template>
	<div></div>
  <div></div>
</template>
```

```vue
<!-- vue3.x -->
<!-- 不会报错 -->
<template>
	<div></div>
  <div></div>
</template>
```

vue 3.x 中能够支持多根节点的原因就是 vue 在编译阶段额外生成了一个 Fragment 根节点，我们写的多节点作为了这个 Fragment 根节点的子节点。看上去就像一个模板文件只有一个根 Fragment 节点，Fragment 节点下有多个节点。

```vue
<template>
	<div></div>
  <div></div>
</template>
<!-- 转换成了下面这种形式 -->
<template>
  <Fragment>
  	<div></div>
    <div></div>
  </Fragment>
</template>
```

我们可以看一下模板编译的结果。

![image-20231214105619391](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214105619391.png)

可以看到在生成的 render 函数中，两个 div 作为了 Fragment 的子节点。

我们用下面这个例子来探究 Fragment 节点的处理过程，在这个例子中，我们采用了 template 来编写模板，这个模板最终会被编译成 render 函数。

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            h,
            createApp,
        } = Vue;

        const App = {
            name: "App",
            template: '<div>1</div><div>2</div>'
        }

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

我们将断点打到 `patch` 方法中

![image-20231214113612998](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214113612998.png)

首先被处理的还是最外层的根组件，这部分的处理上一节已经介绍过，我们直接看子节点的处理流程。

我们将断点打到 `setupRenderEffect` 中的 `renderComponentRoot` 这块地方，然后继续断点到这一步。![image-20231214113926585](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214113926585.png)

这一步在 `renderComponentRoot` 中会调用 render 函数生成虚拟子节点树。

![image-20231214114350060](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214114350060.png)

我们可以看到生成的虚拟节点树就是之前说的那种结构，最外层是一个 Fragment 类型的节点，子节点是两个 div。接着就是递归处理子节点的过程，第一个要处理的子节点就是 Fragment 节点。

![image-20231214114603110](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214114603110.png)

Fragment 类型的节点会经过 `processFragment`的处理。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 处理Fragment节点
   * @param n1 老的虚拟节点
   * @param n2 新的虚拟节点
   * @param container 容器
   * @param anchor 锚点元素
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
   */
  const processFragment = (n1, n2, container, anchor, parentComponent, optimized) => {
    let { patchFlag, dynamicChildren } = n2;
    // 省略部分代码
    if (n1 === null) {
			// 省略部分代码
      const { children } = n2;
      mountChildren(children, container, fragmentEndAnchor, parentComponent, optimized);
    } else {
      // 省略更新逻辑
    }
  };
```

在`processFragment`中，如果是初始化挂载阶段会直接执行 `mountChildren` 方法处理子节点。

![image-20231214115016224](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214115016224.png)

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 递归处理子节点
   * @param children 子节点
   * @param container 父容器
   * @param anchor 锚点元素
   * @param parentComponent 父组件实例
   * @param start children子节点起始遍历的位置
   * @param optimized 是否优化
   */
  const mountChildren = (
    children,
    container,
    anchor,
    parentComponent,
    optimized,
    start = 0
  ) => {
    for (let i = start; i < children.length; i++) {
      patch(null, children[i], container, anchor, parentComponent, optimized);
    }
  };
```

`mountChildren` 的逻辑就更简单了，就是循环子节点调用 `patch` 方法，后续流程就和之前讲的一样了。

![image-20231214140803389](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214140803389.png)



### 二、Comment 注释节点

我们先看一下 vue 3.x 编译注释节点的结果。

![image-20231214154436505](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214154436505.png)

可以看到注释节点这块的编译结果就是调用了 `_createCommentVNode` 方法，内部创建了 `Comment` 类型的 vnode 节点，而注释内容作为其子节点处理。

```typescript
// vue3 源码部分
/**
 * @private
 */
export function createCommentVNode(
  text: string = '',
  // when used as the v-else branch, the comment node must be created as a
  // block to ensure correct updates.
  asBlock: boolean = false
): VNode {
  return asBlock
    ? (openBlock(), createBlock(Comment, null, text))
    : createVNode(Comment, null, text)
}
```

下面我们修改一下之前的例子

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            h,
            createApp,
        } = Vue;

        const App = {
            name: "App",
            template: '<!-- 注释测试 -->'
        }

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

还是将断点打到 `patch` 和 `renderComponentRoot` 方法中，在 `renderComponentRoot` 中主要看一下注释节点对应的 vnode 结构。

![image-20231214163353559](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214163353559.png)

继续断点到 `patch`部分，此时会进入 `processCommentNode` 这个方法，这个方法用来处理注释节点。

![image-20231214163507066](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214163507066.png)

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 处理注释节点
   * @param n1 老的虚拟节点
   * @param n2 新的虚拟节点
   * @param container 容器
   * @param anchor 锚点元素
   */
  const processCommentNode = (n1, n2, container, anchor) => {
    if (n1 === null) {
      const el = (n2.el = hostCreateComment(n2.children));
      hostInsert(el, container, anchor);
    } else {
      // 省略更新逻辑
    }
  };

// packages/runtime-dom/src/nodeOps.ts
 /**
   * @author: Zhouqi
   * @description: 创建注释节点
   * @param text 注释内容
   * @return 注释节点
   */
  createComment: (text) => document.createComment(text),
```

在`processCommentNode`中会调用 `hostCreateComment` 方法创建注释节点，这里的`hostCreateComment`就是原生的 `createComment` 方法，最后将节点插入到容器中 `hostInsert` 完成注释节点的处理。

此时我们可以看到页面上已经有了这个节点。

![image-20231214163852049](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214163852049.png)



### 三、Text 文本节点

文本节点存在多种处理情况。

1. 元素下只有一个文本子节点
2. 元素下有文本节点和其它类型节点混合

先看第一种情况，元素下只有一个文本子节点。

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            h,
            createApp,
        } = Vue;

        const App = {
            name: "App",
            template: '<div>文本测试</div>'
        }

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

继续看节点编译结果。

![image-20231214165404006](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214165404006.png)

可以看到编译结果单纯将文本内容作为了 div 节点的子节点处理，接着看 `patch` 流程。

![image-20231214165546332](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214165546332.png)

由于当前处理的是 div 节点，因此会进入 `processElement`。这个方法之前介绍过，在初始化挂载阶段会执行 `mountElement` 方法。

![image-20231214165749841](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214165749841.png)

在 `mountElement` 方法中先创建了 div 对应的真实 DOM 节点，之后走进了文本子节点的处理逻辑。这块 div 对应的 vnode 中`ShapeFlag`的值是`TEXT_CHILDREN`，上一节提到过在创建 vnode 的时候会规范化子节点，如果子节点是唯一文本子节点，则当前父节点的 `ShapeFlag` 会通过位运算合并`TEXT_CHILDREN  `这个值。

```typescript
// packages/runtime-core/src/vnode.ts
// 规范化子节点，子节点的类型有多种，比如string、function、object等等
export function normalizeChildren(vnode, children) {
  let type = 0;
  const { shapeFlag } = vnode;

  if (children == null) {
    children = null;
  } else if (isArray(children)) {
    type = ShapeFlags.ARRAY_CHILDREN;
  } else if (isObject(children)) {
    if (shapeFlag & ShapeFlags.COMPONENT) {
      // 子节点是对象表示插槽节点
      type = ShapeFlags.SLOTS_CHILDREN;
    }
  } else if (isFunction(children)) {
    // 如果子节点是一个函数，则表示默认插槽
    type = ShapeFlags.SLOTS_CHILDREN;
    children = { default: children };
  } else {
    children = String(children);
    type = ShapeFlags.TEXT_CHILDREN;
  }
  vnode.children = children;
  vnode.shapeFlag |= type;
}
```

接着 `mountElement` 的逻辑，如果当前的子节点是唯一本子节点，那么会调用 `hostSetElementText` 方法将文本节点设置到 div 元素中。

![image-20231214170526848](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214170526848.png)

![image-20231214173832662](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214173832662.png)

第二种情况，元素下有文本节点和其它类型节点混合。

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            h,
            createApp,
        } = Vue;

        const App = {
            name: "App",
            template: '<div><p>123</p>文本测试</div>'
        }

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

可以看到此时生成 vnode 结构中，文本节点不再是单纯的文本字符串，而是一个 vnode 对象，其类型是 `Symbol(Text)。`

![image-20231214173955892](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214173955892.png)

接着在 `patch` 逻辑中会进入 `processText` 这个方法。

![image-20231214174712256](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214174712256.png)

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 处理节点为Text类型的虚拟节点
   * @param  n1 老的虚拟节点
   * @param  n2 新的虚拟节点
   * @param  container 容器
   * @param  anchor 锚点元素
   */
  const processText = (n1, n2, container, anchor) => {
    if (n1 === null) {
      // 老的虚拟节点不存，则表示创建节点
      const { children } = n2;
      const el = (n2.el = hostCreateText(children));
      hostInsert(el, container, anchor);
    } else {
      // 省略更新逻辑
    }
  };

// packages/runtime-dom/src/nodeOps.ts
/**
   *
   * @author: Zhouqi
   * @description: 创建文本节点
   * @param text 文本内容
   * @return 文本节点
   */
  createText(text) {
    return document.createTextNode(text);
  },
```

在 `processText` 方法中会通过 `hostCreateText` 方法创建文本节点，然后插入到父节点下。

![image-20231214175125883](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231214175125883.png)



以上就是其它几种常见类型节点的初始化挂载逻辑，当然节点类型不止这些，部分类型节点跟内置组件相关，这块后续分析对应组件时再讲解。