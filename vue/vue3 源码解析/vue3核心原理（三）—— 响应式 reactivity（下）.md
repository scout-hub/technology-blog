# vue3核心原理（三）—— 响应式 reactivity（下）



### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



### 一、`reactive`的不足

在之前的章节中提到，我们可以通过`reactive`这个方法来创建响应式数据。但是这个方法有局限性，它只能将一个对象变成响应式数据，不能将基本类型的数据转换成响应式数据。假设我们有一个变量`flag`，它的数据类型是`boolean`，当这个`flag`变化时需要进行响应式更新，这时候单单用`reactive`就无法实现。

```typescript
import { reactive } from 'vue';
const flag = false;
const proxyFlag = reactive(flag); // value cannot be made reactive: false
```

还有一个场景，假设我们有一个变量`userInfo`，当`userInfo`内部属性变化时需要进行响应式更新，当整个`userInfo`对象被替换时也需要进行响应式更新。前者我们可以通过`reactive`实现，但是后者却不行。

```typescript
import { reactive } from 'vue';
let userInfo = reactive({name:'zs'});
userInfo.name = 'ls'; // 触发更新
userInfo = { name:'ww' }; // 不触发更新
```

当我们给`userInfo`重新赋值一个新的对象时，将原本和响应式对象之间的引用断开了，整个`userInfo`变成了原始数据对象，自然不会有响应式更新。

想要解决上面这两个问题，就要用到新的响应式 API `ref`。 



### 二、`ref`

`ref` 是 Vue 3 中的一个函数，用于创建一个响应式的数据引用。它可以用于包装基本类型的数据，如数字、字符串等，也可以用于包装对象、数组等引用类型的数据。

以下是一个示例：

```typescript
import { ref } from 'vue'

const count = ref(0)

// 访问 count 的值将返回一个响应式的数据引用
console.log(count.value) // 输出 0

// 更改 count 的值将触发视图的重新渲染
count.value++

// ref 还可以用于包装对象、数组等引用类型的数据
const obj = ref({ prop: 'value' })
console.log(obj.value.prop) // 输出 'value'

obj.value = { prop: 'new value' }
```

在上面的示例中，我们使用 `ref` 创建了一个名为 `count` 的响应式数据引用，并将其初始值设置为 `0`。我们可以通过访问 `count.value` 来获取其值，并且当我们更改 `count.value` 的值时，会触发视图的重新渲染。

我们还使用 `ref` 创建了一个名为 `obj` 的响应式数据引用，并将其初始值设置为一个包含一个属性 `prop` 的对象。我们可以通过访问 `obj.value.prop` 来获取其属性值，并且当我们更改 `obj.value` 的值时，也会触发视图的重新渲染。

`ref`的原理：

```typescript
// packages/reactivity/src/ref.ts
export function ref(value) {
  return createRef(value, false);
}

// 创建ref的工厂函数
function createRef(value, shallow) {
  if (isRef(value)) return value;
  return new RefImpl(value, shallow);
}

class RefImpl {
  private _value;
  public deps;
  private _rawValue;
  public readonly __v_isRef = true;

  constructor(value, readonly __v_isShallow = false) {
    this._value = toReactive(value);
    this._rawValue = toRaw(value);
    this.deps = new Set();
  }

  get value() {
    trackRefValue(this);
    return this._value;
  }

  set value(newValue) {
    newValue = toRaw(newValue);
    // 比较的时候拿原始值去比较
    if (hasChanged(newValue, this._rawValue)) {
      this._rawValue = newValue;
      this._value = toReactive(newValue);
      triggerRefValue(this);
    }
  }
}
```

`ref`的核心就在`RefImpl`这个类中，在`RefImpl`中定义了`constructor`方法，在这个方法中会将传入的`value`值通过`toReactive`方法转换成响应式数据，因此对于传入值为对象的情况，是完全支持响应式更新的。

