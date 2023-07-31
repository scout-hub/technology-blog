# React 18源码解析（九）—— Hooks（中）



### 该部分解析基于我们实现的简单版react18中的代码，是react18源码的阉割版，希望用最简洁的代码来了解react的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



上一章节介绍了 React Hooks 中的`useState`和`useReducer`的原理，这一章节将介绍`useEffect`和`useLayoutEffect`的原理。 



### 一、useEffect

`useEffect` 是 React 中的一个 Hook，它用于在函数组件中执行副作用操作。副作用操作指的是那些不直接与组件渲染相关的操作，例如访问 DOM 元素、发起网络请求、订阅事件等。

`useEffect` 接受两个参数：第一个参数是一个函数，用于执行副作用操作；第二个参数是一个数组，用于指定依赖项。当依赖项发生变化时，`useEffect` 会重新执行。如果第二个参数为空数组，那么 `useEffect` 只会在组件挂载和卸载时执行一次。如果第二个参数不为空数组，那么 `useEffect` 会在组件挂载时执行一次，并且在依赖项发生变化时重新执行。

下面是一个示例：

```jsx
const { useState, useEffect } = React;

const App = () => {
  const [num, setNum] = useState(0);

  useEffect(() => {
    console.log(num);
  }, [num]);

  return (
    <div className="red">
      <button
        onClick={(e) => {
          setNum(num + 1);
        }}
      >
        计数
      </button>
    </div>
  );
};

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

在这个示例中，点击按钮元素，num值会加一，控制台会打印出新的 num 值。



下面我们开始分析`useEffect`的原理，首先是首次渲染的`mountEffect`逻辑：

```typescript
// packages/react-reconciler/src/ReactFiberFlags.ts
export const Passive = 0b00000000000000100000000000;
export const PassiveStatic = 0b00100000000000000000000000;

// packages/react-reconciler/src/ReactHookEffectTags.ts
export const Passive = 0b1000;

// packages/react-reconciler/src/ReactFiberHooks.ts
/**
 * @description: mount阶段的useEffect函数
 */
function mountEffect(
  create: () => (() => void) | void,
  deps: Array<any> | null
) {
  return mountEffectImpl(
    PassiveEffect | PassiveStaticEffect,
    HookPassive,
    create,
    deps
  );
}

