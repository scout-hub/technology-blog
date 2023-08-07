# React 18源码解析（十一）—— 类组件



### 该部分解析基于我们实现的简单版react18中的代码，是react18源码的阉割版，希望用最简洁的代码来了解react的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



本章节我们将介绍 React 类组件及其内部相关方法的实现原理。



### 一、类组件

React 类组件是一种 React 组件类型，它是通过继承 React.Component 类来定义的。类组件可以包含状态 (state) 和生命周期方法 (lifecycle methods)，并且可以通过 render 方法来渲染组件的 UI。类组件通常用于处理复杂的 UI 逻辑和状态管理。

下面是一个简单的类组件示例：

```jsx
const { Component } = React;

class Child extends Component {
  constructor(props) {
    super(props);
    this.state = {
      title: "Child",
    };
  }

  render() {
    const { title } = this.state;
    return <div>{title}</div>;
  }
}
```

下面我们将分析类组件的实现原理。

首先，所有 react 类组件都是继承自 React.Component：

```typescript
// packages/react/src/ReactBaseClasses.ts
class Component {
  public isReactComponent;
  public updater;
  constructor(public props) {}

  setState(partialState, callback) {
    this.updater.enqueueSetState(this, partialState, callback);
  }
}

// 标记是不是react class component
Component.prototype.isReactComponent = true;
```

在 Component 类中有两个属性和一个方法：

- isReactComponent： 标记是不是 react class component
- updater：更新器对象
- setState：更新组件状态，内部会调用更新器上的方法去调度更新，当我们自定义的类组件继承 React.Component 后就可以使用`this.setState`去更新组件状态。

接着就是类组件被解析的过程。首先，类组件会被 react.jsx 编译为 jsx 对象，这个 jsx 对象在`beginWork`阶段被解析成对应的 fiber 节点，最后生成对应的 DOM 节点并渲染到页面上。类组件被处理的过程主要在`beginWork`阶段：

```typescript
// packages/react-reconciler/src/ReactFiberBeginWork.ts
export function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
) {
  // 省略其他代码
  switch (workInProgress.tag) {
      case ClassComponent: {
        const Component = workInProgress.type;
        const resolvedProps = workInProgress.pendingProps;
        return updateClassComponent(
          current,
          workInProgress,
          Component,
          resolvedProps,
          renderLanes
        );
    }
  }
}
```

在`beginWork`阶段，react 会根据 fiber 的类型做不同的处理，对于类组件来说会调用`updateClassComponent`：

```typescript
// packages/react-reconciler/src/ReactFiberBeginWork.ts
/**
 * @description: 更新class组件
 */
function updateClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes
) {
  const instance = workInProgress.stateNode;
  let shouldUpdate;
  // 实例不存在的情况
  if (instance === null) {
    constructClassInstance(workInProgress, Component, nextProps);
    mountClassInstance(workInProgress, Component, nextProps, renderLanes);
    shouldUpdate = true;
  } else if (current === null) {
    // 存在实例，但是没有current fiber
    throw Error("updateClassComponent current is null");
  } else {
    // update
    shouldUpdate = updateClassInstance(
      current,
      workInProgress,
      Component,
      nextProps,
      renderLanes
    );
  }
  // render
  const nextUnitOfWork = finishClassComponent(
    current,
    workInProgress,
    Component,
    shouldUpdate,
    false,
    renderLanes
  );
  return nextUnitOfWork;
}
```

在`updateClassComponent`中会根据 fiber 上的 `stateNode`属性区分首次渲染和更新两个阶段，`stateNode`存储的就是组件实例。首次渲染时，`stateNode`不存在，会先调用`constructClassInstance`创建组件实例：

```typescript
// packages/react-reconciler/src/ReactFiberClassComponent.ts
/**
 * @description: 生成class组件实例
 */
export function constructClassInstance(
  workInProgress: Fiber,
  ctor: any,
  props: any
) {
  let instance = new ctor(props);
  // 获取组件实例上定义的state
  const state = instance.state;
  workInProgress.memoizedState = state == null ? null : state;
  adoptClassInstance(workInProgress, instance);
  return instance;
}
```

