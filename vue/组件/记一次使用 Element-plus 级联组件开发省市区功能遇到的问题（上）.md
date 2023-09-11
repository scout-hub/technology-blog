# 记一次使用 Element-plus 级联组件开发省市区功能遇到的问题（上）

事情的经过大概是这样的：身边的同事在开发一个新的平台，这个平台上的一个功能模块中的数据来自另一个软件，这份数据中带有地址相关的信息，需要回显并支持修改地址信息。

刚看到这儿不是不是觉得这个需求很简单？直接搬上`element-plus`中的级联组件，然后把地址数据传进去不就完事儿了。我一开始也是这么想的，直接上代码：

```vue
<template>
	<el-cascader :options="country" v-model="value" />
</template>
<script setup lang="ts">
const country = [{
  label: '浙江省',
  value: 'zjs',
  children: [
    {
      label: '杭州市',
      value: 'hzs',
      children: [{
        label: '西湖区',
        value: 'xhq',
      }]
    },
    {
      label: '宁波市',
      value: 'nbs',
      children: [{
        label: '鄞州区',
        value: 'yzq',
      }]
    }
  ]
}];
const value = ['zjs', 'hzs', 'xhq'];
</script>
```

在上面的代码示例中我们自己模拟省市区的数据，真实数据来自接口，我们看一下效果。

![image-20230911150115657](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911150115657.png)

可以看到，数据已经正常回显上去了，并且也支持修改数据。

这不就已经解决问题了么？实际不然。在这个业务需求中，省市区字段需要的是一个中文值，而不是我们示例中的`['zjs', 'hzs', 'xhq']`。不过这也不是什么难题，`element-plus`的级联组件支持自定义文本字段名和值字段名，就像这样：

```vue
<template>
	<el-cascader :props="props" :options="country" v-model="value" />
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'
import { ref, watch } from 'vue';

const props: CascaderProps = {
  value: 'label',
  label: 'label',
}

const country = [{
  label: '浙江省',
  value: 'zjs',
  children: [
    {
      label: '杭州市',
      value: 'hzs',
      children: [{
        label: '西湖区',
        value: 'xhq',
      }]
    },
    {
      label: '宁波市',
      value: 'nbs',
      children: [{
        label: '鄞州区',
        value: 'yzq',
      }]
    }
  ]
}];

const value = ref(['浙江省', '杭州市', '西湖区']);

watch(value, (val) => {
  console.log(val);
});
</script>
```

级联组件支持传入一个`props`对象属性，在这个对象中我们可以传入`value`和`label`属性，指定为数据字段中对应的字段名。通过这种方式，我们依旧可以实现这个需求。

![image-20230911151421572](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911151421572.png)

![image-20230911151449133](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911151449133.png)

数据修改后，对应的值也变成了中文。

这就完了吗？没完。想必大家都知道直辖市，像北京市这种，对应的省和市其实是一个地方。在后端的接口数据中，省对应的北京市和市对应的北京市的文本名是一样的，值字段不一样。附上代码：

```vue
<template>
	<el-cascader :props="props" :options="country" v-model="value" />
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'
import { ref } from 'vue';

const props: CascaderProps = {
  value: 'label',
  label: 'label',
}

const country = [{
  label: '浙江省',
  value: 'zjs',
  children: [
    {
      label: '杭州市',
      value: 'hzs',
      children: [{
        label: '西湖区',
        value: 'xhq',
      }]
    }
  ]
},
{
  label: '北京市',
  value: 'bj',
  children: [
    {
      label: '北京市',
      value: 'bjs',
      children: [{
        label: '东城区',
        value: 'dcq',
      }]
    }
  ]
}];

const value = ref([]);
// const value = ref(['北京市', '北京市', '东城区']);
</script>
```

![image-20230911155238280](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911155238280.png)

像这种我们也是可以正常选择地址并且也能正常回显。

接下去就是问题的出现的时候了。在正常开发中，如果这种树形数据量很大，我们一般是通过懒加载的方式去实现的，而不是一开始就返回整棵树的数据，这属于性能层面的优化。

