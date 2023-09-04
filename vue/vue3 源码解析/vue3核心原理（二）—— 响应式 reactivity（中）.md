# vue3核心原理（二）—— 响应式 reactivity（中）



### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



### 1. shallowReactive

`shallowReactive`是Vue 3中的一个API，用于创建一个响应式的对象，它只会监听对象的第一层属性，而不会递归监听对象内部的属性。这意味着，如果对象的属性值是一个对象，那么这个对象不会被转换成响应式对象。

```typescript
import { shallowReactive } from 'vue'

const state = shallowReactive({
  count: 0,
  person: {
    name: 'John',
    age: 30
  }
})

console.log(state.count) // 0
console.log(state.person.name) // John

state.count++
console.log(state.count) // 1

state.person.name = 'Jane'
console.log(state.person.name) // Jane

state.person.age++
console.log(state.person.age) // 31
```

在上面的例子中，我们使用`shallowReactive`创建了一个响应式对象`state`，它包含了一个数字类型的属性`count`和一个对象类型的属性`person`。当我们修改`count`属性的值时，`state`对象会触发更新，因为`count`是一个响应式属性。但是，当我们修改`person`对象内部的属性时，`state`对象不会触发更新，因为`person`对象不是一个响应式对象。



`shallowReactive`的原理：

```typescript
// packages/reactivity/src/reactive.ts
export function shallowReactive(raw: object) {
  return createReactiveObject(
    raw,
    shallowReactiveHandler,
    shallowCollectionHandlers,
    shallowReactiveMap
  );
}

// packages/reactivity/src/baseHandlers.ts
// 封装proxy get函数
const createGetter = function (isReadOnly = false, isShallow = false) {
  return function (target: Target, key: string, receiver: object) {
    // 省略其他代码
    const result = Reflect.get(target, key, receiver);
    // 省略其他代码
    track(target, key);
    
    // 浅响应
    if (isShallow) {
      return result;
    }
    
    // 深响应，如果访问的属性是一个对象则继续处理对象
    if (isObject(result)) {
      return reactive(result);
    }

    return result;
  };
};
// 初始化的时候创建
const reactiveGetter = createGetter();
```

`shallowReactive`其实和`reactive`类似，只是在`createGetter`这个函数中传入的`isShallow`为`true`。当`isShallow`为`true`时，直接返回读取到的结果，不需要对读取到的结果再进行深度响应式处理。



### 2. readonly 

在 Vue 3 中，`readonly` 用于定义只读的响应式数据。这意味着只读的响应式数据可以被访问，但不能被修改。以下是一个使用 `readonly` 定义只读响应式数据的例子：

```vue
<template>
  <div>
    <p>Count: {{ count }}</p>
    <button @click="increment">Increment</button>
  </div>
</template>

<script>
import { reactive, readonly } from 'vue';

export default {
  setup() {
    const state = reactive({
      count: 0
    });

    const readonlyState = readonly(state);

    function increment() {
      readonlyState.count++;
    }

    return {
      count: readonlyState.count,
      increment
    };
  }
};
</script>
```

在上面的例子中，我们使用 `reactive` 创建了一个响应式对象 `state`，然后使用 `readonly` 创建了一个只读的响应式对象 `readonlyState`。在 `increment` 函数中，我们试图修改 `readonlyState.count`，但这会导致一个错误，因为 `readonlyState.count` 是只读的。



`readonly`的原理：

