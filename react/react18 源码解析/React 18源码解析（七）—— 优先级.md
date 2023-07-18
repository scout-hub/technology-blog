# React 18源码解析（七）—— 优先级



### 该部分解析基于我们实现的简单版react18中的代码，是react18源码的阉割版，希望用最简洁的代码来了解react的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



### 一、概念

在前面几个章节中，我们时常提到优先级这个词，但是并没有深入了解。优先级就是字面意思，它是一种约定，优先级高的先做，优先级低的后做。平常开发中肯定也有接触过类似概念的东西，最常见的如网络请求。如下图，Priority 这一列显示的就是资源的优先级，高优先级的资源会优先加载。

![image-20230712172449310](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712172449310.png)

优先级的作用就是定义哪些事情需要优先处理，哪些事情可以延后处理，及时响应用户的需要。在`react`中就有一套优先级机制，这套机制就是 lane 模型，我们在前面讲的任务调度就是基于 lane 模型的优先级调度。



### 二、lane 模型

在`react`中设计一套优先级机制需要考虑以下几点：

- 能够区分不同优先级
- 可能存在同一个优先级的多个任务调度，需要符合`批`的概念
- 方便进行优先级计算

基于以上几个要点，`react`设计了 lane 模型。



1. 区分优先级

`lane`的中文含义是车道，在现实赛车比赛中有很多赛车道，外圈赛道总长会比内圈赛道更长，某几个相邻的赛道总长相差无几。`react` 借鉴了这个概念，使用31位二进制表示不同的赛道，1 表示可以用，0 表示不开用，位数越小优先级越高。例如，同步优先级会标识为`0b0000000000000000000000000000001`，它占据了 31 位中的第一位。

```typescript
// react-reconciler/src/ReactFiberLane.ts
// lane使用31位二进制来表示优先级车道共31条, 位数越小（1的位置越靠右）表示优先级越高
export const TotalLanes = 31;
export const NoTimestamp = -1;
// 无优先级，初始化的优先级
export const NoLanes: Lanes = 0b0000000000000000000000000000000;
export const NoLane: Lane = 0b0000000000000000000000000000000;
// 同步更新的优先级为最高优先级
export const SyncLane = 0b0000000000000000000000000000001;
export const InputContinuousLane: Lane = 0b0000000000000000000000000000100;
// 默认优先级，例如使用setTimeout，请求数据返回等造成的更新
export const DefaultLane: Lane = 0b0000000000000000000000000010000;
export const IdleLane: Lane = 0b010000000000000000000000000000;
const TransitionLanes: Lanes = 0b0000000011111111111111110000000;
// 所有未闲置的1的位置，通过 & NonIdleLanes就能知道是否有未闲置的任务 1 & 1 ==> 1 1 & 0 ==> 0
const NonIdleLanes: Lanes = 0b0001111111111111111111111111111;
```

2. `批`的概念

上面这些优先级中有一个优先级占据了多个 1—— TransitionLanes，这就是`批`的概念。这种被成为`lanes`，不同与 lane，它占据了一定的范围。



3. 方便计算

对于二进制的数据的计算无非就是位运算。

```typescript
// react-reconciler/src/ReactFiberLane.ts
// 合并运算
export function mergeLanes(a: Lanes | Lane, b: Lanes | Lane): Lanes {
  return a | b;
}

// 判断交集
export function includesSomeLane(a: Lanes | Lane, b: Lanes | Lane) {
  return (a & b) !== NoLanes;
}

// 判断子集
export function isSubsetOfLanes(set, subset) {
  return (set & subset) === subset;
}

// 移除运算
export function removeLanes(set: Lanes, subset: Lanes | Lane): Lanes {
  return set & ~subset;
}
```

我们在前面讲的更新调度并没有涉及到优先级的概念，实际上`react`中的调度是一套基于优先级的调度，每一个调度任务都有属于它的优先级。



### 三、基于优先级的调度

