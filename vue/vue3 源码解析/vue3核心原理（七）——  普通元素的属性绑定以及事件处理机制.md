# vue3核心原理（七）——  普通元素的属性绑定以及事件处理机制

### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



在平时写代码的时候，如果要给元素添加属性，我们会像写 html 一样直接在元素上加属性。

```html
<div class="div" id="id">Hello World</div>
```

这种方式在 `vue` 中也同样生效，甚至在 `vue`  中还支持动态绑定。

```vue
<div class="div" id="id" :style={color:fontColor}>Hello World</div>
```

通过 `v-bind` 或者 `:` 的形式可以进行属性绑定，传入动态变化的值，上面这个例子中的 `fontColor` 就是一个动态的值。当然，跟写原生 html 不同的是，`vue ` 有一套自身的属性解析规则，在解析模版的时候会转换成 `render` 函数，元素上的属性会被解析成 `key: value` 的形式传入到创建虚拟节点的函数中。

![image-20231218154441782](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231218154441782.png)

我们以下面 demo 为例来看 `vue` 是如何处理普通元素属性的。

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
            createApp
        } = Vue;

        const App = {
            name: "App",
            template: '<div :class="fontColor" id="id">Hello World</div>',
            setup() {
                return {
                    fontColor: 'red'
                }
            }
        }

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

在前面的章节中，我们提到过，普通元素的处理是在 `processElement` 这个方法中。

```typescript
const processElement = (n1, n2, container, anchor, parentComponent, optimized) => {
    // 旧的虚拟节点不存在，说明是初始化渲染
    if (n1 === null) {
      mountElement(n2, container, anchor, parentComponent, optimized);
    } else {
      // 更新
      patchElement(n1, n2, parentComponent, optimized);
    }
  };
```

初始挂载会走 `mountElement` 这个方法，我们将断点打到 `mountElement`以及 `renderComponentRoot` 方法中，在 `renderComponentRoot` 中主要看生成的 Vnode。

![image-20231218163459453](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231218163459453.png)

启动断点调试，我们可以看到断点正常停在了 `renderComponentRoot` 方法内部，这里就是根组件内部执行`render`函数生成子虚拟节点的地方。我们执行单步调试看看这个 `render` 函数

![image-20231218163759944](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231218163759944.png)

在 `render` 函数中我们可以看到 class 和 id 已经作为对象属性传入。其中 id 属性的值就是 id，class  的值由于是动态的，所以会使用 `_ctx.fontColor`， `_ctx`就是组件内部数据访问的代理对象，代理对象的创建在前面初始化渲染章节中有提及。

我们继续单步调试，看看一下 `_ctx.fontColor` 的访问过程。

![image-20231218164723720](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231218164723720.png)

断点进入到 `PublicInstanceProxyHandlers` 的 `get` 方法中

```typescript
// packages/runtime-core/src/componentPublicInstance.ts

// 建立map映射对应vnode上的属性，利于扩展
const publicPropertiesMap = {
  $el: (i) => i.vnode.el,
  $slots: (i) => shallowReadonly(i.slots),
  $props: (i) => shallowReadonly(i.props),
  $attrs: (i) => shallowReadonly(i.attrs),
  $data: (i) => i.data,
};

export const PublicInstanceProxyHandlers = {
  get({ _: instance }, key) {
    const { setupState, props, accessCache, data, propsOptions } = instance;
    /**
     * 这里每次都会通过hasOwn去判断属性是不是存在于某一个对象中，这样是不如直接访问对应对象属性来的更快
     * 为了能够直接访问到指定对象上的属性，需要建立一个映射表，在第一次读取值的时候将属性和对象之间进行关联
     * 比如第一次读取属性值时，通过hasOwn发现是setupState上的属性，那就标记该属性是SETUP，下一次访问
     * 该属性时，判断到标记为SETUP，则直接从setupState获取值
     */
    if (key[0] !== "$") {
      const t = accessCache[key];
      if (t !== undefined) {
        switch (t) {
          case AccessTypes.SETUP:
            return setupState[key];
          case AccessTypes.DATA:
            return data[key];
          case AccessTypes.PROPS:
            return props[key];
        }
      }
      if (setupState !== EMPTY_OBJ && hasOwn(setupState, key)) {
        accessCache[key] = AccessTypes.SETUP;
        return setupState[key];
      } else if (data !== EMPTY_OBJ && hasOwn(data, key)) {
        accessCache[key] = AccessTypes.DATA;
        return data[key];
      } else if (hasOwn(propsOptions[0], key)) {
        accessCache[key] = AccessTypes.PROPS;
        return props[key];
      }
    }

    // 属性映射表上有对应的属性则返回对应的属性值
    const publicGetter = publicPropertiesMap[key];
    if (publicGetter) {
      return publicGetter(instance);
    }

    console.warn(
      `Property ${JSON.stringify(key)} was accessed during render ` +
        `but is not defined on instance.`
    );
  }
};
```

