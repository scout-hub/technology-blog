# vue3核心原理（十）——  更新（上）

### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



前几个章节我们介绍了各类节点的初始化渲染以及节点属性、事件的初始化绑定过程，本节将开始介绍各类节点的更新机制。



## 一、基本概念

在 Vue3 中，所有的更新都是基于组件级别的更新。当组件的状态（例如，响应式数据或计算属性）发生变化时，Vue3 会重新渲染并更新该组件。这种基于组件的更新策略有助于提高应用程序的性能，因为只有实际需要更新的组件才会被重新渲染。

在初始化渲染部分曾提到执行组件渲染和更新的方法 `setupRenderEffect`：

```typescript
// packages/runtime-core/src/renderer.ts
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
  const effect = new ReactiveEffect(componentUpdateFn);
  const update = (instance.update = effect.run.bind(effect));
  update();
};
```

在 `setupRenderEffect`中会实例化一个 `ReactiveEffect`并传入 `componentUpdateFn` 作为依赖函数。在初始化渲染阶段会执行组件的 `render` 方法生成 `VNode`，这个过程中如果访问了响应式数据，那么就会进行依赖收集，将创建的 `ReactiveEffect`收集进去。当响应式数据变化后会执行 `componentUpdateFn`中的更新逻辑，如果数据没有变化，则不会触发。

从这里我们便知道在 Vue3 中更新的细粒度控制到了组件级别，那么为什么不将细粒度控制到某个节点级别？这不是更加高效吗？其实在 Vue 的早期版本中确实是这么做的，但是在大型应用中，页面的节点数量是非常庞大。如果每个节点都对应一个依赖关系，那么依赖数量也会非常庞大，势必会造成一定的性能开销。

既然 Vue 将更新的细粒度控制到组件级别，那么组件中的节点又该如何高效地更新？总不可能组件触发了更新，页面上所有节点都需要更新。假设页面中有 100000 个节点，其中一个节点依赖的数据变化了，但却让这 100000 个节点都更新一遍，这显然也是不合理的。

基于上述原因，Vue 采用了 `VNode` 加 `Diff`算法来实现组件内部节点的更新。`VNode` 在初始化渲染部分已经讲述过了，这里我们了解下什么是 `Diff` 算法。

------

**Diff 算法：**

**Vue中的diff算法主要用于比较新旧两个虚拟DOM树的差异，然后将这些差异应用到实际的DOM上，以达到高效更新视图的目的。**

**这个算法的核心思想是只比较同级节点，不跨级比较。在 DOM 树的比较过程中，如果不采用 "同层比较"，而是跨层比较，那么比较的复杂度将会是指数级别的，这将会消耗大量的计算资源，导致性能下降**。

**而 "同层比较" 的策略将比较的复杂度降低到了线性级别，大大提高了比较的效率。这是因为在实际的 Web 开发中，大部分的 DOM 变动都是在同一层级进行的，很少会有跨层级的 DOM 变动，因此 "同层比较" 的策略在大多数情况下都能够很好地工作。**

------

在同层比较的前提下，新旧节点的是否相同依赖两个条件：节点类型`type`以及标记节点身份的 `key`。在源码中比较新旧节点是否相同会调用 `isSameVNodeType`：

```typescript
// packages/runtime-core/src/vnode.ts
export function isSameVNodeType(n1, n2) {
  return n1.type === n2.type && n1.key === n2.key;
}
```

从这个方法中可以知道只有节点类型和 `key`都相同的情况下才会被认为是同一节点。在 Vue中，如果新旧节点相同则会复用老的节点并更新，否则会先删除老的节点及其所有后代节点并创建新的节点。



## 二、更新流程

首先我们完善一下组件的更新逻辑：