对于普通数据类型，会在读取和设置`value`属性的时候去处理，在`get value`方法中会调用`trackRefValue`这个方法进行依赖收集。在`trackRefValue`中会传入`ref`自身的`deps`，将依赖都收集到自身的`deps`中。

```typescript
// packages/reactivity/src/ref.ts
// 收集ref的依赖函数
export function trackRefValue(ref) {
  if (canTrack()) {
    trackEffects(ref.deps || (ref.deps = createDep()));
  }
}
```

在`set value`中会通过先比较新旧值是否相同，如果不同则更新`value`并调用`triggerRefValue`触发依赖函数。`triggerRefValue`的逻辑就是执行自身`deps`中存储的依赖。

```typescript
// packages/reactivity/src/ref.ts
// 触发ref依赖函数
export function triggerRefValue(ref) {
  if (ref.deps) {
    triggerEffects(ref.deps);
  }
}
```

通过自定义 set 和 get value 的方式增加依赖收集和派发更新的逻辑来实现普通数据类型的响应式处理。

这就是`ref`的简单实现，下面介绍几个跟`ref`相关的 API 实现。



### 1. `shallowRef`

`shallowRef` 是 Vue 3 中的一个函数，用于创建一个浅响应式的数据引用。它与 `ref` 的区别在于，当包装一个对象时，`shallowRef` 只会对对象本身进行响应式处理，而不会对对象内部的属性进行响应式处理。

以下是一个示例：

```typescript
import { shallowRef } from 'vue'

const obj = { prop: 'value' }
const shallowObj = shallowRef(obj)

// 访问 shallowObj 的值将返回一个浅响应式的数据引用
console.log(shallowObj.value) // 输出 { prop: 'value' }

// 更改 shallowObj.value.prop 的值不会触发视图的重新渲染
shallowObj.value.prop = 'new value'
console.log(shallowObj.value) // 输出 { prop: 'new value' }
```

在上面的示例中，我们使用 `shallowRef` 创建了一个名为 `shallowObj` 的浅响应式数据引用，并将其初始值设置为一个包含一个属性 `prop` 的对象。我们可以通过访问 `shallowObj.value` 来获取其值，并且当我们更改 `shallowObj.value.prop` 的值时，不会触发视图的重新渲染。

`shallowRef`的原理：

```typescript
// packages/reactivity/src/ref.ts
class RefImpl {
  private _value;
  public deps;
  private _rawValue;
  public readonly __v_isRef = true;

  constructor(value, readonly __v_isShallow = false) {
    // 如果不是shallow的情况且value是obj时需要响应式处理
    this._value = __v_isShallow ? value : toReactive(value);
    // 如果不是shallow的情况且value如果是响应式的，则需要拿到原始对象
    // ref.spec.ts（should make nested properties reactive）
    this._rawValue = __v_isShallow ? value : toRaw(value);
    this.deps = new Set();
  }

  get value() {
    trackRefValue(this);
    return this._value;
  }

  set value(newValue) {
    // 如果不是shallow的情况且value如果是响应式的，则需要拿到原始对象 ref.spec.ts（should not trigger when setting value to same proxy）
    newValue = this.__v_isShallow ? newValue : toRaw(newValue);
    // 比较的时候拿原始值去比较
    if (hasChanged(newValue, this._rawValue)) {
      this._rawValue = newValue;
      // 如果不是shallow的情况且新的值时普通对象的话需要去响应式处理
      this._value = this.__v_isShallow ? newValue : toReactive(newValue);
      triggerRefValue(this);
    }
  }
}
```

在之前`RefImpl`类的基础上增加了`__v_isShallow`的处理，当传入的`__v_isShallow`为`true`时不会对`value`进行深层次的`toReactive`响应式处理，只有`value`属性会进行响应式处理。



### 2. `isRef`

`isRef` 是 Vue 3 中的一个函数，用于检查一个值是否为响应式的数据引用。如果是响应式的数据引用，则返回 `true`，否则返回 `false`。

以下是一个示例：