function mountEffectImpl(
  fiberFlags: Flags,
  hookFlags: HookFlags,
  create: () => (() => void) | void,
  deps: Array<any> | null
) {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 标记上passive的标记，这样在commit阶段就能执行副作用
  currentlyRenderingFiber!.flags |= fiberFlags;
  // useEffect的state就是effect
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    undefined,
    nextDeps
  );
}
```

在`mountEffect`中会调用`mountEffectImpl`方法，传入 fiberFlags、hookFlags、副作用函数 create 以及 依赖 deps。其中 create 副作用函数和 deps 依赖由开发者传入，fiberFlags 和 hookFlags 对应副作用标记，fiberFlags 用来表示组件存在 effect 副作用，hookFlags 用来表示组件中的 effect 副作用是否需要执行，类似于之前讲到过的 Placement 和 ChildDeletion。有了这些副作用标记，react 会在指定时机根据这些标记做对应的处理。

`mountEffectImpl`的逻辑：

1. 调用`mountWorkInProgressHook`创建对应的 hook 对象（跟之前讲的`useState`一样）。
2. 对 deps 值做初始化处理。
3. 给 fiber 标记上 HookPassive（Passive）副作用标记。
4. 调用`pushEffect`创建对应的 effect 对象，并将其保存到 hook 对象上的 memoizedState 属性中。



`pushEffect`逻辑：

1. 创建 effect 对象，这个对象上有 5 个属性：
   - tag: hookFlags 副作用标记
   - create：副作用函数
   - destory：销毁函数，也就是 effe ct 副作用函数的返回值
   - deps：依赖
   - next：指向下一个 effect 对象
2. 判断 fiber 上是不是存在 updateQueue，这个 updateQueue 保存的是该组件上所有的 effect 对象。如果不存在，则调用`createFunctionComponentUpdateQueue`创建一个 updateQueue，将 effect 对象保存到 lastEffect 属性中，并且自身和自身形成环状链表；如果存在，则追加 effect 并构成环状链表。

```typescript
// packages/react-reconciler/src/ReactFiberHooks.ts
function pushEffect(
  tag: HookFlags,
  create: () => (() => void) | void,
  destroy: (() => void) | void,
  deps: Array<any> | null
): Effect {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    next: null as any,
  };
  let componentUpdateQueue: null | FunctionComponentUpdateQueue =
    currentlyRenderingFiber!.updateQueue;
  // 创建updateQueue不存在的话则创建一个updateQueue
  if (componentUpdateQueue === null) {
    componentUpdateQueue = createFunctionComponentUpdateQueue();
    currentlyRenderingFiber!.updateQueue = componentUpdateQueue;
    // 自身和自身构成循环链表
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      effect.next = firstEffect;
      lastEffect.next = effect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect;
}
```

现在我们知道了`useEffect`初始化的逻辑，那么`useEffect`的副作用函数是在什么时候执行的？答案就在 commit 流程中，在之前介绍 commit 流程的时候我们省略了 effect 副作用执行的逻辑：

1. 根据 flags 和 subtreeFlags 判断当前 fiber 或者子 fiber 上是不是存在 effect 副作用，显然在调用`useEffect`的时候就会被标记上 Passive。
2. 如果需要执行副作用则标记 rootDoesHavePassiveEffects 为 true。
3. 用 Scheduler 调度 flushPassiveEffects，调度优先级为 NormalSchedulerPriority。Scheduler 调度我们之前也介绍过，采用 postMessage 的方式注册宏任务进行任务处理。这也就意为着`useEffect`的副作用函数会在布局完成后执行（下一帧），不会阻塞本次渲染。
4. 假如本次提交存在副作用，则会将 rootWithPendingPassiveEffects 赋值为 root。

```typescript
// packages/react-reconciler/src/ReactFiberFlags.ts
export const PassiveMask = Passive | ChildDeletion;

// packages/react-reconciler/src/ReactFiberWorkLoop.ts
let rootDoesHavePassiveEffects: boolean = false;

function commitRootImpl(root: FiberRoot) {
  const finishedWork = root.finishedWork;
  // 省略其他代码
  // 处理effect副作用
  if (
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags
  ) {
    if (!rootDoesHavePassiveEffects) {
      rootDoesHavePassiveEffects = true;
      scheduleCallback(NormalSchedulerPriority, () => {
        flushPassiveEffects();
        return null;
      });
    }
  }
  
  // 省略其他代码

  // 本次提交存在副作用，在布局完成后去调度这些副作用回调
  if (rootDoesHavePassiveEffects) {
    rootDoesHavePassiveEffects = false;
    rootWithPendingPassiveEffects = root;
  }
}
```

当调度触发时会执行`flushPassiveEffects`：

```typescript
// packages/react-reconciler/src/ReactFiberCommitWork.ts
/**
 * @description: 处理effect副作用，返回是否存在副作用的标志
 */
function flushPassiveEffects() {
  if (rootWithPendingPassiveEffects !== null) {
    try {
      return flushPassiveEffectsImpl();
    } catch (error: any) {
      throw Error(error);
    }
  }
  return false;
}

function flushPassiveEffectsImpl() {
  if (rootWithPendingPassiveEffects === null) return false;
  const root = rootWithPendingPassiveEffects;
  rootWithPendingPassiveEffects = null;
  // 对之前的副作用进行清理（执行destory）
  commitPassiveUnmountEffects(root.current);
  // 处理新生成的副作用
  commitPassiveMountEffects(root, root.current);
  return true;
}
```

`flushPassiveEffects`的处理也是一个递和归的过程。首次渲染先不考虑`commitPassiveUnmountEffects`，这个方法的作用是执行之前的 destroy 销毁函数。这里主要看`commitPassiveMountEffects`处理新生成的 effect 副作用。在`commitPassiveMountEffects`：

```typescript
// packages/react-reconciler/src/ReactFiberCommitWork.ts
export function commitPassiveMountEffects(
  root: FiberRoot,
  finishedWork: Fiber
) {
  nextEffect = finishedWork;
  commitPassiveMountEffects_begin(finishedWork, root);
}