```typescript
// packages/runtime-core/src/renderer.ts
const setupRenderEffect = (instance, initialVNode, container, anchor) => {
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      const subTree: Record<string, any> = (instance.subTree =
                                            renderComponentRoot(instance));
      patch(null, subTree, container, anchor, instance);
      // 到这一步说明元素都已经渲染完成了，也就能够获取到根节点，这里的subTree就是根组件
      initialVNode.el = subTree.el;
      // 表示组件Dom已经创建完成
      instance.isMounted = true;
    } else {
      // 新增代码
      const nextTree = renderComponentRoot(instance);
      const prevTree = instance.subTree;
      instance.subTree = nextTree;
      patch(prevTree, nextTree, container, anchor, instance);
    }
  };
  const effect = new ReactiveEffect(componentUpdateFn);
  const update = (instance.update = effect.run.bind(effect));
  update();
};
```

前面提到过，更新部分包含新旧节点树的比较。因此，在更新的时候依旧需要调用 `renderComponentRoot` 方法生成新的子 `VNode`树，这棵子树我们可以成为 `nextTree`，而老的子 `VNode`树就是之前生成的 `subTree`，我们可以称之为 `prevTree`。获取到新旧 `VNode`树之后，我们需要更新一下组件实例上的 `subTree`，使其总是最新的 `VNode`树，最后调用`patch` 方法进行更新。

```typescript
// packages/runtime-core/src/renderer.ts
const patch = (n1, n2, container, anchor = null, parentComponent, optimized = !!n2.dynamicChildren) => {
  if (n1 === n2) return;
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
      } 
  }
};
```

`patch`方法还是和之前讲的一样，内部会根据节点类型进入不同类型的处理流程，我们依次来看一下这些节点的更新逻辑。

1.  `Fragment`节点

```typescript
// packages/runtime-core/src/renderer.ts
const processFragment = (n1, n2, container, anchor, parentComponent, optimized) => {
  // 省略部分代码
  if (n1 === null) {
    const { children } = n2;
    mountChildren(children, container, fragmentEndAnchor, parentComponent, optimized);
  } else {
    // 更新子节点（新增代码）
    patchChildren(n1, n2, container, fragmentEndAnchor, parentComponent, optimized);
  }
};
```

在初始化章节层提到，`Fragment`类型的节点本身没有任何处理，也没有真实的 `DOM` 节点，只会去创建子节点。因此，在更新的时候，也只会调用 `patchChildren` 方法更新子节点。



2. `Comment`节点

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
    // 不支持动态更新注释节点，只会复用（新增代码）
    n2.el = n1.el;
  }
};
```

`comment`注释节点不支持动态更新，因此只会复用老的节点。



3. 文本节点

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
    const { children } = n2;
    const el = (n2.el = hostCreateText(children));
    hostInsert(el, container, anchor);
  } else {
    // 更新文本节点（新增代码）
    const el = (n2.el = n1.el);
    if (n1.children !== n2.children) {
      hostSetText(el, n2.children);
    }
  }
};
```

对于文本子节点，文本内容就是 `children`，因此只需要比较 `children` 前后是否相同再做更新即可。



4. 普通元素类型

```typescript
// packages/runtime-core/src/renderer.ts 
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

/**
   * @author: Zhouqi
   * @description: 更新元素
   * @param n1 旧的虚拟节点
   * @param n2 新的虚拟节点
   * @param parentComponent 父组件实例
   */
const patchElement = (n1, n2, parentComponent, optimized) => {
  // 新的虚拟节点上没有el，需要继承老的虚拟节点上的el
  const el = (n2.el = n1.el);
  // 更新子节点
  patchChildren(n1, n2, el, null, parentComponent, false);
  // 更新当前节点属性
  const oldProps = n1.props || EMPTY_OBJ;
  const newProps = n2.props || EMPTY_OBJ;
  patchProps(el, n2, oldProps, newProps);
};
```

对于普通元素的更新会先调用 `patchChildren`方法更新子节点，再调用 `patchProps` 更新当前节点。

