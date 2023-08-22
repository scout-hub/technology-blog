# vue3核心原理（一）—— 响应式 reactivity（上）



### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



### 一、数据响应式

数据响应式是指当数据发生变化时，程序能够自动去执行某些逻辑。在 vue3 中的提现就是，当数据变化后能够执行相关视图更新的逻辑，达到自动更新视图的效果。在前端开发中，数据响应式是非常重要的，因为它可以使得开发者不需要手动去添加数据修改后的更新逻辑，从而提高开发效率和代码可维护性。



### 二、数据响应式的发展

假设有这样一个场景，页面上有一个显示内容的容器和改变内容的按钮，内容是我们定义的 count 值，在点击按钮的时候 count 会自增，同时通过 innerText 的方法更新页面上的内容。

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="text"></div>
    <button id="btn">更新 div</button>
    <script>
        let count = 0;
        document.getElementById('btn').addEventListener('click', () => {
            count++;
            document.getElementById('text').innerText = count;
        });
    </script>
</body>

</html>
```

在这个例子中，确实实现了数据改变的同时让视图也发生改变。但是，假如这个 count 不仅仅在点击按钮的时候会发生变化，其他操作也会导致 count 发生变化，这个时候对于开发者来讲，需要在每一个可能导致 count 变化的地方都添加视图更新的代码。这会大大增加开发难度，并且代码也会变得难以维护。

如下所示，假设有一个定时器也会改变 count，在改变之后仍然需要用 innerText 方法去更新视图。

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="text"></div>
    <button id="btn">更新 div</button>
    <script>
        let count = 0;
        document.getElementById('btn').addEventListener('click', () => {
            count++;
            document.getElementById('text').innerText = count;
        });
        setInterval(() => {
            count++;
            document.getElementById('text').innerText = count;
        }, 1000);
    </script>
</body>

</html>
```

基于上面这个问题，我们需要考虑的是，有没有办法自动监测数据的变化，如果数据变化了就去执行页面更新逻辑，而不是在每一处数据变化的代码后面手动添加更新逻辑。好在 JavaScript 为我们提供了这种监测的 API。



 ### 1. `Object.defineProperty`

`Object.defineProperty`是 JavaScript 中的一个方法，用于在一个对象上定义一个新属性或修改一个已有属性的特性。它可以用来实现数据响应式，即当一个对象的属性发生变化时，相关的视图会自动更新。

我们用这个 API 来改写一下上面这个例子：

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="text"></div>
    <button id="btn">更新 div</button>
    <script>
        const defineReactive = (obj, key, val) => {
            Object.defineProperty(obj, key, {
                get() {
                    return val;
                },
                set(newVal) {
                    if(newVal !== val) {
                        val = newVal;
                        document.getElementById('text').innerHTML = newVal;
                    }
                }
            });
        }
        const data = {
            count: 0
        };
      
        for(const key in data) {
            if(Object.hasOwnProperty.call(data, key)) {
                defineReactive(data, key, data[key]);
            }
        }

        document.getElementById('btn').addEventListener('click', () => {
            data.count++;
        });
        setInterval(() => {
            data.count++;
        }, 1000);
    </script>
</body>

</html>
```

![chrome-capture-2023-7-10](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-7-10.gif)



通过这种方式，我们就无需在每一处`data.count++`后面去写视图更新的代码，达到一种响应式更新的效果。这也是 vue2 所采用的数据响应式的方式。

这种方式虽然能够实现响应式，但是它也存在着一些缺陷：

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="text"></div>
    <button id="btn">更新 div</button>
    <button id="btn1">更新 div</button>
    <script>
        const defineReactive = (obj, key, val) => {
            Object.defineProperty(obj, key, {
                get() {
                    return val;
                },
                set(newVal) {
                    if(newVal !== val) {
                        val = newVal;
                        document.getElementById('text').innerHTML = newVal;
                    }
                }
            });
        }
        const data = {
            count: 0
        };

        for(const key in data) {
            if(Object.hasOwnProperty.call(data, key)) {
                defineReactive(data, key, data[key]);
            }
        }

        document.getElementById('btn').addEventListener('click', () => {
            data.count++;
        });

        document.getElementById('btn1').addEventListener('click', () => {
            data.count1++;
        });
    </script>
</body>

</html>
```

