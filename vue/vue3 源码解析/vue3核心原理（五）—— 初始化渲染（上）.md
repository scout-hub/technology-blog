# vue3核心原理（五）—— 初始化渲染（上）



### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



在开发`vue`应用的时候，我们首先会调用`Vue.createApp`函数来创建根组件：

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
            render() {
              return h('div');
            }
        }

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

我们先看一下`createApp`的实现原理：

```typescript
// packages/runtime-dom/src/index.ts
export const createApp = (...args: [any]) => {
  const app = ensureRenderer().createApp(...args);
  // 劫持app实例上原有的mount函数
  const { mount } = app;
  app.mount = (containerOrSelector) => {
    const container = normalizeContainer(containerOrSelector);
    if (!container) return;
    mount(container);
  };
  return app;
};

/**
 * @description: 识别容器，如果是dom则直接返回；如果是字符串，则通过字符串获取dom
 * @param container 挂载元素
 */
function normalizeContainer(container) {
  if (typeof container === "string") {
    return document.querySelector(container);
  }
  return container;
}
```

在`createApp`方法中会通过`ensureRenderer().createApp`方法创建`app`实例，然后覆写扩展其实例上的`mount`方法，扩展的内容主要就是获取`mount`中传入的容器，然后调用实例原本的`mount`方法将应用挂载到容器上。其中`normalizeContainer`方法就是用来识别容器。

渲染的核心逻辑还是在`ensureRendere`上：

```typescript
// packages/runtime-dom/src/index.ts
function ensureRenderer() {
  return createRenderer(rendererOptions);
}

// packages/runtime-core/src/renderer.ts
/**
 * @description: 自定义渲染器
 * @param options 传入的平台渲染方法集合
 */
export function createRenderer(options) {
  return baseCreateRenderer(options);
}

/**
 * @description: 创建基础渲染器函数
 * @param options 传入的平台渲染方法集合
 */
function baseCreateRenderer(options) {
  // 省略渲染更新相关的方法
  return {
    render,
    createApp: createAppApi(render),
  };
}
```

在`ensureRenderer`方法中会调用`createRenderer`创建渲染器。这个渲染器接收一个配置对象，这个配置对象可以传自定义渲染器方法，当不传自定义渲染器配置时就会使用内部默认的渲染方法。创建渲染器的核心逻辑在`baseCreateRenderer`中，其内部定义了很多关于更新渲染相关的方法，最后返回`render`和`createApp`函数。

这里的`createApp`函数是由`createAppApi`方法创建的：

```typescript
// packages/runtime-core/src/apiCreateApp.ts
import { createVnode } from "./vnode";

export function createAppApi(render) {
  return function createApp(rootComponent) {
    const app = {
      _container: null,
      use() { },
      mixin() { },
      component() { },
      directive() { },
      mount(rootContainer) {
        // 创建虚拟节点
        const vnode = createVnode(rootComponent);
        // 渲染真实节点
        render(vnode, rootContainer);
        app._container = rootContainer;
      },
      unmount() {
        render(null, app._container);
      },
      provide() { },
    };
    return app;
  };
}
```

可以看到，当调用此`createApp`方法时才会创建真正的`app`实例对象。这个对象上定义了很多方法和属性，其中最关键的就是`mount`方法，在这个方法中会调用`createVnode`方法，根据组件配置创建其虚拟节点，最后调用渲染器`render`方法渲染真实节点。

因此，我们可以直到整个初始化流程大概就是：`createApp`传入组件配置——>`createVnode`创建虚拟节点——>`render`渲染真实节点。第一步我们已经看过了，接下去看第二步虚拟节点的创建。在看虚拟节点创建的逻辑前，我们首先要直到什么是虚拟节点，它有什么作用，为什么需要用虚拟节点。



什么是虚拟节点？

##### 虚拟节点（Virtual DOM）是一个 JavaScript 对象，它描述了真实 DOM 中的节点信息。



 使用虚拟节点的好处包括：

1. ##### 更快的渲染速度：虚拟 DOM 可以减少 DOM 操作的次数，从而减少浏览器的重绘和回流，提高渲染速度。

2. ##### 更好的性能：虚拟 DOM 可以在内存中进行更改，而不是在浏览器中进行实际的 DOM 操作，从而减少了浏览器的负担，提高了应用程序的性能。