```typescript
// packages/reactivity/src/reactive.ts
// 创建只读对象
export function readonly(raw: object) {
  return createReactiveObject(
    raw,
    readonlyHandler,
    readonlyCollectionHandlers,
    readonlyMap
  );
}

// packages/reactivity/src/baseHandlers.ts
const readonlyGetter = createGetter(true);

// 只读处理器
export const readonlyHandler: ProxyHandler<object> = {
  get: readonlyGetter,
  set(target: Target, key: string, newValue: unknown, receiver: object) {
    console.warn(`${key} is readonly`);
    return true;
  },
  deleteProperty(target: Target, key: unknown) {
    console.warn(`${key} is readonly`);
    return true;
  },
};

// packages/reactivity/src/baseHandlers.ts
const createGetter = function (isReadOnly = false, isShallow = false) {
  return function (target: Target, key: string, receiver: object) {
    // 省略其他代码
    const result = Reflect.get(target, key, receiver);
    // 省略其他代码
    
    // 只读属性不能设置值，所以无需建立依赖关系
    if (!isReadOnly) {
      track(target, key);
    }
    
    // 浅响应
    if (isShallow) {
      return result;
    }
    
    // 深响应，如果访问的属性是一个对象则继续处理对象
    if (isObject(result)) {
      return isReadOnly ? readonly(result) : reactive(result);
    }

    return result;
  };
};
```

对于只读对象，在`createGetter`中会传入`isReadOnly`为`true`，对于`isReadOnly`为`true`的对象不会进行依赖收集。同时，对于`set`和`deleteProperty`操作也会做出警告提示。



### 3. shallowReadonly

`shallowReadonly` 是 Vue 3 中的一个函数，它接收一个对象作为参数，返回一个只读的代理对象，该代理对象的所有属性都是只读的，但是不会递归地将嵌套对象的属性变为只读。

以下是一个使用 `shallowReadonly` 的示例：

```vue
<template>
  <div>
    <p>Count: {{ state.count }}</p>
    <p>Name: {{ state.user.name }}</p>
    <p>Age: {{ state.user.age }}</p>
  </div>
</template>

<script>
import { shallowReadonly } from 'vue'

export default {
  setup() {
    const state = shallowReadonly({
      count: 0,
      user: {
        name: 'John',
        age: 30
      }
    })

    // 以下代码会报错，因为 state.count 是只读的
    state.count = 1

    // 以下代码不会报错，因为 state.user 是可写的
    state.user.age = 31

    return {
      state
    }
  }
}
</script>
```

在上面的示例中，我们使用 `shallowReadonly` 将 `state` 对象变为只读。由于 `shallowReadonly` 不会递归地将嵌套对象的属性变为只读，因此 `state.user` 对象的属性仍然是可写的。在模板中，我们可以访问 `state` 对象的属性，但是不能修改它们。

`shallowReadonly`的原理就是`isShallow`和`isReadOnly`的结合，`isShallow`为`true`时不会对深层对象进行处理，因此深层对象依旧可以修改，不会受到`isReadOnly`的影响。



### 3. isReactive

`isReactive`是Vue 3中的一个函数，用于检查一个对象是否是响应式的。如果对象是响应式的，则返回`true`，否则返回`false`。

以下是一个示例：

```typescript
import { reactive, isReactive } from 'vue'

const state = reactive({
  count: 0
})

console.log(isReactive(state)) // true
console.log(isReactive({})) // false
```

`isReactive`的原理就是访问该对象上的`IS_REACTIVE`属性值。在 get 拦截器中会增加读取`IS_REACTIVE`属性值的处理，返回`!isReadOnly`的值，这个值在对象响应式处理的时候就会被闭包缓存。

```typescript
// packages/reactivity/src/reactive.ts
// 对象是不是响应式的
export function isReactive(variable: unknown): boolean {
  return !!(variable as Target)[ReactiveFlags.IS_REACTIVE];
}

// packages/reactivity/src/baseHandlers.ts
const createGetter = function (isReadOnly = false, isShallow = false) {
  return function (target: Target, key: string, receiver: object) {
    // 如果访问的是__v_reactive，则返回!isReadOnly的值
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadOnly;
    }
    // 省略其他代码
    const result = Reflect.get(target, key, receiver);
    // 省略其他代码
    
    // 只读属性不能设置值，所以无需建立依赖关系
    if (!isReadOnly) {
      track(target, key);
    }
    
    // 浅响应
    if (isShallow) {
      return result;
    }
    
    // 深响应，如果访问的属性是一个对象则继续处理对象
    if (isObject(result)) {
      return isReadOnly ? readonly(result) : reactive(result);
    }

    return result;
  };
};
```



### 4. isReadonly

