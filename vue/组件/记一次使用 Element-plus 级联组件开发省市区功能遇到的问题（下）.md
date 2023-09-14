# 记一次使用 Element-plus 级联组件开发省市区功能遇到的问题（下）

上一节讲述了同事在用`element-plus`级联组件开发省市区选择功能时遇到的问题并提出了几种解决方案，本节将深入`element-plus`源码，探究之前遇到的问题。



### 一、克隆一份`element-plus`的源码

```
git clone https://github.com/element-plus/element-plus.git
```

克隆完成后的目录如下图所示，其中`packages`目录中包含核心组件的逻辑，`play`目录则是用来调试的目录，我们可以在`play`目录中添加 demo 代码。

![image-20230913153858522](../../Library/Application Support/typora-user-images/image-20230913153858522.png)

### 二、安装依赖

````
pnpm i
````



### 三、编写测试代码

前面说了`play`目录可以用来进行组件测试，我们在`play`目录中添加对应的 demo 用例。

![image-20230913154505039](../../Library/Application Support/typora-user-images/image-20230913154505039.png)



### 四、查找源代码相关逻辑

我们的问题是在懒加载模式出现的，因此只要找到`lazyload`相关的处理逻辑即可。这个逻辑在`cascader-panel`的`index.vue`文件中，我们在对应方法处打上一个断点

![image-20230913155010432](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913155010432.png)

### 五、开启`vscode`调试

在`play`目录中有一个`package.json`文件，在文件里面会显示调试按钮，点击按钮并选择`dev`模式即可开启调试。

![image-20230913155209869](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913155209869.png)![image-20230913155442709](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913155442709.png)

服务启动后打开对应的地址后可以看到已经进入了断点调试，并且断点已经停在我们之前打住的那个`lazyload`方法上。`vscode`调试模式跟`chrome devtools`调试其实差不多，它的调试面板里面功能非常全面，如果要调试源码的话用`vscode`调试还是很香的。

![image-20230913155616627](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913155616627.png)

![image-20230913160026793](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913160026793.png)

注意：在断点调试的时候可能会发现断点漂移的情况，大概情况进入某个方法时，断点并没有定位到方法第一行，而是定位到了下面某行代码上，实际逻辑运行还是在第一行。这个问题可能是由于某些插件导致的，在对插件进行排除的过程中，我发现是`rollup-plugin-esbuild`的问题，因此我们需要把这个插件注释了。

![image-20230914160647583](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914160647583.png)



### 六、断点调试

#### 问题1：['北京市', '北京市', '东城区'] 数据回显问题

首次进入`lazyload`方法时还没有生成树节点，因此会先创建一个根节点`new Node`，将节点的`loading`属性标记为`true`，表示正在加载节点，之后进入我们在`props`中定义的`lazyload`方法，去获取对应的子节点数据。

![image-20230913162616796](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913162616796.png)

首先获取的就是`province`的数据，数据获取到后需要执行组件抛出的`resolve`方法并传入数据。

![image-20230913163231845](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913163231845.png)

`resolve`方法内部会将获取到的节点存储起来，其核心逻辑就是`appendNodes`，在`appendNodes`中最终会调用`appendNode`将数据存储到`allNodes`中。其中，根节点还会存储到`nodes`中，叶子节点还会存储到`leafNodes`中。

![image-20230913163632052](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913163632052.png)

![image-20230913163841013](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913163841013.png)

接下去执行`cb(dataList)`回调，在回调中会执行`syncCheckedValue`。

![image-20230913164244187](../../Library/Application Support/typora-user-images/image-20230913164244187.png)

在`syncCheckedValue`中会获取一些数据，比如此时传入的`modelValue`值，是否是懒加载`lazy`以及节点是否加载完成`loaded`。根据此时获取到的值会走`if (lazy && !loaded) `这条分支代码。

![image-20230913164401248](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913164401248.png)

在这个条件分支中第一步就是对`modelValue`值进行处理，处理完成后的值如下所示。我们可以看到先前传入的两个北京市变成了一个北京市，也就是数据被去重了。其实看到这一步我们就能发现一个问题，`modelValue`中的两个北京市由于数据去重`unique`导致剩下一个北京市和一个东城区，这个北京市能够匹配到例子中`province`中的北京市数据，但是`province`的下一级`city`中也有一条北京市数据。根据一一对应关系，`city`中的北京市数据和`modelValue`中的第二条数据`东城区`是匹配不了的，这就可能造成之前我们看到的那个现象，第一个北京市一直在转圈圈。

