# vue3核心原理（九）——  组件的事件处理机制

### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



## 一、组件的事件处理机制

在 Vue 3 中，自定义事件是组件之间通信的一种方式。子组件可以通过 `$emit` 方法触发事件，父组件可以通过 `v-on` 指令或 `@` 符号监听这些事件。

以下是在 Vue 3 中使用自定义事件的一个例子：

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            h,
            createApp
        } = Vue;

        const Child = {
            name: "Child",
            template: '<button @click="say">按钮</button>',
            setup(props, {
                emit
            }) {
                const say = (e) => {
                    emit('sayParent');
                };
                return {
                    say
                };
            }
        }

        const App = {
            name: "App",
            components: {
                Child
            },
            template: '<Child @sayParent="sayParent"/>',
            setup() {
                const sayParent = () => {
                    console.log('I am parent');
                };
                return {
                    sayParent
                };
            }
        };
        createApp(App).mount('#app')
    </script>
</body>

</html>
```

在这个例子中，父组件 `App` 在子组件上监听了 `sayParent` 事件，子组件 `Child` 内部有一个按钮元素，点击这个按钮元素会触发 `say` 事件，`say`事件内部通过 `emit` 方法触发父组件监听的 `sayParent` 事件，打印 `I am parent`。

![image-20231225165625178](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231225165625178.png)

下面我们看一下源码如何处理组件的自定义事件。

通过上一章节 `props`的学习，我们知道在组件上定义的属性都会被解析成 `props` 对象，自定义事件也不例外，当然这个 `props` 并不是组件实例上的 `props`，而是 `VNode` 上的 `props`。

![image-20231226164237236](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226164237236.png)

既然我们已经知道了自定义事件的存储位置，那只需要 `emit` 的时候找到对应的事件然后执行就可以了。

```typescript
// packages/runtime-core/src/componentEmits.ts
/**
 * @author: Zhouqi
 * @description: 事件触发函数
 * @param instance 组件实例
 * @param event 事件名
 */
export function emit(instance, event, ...rawArg) {
  // 这里需要从instance的vnode上获取事件，在instance上的是找不到注册的事件，因为在initProps的时候
  // instance上面的props对象中的数据必须是通过props选项定义过的
  const { props } = instance.vnode;

  // 校验参数合法性
  const { emitsOptions } = instance;
  if (emitsOptions) {
    const validator = emitsOptions[event];
    if (validator) {
      const isValid = validator(...rawArg);
      if (!isValid) {
        console.warn("参数不合法");
      }
    }
  }

  /**
   * 针对两种事件名做处理
   * add-name 烤肉串命名
   * addName 驼峰命名
   * 如果是烤肉串命名，先转换为驼峰命名，再转化为AddName这种名称类型
   */
  const handler =
    props[toHandlerKey(event)] || props[toHandlerKey(camelize(event))];

  if (handler) {
    handler(...rawArg);
  }
}
```

`emit` 的实现逻辑：

1. 获取组件 `VNode` 上的 `props`对象。
2. 获取组件实例上的 emit 选项，对于 emit 选项这块我们下面会分析。
3. 根据 emit 选项中的校验函数校验传入的参数是否合法，我们在调用 `emit`的时候是可以传递参数过去的 `emit(自定义事件名称, 参数)`，这个参数会被父组件定义的事件方法所接收。
4. 从 `props` 对象中获取对应的事件，这里会对事件名做统一转换处理。
5. 执行事件函数。

我们通过断点来调试一下触发事件的过程，这里直接把断点打到 `emit` 方法内部即可，然后点击一下页面中的按钮进入断点。

![image-20231226165605310](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226165605310.png)

此时断点已经正常进入。

![image-20231226165652410](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226165652410.png)

继续往下走，我们可以看到 `props`和 `emitsOptions` 的值，由于我们没有在子组件上定义 emit 选项，因此不会进入 emit 选项的参数校验逻辑。

![image-20231226165848255](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226165848255.png)

接下去会根据事件名 `event`从 `props` 中获取对应的事件函数。

![image-20231226170111614](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226170111614.png)

最后执行父组件定义的事件函数并获取子组件传递过来的参数 `click`，这就是 `emit` 触发事件的过程，下面我们看一下 emit 选项的功能。



## 二、 emit 选项

首先我们改一下例子：

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            h,
            createApp
        } = Vue;

        const Child = {
            name: "Child",
            template: '<button @click="say">按钮</button>',
            setup(props, {
                emit
            }) {
                const say = (e) => {
                    emit('click', 'click');
                };
                return {
                    say
                };
            }
        }

        const App = {
            name: "App",
            components: {
                Child
            },
            template: '<Child @click="sayParent"/>', // 修改部分
            setup() {
                const sayParent = (params) => {
                    console.log('I am parent', params);
                };
                return {
                    sayParent
                };
            }
        };
        createApp(App).mount('#app')
    </script>
</body>

</html>
```

