# React 18源码解析（六）——  时间切片（下）



### 该部分解析基于我们实现的简单版react18中的代码，是react18源码的阉割版，希望用最简洁的代码来了解react的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。

上一章节我们介绍了时间切片相关的概念，这一章我们将深入`react`源码来了解其实现过程，其中涉及到优先级会在后续章节中介绍。

### 一、并发调度（concurrent）

在初始化渲染章节，我们曾提到`react`调度的两种模式，同步模式（sync）和并发模式（concurrent）。同步模式在之前的章节中已经大致介绍过，而时间切片属于并发模式里的功能。因此，这里将默认以并发模式的调度方式进行讲解。

经过前几个章节的介绍我们知道，在`react`中所有调度更新都会通过`scheduleUpdateOnFiber`发起，在`scheduleUpdateOnFiber`中会调用`ensureRootIsScheduled`发起真正的应用调度。

```typescript
function ensureRootIsScheduled(root: FiberRoot, eventTime: number) {
  // 省略其它代码
  let newCallbackNode;
  if (newCallbackPriority === SyncLane){
    // 省略同步调度
  } else{
    // 省略优先级计算
    newCallbackNode = scheduleCallback(
      schedulerPriorityLevel,
      performConcurrentWorkOnRoot.bind(null, root)
    );
  }
}
```

在`ensureRootIsScheduled`中，假如是非同步任务，则会通过`scheduler`去调度任务，也就是这个`scheduleCallback`。`scheduleCallback`引用的是 `Schedular`模块下的`unstable_scheduleCallback`方法：

```typescript
// react-reconciler/src/Scheduler.ts
import * as Scheduler from "scheduler";
export const scheduleCallback = Scheduler.unstable_scheduleCallback;

// scheduler/src/forks/Scheduler.ts
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

在`unstable_scheduleCallback`中大部分逻辑都和优先级调度有关，我们暂时不需要了解，真正发起调度的只有`requestHostCallback(flushWork)`。`requestHostCallback`接受一个`callback`参数，这个参数值就是`flushWork`。在`requestHostCallback`中通过`isMessageLoopRunning`变量来防止重复调度，假如`isMessageLoopRunning`是 false，则会调用`schedulePerformWorkUntilDeadline`注册一个宏任务，node 环境下是`setImmediate`，浏览器环境下则是`MessageChannel`。至于为什么选择`MessageChannel`以及其使用方式可以回顾上一章的内容。

```typescript
// scheduler/src/forks/Scheduler.ts
function requestHostCallback(callback) {
  scheduledHostCallback = callback;
  if (!isMessageLoopRunning) {
    isMessageLoopRunning = true;
    schedulePerformWorkUntilDeadline();
  }
}

// 空闲时间进行任务调度逻辑
let schedulePerformWorkUntilDeadline;
// 利用messageChannel模拟实现requestIdleCallback
// 模拟实现requestIdleCallback的两个条件
// 1 模拟实现的requestIdleCallback能够主动让出线程，让浏览器去一些事情，例如渲染
// 2 一次事件循环中只执行一次，因为执行完一次调度任务后还会去申请下一个时间片
// 满足上述条件的只有宏任务，因为宏任务是在下一次事件循环开始的时候执行，并不会阻塞本次更新，并且宏任务在一次事件循环中也只逆行一次。