`isReadonly`是Vue 3中的一个函数，用于检查一个对象是否是只读的。如果对象是只读的，则返回`true`，否则返回`false`。

以下是一个示例：

```typescript
import { readonly, isReadonly } from 'vue'

const state = readonly({
  count: 0
})

console.log(isReadonly(state)) // true
console.log(isReadonly({})) // false
```

`isReadonly` 的原理和`isReactive`类似，在读取属性`IS_READONLY`的时候返回`isReadonly`的值。

```typescript
// packages/reactivity/src/reactive.ts
// 对象是不是只读的
export function isReadonly(variable: unknown): boolean {
  return !!(variable as Target)[ReactiveFlags.IS_READONLY];
}

// packages/reactivity/src/baseHandlers.ts
const createGetter = function (isReadOnly = false, isShallow = false) {
  return function (target: Target, key: string, receiver: object) {
    // 如果访问的是__v_reactive，则返回!isReadOnly的值
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadOnly;
    }
    // 如果访问的是__v_isReadonly，则返回isReadOnly值
    if (key === ReactiveFlags.IS_READONLY) {
      return isReadOnly;
    }
    // 省略其他代码
    const result = Reflect.get(target, key, receiver);
    // 省略其他代码
    
    // 只读属性不能设置值，所以无需建立依赖关系
    if (!isReadOnly) {
      track(target, key);
    }
    
    // 浅响应
    if (isShallow) {
      return result;
    }
    
    // 深响应，如果访问的属性是一个对象则继续处理对象
    if (isObject(result)) {
      return isReadOnly ? readonly(result) : reactive(result);
    }

    return result;
  };
};
```



### 4. isProxy

`isProxy`是Vue 3中的一个函数，用于检查一个对象是否是响应式或只读的代理对象。如果对象是响应式或只读的代理对象，则返回`true`，否则返回`false`。

以下是一个示例：

```typescript
import { reactive, readonly, isProxy } from 'vue'

const state = reactive({
  count: 0
})

const readonlyState = readonly(state)

console.log(isProxy(state)) // true
console.log(isProxy(readonlyState)) // true
console.log(isProxy({})) // false
```

`isProxy`的原理就是`isReadonly`和`isReactive`整合：

```typescript
// packages/reactivity/src/reactive.ts
// 对象是不是readonly或者reactive的
export function isProxy(variable: unknown): boolean {
  return isReactive(variable) || isReadonly(variable);
}
```



### 5. toRaw

`toRaw`是Vue 3中的一个函数，用于获取一个响应式对象的原始对象。如果传入的对象不是响应式对象，则返回该对象本身。

以下是一个示例：

```typescript
import { reactive, toRaw } from 'vue'

const o ={
  count: 0
}

const state = reactive(o)

const rawState = toRaw(state)

console.log(rawState === o) // true
```

`toRaw`的原理：

```typescript
// packages/reactivity/src/reactive.ts
// 返回代理对象的原始对象
export function toRaw<T>(observed: T): T {
  const raw = observed && observed[ReactiveFlags.RAW];
  // toRaw返回的对象依旧是代理对象，则递归去找原始对象
  return raw ? toRaw(raw) : observed;
}

// packages/reactivity/src/baseHandlers.ts
const createGetter = function (isReadOnly = false, isShallow = false) {
  return function (target: Target, key: string, receiver: object) {
    // 如果访问的是__v_reactive，则返回!isReadOnly的值
    if (key === ReactiveFlags.IS_REACTIVE) {
      return !isReadOnly;
    }
    // 如果访问的是__v_isReadonly，则返回isReadOnly值
    if (key === ReactiveFlags.IS_READONLY) {
      return isReadOnly;
    }
    
    // 如果访问的是__v_raw属性，就返回原始对象
    if (key === ReactiveFlags.RAW) {
      return target;
    }
    
    // 省略其他代码
    const result = Reflect.get(target, key, receiver);
    // 省略其他代码
    
    // 只读属性不能设置值，所以无需建立依赖关系
    if (!isReadOnly) {
      track(target, key);
    }
    
    // 浅响应
    if (isShallow) {
      return result;
    }
    
    // 深响应，如果访问的属性是一个对象则继续处理对象
    if (isObject(result)) {
      return isReadOnly ? readonly(result) : reactive(result);
    }

    return result;
  };
};
```