![image-20230913164830155](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913164830155.png)

上面也仅仅是初步的猜测，下面就需要印证这个猜测。我们可以修改一下源代码，在处理`modelValue`的时候去掉`unique`函数。

![image-20230913165748859](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913165748859.png)

当我们去掉`unique`函数后确实看到北京市没有被去重，我们可以放开断点看看视图中的数据能不能正常回显。

![image-20230913165954873](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913165954873.png)

 结果还是不尽如意人，北京市依旧在圈圈，那么问题可能不止这一个点，我们需要继续往下调试。接下去会再次执行`lazyload`去加载当前节点的子节点数据，接下去的子节点就是北京市的下一级节点，在 demo 中就是`city`中的北京市。

![image-20230913170136790](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913170136790.png)

当下一级数据加载完成后会继续走`resolve`中的`cb`回调`syncCheckedValue`。在这次的`syncCheckedValue`中我们看到通过`values`匹配到的节点数据是空的，这似乎有点不太合理，第一个`北京市`由于已经匹配过`province`中的`北京市`，所以在`filter`的时候被过滤了。而`东城区`由于还没加载到节点数据，所以匹配不到。但是第二个`北京市`是第一次匹配数据，应该能够匹配到`city`中的`北京市`，显然在`getNodeByValue`中存在一些逻辑使得匹配失效了。

![image-20230913170811135](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913170811135.png)

我们尝试在`DEBUG CONSOLE`中打印输出一下`values.map((val) => store?.getNodeByValue(val))`。

![image-20230913171456954](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913171456954.png)

可以看到数据确实有三条，前两条属于`北京市`匹配到的数据，最后一条`东城区`是没有匹配到的，所以为`null`。我再展开前两条数据看看

![image-20230913171614507](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913171614507.png)

这里就比较奇怪了，这两条节点数据居然是一样的，而且是`province`中的北京市节点数据。显然我们需要看一下`getNodeByValue`的逻辑。

![image-20230913171920829](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913171920829.png)

![image-20230913172407969](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913172407969.png)

这下可算明白了，在获取节点时会从`allNodes`中通过`find`的方式去查找，在`allNodes`中存在两条`北京市`的节点数据，并且`value`值也是相同的（demo 里面将 label 字段作为了 value 字段）。虽然我们认为`modelValue`中的两个`北京市`是不一样的值，但是按照数据查找的先后顺序会找到`province`中的北京市节点数据，因此才会返回两条一模一样节点数据，并且节点数据在之前已经加载过，因此被`filter`之后就变成空数组了。

看到这里，作为一个开发者，我们需要思考源码这么处理是否合理，毕竟实际开发过程中同事遇到了这种特殊的场景。我个人认为，在单个选项中`value`值重复不应该算是有问题的，毕竟值是非同级的，它属于一种父子关系。同一层级上的`value`重复会有问题，但是不同层级的`value`重复是可以接受的。虽然结果是一个数组，看上去是平级关系。

既然个人觉得是合理的，那就需要看如何修改源码才能使数据正常回显。我们再回顾一下`getNodeByValue`这个方法，在这个方法中会去查找节点，查找的规则是`isEqual(node.value, value) || isEqual(node.pathValues, value)`，其中这个`pathValues`是什么？

![image-20230913174646510](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913174646510.png)

我们在`DEBUG CONSOLE`中打印了一下所有节点信息，可以看到`province`和`city`中的北京市的`pathValues`是不同的，而这个`pathValues`值刚好带有层级的意思。那首先我们是不是可以考虑优先匹配`pathValue`，其次匹配`value`？至少我觉得在我们这个场景中应该是可以的，如果其它场景不支持还需要加配置判断，我们暂时不考虑其它影响点。其次，`pathValue`具有层级关系，而我们的`modelValue`是没有层级关系的，但是我们可以把它处理成有层级关系的值。

第一步，调整判断顺序，先匹配`pathValues`。

![image-20230913180001764](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913180001764.png)