`patchProps`的意思是更新属性：

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 更新props属性
   * @param el 容器
   * @param vnode 新的虚拟节点
   * @param oldProps 旧的props
   * @param newProps 新的props
   */
const patchProps = (el, vnode, oldProps, newProps) => {
  if (oldProps === newProps) return;

  if (oldProps !== EMPTY_OBJ) {
    for (const key in oldProps) {
      if (!(key in newProps)) {
        hostPatchProp(el, key, oldProps[key], null);
      }
    }
  }

  for (const key in newProps) {
    const nextValue = newProps[key];
    const prevValue = oldProps[key];
    if (nextValue !== prevValue) {
      hostPatchProp(el, key, prevValue, nextValue);
    }
  }
};
```

Vue3 将属性的更新分为以下几种情况：

1. 新旧 `props`上都存在某个 key 但是对应的值不一样 ———— 更新属性
2. 新旧 `props`上都存在某个 key 但是新的值不存在  ———— 删除属性
3. 旧的 `props`上某个 key 在新的 `props`中不存在 ———— 删除属性
4. 新的 `props`上某个 key 在旧的 `props`中不存在 ———— 新增属性

核心属性处理逻辑 `hostPatchProp` 在第七章有介绍。



5. 组件类型

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
    mountComponent(n2, container, anchor, parentComponent);
  } else {
    // 更新组件
    updateComponent(n1, n2);
  }
};


/**
   * @author: Zhouqi
   * @description: 更新组件
   * @param  n1 老的虚拟节点
   * @return  n2 新的虚拟节点
   */
const updateComponent = (n1, n2) => {
  const instance = (n2.component = n1.component);
  if (shouldUpdateComponent(n1, n2)) {
    instance.next = n2;
    instance.update();
  } else {
    n2.el = n1.el;
    instance.vnode = n2;
  }
};
```

对于组件的更新会调用 `updateComponent` 方法，在 `updateComponent` 方法中会先判断组件是否需要更新。这里是一个优化点，对于不需要更新的组件直接就复用老的 `DOM`节点和 `VNode` 节点。对于需要更新的组件会先将新的 `VNode`挂载到 `next`属性上，再调用组件实例上的 `update` 方法进行更新。

判断组件是否需要更新的核心逻辑在 `shouldUpdateComponent` 中：

```typescript
// packages/runtime-core/src/componentRenderUtils.ts
/**
 * @author: Zhouqi
 * @description: 是否需要更新组件
 * @param n1 旧的虚拟节点
 * @param n2 新的虚拟节点
 */
export function shouldUpdateComponent(n1, n2) {
  const { props: prevProps, children: prevChildren } = n1;
  const { props: nextProps, children: nextChildren } = n2;

  if (prevChildren || nextChildren) {
    // 旧的子节点存在，新的子节点不存在，需要更新
    if (!nextChildren) {
      return true;
    }
  }

  if (prevProps === nextProps) {
    return false;
  }

  if (!prevProps) {
    // 旧的 `props` 不存在，新的 `props` 存在，需要更新，否测不需要更新
    return !!nextProps;
  }

  // 旧的 `props`存在，新的 `props`不存在，需要更新
  if (!nextProps) {
    return true;
  }

  return hasPropsChanged(prevProps, nextProps);
}

/**
 * @author: Zhouqi
 * @description: 比较新旧props是否变化
 * @param prevProps
 * @param nextProps
 */
function hasPropsChanged(prevProps, nextProps) {
  if (Object.keys(prevProps).length !== Object.keys(prevProps).length) {
    return false;
  }
  for (const key in nextProps) {
    if (nextProps[key] !== prevProps[key]) {
      return true;
    }
  }
  return false;
}
```

在这个方法中，判断组件需要更新的条件：

- 旧的子节点存在，新的子节点不存在，需要更新
- 旧的 `props` 不存在，新的 `props` 存在，需要更新
- 旧的 `props`存在，新的 `props`不存在，需要更新
- 新旧 `props` 都存在，并且属性有变化，需要更新