// node环境下使用setImmediate
// 浏览器和web worker环境下，这里不用setTimeout的原因是递归调用的时候，延迟最小是4ms
if (typeof MessageChannel !== "undefined") {
  const channel = new MessageChannel();
  const port = channel.port2;
  // message回调是宏任务，在下一个事件循环中执行这个回调
  channel.port1.onmessage = performWorkUntilDeadline;
  schedulePerformWorkUntilDeadline = () => {
    port.postMessage(null);
  };
} else {
  schedulePerformWorkUntilDeadline = () => {
    localSetTimeout!(performWorkUntilDeadline, 0);
  };
}
```

当注册完宏任务后，这个宏任务的回调会在下一次事件循环中执行，也就是`performWorkUntilDeadline`，这个就是切片的入口函数。



### 二、时间切片

我们继续看`performWorkUntilDeadline`这个函数。在`performWorkUntilDeadline`中会计算本次切片任务的开始时间`currentTime`，接着调用`scheduledHostCallback`函数，这个函数就是我们上面提到的`flushWork`。

```typescript
// scheduler/src/forks/Scheduler.ts
function performWorkUntilDeadline() {
  // 这个scheduledHostCallback就是flushWork
  if (scheduledHostCallback !== null) {
    const currentTime = getCurrentTime();
    startTime = currentTime;

    // 是否有剩余时间
    const hasTimeRemaining = true;
    let hasMoreWork = true;

    try {
      // 执行flushWork
      hasMoreWork = scheduledHostCallback(hasTimeRemaining, currentTime);
    } finally {
      // 如果队列中还有任务，则继续为其创建一个宏任务以继续执行
      if (hasMoreWork) {
        schedulePerformWorkUntilDeadline();
      } else {
        isMessageLoopRunning = false;
        scheduledHostCallback = null;
      }
    }
  } else {
    isMessageLoopRunning = false;
  }
}
```

在`flushWork`中主要就是调用`workLoop`去执行调度任务。

```typescript
// scheduler/src/forks/Scheduler.ts
function flushWork(hasTimeRemaining: boolean, initialTime: number) {
  isHostCallbackScheduled = false;
  isPerformingWork = true;
  try {
    return workLoop(hasTimeRemaining, initialTime);
  } finally {
    currentTask = null;
    isPerformingWork = false;
  }
}
```

`workLoop`的逻辑很多，我们一一进行梳理：

1. 从`taskQueue`中取当前优先级最高的任务`currentTask`，这个`taskQueue`是个优先队列。
2. 循环处理`currentTask`，假如`currentTask`还没过期，并且当前帧没有空闲时间则直接退出循环，这里会借助`shouldYieldToHost`来判断是否还有空闲时间。在`shouldYieldToHost`逻辑中有涉及到一个全新的浏览器 API `isInputPending`，用于判断是否有输入事件等待处理。
3. 取出`currentTask`中的`callback`执行，这个`callback`其实就是`performConcurrentWorkOnRoot`（见`unstable_scheduleCallback`），执行之前需要把`currentTask`重置为 null。
4. 如果`callback`，也就是`performConcurrentWorkOnRoot`执行后返回了一个函数`continuationCallback`，则说明`currentTask`因为浏览器没有空闲时间而被中断了，需要继续调度，并且`currentTask`的`callback`需要赋值为新的`continuationCallback`； 否则`currentTask`需要从`taskQueue`中弹出。
5. 取出下一个任务继续执行，这个任务可能是之前被中断的任务，因为这个任务没有出队列。这里有个疑问，为什么之前任务被中断后不直接中断循环，反而要将这个任务取出来再执行一次？因为有一种可能，这个任务马上会过期，对于过期的任务`react`会采用同步的方式去执行。过期这个概念会在后续结合优先级一起讲。
6. 循环结束后假如`currentTask`存在，说明任务被中断了，返回`true`的标记。这个标记会在`performWorkUntilDeadline`中被`hasMoreWork`这个变量接受，如果是`true`则会调用`schedulePerformWorkUntilDeadline`继续注册一个新的宏任务；如果`currentTask`不存在，则说明`taskQueue`的任务都执行完，接下去需要查看`timerQueue`中是否有任务需要执行。

```typescript
// scheduler/src/forks/Scheduler.ts
function workLoop(hasTimeRemaining: boolean, initialTime: number) {
  let currentTime = initialTime;
  // 取出当前优先级最高的任务
  currentTask = peek(taskQueue);
  while (currentTask !== null) {
    // 如果任务还没过期（没执行）且当前帧已经执行完了，没有空闲时间了，则退出循环
    if (
      currentTask.expirationTime > currentTime &&
      (!hasTimeRemaining || shouldYieldToHost())
    ) {
      break;
    }

    // 获取真正的更新函数
    const callback = currentTask.callback; // performConcurrentWorkOnRoot
    if (typeof callback === "function") {
      currentTask.callback = null;
      currentPriorityLevel = currentTask.priorityLevel;
      const didUserCallbackTimeout = currentTask.expirationTime <= currentTime;
      const continuationCallback = callback(didUserCallbackTimeout);
      // 如果continuationCallback是一个函数，当前任务因为浏览器没有空闲时间而被中断了，需要继续调度一次
      // 这个continuationCallback就是performConcurrentWorkOnRoot
      if (isFunction(continuationCallback)) {
        currentTask.callback = continuationCallback;
      } else if (currentTask === peek(taskQueue)) {
        // 弹出当前执行的任务
        pop(taskQueue);
      }
    } else {
      pop(taskQueue);
    }
    // 取出下一个任务执行
    currentTask = peek(taskQueue);
  }
  // 当前帧已经执行完了，但是过期任务还没到过期时间，这个任务放到下一帧取执行
  if (currentTask !== null) {
    return true;
  } else {
    // taskQueue任务都执行完了，去看看timerQueue是否有任务需要执行
    const firstTimer = peek(timerQueue);
    if (firstTimer !== null) {
      // 让一个未过期的任务达到过期的状态，只需要延迟startTime-currentTime毫秒就行了
      requestHostTimeout(handleTimeout, firstTimer.startTime - currentTime);
    }
    return false;
  }
}