我们回头再从初始化渲染开始讲解基于优先级的调度逻辑，这次会把优先级相关代码全都放出来。首先是`updateContainer`，在这个方法中我们之前省略了`requestUpdateLane`的讲解：

1. 调用`getCurrentUpdatePriority`获取当前更新的优先级`updateLane`，初始化下默认获取到的是`NoLane`。
2. 如果`updateLane`不是`NoLane`直接返回，否则调用`getCurrentEventPriority`获取优先级。
3. 在`getCurrentEventPriority`中会调用`getEventPriority`获取事件优先级。在`getEventPriority`中定义所有页面事件对应的优先级，如果没有对应的事件优先级，则默认返回默认优先级`DefaultEventPriority = 0b0000000000000000000000000010000`。在这里我们只添加了`click`和`mousedown`的事件优先级。

初始化渲染时的`lane`就是`DefaultEventPriority`。

```typescript
// react-reconciler/src/ReactFiberReconciler.ts
export function updateContainer(element, container) {
  const current = container.current;
  // 计算事件的开始事件
  const eventTime = requestEventTime();
  const lane = requestUpdateLane(current);
  // 创建更新，目前只有hostRoot使用（hostRoot和classComponent共用同一种update结构，和function component不同）
  const update = createUpdate(eventTime, lane);
  // hostRoot的payload对应为需要挂载在根节点的组件
  update.payload = { element };
  // 存储更新，添加到更新队列中
  enqueueUpdate(current, update);
  // 调度该fiber节点的更新
  const root = scheduleUpdateOnFiber(current, lane, eventTime);
  if (root !== null) {
    // TODO 处理非紧急更新 transition
  }
}

// react-reconciler/src/ReactFiberWorkLoop.ts
export function requestUpdateLane(fiber: Fiber): Lane {
  const updateLane = getCurrentUpdatePriority();
  if (updateLane !== NoLane) {
    return updateLane;
  }
  // 大部分事件更新产生的优先级
  const eventLane = getCurrentEventPriority();
  return eventLane;
}

// react-dom/src/client/ReactDOMHostConfig.ts
export function getCurrentEventPriority() {
  const currentEvent = window.event;
  if (currentEvent === void 0) {
    return DefaultEventPriority;
  }
  return getEventPriority(currentEvent.type as DOMEventName);
}

// react-dom/src/events/ReactDOMEventListener.ts
export function getEventPriority(domEventName: DOMEventName) {
  switch (domEventName) {
    case "click":
    case "mousedown":
      // 同步优先级，最高
      return DiscreteEventPriority;
    default:
      return DefaultEventPriority;
  }
}
```

当获取到调度的优先级后会存入`update`更新对象中，随后开始调度`scheduleUpdateOnFiber`：

1. 先调用`markUpdateLaneFromFiberToRoot`向上查找当前应用的根节点，该过程会更新当前被调度的`fiber`上的`lanes`以及每一个祖先`fiber`的`childLanes`。因为有新的`lane`进来了，这个`lane`需要合并上去。
2. 调用`markRootUpdated`给`root`节点打上更新标记`pendingLanes`。在之前介绍`fiberRoot`结构时候讲过，这个属性表示所有将要执行的任务的`lane`。随后在`eventTimes`上标记当前优先级事件对应的开始时间。这个`eventTimes`也是`fiberRoot`上的一个属性，它也有31 位，对应 31 位的`lane`，主要用来记录任务的开始时间。
3. 调用`ensureRootIsScheduled`调用应用。