在访问器方法中，首先会判断访问的 key 是不是 $ 开头的，以 $ 开头的属性定义为内部属性。对于内部属性的访问会通过 `publicPropertiesMap` 找到对应的映射函数并执行拿到结果，比如获取 `$el`、`$slots`、`$props`、`$attrs`、`$data` 等等。**作为开发者我们需要注意千万别用 $ 开头的变量作为属性的 key，$ 是作为内部属性的一个标识**。

对于非 $ 开头的属性，会先从 `accessCache[key]` 中获取属性对应的类型，这个类型指的是数据是定义在哪部分的。比如， `setup` 方法返回的数据对象是属于 `setup`里面的，组件配置中的 `data(){ return {} }`中的数据是属于 `data` 里面的，`props` 中的数据是属于 `props` 里面的，在进行数据访问的时候需要在对应的数据对象中取值。

第一次访问数据的时候其实并不知道要访问的 key 是存在哪部分数据当中的，因此会按照查找优先级去查找数据。根据 if 逻辑的优先级可以看到数据的访问顺序是：setupState > data > props，假如在 setup、 data 、 props 中同时存在某个 key，则优先会获取 setup 中定义的数据，其次是 data，最后是 props 。第一次访问数据后，会将数据存在的位置缓存到 `accessCache` 中。

这么设计的好处在于，二次访问的时候可以直接定位到数据的位置，不需要再走 if 逻辑以及 hasOwn 的判断，提升数据访问效率。 

在我们的例子中，fontColor 是定义在 setup 中的，因此会从 `setupState` 中取值。

![image-20231218172436549](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231218172436549.png)

在访问到 fontColor 的值之后还需要经过 `_normalizeClass` 的处理。

![image-20231218173009811](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231218173009811.png)

```typescript
// packages/shared/src/normalizeProp.ts
export function normalizeClass(value) {
  // 处理的情况无非就是三种：字符串，数组，对象
  let result = "";
  if (isString(value)) {
    // 是字符串则直接拼接
    result += value;
  } else if (isArray(value)) {
    // 是数组情况就递归调用normalizeClass
    for (let i = 0; i < value.length; i++) {
      result += `${normalizeClass(value[i])} `;
    }
  } else if (isObject(value)) {
    for (const key in value) {
      // 值为true的class才需要拼接
      if (value[key]) {
        result += `${key} `;
      }
    }
  }
  return result.trim();
}
```

在实际开发中，动态绑定的 class 的值有多种。第一种就是最简单的，值直接是字符串。

```typescript
class: 'red'
```

第二种，值是一个数组，用于绑定多个类名。在 `normalizeClass` 中会通过循环数组的方式进行拼接处理。

```typescript
class: ['red green'],
```

第三种，值是一个对象。在 `normalizeClass` 中会遍历对象，只有 value 为真的 key 才会拼接到元素类中。

```typescript
class: {
   red: false, // 不会添加 red 这个类
   green: true // 会添加 green 这个类
},
```

属性值处理完成后会作为 props 绑定到 Vnode 对象中。

接下去继续断点到 `mountElement` 这个方法中，之前这个方法中没有写 props 的处理，这里我们加上。

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

    // 新增的代码
    if (props) {
      // 处理props
      for (const key in props) {
        hostPatchProp(el, key, null, props[key]);
      }
    }
  };