第二步处理传入的`modelValue`值，将`[1, 2, 3]`处理成`[[1], [1, 2], [1, 2 ,3]]`这种有层级关系的数据（写法可能不算最完美的）。

![image-20230913180126618](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230913180126618.png)

最后一步，看结果。

![chrome-capture-2023-8-13](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-8-13.gif)

可以看到数据能够正常回显了，说明我们之前的分析还是有一丢丢道理的。



#### 问题2：当点击某个节点加载子节点并且子节点数据为空时，当前节点无法被选中

我们先调整一下 demo

![image-20230914155620339](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914155620339.png)

接着再回顾一下这个问题

![chrome-capture-2023-8-14](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-8-14.gif)

当我们点击`Option-4`选项时会去加载下一级节点，但是此时获取到的是空数据，因此`Option-4`此时应该是叶子节点，但是当我们再次点击这个节点时数据并没有显示到输入框中，再次展开下拉时数据是勾选状态。从操作上看，这一块属于点击节点时的处理，我们需要找到源码中点击节点相关的逻辑，这块逻辑就是`node.vue`中的`handleClick`。

![image-20230914161006091](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914161006091.png)

我们把断点打到了这个方法上，然后进行点击操作，当我点击`Option-4`的时候，由于它的`leaf`属性是`false`，所以会走`handleExpand`方法。

![image-20230914161441627](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914161441627.png)

`handleExpand`方法中会执行`doLoad`方法，在`doLoad`方法中会调用之前讲过的`lazyLoad`方法去加载节点数据，加载完成后会执行 cb 回调，这里的回调就是`()=> if(!isLeaf.value) doExpand()`，这里读取`isLeaf.value`时会触发`get`操作，`get`操作中会对`leaf`的值进行计算。

![image-20230914165223219](../../Library/Application Support/typora-user-images/image-20230914165223219.png)

在对节点`Option-4`的`leaf`计算时，由于其子节点`childrenData`是空数组，因此会将`leaf`标记为`true`。在我们刚才操作的过程中也可以看到，当点击`Option-4`节点时会处于一个加载状态，随后这个节点右边的箭头消失了，也就是说此时`Option-4`确实变成了叶子节点，但是为什么再次点击这个节点时却无法选中数据？我们继续看逻辑。

当我们再次点击`Option-4`时会触发`handleClick`中的`handleCheck(true)`。

![image-20230914171108410](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914171108410.png)

在`handleCheck`中会对节点进行勾选操作，之后会计算勾选后的值`calculateCheckedValue`

![image-20230914172235790](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914172235790.png)

问题其实就出现在`calculateCheckedValue`中，在`calculateCheckedValue`中会通过`getCheckedNodes`方法获取所有选中的节点，这里还传入了一个变量`checkStrictly`，它是父子节点不关联的一个配置，值时`false`。

![image-20230914172310068](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914172310068.png)

当`checkStrictly`值是`false`时，对应方法的入参`leafOnly`就是`true`，此时会去`leafNodes`中去查找数据，但是！！！！这个`leafNodes`中没有这个`Option-4`节点。为什么？

![image-20230914180902899](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914180902899.png)

![image-20230914180917813](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914180917813.png)

其实这个`Option-4`节点在一开始是父节点，它并不会添加到`leafNodes`数组中，只会添加到`allNodes`数组中。当点击该节点时，由于没加载到数据，所以会将其身上的`leaf`属性变成`true`，关键就在这儿，属性变成`false`了但是这个节点没有给它添加到`leafNodes`中，所以在从`leafNodes`中获取时就获取不到节点。因此当节点的`leaf`变成`true`时应该将其放到`leafNodes`中，这里我们简单处理一下，因为在`leafNodes`中获取不到数据，所以我们考虑从`allNodes`中获取数据，操作也很简单，把`getCheckedNodes`中的变量去掉，让`leafOnly`的值变成`false`，这样就会从`allNodes`里面去获取数据。实际这么改肯定是不行的，这里也只是为了验证问题。

![image-20230914180530796](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230914180530796.png)

![chrome-capture-2023-8-14-a](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-8-14-a.gif)

可以看到数据可以正常选中并显示了。

以上便是本次对`element-plus`中级联组件问题的探究。