```vue
<template>
	 <el-cascader :props="props" v-model="value" />
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'
import { ref } from 'vue';

const props: CascaderProps = {
  value: 'label',
  label: 'label',
  lazy: true,
  lazyLoad(node, resolve) {
    const { level } = node;
    setTimeout(() => {
      if (level === 0) {
        resolve(province);
        return;
      }
      const value = node.data?.value;
      if (level === 1) {
        resolve(city.filter(item => item.pid === value));
        return;
      }
      resolve(area.filter(item => item.pid === value));
    }, 500)
  }
}

const province = [{
  label: '浙江省',
  value: 'zjs',
}, {
  label: '北京市',
  value: 'bj',
}];

const city = [{
  label: '北京市',
  value: 'bjs',
  pid: 'bj',
}, {
  label: '杭州市',
  value: 'hzs',
  pid: 'zjs',
}];

const area = [{
  label: '东城区',
  value: 'dcq',
  pid: 'bjs',
  leaf: true
}, {
  label: '西湖区',
  value: 'xhq',
  pid: 'hzs',
  leaf: true
}]

const value = ref([]);
// const value = ref(['北京市', '北京市', '东城区']);
</script>
```

这里我们将代码改成了懒加载的方式，先来看看选择地址的效果。

![2023-8-11-a](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/2023-8-11-a.gif)

可以看到地址还是可以正常选择的，但是地址回显就出现了问题，我们将注释掉的`const value = ref(['北京市', '北京市', '东城区'])`这段代码放出来看看效果。

![2023-8-11-b](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/2023-8-11-b.gif)

我们可以看到，北京市的下一级数据一直处于加载中，我们在定时器中设置的延时是`500ms`，显然这块出现了问题。

我的初步怀疑是前后两个北京市的值一样导致懒加载出现了问题，但是下面这个现象又让我不得不怀疑我的怀疑。我尝试把数据的层级调到 2 级：

```vue
<template>
	 <el-cascader :props="props" v-model="value" />
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'
import { ref } from 'vue';

const props: CascaderProps = {
  value: 'label',
  label: 'label',
  lazy: true,
  lazyLoad(node, resolve) {
    const { level } = node;
    setTimeout(() => {
      if (level === 0) {
        resolve(province);
        return;
      }
      const value = node.data?.value;
      if (level === 1) {
        resolve(city.filter(item => item.pid === value));
        return;
      }
      resolve(area.filter(item => item.pid === value));
    }, 500)
  }
}

const province = [{
  label: '浙江省',
  value: 'zjs',
}, {
  label: '北京市',
  value: 'bj',
}];

const city = [{
  label: '北京市',
  value: 'bjs',
  pid: 'bj',
  leaf: true
}, {
  label: '杭州市',
  value: 'hzs',
  pid: 'zjs',
}];

const area = [{
  label: '东城区',
  value: 'dcq',
  pid: 'bjs',
  leaf: true
}, {
  label: '西湖区',
  value: 'xhq',
  pid: 'hzs',
  leaf: true
}]

const value = ref(['北京市', '北京市']);
</script>
```

再来看看效果。

![2023-8-11-c](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/2023-8-11-c.gif)

这次居然发现数据回显了，真是离了大谱。显然，现在已经不能凭空猜测问题了，要想真正知道原因就必须要去调试源码。这部分放到下一章节，我们先来看看如果解决这个问题。其实这个问题我们可以猜到是值重复导致的，也就是两个`北京市`的问题，如果值唯一，这个问题是不存在的。

解决办法也有很多，最简单的就是用唯一值作为组件的`value`，也就是一开始的例子。数据的`value`字段是唯一的，`label`字段会有重复，那我们就定义`value`字段作为值字段就没问题了。但是前面也提到了，在另一个软件中的数据用的是`label`值，并且已经有大量用户数据都是这样，似乎用`label`作为值字段已经不可避免了。

既然只能用`label`字段，那我们只需要保证`label`字段值也是唯一即可。要保证数据唯一，无非就是两种方式。第一种就是修改源数据，后端支持。第二种方式，数据拿到后前端自己修改数据。最合理的还是修改源数据，毕竟数据源改了，其它地方就都统一了，如果让前端去修改数据，那修改的地方可能会很多。

```vue
<script setup lang="ts">
const province = [{
  label: '浙江省',
  value: 'zjs',
}, {
//  label: '北京市',
  label: '北京',
  value: 'bj',
}];
</script>
```

![2023-8-11-d](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/2023-8-11-d.gif)

我们将`province`中的北京市改成了北京后可以看到数据正常回显了。

但是这是在我们知道会有两个北京市的前提下去操作的，假设场景再普通一点，纯粹就是一个树形结构的渲染，并且不知道下一级数据是否跟父级有重复的值。如下所示：

```vue
<template>
	 <el-cascader :props="props" v-model="value" />
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'
import { ref } from 'vue';

const props: CascaderProps = {
  value: 'value',
  label: 'label',
  lazy: true,
  lazyLoad(node, resolve) {
    const { level } = node;
    setTimeout(() => {
      resolve([{
        label: `option-${level + 1}`,
        value: '1',
        leaf: level >= 2
      }])
    }, 300)
  }
}

const value = ref(['1', '1', '1']);
</script>
```

