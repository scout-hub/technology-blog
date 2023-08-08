# React 18源码解析（十二）—— PureComponent和memo



### 该部分解析基于我们实现的简单版react18中的代码，是react18源码的阉割版，希望用最简洁的代码来了解react的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



PureComponent 和 memo 都是 react 中用来优化组件更新的方法。



### 一、PureComponent

`PureComponent` 是 React 中的一个组件类，它是 `Component` 的一个变体。与 `Component` 不同，在组件更新时，`PureComponent` 会自动进行浅层比较，以决定是否需要重新渲染组件。如果 `PureComponent` 检测到 `props` 或 `state` 没有发生变化，它将阻止重新渲染，从而提高性能。

下面是一个示例：

```jsx
const { PureComponent, Component, useState } = React;

class Child extends PureComponent {
  constructor(props) {
    super(props);
    this.state = {
      num: 0,
      age: 14,
    };
  }

  render() {
    const { num } = this.state;
    return (
      <div>
        子节点num:{num}
      </div>
    );
  }
}

const App = () => {
  const [num, setNum] = useState(0);

  return (
    <div>
      <Child />
      父节点num:{num}
      <button
        onClick={() => {
          setNum(num + 1);
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

在示例中，父组件 App 里面嵌入了一个子组件 Child，但是子组件不接收任何 props。当点击父组件上的按钮时会触发父组件更新，如果 Child 继承的是 `React.Component`，则 Child 每次都会重新渲染；如果 Child 继承了`react.PureComponent`，则不会发生重新渲染。这就是 PureComponent 的作用。

下面我们看一下`React.PureComponent`的原理：

```typescript
// packages/react/src/ReactBaseClasses.ts
class PureComponent {
  public updater;
  public isPureReactComponent;
  constructor(public props) {}
}

const pureComponentPrototype = (PureComponent.prototype = Object.create(
  Component.prototype
));

pureComponentPrototype.constructor = PureComponent;

// 将Component原型上的属性方法合并过来，减少通过原型链去访问的开销
assign(pureComponentPrototype, Component.prototype);

// 标记是不是react class pureComponent
pureComponentPrototype.isPureReactComponent = true;
```

我们可以看大 PureComponent 也是继承了 Component 类，并且在原型上标记了 isPureReactComponent 为 true，表示自己是 PureComponent。

当组件发生更新且没有定义`shouldComponentUpdate`生命周期时 PureComponent 会自动进行浅比较：

```typescript
// packages/react-reconciler/src/ReactFiberClassComponent.ts
/**
 * @description: 检查组件是否需要更新，pureComponent浅比较以及shouldComponentUpdate生命周期函数
 */
function checkShouldComponentUpdate(
  workInProgress: Fiber,
  ctor: any,
  oldProps: any,
  newProps: any,
  oldState: any,
  newState: any
) {
  const instance = workInProgress.stateNode;
  const shouldComponentUpdate = instance.shouldComponentUpdate;
  // 调用shouldComponentUpdate生命周期函数
  if (isFunction(shouldComponentUpdate)) {
    const shouldUpdate = shouldComponentUpdate(newProps, newState);
    return shouldUpdate;
  }

  // pureComponent的浅比较
  const pureComponentPrototype = ctor.prototype;
  if (pureComponentPrototype && pureComponentPrototype.isPureReactComponent) {
    return (
      !shallowEqual(oldState, newState) || !shallowEqual(oldProps, newProps)
    );
  }

  return true;
}
```

以上就是 PureComponent 的实现原理，看上去并没有过多复杂的处理。



### 二、`React.memo`

`React.memo`是一个高阶组件，用于优化React函数组件的性能。它可以在组件的 props 没有改变的情况下避免不必要的重新渲染。当组件的 props 发生变化时，`React.memo`会对新旧 props 进行浅比较，如果它们相等，则组件不会重新渲染。如果它们不相等，则组件会重新渲染。这可以减少不必要的渲染，提高应用程序的性能。

`React.memo`接受两个参数：要优化的函数组件和一个可选的比较函数。比较函数用于自定义 props 的比较逻辑。如果没有提供比较函数，则默认使用浅比较。如果提供了比较函数，则该函数将在新旧 props 之间进行比较。如果比较函数返回`true`，则组件不会重新渲染。如果比较函数返回`false`，则组件将重新渲染。

下面是一个示例：

```jsx
const { useState, useMemo, useCallback, memo } = React;

const Child = memo(
  () => {
    console.log("re-render child");
    return <div>子组件</div>;
  },
  (prevProps, nextProps) =>
    prevProps.userInfo === nextProps.userInfo &&
    prevProps.click === nextProps.click
);

const App = () => {
  const [count, setCount] = useState(0);

  // const userInfo = { name: "zs", age: 14 };

  const userInfo = useMemo(() => ({ name: "zs", age: 14 }), []);

  const increment = () => {
    setCount(count + 1);
  };

  // const onClick = (userInfo) => {
  //   setUserInfo(userInfo);
  // };
  const onClick = useCallback((userInfo) => {
    setUserInfo(userInfo);
  }, []);

  return (
    <div>
      <button onClick={increment}>点击次数：{count}</button>
      {/* <Child /> */}
      {/* <Child count={count} /> */}
      <Child userInfo={userInfo} click={onClick} />
    </div>
  );
};

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

下面是`React.memo`的实现原理：