通过 new 的方式就可以创建组件实例，同时将实例上的`state`状态作为 fiber 的`memoizedState`。创建完实例后需要调用`adoptClassInstance`绑定相关属性并将组件实例和对应的 fiber 进行绑定，为了后续能访问到组件实例。

```typescript
// packages/shared/src/ReactInstanceMap.ts
// setInstance
export function set(key, value) {
  key._reactInternals = value;
}

// packages/react-reconciler/src/ReactFiberClassComponent.ts
function adoptClassInstance(workInProgress: Fiber, instance: any): void {
  instance.updater = classComponentUpdater;
  workInProgress.stateNode = instance;
  // 在实例上绑定当前class组件的fiber，在需要调度的时候可以获取这个fiber
  setInstance(instance, workInProgress);
}
```

当组件实例创建完成后就需要挂载组件实例，挂载逻辑就在`mountClassInstance`中：

```typescript
// packages/react-reconciler/src/ReactFiberClassComponent.ts
/**
 * @description: 挂载class组件实例
 */
export function mountClassInstance(
  workInProgress: Fiber,
  ctor: any,
  newProps: any,
  renderLanes: Lanes
) {
  const instance = workInProgress.stateNode;
  instance.props = newProps;
  instance.state = workInProgress.memoizedState;
  // 初始化更新队列
  initializeUpdateQueue(workInProgress);
  // 省略其他代码
}
```

这块逻辑主要就是绑定一些属性，比如`state`、`props`以及初始化更新队列`updateQueue`。这里省略了生命周期相关的代码，这块会在下面介绍。

当执行完挂载逻辑后就需要执行渲染函数去生成子 fiber，也就是执行组件内部的 render 函数，这部分逻辑就在`finishClassComponent`中：

1. 判断是否需要更新，不需要的话直接 bailout。bailout 的逻辑很简单，判断当前 fiber 的子 fiber 是否有需要执行的任务，如果没有直接返回 null。
2. 获取组件实例，调用实例上的 render 方法创建子 jsx 对象，调用`reconcileChildren`进行子 fiber 的生成并返回第一个子 fiber。

```typescript
// packages/react-reconciler/src/ReactFiberBeginWork.ts
/**
 * @description: 执行class组件的最终渲染
 */
function finishClassComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  shouldUpdate: boolean,
  hasContext: boolean,
  renderLanes: Lanes
) {
  // 不需要更新，直接bailout
  if (!shouldUpdate) {
    return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
  }
  const instance = workInProgress.stateNode;
  const nextChildren = instance.render();
  // 处理子节点
  reconcileChildren(current, workInProgress, nextChildren, renderLanes);
  return workInProgress.child;
}


function bailoutOnAlreadyFinishedWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes
): Fiber | null {
  // 判断子fiber是否有任务需要进行，如果也没有，则直接返回
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) return null;
  // 当前这个fiber不需要工作了，但是它的子fiber还需要工作，这里克隆一份子fiber
  cloneChildFibers(current, workInProgress);
  return workInProgress.child;
}
```

以上便是类组件首次渲染时的逻辑，更新逻辑会结合`this.setState`进行讲解。



### 二、生命周期

React 类组件生命周期是一组方法，它们在组件的不同阶段被调用。这些方法允许我们在组件的不同生命周期阶段执行操作，例如在组件挂载时执行某些操作，或在组件更新时执行某些操作。以下是 React 类组件的生命周期方法（react 16 之后）：

