# React 18源码解析（十）—— Hooks（下）



### 该部分解析基于我们实现的简单版react18中的代码，是react18源码的阉割版，希望用最简洁的代码来了解react的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



上一章节介绍了 React Hooks 中的`useEffect`和`useLayoutEffect`的原理，这一章节将介绍`useCallback`和`useMemo`的原理，在最后我们会实现一个新的 hook `useDebounce`来巩固 hooks 学习。



### 一、useMemo

`useMemo` 是 React 中的一个 Hook，它可以用来缓存计算结果，避免重复计算。当你需要在组件中进行一些昂贵的计算时，可以使用 `useMemo` 来优化性能。它接受两个参数：第一个参数是一个函数，用来计算需要缓存的值；第二个参数是一个数组，用来指定依赖项，只有依赖项发生变化时，才会重新计算缓存的值。如果依赖项没有变化，则会直接返回缓存的值，避免重复计算。

下面是一个示例：

```jsx
const { useState, useMemo } = React;

const App = () => {
  const [count, setCount] = useState(0);
  const fib = (n) => n <= 1 ? 1 : fib(n - 1) + fib(n - 2);
  const num = useMemo(() => fib(34), []);
  return (
    <div>
      {count}-----{num}
      <button
        onClick={() => {
          setCount(count + 1);
        }}
      >
        更新
      </button>
    </div>
  );
};

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

在这个示例中，每次点击按钮都会触发组件更新，使用`useMemo`可以保证 fib(34) 不会重新计算，否则每次更新时都会重新计算 fib(34) 的结果。

下面看一下`useMemo`的实现原理，首先还是首次渲染时的逻辑`mountMemo`：

1. 生成`useMemo`对应的 hook 对象。
2. 初始化依赖 deps。
3. 执行计算函数 nextCreate，这是我们传入的计算函数。
4. 将计算结果和依赖信息保存到 hook 对象上的 memoizedState 属性中并将计算结果返回。

```typescript
// packages/react-reconciler/src/ReactFiberHooks.ts
/**
 * @description: mount阶段的useMemo
 */
function mountMemo<T>(nextCreate: () => T, deps: Array<any> | void | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

接着看更新时的逻辑`updateMemo`：

1. 获取`useMemo`对应的 hook 对象。
2. 初始化新的依赖 deps。
3. 比较前后依赖 deps 是否相同，如果相同则返回之前计算的值，否则重新调用计算函数 nextCreate 计算新的值并返回。

```typescript
// packages/react-reconciler/src/ReactFiberHooks.ts
/**
 * @description: update阶段的useMemo
 */
function updateMemo<T>(nextCreate: () => T, deps: Array<any> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps = prevState[1];
      // 如果新旧依赖相同，则返回之前的结果
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  // 否则重新执行函数，获取新的结果
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

我们可以看到`useMemo`就是简单的缓存逻辑。



### 二、useCallback

`useCallback`是React Hooks中的一个hook，它用于优化函数组件的性能。它的作用是返回一个记忆化的函数引用，当依赖项发生变化时，才会重新创建新的函数引用。这样可以避免在每次渲染时都创建新的函数引用，从而提高组件的性能。其中第一个参数是需要记忆化的函数，第二个参数是依赖项数组。只有当依赖项数组中的值发生变化时，才会重新创建新的函数引用。如果依赖项数组为空，则每次渲染都会返回同一个函数引用。

下面是一个例子：

```jsx
const { useCallback, useState, memo } = React;
const Child = memo(
  () => {
    console.log(1);
    return <div>Child</div>;
  },
  (prevProps, newProps) => newProps.callback !== prevProps.callback
);

const App = () => {
  const [state, setState] = useState(0);
  const callback = useCallback(() => {}, []);
  return (
    <div>
      {state}
      <Child callback={callback} />
      <button onClick={() => setState(state + 1)}>更新</button>
    </div>
  );
};

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);