在这个例子中，我们把先前绑定的自定义事件 `sayParent` 改成原生的 `click` 事件。

我们试着在页面上点击一次按钮：

![image-20231226171006774](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226171006774.png)

可以看到页面打印了两次，这两次打印的内容中传递过来的参数是不一样的，一个是子组件 `emit` 的时候传递过去的，另一个看着像原生事件参数对象 `event`。这是怎么回事？

为了确定第二次事件触发的源头，我们把断点打到最终执行的方法 `sayParent` 中并点击按钮。

![image-20231226171922189](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226171922189.png)

第一次进入断点的时候，参数是 `click`，可以确定是子组件 `emit` 触发的。

![image-20231226172023031](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226172023031.png)

![image-20231226173926392](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231226173926392.png)

第二次进入断点的时候，我们发现调用栈中，底层触发的源头在我们之间讲的普通元素的事件机制的处理逻辑中，并且在事件参数中可以看到事件的绑定元素就是 `button`。也就是说我们在 `Child` 组件上绑定的原生点击事件不仅在组件自身绑定了一次，同时也在组件的子元素上绑定了一次。

这显然不是我们想要的结果，我们的本意是给组件本身绑定一个自定义点击事件，点击时也只需要触发一次即可。下面需要通过断点调试来分析为什么会多绑定有一次。

在之前的章节中，我们知道普通元素的属性、事件绑定是在 `mountElement`中的。

```typescript
// packages/runtime-core/src/renderer.ts
const mountElement = (vnode, container, anchor, parentComponent, optimized) => {
    const { type, props, children, shapeFlag, transition, dirs } = vnode;
    const el = (vnode.el = hostCreateElement(type));
    if (shapeFlag & ShapeFlags.TEXT_CHILDREN) {
      hostSetElementText(el, children);
    } else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN) {
      mountChildren(children, el, null, parentComponent, optimized);
    }

    if (props) {
      // 处理props
      for (const key in props) {
        hostPatchProp(el, key, null, props[key]);
      }
    }
  };
```

从这里我们可以知道既然 `button`元素绑定上了 `click` 事件，那说明 `button` 元素对应的 `VNode`上的 `props` 内部肯定有这个 `click` 事件。

![image-20240103153116787](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240103153116787.png)

通过断点我们发现这个 `button` 元素身上确实有 `click` 事件，并且有两个。第一个事件是子组件内部给 `button` 绑定的点击事件，第二个事件显然是父组件给子组件绑定的点击事件，但是这个事件同时也被子元素 `button` 继承。

------

**继承：**

**在 Vue 3 中，`attrs` 继承是指父组件向子组件传递属性（attribute）的过程。这些属性不包括 `props`，也就是说，这些属性是没有在子组件中明确定义的。默认情况下，这些属性会被添加到子组件的根元素上。**

这里的 `attrs`在上节有介绍过，它包含了传递给一个组件的 attribute，但不包括那些已经在组件的 `props` 选项中明确定义的 attribute。显然子组件 `props` 选项上并没有明确定义 `click` 这个属性，并且它还是个事件，不能用`props`选项来定义，这里我们就需要用到 `emit`选项这个功能。

------

**emit 选项：**

**在 Vue 3 中，`emit` 选项用于定义组件可以触发的事件。这是一个新的选项，它在 Vue 2 中并不存在。`emit` 选项接收一个数组或对象，用于定义组件可以触发的事件名。如果使用对象形式，还可以为每个事件提供一个验证函数。**

例如：

```typescript
export default {
  emits: ['update', 'close'],
}
```

在这个例子中，组件可以触发 `update` 和 `close` 事件。

如果你想要为事件提供验证函数，你可以这样做：