我们将省市区的例子改成了普通数据重复的例子。在懒加载模式下，数据是一级一级加载的，我们事先无法知道下一级数据的情况。因此，如果要对数据进行处理，我们只能在加载到下一级数据时对其进行修改。

```vue
<template>
	 <el-cascader :props="props" v-model="value" />
</template>
<script setup lang="ts">
type dataType = {
  label: string,
  value: string | number,
  leaf: boolean,
  originValue?: string | number;
}

const props: CascaderProps = {
  value: 'value',
  label: 'label',
  lazy: true,
  lazyLoad(node, resolve) {
    const { level } = node;
    const curNodeValue = node.data?.value as string;
    const data: dataType = {
      label: `option-${level + 1}`,
      value: '1',
      leaf: level >= 2
    };
    setTimeout(() => {
      if (!curNodeValue) {
        resolve([data]);
        return;
      }
      const first = curNodeValue.split('-')[0];
      let value = data.value;
      first === value && (data.value = `1-${level + 1}`);
      data.originValue = value;
      resolve([data]);
    }, 300)
  }
}

const value = ref([]);

watch(value, (val) => {
  console.log(val);
});
</script>
```

在这个例子中，假设下一级数据的`value`值和父级的`value`值相同，我们就将当前数据的`value`值后面拼接上`-`加上`level`的值，`level`在每一级是不同的，可以保证值的唯一，当然我们不考虑同级还有相同`value`值的情况。

![image-20230911173624919](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911173624919.png)

通过这种方式，我们获取到的数据就是`['1', '1-2', '1-3']`，这显然不是最原始的值，我们还需要对结果值进行转换处理。

```vue
<script setup lang="ts">
// 省略其它代码
watch(value, (val) => {
  const result: string[] = [];
  val.forEach((item: string) => {
    result.push(item.indexOf('-') !== -1 ? item.split('-')[0] : item);
  });
  console.log(result);
});
</script>
```

在`watch`中我们对结果值进行处理，获取第一个`-`之前的数据，这个数据就是原始数据。

![image-20230911174459474](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911174459474.png)

上面是选择值的逻辑，下面是数据回显的逻辑，当后端数据返回`['1', '1', '1']`时，我们需要转化成组件能够接受的值。

```vue
<script setup lang="ts">
// 省略其它代码
let preValue;
const originValue = ['1', '1', '1'].map((item, index) => {
  if (!preValue) {
    preValue = item;
    return item;
  }
  preValue = item;
  item = `${item}-${index + 1}`;
  return item;
});
const value = ref(originValue);
</script>
```

我们对原始数据进行了处理转化成了`['1', '1-2', '1-3']`。

![2023-8-11-e](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/2023-8-11-e.gif)

可以看到数据能够正常回显。

上面只是我用来解决问题的 demo 写法，对于相同 value 的处理可以自由选择转换规则，只要保证处理后的数据唯一即可。当然，我们也要考虑数据转换和反转的方便性，不能将数据处理得极度复杂，不容易转换。



上面是同事遇到的第一个问题，下面看第二个问题。

```vue
<template>
	 <el-cascader :props="props"/>
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'

let id = 0
const props: CascaderProps = {
  lazy: true,
  lazyLoad(node, resolve) {
    const { level } = node
    setTimeout(() => {
      if (level >= 3) {
        return resolve([]);
      }
      const nodes = Array.from({ length: level + 1 }).map((item) => ({
        value: ++id,
        label: `Option - ${id}`
      }))
      resolve(nodes);
    }, 300)
  },
}
</script>
```

在这个例子中，数据通过点击懒加载的方式获取，当最后一级数据为空时，它的上一级数据无法被选中，如下所示：

![2023-8-11-f](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/2023-8-11-f.gif)

从动图上可以看到最后一级数据并没有显示到下拉框中，但是打开下拉框时，最后一级数据已经是选中状态。

我无法确定这算不算是组件的 bug，毕竟第一次点击时数据没有显示，但是再次打开下拉框时又处于勾选状态，这个问题我们可以归结为叶子节点的处理问题。抛开组件层面的问题，单纯从使用者角度来看应该如何解决？

