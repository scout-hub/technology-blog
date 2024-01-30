# vue3核心原理（十一）——  更新（下）

### 该部分解析基于我们实现的simplify-vue3中的代码，是vue3源码的阉割版，希望用最简洁的代码来了解vue3的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。

本节我们将介绍`DOM`更新中最复杂的处理场景以及核心`Diff`算法最长递增子序列的应用。

上一节提到，在 `DOM Diff`的所有场景中，最复杂的就是新旧子节点都是数组的场景。在此场景下，新旧子节点的情况会非常复杂，对应的`DOM`操作也不仅仅只是新增和删除，还包含移动。

核心 `DOM Diff` 的逻辑在 `patchKeyedChildren`中，这里将子节点的比较拆分成了多个处理分支，对应不同的情况。



### 一、新旧数组子节点存在相同的前置节点

![image-20240129154822596](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240129154822596.png)

如上图所示，旧的子节点为 `a b e`，新的子节点为 `a b d e`。我们可以很直观地看到新旧子节点中，开头的`a b`节点没有发生任何位置上的变化，对于这种情况只需要直接更新节点即可（属性、事件等等），不需要移动节点。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  const l2 = c2.length;
  let i = 0;
  let oldEnd = c1.length - 1;
  let newEnd = l2 - 1;
  // 双指针从头开始遍历新旧数组，如果新旧节点是同一个节点，则直接对节点进行更新
  while (i <= oldEnd && i <= newEnd) {
    const n1 = c1[i];
    const n2 = c2[i];
    // 新相同节点，调用 patch 更新
    if (isSameVNodeType(n1, n2)) {
      patch(n1, n2, container, parentAnchor, parentComponent, optimized);
    } else {
      // 新旧节点不一样，结束前置遍历
      break;
    }
    i++;
  }
}
```

在 `patchKeyedChildren` 方法中，会优先进行数组的前置遍历，处理新旧数组中存在相同前置节点的情况，直到遇到新旧节点不同为止。



### 二、新旧数组子节点存在相同的后置节点

既然有相同的前置节点，那必然也有相同的后置节点。

![image-20240129154846783](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240129154846783.png)

图中新旧 `e`节点都处于数组的末尾部分，因此，这种情况也只需要更新节点，不需要移动节点。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历的代码
  
  // 双指针从尾部开始遍历新旧数组，如果新旧节点是同一个节点，则直接对节点进行更新
  while (i <= newEnd && i <= oldEnd) {
      const n1 = c1[oldEnd];
      const n2 = c2[newEnd];
      // 新相同节点，调用 patch 更新
      if (isSameVNodeType(n1, n2)) {
        patch(n1, n2, container, parentAnchor, parentComponent, optimized);
      } else {
        // 新旧节点不一样，结束后置遍历
        break;
      }
      oldEnd--;
      newEnd--;
    }
}
```

在 `patchKeyedChildren` 方法中，会进行数组的后置遍历，处理新旧数组中存在相同后置节点的情况，直到遇到新旧节点不同为止。



###  三、前/后置遍历后只存在新增的节点

![image-20240130101059230](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130101059230.png)

通过前置遍历处理完 `a b`节点，并且通过后置遍历处理完 `e`节点后，新的数组节点仅存在一个 `d`节点，这个节点就是需要新增的节点。

在图上可以很直观地看到有新增的节点，但是在代码里面又该如何判断有新增的节点？既然只有新增的节点，那意味着此时老的节点都已经遍历完，但是新的节点还没有遍历完。那么只有新增节点的条件就是 `i > oldEnd && i <= newEnd`。

1. `i > oldEnd`：这个条件成立说明旧的子节点已经全部遍历完毕。