- `constructor`：在组件被创建时调用，用于初始化组件的状态和绑定事件处理程序
- `static getDerivedStateFromProps`：在组件接收到新的 props 时调用，用于更新组件的状态
- `render`：在组件挂载或更新时调用，用于渲染组件的UI
- `componentDidMount`：在组件挂载后调用，用于执行一些需要DOM节点的操作，例如获取数据或添加事件监听器
- `shouldComponentUpdate`：在组件更新前调用，用于控制组件是否需要更新，可以通过返回 false 来阻止组件更新
- `getSnapshotBeforeUpdate`：在组件更新前调用，用于获取组件更新前的快照
- `componentDidUpdate`：在组件更新后调用，用于执行一些需要DOM节点的操作，例如更新数据或添加事件监听器
- `componentWillUnmount`：在组件卸载前调用，用于清理组件的状态和资源



还有一些生命周期是 react 16 之前的，在 react 16 之后这些生命周期依然可以使用，但是被 react 标记为不安全的生命周期，因此尽量不要使用：

`componentWillMount`、`componentWillReceiveProps`、`componentWillUpdate`。



下面分析生命周期的原理。

首次渲染阶段：

- 在`constructClassInstance`中通过`new ctor(props)`实例化时会先执行组件内部的`constructor`函数。
- 在`mountClassInstance`中会尝试获取实例上的`getDerivedStateFromProps`这个生命周期方法，如果定义了则调用`applyDerivedStateFromProps`。在`applyDerivedStateFromProps`中会调用实例上的`getDerivedStateFromProps`方法并传入 props 和 state，方法的返回值如果不是 null 或者 undefined 则会和当前 state 进行数据合并作为新的 state。
- 判断是否使用了新的生命周期 API（`getDerivedStateFromProps`，`getSnapshotBeforeUpdate`），如果没有使用且定义了`componentWillMount`则会调用`callComponentWillMount`。`callComponentWillMount`中会执行实例上的`componentWillMount`方法，在`componentWillMount`中可能会对 state 进行操作，如果我们通过 this.state = xxx 直接对 state 进行赋值操作的话，react 会调用`enqueueReplaceState`去替换 state（这个方法会和`setState`一起介绍）。
  由于`callComponentWillMount`中可能存在状态更新，因此需要调用`processUpdateQueue`去处理这些更新。
- 如果定义了`componentDidMount`生命周期，则会在 fiber 上添加 Update 副作用标记。这里不像上面一样直接执行生命周期函数是因为`componentDidMount`需要在组件挂载完成后执行，也就是 commit 阶段中的 layout 阶段执行。
- 最后在`finishComponent`中执行组件的 render 方法。

````typescript
// packages/react-reconciler/src/ReactFiberClassComponent.ts
/**
 * @description: 生成class组件实例
 */
export function constructClassInstance(
  workInProgress: Fiber,
  ctor: any,
  props: any
) {
  let instance = new ctor(props);
  // 获取组件实例上定义的state
  const state = instance.state;
  workInProgress.memoizedState = state == null ? null : state;
  adoptClassInstance(workInProgress, instance);
  return instance;
}

/**
 * @description: 挂载class组件实例
 */
export function mountClassInstance(
  workInProgress: Fiber,
  ctor: any,
  newProps: any,
  renderLanes: Lanes
) {
  const instance = workInProgress.stateNode;
  instance.props = newProps;
  instance.state = workInProgress.memoizedState;
  // 初始化更新队列
  initializeUpdateQueue(workInProgress);
  // class（不是实例上获取）上获取getDerivedStateFromProps生命周期函数
  const getDerivedStateFromProps = ctor.getDerivedStateFromProps;

  if (isFunction(getDerivedStateFromProps)) {
    // 执行getDerivedStateFromProps生命周期函数，这个函数是在finishClassComponent（render）之前执行的
    applyDerivedStateFromProps(
      workInProgress,
      ctor,
      getDerivedStateFromProps,
      newProps
    );
    // 更新instance上的state
    instance.state = workInProgress.memoizedState;
  }

  // 判断是否有新的生命周期函数getDerivedStateFromProps、getSnapshotBeforeUpdate
  const hasNewLifecycles =
    isFunction(getDerivedStateFromProps) ||
    isFunction(instance.getSnapshotBeforeUpdate);

  if (!hasNewLifecycles && isFunction(instance.componentWillMount)) {
    callComponentWillMount(workInProgress, instance);
    // componentWillMount可能存在额外的状态更新，这里去处理这些状态更新
    processUpdateQueue(workInProgress, newProps, instance, renderLanes);
    // 更新一下实例上的state
    instance.state = workInProgress.memoizedState;
  }

  // 如果componentDidMount存在，标记上Update
  if (isFunction(instance.componentDidMount)) {
    workInProgress.flags |= Update;
  }
}