3. ##### 更好的开发体验：虚拟 DOM 可以使前端开发更加模块化和可维护，因为它可以将应用程序的状态和 UI 分离开来，使代码更易于理解和维护。

4. ##### 更好的跨平台支持：虚拟 DOM 可以在不同的平台上运行，包括 Web、移动设备和桌面应用程序。

5. ##### 更好的可测试性：虚拟 DOM 可以使应用程序更易于测试，因为它可以模拟应用程序的状态和 UI，从而使测试更加简单和可靠。



其实这只是对虚拟节点的简单理解，要深究虚拟节点的应用还得从编程范式说起，这里要讲述的是声明式编程和命令式编程。

##### 命令式编程：命令式编程是一种编程范式，其中程序员编写一系列指令，以指定计算机执行的确切操作。这种编程风格通常更加具体和低级，因为它需要程序员显式地指定每个操作的细节。在命令式编程中，程序员更关注如何解决问题，而不是问题的本质。在 JavaScript 中，命令式编程通常涉及使用语句和控制结构（例如 if/else 和 for 循环）来指定计算机执行的操作。

##### 声明式编程：声明式编程是一种编程范式，其中程序员声明要执行的操作，而不是指定如何执行它们。这种编程风格通常更加抽象和高级，因为它隐藏了底层的实现细节。相反，命令式编程是一种编程范式，其中程序员编写一系列指令，以指定计算机执行的确切操作。这种编程风格通常更加具体和低级，因为它需要程序员显式地指定每个操作的细节。在声明式编程中，程序员通常更关注问题的本质，而在命令式编程中，程序员更关注如何解决问题。在 JavaScript 中，声明式编程通常涉及使用类似于 JSX 的语法来声明 UI 组件，而不是直接操作 DOM 元素。



举个例子，假如我们现在要创建两个`div`元素，元素的宽高为 30px，前者 id 为 div，后者为 div1，当 visible 为 true 时显示在页面中。下面分别用声明式和命令式的写法来实现这个需求：

```typescript
// 命令式
const flag = true; // or false
const div = document.createElement('div');
div.style.width = '30px';
div.style.height = '30px';
div.id = 'div';
const div1 = document.createElement('div');
div1.style.width = '30px';
div1.style.height = '30px';
div1.id = 'div1';
if (flag){
	document.body.appendChild(div);
  document.body.appendChild(div1);
}
```

```vue
<!-- 声明式 -->
<template>
  <div v-if="flag" id="div" style="width:30px;height:30px;"></div>
  <div v-if="flag" id="div1" style="width:30px;height:30px;"></div>
</template>
```

可以看到，命令式编程就是按照需求的步骤一步一步用代码实现，包括元素的创建等都需要自己编写。而声明式编程只需要按照特定的声明方式去编写即可，不需要实现元素的创建和添加，这部分已经由底层框架实现。



声明式编程的好处包括：

- ##### 更容易理解和维护：声明式编程使代码更易于理解，因为它更接近自然语言。这使得代码更容易维护和修改，因为开发人员可以更快地理解代码的意图。

- ##### 更少的副作用：声明式编程通常会减少副作用，这使得代码更加可预测和可靠。

- ##### 更高的抽象级别：声明式编程使开发人员能够更高级别地思考问题，而不必担心实现细节。这使得代码更加模块化和可重用。

- ##### 减少重复代码：声明式编程通常会减少重复代码，这使得代码更加简洁和易于维护。

相比之下命令式编程就会显得比较繁琐，所有的步骤都需要个人实现，并且很容易就会出现重复代码，经常写 JQuery 的开发者应该深有体会，大量繁琐的 DOM 操作其实很令人头疼，并且不好维护。而`vue`和`react`这类框架就省去了 DOM 操作，该部分都由框架底层实现，让开发者更专注于功能逻辑的编写，减少其它工作量。



那是不是意味着声明式一定优于命令式？其实不然，从性能角度上来说命令式一定是优于声明式的。像`vue`和`react`这类的声明式框架，其底层会有很多其它额外的处理逻辑，这些逻辑的执行都是会耗费时间和资源的。相对来说，命令式就显得更加直接高效。