通过递归的方式不停地去读取对象上的`RAW`属性，在`createGetter`方法中，如果读取的是`RAW`属性，则会返回原始对象`target`。



### 6. markRaw

`markRaw` 是 Vue 3 中的一个函数，用于标记一个对象，使其在响应式系统中不被视为响应式的。这意味着当对象被更改时，不会触发视图的重新渲染。

以下是一个示例：

```typescript
import { reactive, markRaw } from 'vue'

const state = reactive({
  count: 0,
  obj: markRaw({ prop: 'value' })
})

// 更改 state.count 的值将触发视图的重新渲染
state.count++

// 更改 state.obj.prop 的值不会触发视图的重新渲染
state.obj.prop = 'new value'
```

`markRaw`的原理：

```typescript
// packages/reactivity/src/reactive.ts
export function markRaw<T extends object>(value: T): T {
  def(value, ReactiveFlags.SKIP, true);
  return value;
}

function targetTypeMap(type: string) {
  switch (type) {
    case "Object":
    case "Array":
      return TargetType.COMMON;
    case "Map":
    case "Set":
    case "WeakMap":
    case "WeakSet":
      return TargetType.COLLECTION;
    default:
      return TargetType.INVALID;
  }
}

// 获取当前数据的类型
function getTargetType(value: Target) {
  // 如果对象是被标记为永远不被响应式处理(markRaw)或者对象不能被扩展的，则直接返回INVALID类型
  // 否则返回对应的类型
  return value[ReactiveFlags.SKIP] || !Object.isExtensible(value)
    ? TargetType.INVALID
    : targetTypeMap(toRawType(value));
}

// packages/reactivity/src/reactive.ts
function createReactiveObject(
  raw: Target,
  baseHandler: ProxyHandler<object>,
  collectionHandlers: ProxyHandler<any>,
  proxyMap: WeakMap<Target, any>
) {
  // 如果已经是响应式对象了，直接返回。
  // TODO 除非是将响应式对象转化为readonly
  if (raw[ReactiveFlags.RAW]) {
    return raw;
  }

  const existingProxy = proxyMap.get(raw);
  if (existingProxy) return existingProxy;

  const targetType = getTargetType(raw);
  // 如果对象被指定为永远不需要响应式处理或者对象不可扩展，则直接返回原始值
  if (targetType === TargetType.INVALID) {
    return raw;
  }

  const proxy = new Proxy(
    raw,
    // 集合类型例如Set、WeakSet、Map、WeakMap需要另外的handler处理
    targetType === TargetType.COLLECTION ? collectionHandlers : baseHandler
  );
  proxyMap.set(raw, proxy);
  return proxy;
}
```

使用了`markRaw`的对象会被标记上`SKIP`属性，当创建响应式对象时会通过`getTargetType`获取到原始对象的类型，如果原始对象上有`SKIP`属性，则会返回`INVALID`类型。对于`INVALID`的类型的数据会返回其原始对象，不会进行响应式处理。



### 7. computed

`computed` 是 Vue 3 中的一个函数，用于创建一个计算属性。计算属性是一种派生数据，它的值是基于其他响应式数据计算而来的，当依赖的数据发生变化时，计算属性的值也会相应地更新。

以下是一个示例：

```typescript
import { reactive, computed } from 'vue'

const state = reactive({
  count: 0
})

const doubleCount = computed(() => state.count * 2)

// 访问 doubleCount 的值将返回 state.count 的两倍
console.log(doubleCount.value) // 输出 0

// 更改 state.count 的值将触发 doubleCount 的重新计算
state.count++
console.log(doubleCount.value) // 输出 2
```

`computed`原理：