function commitPassiveMountEffects_begin(subtreeRoot: Fiber, root: FiberRoot) {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const firstChild = fiber.child;
    // 找到第一个subtreeFlags中不存在PassiveMask副作用标记的节点
    if ((fiber.subtreeFlags & PassiveMask) !== NoFlags && firstChild !== null) {
      firstChild.return = fiber;
      nextEffect = firstChild;
    } else {
      commitPassiveMountEffects_complete(subtreeRoot, root);
    }
  }
}
```

在`commitPassiveMountEffects`中会调用`commitPassiveMountEffects_begin`方法，这个方法采用深度优先遍历的方式找到第一个 subtreeFlags 中不存在PassiveMask 副作用标记的节点（递的过程）。找到该节点后会调用`commitPassiveMountEffects_complete`：

```typescript
// packages/react-reconciler/src/ReactFiberCommitWork.ts
function commitPassiveMountEffects_complete(
  subtreeRoot: Fiber,
  root: FiberRoot
) {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    // 如果当前fiber存在Passive副作用标记，则去处理副作用
    if ((fiber.flags & Passive) !== NoFlags) {
      try {
        commitPassiveMountOnFiber(root, fiber);
      } catch (error: any) {
        throw Error(error);
      }
    }

    // 处理完了
    if (fiber === subtreeRoot) {
      nextEffect = null;
      return;
    }

    // 处理兄弟节点的副作用
    const sibling = fiber.sibling;
    if (sibling !== null) {
      sibling.return = fiber.return;
      nextEffect = sibling;
      return;
    }

    // 处理父fiber的副作用
    nextEffect = fiber.return;
  }
}
```

在`commitPassiveMountEffects_complete`中循环 nextEffect，也就是 fiber。判断 fiber 上是否被标记了 Passive，如果被标记了则会执行`commitPassiveMountOnFiber`；如果没有，则判断 fiber 是不是遍历完，遍历完了则直接结束。否则继续处理处理兄弟节点的副作用，兄弟节点也处理完了则会网上处理父节点（归的过程）。接下去看`commitPassiveMountOnFiber`的逻辑：

```typescript
// packages/react-reconciler/src/ReactFiberCommitWork.ts
/**
 * @description: 提交副作用处理
 */
function commitPassiveMountOnFiber(
  finishedRoot: FiberRoot,
  finishedWork: Fiber
) {
  switch (finishedWork.tag) {
    case FunctionComponent: {
      commitHookEffectListMount(HookPassive | HookHasEffect, finishedWork);
      break;
    }
    case HostRoot: {
      throw Error("HostRoot");
    }
  }
}
```

在`commitPassiveMountOnFiber`中会根据 fiber 的类型走不同的副作用处理逻辑，在`useEffect`中我们只考虑函数式组件的处理逻辑，也就是`commitHookEffectListMount`：

```typescript
// packages/react-reconciler/src/ReactFiberCommitWork.ts
function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    finishedWork.updateQueue;
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // 取出传入的副作用函数
        const create = effect.create;
        // destory就是用户传入的副作用函数中的返回值，这个返回值就是销毁函数
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

在`commitHookEffectListMount`中会先获取 fiber 上的 updateQueue。上面提到过，所有的 effect 会保存到 updateQueue 上的 lastEffect 属性中。接着这循环 effect 环状链表，如果对应的 effect 副作用需要执行（effect.tag & (HookPassive | HookHasEffect) === HookPassive | HookHasEffect），则会调用 effect 对象上的 create 副作用函数，其返回值会作为销毁函数保存到 effect.destroy 中。

以上便是`useEffect`在首次渲染时的逻辑，接下去看更新时的处理`updateEffect`：