在这个例子中我们追加了一个按钮 btn1，btn1 的点击事件中 data.count1 会自增，这个 count1 属性一开始不存在于 data 对象中，此时点击按钮页面并不会更新。

![chrome-capture-2023-7-11-2](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-7-11-2.gif)

这是因为`Object.defineProperty`无法监听到对象属性的新增或者删除。这只是其中一个问题，它还有以下几个问题：

1. 只能劫持对象的属性，无法劫持整个对象，需要对每个属性进行遍历劫持，代码量较大
2. 无法监听数组下标的变化，需要使用特殊的方法来实现
3. 对象深层次嵌套时，需要递归遍历每个属性进行劫持，性能较差



针对这些问题 vue3 采用了新的响应式方案 —— Proxy



### 2. `Proxy`

`Proxy`是 ES6 中的一个新特性，它可以用来创建一个对象的代理，从而可以拦截并重定义对象的基本操作，比如读取属性、写入属性、删除属性等。`Proxy`可以在对象的基本操作上添加自定义的行为，从而实现更加灵活和高效的编程。基本上`Object.defineProperty`中的问题`Proxy`都能解决。

我们用`Proxy`来改写一下上面这个例子：

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="text"></div>
    <button id="btn">更新 div</button>
    <button id="btn1">更新 div</button>
    <script>
        const data = {
            count: 0
        };

        const reactive = (data) => {
            return new Proxy(data, {
                get(target, key) {
                    return Reflect.get(target, key);
                },
                set(target, key, value) {
                    Reflect.set(target, key, value);
                    document.getElementById('text').innerHTML = value;
                    return true;
                }
            });
        }

        const proxyData = reactive(data);

        document.getElementById('btn').addEventListener('click', () => {
            proxyData.count++;
        });

        document.getElementById('btn1').addEventListener('click', () => {
            proxyData.count1 ? proxyData.count1++ : proxyData.count1 = 1;
        });
    </script>
</body>

</html>
```

![chrome-capture-2023-7-11-3](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-7-11-3.gif)

可以看到，当点击两个按钮时，页面上的视图都会更新。

`Proxy`的 handler 中除了可以拦截读写操作，还有以下拦截功能：

- `deleteProperty(target, key)`：拦截对象属性的删除操作。
- `has(target, key)`：拦截 `in` 操作符。
- `apply(target, thisArg, argumentsList)`：拦截函数的调用操作。
- `construct(target, argumentsList, newTarget)`：拦截 `new` 操作符。



### 三、Vue3 中的响应式

在 Vue3 中也是基于依赖收集和派发更新的响应式机制。当访问响应式数据时会收集对应的依赖函数，当响应式数据更新时会触发对应的依赖函数执行。我们用 Vue3 中的写法来改写上面提到的例子：

```typescript
import { effect, reactive } from "../src/reactive";
const data = reactive({
  count:0
});
effect(() => {
  document.getElementById('text').innerHTML = data.count;   
});
data.count++;
```

`effect（watchEffect）` 是 Vue 3 中的一个响应式 API，它可以用来追踪响应式依赖，并在响应式数据变更时执行依赖函数。在这个例子中，我们在`effect`方法中传入了依赖函数，函数内部访问了响应式数据 data 的 count 属性，这次访问会让依赖函数和响应式数据之间建立依赖关系，当 data 的 count 属性值变化时，这个依赖函数会再次执行。

我们看一下`effect`的实现原理：

```typescript
// packages/reactivity/src/effect.ts
/**
 * options:{
 *    scheduler: 用户自定义的调度器函数
 *    onStop: 清除响应式时触发回调函数;
 *    lazy: 是否懒执行，即第一次不执行fn
 * }
 */