2. `i <= newEnd`：这个条件成立说明新的子节点还有没有遍历的，这些节点就是新增的节点

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  
  // 处理只有新增节点的情况
  if (i > oldEnd && i <= newEnd) {
    // 锚点索引
    const anchorIndex = newEnd + 1;
    // 锚点元素
    const anchor = anchorIndex < l2 ? c2[anchorIndex].el : parentAnchor;
    while (i <= newEnd) {
      // patch 第一个参数是 null，表示新增节点
      patch(null, c2[i], container, anchor, parentComponent, optimized);
      i++;
    }
  }
}
```

在新增节点的场景下，不仅需要考虑有哪些节点需要新增，还需要计算节点插入的位置。这里就要引出锚点节点这个概念，锚点节点就是参考节点的意思。比如图中的 `d` 节点，它的插入方式可以是在 `b` 节点之后，也可以是在 `e` 节点之前，因此参考节点就是 `b` 或者 `e`。但是，这里还要结合 `DOM` 插入的相关方法。在原生 js 中，一般都是通过 `insertBefore`这个方法来执行 `DOM` 插入，这个方法的含义就是在 `xxx`节点之前插入一个新的节点。因此，对于 `b` 和 `e`节点，只能选择 `e`节点作为锚点节点。

![image-20240130101120538](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130101120538.png)

在处理完相同的前后置节点之后，`newEnd`的位置在 `d` 节点上，而需要新增的节点 `d`在 锚点节点 `e` 之前，因此锚点节点的索引位置就是 `newEnd + 1`。在代码中也可以看到 `anchorIndex` 的值就是 `newEnd + 1`，然后根据 `anchorIndex` 获取到锚点节点。

当然，锚点节点可能会不存在，比如下面这种情况。

![image-20240130101330111](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130101330111.png)

当新增的节点刚好是最后一个时，后面已经没有锚点节点了，所以锚点节点是个 `null`，此时插入的效果等同于 `parent.appendChild`。

这里需要关注一下 ` const anchor = anchorIndex < l2 ? c2[anchorIndex].el : parentAnchor` 这段代码。如果 `anchorIndex` 大于等于子节点数组的长度时，显然在当前子节点数组中已经获取不到锚点节点了，这时候会使用 `parentAnchor`,`parentAnchor` 指的是处理到父级节点时的锚点节点。

![image-20240130095528013](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130095528013.png)

结合上图所示， `d` 节点在 `A` 组件内部，而 `e` 节点并不在 `A` 组件内部。当新增 `d` 节点时，在组件 `A` 内部获取到的锚点节点是不存在的，此时 `insertBefore(newNode, anchor)`第二参数为 `null`，效果等同于 `appendChild`，将 `d` 节点添加到父节点的末尾。由于组件本身并不对应某个 `DOM`节点，因此 `a b d` 和 `e` 同属于一个父节点下，那么通过 `insertBefore` 就会将 `d` 节点放到 `e` 节点后面，这显然是有问题的。

虽然在真实的 `DOM` 树中 `a b d e`在同一层，但是在 `VNode` 树中，`a b d`是同一层， `A`组件和 `e`节点是同一层。在遍历处理 `A`组件那一层的 `VNode`时得到的锚点节点就是 `parentAnchor`，这个 `parentAnchor`就是跟 `A` 组件同级的下一个 `VNode` 对应的真实 `DOM` 节点，在图中简单认为就是节点 `e`，此时通过 `insertBefore` 就能正常插入到节点 `e` 之前。



### 四、前/后置遍历后只存在删除的节点

和上面提到的第三点相反，在进行完前/后置遍历后只剩需要删除的节点。

![image-20240130101030974](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130101030974.png)

如上图所示，前置遍历会处理 `a b`节点，后置遍历会处理 `d`节点，遍历完成后，新的子节点数组已经全部访问完成，老的子节点数组中只剩下 `c` 节点未访问，这个节点就是需要删除的节点。

那么，判断是否只剩需要删除的节点依据就是 `i > newEnd && i <= oldEnd`。

1. `i > newEnd`：这个条件成立说明新的子节点已经全部遍历完毕。
2. `i <= oldEnd`：这个条件成立说明旧的子节点还有没有遍历的，这些节点就是需要删除的节点

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  
  // 处理只有删除节点的情况
  else if (i > newEnd && i <= oldEnd) {
    /**
       * 取出i到oldEnd之间的元素，这些元素为需要删除的元素
       * (a) b c (d) ====> (a) (d) ：b、c是需要删除的节点
     */
    while (i <= oldEnd) {
      unmount(c1[i], parentAnchor);
      i++;
    }
  }
}
```

删除的逻辑很简单，循环从当前未访问的节点（`i`）开始一直到后置遍历的结束点（`oldEnd`），中间都是需要删除的节点，对这些节点调用 `unmount` 即可。



### 五、最长递增子序列

