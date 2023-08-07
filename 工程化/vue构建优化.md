<!--
 * @Author: Zhouqi
 * @Date: 2023-07-12 15:55:39
 * @LastEditors: Zhouqi
 * @LastEditTime: 2023-07-12 15:55:40
-->


# vue项目构建优化



## 一、背景

某次，在调整一个项目代码之后发现持续集成构建项目的时间非常久，大概在10-15分钟之间，以当时该项目的复杂度是远不需要这么久的时间。由此问题，我猜测该项目并没有涉及过多的性能优化，包括首屏加载等等。于是我对该项目的首屏加载进行性能测试，发现其中确实存在一些问题。借此机会，我尝试对整个项目进行一次改造优化。



## 二、改造

### 1. 声明

所有的性能优化方案都是在出现性能瓶颈的时候才会去采用，无缘无故的优化可能最终会导致负优化。例如多线程打包优化，多线程打包优化的特点是开启多核cpu，通过多核并行的方式去执行打包任务，从而显著提升打包速度。开启多线程会消耗一定的资源，当开启多线程所消耗的资源更多时，这种优化就属于负优化。

并行处理的性能模型中有一个阿姆达定律，如下图所示。横坐标代表处理器数量，类比开启的cpu核数。纵坐标表示处理速度，类比打包速度。四条曲线表示程序中并行任务所占总任务数的比例。对比图中的四条曲线可以发现，当程序中存在的并行任务越多时，多核处理器的效果会更加显著。并且四条曲线都有一个共同特性，随着处理器的不断增加，程序执行速度会先上升再趋于平稳。这个特性意味着并不是开启越多的cpu，打包速度就会越快。并行优化的一个限制点就是任务必须是并行执行的，如果任务是串行执行的，那么无论开启多少cpu都不会有任何优化，甚至负优化。这也解释了图中曲线为什么最终会趋于平稳，因为并行任务的优化已经达到顶点，剩下的串行任务是无法通过多核处理器并行优化的。

![image-20230327105334652](/Users/scout/Library/Application Support/typora-user-images/image-20230327105334652.png)

因此，盲目的性能优化只会增加程序的复杂度，代码的复杂度，需要多加以思考。



### 2. 架构升级

分析该项目的结构，发现项目用的构建工具是webpack，并且webpack的版本非常低，当时最高版本已经是webpack5了。

![image-20230712154213055](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154213055.png)

查看一下package.json中的依赖信息，发现项目中使用的外部依赖非常多，大概64个，是否真的需要这么多依赖还有待分析。

![image-20230327111802293](/Users/scout/Library/Application Support/typora-user-images/image-20230327111802293.png)

所以第一步的优化就是升级webpack（具体到什么版本看具体情况，一般都是升级到最高版本）。我们可以采用最新的vue-cli脚手架去创建一个webpack5的项目。

```javascript
vue create hello-world
```

项目创建完成后把一些老的文件搬运过去。

![image-20230712154252294](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154252294.png)

分析老项目package.json中的的依赖，将需要的依赖添加到新项目中，无用的直接抛弃（项目中移除了大量*devDependencies*下的依赖）。并且部分依赖的版本可能较低，在项目改动不大的情况下可以尝试升级版本。（插件的升级有可能会影响到之前的代码，在升级的过程中我们需要对源代码也进行一定的改动）



### 3. 构建产物以及首屏加载优化