我们还是继续用上面的例子，现在页面上存在两个 div，我们需要更新第一个 div，将其宽高更新为 40px。按照命令式的方式操作就很直接高效：

```typescript
// 命令式
const div = document.getElementById('div');
div.style.width = '40px';
div.style.height = '40px';
```

而同样让声明式框架去实现的话，其中一种简单的方式就是生成全新的节点，然后整体替换老的节点：

```vue
<!-- 声明式 -->
<template>
  <div v-if="flag" id="div" style="width:40px;height:40px;"></div>
  <div v-if="flag" id="div1" style="width:30px;height:30px;"></div>
</template>
```

显然，这种方式是不合理的，并且会有性能问题。现在需求是更新第一个 div，但是整个页面都重新渲染了一遍，用户体验上也会很差。

但是声明式框架的更新逻辑都是在底层同一封装处理的，不能像命令式框架一样，直接获取第一个元素的 id，然后去更新元素。这就要求框架能够实现新旧节点之间的比较，然后更新有差异的部分。这部分额外的处理消耗其实就是声明式和命令式之间的性能差异，如果能完全磨平这段额外的性能消耗，声明式可以等于命令式，但是实际上磨平是不可能的，有额外执行逻辑就会有额外的性能消耗。因此，框架要考虑的就是尽可能减少这段部分的消耗，而虚拟节点恰恰就是减小这部分消耗的一个点。



在了解了虚拟节点的相关概念后，我们继续回到之前创建虚拟节点的流程`createVnode`：

```typescript
// packages/runtime-core/src/vnode.ts
function createBaseVNode(
  type,
  props,
  children,
  patchFlag = 0,
  dynamicProps = null,
  shapeFlag = type === Fragment ? 0 : ShapeFlags.ELEMENT,
  isBlockNode = false,
  needFullChildrenNormalization = false
) {
  const vnode = {
    type,
    props,
    children,
    el: null,
    shapeFlag,
    component: null,
    key: props && props.key,
    patchFlag,
    dynamicProps,
  };

  if (needFullChildrenNormalization) {
    // 规范化子节点处理，子节点的类型有很多，比如数组，对象，函数等等
    normalizeChildren(vnode, children);
  } else {
    // 能走到这里说明children一定是string或者array类型的
    vnode.shapeFlag |= isString(children)
      ? ShapeFlags.TEXT_CHILDREN
      : ShapeFlags.ARRAY_CHILDREN;
  }
    
  // 省略其它代码
  return vnode;
}
```

`createBaseVNode`接收很多参数，我们暂时只需要了解其中 4 个参数的含义：

- *type*：节点标签名，可以是一个字符串或者一个组件对象
- *props*：节点上的属性集合
- *children*：子节点集合
- *shapeFlag*：节点类型，二进制表示

方法内部会先创建一个`vnode`对象并初始化部分属性，接着如果需要进行子节点处理，会调用`normalizeChildren`方法，最后返回创建的`vnode`对象。其中`normalizeChildren`的主要逻辑就是根据子节点的类型去标记`shapeFlag`，在标记和判断的代码中，我们可以看到很多位运算操作，并且这个`shapeFlag`也是一个二进制。`shapeFlag` 枚举使用位运算的好处是可以用更少的内存来存储多个标志位，同时也可以提高代码的性能。在判断组件类型和修改类型的时候，通过位运算的方式去判断或者比较，性能更优，但是代码可读性相对更低。在修改 `vnode` 类型的时候可以用 `|` 运算，在查找 `vnode` 类型的时候可以用 `&` 运算。

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

// packages/shared/src/shapeFlags.ts
/**
 * 用二进制表示组件类型，在判断组件类型和修改类型的时候通过位运算的方式去判断或者比较，性能更优，但是代码可读性相对更低
 *
 * | 运算 同位置上都为0才为0，否则为1
 * & 运算 同位置上都为1才为1，否则为0
 *
 * 0001 | 0000 = 0001; 0000 | 0000 = 0000; 0100 | 0001 = 0101
 * 0001 & 0001 = 0001; 0000 | 0001 = 0000; 0100 & 0101 = 0100
 *
 * 在修改vnode类型的时候可以用 | 运算，在查找vnode类型的时候可以用 & 运算
 */