当调用了组件实例上的 `update` 方法后又会进入 effect 回调中：

```typescript
// packages/runtime-core/src/renderer.ts
const setupRenderEffect = (instance, initialVNode, container, anchor) => {
  const componentUpdateFn = () => {
    if (!instance.isMounted) {
      const subTree: Record<string, any> = (instance.subTree =
                                            renderComponentRoot(instance));
      patch(null, subTree, container, anchor, instance);
      initialVNode.el = subTree.el;
      instance.isMounted = true;
    } else {
      // 新增代码开始
      let { next, vnode } = instance;
      if (next) {
        // 更新组件的渲染数据
        next.el = vnode.el;
        updateComponentPreRender(instance, next);
      }
      // 新增代码结束
      const nextTree = renderComponentRoot(instance);
      const prevTree = instance.subTree;
      instance.subTree = nextTree;
      patch(prevTree, nextTree, container, anchor, instance);
    }
  };
  const effect = new ReactiveEffect(componentUpdateFn);
  const update = (instance.update = effect.run.bind(effect));
  update();
};
```

在回调中的更新逻辑内部，我们新增了组件存在 `next`属性值时的处理逻辑。`next`属性在上面提到过是新的 `VNode`对象，这里就需要根据新的 `VNode`对象来更新一下组件实例上的数据：

```typescript
// packages/runtime-core/src/renderer.ts  
/**
   * @author: Zhouqi
   * @description: 更新组件上面预渲染的数据
   * @param instance 组件实例
   * @param nextVnode 新的虚拟节点
   */
const updateComponentPreRender = (instance, nextVnode) => {
  nextVnode.component = instance;
  const prevProps = instance.vnode.props;
  instance.vnode = nextVnode;
  instance.next = null;
  // 更新props
  updateProps(instance, nextVnode.props, prevProps);
  // 省略插槽更新逻辑
};
```

`updateComponentPreRender`的核心逻辑就是更新实例上的数据，比如 `vnode`、`props`等等，核心还是 `props`的更新。

```typescript
// packages/runtime-core/src/componentProps.ts
/**
 * @author: Zhouqi
 * @description: 更新组件props
 * @param instance 组件实例
 * @param rawProps 新的props
 * @param rawPrevProps 旧的props
 */
export function updateProps(instance, rawProps, rawPrevProps) {
  const { props, attrs } = instance;
  // 简单一点，全量更新
  setFullProps(instance, rawProps, props, attrs);
  // 删除不存在的props
  for (const key in rawPrevProps) {
    if (!(key in rawProps)) {
      delete props[key];
    }
  }
}
```

我们自己实现的`updateProps`比较简单暴力，没有做什么比对逻辑，直接就是全量更新，在真正的源码中还是有各种比对更新的。



## 三、节点更新核心逻辑 patchChildren

最后我们来看一下 `patchChildren` 这个方法，它在普通元素以及 `Fragment`节点的更新中都使用了，`DOM`更新的核心逻辑就在这个方法中。

