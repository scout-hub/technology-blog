# vue3核心原理（四）—— watch



### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



### 一、什么是`watch`

在Vue 3中，`watch`是一个用于监听数据变化并执行回调函数的API。它可以监听响应式数据、计算属性和ref对象的变化。下面是一个使用`watch`监听响应式数据变化的例子：

```typescript
import { reactive, watch } from 'vue';

const state = reactive({
  count: 0,
});

watch(() => state.count, (newVal, oldVal) => {
  console.log(`count changed from ${oldVal} to ${newVal}`);
});

state.count++;
```

在这个例子中，我们使用`reactive`创建了一个响应式对象`state`，并使用`watch`监听了`state.count`的变化。当`state.count`的值发生变化时，回调函数会被执行。在这个例子中，我们将`state.count`的值从0增加到1，因此回调函数会输出`count changed from 0 to 1`。



### 二、`watch`的原理

`watch`其实和`effect`有类似之处，例如：

```typescript
// effect
const o = reactive({name:'zs'});
effect(()=>{
  console.log(o.name)
});
o.name = 'ls';

// watch
watch(o, (newVal, oldVal) => {
  console.log(o.name)
});
o.name = 'ls';
```

我们可以看到，两者在用法上大致相同，其实这个`effect`就是`watch` api 中的`watchEffect`，而`watch`核心就是`ReactiveEffect`。

1. `watch`

   ```typescript
   // packages/runtime-core/src/apiWatch.ts
   export function watch(source, cb, options?) {
     return doWatch(source, cb, options);
   }
   
   function doWatch(
     source,
     cb,
     { flush, deep, immediate }: Record<string, any> = EMPTY_OBJ
   ) {
     // 定义getter函数
     let getter;
     // 旧的值
     let oldValue;
     // 新的值
     let newValue;
   
     // 根据source的类型生成不同的getter
     if (isRef(source)) {
       getter = () => source.value;
     } else if (isReactive(source)) {
       getter = () => source;
       // 传入的如果是reactive的对象，默认把deep置为true进行深度监听
       deep = true;
     } else if (isArray(source)) {
       getter = () =>
         source.map((s) => {
           if (isRef(s)) {
             return s.value;
           } else if (isReactive(source)) {
             return traverse(s);
           } else if (isFunction(s)) {
             return s();
           }
         });
     } else if (isFunction(source)) {
       // 如果传入的是方法，直接赋值给getter
       getter = source;
     }
   
     // 包装回调任务
     const job = () => {
       if (cb) {
         // newValue在值变化后触发的scheduler里面获取
         newValue = effect.run();
         cb(newValue, oldValue, onCleanup);
         // 重新赋值给旧的
         oldValue = newValue;
       } else {
         // 没有cb说明是watchEffect，直接执行副作用函数
         effect.run();
       }
     };
   
     if (cb && deep) {
       const baseGetter = getter;
       // 深度读取依赖的函数
       getter = () => traverse(baseGetter());
     }
   
     // watch的原理就是监听值的变化，通过自定义调度器来执行回调。当监听到的依赖的值变化时会触发effect上的schedular函数，从而触发回调函数
     let scheduler = () => {
         job();
     };
     
     const effect = new ReactiveEffect(getter, scheduler);
   
     if (cb) {
       oldValue = effect.run();
     } else {
       effect.run();
     }
   
     return () => {
       // 移除依赖
       effect.stop();
     };
   }
   ```

   `watch`的核心逻辑就在`doWatch`中，`doWatch`接受三个参数：

   - *source*：监听对象
   - *cb*：监听对象改变后触发的回调，回调接受三个参数，第一个是新值，第二个是旧值，第三个是副作用过期的回调
   - *options*：其它配置对象，包括 *flush* 触发时机（立即触发，组件更新前触发，组件更新后触发）；*deep* 深度监听；*immediate* 是否立即触发一次回调

   

   `doWatch`开头定义了三个变量：

   - *getter*：经过`doWatch`处理后的监听函数
   - *oldValue*：旧值
   - *newValue*：新值

   

   下面会对`getter`进行赋值处理。如果`source`是一个`ref`对象，则会监听其`value`值；如果`source`是一个`reactive`对象，则直接监听这个对象，并且会开启深度监听（deep = true）;如果`source`是一个函数，则直接使用这个监听函数；如果`source`是一个数组，则会依次遍历数组中的每一条数据，做前面类似的处理。

   ```typescript
   function doWatch(
     source,
     cb,
     { flush, deep, immediate }: Record<string, any> = EMPTY_OBJ
   ) {
     // 定义getter函数
     let getter;
     // 旧的值
     let oldValue;
     // 新的值
     let newValue;
   
     // 根据source的类型生成不同的getter
     if (isRef(source)) {
       getter = () => source.value;
     } else if (isReactive(source)) {
       getter = () => source;
       // 传入的如果是reactive的对象，默认把deep置为true进行深度监听
       deep = true;
     } else if (isArray(source)) {
       getter = () =>
         source.map((s) => {
           if (isRef(s)) {
             return s.value;
           } else if (isReactive(source)) {
             return traverse(s);
           } else if (isFunction(s)) {
             return s();
           }
         });
     } else if (isFunction(source)) {
       // 如果传入的是方法，直接赋值给getter
       getter = source;
     }
   }
   ```

   创建`ReactiveEffect`实例，传入`getter`函数以及`scheduler`自定义调度器。调度器内部会执行包装好的`job`回调任务，在回调任务中会判断是否传了监听回调`cb`。如果传了`cb`需要执行`effect.run`，也就是`getter`函数获取到新的值，之后执行`cb`回调并传入新旧值；如果没传`cb`，其实就是`effect`的用法，只需要执行`effect.run`/`getter`函数。

   ```typescript
   // 包装回调任务
   const job = () => {
     if (cb) {
       // newValue在值变化后触发的scheduler里面获取
       newValue = effect.run();
       cb(newValue, oldValue);
       // 重新赋值给旧的
       oldValue = newValue;
     } else {
       // 没有cb说明是watchEffect，直接执行副作用函数
       effect.run();
     }
   };
   // watch的原理就是监听值的变化，通过自定义调度器来执行回调。当监听到的依赖的值变化时会触发effect上的schedular函数，从而触发回调函数
   let scheduler = () => {
       job();
   };
   const effect = new ReactiveEffect(getter, scheduler);
   ```

   创建完`ReactiveEffect`之后需要默认执行一次`effect.run`记录一下旧的值，在执行`effect.run`的时候其实就已经开始进行依赖收集了，这个在前面的章节也有介绍。

   ```typescript
   if (cb) {
     oldValue = effect.run();
   } else {
     effect.run();
   }
   ```

   最后返回一个函数，这个函数用来销毁监听。

   ```typescript
   return () => {
     // 移除依赖
     effect.stop();
   };
   ```

   这就是`watch`的简单实现，总结一下其实就是利用了`ReactiveEffect`的特性，构建一个`effect`。当`effect`执行`run`的时候就会对`getter`函数中访问的响应式数据进行依赖收集，当响应式数据发生变化时直接执行回调。