```

如果 props 存在，则会循环其每一个属性和值，调用 `hostPatchProp` 与元素进行绑定。在看 `hostPatchProp` 这个方法之前我们需要了解 HTML Attribute 和 DOM property 的关系。

------

**HTML Attribute 是 HTML 元素中的一部分，它为元素提供了额外的信息。这些信息可以用于控制元素的行为或提供元素的元数据。HTML Attribute 总是在 HTML 元素的开始标签中定义，并且它们总是以名称/值对的形式出现**。

例如，在以下的 HTML 元素中，`src`、`alt` 和 `width` 就是 HTML Attributes：

```html
<img src="image.jpg" alt="My Image" width="500">
```

在这个例子中，`src` 属性告诉浏览器图片的位置，`alt` 属性提供了图片无法显示时的替代文本，`width` 属性定义了图片的宽度。

------

**DOM property 是指 JavaScript 可以访问和操作的 HTML 元素的属性。这些属性是 DOM（文档对象模型）中节点对象的属性。**

例如，如果你有一个 HTML 元素如下：

```html
<input id="myInput" value="Hello World">
```

你可以使用 JavaScript 来获取或修改这个元素的 `value` 属性，这个 `value` 就是一个 DOM property：

```javascript
let inputElement = document.getElementById('myInput');
console.log(inputElement.value); // 输出 "Hello World"

inputElement.value = "新的值";
console.log(inputElement.value); // 输出 "新的值"
```

在这个例子中，`value` 是 `inputElement` 对象的一个 DOM property，你可以通过 JavaScript 来读取或修改它。这是因为 `input` 元素在 DOM 中被表示为一个对象，而这个对象的属性就是 DOM property。

------

**HTML Attribute 和 DOM Property 都是用于描述 HTML 元素特性的方式，但它们之间存在一些关键区别。**

**HTML Attribute 是在 HTML 文档中定义的，它们为元素提供了初始值。当浏览器解析 HTML 文档并创建 DOM（文档对象模型）时，它会将这些属性转化为 DOM property。HTML Attribute 是在 HTML 源代码中设置的，只能在加载页面时读取一次。**

**DOM Property 则是在 DOM 中的 JavaScript 对象，它们可以动态改变。DOM Property 的值可以在运行时改变，而这些改变不会反映到 HTML Attribute 上。**

**例如，当你在 HTML 中设置一个 input 元素的 value 属性时，这将设置元素的默认值。但当用户在浏览器中输入新的值时，DOM 中的 value property 会改变，但 HTML attribute 保持不变。**

**总的来说，HTML Attribute 和 DOM Property 都是描述 HTML 元素特性的方式，但它们在何时以及如何使用上有所不同。**

------

在了解了 HTML Attribute 和 DOM Property 之后再来看 `patchProp` 这个方法会更加容易：

```typescript
// packages/runtime-dom/src/patchProp.ts
/**
 * @description: 处理props属性
 * @param el 元素
 * @param key 属性名
 */
export function patchProp(el, key, preValue, nextValue) {
  if (shouldSetAsProp(el, key, nextValue)) {
    patchDOMProp(el, key, nextValue);
  } else if (key === "class") {
    // class通过className去设置性能最好
    el.className = nextValue;
  } else {
    // 更新html属性
    patchAttr(el, key, nextValue);
  }
}

// 是否可以直接设置DOM property
function shouldSetAsProp(el, key, value) {
  // form属性是只读的，只能通过setAttribute设置属性
  if (key === "form") {
    return false;
  }
  return key in el;
}

// packages/runtime-dom/src/modules/props.ts
/**
 * @description: 更新dom属性
 * @param el dom元素
 * @param key 属性
 * @param value 值
 */
export function patchDOMProp(el, key, value) {
  // 省略部分代码
  el[key] = value;
}