function applyDerivedStateFromProps(
  workInProgress: Fiber,
  ctor: any,
  getDerivedStateFromProps: (props: any, state: any) => any,
  nextProps: any
) {
  const prevState = workInProgress.memoizedState;
  // 执行getDerivedStateFromProps函数获取新的state
  const partialState = getDerivedStateFromProps(nextProps, prevState);
  // 如果partialState是null或者undefined，则不更新当前的state，否则合并新旧state
  const memoizedState =
    partialState == null ? prevState : assign({}, prevState, partialState);
  workInProgress.memoizedState = memoizedState;
  // 如果当前没有任务需要更新，则直接将state作为基状态
  // TODD 在beginWork阶段，会把workInProgress.lanes赋值为noLanes，下面这个判断一直是true？？？？
  if (workInProgress.lanes === NoLanes) {
    const updateQueue: UpdateQueue<any> = workInProgress.updateQueue;
    updateQueue.baseState = memoizedState;
  }
}

function callComponentWillMount(workInProgress: Fiber, instance: any) {
  const oldState = instance.state;
  instance.componentWillMount();

  /**
   * 在componentWillMount可能会直接修改state，即this.state = xxx，实际上应该用setState
   * 但是react也会帮你进行state的替换
   */
  if (oldState !== instance.state) {
    instance.updater.enqueueReplaceState(instance, instance.state, null);
  }
}
````

更新阶段：

- 判断是否有用新的生命周期函数，如果没有且新旧 props 不同，会调用`callComponentWillReceiveProps`执行`componentWillReceiveProps`生命周期方法。
  在`componentWillReceiveProps`中也有可能会修改 state 状态，同理`componentWillMount`需要进行更新处理。
- 执行`getDerivedStateFromProps`生命周期函数。
- 调用`checkShouldComponentUpdate`判断组件是否需要更新，如果组件定义了`shouldComponentUpdate`则会调用这个方法获取返回值作为是否需要更新的标记，如果没有则返回 true，表示每次都要更新。
- 如果组件需要更新且没有使用新的生命周期函数，则会尝试执行组件上的`componentWillUpdate`生命周期。
- 如果组件定义了`componentWillUpdate`，则会给 fiber 标记上 Update，同`componentDidMount`。
- 如果组件定义了`getSnapshotBeforeUpdate`，则会给 fiber 标记上 Snapshot。
- 如果组件不需要更新，则会根据新旧 props 和 state 是否相同来确定是否需要处理`componentWillUpdate`和`getSnapshotBeforeUpdate`。