1. 调用`updateWorkInProgressHook`获取当前`useEffect`对应的 hook 对象。
2. 初始化更新时的 deps。
3. 判断前后依赖的 deps 是否相同，如果相同则会添加一个没有 HookHasEffect 标记的 effect 对象。在上面介绍的`commitHookEffectListMount`中我们提到过，只有 effect.tag & (HookPassive | HookHasEffect) === HookPassive | HookHasEffect 才会执行副作用，也就是 HookPassive 和 HookHasEffect 都要存在。因此，如果依赖没有变化就不会添加 HookHasEffect 标记，后续也就不会执行副作用回调；如果依赖不同，则会添加一个有 HookHasEffect 标记的 effect 对象，并且获取旧的 effect 上的 destroy 值作为自身的 destroy 值。

```typescript
// packages/react-reconciler/src/ReactFiberHooks.ts
/**
 * @description: update阶段的useEffet函数
 */
function updateEffect(
  create: () => (() => void) | void,
  deps: Array<any> | null
) {
  return updateEffectImpl(PassiveEffect, HookPassive, create, deps);
}

function updateEffectImpl(
  fiberFlags: Flags,
  hookFlags: HookFlags,
  create: () => (() => void) | void,
  deps: Array<any> | null
) {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  let destroy;
  if (currentHook !== null) {
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      // 前后依赖项相同
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 这里的pushEffect第一个参数没有添加HookHasEffect标记，所以不会执行副作用
        hook.memoizedState = pushEffect(hookFlags, create, destroy, nextDeps);
        return;
      }
    }
  }
  currentlyRenderingFiber!.flags |= fiberFlags;
  // 标记了HookHasEffect会在commit阶段触发副作用
  hook.memoizedState = pushEffect(
    HookHasEffect | hookFlags,
    create,
    destroy,
    nextDeps
  );
}
```

后续 commit 阶段的处理逻辑同首次渲染类似，只不过多了我们之前省略的`commitPassiveUnmountEffects`逻辑：

```typescript
// packages/react-reconciler/src/ReactFiberCommitWork.ts
/**
 * @description: 副作用销毁
 */
export function commitPassiveUnmountEffects(firstChild: Fiber): void {
  nextEffect = firstChild;
  commitPassiveUnmountEffects_begin();
}

function commitPassiveUnmountEffects_begin() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    const child = fiber.child;
    
    if ((fiber.subtreeFlags & PassiveMask) !== NoFlags && child !== null) {
      child.return = fiber;
      nextEffect = child;
    } else {
      commitPassiveUnmountEffects_complete();
    }
  }
}

function commitPassiveUnmountEffects_complete() {
  while (nextEffect !== null) {
    const fiber = nextEffect;
    if ((fiber.flags & Passive) !== NoFlags) {
      commitPassiveUnmountOnFiber(fiber);
    }

    // 处理兄弟节点
    const sibling = fiber.sibling;
    if (sibling !== null) {
      sibling.return = fiber.return;
      nextEffect = sibling;
      return;
    }

    // 处理父节点
    nextEffect = fiber.return;
  }
}

function commitPassiveUnmountOnFiber(finishedWork: Fiber) {
  switch (finishedWork.tag) {
    case FunctionComponent: {
      commitHookEffectListUnmount(HookPassive | HookHasEffect, finishedWork);
      break;
    }
  }
}

function commitHookEffectListUnmount(flags: HookFlags, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    finishedWork.updateQueue;
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        const destroy = effect.destroy;
        effect.destroy = undefined;
        // 执行上一个effect返回的销毁函数
        if (destroy != null) {
          destroy();
        }
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

`commitPassiveUnmountEffects`和`commitPassiveMountEffects`类似，也是一个递和归的过程，不同的地方在于`commitHookEffectListUnmount`中执行的只有上一次调用副作用函数后返回的销毁函数。





### 二、useLayoutEffect

`useLayoutEffect` 是 React 中的一个 Hook，它类似于 `useEffect`，但是它会在所有 DOM 变更完成后同步执行。这意味着它会在浏览器有机会绘制屏幕之前运行，这对于在元素呈现之前测量元素的大小或位置非常有用。

下面是一个示例：

```jsx
import React, { useLayoutEffect, useRef } from 'react';