前面几种情况可以说是一种比较理想的节点存在关系，刚好存在相同的前、后置节点，当前后置节点处理完成后刚好又只剩下需要新增/删除的节点。在实际开发中，我们可能会面对更为复杂的节点关系。

![image-20240130103207926](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130103207926.png)

如上图所示，新旧子节点除了前置节点 `a`和后置节点 `e`，中间部分的节点关系非常复杂。其中，`b c d`节点只是位置发生了变化，`f` 节点需要删除，`g` 节点需要新增。

其实无论情况有多复杂，最终需要知道三个点：

- 是否存在节点移动的情况。
- 如果存在节点移动，如何知道哪些节点需要移动，怎么移动。
- 找到需要新增或者删除的节点。

第三点相对来说比较简单，这里重点讲解前面两个点。



### 1. 确定节点是否需要移动

节点移动无非就是节点顺序在更新前后发生了变化，用数组索引来表示就是各自的索引发生了变化。

![image-20240130105946321](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130105946321.png)

在上图中，`a b c`节点在变化前后的索引没有发生任何变化，因此它们的位置关系也没有发生变化，此时不需要移动节点。

下面再看一下节点需要移动的情况。

![image-20240130110357796](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130110357796.png)

图中，老的数组节点的位置关系是 `a b c` ===> `0 1 2`，而在新的数组节点中 `a b c` ===> `0 2 1`，显然 `b c` 节点的索引位置发生了变化，因此需要进行节点移动。

那是不是意味着只要变化前后节点的索引发生变化就需要移动节点了？答案是：不一定。

![image-20240130111832943](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130111832943.png)

在上面这张图中，`a b c`节点的索引在变化前后为 `0 1 2`和 `1 2 3`，这时候需要移动 `a b c`节点吗？显然不需要，从图中我们也能发现，变化前后只需要删除 `e`节点，然后在 `a`节点之前新增一个 `f`节点即可，完全不需要移动 `a b c `节点。

既然节点索引的变化不是用来判断节点是否需要移动的依据，那么节点移动的依据到底是什么？

回顾一下上面两张图中节点索引的关系：

1. `a b c ===> a c b`，`a b c`节点的索引变化关系：`0 1 2 ===> 0 2 1`
2. `a b c e` ===> `f a b c`，`a b c`节点的索引变化关系：`0 1 2 ===> 1 2 3`

不难发现，从 `0 1 2` 变化到 `0 2 1`，索引的关系不再是一个递增的关系，而 `0 1 2` 变化到 `1 2 3`，索引依旧呈现递增关系。而保持递增关系的情况下不需要进行节点移动，相反则需要进行节点移动。

因此，判断节点是否需要移动的依据就是：**变化前后节点的索引关系是否是递增关系**。

现在知道了什么情况下需要进行节点移动，接下去就需要知道哪些节点需要移动，怎么移动。



### 2. 哪些节点需要移动，如何移动

![image-20240130110357796](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130110357796.png)

假设移动 `a` 节点，那么步骤就是：

1. 将 `a` 节点移动到 `c` 节点之前。
2. 将 `b` 节点移动到 `c` 节点之后。

![image-20240130114407053](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130114407053.png)

假设移动 `b` 节点，步骤就是：

1. 将 `b` 节点移动到 `c` 节点之后。

![image-20240130114531715](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130114531715.png)

假设移动 `c` 节点，步骤就是：

1. 将 `c` 节点移动到 `b`节点之前。

![image-20240130114620075](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130114620075.png)

直观上可以看到，这三种移动方式中，只有第一种，先移动 `a` 节点的情况下会多一步。因此，合理的移动方式应该是后面两种。

那么从逻辑上应该如何辨别移动方式？这里依旧需要用到第一点的结论：索引的递增关系。

在新旧子节点数组中 `a b c`的关系从 `0 1 2`变化到`0 2 1`。从索引递增的角度看，`a`节点的索引 `0`，在更新前后总是能和 `b` 或者 `c` 节点保持一个递增关系 `0 1`、`0 2`，而变化后只有 ` 2 1` 不是一个递增关系。因此，`a b` 或者 `a c` 两个节点是不需要移动的，相反只需要移动 `c`或者 `b`节点即可，这也印证了上面后两种移动方式是合理的。