```typescript
// react-reconciler/src/ReactFiberWorkLoop.ts
export function scheduleUpdateOnFiber(fiber, lane: Lane, eventTime: number) {
  /**
   * react在render阶段从当前应用的根节点开始进行树的深度优先遍历处理，
   * 在更新的时候，当前处理的fiber节点可能不是当前应用的根节点，因此需要通过
   * markUpdateLaneFromFiberToRoot向上去查找当前应用的根节点，同时对查找路径上的
   * fiber进行lane的更新
   */
  const root = markUpdateLaneFromFiberToRoot(fiber, lane);
  if (root === null) {
    return null;
  }
  // 给root节点加上更新标记，pendingLanes
  markRootUpdated(root, lane, eventTime);
  // 调度应用
  ensureRootIsScheduled(root, eventTime);
  return root;
}

function markUpdateLaneFromFiberToRoot(
  sourceFiber: Fiber,
  lane: Lane
): FiberRoot | null {
  // 更新当前fiber节点的lannes，表示当前节点需要更新
  sourceFiber.lanes = mergeLanes(sourceFiber.lanes, lane);
  let alternate = sourceFiber.alternate;

  // alternate存在表示更新阶段，需要同步更新alternate上的lanes
  if (alternate !== null) {
    alternate.lanes = mergeLanes(alternate.lanes, lane);
  }

  // 从当前需要更新的fiber节点向上遍历，遍历到根节点（root fiber）并更新每个fiber节点上的childLanes属性
  // 这样就可以通过父节点的childLanes来知道子树所有Update的lanes，不需要遍历子树
  let node = sourceFiber;
  let parent = sourceFiber.return;
  while (parent !== null) {
    // 收集需要更新的子节点的lane，存放在父fiber上的childLanes上，childLanes有值表示当前节点下有子节点需要更新
    parent.childLanes = mergeLanes(parent.childLanes, lane);
    alternate = parent.alternate;
    // 同步更新alternate上的childLanes
    if (alternate !== null) {
      alternate.childLanes = mergeLanes(alternate.childLanes, lane);
    }

    node = parent;
    parent = parent.return;
  }
  if (node.tag === HostRoot) {
    return node.stateNode;
  } else {
    return null;
  }
}

/**
 * @description:  将当前需要更新的lane添加到fiberRoot的pendingLanes属性上，表示有新的更新任务需要被执行
 * 通过计算出当前lane的位置，添加事件触发时间到eventTimes中
 */
export function markRootUpdated(
  root: FiberRoot,
  updateLane: Lane,
  eventTime: number
) {
  // 需要进行但是还没有进行的lanes
  root.pendingLanes |= updateLane;

  // 进入到scheduleUpdateOnFiber是紧急更新，如果存在非紧急更新需要先unblock
  // 如果updateLane是IdleLane的优先级非常低，不需要去管，因为它不会去阻塞其它任务
  // if (updateLane !== IdleLane) {
  //   root.suspendedLanes = NoLanes;
  //   root.pingedLanes = NoLanes;
  // }

  // 一个三十一位的数组，分别对应着31位lane
  const eventTimes = root.eventTimes;
  const index = laneToIndex(updateLane);
  eventTimes[index] = eventTime;
}
```

接下去是`ensureRootIsScheduled`：

1. 标记过期任务`markStarvedLanesAsExpired`，这里是为了解决饥饿问题。在`react`中可能存在某些低优先级的任务由于高优先级任务的存在一直不能执行，因此这里会给每一种优先级任务设置一个超时时间，如果某个优先级任务超时了就需要立即执行。
2. 调用`getNextLanes`获取下一个需要更新的任务`lanes`，假如`lanes`存在，则说明有任务需要执行，否则直接结束。
3. 获取上一步`lanes`中优先级最高的`lane`，用这个`lane`来表示后续调度任务的优先级。
4. 如果上一个调度任务的优先级和本次调度任务的优先级一样则直接结束（`existingCallbackPriority === newCallbackPriority`）,这也是同一个`onclick`事件里面多次`setState`只会触发一次更新的原因（后续会分析这个问题）。
5. 判断上一个调度任务是否存在，存在则需要打断这个任务的执行，因为本次调度任务的优先级一定是最高的。
6. 调度一个新的回调任务`newCallbackNode`，这个任务会根据，我们在上面提到初始化渲染时的优先级是默认的优先级`DefaultEventPriority`，因此它会走并发更新的逻辑分支。
7. 通过`lanesToEventPriority`将`lane`优先级转化为`react`事件的优先级，再将事件的优先级转化为`schedular`调度的优先级，最后调用`scheduleCallback`发起任务调度。