function shouldYieldToHost() {
  // 计算任务的执行时间
  const timeElapsed = getCurrentTime() - startTime;
  if (timeElapsed < frameInterval) {
    return false;
  }
  // if (enableIsInputPending) {
  //   // 如果还没有超出50ms的时间，通过isInputPending来判断是否有离散输入事件需要处理
  //   if (timeElapsed < continuousInputInterval) {
  //     if (isInputPending !== null) {
  //       return isInputPending();
  //     }
  //   } else if (timeElapsed < maxInterval) {
  //     // 如果还没有超出300ms的时间，通过isInputPending来判断是否有离散或者连续输入的事件需要处理
  //     // 例如mousemove、pointermove事件
  //     if (isInputPending !== null) {
  //       return isInputPending(continuousOptions);
  //     }
  //   } else {
  //     return true;
  //   }
  // }
  return true;
}
```

接着看核心并发入口函数`performConcurrentWorkOnRoot`的逻辑：

1. 取出`callbackNode`，这个`callbackNode`会在后面用来判断任务是否被中断执行。
2. 判断是否需要开启时间切片，判断的依据有两个：是否有阻塞、任务是否过期。
3. 如果需要开启切片则会调用`renderRootConcurrent`，否则调用`renderRootSync`。
4. 根据`root.callbackNode === originalCallbackNode`来判断任务是否被中断了，这个`originalCallbackNode`就是第一步取出的`callbackNode`。这个`callbackNode`会在`commitRoot`阶段被重置为 null，这个时候肯定是顺利进行的，没有被中断，否则说明被中断了。如果任务被中断则会返回`performConcurrentWorkOnRoot`函数，与前面讲的`workLoop`逻辑串联起来了。

```typescript
// react-reconciler/src/ReactFiberWorkLoop.ts
function performConcurrentWorkOnRoot(root: FiberRoot, didTimeout: boolean) {
  const originalCallbackNode = root.callbackNode;
  // 省略优先级代码
  // 判断是否需要开启时间切片 1、是否有阻塞 2、任务是否过期，过期了需要尽快执行（同步）
  const shouldTimeSlice = true;
  // !includesBlockingLane(root, lanes) &&
  // !includesExpiredLane(root, lanes) &&
  // !didTimeout;
  const exitStatus = shouldTimeSlice
    ? renderRootConcurrent(root, lanes)
    : renderRootSync(root, lanes);
  if (exitStatus !== RootInProgress) {
    if (exitStatus === RootCompleted) {
      const finishedWork = root.current.alternate;
      root.finishedWork = finishedWork;
      root.finishedLanes = lanes;
      finishConcurrentRender(root, exitStatus);
    }
  }
  /**
   * 说明本次调度的回调任务被中断了，这时需要返回performConcurrentWorkOnRoot，以延续之前中断的任务
   *
   * 这两个值不相等的情况：
   * 1、任务顺利调度完了，root.callbackNode会变成null
   * 2、有高优先级任务打断了低优先级任务
   */
  if (root.callbackNode === originalCallbackNode) {
    return performConcurrentWorkOnRoot.bind(null, root);
  }
  return null;
}
```

`renderRootConcurrent`中的逻辑其实和之前讲的的`renderRootSync`大致相同，唯一不同的就是`workLoopConcurrent`内部会调用`shouldYield`判断当前帧是否还有空闲时间执行调度。

```typescript
// react-reconciler/src/ReactFiberWorkLoop.ts
function renderRootConcurrent(root, lanes: Lanes) {
  // 如果根应用节点或者优先级改变，则创建一个新的workInProgress
  if (workInProgressRoot !== root || workInProgressRootRenderLanes !== lanes) {
    // 为接下去新一次渲染工作初始化参数
    prepareFreshStack(root, lanes);
  }
  do {
    try {
      workLoopConcurrent();
      break;
    } catch (e: any) {
      throw Error(e);
    }
  } while (true);

  /**
   * 如果workInProgress还存在，说明任务可能被中断了
   * 即在处理当前这个workInProgress fiber的时候，游览器没有空闲时间了，此时会中断workLoopConcurrent的循环
   * workInProgress保留当前被中断的fiber，下一次要恢复这个workInProgress fiber的执行
   */
  if (workInProgress !== null) {
    // 可能是因为中断了进入了这里
    return RootInProgress;
  } else {
    workInProgressRoot = null;
    workInProgressRootRenderLanes = NoLanes;
    return workInProgressRootExitStatus;
  }
}

function workLoopConcurrent() {
  // 留给react render的时间片不够就会中断render
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```



### 三、结束

至此，整个时间切片的大致逻辑就告一段落，下面附上流程图。

![image-20230711193426891](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230711193426891.png)

![image-20230711194514758](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230711194514758.png)