```typescript
// packages/react-reconciler/src/ReactFiberClassComponent.ts
/**
 * @description: 更新class组件实例
 */
export function updateClassInstance(
  current: Fiber,
  workInProgress: Fiber,
  ctor: any,
  newProps: any,
  renderLanes: Lanes
): boolean {
  const instance = workInProgress.stateNode;
  cloneUpdateQueue(current, workInProgress);
  const oldProps = workInProgress.memoizedProps;
  instance.props = oldProps;

  const getDerivedStateFromProps = ctor.getDerivedStateFromProps;

  // 判断是否有新的生命周期函数getDerivedStateFromProps、getSnapshotBeforeUpdate
  const hasNewLifecycles =
    isFunction(getDerivedStateFromProps) ||
    isFunction(instance.getSnapshotBeforeUpdate);

  // 在下面这些生命周期中，只有在componentDidUpdate中的实例上的props和state才是新的，其他都是老的

  // 当用了新的生命周期函数时，不应该再调用不安全的生命周期函数
  if (!hasNewLifecycles && isFunction(instance.componentWillReceiveProps)) {
    if (oldProps !== newProps) {
      callComponentWillReceiveProps(workInProgress, instance, newProps);
    }
  }

  const oldState = workInProgress.memoizedState;
  let newState = (instance.state = oldState);
  processUpdateQueue(workInProgress, newProps, instance, renderLanes);
  // 获取结果processUpdateQueue处理后的新的memoizedState
  newState = workInProgress.memoizedState;
  if (isFunction(getDerivedStateFromProps)) {
    applyDerivedStateFromProps(
      workInProgress,
      ctor,
      getDerivedStateFromProps,
      newProps
    );
    newState = workInProgress.memoizedState;
  }

  // 是否需要更新
  const shouldUpdate = checkShouldComponentUpdate(
    workInProgress,
    ctor,
    oldProps,
    newProps,
    oldState,
    newState
  );

  // 需要更新组件
  if (shouldUpdate) {
    if (!hasNewLifecycles && isFunction(instance.componentWillUpdate)) {
      instance.componentWillUpdate(newProps, newState);
    }
    if (isFunction(instance.componentDidUpdate)) {
      workInProgress.flags |= Update;
    }
    if (isFunction(instance.getSnapshotBeforeUpdate)) {
      workInProgress.flags |= Snapshot;
    }
  } else {
    // componentDidUpdate存在的并且新旧props或state不相等的情况下，标记上Update flag
    if (isFunction(instance.componentDidUpdate)) {
      if (
        oldProps !== current.memoizedProps ||
        oldState !== current.memoizedState
      ) {
        workInProgress.flags |= Update;
      }
    }

    // getSnapshotBeforeUpdate存在的并且新旧props或state不相等的情况下，标记上Snapshot flag
    if (isFunction(instance.getSnapshotBeforeUpdate)) {
      if (
        oldProps !== current.memoizedProps ||
        oldState !== current.memoizedState
      ) {
        workInProgress.flags |= Snapshot;
      }
    }

    // 即使shouldUpdate是false，这里也要更新memoizedProps和memoizedState，表示可以复用
    workInProgress.memoizedProps = newProps;
    workInProgress.memoizedState = newState;
  }

  // 更新实例上的状态
  instance.props = newProps;
  instance.state = newState;

  return shouldUpdate;
}

function callComponentWillReceiveProps(workInProgress, instance, newProps) {
  const oldState = instance.state;
  instance.componentWillReceiveProps(newProps);

  // callComponentWillReceiveProps可能会同步修改state，即this.state = xxx，实际上应该用setState
  if (instance.state !== oldState) {
    instance.updater.enqueueReplaceState(instance, instance.state, null);
  }
}

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

下面看一下`componentDidMount`、`componentDidUpdate`以及`getSnapshotBeforeUpdate`生命周期执行时机：

```typescript
// packages/react-reconciler/src/ReactFiberCommitWork.ts
function commitBeforeMutationEffectsOnFiber(finishedWork: Fiber) {
  const current = finishedWork.alternate;
  const flags = finishedWork.flags;
  if ((flags & Snapshot) !== NoFlags) {
    switch (finishedWork.tag) {
      case ClassComponent: {
        // 更新阶段执行的操作
        if (current !== null) {
          const prevProps = current.memoizedProps;
          const prevState = current.memoizedState;
          const instance = finishedWork.stateNode;
          const snapshot = instance.getSnapshotBeforeUpdate(
            prevProps,
            prevState
          );
          // 将getSnapshotBeforeUpdate的返回值挂在到__reactInternalSnapshotBeforeUpdate属性上，将来作为componentDidUpdate的第三个参数
          instance.__reactInternalSnapshotBeforeUpdate = snapshot;
        }
        break;
      }
    }
  }
}
```

对于`getSnapshotBeforeUpdate`，它是在组件渲染到页面之前执行的，因此它执行的时机就是  beforeMutation 阶段。`getSnapshotBeforeUpdate`的返回值会作为`componentDidUpdate`的第三个参数。

```typescript
// packages/react-reconciler/src/ReactFiberCommitWork.ts
function commitLayoutEffectOnFiber(
  finishedRoot: FiberRoot,
  current: Fiber | null,
  finishedWork: Fiber
) {
  if ((finishedWork.flags & LayoutMask) !== NoFlags) {
    switch (finishedWork.tag) {
      // 省略部分代码
      case ClassComponent: {
        // 获取组件实例
        const instance = finishedWork.stateNode;
        if (finishedWork.flags & Update) {
          if (current === null) {
            // mount阶段执行componentDidMount
            instance.componentDidMount();
          } else {
            const prevProps = current.memoizedProps;
            const prevState = current.memoizedState;
            // 更新阶段，执行componentDidUpdate
            instance.componentDidUpdate(
              prevProps,
              prevState,
              instance.__reactInternalSnapshotBeforeUpdate
            );
          }
        }
        // 省略部分代码
        break;
      }
    }
  }
}
```

`componentDidMount`、`componentDidUpdate`则是在组件渲染完成后执行，它的执行时机和之前讲得`useEffectLayout`一样在 layout 阶段。`commitLayoutEffectOnFiber`这个方法之前介绍过，当时省略了类组件的处理逻辑，这块逻辑就是处理`componentDidMount`和`componentDidUpdate`。



### 三、setState

setState是类组件中用于更新组件状态的方法。它接受一个对象或一个函数作为参数，用于指定状态的更新。 

setState有多种用法，首先是第一种：

```jsx
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }
  handleClick() {
    this.setState({ count: this.state.count + 1 });
  }
  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={() => this.handleClick()}>Increment</button>
      </div>
    );
  }
}
```

当点击按钮时会触发`handleClick`方法，里面的`setState`传入一个对象，用来更新 state 中的 count 值。需要注意的是，setState是一个异步方法，因此不能立即获取更新后的状态值。如果需要在更新状态后执行某些操作，可以在setState方法的第二个参数中传入一个回调函数。

```jsx
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }
  handleClick() {
    this.setState({ count: this.state.count + 1 }, ()=>{
				console.log(this.state.count); // 1
		});
    console.log(this.state.count); // 0
  }
  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={() => this.handleClick()}>Increment</button>
      </div>
    );
  }
}
```

第二种：

```jsx
this.setState((prevState, props) => {
  // 根据prevState和props计算并返回新的状态对象
  return newState;
});
```



`setState`的原理：

```typescript
// packages/react-reconciler/src/ReactFiberClassComponent.ts
const classComponentUpdater = {
  enqueueSetState(instance, payload, callback) {
    // 根据实例获取对应的fiber
    const fiber = getInstance(instance);
    const eventTime = requestEventTime();
    const lane = requestUpdateLane(fiber);
    // 创建一个update
    const update = createUpdate(eventTime, lane);
    // 赋值参数
    update.payload = payload;
    // 绑定callback
    if (callback != null) {
      update.callback = callback;
    }
    // update入队
    enqueueUpdate(fiber, update);
    // 开始调度更新
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  },
  enqueueReplaceState(instance, payload, callback) {
    const fiber = getInstance(instance);
    const lane = requestUpdateLane(fiber);
    const eventTime = requestEventTime();

    const update = createUpdate(eventTime, lane);
    update.payload = payload;
    // 标记替换为ReplaceState
    update.tag = ReplaceState;

    if (callback != null) {
      update.callback = callback;
    }

    enqueueUpdate(fiber, update);
    scheduleUpdateOnFiber(fiber, lane, eventTime);
  },
};
```

`classComponentUpdater`是类组件的更新器对象，这个对象绑定在了`React.Component`上。它有两个方法`enqueueSetState`和`enqueueReplaceState`，分别对应`setState`和`replaceState`，两个方法的逻辑大致相同，只是`replaceState`多了一个 tag 为 ReplaceState 的标记。

在调度更新的逻辑上跟之前讲得更新流程差不多，获取调度优先级，创建更新对象 update 并入队，最后调度更新`scheduleUpdateOnFiber`。我们主要看的是调度更新时对应的 fiber 会如何处理：

```typescript
export function updateClassInstance(
  current: Fiber,
  workInProgress: Fiber,
  ctor: any,
  newProps: any,
  renderLanes: Lanes
): boolean {
  // 省略其他代码
  processUpdateQueue(workInProgress, newProps, instance, renderLanes);
  // 省略其他代码
}