```typescript
export default {
  emits: {
    update: null,
    close: payload => {
      if (typeof payload === 'string') {
        return true;
      } else {
        console.warn(`close event's payload should be a string`);
        return false;
      }
    },
  },
}
```

在这个例子中，`close` 事件的 payload 必须是一个字符串，否则会在控制台输出警告。

使用 `emit` 选项可以让你的组件更加清晰和易于理解，因为它明确地定义了组件可以触发的事件。

emit 选项的原理：

```typescript
// packages/runtime-core/src/component.ts
/**
 * @author: Zhouqi
 * @description: 创建组件实例
 * @param vnode 虚拟节点
 * @param parent 父组件实例
 */
export function createComponentInstance(vnode, parent) {
  const type = vnode.type;

  const componentInstance = {
    // 省略其他属性
    propsOptions: normalizePropsOptions(type),
    emitsOptions: normalizeEmitsOptions(type), // 新增代码
  };

  // 在_属性中存储组件实例对象
  componentInstance.ctx = { _: componentInstance };
  componentInstance.emit = emit.bind(null, componentInstance) as any;
  return componentInstance;
}
```

首先在 `createComponentInstance` 中不仅会初始化 `props` 选项，还会初始化 `emit` 选项。

```typescript
// packages/runtime-core/src/componentEmits.ts
/**
 * @author: Zhouqi
 * @description: 解析emits选项
 * @param component 组件实例
 * @return 解析后的结果
 */
export function normalizeEmitsOptions(component) {
  // TODO 缓存emits的解析结果，如果组件的emits被解析过了，就不再次解析
  const emits = component.emits;
  const normalizeResult = {};
  if (isArray(emits)) {
    // 数组形式直接遍历赋值为null
    emits.forEach((key) => {
      normalizeResult[key] = null;
    });
  } else {
    // 如果是对象，则可能有校验函数，直接合并到结果中即可
    extend(normalizeResult, emits);
  }
  return normalizeResult;
}
```

在`normalizeEmitsOptions`方法中会判断 `emit` 选项的类型，如果是数组则会把数组中的值当作 key  存储到 `normalizeResult`对象中，如果是对象则直接通过 `Object.assign` 进行合并。

接下去我们需要对 `attrs` 进行处理：

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
      if (key === "ref" || key === "key")  continue;
      const value = rawProps[key];
      let camelKey;
      if (normalized && hasOwn(normalized, (camelKey = camelize(key)))) {
        props[camelKey] = value;
      }
      // 新增代码
      else if (!isEmitListener(instance.emitsOptions, key)) {
        // 处理attrs
        if (!(key in attrs) || attrs[camelKey] !== value) {
          attrs[camelKey] = value;
        }
      }
    }
  }

  // 特殊处理
  if (needCastKeys) {
    // 省略代码
  }
}
```

在上节介绍的 `setFullProps`方法中我们增加一个 `else if` 分支，当 `props`中的属性在 `emit` 选项中有定义时，就无需将其放到 `attrs` 上，这样子元素就不会继承对应的属性。

我们在前面的例子中加上 `emit`选项：

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            h,
            createApp
        } = Vue;

        const Child = {
            name: "Child",
            emits: ['click'], // 新增代码
            template: '<button @click="say">按钮</button>',
            setup(props, {
                emit
            }) {
                const say = (e) => {
                    emit('click', 'click');
                };
                return {
                    say
                };
            }
        }

        const App = {
            name: "App",
            components: {
                Child
            },
            template: '<Child @click="sayParent"/>', // 修改部分
            setup() {
                const sayParent = (params) => {
                    console.log('I am parent', params);
                };
                return {
                    sayParent
                };
            }
        };
        createApp(App).mount('#app')
    </script>
</body>

</html>
```

接着在前面提到的方法中打上断点：

![image-20240103163312385](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240103163312385.png)

首先是 `emit`选项的处理，这里的 `emits`是个数组，因此会进入数组的处理逻辑并得到结果。

![image-20240103163440443](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240103163440443.png)

![image-20240103163601242](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240103163601242.png)

接着在 `setFullProps`中会判断 key 是不是在 `emit `选项中有定义，有的话就不会添加到`attrs` 上。

![image-20240103163803035](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240103163803035.png)

![image-20240103164236357](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240103164236357.png)

最后我们可以看到 `button`元素的 `props`上只有子组件内部给元素绑定的点击事件了，这就是 `emit`选项的作用。



## 三、继承的原理

上面这个问题还有另外一种解决方式，那就是不让子元素继承属性，这就涉及到继承的原理：

```typescript
/**
 * @author: Zhouqi
 * @description: 生成组件的vnode
 * @param instance 组件实例
 */