export const enum ShapeFlags {
  ELEMENT = 1, // 0000000001
  FUNCTIONAL_COMPONENT = 1 << 1, // 0000000010
  STATEFUL_COMPONENT = 1 << 2, // 0000000100
  TEXT_CHILDREN = 1 << 3, // 0000001000
  ARRAY_CHILDREN = 1 << 4, // 0000010000
  SLOTS_CHILDREN = 1 << 5, // 0000100000
  TELEPORT = 1 << 6, // 0001000000
  COMPONENT_SHOULD_KEEP_ALIVE = 1 << 8, // 0100000000
  COMPONENT_KEPT_ALIVE = 1 << 9, // 1000000000
  COMPONENT = ShapeFlags.STATEFUL_COMPONENT | ShapeFlags.FUNCTIONAL_COMPONENT, // 0000000110
}
```

例子中初步生成的`vnode`是这样的：

![image-20231129163724912](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231129163724912.png)

最后一步就是`render`：

```typescript
// packages/runtime-core/src/renderer.ts  
/**
  * @description: 渲染函数
  * @param vnode 虚拟节点
  * @param container 容器
  */
 const render = (vnode, container) => {
   // 新的虚拟节点为null，说明是卸载操作
   if (vnode === null && container._vnode) {
     unmount(container._vnode, null);
   } else {
     patch(container._vnode || null, vnode, container, null, null);
   }
   // 执行lifecycle hook
   flushPostFlushCbs();
   // 缓存当前vnode，下一次更新的时候，该值就是旧的vnode
   container._vnode = vnode;
 };
```

在`render`方法中会判断新老`vnode`是否存在，如果新的不存在，老的存在，则表示是卸载操作，此时会调用`unmount`方法进行销毁，否则就会调用`patch`方法更新视图。在初始化流程中会执行`patch`方法创建视图。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 更新函数
   * @param n1 老的虚拟节点
   * @param n2 新的虚拟节点
   * @param container 容器
   * @param anchor 锚点元素
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
   */
  const patch = (n1, n2, container, anchor = null, parentComponent, optimized = !!n2.dynamicChildren) => {
    if (n1 === n2) return;
    // 省略部分代码
    const { shapeFlag, type } = n2;

    switch (type) {
      // 特殊虚拟节点类型处理
      case Fragment:
        // 处理type为Fragment的节点（插槽）
        processFragment(n1, n2, container, anchor, parentComponent, optimized);
        break;
      case Comment:
        // 处理注释节点
        processCommentNode(n1, n2, container, anchor);
        break;
      case Text:
        // 处理文本节点
        processText(n1, n2, container, anchor);
        break;
      default:
        // if is element
        if (shapeFlag & ShapeFlags.ELEMENT) {
          processElement(n1, n2, container, anchor, parentComponent, optimized);
        } else if (shapeFlag & ShapeFlags.COMPONENT) {
          // 有状态、函数式组件
          processComponent(n1, n2, container, anchor, parentComponent);
        } else if (shapeFlag & ShapeFlags.TELEPORT) {
          type.process(n1, n2, container, anchor, parentComponent, internals);
        }
    }
  };
```

在`patch`方法中会根据`vnode`的类型进入不同的处理方法。示例中会先处理根组件 App 对应的`vnode`，

![image-20231129164845041](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231129164845041.png)

它的 type 是一个对象，这个对象就是示例中 App 组件配置对象，`shapeFlag` 是 4，对应 `ShapeFlags.COMPONENT`。因此它会进入有状态组件的处理`processComponent`的处理逻辑，进入之前可以看到`n1`、`n2`、`container`等变量的值。

![image-20231129165218095](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231129165218095.png)

下面是`processComponent`的大致逻辑：

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 处理组件
   * @param n1 旧的虚拟节点
   * @param n2 新的虚拟节点
   * @param container 容器
   * @param anchor 锚点元素
   * @param  parentComponent 父组件实例
   */
  const processComponent = (n1, n2, container, anchor, parentComponent) => {
    // n1为null表示初始化组件
    if (n1 == null) {
      // 如果组件是keep-alive缓存过的组件，则不需要重新挂载，只需要从隐藏容器中取出缓存过的DOM即可
      if (n2.shapeFlag & ShapeFlags.COMPONENT_KEPT_ALIVE) {
        // 省略 keep-alive 组件的逻辑
      } else {
        mountComponent(n2, container, anchor, parentComponent);
      }
    } else {
      // 更新组件
      updateComponent(n1, n2);
    }
  };
