# vue3核心原理（八）——  props 和 attrs

### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



## 一、props

在 Vue 3 中，props 是父组件向子组件传递数据的一种方式。它们是子组件的一部分，可以在子组件内部使用。props 是属性的简写，它们是只读的，意味着你不能在子组件内部修改它们。

下面是一个 Vue 3 中使用 props 的例子：

```typescript
// 父组件
const ParentComponent = {
  template: `
    <div>
      <ChildComponent :message="parentMessage" />
    </div>
  `,
  setup() {
    return {
      parentMessage: 'Hello from parent'
    }
  },
  components: {
    ChildComponent
  }
}

// 子组件
const ChildComponent = {
  props: ['message'],
  template: `
    <div>
      {{ message }}
    </div>
  `
}
```

在上面的例子中，`ParentComponent` 是父组件，它向 `ChildComponent` 子组件传递了一个名为 `message` 的 prop。在子组件中，我们通过 `props: ['message']` 定义了 `message` prop，并在模板中使用了它。

下面我们来看一下 props 的实现机制，首先还是编写一个 demo：

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        .red {
            color: red;
        }

        .green {
            color: green;
        }
    </style>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            createApp,
        } = Vue;

        const Child = {
            name: "Child",
            props: ['msg'],
            setup(props) {
                console.log(props);
            }
        }

        const App = {
            name: "App",
            components: {
                Child
            },
            setup() {
                return {
                    msg: 'test123',
                }
            },
            template: `<Child id="div" class="red green" :msg="msg"/>`
        };

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

在这个例子中，父组件 App 里面引用了子组件 Child，并且传递了 id、class、msg 属性，子组件在 setup 方法中打印父组件传递过来的 props。

之前介绍初始化渲染流程中，我们知道组件的初始挂载逻辑在 `mountComponent` 这个方法中。它里面有三步骤：初始化组件实例（`createComponentInstance`），执行组件setup逻辑（`setupComponent`），执行组件渲染（`setupRenderEffect`）。

在初始化组件实例的方法中会初始化 props 选项，也就是组件配置对象上的 props 属性，只有在 props 选项中定义的属性才会真正被 props 接收。在上面的例子中，子组件的 props 选项中只定义了 msg 属性，因此子组件 props 对象中只会有 msg 的值，剩下的两个属性 id 和 class 不会在 props 对象中。

props 选项的初始化在 `createComponentInstance` 中：

```typescript
// packages/runtime-core/src/component.ts
export function createComponentInstance(vnode, parent) {
  const componentInstance = {
    propsOptions: normalizePropsOptions(type),
    props: EMPTY_OBJ,
  };
  // 省略其它代码
}
```

在初始化组件实例的时候会初始化 props 和 propsOptions， propsOptions 的初始化逻辑在`normalizePropsOptions` 方法中：

```typescript
// packages/runtime-core/src/componentProps.ts
export function normalizePropsOptions(comp) {
  /**
   * propsOptions类型
   * 1、数组 =====> props:['name','value']
   * 2、对象 =====> props:{name:{type:'xxx'……}} || props:{name:Person} || props:{name:[String,……]}
   */
  const { props: rawPropsOptions } = comp;
  const normalized = {};

  if (isArray(rawPropsOptions)) {
    const propsLength = rawPropsOptions.length;
    for (let i = 0; i < propsLength; i++) {
      const normalizedKey = camelize(rawPropsOptions[i]);
      if (validatePropName(normalizedKey)) {
        // 如果属性名符合规范，则默认初始化为空对象
        normalized[normalizedKey] = EMPTY_OBJ;
      }
    }
  } else if (isObject(rawPropsOptions)) {
    for (const key in rawPropsOptions) {
      const normalizedKey = camelize(key);
      if (validatePropName(normalizedKey)) {
        const options = rawPropsOptions[key];
        // 如果属性名对应的值是一个数组或者函数，那么这个值就是name对应的type
        normalized[normalizedKey] =
          isArray(options) || isFunction(options)
            ? { type: options }
            : options;
      }
    }
  }
  return [normalized];
}
```

props 选项有数组和对象两种定义方式。

第一种数组的方式：

```typescript
const Child = {
  props:['xxx','yyy']Ï
}
```