export function processUpdateQueue<State>(
  workInProgress: Fiber,
  props: any,
  instance: any,
  renderLanes: Lanes
) {
  const queue: UpdateQueue<State> = workInProgress.updateQueue;

  let firstBaseUpdate = queue.firstBaseUpdate;
  let lastBaseUpdate = queue.lastBaseUpdate;

  // pending始终指向的是最后一个添加进来的Update
  let pendingQueue = queue.shared.pending;

  // 检测shared.pending是否存在进行中的update将他们转移到baseQueue
  if (pendingQueue !== null) {
    queue.shared.pending = null;
    const lastPendingUpdate = pendingQueue;
    // 获取第一个Update
    const firstPendingUpdate = lastPendingUpdate.next;
    // pendingQueye队列是循环的。断开第一个和最后一个之间的指针，使其是非循环的
    lastPendingUpdate.next = null;
    // 将shared.pending上的update接到baseUpdate链表上
    if (lastBaseUpdate === null) {
      firstBaseUpdate = firstPendingUpdate;
    } else {
      firstBaseUpdate = lastBaseUpdate.next;
    }
    lastBaseUpdate = lastPendingUpdate;
    const current = workInProgress.alternate;

    // 如果current也存在，需要将current也进行同样的处理，同fiber双缓存相似

    // Fiber节点最多同时存在两个updateQueue：
    // current fiber保存的updateQueue即current updateQueue
    // workInProgress fiber保存的updateQueue即workInProgress updateQueue
    // 在commit阶段完成页面渲染后，workInProgress Fiber树变为current Fiber树，workInProgress Fiber树内Fiber节点的updateQueue就变成current updateQueue。
    if (current !== null) {
      const currentQueue = current.updateQueue;
      const currentLastBaseUpdate = currentQueue.lastBaseUpdate;

      // 如果current的updateQueue和workInProgress的updateQueue不同，则对current也进行同样的处理，用于结构共享
      if (currentLastBaseUpdate !== lastBaseUpdate) {
        if (currentLastBaseUpdate === null) {
          currentQueue.firstBaseUpdate = firstPendingUpdate;
        } else {
          currentLastBaseUpdate.next = firstPendingUpdate;
        }
        currentQueue.lastBaseUpdate = lastPendingUpdate;
      }
    }
  }

  if (firstBaseUpdate !== null) {
    let newLanes = NoLanes;

    let newState = queue.baseState;
    let newBaseState: State | null = null;

    let newLastBaseUpdate = null;
    let newFirstBaseUpdate = null;

    let update: Update<State> | null = firstBaseUpdate;
    // TODO 优先级调度
    do {
      newState = getStateFromUpdate(
        workInProgress,
        queue,
        update,
        newState,
        props,
        instance
      );
      // 存在callback，则执行callback回调，比如setState第二个参数
      const callback = update.callback;
      // lane要存在，如果已经提交了，那不应该再触发回调
      if (update.lane === NoLane) {
        console.warn("update.lane === NoLane");
      }
      if (callback && update.lane !== NoLane) {
        // 标记上Callback
        workInProgress.flags |= Callback;
        const effects = queue.effects;
        // 将callback不为null的effect添加到effects中，将来统一执行副作用(update.callback)
        effects == null ? (queue.effects = [update]) : effects.push(update);
      }
      // 可能当所有的update都处理完的时候，payload的执行又产生的新的update被添加到了updateQueue.shared.pending
      // 这个时候还需要继续执行新的更新
      update = update.next;
      if (update === null) {
        pendingQueue = queue.shared.pending;
        if (pendingQueue === null) {
          break;
        } else {
          // 产生了新的更新，进行上述同样的链表操作
          const lastPendingUpdate = pendingQueue;
          const firstPendingUpdate = lastPendingUpdate.next!;
          lastPendingUpdate.next = null;
          update = firstPendingUpdate;
          queue.lastBaseUpdate = lastPendingUpdate;
          queue.shared.pending = null;
        }
      }
    } while (true);

    if (newLastBaseUpdate === null) {
      newBaseState = newState;
    }
    queue.baseState = newBaseState!;
    queue.firstBaseUpdate = newFirstBaseUpdate;
    queue.lastBaseUpdate = newLastBaseUpdate;
    workInProgress.memoizedState = newState;
    workInProgress.lanes = newLanes;
  }
}
```

上面分析类组件更新流程时我们省略了状态更新的逻辑，其实`processUpdateQueue`在组件初始化流程中已经简单介绍过，并且它的逻辑和之前讲的 React Hooks 中的`setState`逻辑很相似。所以，部分逻辑这里就不再赘述，我们主要看`do while`这个循环做的事情。

`do while`会循环 update 对象并进行 state 计算。首先是调用`getStateFromUpdate`获取 state：

```typescript
/**
 * @description: 从update中计算新的state
 */