```

初始化流程中 `n1` 为 null，因此会进入`mountComponent`创建组件的逻辑。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 创建元素
   * @param  initialVNode 初始虚拟节点
   * @param  container 容器
   * @param  anchor 锚点元素
   * @param  parentComponent 父组件实例
   */
  const mountComponent = (initialVNode, container, anchor, parentComponent) => {
    // 获取组件实例
    const instance = (initialVNode.component = createComponentInstance(
      initialVNode,
      parentComponent
    ));
    
    // 初始化组件
    setupComponent(instance);
    setupRenderEffect(instance, initialVNode, container, anchor);
  };
```

`mountComponent`中有三个阶段：

- `createComponentInstance`：创建组件实例，并将其绑定到 `vnode` 对象中的 `component` 属性上
- `setupComponent`：初始化组件数据
- `setupRenderEffect`：执行组件渲染和更新

首先是组件实例创建阶段`createComponentInstance`：

```typescript
// packages/runtime-core/src/component.ts
/**
 * @author: Zhouqi
 * @description: 创建组件实例
 * @param vnode 虚拟节点
 * @param parent 父组件实例
 */
export function createComponentInstance(vnode, parent) {
  const type = vnode.type;

  const componentInstance = {
    vnode,
    type,
    parent,
    isMounted: false,
    subTree: null,
    emit: null,
    next: null,
    proxy: null,
    provides: parent ? parent.provides : Object.create(null),
    propsOptions: normalizePropsOptions(type),
    emitsOptions: normalizeEmitsOptions(type),

    components: null,
    directives: null,

    props: EMPTY_OBJ,
    data: EMPTY_OBJ,
    attrs: EMPTY_OBJ,
    slots: EMPTY_OBJ,
    ctx: EMPTY_OBJ,
    setupState: EMPTY_OBJ,

    accessCache: null,
    inheritAttrs: type.inheritAttrs,

    // lifecycle hooks
    bm: null,
    m: null,
    bu: null,
    u: null,
    bum: null,
    um: null,
  };

  // 在_属性中存储组件实例对象
  componentInstance.ctx = { _: componentInstance };
  componentInstance.emit = emit.bind(null, componentInstance) as any;
  return componentInstance;
}
```

方法内部定义了一个`componentInstance`对象，这个就是初始的组件实例对象，内部定义了所有需要的属性和方法，最后返回这个对象。我们可以在控制台或者侧边栏查看生成的实例对象。

![image-20231201153326274](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231201153326274.png)

接下去初始化组件实例上的数据 `setupComponent`：

```typescript
// packages/runtime-core/src/component.ts
/**
 * @author: Zhouqi
 * @description: 初始化组件
 * @param instance 组件实例
 */
export function setupComponent(instance) {
  const { props, children } = instance.vnode;
  const isStateful = isStatefulComponent(instance);
  // 初始化props
  initProps(instance, props, isStateful);
  // 初始化slots
  initSlots(instance, children);
  const setupResult = isStateful ? setupStatefulComponent(instance) : undefined;
  return setupResult;
}
```

这个方法内部也有三个比较重要的阶段：

- `initProps`：初始化 props 属性
- `initSlots`：初始化 slot 插槽
- `setupStatefulComponent`：处理有状态组件

我们先不考虑 `initProps` 和 `initSlots` 的逻辑，直接看组件状态的处理 `setupStatefulComponent`，不过在这之前需要了解什么是有/无状态组件。

------

**在 Vue 中，有状态组件和无状态组件的概念与 React 中的类似。**

**有状态组件：这些是可以包含自己的状态（即数据）和方法的组件。在Vue中，我们通常在组件的data选项中定义状态，并在methods选项中定义方法。这些组件通常用于处理用户交互，数据更新等。**

**无状态组件：这些是只依赖于传入的props，不包含自己的状态和方法的组件。它们通常用于简单的展示组件，只依赖于外部的props。**

------

接着 `setupStatefulComponent` 的逻辑：

