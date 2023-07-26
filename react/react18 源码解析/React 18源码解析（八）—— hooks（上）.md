# React 18源码解析（八）—— Hooks（上）



### 该部分解析基于我们实现的简单版react18中的代码，是react18源码的阉割版，希望用最简洁的代码来了解react的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



### 一、什么是 React Hooks

React Hooks是React 16.8版本中引入的一项新特性，它允许在函数组件中使用状态和其他React特性，而无需编写类组件。使用Hooks可以使代码更加简洁、易于理解和重用。React Hooks包括一系列的钩子函数，如useState、useEffect、useContext等，它们可以让开发者在函数组件中使用状态、副作用和上下文等特性。使用Hooks可以使代码更加模块化，减少重复代码，提高代码的可读性和可维护性。



###  二、为什么需要 React Hooks

- 组合大于继承：React Hooks 提倡组合而不是继承的思想。通过使用对象组合的方式，可以更灵活、更简洁地实现代码的复用和扩展，而不需要依赖于类的继承关系。相比之下，类组件的继承机制有一些限制。如果我们想要在多个组件中复用相同的逻辑，需要使用继承来实现。但是这种继承的方式会导致组件之间的紧耦合，也会造成代码的冗余和难以维护。
- 函数式编程：Hooks 使用函数式编程的方式来处理组件的状态和副作用，使代码更加函数式、可测试和可预测。
- 没有 this：Hooks 不需要使用 this 关键字，避免了 this 绑定问题和使用箭头函数的限制。



### 三、Hooks 原理