function getStateFromUpdate<State>(
  workInProgress: Fiber,
  queue: UpdateQueue<State>,
  update: Update<State>,
  prevState: State,
  nextProps: any,
  instance: any
) {
  switch (update.tag) {
    case UpdateState: {
      const payload = update.payload;
      const partialState = isFunction(payload)
        ? payload.call(instance, prevState, nextProps)
        : payload;
      if (partialState == null) {
        // 不需要更新
        return prevState;
      }
      return assign({}, prevState, partialState);
    }
    case ReplaceState: {
      const payload = update.payload;
      return isFunction(payload)
        ? payload.call(instance, prevState, nextProps)
        : payload;
    }
  }
}
```

在`getStateFromUpdate`中会区分 tag，对于正常 UpdateState 会采用合并 state 的方式去更新 state，而对于 ReplaceState 则直接采用新传入的 state。

当 state 计算完毕会后会判断`setState`是否传入了更新后的回调，如果有则会在 fiber 上添加 Callback 这个副作用标记，并将回调放入 queue.effects 中。在 commit 阶段中的 layout 阶段会执行这些回调：

```typescript
// packages/react-reconciler/src/ReactUpdateQueue.ts
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
        // 获取组件实例
        const instance = finishedWork.stateNode;
        if (finishedWork.flags & Update) {
          if (current === null) {
            // mount阶段执行componentDidMount
            instance.componentDidMount();
          } else {
            const prevProps = current.memoizedProps;
            const prevState = current.memoizedState;
            // 更新阶段，执行componentDidUpdate
            instance.componentDidUpdate(
              prevProps,
              prevState,
              instance.__reactInternalSnapshotBeforeUpdate
            );
          }
        }
        const updateQueue = finishedWork.updateQueue;
        if (updateQueue !== null) {
          commitUpdateQueue(finishedWork, updateQueue, instance);
        }
        break;
      }
    }
  }
}

/**
 * @description: 执行updateQueue上的副作用（update的callback）
 */
export function commitUpdateQueue<State>(
  finishedWork: Fiber,
  finishedQueue: UpdateQueue<State>,
  instance: any
) {
  const effects = finishedQueue.effects;
  finishedQueue.effects = null;
  if (effects) {
    for (let i = 0; i < effects.length; i++) {
      const effect = effects[i];
      const callback = effect.callback;
      if (callback) {
        effect.callback = null;
        isFunction(callback) && callback.call(instance);
      }
    }
  }
}
```

这也是`setState`的第二个回调里面可以拿到最新的状态的原理。