```typescript
// packages/runtime-core/src/component.ts
/**
 * @author: Zhouqi
 * @description: 有状态组件
 * @param instance
 */
function setupStatefulComponent(instance) {
  const { type: component, props, emit, attrs, slots } = instance;
  const { setup } = component;

  // 这里accessCache不能赋值为{}，因为通过this.xxx读取的xxx属性可能是对象内置的属性，比如hasOwnProperty，这会影响
  // instance.proxy中get的读取逻辑
  instance.accessCache = Object.create(null);

  // 这里只是代理了instance上的ctx对象
  // 在处理函数中由于需要instance组件实例，因此需要在ctx中增加一个变量_去存储组件实例，供处理函数内部访问
  // 通过这个代理，我们就能用this.xxx去访问数据了
  instance.proxy = new Proxy(instance.ctx, PublicInstanceProxyHandlers);

  // 调用组件上的setup方法获取到数据
  if (setup) {
    setCurrentInstance(instance);
    // props是浅只读的，在开发模式下是shallowReadonly类型，生产环境下不会进行shallowReadonly处理，这里默认进行shallowReadonly处理
    const setupResult =
      setup(shallowReadonly(props), { emit, attrs, slots }) || {};
    unsetCurrentInstance();

    handleSetupResult(instance, setupResult);
  } else {
    finishComponentSetup(instance);
  }
}
```

在 `setupStatefulComponent` 中创建一个 `proxy`，它代理了组件实例，`PublicInstanceProxyHandlers` 就是具体的代理操作。有了这个代理，我们就可以通过 `this.xxx` 的方式去访问组件实例上的数据。接下去会判断组件对象上是否定义了 `setup` 方法，如果定义则会执行该方法并传入参数，这些参数有 `props`、`emit`、`attrs`、`slots`，从这里我们可以知道在 `setup` 中可以获取哪些值和方法。注意，这里的 `props` 用 `shallowReadonly`做了一层包裹，也就是说我们不能去修改 `props`的值，只能读，这也符合数据单向流的设计模式。

------

**数据单向流是一种编程模式，它在许多现代前端框架（如React和Vue）中被广泛采用。这种模式的主要优点如下：**

1. **可预测性：数据的流动方向是固定的，这使得应用的状态变化更加可预测和可追踪。**
2. **易于调试：由于数据的流动方向是单向的，当出现问题时，我们可以更容易地追踪数据的来源和变化。**
3. **降低复杂性：数据单向流动可以减少数据在组件之间的相互影响，降低应用的复杂性。**
4. **易于理解：对于新手来说，理解数据如何在应用中流动和更新会更加直观。**

------

执行完 `setup` 方法后会拿到返回值，这个返回值需要经过`handleSetupResult`的处理。

```typescript
// packages/runtime-core/src/component.ts
/**
 * @author: Zhouqi
 * @description: 处理setup返回值
 * @param instance 组件实例
 * @param setupResult setup返回值
 */
function handleSetupResult(instance, setupResult) {
  if (isObject(setupResult)) {
    // 如果结果是对象，说明返回的是数据
    instance.setupState = proxyRefs(setupResult);
  } else if (isFunction(setupResult)) {
    // 如果是函数，则表示渲染函数
    instance.render = setupResult;
  }
  finishComponentSetup(instance);
}
```

如果 `setup` 方法的返回值是一个对象，则表示返回的是数据，该数据会绑定到组件实例上的 `setupState` 属性中。其中数据还会经过 `proxyRefs` 的处理，使其在模板上访问时不需要通过 .value 访问值，也就是我们可以在模板上访问 `ref`这类数据时不需要通过 `.value` 访问的原因。如果返回值是一个方法，则说明是渲染函数，此时会将方法绑定到实例上的 `render` 属性中。

当 `setup` 返回值处理完成后就会进入 `finishComponentSetup` 方法：

```typescript
// packages/runtime-core/src/component.ts
function finishComponentSetup(instance) {
  const { type: component } = instance;

  if (!instance.render) {
    if (!component.render) {
      const { template } = component;
      if (template) {
        component.render = compile(template);
      }
    }
    instance.render = component.render || NOOP;
  }

  // 处理 vue2 options api
  // applyOptions(instance);
}
```

`finishComponentSetup` 方法主要作用就是处理 `render` 函数。首先 `instance.render`是实例上的方法，前面有提到当 `setup` 函数返回一个方法时表示是个渲染函数，这个函数会绑定到 `instance.render` 中。