2. 深度监听`deep`
   上面说到了`watch`方法的第三个参数`options`中有一个`deep`属性的配置，如果值为`true`就会开启深度监听的模式。

   ```typescript
   // packages/runtime-core/src/apiWatch.ts
   function doWatch(
     source,
     cb,
     { flush, deep, immediate }: Record<string, any> = EMPTY_OBJ
   ) {
     // 省略其它代码
     if (cb && deep) {
       const baseGetter = getter;
       // 深度读取依赖的函数
       getter = () => traverse(baseGetter());
     }
     // 省略其它代码
   }
   
   // 深度读取属性
   function traverse(value, seen = new Set()) {
     // 如果值被读取过或者值是一个普通类型则直接返回值
     if (!isObject(value) || value === null || seen.has(value)) return value;
   
     // 通过Set来防止添加重复的值
     seen.add(value);
     if (isRef(value)) {
       traverse(value.value, seen);
     } else if (isPlainObject(value)) {
       // 是对象则遍历每一个属性，递归调用traverse
       for (const key in value) {
         traverse(value[key], seen);
       }
     }
     return value;
   }
   ```

   `deep`原理就是通过`traverse`方法对值进行递归遍历，读取对象上的每一个属性值，在读取值的时候就会对每个属性进行依赖收集，当我们对某个深层次的对象属性进行更新操作时就会触发更新，从而执行依赖回调。

3. `immediate`立即执行

   ```typescript
   // packages/runtime-core/src/apiWatch.ts
   function doWatch(
     source,
     cb,
     { flush, deep, immediate }: Record<string, any> = EMPTY_OBJ
   ) {
     // 省略其它代码
     if (cb) {
       if (immediate) {
         // 立即执行一次回调
         job();
       } else {
         // 默认执行一次，获取旧的值
         oldValue = effect.run();
       }
     }
     // 省略其它代码
   }
   ```

   `immediate`的原理很简单，如果为`true`则立即执行一次包装的回调任务`job`。

4. `flush`回调执行时机

   ```typescript
   // packages/runtime-core/src/apiWatch.ts
   function doWatch(
     source,
     cb,
     { flush, deep, immediate }: Record<string, any> = EMPTY_OBJ
   ) {
     // 省略其它代码
     let scheduler;
     if (flush === "sync") {
       // 立即执行，没有加入到回调缓冲队列中
       scheduler = job;
     } else if (flush === "post") {
       // 放进微任务队列中执行（组件更新后） 
       scheduler = () => queuePostRenderEffect(job); 
     } else {
       // pre
       scheduler = () => {
         // 组件更新前调用
         scheduler = () => queuePreFlushCb(job); 
       };
     }
     // 省略其它代码
   }
   ```

   `flush`的作用就是控制`watch`回调的执行时机，如果`flush`是`sync`则当`watch`监听到变化后会立即执行`job`回调；如果是`post`则会将`job`回调加入到`postFlushCbs`类型的队列中，等待组件更新完成后执行；如果是`pre`，则会添加到`preFlushCbs`类型的队列中，在组件更新前执行。

5. `watchEffect`、`watchPostEffect`、`watchSyncEffect`
   `watchEffect`的原理其实就是之前讲的`effect`函数的原理，它不需要传第二个回调函数，它的第一个参数即依赖函数，当依赖函数内部的响应式数据变化时会自动执行依赖函数。`watchPostEffect`的原理就是`flush`的值为`post`，`watchSyncEffect`的原理就是`flush`为`sync`。



以上便是`watch`相关的 api 解析。