webpack中常用的包分析工具主要是webpack-bundle-analyzer，它使用交互式可缩放树图来可视化webpack输出文件的大小。首屏加载优化主要借助lighthouse进行分析，lighthouse 是一个网站性能测评工具， 它是 Google Chrome 推出的一个开源自动化工具，能够对 [PWA](https://so.csdn.net/so/search?q=PWA&spm=1001.2101.3001.7020) 和网页多方面的效果指标进行评测，并给出最佳实践的建议以帮助开发者改进网站的质量。我们先跳过包分析这一步，直接使用lighthouse进行分析。

首先，对项目进行一次构建，将构建产物放入开发环境中进行性能测试，结果如下。从分析报告上可以看到页面在首屏加载的时候是存在一部分问题的。对于这些问题，ligthhouse会解释原因以及对应的解决方案。

![image-20230712154531131](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154531131.png) 

![image-20230712154548343](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154548343.png)  															

当我们点击learn more的时候会打开一个网页，网页中的内容涉及到该问题相关的性能知识。这个网站属于chrome官方网站，里面有一系列非常优秀的文章，大部分都是相关领域的专家所写。

![image-20230712154706872](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154706872.png) 

![image-20230712154729483](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154729483.png)

借助lighthouse给出的报告以及相关文章，我们可以解决很多性能问题，这里主要挑几个常见的问题进行优化。



### 3.1 资源大小优化

性能报告中显示，在资源加载的过程中，部分资源的传输体积较大，特别是chunk-venderos文件，超过了1MB。虽然有未开启gzip压缩的原因，但原始体积也是过大的，因此需要优化。

![image-20230327164118310](/Users/scout/Library/Application Support/typora-user-images/image-20230327164118310.png)

前面提到过，对于包体积分析，可以借助webpack-bundle-analyzer。

```javascript
 const {
   BundleAnalyzerPlugin
 } = require('webpack-bundle-analyzer');
  configureWebpack: (config) => {
    if(prod) {
      config.plugins.push(new BundleAnalyzerPlugin())
    }
  }
```

通过包分析工具我们可以看到chunk-vendors这个文件包含了大量的内容，其中有个别包甚至是全量引入的，比如iview和lodash，这里先对这两个包做优化处理。

![image-20230327183004607](/Users/scout/Library/Application Support/typora-user-images/image-20230327183004607.png)

首先在plugins目录里面新建iview.js，并添加项目中需要用到的组件。

```javascript
// iview.js
import Vue from 'vue';
import {
    Button,
    Icon,
    Dropdown,
    DropdownMenu,
    DropdownItem,
    Spin,
    Page,
    Table,
    Poptip,
    Checkbox,
    CheckboxGroup,
    Input,
    Option,
    Select,
    Message,
    Modal,
    Progress
} from 'iview';

const iview = [
    Button,
    Icon,
    Dropdown,
    DropdownMenu,
    DropdownItem,
    Spin,
    Page,
    Table,
    Poptip,
    Checkbox,
    CheckboxGroup,
    Select,
    Input,
    Option,
    Modal,
    Progress
];

iview.forEach(el => {
    Vue.component(el.name, el);
});
Vue.prototype.$Message = Message
```

在main.js里面引入iview.js，并把之前全局引入的代码删除。

```javascript
- import iView from 'iview';
- Vue.use(iView);
+ import '@/plugins/iview';
```

最后在babel.config.js中添加iview的按需加载配置。

```javascript
  module.exports = {
  presets: [
    '@vue/cli-plugin-babel/preset'
  ],
  plugins: [
    "transform-vue-jsx",
    "@babel/plugin-transform-runtime",
    // ……省略其它内容
    ["import", {
      "libraryName": "iview",
      "libraryDirectory": "src/components"
    }]
  ]
}
```

打包一次看看效果

![image-20230327184328360](/Users/scout/Library/Application Support/typora-user-images/image-20230327184328360.png)

对比一下优化前的包体积，效果是显著的。																														

![image-20230327184517967](/Users/scout/Library/Application Support/typora-user-images/image-20230327184517967.png)	

接下去是loadsh的优化，这里推荐使用lodash-es代替lodash，lodash-es使用的是esmodule的方式，对tree-shaking更加友好。

首先安装lodash-es，utils里面新建lodash.js，将项目中用到的lodash方法全部从这里引入并导出，方便统一管理和扩展。

```javascript
// lodash.js
import {
    debounce
} from 'lodash-es';

export default {
    debounce
}
// 另外的写法，取决于之前项目中是怎么使用lodash的
// 比如import _ from 'lodash'; _.debounce();这种的用上面这种方式改动就比较小。
// import { debounce } from 'lodash';这种的通过下面这种方式就行。
// export { debounce, isEqual } from 'lodash-es';
```

再打包一次看看效果

![image-20230327191043681](/Users/scout/Library/Application Support/typora-user-images/image-20230327191043681.png)

对比一下之前的，效果也是显著的。

![企业微信截图_1032363c-eeca-4678-8f5e-3fb5e9713b67](/Users/scout/Library/Containers/com.tencent.WeWorkMac/Data/Documents/Profiles/8BB4FE9E7FA06295D8E5F68B8FCBDD31/Caches/Images/2023-03/2996eaa72da7c6e22476cd39bff1d5a0_HD/企业微信截图_1032363c-eeca-4678-8f5e-3fb5e9713b67.png)

将上面优化后的包放到开发环境再用lighthouse测试一次。

![image-20230712154834758](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154834758.png) 

![image-20230712154850611](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154850611.png) 

对比优化前的，还是有效果的。

![image-20230712154924944](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154924944.png) 

![image-20230712154948120](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712154948120.png) 

上面通过资源按需打包的方式，实现了一定的优化。但是chunk-vendors体积还是略大，因为大部分的依赖插件都会安装到node_modues里面，导致node_modules里面有很多的内容，像vue，vuex，elementui等等。另外，由于nodule_modules里面可能会存在一些经常变动的插件，使得整个node_modules在打包的时候会生成一份新的hash文件，近而导致缓存失效，缓存失效后又会去请求全新的chunk-vendors大文件。显然，类似vue这种框架级别的包变动频率不大，因此需要将这些包提取出来，单独使用缓存策略，避免受到其它插件变动带来的影响。



### 3.2 分包策略

这里采用splitChunks来配置分包策略，提取一些个人认为不会经常变动，并且包体积相对较大的包。这里要注意权重问题，由于这些包是包含在node_modules里面的，所以整个node_modules的策略权重应该比内部其它包的策略低，否则就会提取不出来。

```javascript
// vue.config.js
// 省略其它配置
chainWebpack(config) {
    if(prod) {
      config.optimization.splitChunks({
        cacheGroups: {
          vendors: {
            name: 'chunk-vendors',
            test: /[\\/]node_modules[\\/]/,
            chunks: 'all',
            priority: 2,
          },
          vueQr: {
            name: 'vue-qr',
            test: /[\\/]node_modules[\\/]vue-qr[\\/]/,
            chunks: 'async',
            priority: 3,
          },
          elementUI: {
            name: 'element-ui',
            test: /[\\/]node_modules[\\/]element-ui[\\/]/,
            chunks: 'all',
            priority: 3,
          },
          iview: {
            name: 'iview',
            test: /[\\/]node_modules[\\/]iview[\\/]/,
            chunks: 'all',
            priority: 3,
          },
          // lodashES: {
          //   name: 'lodash-es',
          //   test: /[\\/]node_modules[\\/]lodash-es[\\/]/,
          //   chunks: 'all',
          //   priority: 3,
          // },
          vue: {
            name: 'vue',
            test: /[\\/]node_modules[\\/]vue[\\/]/,
            chunks: 'all',
            priority: 3,
          },
          vueRouter: {
            name: 'vue-router',
            test: /[\\/]node_modules[\\/]vue-router[\\/]/,
            chunks: 'all',
            priority: 3,
          },
          vuex: {
            name: 'vuex',
            test: /[\\/]node_modules[\\/]vuex[\\/]/,
            chunks: 'all',
            priority: 3,
          },
        }
      });
    }
  },
```

配置完成后再次打包。从图上可以看到，这些插件已经被单独提取成一份文件，并且chunk-vendors的体积小了很多。接下去再放到开发环境进行测试。

![image-20230327194317997](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230327194317997.png)

从打包结果上看还是有一点点提升的。

![image-20230327195339728](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230327195339728.png)



### 3.3 首屏关键资源提取以及预加载 

eliminate render-blocking resources，意思是消除阻塞渲染的资源。通过详细面板可以看到有三份css资源阻塞了渲染，css阻塞渲染很容易理解，在前端渲染的关键路径中需要先生成DOM树和CCSOM树，接着还要构建Render树、Layout树、分层等等。

![image-20230712155050711](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712155050711.png)

通过Coverage面板可以查看资源使用情况。点击reload按钮之后会生成一份资源使用列表，红色表示未使用的部分，绿色表示使用的部分。通过Coverage面板可以看到阻塞渲染那部分的css其实大部分都没有使用到，对于完全没有使用到的css应该将它从首屏加载中移除。对于非关键的css资源（这里可以理解为未使用部分比较多的css资源），首选css的提取，将首屏需要使用的css代码提取出来，通过内联的方式去加载，剩下的非关键部分通过异步加载的方式去获取（非阻塞）。内联的好处就是不需要发起网络请求，坏处就是可能会延长DOM解析的时间，需要权衡利弊。

![image-20230328112544400](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230328112544400.png)

关键资源的提取可以使用第三方工具实现，比如html-critical-webpack-plugin，也可以自己手动提取。

```javascript
  // vue.config.js
  configureWebpack: (config) => {
    // 自动提取首屏css和非关键css资源
    // 本地打包正常，持续集成构建时失败 https://github.com/anthonygore/html-critical-webpack-plugin
    // https://github.com/puppeteer/puppeteer/blob/main/docs/troubleshooting.md 
    config.plugins.push(new HtmlCriticalWebpackPlugin({
      base: path.resolve(__dirname, 'dist'),
      src: 'index.html',
      dest: 'index.html',
      inline: true,
      minify: true,
      extract: true,
      width: 1920,
      height: 900,
      penthouse: {
        blockJSRequests: false,
      }
    }));
  },
```

再次打包后发现，生成的html比之前多了style样式（关键CSS资源），并且其它非关键css资源都加上了rel=preload，即预加载。预加载的作用就是让浏览器知道一些较晚会被发现的资源并提前去加载它们。比如当我们提取了关键css和非关键css之后，关键的css会内联到head中，非关键的css可能会通过script脚本去动态获取（比如通过滚动屏幕的方式动态加载剩余样式）。如果资源在脚本执行的时候才去加载，那么就有可能造成一小段时间的空白期（不显示样式），尤其在网络波动比较大的时候。因此，尽可能提前去加载这些资源并缓存起来，在执行脚本的时候就可以直接使用缓存来加快页面渲染。

![image-20230328115454403](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230328115454403.png)

![image-20230328115822632](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230328115822632.png)

再次进行打包测试可以看到之前的render-blocking的问题已经解决了。

![image-20230328140550468](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230328140550468.png)

通过html-critical-webpack-plugin可以实现关键css的提取以及非关键css资源的加载。但是经过测试，这个插件在持续集成构建执行的时候有报错，导致构建失败，具体原因在上图配置代码部分有说明。以下是另外一种实现预加载的方式——@vue/preload-webpack-plugin。

```javascript
// vue.config.js
 config
        .plugin('preload')
        .use(PreloadPlugin, [{
          as(entry) {
            //资源类型
            if(/\.css$/.test(entry)) return 'style';
            if(/\.woff$/.test(entry)) return Ï'font';
            if(/\.png$/.test(entry)) return 'image';
            return 'script';
          },
          include: 'initial',
          rel: 'preload',
          fileBlacklist: [/\.js/, /\.png/, /\.svg/],
        }]);
```

除了上面所说的css资源预加载，还有一种资源的加载也经常会用到预加载——web font字体文件。字体文件一般作为关键资源在最后才会被请求，如下图所示，字体文件的请求排在了关键css、js请求之后。字体通常是加载时间较慢的大文件。一些浏览器在字体加载之前隐藏文本，导致布局偏移（CLS）以及不可见文本闪烁。

![](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230328140550468.png)

布局偏移量（累计位移偏移）是前端页面性能中的一项指标，指的是页面上非预期的位移波动。比如img元素如果不指定宽高占位，当获取到图片后容器大小会发生突变，会导致布局内容发生偏移等问题。lighthouse报告中也会有相关提示，解决方法就是指定width和height占位。

![image-20230329115326896](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230329115326896.png)

```vue
- <img src="@/resource/images/review.png">
+ <img src="@/resource/images/review.png" width="133" height="86" >
```

字体图标也是类似的道理，当获取到字体文件资源后，浏览器会将后备字体与新字体交换，导致无样式的文本闪烁；新字体渲染到页面前显示不可见文本，导致不可见文本闪烁。

在lighthouse性能报告上也可以看到相关的提示，报告中说利用字体显示CSS功能确保文本在加载网络字体时用户可见。因此需要尽可能快地获取到字体文件资源，并且通过font-display属性避免加载阶段显示不可见的文本。

![image-20230329133110784](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230329133110784.png)

下面使用预加载以及font-display的方式来优化字体文件的加载，这里不考虑第三方包中字体的处理（ionicons.woff2和element-icons.woff）

```css
/* iconfont.css */
@font-face {
  /* 省略部分代码 */
  font-display: swap; /* 当字体文件处于加载中时使用系统默认字体 */
}
```

```html
<!-- index.html -->
<!-- 省略其它代码 -->
<link rel="preload" href="./fonts/iconfont.822bf508.ttf" as="font" type="font/ttf" crossorigin>
```

打包后再次测试发现，对应字体资源请求的优先级已经提升了，并且性能报告中关于该资源的异常也没有了。

![image-20230712155252654](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230712155252654.png)

![image-20230329134819032](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230329134819032.png)



### 3.4 页面组件按需加载

代码中的invitation-modal是一个弹窗组件，这个组件在首屏加载的时候是不需要显示的，如果不做任何按需处理，则这个组件会被直接打包进主文件中，增加主文件体积。因此这个弹窗组件可以在需要的时候再进行加载，比如触发显示弹窗的事件内去获取，并结合component :is来实现组件的按需加载。

代码中的webpackPrefetch的意思是通过prefetch的方式在浏览器空闲的时候去加载弹窗组件对应的资源并缓存在浏览器中，这样就能在弹窗显示事件触发的时候更快地获取到弹窗组件的内容。不管是preload还是prefetch都是为了在将来要用到资源的时候可以尽快获取内容，当然prefetch的资源获取优先级和时机都不如preload高。但是，假如某个组件被用到的概率很低，那就不需要提前去获取，毕竟加载一个大概率不会被用到的资源也是一种浪费。

webpack在打包的时候会对这种动态import的组件单独生成一份文件，这样做的好处在于可以减小主文件的体积，将非关键部分代码提取出来做异步加载。

```javascript
// Partner.vue
const getInvitationModal = () => import(
  /* webpackChunkName: 'invitationModal' */
  /* webpackPrefetch: true */
  "./invitation-modal");

// click事件
   openInvitationModal(type) {
      !this.invitationModal && getInvitationModal().then(comp => {
        this.invitationModal = comp.default;
      });
    },
      
// template
 <component :is="invitationModal" v-if="showModal" :showModal="showModal" @close="showModal = false"></component>
```

在other面板中可以看到弹窗组件的js加载，它的加载优先级是非常低的。并且当我们点击按钮打开弹窗时，弹窗组件的js文件是从prefetch cache中获取的。

![image-20230328145534342](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230328145534342.png)

![image-20230328150913982](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230328150913982.png)



### 3.5 cdn优化

像vue、vue-router、vuex等资源还可以采用cdn引入的方式，毕竟cdn很快。在构建阶段可以通过externals排除这些包的构建，加快构建速度，在没有做分包的情况下也能减小主文件体积。

```javascript
// vue.config.js
const cdns = [
  'https://cdn.bootcdn.net/ajax/libs/vue/2.6.14/vue.min.js',
  'https://cdn.bootcdn.net/ajax/libs/vue-router/3.0.6/vue-router.min.js',
  'https://cdn.bootcdn.net/ajax/libs/vuex/3.1.2/vuex.min.js',
  'https://cdn.bootcdn.net/ajax/libs/axios/0.19.0/axios.min.js'
];
config.plugin('html').tap(args => {
        args[0].cdns = cdns
        return args
      });
      config.externals({
        vue: 'Vue',
        'vue-router': 'VueRouter',
        vuex: 'Vuex',
        axios: 'axios'
      });
```

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="">

<head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width,initial-scale=1.0">
  <link rel="icon" href="<%= BASE_URL %>favicon.ico">
  <title>
    <%= htmlWebpackPlugin.options.title %>
  </title>
   <!-- 新增部分 -->
  <% for (var i in htmlWebpackPlugin.options.cdns) { %>
    <script defer src="<%= htmlWebpackPlugin.options.cdns[i] %>"></script>
    <% } %>
</head>

<body>
  <noscript>
    <strong>We're sorry but <%= htmlWebpackPlugin.options.title %> doesn't work properly without JavaScript enabled.
        Please enable it to continue.</strong>
  </noscript>
  <div id="app"></div>
  <!-- built files will be auto injected -->
</body>

</html>
```

再打包一次可以发现，在生成的html中加上了指定资源的cdn引入。

![image-20230329142459438](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230329142459438.png)



## 三、结束

以上就是项目中涉及到的其中一部分优化。当然，优化手段还有很多，关键需要分析哪里有性能问题，对症下药。lighthouse在一定程度上能够帮助我们找到一些常规问题，也能提供一些解决问题的思路。但是涉及到业务代码逻辑的优化，比如如何设计组件、拆分组件、按需引入组件等需要靠经验积累。

最后，如果对性能优化有不了解的地方，建议多点点learn more。

![image-20230328163123326](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230328163123326.png)