// packages/runtime-dom/src/modules/attrs.ts
export function patchAttr(el, key, value) {
  // 新的值不存在，则表示删除属性
  if (value == null) {
    el.removeAttribute(key);
  } else {
    el.setAttribute(key, value);
  }
}
```

方法内部首先会调用 `shouldSetAsProp` 判断属性是否可以通过 DOM Property 的方式设置，如果可以则会调用 `patchDOMProp` 更新属性值；如果属性是 class，则会使用 `el.className` 的方式更新值；否则属性会作为 HTML Attribute 并通过 `setAttribute` 和 `removeAttribute` 更新属性值。

下面是断点调试过程：

首先进入到循环之前可以看到 props 上有 class 和 id 两个属性：

![image-20231219161054748](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231219161054748.png)

第一个进入循环的是 class 属性，直接通过 `el.className` 设置属性值：

![image-20231219161252377](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231219161252377.png)

第二个属性是 id，它是存在于 DOM 对象中的，因此会当作 DOM Property 去设置属性值：

![image-20231219161517338](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231219161517338.png)

更新完成后我们可以看到元素已经有了这两个属性，并且应用上了样式：

![image-20231219161726202](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231219161726202.png)

上面是简单的属性绑定过程，我们再加上事件的绑定处理：

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
            createApp
        } = Vue;

        const App = {
            name: "App",
            template: '<div :class="fontColor" id="id" @click="log">Hello World</div>',
            setup() {
                const log = () =>{
                  console.log('log');
                };
                return {
                    fontColor: 'red',
                  	log
                }
            }
        }

        createApp(App).mount('#app')
    </script>
</body>

</html>
```

在这里例子中，我们通过 `@` 指令绑定了一个 click 事件，事件函数是 `log`，点击的时候可以打印 log。

我们将断点打到 `mountElement` 中：

![image-20231219170436826](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231219170436826.png)

可以看到这次多了一个 onClick 属性，值是函数，接下去会继续调用 `hostPatchProp` 方法对事件进行处理。之前讲解这个方法时没有提供对于事件属性的处理，我们在这里加上这段逻辑：

```typescript
// packages/runtime-dom/src/patchProp.ts
export function patchProp(el, key, preValue, nextValue) {
  if (shouldSetAsProp(el, key, nextValue)) {
    patchDOMProp(el, key, nextValue);
  } else if (key === "class") {
    // class通过className去设置性能最好
    el.className = nextValue;
  }
  // 新增部分代码
  else if (isOn(key)) {
    if (!isModelListener(key)) {
      // 注册事件
      patchEvent(el, key, preValue, nextValue);
    }
  } else {
    // 更新html属性
    patchAttr(el, key, nextValue);
  }
}

// packages/shared/src/index.ts
// 判断是否是事件属性：onClick …………
export const isOn = (key: string) => /^on[^a-z]/.test(key);
// 是否是v-model
export const isModelListener = (key: string) => key.startsWith("onUpdate:");
```

在条件分支中，我们加上了事件属性的处理，通过 `isOn` 这个方法判断是不是 on 开头的事件属性。紧接着还会判断是不是 `v-model` 绑定的事件，不是的话则会通过 `patchEvent` 进行事件绑定。

在平时开发的时候，假如是写原生 js，一般都会用到 `addEventListener` 和 `removeEventListener` 这两个 api 去注册和解绑事件。

注册事件：

```typescript
xxx.addEventListener('click',click);
```

解绑事件：

```typescript
xxx.removeEventListener('click',click);
```

更新事件：

```typescript
let flag = true;
if (flag) {
  xxx.removeEventListener('click',click);
  xxx.addEventListener('click',click1);
} else {
  xxx.removeEventListener('click',click1);
  xxx.addEventListener('click',click);
}

```

对于注册和解绑事件，如果只是单个事件，那确实很方便。但是，如果有多个事件则需要多次调用 api，这就变得有些繁琐。更新事件亦是如此，需要同时调用解绑和注册两个 api。不过对于事件更新我们也可以考虑在逻辑上进行处理，如下所示

```typescript
let flag = true;
const handlerClick = () => {
  if (flag) {
    click();
  } else {
    click1();
  }
}
xxx.addEventListener('click', handlerClick);
```

这种方式相当于构建了一个统一的代理函数，由这个代理函数去触发真正需要执行的方法，这样就不需要频繁地解绑和注册，唯一要做的就是维护好这个代理函数。这种方式也同样适合绑定/解绑多个事件的情况，我们顺着这个思路做一个整体的事件控制机制：