function MyComponent() {
  const ref = useRef(null);

  useLayoutEffect(() => {
    console.log(ref.current.getBoundingClientRect());
  }, []);

  return <div ref={ref}>Hello, world!</div>;
}
```

在这个示例中，`useLayoutEffect` 用于在渲染 `div` 元素之前记录其大小和位置。`useRef` hook 用于创建对 `div` 元素的引用，然后将其传递给 `div` 的 `ref` 属性。

我们看一下`useLayoutEffect`的原理，首先是首次渲染时的逻辑`mountLayoutEffect`：

```typescript
// packages/react-reconciler/src/ReactFiberFlags.ts
export const Update = 0b00000000000000000000000100;

// packages/react-reconciler/src/ReactHookEffectTags.ts
export const Layout = 0b0100;

// packages/react-reconciler/src/ReactFiberHooks.ts
/**
 * @description: mount阶段的useLayoutEffect
 */
function mountLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<any> | null
) {
  let fiberFlags: Flags = UpdateEffect;
  return mountEffectImpl(fiberFlags, HookLayout, create, deps);
}
```

我们可以看到`mountLayoutEffect` 中调用的也是`mountEffectImpl`，和`useEffect`一样，只是传入的 fiberFlags 和 hookFlags 是对应的 UpdateEffect 和 HookLayout。`mountEffectImpl`的逻辑上面已经介绍过了，这里就不再分析。

接下来就是`useLayoutEffect`回调执行的时机，它在 commit 阶段中的 layout 阶段执行的：

```typescript
// packages/react-reconciler/src/ReactFiberWorkLoop.ts
function commitRootImpl(root: FiberRoot) {
	// layout阶段，componentDidMount、componentDidUpdate调用的地方
	commitLayoutEffects(finishedWork, root);
}

// packages/react-reconciler/src/ReactFiberCommitWork.ts
/**
 * @description: 提交layout阶段的副作用处理
 */
export function commitLayoutEffects(
  finishedWork: Fiber,
  root: FiberRoot
): void {
  nextEffect = finishedWork;
  commitLayoutEffects_begin(finishedWork, root);
}

function commitLayoutEffects_begin(subtreeRoot: Fiber, root: FiberRoot) {
  while (nextEffect !== null) {
    let fiber = nextEffect;
    let firstChild = fiber.child;
    // 找到第一个subtreeFlags中不存在LayoutMask副作用标记的节点
    if ((fiber.subtreeFlags & LayoutMask) !== NoFlags && firstChild !== null) {
      firstChild.return = fiber;
      nextEffect = firstChild;
    } else {
      commitLayoutMountEffects_complete(subtreeRoot, root);
    }
  }
}

function commitLayoutMountEffects_complete(
  subtreeRoot: Fiber,
  root: FiberRoot
) {
  while (nextEffect !== null) {
    const fiber = nextEffect;

    if ((fiber.flags & LayoutMask) !== NoFlags) {
      try {
        commitLayoutEffectOnFiber(root, fiber.alternate, fiber);
      } catch (error: any) {
        throw Error(error);
      }
    }

    if (fiber === subtreeRoot) {
      nextEffect = null;
      return;
    }

    // 处理兄弟节点
    const sibling = fiber.sibling;
    if (sibling !== null) {
      sibling.return = fiber.return;
      nextEffect = sibling;
      return;
    }
    // 处理父节点
    nextEffect = fiber.return;
  }
}

function commitLayoutEffectOnFiber(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber
) {
  if ((finishedWork.flags & LayoutMask) !== NoFlags) {
    switch (finishedWork.tag) {
      case FunctionComponent: {
        commitHookEffectListMount(HookLayout | HookHasEffect, finishedWork);
        break;
      }
      case ClassComponent: {
        // 省略其他代码
      }
    }
  }
}