export function effect(effectFn: Function, options: RectiveEffectOptions = {}) {
  const _effect = new ReactiveEffect(effectFn, options.scheduler);
  options && extend(_effect, options);
  // 如果不是懒执行，则执行一次依赖函数
  if (!options.lazy) _effect.run();
  const runner: any = _effect.run.bind(_effect);
  runner._effect = _effect;
  return runner;
}
```

`effect`的核心是`ReactiveEffect`:

```typescript
export class ReactiveEffect {
  deps: Dep[] = [];
  onStop?: Function;
  private active = true;

  constructor(public effectFn, public scheduler?) {}

  run() {
    try {
      activeEffect = this;
      // 省略其他代码
      const result = this.effectFn();
      // 省略其他代码
      activeEffect = undefined
      return result;
    } finally {
      // 省略其他代码
    }
  }
  
  // 省略其他代码
}
```

`ReactiveEffect`中有一个`run`方法，该方法执行时会将`activeEffect`赋值为自身实例，这个用来记录当前正在处理的依赖对象。接着执行传入的依赖函数`effectFn`，在函数执行过程中，如果函数内部有访问响应式数据，那么该响应式数据会和`activeEffect`建立依赖关系，当数据发生变化时会再次执行依赖函数。这种依赖收集和派发更新的逻辑就存在于`Proxy`中，下面我们将深入了解这个过程。



#### 1. reactive

在 Vue 3 中，`reactive` 函数可以用来创建一个响应式对象。该函数接收一个普通对象作为参数，并返回一个 Proxy 对象，对这个对象的访问和修改都会被拦截并触发更新。以下是一个简单的示例：

```typescript
import { reactive } from 'vue';

const data = {
  count: 0,
  message: 'Hello, world!'
};

const state = reactive(data);

console.log(state.count); // 0
console.log(state.message); // 'Hello, world!'

state.count++;
state.message = 'Hello, Vue 3!';

console.log(state.count); // 1
console.log(state.message); // 'Hello, Vue 3!'
```

在上面的示例中，我们使用 `reactive` 函数创建了一个响应式对象 `state`，并对其进行了访问和修改。由于 `state` 是一个 Proxy 对象，所以对其属性的访问和修改都会被拦截并触发更新。

接下去看一下`reactive`的原理：

```typescript
// packages/reactivity/src/reactive.ts
// 创建响应式对象
export function reactive(raw: object) {
  // 如果对象是一个只读的proxy，则直接返回
  if (isReadonly(raw)) {
    return raw;
  }
  return createReactiveObject(
    raw,
    reactiveHandler,
    mutableCollectionHandlers,
    reactiveMap
  );
}
```

`reactive`接受一个原始数据对象，如果这个对象已经是被 proxy 处理过的只读对象则直接返回，否则继续调用`createReactiveObject`：

```typescript
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

  /**
   * 如果映射表里有原始对象对应的代理对象，则直接返回，避免因同一个原始对象而创建出的代理对象不同导致比较失败
   * 例如：
   * const obj = {};
   * const arr: any = reactive([obj]);
   * arr.includes(arr[0])应该是true，但是返回了false
   * 因为arr[0]是obj的响应式对象，arr.includes通过下标找到arr[0]时也是obj的响应式对象
   * 如果不缓存同一个target对应的代理对象，会导致因重复创建而比较失败的情况
   */
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

`createReactiveObject`接受四个参数：

- *raw*：原始数据对象
- *baseHandler*：普通对象类型的处理器
- *collectionHandlers*：集合类型的处理器，像 Set、 Map这类数据的处理
- *proxyMap*：缓存已经响应式处理过的原始对象