现在这个问题就变成了：**找到数组中最长递增的索引序列，这些索引序列对应的节点不需要移动，而其它节点则需要移动、新增或者删除 ———— 最长递增子序列的应用**



### 六、具体实现

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  // 省略处理只有删除节点的代码

  // 处理最复杂的节点情况
  else {
    // 未处理的节点数量
    const toBePatched = newEnd - i + 1;
    // 初始化索引数组
    const newIndexToOldIndexMap = new Array(toBePatched).fill(-1);
  }
}
```

第一步，构建一个新的数组 `newIndexToOldIndexMap`，这个数组初始化长度为新的子节点数组中未处理的节点数量，初始值为 -1，未处理的节点数量 `toBePatched = newEnd - i + 1`。（下图中红色部分）

这个数组的作用是存储新的子节点数组中未处理的节点在老的子节点数组中对应的索引位置，以便计算出它的最长递增子序列来辅助 `DOM` 操作。

初始化之后的 `newIndexToOldIndexMap`就是 `[-1, -1, -1, -1]`。

![image-20240130141247691](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130141247691.png)

第二步，构建一个索引表 `keyToNewIndexMap`，存储新的子节点数组中节点的索引信息，用以辅助填充 `newIndexToOldIndexMap` 数组。这个索引表的 `key`是节点的 `key`，也就是我们平时开发中给节点加唯一标识的 `key`属性，索引表的值就是索引位置。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  // 省略处理只有删除节点的代码

  // 处理最复杂的节点情况
  else {
    // 1. 构建一个新的数组 newIndexToOldIndexMap。
    // 未处理的节点数量
    const toBePatched = newEnd - i + 1;
    // 初始化索引数组
    const newIndexToOldIndexMap = new Array(toBePatched).fill(-1);
    // 2. 构建索引表 keyToNewIndexMap
    const newStart = i;
    const oldStart = i;
    const keyToNewIndexMap = new Map();
    // 索引表的 key 是节点的 key 属性值，值是索引
    for (let i = newStart; i <= newEnd; i++) {
      keyToNewIndexMap.set(c2[i].key, i);
    }
  }
}
```

初始化之后的 `keyToNewIndexMap`就是 `{ c: 1, d: 2, b: 3, g: 4}`（如下图所示）。

![image-20240130141838547](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130141838547.png)

第三步，填充 `newIndexToOldIndexMap` 并判断是否需要进行节点移动或者节点删除。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  // 省略处理只有删除节点的代码

  // 处理最复杂的节点情况
  else {
    // 1. 构建一个新的数组 newIndexToOldIndexMap。
    // 未处理的节点数量
    const toBePatched = newEnd - i + 1;
    // 初始化索引数组
    const newIndexToOldIndexMap = new Array(toBePatched).fill(-1);
    // 2. 构建索引表 keyToNewIndexMap
    const newStart = i;
    const oldStart = i;
    const keyToNewIndexMap = new Map();
    // 索引表的 key 是节点的 key 属性值，值是索引
    for (let i = newStart; i <= newEnd; i++) {
      keyToNewIndexMap.set(c2[i].key, i);
    }

    // 3. 填充newIndexToOldIndexMap，并判断是否需要进行节点移动或者节点删除。
    // 判断DOM是否需要移动
    let moved = false;
    // 存储旧子节点在新子节点数组中的最大索引值
    let maxNewIndexSoFar = 0;
  }
}
```

在上面这段代码中新增了两个变量，`moved` 用于标记是否存在节点移动的情况。`maxNewIndexSoFar` 用来记录遍历到的旧子节点在新子节点数组中的最大索引位置。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  // 省略处理只有删除节点的代码

  // 处理最复杂的节点情况
  else {
    // 1. 构建一个新的数组 newIndexToOldIndexMap。
    // 未处理的节点数量
    const toBePatched = newEnd - i + 1;
    // 初始化索引数组
    const newIndexToOldIndexMap = new Array(toBePatched).fill(-1);
    // 2. 构建索引表 keyToNewIndexMap
    const newStart = i;
    const oldStart = i;
    const keyToNewIndexMap = new Map();
    // 索引表的 key 是节点的 key 属性值，值是索引
    for (let i = newStart; i <= newEnd; i++) {
      keyToNewIndexMap.set(c2[i].key, i);
    }

    // 3. 填充newIndexToOldIndexMap，并判断是否需要进行节点移动或者节点删除。
    // 判断DOM是否需要移动
    let moved = false;
    // 存储旧子节点在新子节点数组中的最大索引值
    let maxNewIndexSoFar = 0;
    // 遍历未处理的旧子节点
    for (let i = oldStart; i <= oldEnd; i++) {
      const oldVnode = c1[i];
      let keyIndex;
      if (oldVnode.key != null) {
        keyIndex = keyToNewIndexMap.get(oldVnode.key);
      } else {
        // 如果用户没有传入key，则需要再加一层循环去寻找节点（传入key的重要性，避免O(n²)的时间复杂度）
        for (let j = newStart; j <= newEnd; j++) {
          if (isSameVNodeType(oldVnode, c2[j])) {
            keyIndex = j;
            break;
          }
        }
      }
    }
  }
}
```