```typescript
// react-reconciler/src/ReactFiberWorkLoop.ts
function ensureRootIsScheduled(root: FiberRoot, eventTime: number) {
  const existingCallbackNode = root.callbackNode;

  // 判读某些lane上的任务是否已经过期，过期的话就标记为过期，然后接下去就可以用同步的方式执行它们（解决饥饿问题）
  markStarvedLanesAsExpired(root, eventTime);

  // 获取下一个需要更新的任务的优先级（有没有任务需要调度）
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes
  );

  // 如果nextLanes为空则表示没有任务需要执行，直接结束调度
  if (nextLanes === NoLanes) {
    // existingCallbackNode不为空表示有任务使用了concurrent模式被scheduler调用，但是还未执行
    // nextLanes为空了则表示没有任务了，就算这个任务执行了但是也做不了任何更新，所以需要取消掉
    if (existingCallbackNode !== null) {
      // 使用cancelCallback会将任务的callback置为null
      // 在scheduler循环taskQueue时，会检查当前task的callback是否为null
      // 为null则从taskQueue中删除，不会执行
      cancelCallback(existingCallbackNode);
    }
    root.callbackNode = null;
    root.callbackPriority = NoLane;
    return;
  }

  // 使用优先级最高的任务的优先级来表示回调函数的优先级
  const newCallbackPriority = getHighestPriorityLane(nextLanes);
  const existingCallbackPriority = root.callbackPriority;

  /**
   * 与现有的任务优先级一样的情况，直接返回
   * 这就是一个onclick事件中多次setState只会触发一次更新的原因，
   * 同一优先级的Update只会被调度一次（复用），而所有产生的Update已经通过链表的形式存储在queue.pending中，
   * 这些Update在后续调度过程中一起调度即可
   */
  if (existingCallbackPriority === newCallbackPriority) {
    return;
  }

  // 走到这儿说明新任务的优先级大于现有任务的优先级，如果存在现有任务则取消现有的任务的执行（高优先级打断低优先级）
  if (existingCallbackNode != null) {
    cancelCallback(existingCallbackNode);
  }

  // 调度一个新的回调
  let newCallbackNode;
  if (newCallbackPriority === SyncLane) {
    // 同步任务的更新
    scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root));
    // 注册一个微任务
    scheduleMicrotask(flushSyncCallbacks);
    newCallbackNode = null;
  } else {
    // 设置任务优先级
    let schedulerPriorityLevel;
    // 计算任务超时等级
    // lanesToEventPriority函数将lane的优先级转换为React事件的优先级，然后再根据React事件的优先级转换为Scheduler的优先级
    switch (lanesToEventPriority(nextLanes)) {
      case DiscreteEventPriority:
        schedulerPriorityLevel = ImmediateSchedulerPriority;
        break;
      case ContinuousEventPriority:
        schedulerPriorityLevel = UserBlockingSchedulerPriority;
        break;
      case DefaultEventPriority:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
      case IdleEventPriority:
        schedulerPriorityLevel = IdleSchedulerPriority;
        break;
      default:
        schedulerPriorityLevel = NormalSchedulerPriority;
        break;
    }
    // 非同步任务通过scheduler去调度任务
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      // 绑定上当前的root
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }

  root.callbackNode = newCallbackNode;
  root.callbackPriority = newCallbackPriority;
}
```

上述流程中涉及到几个方法的介绍，首先是`markStarvedLanesAsExpired`，这个方法用来标记任务过期时间：