```typescript
new Vue({
  setup(){
    return (h) => {
      return h('div');
    }
  }
});
```

如果 `instance.render` 不存在则会判断 `component.render` 是不是存在，`component.render` 是我们在组件配置上写的 `render` 函数。

```typescript
new Vue({
  render(h){
    return h('div')
  }
});
```

最后就是 `template` 上的模板字符串，它会经过 `compile` 处理编译成 `render`  函数。

```typescript
new Vue({
  template:'<div></div>'
});
```

三种 render 的优先级是： setup 返回的渲染函数 > 组件配置上的渲染函数 > template 模板字符串编译后的渲染函数。

至此，整个 `setupComponent` 流程就结束了，接下去就是执行渲染和更新的逻辑 `setupRenderEffect`：

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 执行渲染和更新
   * @param  instance 组件实例
   * @param  initialVNode 初始虚拟节点
   * @param  container 容器
   * @param  anchor 锚点元素
   */
  const setupRenderEffect = (instance, initialVNode, container, anchor) => {
    const componentUpdateFn = () => {
      // 通过isMounted判断组件是否创建过，如果没创建过则表示初始化渲染，否则为更新
      if (!instance.isMounted) {
        const subTree: Record<string, any> = (instance.subTree =
          renderComponentRoot(instance));
        patch(null, subTree, container, anchor, instance);
        // 到这一步说明元素都已经渲染完成了，也就能够获取到根节点，这里的subTree就是根组件
        initialVNode.el = subTree.el;
        // 表示组件Dom已经创建完成
        instance.isMounted = true;
      } else {
        // 省略更新的逻辑
      }
    };
    const effect = new ReactiveEffect(componentUpdateFn, () => {
      // 自定义调度器，当多个同步任务触发更新时，将任务放入微任务队列中，避免多次更新
      queueJob(instance.update);
    });
    const update = (instance.update = effect.run.bind(effect));
    // 收集依赖，在依赖的响应式数据变化后可以执行更新
    update();
  };
```

在 `setupRenderEffect` 方法中创建了一个自定义调度的 `ReactiveEffect`。在执行组件渲染的过程中会访问响应式数据触发依赖收集，当响应式数据发生变化后就会触发对应的依赖函数，这个依赖函数就是 `queueJob(instance.update)`，也就是更新和渲染的核心方法。

首次执行挂载的时候会手动调用一次 `update` 函数，在 `update` 方法中会先调用 `renderComponentRoot` 执行 `render`函数渲染子树：

```typescript
// packages/runtime-core/src/componentRenderUtils.ts
/**
 * @author: Zhouqi
 * @description: 生成组件的vnode
 * @param instance 组件实例
 */
export function renderComponentRoot(instance) {
  const {
    attrs,
    props,
    render,
    proxy,
    vnode,
    type: Component,
  } = instance;
  // 省略部分代码
  let result;

  if (vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
    // 生成 vnode
    result = normalizeVNode(render.call(proxy, proxy));
  }

  return result;
}

// packages/runtime-core/src/vnode.ts
/**
 * @author: Zhouqi
 * @description: 解析vnode（这里暂时只处理有状态组件的情况）
 * @param child 虚拟节点
 */