```typescript
import { ref, isRef } from 'vue'

const count = ref(0)

console.log(isRef(count)) // 输出 true
console.log(isRef(count.value)) // 输出 false
```

在上面的示例中，我们使用 `ref` 创建了一个名为 `count` 的响应式数据引用，并将其初始值设置为 `0`。我们可以通过访问 `count.value` 来获取其值，并且当我们调用 `isRef(count)` 时，会返回 `true`，表示 `count` 是一个响应式的数据引用。当我们调用 `isRef(count.value)` 时，会返回 `false`，因为 `count.value` 不是一个响应式的数据引用。

`isRef`原理：

```typescript
// packages/reactivity/src/ref.ts
// 判断一个值是不是ref
export function isRef(ref) {
  return !!(ref && ref.__v_isRef === true);
}
```

判断对象上的`__v_isRef`属性是否为`true`，在创建`RefImpl`实例的时候这个标记是个`true`。



### 3. `unRef`

`unRef` 是 Vue 3 中的一个函数，用于获取一个响应式数据引用的值。如果传入的值不是一个响应式数据引用，则直接返回该值本身。

以下是一个示例：

```typescript
import { ref, unRef } from 'vue'

const count = ref(0)

// 访问 count 的值将返回一个响应式的数据引用
console.log(count.value) // 输出 0

// 使用 unRef 获取 count 的值
console.log(unRef(count)) // 输出 0

// 如果传入的值不是一个响应式数据引用，则直接返回该值本身
console.log(unRef(123)) // 输出 123
```

在上面的示例中，我们使用 `ref` 创建了一个名为 `count` 的响应式数据引用，并将其初始值设置为 `0`。我们可以通过访问 `count.value` 来获取其值，并且当我们调用 `unRef(count)` 时，会返回 `0`，即 `count` 的值。当我们调用 `unRef(123)` 时，会直接返回 `123`，因为 `123` 不是一个响应式数据引用。

`unRef`原理：

```typescript
// packages/reactivity/src/ref.ts
export function unRef(ref) {
  return isRef(ref) ? ref.value : ref;
}
```

如果对象是一个`ref`，则返回`value`值，否则返回参数本身。



### 4. `toRef`

`toRef` 是 Vue 3 中的一个函数，可以用来为源响应式对象上的某个 property 新创建一个 ref。然后，ref 可以被传递，它会保持对其源 property 的响应式连接。

以下是一个示例：

```typescript
import { reactive, toRef } from 'vue'

const state = reactive({
  count: 0
})

const countRef = toRef(state, 'count')

// 访问 countRef 的值将返回 state.count 的值
console.log(countRef.value) // 输出 0

// 更改 countRef 的值将触发 state.count 的更改
countRef.value++

// state.count 的值也会相应地更新
console.log(state.count) // 输出 1
```

在上面的示例中，我们使用 `reactive` 创建了一个名为 `state` 的响应式对象，并将其初始值设置为 `{ count: 0 }`。我们使用 `toRef` 创建了一个名为 `countRef` 的响应式数据引用，它指向 `state` 对象上的 `count` 属性。我们可以通过访问 `countRef.value` 来获取 `state.count` 的值，并且当我们更改 `countRef.value` 的值时，会触发 `state.count` 的更改。

`toRef`原理：

```typescript
// packages/reactivity/src/ref.ts
class ObjectRefImpl {
  public readonly __v_isRef = true;
  constructor(private readonly _target, private readonly _key) { }
  get value() {
    const val = this._target[this._key];
    return val;
  }
  set value(newValue) {
    this._target[this._key] = newValue;
  }
}
export function toRef(object, key) {
  return isRef(object) ? object : new ObjectRefImpl(object, key);
}
```

将非`ref`类型的数据通过`ObjectRefImpl`转化为一个类似`ref`的数据对象，这个对象上也有自定义的`get value`和`set value` 方法，方法中操作的都是源数据，因此会保持对源属性上的响应式连接。



### 5. `toRefs`

`toRefs` 是 Vue 3 中的一个函数，它的作用是对传入的数据对象上的每一个属性都进行`toRef`操作。