1. 循环`lanes`，通过`pickArbitraryLaneIndex`获取二进制中最左边 1 的位置，比如二进制是`0b11`，那么最左边 1 的位置就是 1。
2. 根据位置通过位运算得到优先级，上一步中 1<< 1 就是 2。
3. 获取当前位置上任务的过期时间，假如没有过期时间则调用`computeExpirationTime`初始化一个过期时间，这个过期时间会根据优先级来设置。
4. 假如任务已经过期`expirationTime <= currentTime`，则需要将优先级添加到`root.expiredLanes`中。
5. 移除已经处理过的`lane`，继续下一次循环。

```typescript
// react-reconciler/src/ReactFiberLane.ts
export function markStarvedLanesAsExpired(
  root: FiberRoot,
  currentTime: number
) {
  const pendingLanes = root.pendingLanes;
  const expirationTimes = root.expirationTimes;

  let lanes = pendingLanes;
  while (lanes > 0) {
    // 获取当前31位lanes中最左边1的位置
    const index = pickArbitraryLaneIndex(lanes);
    // 根据位置计算lane值
    const lane = 1 << index;
    // 获取该位置上任务的过期时间
    const expirationTime = expirationTimes[index];
    // 如果没有过期时间，则创建一个过期时间
    if (expirationTime === NoTimestamp) {
      expirationTimes[index] = computeExpirationTime(lane, currentTime);
    } else if (expirationTime <= currentTime) {
      // 如果任务已经过期，将当前的lane添加到expiredLanes中
      root.expiredLanes |= lane;
    }
    // 从lanes中删除lane, 每次循环删除一个，直到lanes等于0
    lanes &= ~lane;
  }
}

function pickArbitraryLaneIndex(lanes: Lanes) {
  return 31 - Math.clz32(lanes);
}

function computeExpirationTime(lane: Lane, currentTime: number) {
  switch (lane) {
    case SyncLane:
      return currentTime + 250;
    case DefaultLane:
      return currentTime + 5000;
    default: {
      return NoTimestamp;
    }
  }
}
```

`getNextLanes`：获取优先级最高的任务，这个任务可能是批任务，所以是`lanes`：

1. 获取所有将要执行的任务`root.pendingLanes`，如果为空则说明任务都执行完了，直接返回。
2. 通过位运算`&`判断是否有未闲置的任务，如果有则需要获取未闲置任务中优先级最高的任务。
3. 判断是否有正在执行的任务，假如有且优先级和新任务优先级不同则返回两者中优先级最高的任务。

```typescript
// react-reconciler/src/ReactFiberLane.ts
/**
 * @description: 获取当前任务的优先级
 *  wipLanes是正在执行任务的lane，nextLanes是本次需要执行的任务的lane
 *
 */
export function getNextLanes(root: FiberRoot, wipLanes: Lanes): Lanes {
  // 每一个任务都会将它们各自的优先级添加到fiberRoot的pendingLanes的属性上，这里获取了所有将要执行的任务的lane
  const pendingLanes = root.pendingLanes;

  // pendingLanes为空，说明所有的任务都执行完了
  if (pendingLanes === NoLanes) {
    return NoLanes;
  }

  let nextLanes = NoLanes;

  // 是否有非空闲的任务，如果有非空闲的任务，需要先执行非空闲的任务，不要去执行空闲的任务
  const nonIdlePendingLanes = pendingLanes & NonIdleLanes;

  // 有未闲置的任务
  if (nonIdlePendingLanes !== NoLanes) {
    // 获取除挂起任务外最高优先级任务，这里暂时不考虑挂起任务
    nextLanes = getHighestPriorityLanes(nonIdlePendingLanes);
    // TODO 从挂起任务中获取最高优先级任务的优先级
  } else {
    // TODO
  }

  if (nextLanes === NoLanes) {
    return NoLanes;
  }

  // 说明在渲染阶段插入了一个新的任务 ???
  if (wipLanes !== NoLanes && wipLanes !== nextLanes) {
    const nextLane = getHighestPriorityLane(nextLanes);
    const wipLane = getHighestPriorityLane(wipLanes);

    // 如果新添加任务优先级低，则依旧返回当前渲染中的任务的优先级
    if (nextLane >= wipLane) {
      return wipLanes;
    }
  }

  return nextLanes;
}

// 所有未闲置的1的位置，通过 & NonIdleLanes就能知道是否有未闲置的任务 1 & 1 ==> 1 1 & 0 ==> 0
const NonIdleLanes: Lanes = 0b0001111111111111111111111111111;
```