接下去从第一个未处理的旧子节点开始循环，依次找到每一个旧子节点在新子节点数组中的索引位置，查找方式就是根据 `key` 去查找，如果没有定义 `key`，那么就会在内部再建立一个循环去查找。从这里我们也能得知 `key`的重要性，如果定义了 `key`，可以避免 `O(n²)`的复杂度。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  // 省略处理只有删除节点的代码

  // 处理最复杂的节点情况
  else {
    // 1. 构建一个新的数组 newIndexToOldIndexMap。
    // 未处理的节点数量
    const toBePatched = newEnd - i + 1;
    // 初始化索引数组
    const newIndexToOldIndexMap = new Array(toBePatched).fill(-1);
    // 2. 构建索引表 keyToNewIndexMap
    const newStart = i;
    const oldStart = i;
    const keyToNewIndexMap = new Map();
    // 索引表的 key 是节点的 key 属性值，值是索引
    for (let i = newStart; i <= newEnd; i++) {
      keyToNewIndexMap.set(c2[i].key, i);
    }

    // 3. 填充newIndexToOldIndexMap，并判断是否需要进行节点移动或者节点删除。
    // 判断DOM是否需要移动
    let moved = false;
    // 存储旧子节点在新子节点数组中的最大索引值
    let maxNewIndexSoFar = 0;
    // 遍历未处理的旧子节点
    for (let i = oldStart; i <= oldEnd; i++) {
      const oldVnode = c1[i];
      let keyIndex;
      if (oldVnode.key != null) {
        keyIndex = keyToNewIndexMap.get(oldVnode.key);
      } else {
        // 如果用户没有传入key，则需要再加一层循环去寻找节点（传入key的重要性，避免O(n²)的时间复杂度）
        for (let j = newStart; j <= newEnd; j++) {
          if (isSameVNodeType(oldVnode, c2[j])) {
            keyIndex = j;
            break;
          }
        }
      }
      if (keyIndex !== undefined) {
        // 填充 newIndexToOldIndexMap
        newIndexToOldIndexMap[keyIndex - newStart] = i;
        // 递增关系，不需要移动，重新赋值maxNewIndexSoFar
        if (keyIndex >= maxNewIndexSoFar) {
          maxNewIndexSoFar = keyIndex;
        } else {
          // 表示DOM需要移动
          moved = true;
        }
        // 找到了节点，更新
        patch(
          oldVnode,
          c2[keyIndex],
          container,
          parentAnchor,
          parentComponent,
          optimized
        );
      } else {
        // 没找到节点，删除
        unmount(oldVnode, parentComponent);
      }
    }
  }
}
```

如果当前旧子节点能在新子节点数组中找到索引（ `keyIndex !== undefined`），说明更新后该子节点依旧存在，此时需要填充 `newIndexToOldIndexMap` 并更新该节点（ `patch`），否则说明该节点不存在，需要删除 `unmount`。

假设 `keyIndex` 存在，如果 `keyIndex`比 `maxNewIndexSoFar`还大，说明当前的旧子节点在更新后依旧和之前遍历过的节点保持着递增关系，因此不需要移动，只需要更新 `maxNewIndexSoFar`，否则说明需要移动节点，将 `moved` 标记为 `true`。

下面用图来描述一下这个过程。

![image-20240130151238879](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130151238879.png)

第一个遍历到的旧子节点是 `b`，`i` 和 `newStart`都是 1。从 `keyToNewIndexMap` 中 `b`可以获取到 `b`的新索引位置 `keyIndex` 是 `3`，`keyIndex - newStart` 就是 2，此时 `newIndexToOldIndexMap` 更新为 `[-1, -1, 1, -1]`。由于 `keyIndex: 3` 大于初始的 `maxNewIndexSoFar: 0`，因此目前不存在移动节点的情况，只需要更新 `maxNewIndexSoFar = 3` 并且更新 `b`节点。如下图所示。

![image-20240130152313697](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130152313697.png)

第二个遍历到的旧子节点是 `c`, `i`是 2，`newStart`是 1。`keyIndex` 是 `1`，`keyIndex - newStart` 是 0，`newIndexToOldIndexMap` 更新为 `[2, -1, 1, -1]`。由于 `keyIndex: 1`要比当前的 `maxNewIndexSoFar: 3`小，因此目前存在节点移动的情况，需要将 `moved`置为 `true`并更新 `c` 节点。如下图所示。

![image-20240130152159228](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130152159228.png)

第三个遍历到的旧子节点是 `d`，`i` 是 `3`，`newStart`是 1，`keyIndex`是 2，`keyIndex - newStart` 是 1，`newIndexToOldIndexMap` 更新为 `[2, 3, 1, -1]`。由于 `keyIndex: 2`要比当前的 `maxNewIndexSoFar: 3`小，因此跟节点 `c` 的处理情况相同。如下图所示。

![image-20240130152839607](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130152839607.png)

第四个遍历到的是节点 `f`，`f`在 `keyToNewIndexMap`中并不能找到对应的索引信息，因此判定为需要删除的情况。如下图所示。

![image-20240130153133088](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130153133088.png)

到目前为止就已经填充完 `newIndexToOldIndexMap`并标记了节点是否需要移动以及多余节点的删除。

对于节点删除部分还有一个优化点。先前我们曾记录了一个变量 `toBePatched`，用来表示有多少节点需要更新。在循环旧子节点的过程中，我们可以加一个标记 `patched`，用来标识已经更新了多少节点，假如 `patched === toBePatched`，意味着后续遍历到的旧子节点都可以删除。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  // 省略处理只有删除节点的代码

  // 处理最复杂的节点情况
  else {
    // 1. 构建一个新的数组 newIndexToOldIndexMap。
    // 未处理的节点数量
    const toBePatched = newEnd - i + 1;
    // 初始化索引数组
    const newIndexToOldIndexMap = new Array(toBePatched).fill(-1);
    // 2. 构建索引表 keyToNewIndexMap
    const newStart = i;
    const oldStart = i;
    const keyToNewIndexMap = new Map();
    // 索引表的 key 是节点的 key 属性值，值是索引
    for (let i = newStart; i <= newEnd; i++) {
      keyToNewIndexMap.set(c2[i].key, i);
    }

    // 3. 填充newIndexToOldIndexMap，并判断是否需要进行节点移动或者节点删除。
    // 判断DOM是否需要移动
    let moved = false;
    // 存储旧子节点在新子节点数组中的最大索引值
    let maxNewIndexSoFar = 0;

    // 记录已经更新过的子节点
    let patched = 0;

    // 遍历未处理的旧子节点
    for (let i = oldStart; i <= oldEnd; i++) {
      const oldVnode = c1[i];
      // 当已经更新过的子节点数量大于需要遍历的新子节点数组时，表示旧节点数量大于新节点数量，需要删除

      if (patched >= toBePatched) {
        unmount(oldVnode, parentComponent);
        continue;
      }

      let keyIndex;
      if (oldVnode.key != null) {
        keyIndex = keyToNewIndexMap.get(oldVnode.key);
      } else {
        // 如果用户没有传入key，则需要再加一层循环去寻找节点（传入key的重要性，避免O(n²)的时间复杂度）
        for (let j = newStart; j <= newEnd; j++) {
          if (isSameVNodeType(oldVnode, c2[j])) {
            keyIndex = j;
            break;
          }
        }
      }
      if (keyIndex !== undefined) {
        // 填充 newIndexToOldIndexMap
        newIndexToOldIndexMap[keyIndex - newStart] = i;
        // 递增关系，不需要移动，重新赋值maxNewIndexSoFar
        if (keyIndex >= maxNewIndexSoFar) {
          maxNewIndexSoFar = keyIndex;
        } else {
          // 表示DOM需要移动
          moved = true;
        }
        // 找到了节点，更新
        patch(
          oldVnode,
          c2[keyIndex],
          container,
          parentAnchor,
          parentComponent,
          optimized
        );
        
        // 表示又更新了一个节点
        patched++;
      } else {
        // 没找到节点，删除
        unmount(oldVnode, parentComponent);
      }
    }
  }
}
```