```typescript
// packages/react/src/ReactMemo.ts
export function memo(
  type,
  compare?: (oldProps: any, newProps: any) => boolean
) {
  // 包装了一个新的组件对象React.memo
  const elementType = {
    $$typeof: REACT_MEMO_TYPE,
    type,
    compare: compare === undefined ? null : compare,
  };
  return elementType;
}

// packages/react-reconciler/src/ReactFiber.ts
export function createFiberFromTypeAndProps(
  type: any,
  key: any,
  pendingProps: any,
  lanes: Lanes
): Fiber {
  let fiberTag: WorkTag = IndeterminateComponent;
  if (isFunction(type)) {
    // 判断是不是class组件
    if (shouldConstruct(type)) {
      fiberTag = ClassComponent;
    }
  } else if (isString(type)) {
    // 说明是普通元素节点
    fiberTag = HostComponent;
  } else {
    getTag: switch (type) {
      case REACT_FRAGMENT_TYPE: {
        return createFiberFromFragment(pendingProps.children, lanes, key);
      }
      default:
        if (isObject(type)) {
          switch (type.$$typeof) {
            case REACT_MEMO_TYPE:
              fiberTag = MemoComponent;
              break getTag;
          }
        }
    }
  }
```

`React.memo`组件的类型为`REACT_MEMO_TYPE`，当创建其 fiber 对象时，fiberTag 默认会被标记为`MemoComponent`。在`beginWork`阶段会对`MemoComponent`进行处理：

```typescript
// packages/react-reconciler/src/ReactFiberBeginWork.ts
export function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  // 省略其他代码
  switch (workInProgress.tag) {
    // 省略其他代码
    case MemoComponent: {
      const type = workInProgress.type;
      const resolvedProps = workInProgress.pendingProps;
      return updateMemoComponent(
        current,
        workInProgress,
        type,
        resolvedProps,
        renderLanes
      );
    }
    case SimpleMemoComponent: {
      return updateSimpleMemoComponent(
        current,
        workInProgress,
        workInProgress.type,
        workInProgress.pendingProps,
        renderLanes
      );
    }
  }
  return null;
}

/**
 * @description: 更新React.memo组件
 */
function updateMemoComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
) {
  // mount阶段
  if (current === null) {
    const type = Component.type;
    // 判断是不是简单的函数组件，即没有定义defaultProps以及只有浅比较的简单memo组件的一种快捷路径
    if (
      isSimpleFunctionComponent(type) &&
      Component.compare === null &&
      Component.defaultProps === undefined
    ) {
      // 标记SimpleMemoComponent的tag
      workInProgress.tag = SimpleMemoComponent;
      workInProgress.type = type;
      return updateSimpleMemoComponent(
        current,
        workInProgress,
        type,
        nextProps,
        renderLanes
      );
    }
    const child = createFiberFromTypeAndProps(
      Component.type,
      null,
      nextProps,
      renderLanes
    );
    child.return = workInProgress;
    workInProgress.child = child;
    return child;
  }
  // update阶段
  const currentChild = current.child!;
  const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
    current,
    renderLanes
  );
  // 没有任务需要执行
  if (!hasScheduledUpdateOrContext) {
    const prevProps = currentChild.memoizedProps;
    let compare = Component.compare;
    compare = compare !== null ? compare : shallowEqual;
    // 如果用户传入的compare函数执行结果为true，则直接bailout，否则进入更新
    if (compare(prevProps, nextProps)) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }

  const newChild = createWorkInProgress(currentChild, nextProps);
  newChild.return = workInProgress;
  workInProgress.child = newChild;
  return newChild;
}

// packages/react-reconciler/src/ReactFiber.ts
/**
 * @description: 判断是不是简单的函数组件
 */
export function isSimpleFunctionComponent(type: any) {
  return (
    isFunction(type) &&
    !shouldConstruct(type) &&
    type.defaultProps === undefined
  );
}

/**
 * @description: 通过原型上的isReactComponent判断是不是class组件
 */
function shouldConstruct(Component: Function) {
  const prototype = Component.prototype;
  return !!(prototype && prototype.isReactComponent);
}
```

在`updateMemoComponent`中会区分首次渲染和更新的情况。

对于首次渲染会先判断组件是不是一个简单的函数组件，即没有定义`defaultProps`且没有传入比较函数的函数式组件。这种 memo 组件被称为简单的 memo 组件，它的 tag 会被标记为`SimpleMemoComponent`。在更新阶段会对`SimpleMemoComponent`和非`SimpleMemoComponent`进行区分处理。

在更新阶段，对于非`SimpleMemoComponent`会执行传入的`compare`比较函数，如果返回`true`则直接 bailout。对于`SimpleMemoComponent`会走`updateSimpleMemoComponent`逻辑：

```typescript
// packages/react-reconciler/src/ReactFiberBeginWork.ts
/**
 * @description: 更新简单的memo组件
 */
function updateSimpleMemoComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
): null | Fiber {
  // TODO react在源码中注释说到，即使是mount阶段，current也有可能不是null（内部渲染暂停），这个问题需要评估是否会产生任何其他影响
  // 这里默认当作update阶段的执行
  if (current !== null) {
    const prevProps = current.memoizedProps;
    // 浅比较
    if (shallowEqual(prevProps, nextProps)) {
      didReceiveUpdate = false;
      // ？？？
      workInProgress.pendingProps = nextProps = prevProps;
      // 没有需要进行的工作了，直接bailout
      if (!checkScheduledUpdateOrContext(current, renderLanes)) {
        // TODO 理解lanes赋值这一步操作
        // workInProgress.lanes = current.lanes;
        return bailoutOnAlreadyFinishedWork(
          current,
          workInProgress,
          renderLanes
        );
      }
    }
  }
  return updateFunctionComponent(
    current,
    workInProgress,
    Component,
    nextProps,
    renderLanes
  );
}
```

在`updateSimpleMemoComponent`中会采用浅比较的方式进行比较，这就是`React.memo`的原理。