```typescript
/**
   * @author: Zhouqi
   * @description: 更新孩子节点
   * @param n1 旧的虚拟节点
   * @param n2 新的虚拟节点
   * @param container 容器
   * @param anchor 锚点元素
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
   */
  const patchChildren = (n1, n2, container, anchor, parentComponent, optimized = false) => {
    const c1 = n1.children;
    const c2 = n2.children;
    const prevShapeFlag = n1 ? n1.shapeFlag : 0;
    const { shapeFlag } = n2;

    // 更新的几种情况
    if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
      // 1. 新的子节点是一个文本节点，旧的子节点是一个数组，则删除旧的节点元素，然后创建新的文本节点
      if (prevShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
        unmountChildren(c1, parentComponent);
      }
      // 2. 旧的虚拟节点也是一个文本节点，但是文本内容不同，此时只需要更新文本内容
      if (c1 !== c2) {
        hostSetElementText(container, c2);
      }
    } else {
      // 走进这里说明新的孩子节点不存在或者是数组类型
      if (prevShapeFlag & ShapeFlags.ARRAY_CHILDREN) {
        if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
          // 3. 新旧孩子节点都是数组的情况下需要进行 dom diff，这种情况也是最复杂的
          patchKeyedChildren(c1, c2, container, anchor, parentComponent, optimized);
        } else {
          // 4. 新的节点不存在，则删除旧的子节点
          unmountChildren(c1, parentComponent);
        }
      } else {
        // 旧的孩子节点为文本节点。这种情况不管怎样，旧的文本节点都必须清空
        if (prevShapeFlag & ShapeFlags.TEXT_CHILDREN) {
          // 5. 旧的是一个文本节点，新的子节点不存在，将文本清空
          hostSetElementText(container, "");
        }
        if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
          // 6. 旧的是文本节点，新的是数组节点，则清空文本并创建新的子节点
          mountChildren(c2, container, anchor, parentComponent, optimized);
        }
      }
    }
  };
```

`patchChildren` 本意为更新子节点，Vue3 将子节点的更新分为 6 种情况：

1. 新的子节点是唯一文本节点，旧的子节点是一个数组 ———— 删除所有旧的节点并创建新的文本节点
2. 新旧都是文本子节点但是文本内容不同 ———— 更新节点文本内容
3. 新旧子节点都是数组 ———— 核心 `DOM Diff`的过程
4. 新的子节点不存在 ———— 删除所有旧的节点
5. 旧的是唯一文本子节点，新的子节点不存在 ———— 清空文本节点内容
6. 旧的是唯一文本子节点，新的节点是数组 ———— 清空文本节点并创建新的节点



在这 6 种情况中，第三种最为复杂，涉及到核心 `DOM Diff`的逻辑，我们放到下节讲解，剩下 5 种情况我们用 5 个例子来描述。

1. 新的子节点是唯一文本节点，旧的子节点是一个数组

```vue
<!-- 旧的节点 -->
<template>
	<div>
    <p>1</p>
    <p>2</p>
  </div>
</template>
<!-- 新的节点 -->
<template>
	<div>1</div>
</template>
```

更新时会先删除 `div`下的所有 `p`节点，再设置 `div`的文本内容为 `1`。

2. 新旧都是文本子节点但是文本内容不同

```vue
<!-- 旧的节点 -->
<template>
	<div>1</div>
</template>
<!-- 新的节点 -->
<template>
	<div>2</div>
</template>
```

更新时直接更新 `div`的文本内容为 `2`。

3. 新的子节点不存在

```vue
<!-- 旧的节点 -->
<template>
	<div>
    <p>1</p>
    <p>2</p>
  </div>
</template>
<!-- 新的节点 -->
<template>
	<div></div>
</template>
```

更新时删除 `div`下的所有 `p`节点。

4. 旧的是唯一文本子节点，新的子节点不存在

```vue
<!-- 旧的节点 -->
<template>
	<div>1</div>
</template>
<!-- 新的节点 -->
<template>
	<div></div>
</template>
```

更新时清空 `div`的文本内容。

5. 旧的是唯一文本子节点，新的节点是数组

```vue
<!-- 旧的节点 -->
<template>
	<div>1</div>
</template>
<!-- 新的节点 -->
<template>
	<div>
  	<p>1</p>
    <p>2</p>
  </div>
</template>
```

更新时先清空 `div`的文本内容，再创建 `p`节点。



## 四、总结

本章大致介绍了页面更新时各类节点的更新流程，其中核心`DOM` 比对更新逻辑在 `patchChildren` 中。`patchChildren`内部包含 6 种更新情况，其中新旧节点都是数组的情况最为复杂，涉及到核心 `Diff`算法，下一节将进入核心 `Diff`的实现流程。