1. `useState`

   `useState` 是 React 中的一个 Hook，它可以让你在函数组件中添加状态。它返回一个数组，包含当前状态值和一个更新该值的函数。

   ```jsx
   const { useState } = React;
   
   const App = () => {
     const [num, setNum] = useState(0);
     return (
       <div className="red">
         <h1>
           {num}
         </h1>
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

   我们看一下 `useState`的实现：

   ```typescript
   // packages/react/src/ReactHooks.ts
   function resolveDispatcher() {
     const dispatcher = ReactCurrentDispatcher.current;
     return dispatcher;
   }
   
   export function useState(initialState) {
     const dispatcher = resolveDispatcher()!;
     return dispatcher.useState(initialState);
   }
   ```

   当调用`useState`的时候，真正发起的是`dispatcher.useState()`，这个`dispatcher`是一个全局共用的`ReactCurrentDispatcher`:

   ```typescript
   // packages/react/src/ReactCurrentDispatcher.ts
   import type { Dispatcher } from "react-reconciler";
   
   const ReactCurrentDispatcher: { current: null | Dispatcher } = {
     current: null,
   };
   
   export default ReactCurrentDispatcher;
   ```

   我们看到初始化的时候`ReactCurrentDispatcher.current`是 null，显然有一个地方需要给这个 current 属性赋上 hooks 对象。在介绍渲染流程的时候，我们曾了解过函数式组件的渲染逻辑`mountIndeterminateComponent`，在`mountIndeterminateComponent`中会调用`renderWithHooks`来渲染函数式组件：

   ```typescript
   // packages/react-reconciler/src/ReactFiberHooks.ts
   export function renderWithHooks(
     current: Fiber | null,
     workInProgress,
     Component: Function,
     props: any,
     secondArg: any,
     nextRenderLanes: Lanes
   ) {
     renderLanes = nextRenderLanes;
     // 赋值currentlyRenderingFiber为当前的workInProgress
     currentlyRenderingFiber = workInProgress;
   
     // 要开始新的一次hook的memoizedState和updateQueue计算了，这里把之前的数据重置一下
     workInProgress.memoizedState = null;
     workInProgress.updateQueue = null;
     workInProgress.lanes = NoLanes;
   
     ReactCurrentDispatcher.current =
       current === null || current.memoizedState === null
         ? HooksDispatcherOnMount
         : HooksDispatcherOnUpdate;
   
     const children = Component(props, secondArg);
     // console.log(children);
     currentlyRenderingFiber = null;
     renderLanes = NoLanes;
     currentHook = null;
     workInProgressHook = null;
     return children;
   }
   ```

   在`renderWithHooks`中会区分首次渲染和更新的情况对应`ReactCurrentDispatcher.current`赋值，首次渲染会使用`HooksDispatcherOnMount`，更新渲染会使用`HooksDispatcherOnUpdate`：

   ```typescript
   // packages/react-reconciler/src/ReactFiberHooks.ts
   const HooksDispatcherOnMount: Dispatcher = {
     useState: mountState,
     useEffect: mountEffect,
     useLayoutEffect: mountLayoutEffect,
     useReducer: mountReducer,
     useCallback: mountCallback,
     useMemo: mountMemo,
     useDebounce: mountDebounce,
   };
   
   const HooksDispatcherOnUpdate: Dispatcher = {
     useState: updateState,
     useEffect: updateEffect,
     useLayoutEffect: updateLayoutEffect,
     useReducer: updateReducer,
     useCallback: updateCallback,
     useMemo: updateMemo,
     useDebounce: updateDebounce,
   };
   ```

   现在我们知道了 react hooks 初始化的方式，并且同一个 hooks api 在首次渲染和更新渲染时会执行不同的逻辑。因此，后续每一个 hooks 的解析都会区分首次渲染和更新的情况。

   我们继续分析首次渲染时`useState`的逻辑：

   ```typescript
   // packages/react-reconciler/src/ReactFiberHooks.ts
   /**
    * @description: mount阶段的useState函数
    */
   function mountState<S, A>(
     initialState: S | (() => S)
   ): [S, Dispatch<BasicStateAction<S>>] {
     const hook = mountWorkInProgressHook();
     if (isFunction(initialState)) {
       initialState = (initialState as () => S)();
     }
     // 首次使用hook时，hook.memoizedState就是initialState
     hook.memoizedState = hook.baseState = initialState;
     const queue: UpdateQueue<S, A> = {
       pending: null, // 指向末尾的Update
       dispatch: null,
       lastRenderedReducer: basicStateReducer,
       lastRenderedState: initialState as any,
     };
     // hook上的queue和Update上的queue一样，是一个环状链表
     hook.queue = queue;
     const dispatch: Dispatch<BasicStateAction<S>> = (queue.dispatch =
       // 绑定上当前的fiber和queue
       dispatchSetState.bind(null, currentlyRenderingFiber!, queue));
   
     return [hook.memoizedState, dispatch];
   }
   ```

   1. 调用`mountWorkInProgressHook` 生成 workInProgressHook，workInProgressHook 是一条单向链表。我们在写组件可以在组件内调用多个 hook，这些 hook 所对应的 workInProgressHook 会以单向链表的方式拼接起来。这也是我们在使用 hooks 时不能用条件判断的原因，因为在组件更新时会从头开始遍历这条 workInProgressHook 链表，链表中的每一个节点都和首次渲染的 hooks 执行顺序对应。假设其中有一个 hook 使用了条件判断，比如更新时不执行这个 hook，那么就会导致后续的 hook 对应不上 workInProgressHook 链表中的节点。

      举个简单的例子，首次渲染时 hooks 的顺序是 1->2->3，正常组件发生更新时，hooks 执行的顺序也是 1->2->3。现在 hook 2 使用了条件判断，在更新时不执行，那么组件中 hooks 的执行顺序就是 1->3。但是 workInProgressHook 链表中的顺序依旧是 1->2->3，这也就意味着组件执行到 hook 3 时会和 workInProgressHook 中的 2 对应，这显然是有问题的。

   ```typescript
   // packages/react-reconciler/src/ReactFiberHooks.ts
   /**
    * @description: 生成workInProgressHook
    */
   function mountWorkInProgressHook(): Hook {
     const hook: Hook = {
       memoizedState: null,
       baseState: null,
       baseQueue: null,
       queue: null,
       next: null,
     };
     // workInProgressHook是一个链表，通过next去添加下一个hook
     if (workInProgressHook == null) {
       // hook只能在function component中使用，而function component函数会在renderWithHooks中调用，
       // 在调用之前，currentlyRenderingFiber会被赋值为当前function component所对应的workInProgress
       currentlyRenderingFiber!.memoizedState = hook;
     } else {
       // 第二次使用hook时，将当前hook添加到workInProgressHook的末尾
       workInProgressHook!.next = hook;
     }
     // 设置workInProgressHook为当前hook
     workInProgressHook = hook;
     return workInProgressHook!;
   }
   ```

   2. 设置初始值，如果传入的是一个函数，则会获取函数返回值，赋值到 workInProgressHook 中的 memoizedState 和 baseState 属性中。
   3. 创建更新队列 queue 并绑定到 workInProgressHook 上，这个 queue 的道理和我们初始化渲染时介绍的 updateQueue 类似。其中 pending 属性指向末尾的 update 对象，dispatch 是调度更新的方法。
   4. 绑定 dispatch 更新方法，返回初始值和更新方法。

   


   我们继续看更新时的`useState`逻辑：

   ```typescript
   // packages/react-reconciler/src/ReactFiberHooks.ts
   /**
    * @description: update阶段的useState函数
    */
   function updateState<S>(
     initialState: (() => S) | S
   ): [S, Dispatch<BasicStateAction<S>>] {
     return updateReducer(basicStateReducer, initialState);
   }
   
   /**
    * @description: update阶段的useReducer
    */
   function updateReducer<S, I, A>(
     reducer: (S, A) => S,
     initialArg: I,
     init?: (I) => S
   ): [S, Dispatch<A>] {
     const hook = updateWorkInProgressHook();
     const queue = hook.queue;
     if (queue === null) throw Error("queue is null");
   
     queue.lastRenderedReducer = reducer;
     const current = currentHook!;
     let baseQueue = current.baseQueue;
     // pending指向链表的最后一位
     const pendingQueue = queue.pending;
   
     // 最后挂起的Update还没处理，把它加到baseQueue上
     if (pendingQueue !== null) {
       if (baseQueue != null) {
         // 合并baseQueue和pendingQueue
         // 1、假设baseQueue为 1->2->1  baseQueue.next是1
         const baseFirst = baseQueue.next;
         // 2、假设pendingQueue为 3->4->3 pendingQueue.next是3
         const pendingFirst = pendingQueue.next;
         // 3、2->3   ===>  1->2->3->4
         baseQueue.next = pendingFirst;
         // 4、4->1   ===>  1->2->3->4->1
         pendingQueue.next = baseFirst;
       }
   
       // baseQueue为 1->2->3->4->1
       current.baseQueue = baseQueue = pendingQueue;
       queue.pending = null;
     }
   
     // 存在baseQueue需要处理
     if (baseQueue !== null) {
       // 获取第一个Update
       const first = baseQueue.next;
       let newState = current.baseState;
   
       let newBaseState = null;
       let newBaseQueueFirst: Update<S, A> | null = null;
       let newBaseQueueLast: Update<S, A> | null = null;
       let update = first;
   
       // 循环处理所有Update，进行state的计算
       // 循环终止条件：update不存在或者且update !== first（只有一个Update的情况）
       do {
         const updateLane = update.lane;
         /**
          * 这里涉及到高优先级打断低优先级的情况
          *
          * 例如：
          * baseState:0
          * {                          {                         {
          *    lane:16,                   lane:1,                   lane:16
          *    action:1   ->              action:(n)=>n+1  ->       action:(n)=>n+1
          * }                          }                         }
          * 正常更新不管优先级的话就是按顺序执行，结果应该为:1 -> 1+1=2 -> 2+1=3
          * 如果加上优先级的处理，则先执行高优先级的，再执行低优先级的：0+1=1 -> 1=1 -> 1+1=2，显然结果已经不对了
          * 原因就是当开始执行第一个被跳过的Update时，它所依赖基状态和后续执行Update的过程可能不再和顺序执行时一致，最终导致结果不对
          *
          * 因此这里需要做的就是：
          * 1、在第一次跳过Update时，记录当前的baseState，作为下一次该Update执行时的基状态（保证再次执行这个Update时基状态和顺序执行时一致）
          * 2、把第一次跳过的Update以及后续执行的Update都接到一个新BaseQueue上（保证被跳过的Update执行时，后续的执行过程还和顺序执行时一致）
          */
         if (!isSubsetOfLanes(renderLanes, updateLane)) {
           // 当前的update是否有足够的优先级，如果不够，则跳过这个更新
           const clone: Update<S, A> = {
             lane: updateLane,
             action: update.action,
             hasEagerState: update.hasEagerState,
             eagerState: update.eagerState,
             next: null as any,
           };
           // 我们将优先级不足的Update放到newBaseQueue上
           if (newBaseQueueLast === null) {
             // 这是第一个优先级不足的Update
             newBaseQueueFirst = newBaseQueueLast = clone;
             // 这次跳过更新的Update的基状态就是newState（即下一次该Update执行时的基状态）
             newBaseState = newState;
           } else {
             // 接下去的都拼接在next上
             newBaseQueueLast = (newBaseQueueLast.next as Update<S, A>) = clone;
           }
           // 把当前被跳过的Update的优先级设置到currentlyRenderingFiber上，后面可以被冒泡到hostRoot，从而再次被调度
           currentlyRenderingFiber!.lanes = mergeLanes(
             currentlyRenderingFiber!.lanes,
             updateLane
           );
         } else {
           /**
            * 如果之前存在Update优先级不足被跳过，则将本次的Update接到newBaseQueueLast后面
            * 这样就能保证被跳过的Update执行时，调用的Update顺序还是和未跳过之前一致
            */
           if (newBaseQueueLast !== null) {
             // 将当前的Update克隆一份，设置优先级为NoLane，NoLane是所有位掩码的子集，所以永远不会被跳过
             const clone: Update<S, A> = {
               lane: NoLane,
               action: update.action,
               hasEagerState: update.hasEagerState,
               eagerState: update.eagerState,
               next: null as any,
             };
             newBaseQueueLast = (newBaseQueueLast.next as Update<S, A>) = clone;
           }
           // 使用之前已经计算好的state
           if (update.hasEagerState) {
             newState = update.eagerState;
           } else {
             const action = update.action;
             // 计算新的state
             newState = reducer(newState, action);
           }
         }
   
         update = update.next;
       } while (update !== null && update !== first);
   
       if (newBaseQueueLast === null) {
         // newBaseQueueLast不存在，说明没有被跳过的Update，所以newBaseState就是当前的Update计算的state
         newBaseState = newState;
       } else {
         // 形成循环链表
         newBaseQueueLast.next = newBaseQueueFirst as any;
       }
   
       // 新旧state是否相同，如果不同，标记receivedUpdate为true
       if (!is(newState, hook.memoizedState)) {
         markWorkInProgressReceivedUpdate();
       }
   
       hook.memoizedState = newState;
       hook.baseState = newBaseState;
       hook.baseQueue = newBaseQueueLast;
       queue.lastRenderedState = newState;
     }
   
     const dispatch: Dispatch<A> = queue.dispatch;
     return [hook.memoizedState, dispatch];
   }
   ```

   1. 调用`updateWorkInProgressHook`获取对应的 workInProgressHook。currentHook 是一个全局变量，默认是 null。如果 currentHook 是 null 表示当前要执行的是第一个 hook 的功能，假如组件中使用了三个 hook (1->2->3)，那么第一个将处理 hook1；如果 currentHook 存在，则直接通过 next 属性获取下一个要处理的 hook。currentlyRenderingFiber 是内存中正在构建的 fiber 节点，其 alternate 属性指向的就是对应页面中已经渲染的 fiber 节点。获取 current fiber 上的 memoizedState 属性值，这个值就是首次渲染时挂载的 workInProgressHook。

      判断 currentlyRenderingFiber 中是否已经存在 workInProgressHook，有的话直接使用，否则就会从 current fiber 上拷贝一份数据出来使用，最后返回 workInProgressHook
      ```typescript
      // packages/react-reconciler/src/ReactFiberHooks.ts
      /**
       * @description: 从current hook中复制得到workInProgressHook
       */
      function updateWorkInProgressHook(): Hook {
        // 下一个hook
        let nextCurrentHook: null | Hook;
        // currentHook不存在
        if (currentHook === null) {
          // 获取current Fiber
          const current = currentlyRenderingFiber?.alternate;
          if (current != null) {
            // current存在则获取hook数据
            nextCurrentHook = current.memoizedState;
          } else {
            nextCurrentHook = null;
          }
        } else {
          // currentHook存在则通过next获取下一个hook
          nextCurrentHook = currentHook.next;
        }
      
        let nextWorkInProgressHook: null | Hook;
        if (workInProgressHook === null) {
          nextWorkInProgressHook = currentlyRenderingFiber?.memoizedState;
        } else {
          nextWorkInProgressHook = workInProgressHook.next;
        }
      
        if (nextWorkInProgressHook !== null) {
          // 存在workInProgressHook，直接复用
          workInProgressHook = nextWorkInProgressHook;
          nextWorkInProgressHook = workInProgressHook.next;
          currentHook = nextCurrentHook;
        } else {
          if (nextCurrentHook === null) throw Error("nextCurrentHook is null");
          currentHook = nextCurrentHook;
          const { memoizedState, baseState, baseQueue, queue } = currentHook;
          // 从current hook复制一份
          const newHook: Hook = {
            memoizedState,
            baseState,
            baseQueue,
            queue,
            next: null,
          };
          if (workInProgressHook === null) {
            // 这是第一个hook
            currentlyRenderingFiber!.memoizedState = workInProgressHook = newHook;
          } else {
            // 添加到hook链表的最后
            workInProgressHook = workInProgressHook.next = newHook;
          }
        }
      
        return workInProgressHook!;
      }
      ```

   2. 获取更新队列 queue，获取到 queue.pending（pendingQueue），这个值就是最后一个添加的 update 更新对象。将 pendingQueue 赋值给 baseQueue 后重置 queue.pending，这个 baseQueue 可能保留了之前被中断的 update，所以需要进行 update 合并。这里先不考虑优先级打断的情况，后续单独进行分析。

   3. 初始化变量 newState，记录新的 state 值。循环处理 baseQueue 上的 update 对象，根据数据更新方式计算出新的值，这里的计算方式无非是一个新的值（setState(2)）或者是一个更新函数（setState(state=>state+1)）。这里有个 hasEagerState 的判断，这是个优化路径，第一个调用更新的 update 会再发起调度的时候就计算出新的值，这里就不需要再计算。这里的 reducer 其实和 redux 中的 reducer 类似，用来处理计算逻辑，只不过这里的 reducer 是 useState 专属的（basicStateReducer）。
      ```typescript
       // 使用之前已经计算好的state
        if (update.hasEagerState) {
            newState = update.eagerState;
        } else {
            const action = update.action;
            // 计算新的state
            newState = reducer(newState, action);
        }
      
      function basicStateReducer<S>(state: S, action: BasicStateAction<S>) {
        return isFunction(action) ? (action as (S) => S)(state) : action;
      }
      ```

   4. 将新计算得到的 state 保存到 memoizedState上。

   

   综上来看其实`useState`发起的调度更新和之前将的更新流程类似，都是循环 update 链表，获取每一个 update 做对应的处理。

   我们继续看发起调度的逻辑，也就是我们调用 setState 的地方。这块逻辑其实也和之前将的更新流程类似，无非就是创建一个更新对象 update，然后将 update 入队，入队操作也同样是构建环状链表。只不过这里有个优化路径 hasEagerState，假设当前更新队列是空的，也就是第一次调用 setState 是可以直接计算出下一个 state 状态的，当这个 state 和页面上已经存在的 state 相同时就可以提前结束，不需要再发起调度更新。

   ```typescript
   // packages/react-reconciler/src/ReactFiberHooks.ts
   function dispatchSetState<S, A>(fiber: Fiber, queue: any, action: A) {
     // 计算事件的优先级
     const lane = requestUpdateLane(fiber);
     // 创建一个update
     const update: Update<S, A> = {
       lane,
       action,
       eagerState: null,
       hasEagerState: false,
       next: null as any, // 指向下一个update，用于构建环状链表
     };
     // 判断是否是render阶段产生的更新，即直接在执行function component函数时调用了dispatchSetState
     if (isRenderPhaseUpdate(fiber)) {
       console.warn("dispatchSetState while function component is rendering");
       // return;
     } else {
       // Update入队列
       enqueueUpdate(fiber, queue, update);
       const alternate = fiber.alternate;
       if (
         fiber.lanes === NoLanes &&
         (alternate === null || alternate.lanes === NoLanes)
       ) {
         // 当前队列是空的，意味着在渲染之前我们可以立马计算出下一个state，如果新旧state是一样的就可以提早bail out
         const lastRenderedReducer = queue.lastRenderedReducer;
         if (lastRenderedReducer !== null) {
           // 获取上一次渲染阶段时的state
           const currentState = queue.lastRenderedState;
           // 获取期望的state，也就是通过调用useState返回的dispatch函数，将新的state计算方式传入并调用的结果
           const eagerState = lastRenderedReducer(currentState, action);
           // 缓存新的计算结果
           update.hasEagerState = true;
           update.eagerState = eagerState;
           // 新旧state是一样的就可以提早bail out
           if (is(eagerState, currentState)) {
             return;
           }
         }
       }
       const eventTime = requestEventTime();
       // 调度fiber节点的更新
       scheduleUpdateOnFiber(fiber, lane, eventTime);
     }
   }
   ```

   最后就是优先级调度的情况，假设我们有以下这个例子：

   ```jsx
   const { useState, useEffect } = React;
   
   const App = () => {
     const [num, setNum] = useState(0);
     useEffect(() => {
       const btn = document.getElementById("btn1");
       setTimeout(() => {
         setNum((num) => num + 2);
       }, 1000);
       setTimeout(() => {
         btn.click();
       }, 1010);
     }, []);
     return (
       <div className="red">
         <input type="text" />
         <button
           id="btn1"
           onClick={() => {
             setNum((num) => num + 1);
           }}
         ></button>
         {Array.from(new Array(1)).map((item, i) => (
           <h1 key={i}>
             <span>{num}</span>
           </h1>
         ))}
       </div>
     );
   };
   const root = ReactDOM.createRoot(document.getElementById("root"));
   root.render(<App />);
   
   ```

   在上面这个例子中，useEffect 内部会执行两个定时器操作。第一个定时器在 1s 之后会直接调用 setNum 发起调度更新，第二个定时器再1.01s 之后会触发按钮的 click 事件，在 click 事件中也会调用 setNum 触发更新。

   在之前的 lane 模型章节中提到过优先级的概念，对于 click 触发的事件优先级是一个同步的优先级，而定时器触发的事件优先级是一个普通的优先级，因此这里存在一个优先级高低的情况。并且，低优先级事件会先触发，高优先级事件后触发，这里就存在高优先级打断低优先级任务的情况。我们回顾一下 updateReducer 中的一段逻辑：

   ```typescript
   // updateReducer
   if (!isSubsetOfLanes(renderLanes, updateLane)) {
           // 当前的update是否有足够的优先级，如果不够，则跳过这个更新
           const clone: Update<S, A> = {
             lane: updateLane,
             action: update.action,
             hasEagerState: update.hasEagerState,
             eagerState: update.eagerState,
             next: null as any,
           };
           // 我们将优先级不足的Update放到newBaseQueue上
           if (newBaseQueueLast === null) {
             // 这是第一个优先级不足的Update
             newBaseQueueFirst = newBaseQueueLast = clone;
             // 这次跳过更新的Update的基状态就是newState（即下一次该Update执行时的基状态）
             newBaseState = newState;
           } else {
             // 接下去的都拼接在next上
             newBaseQueueLast = (newBaseQueueLast.next as Update<S, A>) = clone;
           }
           // 把当前被跳过的Update的优先级设置到currentlyRenderingFiber上，后面可以被冒泡到hostRoot，从而再次被调度
           currentlyRenderingFiber!.lanes = mergeLanes(
             currentlyRenderingFiber!.lanes,
             updateLane
           );
         }else {
           /**
            * 如果之前存在Update优先级不足被跳过，则将本次的Update接到newBaseQueueLast后面
            * 这样就能保证被跳过的Update执行时，调用的Update顺序还是和未跳过之前一致
            */
           if (newBaseQueueLast !== null) {
             // 将当前的Update克隆一份，设置优先级为NoLane，NoLane是所有位掩码的子集，所以永远不会被跳过
             const clone: Update<S, A> = {
               lane: NoLane,
               action: update.action,
               hasEagerState: update.hasEagerState,
               eagerState: update.eagerState,
               next: null as any,
             };
             newBaseQueueLast = (newBaseQueueLast.next as Update<S, A>) = clone;
           }
         }
     // 省略中间代码
     if (newBaseQueueLast === null) {
         // newBaseQueueLast不存在，说明没有被跳过的Update，所以newBaseState就是当前的Update计算的state
         newBaseState = newState;
       } else {
         // 形成循环链表
         newBaseQueueLast.next = newBaseQueueFirst as any;
       }
   
      hook.baseState = newBaseState;
      hook.baseQueue = newBaseQueueLast;
   ```

   在这段逻辑中有一个判断`isSubsetOfLanes`，也就是判断当前更新的任务优先级是不是渲染优先级的子集。我们知道当前渲染的优先级一定是优先级最高的，如果当前正在执行的任务优先级不在这个子集内，说明有一个更高优先级的任务执行了，这个任务的执行改变了原先的优先级。因此低优先级任务需要暂时跳过执行，当高优先级任务执行完成后再回来继续执行。

   这些被跳过的任务需要保存到 newBaseQueueFirst 和 newBaseQueueLast 上，newBaseQueueFirst 保存的是第一个被跳过的 update，newBaseQueueLast 保存的是最后一个被跳过的 update，这些 update 最终也会形成环状链表保存到 baseQueue 中。**注意，newBaseQueueLast 上保存的是当前被跳过的 update 以及后面所有的 update，这些 update 中包含高优先级任务，这么做是为了保证最终执行结果还是正确的**。（下面会举例分析）同时，在任务被打断之前可能已经执行了部分低优先级任务，这些任务执行完成的结果需要保存到 newBaseState 上，后续会继续 newBaseState 继续计算。

   最后就是保留这些被跳过的 update 上的 lanes，因为 lane 代表着是否有任务需要执行，当低优先级任务被打断后，这些 lane 需要保留，否则这些 lane 就会被清空，如果 lane 被清空了那么 react 就会认为没有任务需要执行，后续就不会再继续回来执行这些低优先级任务。

   

   上面说的是任务被打断时需要做的事，下面我们看如何恢复执行：

   ```typescript
     // updateReducer  
     if (pendingQueue !== null) {
       if (baseQueue != null) {
         // 合并baseQueue和pendingQueue
         // 1、假设baseQueue为 1->2->1  baseQueue.next是1
         const baseFirst = baseQueue.next;
         // 2、假设pendingQueue为 3->4->3 pendingQueue.next是3
         const pendingFirst = pendingQueue.next;
         // 3、2->3   ===>  1->2->3->4
         baseQueue.next = pendingFirst;
         // 4、4->1   ===>  1->2->3->4->1
         pendingQueue.next = baseFirst;
       }
   
       // baseQueue为 1->2->3->4->1
       current.baseQueue = baseQueue = pendingQueue;
       queue.pending = null;
     }
     if (baseQueue !== null) {
       // 执行 update
     }
   ```

   当高优先级任务执行完成后会先触发一次页面的更新，接着 react 会继续发起一次调度，由于之前被打断的任务的 lane 还保存着，因此调度会继续执行。当再次进到`updateReducer`方法中时，就会进入 baseQueue 的处理，此时 baseQueue 中保存的是之前被打断的 update 及其后面所有的 update。正常情况下，上一次调度完成后 pendingQueue会被清空，因此会直接执行下面 baseQueue的处理，此时执行的只是上一次被打断后的任务。如果新一次调度中又有新的 update 被添加到 pendingQueue 中，那么 pendingQueue 中的任务会被拼接到 baseQueue 之后（先执行之前被打断的，再执行新的）。这就是整个恢复执行的逻辑。

   

   下面举一个例子讲述整个过程：

   ```jsx
   /**
    * 假设调度时有三个 update，其中 lane 为 1 的 update 是高优先级任务，初始 state 为 0
    *
    * baseState:0
    * {                          {                         {
    *    lane:16,                   lane:1,                   lane:16
    *    action:1   ->              action:(n)=>n+1  ->       action:(n)=>n+1
    * }                          }                         }
    *
    * 按照正常更新按顺序执行，结果应该为:1 -> 1+1=2 -> 2+1=3
    *
    * 如果加上优先级的处理，则先执行高优先级的，再执行低优先级的：
    * 1.先执行第二个高优先级任务，此时初始 state 为 0，action 为 (n)=>n+1，计算结果为 0+1=1，此时界面先显示 1
    * 2.再执行第一个低优先级任务，此时 state 为 1，action 为 1，计算结果为 1
    * 3.最后执行第三个低优先级任务，此时 state 为 1，action 为 (n)=>n+1，结算结果为 1+1=2
    * 最终 state 的值是 2，最终界面显示 2，这显然和正常顺序执行的值 3 不同
    * 
    * 原因就是当开始执行第一个被跳过的 update 时，它所依赖基状态和后续执行Update的过程可能不再和顺序执行时一致，最终导致结果不对
    *
    * 因此这里需要做的就是：
    * 1、在第一次跳过Update时，记录当前的baseState，作为下一次该Update执行时的基状态（保证再次执行这个Update时基状态和顺序执行时一致）
    * 2、把第一次跳过的Update以及后续执行的Update都接到一个新BaseQueue上（保证被跳过的Update执行时，后续的执行过程还和顺序执行时一致）
    *
    * 按照正确的优先级调度逻辑应该是这样的：
    * 1. 第一个低优先级任务被打断，保留当前的 state=0 作为后续恢复执行的初始 state。保留当前被中断的 update 以及后续所有的 update，也就是三个 update 
    * 都被保留了。
    * 2. 执行第二个高优先级任务，计算得到 state 为 0+1=1，此时界面显示 1
    * 3. 依次执行之前被中断的 update，此时有三个 update 需要执行，并且被中断时的 state 是 0，因此按照三个 update 的顺序执行：1 -> 1+1=2 -> 2+1=3
    * 最终结果为 3，符合正常结果。
    */
   ```

   这就是为什么要记录被中断时的 state 值已经要把中断以及中断后的 update 都保存起来，其实就是为了保证最终结果是正确的。

   

2. `useReducer`
   `useReducer`和`useState`的原理其实很相似，只不过`useReducer`中的 reducer 是可以自定义的，而`useState`中的 reducer 是内部定义好的，我们可以认为`useState`是特殊的`useReducer`。

   ```jsx
   const { useReducer } = React;
   
   function initChildCount(initialCount) {
     return { count: initialCount };
   }
   
   const childReducer = (state, action) => {
     switch (action.type) {
       case "add":
         return { count: ++state.count };
       case "del":
         return { count: --state.count };
       case "reset":
         return initChildCount(action.payload);
       default:
         break;
     }
   };
   
   const Child = ({ childCount }) => {
     const [state, dispatch] = useReducer(
       childReducer,
       childCount,
       initChildCount
     );
     return (
       <div>
         <button
           onClick={() => {
             dispatch({ type: "add" });
           }}
         >
           增加
         </button>
         <button
           onClick={() => {
             dispatch({ type: "del" });
           }}
         >
           减少
         </button>
         <button
           onClick={() => {
             dispatch({ type: "reset", payload: 10 });
           }}
         >
           重置
         </button>
         child：{state.count}
       </div>
     );
   };
   
   const initState = { count: 0 };
   
   const reducer = (state, action) => {
     switch (action.type) {
       case "add":
         return { count: ++state.count };
       case "del":
         return { count: --state.count };
       default:
         break;
     }
   };
   
   const App = () => {
     const [state, dispatch] = useReducer(reducer, initState);
     return (
       <div>
         <button
           onClick={() => {
             dispatch({ type: "add" });
           }}
         >
           增加
         </button>
         <button
           onClick={() => {
             dispatch({ type: "del" });
           }}
         >
           减少
         </button>
         parent：{state.count}
         <Child childCount={state.count} />
       </div>
     );
   };
   
   const root = ReactDOM.createRoot(document.getElementById("root"));
   root.render(<App />);
   ```

   首次渲染时的 mountReducer 的逻辑很简单，无非就是可以自定义 reducer 逻辑，丰富了逻辑处理能力。更新时则完全和`useState`一样使用 updateReducer，因此整个逻辑也不需要再讲述一遍，看完`useState`的逻辑就懂了。

   ```typescript
   // packages/react-reconciler/src/ReactFiberHooks.ts
   function mountReducer<S, I, A>(
     reducer: (S, A) => S,
     initialArg: I,
     init?: (I) => S
   ): [S, Dispatch<A>] {
     const hook = mountWorkInProgressHook();
     const initialState = init === undefined ? initialArg : init(initialArg);
     hook.memoizedState = hook.baseState = initialState;
     const queue: UpdateQueue<S, A> = {
       pending: null,
       dispatch: null,
       lastRenderedReducer: reducer,
       lastRenderedState: initialState as any,
     };
     hook.queue = queue;
     const dispatch: Dispatch<A> = (queue.dispatch = dispatchReducerAction.bind(
       null,
       currentlyRenderingFiber!,
       queue
     ));
     return [hook.memoizedState, dispatch];
   }
   ```



#### 这里就先讲两个 hooks 的原理，后面章节会继续分析其他 hooks 的实现原理。