对于数组形式的写法会使用 for 循环进行处理，每一个属性名都会通过 `camelize` 方法将烤肉串命名转化为驼峰命名（add-name ===> addName）。接着会调用 `validatePropName` 对属性名进行校验，以 $ 开头的属性名都会被定义为非法名称。这个之前也讲过，$ 开头的属性是内部属性，如果定义这种 $ 开头的属性名很有可能会污染内部属性。最后以对象的形式存到 `normalized` 中。

```typescript
propsOptions:['xxx','yyy']
// 内部处理之后统一变成对象的形式
propsOptions:{
  'xxx': {},
  'yyy': {}
}
```

我们用断点来看一下数组形式的处理过程：

![image-20231222172101938](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231222172101938.png)

进入断点的时候可以看到初始外部配置是 `['msg']`

![image-20231222172251477](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231222172251477.png)

接下去进入数组处理的分支进行 for 循环处理，转化属性名然后判断名称合法性，最后存到 `normalized` 中。

![image-20231222172501490](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231222172501490.png)



第二种对象的方式：

```typescript
const Child = {
  'xxx': {
    type: Number
  },
  'yyy': String,
  'zzz'：[String, Number],
}
```

对于对象方式，我们可以定义类型，这个类型可以用来校验父组件传过来的值是否符合类型要求。类型的定义方式有三种，第一种是上面 `xxx` 的定义方式，第二种是上面 `yyy` 的定义方式，第三种是上面 `zzz` 的定义方式。对于后两种属性值为方法或者数组的情况可以直接认为是类型的值，需要将其统一为第一种方式，最后存在 `normalized` 对象中。

```typescript
'yyy': String
// 转换为
'yyy'：{
  type: String
}
'zzz'：[String, Number]
// 转换为
'zzz'：{
  type: [String, Number]
}
```

我们继续用断点调试这个过程：

![image-20231222172914048](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231222172914048.png)

一开始可以看到配置的形式，它是一个对象，因此会进入对象形式的处理。

![image-20231225111546606](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225111546606.png)

第一个处理的是属性 `x`，`options`一个对象，因此不需要经过其它处理，直接存到 `normalized` 对象上即可。

![image-20231225111717716](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225111717716.png)

第二个处理的是属性 `y`，`options` 是一个函数，因此会构建一个新的对象，将 `options` 作为 `type` 字段的值存入对象中，最后将新创建的对象存到 `normalized` 中。

![image-20231225111940163](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225111940163.png)

第三个 `z` 也是同第二个 `y` 一样的处理。

![image-20231225112030129](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225112030129.png)

最终的 `normalized` 对象就是上面这种形式。至此，`propsOptions` 的初始化过程就结束了。

但是，我们在上图中还可以看到这个初始化过程还返回了一个 `needCastKeys`的值，它其实是用来标记哪些属性的值需要二次处理的。例如 <button disabled></button> 本意为禁用，但是经过模板解析后成了 `props:{ disabled: '' }`，`''`这个值通过`el.disabled`（`disabled`的这个属性值在dom 元素上的类型为 Boolean）设置会转化为 false，也就是不禁用，这显然和用户的真实意图相反，因此对于这种属性需要进行标记，在初始化 `props` 中做额外的处理。



接下去就是初始化 `props`，props 的初始化在中间这一步 `setupComponent` 中。

```typescript
// packages/runtime-core/src/component.ts
export function setupComponent(instance) {
  const { props } = instance.vnode;
  // 初始化props
  initProps(instance, props, isStateful);
  // 省略其它代码
}

// packages/runtime-core/src/componentProps.ts
/**
 * @author: Zhouqi
 * @description: 初始化props
 * @param instance 组件实例
 * @param rawProps 初始状态下的props
 * @param isStateful 是否是有状态组件
 */
export function initProps(instance, rawProps, isStateful) {
  const props = {};

  setFullProps(instance, rawProps, props);

  for (const key in instance.propsOptions[0]) {
    // 没有传入的props值默认置为undefined
    if (!(key in props)) {
      props[key] = undefined;
    }
  }

  // 校验数据是否合法
  validateProps(rawProps || {}, props, instance);

  // 有状态组件
  if (isStateful) {
    instance.props = shallowReactive(props);
  } else {
    // 省略函数式组件的处理
  }
}
```