`getHighestPriorityLanes`：获取`lanes`中优先级最高的任务

```typescript
// react-reconciler/src/ReactFiberLane.ts
/**
 * @description: 从lanes中获取最高优先级的lane
 */
function getHighestPriorityLanes(lanes: Lanes | Lane): Lanes {
  switch (getHighestPriorityLane(lanes)) {
    case SyncLane:
      return SyncLane;
    case DefaultLane:
      return DefaultLane;
    default: {
      return lanes;
    }
  }
}

/**
 * @description: 获得一个二进制数中以最低位1所形成的数
 * 比如 0b101 & -0b101 ===> 0b001 === 1;  0b110 & -0b110 ===> 0b010 === 2
 */
export function getHighestPriorityLane(lanes: Lanes): Lane {
  return lanes & -lanes;
}
```

`lanesToEventPriority`：将`lane`转化为`react`事件优先级。

```typescript
// react-reconciler/src/ReactFiberLane.ts
/**
 * @description: 将lane优先级转化为事件优先级
 */
export function lanesToEventPriority(lanes: Lanes): EventPriority {
  // 找到优先级最高的lane
  const lane = getHighestPriorityLane(lanes);
  // lane的优先级高于DiscreteEventPriority，直接返回DiscreteEventPriority
  if (!isHigherEventPriority(DiscreteEventPriority, lane)) {
    return DiscreteEventPriority;
  }
  if (!isHigherEventPriority(ContinuousEventPriority, lane)) {
    return ContinuousEventPriority;
  }
  // 有lane被占用，但是优先级没有上面的两个高，返回DefaultEventPriority
  if (includesNonIdleWork(lane)) {
    return DefaultEventPriority;
  }
  return IdleEventPriority;
}

```

我们回到`scheduleCallback`，继续接下去的流程：

1. 根据优先级计算超时时间，超时时间越小说明优先级越高。
2. 计算任务过期时间。
3. 创建一个新的任务。
4. 将任务添加到优先队列`taskQueue`中。
5. 发起并发调度。

```typescript
// scheduler/src/forks/Scheduler.ts
const IMMEDIATE_PRIORITY_TIMEOUT = -1; // 需要立即执行
const USER_BLOCKING_PRIORITY_TIMEOUT = 250; // 250ms 超时时间250ms，一般指的是用户交互
const NORMAL_PRIORITY_TIMEOUT = 5000; // 5000ms 超时时间5s，不需要直观立即变化的任务，比如网络请求
const LOW_PRIORITY_TIMEOUT = 10000; // 10000ms 超时时间10s，肯定要执行的任务，但是可以放在最后处理
const IDLE_PRIORITY_TIMEOUT = 1073741823; // 一些没有必要的任务，可能不会执行

/**
 * @description: 调度任务 高优先级任务插队
 * @param priorityLevel 优先级
 * @param callback 需要调度的回调
 */
function unstable_scheduleCallback(priorityLevel, callback) {
  // 获取任务当前时间
  const currentTime = getCurrentTime();
  const startTime = currentTime;

  // 1、根据优先级计算超时时间，超时时间越小说明优先级越高
  let timeout;
  switch (priorityLevel) {
    case ImmediatePriority:
      timeout = IMMEDIATE_PRIORITY_TIMEOUT;
      break;
    case UserBlockingPriority:
      timeout = USER_BLOCKING_PRIORITY_TIMEOUT;
      break;
    case IdlePriority:
      timeout = IDLE_PRIORITY_TIMEOUT;
      break;
    case LowPriority:
      timeout = LOW_PRIORITY_TIMEOUT;
      break;
    case NormalPriority:
      timeout = NORMAL_PRIORITY_TIMEOUT;
      break;
    default:
      timeout = NORMAL_PRIORITY_TIMEOUT;
  }

  // 2、计算过期时间
  const expirationTime = startTime + timeout;
  // 3、创建一个新任务
  const newTask = {
    id: taskIdCounter++,
    callback,
    priorityLevel,
    startTime,
    expirationTime,
    sortIndex: -1, // 任务排序序号，初始化-1
  };

  // TODO 如果任务开始时间大于当前时间，说明任务没有过期，需要放入延时队列timerQueue中
  if (startTime > currentTime) {
    throw Error("startTime > currentTime");
  } else {
    // 任务开始时间<=当前时间，说明任务过期了，需要添加到taskQueue队列中以进行任务调度
    // 过期任务根据过期时间进行排序
    newTask.sortIndex = expirationTime;
    push(taskQueue, newTask);
    // 如果没有处于调度中的任务，并且workLoop没有在执行中，则向浏览器申请时间片（一帧），浏览器空闲的时候执行workLoop
    if (!isHostCallbackScheduled && !isPerformingWork) {
      isHostCallbackScheduled = true;
      requestHostCallback(flushWork);
    }
  }
  return newTask;
}
```