```typescript
// packages/reactivity/src/computed.ts
import { ReactiveEffect } from "./effect";
import { trackRefValue, triggerRefValue } from "./ref";

export type ComputedGetter<T> = (...args: any[]) => T;

class ComputedRefImpl<T> {
  private _dirty = true; // true表示需要重新计算新的值
  private _lazyEffect;
  private _oldValue!: T; // 缓存的值

  constructor(getter: ComputedGetter<T>) {
    this._lazyEffect = new ReactiveEffect(getter, () => {
      // 借助scheduler是否执行去判断依赖的响应式对象是否发生变化
      if (!this._dirty) {
        this._dirty = true;
        // 触发value属性相关的effect函数
        triggerRefValue(this);
      }
    });
  }

  // value属性后才去执行getter函数获取值
  get value() {
    /**
     * 当通过.value访问值时，如果计算属性计算时依赖的响应式对象数据没有发生变化
     * 则返回旧的值，否则重新计算新的值。
     * 这里就需要借助effect去帮助我们实现 当依赖的响应式对象发生变化时 重新计算的逻辑
     */
    trackRefValue(this);
    if (this._dirty) {
      this._dirty = false;
      this._oldValue = this._lazyEffect.run();
    }
    /**
     * 在effect中使用计算属性时会发生嵌套effect的情况
     * 由于计算属性内部有单独的lazy effect，因此内部的响应式数据只会收集该effect
     * 不会收集外层的effect，导致computed内部的响应式数据变化时，外层effect函数不会触发
     * 解决方式：
     * 在访问计算属性时，触发依赖收集，将外层effect和计算属性的value值进行关联
     */
    return this._oldValue;
  }
}

// 计算属性
// 接受一个getter
export function computed<T>(getter: ComputedGetter<T>) {
  return new ComputedRefImpl(getter);
}

```

`computed`函数内部返回了一个`ComputedRefImpl`实例，这个`ComputedRefImpl`在实例化的时候会先创建一个`ReactiveEffect`实例，我们简称它为`lazyEffect`，因为它不会立即执行传入的`getter`依赖函数，只有调用了内部的`run`方法才会执行。这也符合`computed`的思想，当我们用`computed`创建计算属性时，只有访问其`value`属性才会拿到计算结果。因此这里需要定义一个`get value`的逻辑，当访问`value`属性时需要调用`this._lazyEffect.run()`方法执行传入的`getter`依赖函数，拿到计算结果，这个计算结果会被作为`oldValue`。

这里还有一个内部属性`_dirty`，这个`_dirty`被当做是否需要重新计算新值的标记，值为`true`时表示需要重新计算新值，值为`false` 时不会计算。当我们多次访问`value`属性时，只有第一次会计算，后面几次只会使用第一次的计算结果`oldValue`，从而达到缓存的效果。既然`oldValue`是重新计算的一个标记，那么就必须在内部依赖数据改变的时让它重新变成`true`，否则没办法重新计算新的值。这就要借助`effect`的自定义调度功能了。

在创建`lazyEffect`的时候我们传入了第二个参数，这个参数表示自定义调度函数，当`effect`内部依赖的响应式数据改变时不会触发依赖函数，而是会触发我们传入的自定义调度函数。这个逻辑在`triggerEffect`中：

```typescript
// packages/reactivity/src/effect.ts
// 抽离公共的触发依赖逻辑
export function triggerEffects(deps: (Dep | ReactiveEffect)[]) {
  // dep不是数组的话转化成数组，比如ref触发依赖传入的是一个set集合
  const depsToRun = isArray(deps) ? deps : [...deps];
  depsToRun.forEach((dep: any) => {
    if (dep !== activeEffect) {
      const scheduler = dep.scheduler;
      // 触发依赖的时候，如果存在用户自定义调度器，则执行调度器函数，否则执行依赖函数
      scheduler ? scheduler(dep.effectFn) : dep.run();
    }
  });
}
```

`effect`的第二个参数就是这个`scheduler`。在`lazyEffect`中传入了一个自定义调度函数，当依赖数据变化时会触发这个调度函数，函数内部会将`_dirty`重置为`true`，当我们再次访问`value`属性时就会重新计算得到新的结果。这就是整个`computed`的原理。