1. 在`createReactiveObject`中会根据`ReactiveFlags.RAW`判断对象是否已经是响应式数据，`ReactiveFlags.RAW`属性存储的就是响应式对象的原始对象，所有响应式处理过的对象都会添加这个属性
2. 判断当前这个对象是不是已经有对应的响应式对象，如果有则直接复用之前创建过的响应式对象，防止重复创建
3. 判断当前这个对象是不是不可扩展或者不需要响应式处理，如果是则直接返回该对象
4. 调用`new Proxy`创建代理对象并缓存返回结果



对于一般的普通对象，会直接使用`reactiveHandler`作为 proxy 处理器：

```typescript
// packages/reactivity/src/baseHandlers.ts
// 响应处理器
export const reactiveHandler: ProxyHandler<object> = {
  get: reactiveGetter,
  set: reactiveSetter,
  has,
  deleteProperty,
  ownKeys,
};
```

在`reactiveHandler`中分别对`get`、`set`、`has`、`deleteProperty`、`ownKeys`进行了处理，我们一个一个来看。首先是读取操作的拦截处理：

```typescript
// packages/reactivity/src/baseHandlers.ts
// 封装proxy get函数
const createGetter = function (isReadOnly = false, isShallow = false) {
  return function (target: Target, key: string, receiver: object) {
    // 省略其他代码
    const result = Reflect.get(target, key, receiver);
    // 省略其他代码
    track(target, key);
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

在`get`操作中会通过`Reflect.get`的方式读取属性值，接着会通过`track`方法进行依赖收集，最后判断属性值是不是一个对象，如果是对象则将进行深度响应式处理。在了解`track`方法之前，我们首先要知道`Vue3`的依赖存储方式。在`Vue3`中会用一个全局变量`targetMap`来存储所有依赖，这个`targetMap`是一个`WeackMap`的数据结构。`WeakMap`中的 key 为传入的 target 对象，value 为`Map`，在`Map`中 key 对应对象属性，value 为`Set`，`Set`中存储的就是所有的`effect`。通过这种存储方式就可以管理不同数据对象中各个属性对应的依赖。

```typescript
/**
 * WeackMap{
 *    target: Map{
 *        key: Set(effectFn)
 *    }
 * }
 * 这里使用WeakMap是因为当target引用对象被销毁时，它所建立的依赖关系其实已经没有存在的必要了
 * 可以被垃圾回收机制回收
 */
```

接下去我们看一下`track`方法的逻辑：

1. 判断该依赖是否能被收集，能否被收集的依据是`shouldTrack`和`activeEffect`的值。`activeEffect`上面讲过，这是一个当前正在处理的`effect`依赖对象，如果当前没有正在处理的依赖对象则不需要进行收集。`shouldTrack`则是另一个用来标记能否进行依赖收集的标志，它用来配合实现部分不能进行依赖收集场景，这个后续会提及。
2. 获取当前对象存储的依赖映射`depsMap`，如果映射不存在则创建一个映射，并且将当前对象和映射进行关联。
3. 从映射中获取对应属性的依赖集合`dep`，如果没有则调用`createDep`创建一个依赖集合，并且将依赖集合和属性进行关联。
4. 调用`trackEffects`将当前正在被处理的依赖对象`activeEffect`收集到依赖集合中，同时将依赖集合也保存到依赖对象中的 deps 数组中，建立双向联系。

```typescript
// packages/reactivity/src/effect.ts
// 能否能进行依赖收集
export function canTrack(): boolean {
  return !!(shouldTrack && activeEffect);
}

// packages/reactivity/src/dep.ts
export function createDep(): Dep {
  const dep = new Set<ReactiveEffect>() as Dep;
  return dep;
}

// packages/reactivity/src/effect.ts
const targetMap = new WeakMap();
// 依赖收集函数
export function track(target: object, key: unknown) {
  if (!canTrack()) return;
  let depsMap = targetMap.get(target);
  if (!depsMap) {
    depsMap = new Map();
    targetMap.set(target, depsMap);
  }
  let dep = depsMap.get(key);
  if (!dep) {
    depsMap.set(key, (dep = createDep()));
  }
  trackEffects(dep);
}