在 `initProps` 中会调用 `setFullProps` 对原始的 `props` 对象进行处理，比如区分真正需要接收的 `props` 和 `attrs`，还有上面提到的需要二次处理的属性 `needCastKeys`。接着会对没有传入的 `props` 属性，但是在`propsOptions` 上有定义的属性进行初始化赋值（undefined），然后校验整个 `props` 数据的合法性，最后设置到组件实例上的 `props` 属性中。

```typescript
// packages/runtime-core/src/componentProps.ts
/**
 * @author: Zhouqi
 * @description: 处理props和attrs
 * @param instance 组件实例
 * @param rawProps props对象
 * @param props 处理后最终的props
 */
function setFullProps(instance, rawProps, props) {
  const [normalized, needCastKeys] = instance.propsOptions;
  
  if (rawProps) {
    for (const key in rawProps) {
      // 对于保留关键字不需要处理，例如key、ref，这里不像vue3一样用isReservedProp函数做处理
      if (key === "ref" || key === "key")  continue;
      const value = rawProps[key];
      let camelKey;
      // 统一转为驼峰命名
      if (normalized && hasOwn(normalized, (camelKey = camelize(key)))) {
        props[camelKey] = value;
      }
    }
  }

  // 特殊处理
  if (needCastKeys) {
    // 省略代码
  }
}
```

在 `setFullProps` 中会遍历所有的 `props`，如果某个属性在 `propsOptions` 中有定义，则会将属性和值添加到组件实例所需要的 `props`对象中。

`props`校验在 `validateProps`中，其内部会调用 `validateProps`进行校验处理。

```typescript
// packages/runtime-core/src/componentProps.ts
/**
 * @author: Zhouqi
 * @description: 检验props合法性
 * @param rawProps 原始数据
 * @param props 经过setFullProps处理过的props
 * @param instance 组件实例
 */
function validateProps(rawProps, props, instance) {
  const [options] = instance.propsOptions;
  for (const key in options) {
    const opt = options[key];
    if (!opt) continue;
    validateProp(key, props[key], opt, !hasOwn(rawProps, key));
  }
}

/**
 * @author: Zhouqi
 * @description: 校验单个prop的值
 * @param key 属性名
 * @param value 属性值
 * @param propOption 校验选项
 * @param isAbsent 传入的prop中是否缺少指定的prop key
 */
function validateProp(key, value, propOption, isAbsent) {
  const { type, required, validator } = propOption;
  // 如果是必填的但是没有传值，就警告
  if (required && isAbsent) {
    console.warn(`${key} is Required`);
    return;
  }

  // value是null或者undefined且不需要必填时，直接跳过
  if (value == null && !required) {
    return;
  }

  if (type) {
    const types = isArray(type) ? type : [type];
    let isValid = false;
    for (let i = 0; i < types.length && !isValid; i++) {
      const { valid } = assertType(value, types[i]);
      isValid = valid;
    }
    if (!isValid) {
      console.warn("校验不通过");
      return;
    }
  }
  // 自定义校验器
  if (validator && !validator(value)) {
    console.warn("校验不通过");
  }
}
```

在 `validateProps`中会获取 `propsOptions` 中关于校验的配置，比如数据类型 `type`， `required`是否必要以及自定义校验函数 `validator`。

以下几种情况会提示校验不通过：

- `required`为`true`的属性没传
- 传递的值的类型不在 `type`的定义范围内
- 自定义校验函数 `validator` 的返回值是假值

以上便是 `props` 的初始化过程。



## 二、attrs