以下是一个示例：

```typescript
import { reactive, toRefs } from 'vue'

const state = reactive({
  count: 0,
  message: 'Hello'
})

const refs = toRefs(state)

// 访问 refs.count.value 的值将返回 state.count 的值
console.log(refs.count.value) // 输出 0

// 更改 refs.count.value 的值将触发 state.count 的更改
refs.count.value++

// state.count 的值也会相应地更新
console.log(state.count) // 输出 1
```

在上面的示例中，我们使用 `reactive` 创建了一个名为 `state` 的响应式对象，并将其初始值设置为 `{ count: 0, message: 'Hello' }`。我们使用 `toRefs` 将 `state` 对象转换为一个普通对象 `refs`，其中对象的每个属性都是一个指向 `state` 对象上相应属性的响应式数据引用。我们可以通过访问 `refs.count.value` 来获取 `state.count` 的值，并且当我们更改 `refs.count.value` 的值时，会触发 `state.count` 的更改。

`toRefs`原理：

```typescript
// packages/reactivity/src/ref.ts
export function toRefs(object) {
  if (isRef(object)) return object;
  const result = isArray(object) ? new Array(object.length) : {};
  for (const key in object) {
    result[key] = toRef(object, key);
  }
  return result;
}
```

`toRefs`的原理就是循环对`object`上的属性值进行`toRef`的处理。



### 6. `proxyRefs`

`proxyRefs` 是 Vue 3 中的一个函数，用于将一个响应式对象转换为一个普通对象，其中所有的响应式属性都被转换为普通属性。这个函数的作用是为了方便在模板中使用对象的属性，而不需要使用 `.value` 访问器。

以下是一个使用 `proxyRefs` 的例子：

```typescript
import { ref, reactive, proxyRefs } from 'vue'

const state = reactive({
 count: 0,
 message: ref('Hello')
})

const stateProxy = proxyRefs(state)

console.log(stateProxy.count) // 0

console.log(stateProxy.message) // 'Hello'
```

在这个例子中，我们创建了一个响应式对象 `state`，其中包含一个普通属性 `count` 和一个 ref 属性 `message`。然后我们使用 `proxyRefs` 将 `state` 转换为一个普通对象 `stateProxy`，其中所有的响应式属性都被转换为普通属性。最后我们可以直接访问 `stateProxy` 的属性，而不需要使用 `.value` 访问器。

`proxyRefs`的原理：

```typescript
// 代理ref对象，使之不需要要通过.value去访问值（例如在template里面使用ref时不需要.value）
export function proxyRefs(objectWithRefs) {
  // 如果是reactive对象则不需要处理，直接返回对象
  return isReactive(objectWithRefs)
    ? objectWithRefs
    : new Proxy(objectWithRefs, {
      get(target, key, receiver) {
        return unRef(Reflect.get(target, key, receiver));
      },
      set(target, key, newValue, receiver) {
        // 旧的值是ref，但是新的值不是ref时，直接修改.value的值。否则直接设置新值
        const oldValue = target[key];
        if (isRef(oldValue) && !isRef(newValue)) {
          oldValue.value = newValue;
          return true;
        }
        return Reflect.set(target, key, newValue, receiver);
      },
    });
}
```

当传入的`objectWithRefs`为`reactive`响应式对象时直接返回，否则会创建一个`proxy`对象。当读取属性时会通过`unRef`这个方法读取`ref`值，`unRef`上面介绍过，会自动读取`ref`上的`value`属性值，当更改属性值时，如果是`ref`对象会直接操作`value`属性。



### 三、总结

一般来说如果我们要为一个对象创建响应式数据，会通过`reactive`方法，如果使用`ref`的话反而多此一举。因为`ref`内部对于对象也会使用`reactive`处理，外部反而多了一层`get value`和`set value`的操作。但是如果整个对象都需要被替换并触发响应的话，`reactive`反而不能满足。对于基础数据类型只能使用`ref`。