```typescript
let flag = true
const click = () => {
  console.log('click');
};
const click1 = () => {
  console.log('click1');
}
const click2 = () => {
  console.log('click2');
}
// 存储多个事件
const events = [click, flag ? click1 : click2];
const createInvoker = (events) => {
  const handler = () => {
    // 执行事件的时候只需要执行 value 值上的方法即可
    if(Array.isArray(handler.value)) {
      handler.value.forEach(evt => {
        evt();
      });
      return;
    }
    handler.value();
  };
  // 把事件函数存到入口事件对象的 value 属性中
  handler.value = events;
  return handler;
}
const patchEvent = (el, eventType, events) => {
  // 将事件关联关系存到 dom 对象中，每个 dom 都是唯一的
  const invokers = el._vos || (el._vos = {});
  let invoker = invokers[eventType];
  // 新的事件不存在了，说明是销毁事件
  if(!events) {
    // 移除入口事件
    el.removeEventListener(eventType, invoker);
    // 清空关联关系
    invoker[eventType] = null;
    return;
  }
  // 如果没有绑定过入口事件则需要初始化绑定一次
  if(!invoker) {
    invoker = (invokers[eventType] = createInvoker(events));
    el.addEventListener(eventType, invoker);
    return
  }
  // 更新的时候只需要更新 value 值即可，这个 value 值存储的就是所有需要执行的事件
  invoker.value = events;
}
patchEvent(document.body, 'click', events);
```

这其实就是 vue3 中事件绑定/更新的机制，也就是 `patchEvent` 的核心处理逻辑：

```typescript
// packages/runtime-dom/src/modules/events.ts
// props上的事件注册函数
export function patchEvent(el, key, preValue, nextValue) {
  /**
   * 这里创建一个伪造的事件代理函数invoker，将原始事件赋值到invoker的value属性上
   * 将invoker作为最终绑定的事件，在执行invoker函数时内部会执行原始的绑定事件，即执行invoker.value()
   *
   * 新建伪造的事件代理函数有几个作用：
   * 1、方便事件更新
   * 2、控制原始事件的执行（涉及到事件冒泡机制）
   * 3、…………?
   *
   * 由于原生事件类型有很多，为了不互相覆盖，这里需要建立一个map对象invokers，key指代事件类型，值是伪造的事件代理函数
   */
  const invokers = el._vei || (el._vei = {});
  const eventName = key.slice(2).toLowerCase();
  const hasInvoker = invokers[eventName];

  if (nextValue) {
    // 如果存在新的值且旧的事件代理函数存在，则表示更新事件，否则表示添加新的事件绑定
    if (hasInvoker) {
      /**
       * 1、方便事件更新
       * 在更新事件时，不需要销毁原来的事件，再绑定新的事件，而只要更新invoker.value属性即可
       */
      hasInvoker.value = nextValue;
    } else {
      const invoker = (invokers[eventName] = createInvoker(nextValue));
      addEventListener(el, eventName, invoker);
      invoker.attached = getNow();
    }
  } else if (hasInvoker) {
    // 新的值不存在且事件代理函数存在，则表示销毁事件绑定
    removeEventListener(el, eventName, hasInvoker);
    invokers[eventName] = undefined;
  }
}

// 创建事件代理函数
function createInvoker(events) {
  const invoker: any = (e) => {
    const timestamp = e.timeStamp;
    /**
     * 2、控制原始事件的执行（涉及到事件冒泡机制）
     * 假设父vnode上有onClick事件，事件值取决于一个响应式数据的值，比如：onClick: isTrue ? () => console.log(1) : null，
     * 子vnode上有一个绑定事件onClick: () => { isTrue = true }，当点击子vnode时会触发click事件，由于事件冒泡机制，click
     * 会向上冒泡到父节点，由于isTrue初始为false，因此父节点上不应该有绑定的click事件，但是却打印了1。
     * 这是由于vue的更新机制和事件冒泡时机导致的，实际上当isTrue被修改为true时触发了事件更新，更新后父节点上绑定了事件，之后事件才
     * 冒泡到父节点上，执行了父节点绑定的click事件。而解决方式就是在执行子元素事件的时候记录事件执行的时间，在这个时间点之后绑定的事件都
     * 不要去执行，这时候就需要有控制原始事件执行的功能。
     */
    // 事件冒泡时，e会往上传递其中s.timestamp就是事件最开始执行的事件
    if (timestamp < invoker.attached) return;
    // 如果events是一个数组，则循环执行
    isArray(invoker.value)
      ? invoker.value.forEach((fn) => fn(e))
      : invoker.value(e);
  };
  invoker.value = events;
  return invoker;
}
```

以上就是普通元素的属性、事件绑定流程。