export function renderComponentRoot(instance) {
  const {
    attrs,
    props,
    render,
    proxy,
    vnode,
    inheritAttrs,
    slots,
  } = instance;

  let fallthroughAttrs;
  let result;

  if (vnode.shapeFlag & ShapeFlags.STATEFUL_COMPONENT) {
    result = normalizeVNode(render.call(proxy, proxy));
    fallthroughAttrs = attrs;
  } 

  // 新增代码
  // attrs存在且可以继承attrs属性的情况下
  if (fallthroughAttrs && inheritAttrs !== false) {
    const attrsKeys = Object.keys(fallthroughAttrs);
    const { shapeFlag } = result;
    if (
      attrsKeys.length &&
      shapeFlag & (ShapeFlags.ELEMENT | ShapeFlags.COMPONENT)
    ) {
      result = cloneVNode(result, fallthroughAttrs);
    }
  }

  return result;
}
```

在之前介绍的 `renderComponentRoot`方法中，我们只提到了调用 `render`函数生成 `VNode` 这一步。当 `VNode` 生成完毕后，还会对 `VNode` 上属性做进一步处理，其中一步就是属性继承，将 `attrs` 继承到第一个子元素的 `VNode`中。

在新增的逻辑会判断是否有需要继承的属性以及是否允许继承（默认允许继承），如果 `inheritAttrs`是 `false`，第一个子元素同样也不会带上 `attrs` 中的属性。在允许继承的条件会调用 `cloneVNode`进行属性合并。

```typescript
// packages/runtime-core/src/vnode.ts
/**
 * @author: Zhouqi
 * @description: 克隆vnode
 * @param  vnode 虚拟节点
 * @param  extraProps 额外的属性
 */
export function cloneVNode(vnode, extraProps?) {
  // vnode克隆处理
  const { props } = vnode;
  const mergedProps = extraProps ? mergeProps(props || {}, extraProps) : props;
  // 简单处理一下
  vnode = extend(vnode, { props: mergedProps });
  return vnode;
}

/**
 * @author: Zhouqi
 * @description: 合并props属性
 * @param  args 需要合并的props对象数组
 */
export function mergeProps(...args) {
  let result: Record<string, any> = {};
  const argLength = args.length;
  for (let i = 0; i < argLength; i++) {
    const arg = args[i];
    for (const key in arg) {
      const value = arg[key];
      if (key === "class") {
        // 标准化class
        result.class = normalizeClass(value);
      } else if (isOn(key)) {
        // 处理事件相关属性
        const exist = result[key];
        if (
          value &&
          value !== exist &&
          !(isArray(exist) && exist.includes(value))
        ) {
          // 如果新的事件存在且和旧的事件不相同，并且旧的事件集合里面没有新的事件，则合并新旧事件
          result[key] = exist ? [].concat(exist, value) : value;
        }
      } else {
        result[key] = value;
      }
    }
  }
  return result;
}
```

可以看到 `cloneVNode` 中会调用 `mergeProps`将 `attrs`中的属性合并到 `props`中。

对于之前的例子，我们也可以使用 `inheritAttrs: false` 来了解决：

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="app"></div>
    <script src="../../dist/simplify-vue.global.js"></script>
    <script>
        const {
            h,
            createApp
        } = Vue;

        const Child = {
            name: "Child",
            template: '<button @click="say">按钮</button>',
            inheritAttrs: false,
            setup(props, {
                emit
            }) {
                const say = (e) => {
                    emit('click', 'click');
                };
                return {
                    say
                };
            }
        }

        const App = {
            name: "App",
            components: {
                Child
            },
            template: '<Child @click="sayParent"/>',
            setup() {
                const sayParent = (params) => {
                    console.log('I am parent', params);
                };
                return {
                    sayParent
                };
            }
        };
        createApp(App).mount('#app')
    </script>
</body>

</html>
```

![image-20240103165444506](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240103165444506.png)