export function trackEffects(dep: Dep) {
  // 省略其他代码
  dep.add(activeEffect!);
  activeEffect!.deps.push(dep);
}
```

以上就是`get`依赖收集的简单实现，下面看一下`set`派发更新的逻辑：

```typescript
// packages/reactivity/src/baseHandlers.ts
const createSetter = function () {
  return function (
    target: Target,
    key: string,
    newValue: unknown,
    receiver: object
  ) {
    // 先获取旧的值，再去更新值，避免影响触发依赖的判断 oldValue !== newValue
    const oldValue = target[key];
    // 省略部分代码
    const result = Reflect.set(target, key, newValue, receiver);
    // 省略部分代码
    if(hasChanged(newValue, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key);
    }

    return result;
  };
};
```

在`set`逻辑中会先获取旧的属性值，再通过`Reflect.set`设置新的值，最后比较新旧值，如果值不同则调用`trigger`方法触发依赖更新。`trigger`方法接受四个参数：

- target：数据更新的对象
- type：数据更新类型，比如是属性值变化，新增属性，删除属性等等
- key：更新的属性
- newValue：新的属性值

`trigger`的简单执行过程：

1. 获取当前数据对应的依赖映射`depsMap`，如果映射不存在，直接返回。
2. 获取对应属性的依赖集合，将其添加到`deps`数组中。
3. 将`deps`数组中所有的依赖集合`Set`添加到`effects`数组中，最后调用`triggerEffects`循环依赖对象，执行依赖对象上的`run`方法执行依赖函数。

```typescript
// packages/reactivity/src/effect.ts
// 触发依赖函数
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown
) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  let deps: (Dep | undefined)[] = [];
  // 省略部分代码
  // 如果key不是undefined，则获取对应key上的deps依赖集合
  if (key !== void 0) {
    deps.push(depsMap.get(key));
  }
  // 省略部分代码
  // 构建一个新的effect集合，防止无限循环，比如：删除effect的同时又添加effect
  const effects: ReactiveEffect[] = [];

  for (const dep of deps) {
    if (dep) {
      effects.push(...dep);
    }
  }
  triggerEffects(effects);
}

export function triggerEffects(deps: (Dep | ReactiveEffect)[]) {
  // dep不是数组的话转化成数组，比如ref触发依赖传入的是一个set集合
  const depsToRun = isArray(deps) ? deps : [...deps];
  depsToRun.forEach((dep: any) => {
    dep.run();
  });
}
```

以上就是简单的派发更新的逻辑，我们以之前的例子讲解一下整个过程：

1. 初始化一个响应式数据 data。
2. 调用`effect`方法并传入我们定义的依赖函数。`effect`内部会创建一个`ReactiveEffect`的实例，并且会自动调用实例上的`run`方法（不考虑 effect 的 lazy 配置），在`run`方法内部会执行我们传入的依赖函数，依赖函数执行过程中`activeEffect`的值为当前创建的`ReactiveEffect`实例。
3. 依赖函数执行时会访问 data.count，此时会触发`Proxy`中的`get`操作进行依赖收集，将数据对象 { count: 0 }，count 属性和`activeEffect`依赖对象进行关联收集。
4. data.count++ 触发`proxy`的`set`操作，从`targetMap`中取出数据对象 { count: 0 } 中 count 属性对应的依赖集合，循环集合里面的依赖对象，执行对象上的`run`方法后再次触发我们传入的依赖函数。

```typescript
import { effect, reactive } from "../src/reactive";
const data = reactive({
  count:0
});
effect(() => {
  document.getElementById('text').innerHTML = data.count;   
});
data.count++;
```

这就是整个响应式的大致原理。