从我个人理解来看，这种是否是叶子节点的标记应该由后端数据提供，这种树形数据并不是凭空出现的，在软件中肯定有维护数据的地方，在数据维护时就能够知道当前节点是否存在子节点。假设我录入了一条数据 1，再录入一条子数据 2，这时候数据库里面除了新增一条数据 2 的同时，应该也能在数据 1 中标记其存在子节点，比如标记位也叫 leaf，将 leaf 置位 true。这种方式的好处就是前端可以提前知道当前节点是否是叶子节点，如果是叶子几点就不需要再请求接口，在避免上述问题的情况下还能减少接口请求。

当然，我们也要考虑后端不能提供是否是叶子节点的情况下该如何处理。如果组件能够支持在没有获取到子数据时能够更新父节点的`leaf`标记，那这个问题也能够解决，但是在修改过程中我发现了这个它是个`readonly`的字段。

![image-20230911182449317](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911182449317.png)

显然组件不支持我们在这里修改当前点击的节点的`leaf`值。那么既然不返回数据时会有问题，那我们能不能构造一条假数据来解决？

```vue
<template>
	 <el-cascader :props="props"/>
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'

let id = 0
const props: CascaderProps = {
  lazy: true,
  lazyLoad(node, resolve) {
    const { level } = node
    setTimeout(() => {
      if (level >= 3) {
        const { label, value } = node.data!;
        return resolve([{
          label,
          value,
          leaf: true
        }]);
      }
      const nodes = Array.from({ length: level + 1 }).map((item) => ({
        value: ++id,
        label: `Option - ${id}`
      }))

      resolve(nodes);
    }, 300)
  },
}
</script>
```

![image-20230911184619287](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911184619287.png)

我们可以看到数据可以正常被选中，但是选。中的数据就变得有点不太合理，原本数据应该是`['Option-1', 'Option-2', 'Option-4']`，现在末尾又多了一个`Option-4`。这就又涉及到数据的转换，对于我们添加的数据需要标记一个特殊的 value，在最终进行值处理时将这个 value 删除。

```vue
<template>
	 <el-cascader :props="props" v-model="value"/>
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'
import { ref, watch } from 'vue'

let id = 0
const props: CascaderProps = {
  lazy: true,
  lazyLoad(node, resolve) {
    const { level } = node
    setTimeout(() => {
      if (level >= 3) {
        const { label, value } = node.data!;
        return resolve([{
          label,
          value: value + 'custom',
          leaf: true
        }]);
      }
      const nodes = Array.from({ length: level + 1 }).map((item) => ({
        value: ++id,
        label: `Option - ${id}`
      }))

      resolve(nodes);
    }, 300)
  },
}

const value = ref([]);

watch(value, (val) => {
  console.log(val);
});
</script>
```

![image-20230911185045641](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230911185045641.png)

可以看到最后一条数据有我们定义的 custom 值，对于这条数据最后遍历的时候将其去除即可。

同样这是在数据选择时的处理，在数据回显的时候，后端返回的数据不包含我们自定义的 value 值，这时候又该如何回显数据。其实也很简单，组件会匹配我们传入的 value 数组，根据 value 数组中的值自动去加载数据，当加载到最后一级没有数据时，我们可以知道当前被处理的 value，其实基本就是 value 数组中最后一个值。我们只要在 value 数组中再追加一个我们自定义的值即可回显数据。

```vue
<template>
	 <el-cascader :props="props" v-model="orginValue"/>
</template>
<script setup lang="ts">
import type { CascaderProps } from 'element-plus'
import { ref, watch, Ref } from 'vue'

let id = 0
const props: CascaderProps = {
  lazy: true,
  lazyLoad(node, resolve) {
    const { level } = node
    setTimeout(() => {
        if (level >= 3) {
        const { label, value } = node.data!;
        const cutomValue = value + 'custom';
        // 回显时追加数据
        orginValue.value.push(cutomValue)
        return resolve([{
          label,
          value: cutomValue,
          leaf: true
        }]);
      }
      const nodes = Array.from({ length: level + 1 }).map((item) => ({
        value: ++id,
        label: `Option - ${id}`
      }))

      resolve(nodes);
    }, 300)
  },
}

const orginValue: Ref<(string | number)[]> = ref([1, 2, 4]);

watch(orginValue, (val) => {
  console.log(val);
});
</script>
```

![2023-8-11-g](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/2023-8-11-g.gif)

我们依旧可以看到数据可以正常回显。

以上便是这次同事在开发过程中遇到的几个问题，当然最终我们还是跟后端进行了沟通，希望能够在源数据层面进行处理，比如加上 leaf 标记以及提供唯一值字段，这或许是最好的答案。不过，这也不妨碍我们对其它解决方式的探索，万一后端真改不了呢。

下一节我们将通过调试`element-plus`中的源码来一探究竟。