function commitHookEffectListMount(flags: HookFlags, finishedWork: Fiber) {
  const updateQueue: FunctionComponentUpdateQueue | null =
    finishedWork.updateQueue;
  const lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & flags) === flags) {
        // 取出传入的副作用函数
        const create = effect.create;
        // destory就是用户传入的副作用函数中的返回值，这个返回值就是销毁函数
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
```

`commitLayoutEffects`的逻辑也基本和`useEffect`一致，也是一个递和归的过程，不同的地方在于判断的 flags 和 subTreeFlags 以及 effect.tag 的值都换成了`useLayoutEffect`所需要的标记。这部分逻辑也不再需要赘述。

同理，更新时的`updateLayoutEffect`也跟`updateEffect`类似，也都是调用`updateEffectImpl`，区别只在于标记的不同：

```typescript
// packages/react-reconciler/src/ReactFiberHooks.ts
/**
 * @description: update阶段的useLayoutEffect
 */
function updateLayoutEffect(
  create: () => (() => void) | void,
  deps: Array<any> | null
) {
  return updateEffectImpl(UpdateEffect, HookLayout, create, deps);
}
```

既然`useEffect`和`useLayouEffect`逻辑上几乎一样，那么为什么需要两个 hooks？

其实它们最大的区别就是执行时机。`useEffect`的回调会被注册到下一个宏任务中执行，不会阻塞渲染，因此适用于大多数情况下的副作用处理，比如网络请求、DOM操作等。`useLayoutEffect`会在组件渲染完成后同步执行，会阻塞组件的渲染，因此适用于需要同步更新DOM的情况，比如获取DOM元素的尺寸、位置等。

我们用一个例子来模拟它们之前的区别：

```jsx
const { useState, useEffect, useLayoutEffect } = React;

const App = () => {
  const [direction, setDirection] = useState("vertical");

  // 视图会有一个变化的过程（闪烁）
  useEffect(() => {
    let i = 0;
    while(i <= 1000000000) {
      i++;
    }
    setDirection("column");
  }, [direction]);

  return (
    <div
      style={{
        display: "flex",
        justifyContent: "space-around",
        flexDirection: direction,
        flexWrap: "wrap",
        width: 600,
      }}
    >
      {Array.from({ length: 30 }).map((item, index) => {
        const color = "#" + Math.random().toString(16).slice(2, 8);
        return (
          <div
            key={index}
            style={{
              width: 100,
              height: 100,
              margin: 10,
              backgroundColor: color,
            }}
          >
            {index}
          </div>
        );
      })}
    </div>
  );
};

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

这个例子的主要功能就是在页面初始化渲染完成后再触发一次更新来改变布局。首先是`useEffect`中来触发视图更新：

![test123](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/test123.gif)

当我们刷新页面可以明显看到视图变化的过程，这就是上面说的，`useEffect`不影响本次渲染。接下去看一下`useLayoutEffect`的效果：

```jsx
const { useState, useEffect, useLayoutEffect } = React;

const App = () => {
  const [direction, setDirection] = useState("vertical");

  // 直接呈现执行useLayoutEffect后的视图
	useLayoutEffect(() => {
    let i = 0;
    while(i <= 1000000000) {
      i++;
    }
    setDirection("column");
  }, [direction]);

  return (
    <div
      style={{
        display: "flex",
        justifyContent: "space-around",
        flexDirection: direction,
        flexWrap: "wrap",
        width: 600,
      }}
    >
      {Array.from({ length: 30 }).map((item, index) => {
        const color = "#" + Math.random().toString(16).slice(2, 8);
        return (
          <div
            key={index}
            style={{
              width: 100,
              height: 100,
              margin: 10,
              backgroundColor: color,
            }}
          >
            {index}
          </div>
        );
      })}
    </div>
  );
};

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

![testxx.gif](https://github.com/scout-hub/picgo-bed/blob/dev/testxx.gif?raw=true)

当我们刷新页面可以看到并没有看到结构的变化，也就是说`useLayoutEffect` 回调的执行是在本次渲染之前执行的，会阻塞本次渲染。