优先队列`taskQueue`，它是一个小顶堆。小顶堆的按照任务的过期时间点来排序，越早过期的任务优先级越高，所以最顶上的任务优先级是最高的。小顶堆中有三个方法：

- `push`：添加一个任务，添加完成后会调用`siftUp`进行上浮操作，上浮到合适的位置。
- `peek`：获取队首任务，也就是优先级最高的任务。
- `pop`：弹出队首任务，弹出后需要调用`siftDown`进行下浮操作，下浮到合适的位置。

```typescript
// scheduler/src/SchedulerMinHeap.ts
type Heap = Array<Node>;
type Node = {
  startTime: number;
  id: number;
  sortIndex: number;
};

// 往队列中添加任务
export function push(heap: Heap, task: Node) {
  const index = heap.length;
  heap.push(task);
  // 队列任务排序（小顶堆）
  siftUp(heap, task, index);
}

// 获取队首任务，即高优先级任务
export function peek(heap: Heap): Node | null {
  return heap.length ? heap[0] : null;
}

// 弹出队首任务
export function pop(heap: Heap): Node | null {
  if (heap.length === 0) {
    return null;
  }
  const first = heap[0];
  const last = heap.pop()!;
  // 存在多个任务，通过堆排序进行下浮
  if (first !== last) {
    // 队尾元素放到队首
    heap[0] = last;
    // 进行堆排序下浮
    siftDown(heap, last, 0);
  }
  // 只有一个任务的情况，直接返回
  return first;
}

// 小顶堆构建，新添加的任务通过堆排序进行上浮（优先级队列）
function siftUp(heap: Heap, task: Node, i: number) {
  let index = i;
  // 进行堆排序
  while (index > 0) {
    // 根据完全二叉树的性质，如果当前节点索引为i，则它的左孩子的索引为2i+1，右孩子的索引为2i+2
    // 如果当前子节点的索引为i，则可推出父节点索引为(i-1)/2
    const parentIndex = (index - 1) >> 1;
    const parentTask = heap[parentIndex];
    // 如果父任务的优先级低于当前任务，则对父子进行调换
    if (compare(parentTask, task) > 0) {
      heap[parentIndex] = task;
      heap[index] = parentTask;
      index = parentIndex;
    } else {
      return;
    }
  }
}

/**
 * @description: 下浮操作，将当前节点下浮到合适的位置
 */
function siftDown(heap: Heap, task: Node, i: number) {
  let index = i;
  let length = heap.length;
  let halfLength = length >> 1;

  // 比较到倒数第二层，因为最后一层以及没有子树了
  while (index < halfLength) {
    // 左子节点的索引
    const leftIndex = 2 * index + 1;
    // 左子节点
    const left = heap[leftIndex];
    // 右子节点的索引
    const rightIndex = leftIndex + 1;
    // 右子节点
    const right = heap[rightIndex];

    // 先比较根和左子节点，如果左比根小（优先级高），再让左跟右比；如果根小，则根和右比
    if (compare(left, task) < 0) {
      // 如果右子节点比左更小，则直接右和根节点交换，否则左子节点和根节点交换
      if (rightIndex < length && compare(right, left) < 0) {
        heap[index] = right;
        heap[rightIndex] = task;
        index = rightIndex;
      } else {
        heap[index] = left;
        heap[leftIndex] = task;
        index = leftIndex;
      }
    } else if (rightIndex < length && compare(right, task) < 0) {
      // 右比根小，则右和根交换位置
      heap[index] = right;
      heap[rightIndex] = task;
      index = rightIndex;
    } else {
      // 结束
      return;
    }
  }
}

// 比较任务的先后，先根据sortIndex排序，如果一样就根据id排序
function compare(a: Node, b: Node) {
  const diff = a.sortIndex - b.sortIndex;
  return diff !== 0 ? diff : a.id - b.id;
}
```