第四步，根据 `newIndexToOldIndexMap` 计算最长递增子序列，根据最长递增子序列来进行节点移动，并处理节点新增的情况。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  // 省略处理只有删除节点的代码

  // 处理最复杂的节点情况
  else {
    // 省略 newIndexToOldIndexMap 填充以及节点删除的逻辑
    
    // 计算最长递增子序列
    const increasingNewIndexSequence = moved
        ? getSequence(newIndexToOldIndexMap)
        : EMPTY_ARR;
  }
}

/**
 * @author: Zhouqi
 * @description: 最长递增子序列
 * @return 最长递增序列的递增索引
 */
const getSequence = (arr: number[]): number[] => {
  // 用于回溯的数组，记录的是比当前数小的前一个数的下标
  let temp = Array(arr.length);
  // 初始结果默认第一个为0，记录的是arr中数据对应的下标
  let result = [0];
  for (let i = 0; i < arr.length; i++) {
    const num = arr[i];
    if (num > 0) {
      let j = result[result.length - 1];
      // 如果当前遍历到的数字比结果中最后一个值对应的数字还要大，则直接将下标添加到result末尾
      if (num > arr[j]) {
        // j就是比当前这个数小的前一个数的索引，记录它（为了最后修正下标）
        temp[i] = j;
        // 这里记录的是索引
        result.push(i);
        continue;
      }
      // 用二分法查找比当前这个数要大的第一个数的位置并替换它
      let left = 0;
      let right = result.length - 1;
      while (left < right) {
        const mid = (left + right) >> 1;
        if (arr[result[mid]] < num) {
          left = mid + 1;
        } else {
          right = mid;
        }
      }
      // 最终left === right
      // 判断找到的数是不是比当前这个数大
      if (arr[result[left]] > num) {
        if (left > 0) {
          // left - 1就是比当前这个数小的前一个数的索引，记录它
          temp[i] = result[left - 1];
        }
        result[left] = i;
      }
    }
  }
  // 回溯，修正下标
  let resultLen = result.length;
  let k = result[resultLen - 1];
  while (resultLen-- > 0) {
    result[resultLen] = k;
    k = temp[k];
  }
  return result;
};
```

`getSequence`方法用于计算最长递增子序列，返回最长递增子序列的索引信息。例如前面填充完成后的 `newIndexToOldIndexMap`为 `[2, 3, 1, -1]`，而`[2, 3, 1, -1]`分别表示新的 `c d b g`节点在老的子节点数组中对应的索引位置。

![image-20240130155837046](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130155837046.png)

`[2, 3, 1, -1]`中最长递增子序列为 `[2, 3]`，它们的索引为 `[0, 1]`。因此，`getSequence`的返回值就是 `[0, 1]`，这表示在新的未处理的子节点数组中，下标为 0 和 1 的节点不需要移动。

这里我们需要区分清楚新的子节点数组和新的未处理的子节点数组。按照上图所示的例子，新的子节点数组是 `[a, c, d, b, g]`，新的未处理的子节点数组是 `[c, d, b, g]`。而 `getSequence`返回的索引数组是跟 `[c, d, b, g]`对应的，结合前面提到的返回值 `[0, 1]`，表示节点 `c`、`d`不需要移动，其它节点可能需要移动或者新增。

![image-20240130161326467](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130161326467.png)

得到 `increasingNewIndexSequence` 后就要开始处理节点移动和新增了。

```typescript
// packages/runtime-core/src/renderer.ts
/**
   * @author: Zhouqi
   * @description: 快速diff算法
   * @param c1 旧的子节点
   * @param c2 新的子节点
   * @param parentAnchor 锚点元素
   * @param container 容器
   * @param parentComponent 父组件实例
   * @param optimized 是否优化
*/
const patchKeyedChildren = (
  c1,
  c2,
  container,
  parentAnchor,
  parentComponent,
) => {
  // 省略前置遍历和后置遍历的代码
  // 省略处理只有新增节点的代码
  // 省略处理只有删除节点的代码

  // 处理最复杂的节点情况
  else {
    // 省略 newIndexToOldIndexMap 填充以及节点删除的逻辑

    // 计算最长递增子序列
    const increasingNewIndexSequence = moved
    ? getSequence(newIndexToOldIndexMap)
    : EMPTY_ARR;

    let seq = increasingNewIndexSequence.length - 1;
    const j = toBePatched - 1;

    for (let i = j; i >= 0; i--) {
      const pos = i + newStart;
      const newVnode = c2[pos];
      const anchor = pos + 1 < l2 ? c2[pos + 1].el : parentAnchor;
      if (newIndexToOldIndexMap[i] === -1) {
        // 索引为-1说明没有在老的里面找到对应的节点，说明是新节点，需要挂载
        patch(null, newVnode, container, anchor, parentComponent, optimized);
      } else if (moved) {
        // 需要移动的情况：
        // 1、没有最长递增子序列元素可以遍历了（j<0）
        // 2、最长递增子序列对应的元素索引和当前遍历到的元素的索引不一样（i !== increasingNewIndexSequence[seq]）
        if (i !== increasingNewIndexSequence[seq]) {
          // 需要移动节点
          move(newVnode, container, anchor);
        } else {
          // 找到了对应不需要移动的节点，只需要更新seq
          seq--;
        }
      }
    }
  }
}
```

这里采用倒序的方式遍历新的子节点数组，这是为了保证后面的节点先被处理。因为节点的移动需要找到锚点节点也就是当前节点的下一个节点，通过`insertBefore`进行节点移动，因此要确保锚点节点一定是稳定的，只有已经遍历处理过的节点才能保证后续不再变动。

遍历过程中，假如 `newIndexToOldIndexMap[i] === -1`，说明该节点在老的子节点数组中没有找到对应的索引，是属于需要新增的节点（`patch`）。如果当前遍历到的节点位置信息和 `increasingNewIndexSequence`中的位置信息对应，则说明是不需要移动的节点，否则就需要移动节点 (`move`)。

我们继续用图来描述一下这个过程。

![image-20240130164330469](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130164330469.png)

第一次遍历，`i`是 3，`newStart` 是 1，`pos = i + newStart` 为 4，因此第一个处理的节点是 `g`，`g`在 `newIndexToOldIndexMap`中的索引值是 -1，因此 `g`节点属于要新增的节点。

![image-20240130165303718](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130165303718.png)

第二次遍历，`i`是 2，`newStart` 是 1，`pos = i + newStart` 为 3，处理的节点为 `b`，`b`在 `newIndexToOldIndexMap`中的索引值是 1，不属于需要新增的节点。接着看节点是否需要移动，此时`seq`为 1，`increasingNewIndexSequence[seq]` 为 1，因此`i !== increasingNewIndexSequence[seq]`，说明节点 `b` 需要移动，移动到 `g` 节点之前。

![image-20240130170259516](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130170259516.png)

第三次遍历，`i`是 1，`newStart` 是 1，`pos = i + newStart` 为 2，处理的节点为 `d`，`d`在 `newIndexToOldIndexMap`中的索引值是 3，不属于需要新增的节点。此时`seq`为 1，`increasingNewIndexSequence[seq]` 为 1，因此`i === increasingNewIndexSequence[seq]`，说明节点 `d` 不需要移动，更新 `seq`即可。

![image-20240130170656333](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130170656333.png)

最后一个要处理的节点就是 `c`，此时 `i`为 0，`seq`为 0，`increasingNewIndexSequence[seq]` 为 0，因此`i === increasingNewIndexSequence[seq]`，所以 `c`节点也不需要移动。

![image-20240130170839511](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20240130170839511.png)

最终 `DOM` 更新的结果就是 `a c d b g e`。



至此我们便简单了解完了 `Vue3`中关于 `DOM Diff`的部分，其实这部分内容远没有结束，后续还会讲解更新流程中关于快捷路径更新的原理。