```

在这个例子中，子组件 Child 是一个 Memo 组件（React.memo 会在后面介绍），它定义了组件需要重新渲染的逻辑`newProps.callback !== prevProps.callback`。父组件 App 中用`useCallback`定义了一个 callback 函数，这个函数会传递给子组件。当父组件中的 state 发生变化时，callback 函数不会重新生成，因此子组件也不会重新渲染。

下面看一下`useCallback`的原理，首先是首次渲染时的逻辑`mountCallback`：

1. 创建`useCallback`对应的 hook 对象。
2. 初始化依赖 deps。
3. 将用户传入的 callback 和 deps 依赖缓存到 memoizedState 属性中并返回 callback。

```typescript
// packages/react-reconciler/src/ReactFiberHooks.ts
/**
 * @description: mount阶段的useCallback
 */
function mountCallback<T>(callback: T, deps: Array<any> | void | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

接下去看更新时的逻辑`updateCallback`：

1. 获取`useCallback`对应的 hook 对象。
2. 初始化新的依赖 deps。
3. 比较前后依赖 deps 是否相同，如果相同则返回之前缓存的 callback，否则使用新创建的 callback 函数并返回。

```typescript
// packages/react-reconciler/src/ReactFiberHooks.ts
/**
 * @description: update阶段的useCallback
 */
function updateCallback<T>(callback: T, deps: Array<any> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null && nextDeps !== null) {
    // 取出依赖项进行比较
    const prevDeps: Array<any> | null = prevState[1];
    // 如果前后依赖相同，则返回第一次mount时候传入的callback
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];
    }
  }
  // 否则返回新的callback
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

我们可以看到`useCallback`和`useMemo`的逻辑几乎完全一致，只是一个用来缓存传入的函数或者说是缓存引用，一个用来缓存函数的结算结果。



### 三、自定义 Hook ——  useDebounce

看过前面几个 Hook 的原理之后，想必对 React Hooks 有了全面的认知，下面将实现一个新的 hook —— `useDebounce`。`useDebounce`的作用就是对传入的函数进行防抖处理。

首先是首次渲染时的`mountDebounce`：

```typescript
function mountDebounce<T>(
  callback: T,
  options: DebounceOptions = {},
  deps: Array<any> | void | null
): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const debounceCallback = debounce(callback, options) as any;
  hook.memoizedState = [debounceCallback, nextDeps];
  return debounceCallback;
}
```

在这里我们调用`debounce`方法对传入的 callback 进行防抖处理，然后缓存防抖后的函数并返回。

更新时的`udpdateDebounce`：

```typescript
function updateDebounce<T>(
  callback: T,
  options: DebounceOptions = {},
  deps: Array<any> | void | null
): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null && nextDeps !== null) {
    // 取出依赖项进行比较
    const prevDeps: Array<any> | null = prevState[1];
    // 如果前后依赖相同，则返回第一次mount时候传入的callback
    if (areHookInputsEqual(nextDeps, prevDeps)) {
      return prevState[0];
    }
  }
  // 否则返回新的callback
  const debounceCallback = debounce(callback, options) as any;
  hook.memoizedState = [debounceCallback, nextDeps];
  return debounceCallback;
}
```

道理和上面讲的`useMemo`和`useCallback`一样。

我们写一个例子来验证这个 hook 的作用：

```jsx
const { useDebounce, useState } = React;

const App = () => {
  const [state, setState] = useState(0);
  let num = 0;
  const callback = useDebounce(() => {
    document.getElementById("state").innerHTML = num++;
  }, {
    timeout: 1000,
  }, []);

  return (
    <div>
      按钮点击次数：{state}
      <button onClick={() => {
        setState(state + 1)
        callback()
      }}>更新</button>
    </div >
  );
};

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

![chrome-capture-2023-7-1](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-7-1.gif)

我们可以看到，div 内容的变化比较缓慢，显然防抖效果已经有了。