后续的`beginWork`和`completeWork`流程在前几章已经讲述过，这里就不再介绍，最后看一下提交阶段的优先级处理。在提交阶段会之前产生的优先级进行清理，首先是`mergeLanes` 合并剩余可能存在的任务，由于存在高优先级打断低优先级任务的情况，低优先级任务的`lanes`还会保留，需要在下一次继续调度（后续在`hooks`章节介绍）。之后再调用`markRootFinished`进行清理，将已经执行完的优先级都清除。

```typescript
// react-reconciler/src/ReactFiberWorkLoop.ts
function commitRootImpl(root: FiberRoot) {
  const finishedWork = root.finishedWork;

  if (finishedWork === null) return;

  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  // commitRoot总是同步完成的。所以我们现在可以清除这些，以允许一个新的回调被调度。
  root.callbackNode = null;
  root.callbackPriority = NoLane;

  /**
   * 剩余需要调度的lanes为HostRoot的lanes和其子树lanes的并集，childLanes有剩余的情况：
   * 一个子树高优先级任务打断了一个低优先级的任务，低优先级任务的lanes会被保存，并且lanes会
   * 存储到父节点的childLanes属性上，我们可以通过childLanes去获取它。具体逻辑在bubbleProperties中
   */
  let remainingLanes = mergeLanes(finishedWork.lanes, finishedWork.childLanes);
  markRootFinished(root, remainingLanes);

  workInProgressRoot = null;
  workInProgress = null;
  
  // 省略其他代码

  // commit阶段可能会产生新的更新，所以这里再调度一次
  ensureRootIsScheduled(root, now());

  flushSyncCallbacks();
}

// react-reconciler/src/ReactFiberLane.ts
/**
 * @description: 进行本轮更新的收尾工作，将完成工作的lane、eventTime重置，并将它们
 * 从pendingLanes，expiredLanes去除
 */
export function markRootFinished(root: FiberRoot, remainingLanes: Lanes) {
  // 获取不再需要执行的任务
  const noLongerPendingLanes = root.pendingLanes & ~remainingLanes;

  root.pendingLanes = remainingLanes;
  root.expiredLanes &= remainingLanes;

  const eventTimes = root.eventTimes;
  const expirationTimes = root.expirationTimes;

  let lanes = noLongerPendingLanes;

  while (lanes > 0) {
    const index = pickArbitraryLaneIndex(lanes);
    const lane = 1 << index;
    eventTimes[index] = NoTimestamp;
    expirationTimes[index] = NoTimestamp;
    lanes &= ~lane;
  }
}
```