export function normalizeVNode(child) {
  if (child == null) {
    // 如果render函数没有返回对应的vnode，则默认创建一个注释节点
    return createVnode(Comment);
  } else if (isObject(child)) {
    // 显然已经是一个vnode类型的数据
    // 如果vnode上没有el，说明是初始化渲染，直接返回vnode即可
    // 如果是更新的话，需要克隆一份新的vnode
    return child.el === null ? child : cloneVNode(child);
  }
  return child;
}
```

其中 `render.call(proxy, proxy)` 就是执行渲染函数并传入之前在 `setupStatefulComponent` 中创建好的数据方法的代理对象，执行结果会经过 `normalizeVNode` 的处理。在我们的 demo 中，render 函数只有 `h('div')`，并且初始化下 `node.el` 也不存在，因此只需要返回创建的 vnode 即可。

![image-20231211114528906](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231211114528906.png)

![image-20231211114647352](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231211114647352.png)

当 `subTree` 子树创建完成后就会进入到 `patch` 方法中进行真正的视图更新流程。其中，`subTree` 是上一步生成的子节点的 vnode，`container`则是容器节点。

![image-20231211153023627](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231211153023627.png)

`patch` 方法逻辑：

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 更新函数
   * @param n1 老的虚拟节点
   * @param n2 新的虚拟节点
   * @param container 容器
   * @param anchor 锚点元素
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
   */
  const patch = (n1, n2, container, anchor = null, parentComponent, optimized = !!n2.dynamicChildren) => {
    if (n1 === n2) return;
    // 省略部分代码
    const { shapeFlag, type } = n2;

    switch (type) {
      // 特殊虚拟节点类型处理
      case Fragment:
        // 处理type为Fragment的节点（插槽）
        processFragment(n1, n2, container, anchor, parentComponent, optimized);
        break;
      case Comment:
        // 处理注释节点
        processCommentNode(n1, n2, container, anchor);
        break;
      case Text:
        // 处理文本节点
        processText(n1, n2, container, anchor);
        break;
      default:
        // if is element
        if (shapeFlag & ShapeFlags.ELEMENT) {
          processElement(n1, n2, container, anchor, parentComponent, optimized);
        } else if (shapeFlag & ShapeFlags.COMPONENT) {
          // 有状态、函数式组件
          processComponent(n1, n2, container, anchor, parentComponent);
        } else if (shapeFlag & ShapeFlags.TELEPORT) {
          type.process(n1, n2, container, anchor, parentComponent, internals);
        }
    }
  };
```

其实 `patch` 逻辑的结构和 `mount` 逻辑的结构是类似的，根据不同 vnode 的类型走不同的处理逻辑。在我们的例子中，传入的 n2 是 div 对应的 vnode，因此它会走 `processElement` 这个逻辑，这是专门用来处理普通元素节点的方法。

![image-20231211160544750](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231211160544750.png)

`processElement`的逻辑：

```typescript
 /**
   * @author: Zhouqi
   * @description: 处理普通元素
   * @param n1 老的虚拟节点
   * @param n2 新的虚拟节点
   * @param container 父容器
   * @param anchor 锚点元素
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
   */
  const processElement = (n1, n2, container, anchor, parentComponent, optimized) => {
    // 旧的虚拟节点不存在，说明是初始化渲染
    if (n1 === null) {
      mountElement(n2, container, anchor, parentComponent, optimized);
    } else {
      // 更新
      patchElement(n1, n2, parentComponent, optimized);
    }
  };
```

`processElement` 中会根据老的 vnode 是否存在走不同的处理逻辑，在初始化挂载阶段会走`mountElement`的逻辑。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 生成普通元素
   * @param  vnode 虚拟dom
   * @param  container 父容器
   * @param  anchor 锚点元素
   * @param  parentComponent 父组件实例
   */
  const mountElement = (vnode, container, anchor, parentComponent, optimized) => {
    const { type, props, children, shapeFlag, transition, dirs } = vnode;
    const el = (vnode.el = hostCreateElement(type));
    // 处理children
    if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
      // 孩子是一个字符串表示文本类型
      hostSetElementText(el, children);
    } else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      // 处理数组类型的孩子节点
      // #fix bug example defineAsyncComponent 新建子节点的时候不需要锚点元素
      // mountChildren(children, el, anchor, parentComponent);
      mountChildren(children, el, null, parentComponent, optimized);
    }
    hostInsert(el, container, anchor);
  };
```

在 `mountElement` 中会先创建 vnode 对应的节点，在我们的例子中这个 vnode 是个 div 节点，因此这里会调用 `hostCreateElement` 创建 div 对应的真实节点，这个`hostCreateElement`就是`createElement`。

![image-20231211163737161](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231211163737161.png)

接下去会处理子节点，在我们的例子中没有子节点，因此直接执行最后一步 `hostInsert`，将节点插入到父节点中，这个`hostInsert` 就是 `insertBefore`。

当`patch`完成后会将当前创建好的视图节点缓存到 `initialVNode.el` 中并将 `isMounted` 标记为 true。

![image-20231211115612956](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231211115612956.png)

此时我们也能看到视图上有了 div 这个节点。

![image-20231211115312155](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231211115312155.png)

这部分我们用了一个比较简单的例子了解 vue3 最基本的初始化流程，下一节会介绍其它各种节点的处理。