上面介绍 props 的过程中，我们有提到过多次**所有的 props**和**组件实例上的 props**，这两个是不一样的概念。**所有的 props **是指在`VNode` 上存储的 props 对象，这个 `props` 包含所有外部传入的属性，而**组件实例上的 props**则是 `propsOptions`中定义的，也就是组件需要使用的 `props。

既然组件需要用到的 `props` 会存到 props 对象中，那不需要的会存到哪儿？没错，就是 `attrs`。

在 Vue 3 中，`attrs` 是一个特殊的属性，它包含了传递给一个组件的 attribute，但不包括那些已经在组件的 `props` 选项中明确定义的 attribute。

例如，如果你有一个组件定义如下：

```typescript
const MyComponent = {
  props: ['propA', 'propB'],
  setup(props, { attrs }) {
    // ...
  }
}
```

然后你在使用这个组件时这样写：

```vue
<MyComponent propA="Hello" propB="World" customAttr="Vue 3" />
```

<**MyComponent** propA="Hello" propB="World" customAttr="Vue 3" />

在这个例子中，`props` 对象会包含 `propA` 和 `propB`，而 `attrs` 对象则会包含 `customAttr`。

这样做的好处是，你可以在不知道父组件会传递什么 attribute 的情况下，将这些 attribute 应用到子组件的某个元素上。例如，你可以在子组件的模板中这样写：

```vue
<div v-bind="attrs">
  <!-- 其他内容 -->
</div>
```

这样，`customAttr` 就会被应用到这个 `div` 元素上。

先前介绍 `props`的初始化流程中，我们省略了 `attrs` 的处理逻辑，下面我们补上这段逻辑：

```typescript
// packages/runtime-core/src/componentProps.ts
export function initProps(instance, rawProps, isStateful) {
  // 新增部分的逻辑
  const attrs = {};
  const props = {};

  setFullProps(instance, rawProps, props, attrs);

  for (const key in instance.propsOptions[0]) {
    if (!(key in props)) {
      props[key] = undefined;
    }
  }

  validateProps(rawProps || {}, props, instance);

  if (isStateful) {
    instance.props = shallowReactive(props);
  } 
  
  // 新增部分的逻辑
  instance.attrs = attrs;
}

function setFullProps(instance, rawProps, props) {
  const [normalized, needCastKeys] = instance.propsOptions;
  
  if (rawProps) {
    for (const key in rawProps) {
      if (key === "ref" || key === "key")  continue;
      const value = rawProps[key];
      let camelKey;
      if (normalized && hasOwn(normalized, (camelKey = camelize(key)))) {
        props[camelKey] = value;
      } 
      // 新增部分的逻辑
      else if (!isEmitListener(instance.emitsOptions, key)) {
        if (!(key in attrs) || attrs[camelKey] !== value) {
          attrs[camelKey] = value;
        }
      }
    }
  }

  if (needCastKeys) { }
}
```

在 `setFullProps` 中，我们新增了一个 `else if `分支，在这个分支中会判断属性是否是 emit 选项中定义的，如果不是则将其添加到 attrs 对象中。这里的 emit 选项跟组件中的事件绑定和事件触发有关，这部分会在下一节介绍，这里先简单做个了解。

以上便是 `attrs` 在源码中的实现，接下去我们通过断点来调试这个过程。

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        .red {
            color: red;
        }

        .green {
            color: green;
        }
    </style>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            createApp,
        } = Vue;

        const Child = {
            name: "Child",
            props: {
                'msg': {
                    type: String
                },
            },
            setup(props, { attrs }) {
                console.log(props);
            }
        }

        const App = {
            name: "App",
            components: {
                Child
            },
            setup() {
                return {
                    msg: 'test123',
                }
            },
            template: `<Child id="div" class="red green" :msg="msg"/>`
        };

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

在这个例子中，父组件 `App`像子组件`Child`传递了 `id`、`class`以及 `msg`，子组件`Child` 接受了 `msg` 这个 props 属性，并且在 `setup` 中打印了`props` 和 `attrs`。

我们将断点打到 `setFullProps` 中：

![image-20231225145513756](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225145513756.png)

此时传入的 `rawProps` 中有三个属性，对应了我们的例子。

![image-20231225145627006](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225145627006.png)

第一个属性 `id`没有定义在 props 选项中，且不是事件属性，所以会添加到 `attrs` 中。

![image-20231225145754199](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225145754199.png)

第二个 `class` 属性也是同理。

![image-20231225145853431](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225145853431.png)

第三个 `msg` 属性定义在 props 选项中，因此会存到 `props` 中。

我们放开断点可以看到页面上打印的数据。

![image-20231225145954323](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225145954323.png)