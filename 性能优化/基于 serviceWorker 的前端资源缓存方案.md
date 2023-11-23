# 基于 ServiceWorker 的前端资源缓存方案

### 一、什么是 ServiceWorker

Service Worker 是一种在用户浏览器后台运行的脚本，它独立于网页，使得应用能够在离线时运行或者在网络不佳的情况下提供内容。它是一种网络代理，可以控制网页能够访问的网络请求，从而实现如缓存资源、推送通知、后台同步等功能。



### 二、ServiceWorker 的生命周期

Service Worker的生命周期主要包括以下几个阶段：

1. **注册（Registration）**：在主线程中，使用 `navigator.serviceWorker.register` 方法注册 Service Worker。如果注册成功，Service Worker 就会进入安装阶段。
2. **安装（Installation）**：在这个阶段，Service Worker 会执行安装事件的回调函数，通常在这个回调函数中缓存一些资源。如果所有资源都成功缓存，Service Worker 就会进入等待阶段，否则安装失败。
3. **等待（Waiting）**：在这个阶段，新的 Service Worker 等待旧的 Service Worker 结束。如果没有旧的 Service Worker，或者旧的 Service Worker 已经被终止，新的 Service Worker 就会进入激活阶段。
4. **激活（Activation）**：在这个阶段，Service Worker 会执行激活事件的回调函数，通常在这个回调函数中清理旧的缓存。激活后，Service Worker 就可以控制所有打开的页面。
5. **空闲（Idle）**：当 Service Worker 不需要处理任务时，它会进入空闲状态。在空闲状态下，Service Worker 可以被浏览器终止，以节省内存。
6. **终止（Terminated）**：为了节省内存，浏览器会在 Service Worker 空闲时终止它。当有事件需要处理时，浏览器会重新启动 Service Worker。
7. **更新（Update）**：当注册 Service Worker 的脚本有任何改变时，浏览器会尝试更新 Service Worker。如果更新成功，新的 Service Worker 就会进入安装阶段。



### 三、实现

1. ##### 定义目录结构

   在原项目中创建一个新的目录，用于维护 ServiceWorker 相关的文件。
   ![image-20231122154103723](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231122154103723.png)

2. ##### 实现注册过程（Registration）

   在 index.js 中监听页面`load`事件，在事件中调用`navigator.serviceWorker.register`并传入 ServiceWorker 文件地址，等待注册。如果注册成功会提示成功信息，如果失败则提示失败信息。

   ```javascript
   // index.js
   if('serviceWorker' in navigator) {
     window.addEventListener('load', () => {
       navigator.serviceWorker.register('./serviceWorker.js').then((registration) => {
    			console.log('sw registered success');
       }).catch(err => {
         console.log('sw registered error：', err);
       })
     });
   }
   ```

   在 main.js 中引入这个 index.js。

   ```javascript
   // main.js
   import './pwa';
   ```

   运行程序后发现 ServiceWorker 注册失败了。
   ![image-20231122155549979](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231122155549979.png)
   从报错信息可以看到 ServiceWorker 文件的请求地址是 http://localhost:8080/serviceWorker.js，但是项目中这个文件是放在 pwa 目录下的，因此资源访问不到。这里借助`workbox-webpack-plugin`插件实现，`workbox-webpack-plugin` 是一个 webpack 插件，它是 Google 开发的 Workbox 工具库的一部分。这个插件可以帮助你在 webpack 的构建过程中，自动生成 Service Worker 文件。

   `workbox-webpack-plugin` 提供了两个主要的类，`GenerateSW` 和 `InjectManifest`：

   1. **GenerateSW**：这个类会在每次构建结束后，自动生成一个新的 Service Worker 文件，其中包含了预缓存的资源列表。这个类适合于那些希望 Workbox 自动生成并维护 Service Worker 文件的项目。
   2. **InjectManifest**：这个类允许你提供一个自定义的 Service Worker 源文件，然后在构建过程中，将预缓存的资源列表注入到这个源文件中。这个类适合于那些需要更多控制权，希望自己编写 Service Worker 文件的项目。

   这里我们主要使用`InjectManifest`的功能。

   ```javascript
   // vue.config.js
   config.plugins.push(new WorkboxPlugin.InjectManifest({
     swSrc: './src/pwa/serviceWorker.js',
     swDest: 'serviceWorker.js',
     exclude: [
       /\.map$/,
       /manifest$/,
       /serviceWorker\.js$/
     ]
   }));
   ```

   `InjectManifest`中用到了三个属性：

   - **swSrc**：这是 Service Worker 文件的路径。`InjectManifest` 插件会读取这个文件，并在其中注入预缓存的资源列表。
   - **swDest**：这是生成的 Service Worker 文件的输出路径。`InjectManifest` 插件会将处理后的 Service Worker 文件输出到这个路径。
   - **exclude**：这是一个数组，包含了你不想被预缓存的资源的模式。`InjectManifest` 插件会将匹配这些模式的资源排除在预缓存的资源列表之外。

   这里先不考虑注入依赖的功能，只考虑让插件在根目录下能生成一份 Service Worker 文件，重启项目后可以看到 Service Worker 注册成功了。

   ![image-20231122160608916](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231122160608916.png)

3. ##### 实现安装过程（Installation）

   安装过程会触发 install 事件，在这个事件中一般会进行资源的预缓存。在 Service Worker 中进行资源缓存可分为两种，一种是预缓存，另一种是运行时缓存。

   - **预缓存**：预缓存是在 Service Worker 安装阶段，将需要的资源（如 HTML、CSS、JavaScript、图片等）提前请求并存储到缓存中。这样，即使在无网络或网络不稳定的情况下，这些资源也可以被立即从缓存中获取，提高应用的加载速度和可用性。
   - **运行时缓存**：运行时缓存是在 Service Worker 激活后，对用户实际请求的资源进行缓存。例如，可以对 API 请求的结果进行缓存，以便在网络不可用时提供离线数据。

   总的来说，预缓存和运行时缓存是互补的，它们结合使用可以提供最佳的性能和离线体验。

   ![image-20231122163313941](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231122163313941.png)
   在 netWork 面板中可以看到程序运行时加载的 js，我们可以定义一个缓存列表来告诉 Service Worker 需要缓存的资源，这是一种手动添加资源列表的方式。

   ```javascript
   // serviceWorker.js
   const CACHE_NAME = 'cache';
   self.addEventListener('install', event => {
     event.waitUntil((() => {
       const preloadUrlToCache = [
         'chunk-vendors.js',
         'app.js', 'index.html',
         'src_components_BtnGroup_vue-src_components_ShareCard_vue-src_utils_filters_js.js',
         'src_mixins_fileMixin_js.js',
         'partner.js'
       ];
       return caches.open(CACHE_NAME)
         .then(cache => cache.addAll(preloadUrlToCache));
     })());
   });
   ```

   还有一种自动注入静态资源依赖的方式，就是注册阶段介绍的`workbox-webpack-plugin`，它会记录项目构建之后的所有静态资源，并将资源列表存到`__WB_MANIFEST`这个属性中。

   ```javascript
   // serviceWorker.js
   const CACHE_NAME = 'cache';
   self.addEventListener('install', event => {
     event.waitUntil((() => {
       const preloadUrlToCache = self.__WB_MANIFEST.map(item => item.url);
       return caches.open(CACHE_NAME)
         .then(cache => cache.addAll(preloadUrlToCache)).then(() => {
           return self.skipWaiting();
         });
     })());
   });
   ```

   我们打印一下`self.__WB_MANIFEST`，可以看到需要预缓存的资源列表。这种方式的好处在于我们无需手动去维护资源列表，毕竟当项目构建之后文件名可能会带有 hash 后缀，这类文件很难通过手动的方式去维护，并且极有可能出错。

   ![image-20231122173034383](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231122173034383.png)

   `event.waitUntil()` 是 Service Worker 生命周期事件中的一个方法。它用于在 Service Worker 的 `install` 和 `activate` 事件中延长事件的寿命。

   在 Service Worker 中，`install` 和 `activate` 事件是异步的，这意味着浏览器不会等待这些事件完成就结束它们。如果我们在这些事件中进行一些异步操作，如缓存资源或清理旧的缓存，我们需要确保这些操作在事件结束前完成。这就是 `event.waitUntil()` 的作用。

   `event.waitUntil()` 接受一个 Promise 作为参数。只有当这个 Promise 完成（resolve）时，浏览器才会认为 `install` 或 `activate` 事件已经完成。如果 Promise 被拒绝（reject），浏览器会认为安装失败，Service Worker 将不会被激活。

   代码中创建了一个缓存列表`preloadUrlToCache`，里面有需要缓存的文件列表。`caches.open()` 是 Cache API 的一部分，它用于打开一个指定的缓存（CACHE_NAME）。这个方法返回一个 Promise，当 Promise 完成时，它会返回一个 Cache 对象，你可以使用这个对象来读取、添加或删除缓存中的资源。

   这里会使用`cache.addAll`方法缓存资源，`cache.addAll()` 是 Cache API 的一部分，它用于将一组资源添加到缓存中。这个方法接受一个 URL 数组作为参数，然后发起对这些 URL 的请求，将请求的结果添加到缓存中。

   `cache.addAll()` 返回一个 Promise，只有当所有的资源都成功添加到缓存中时，这个 Promise 才会完成（resolve）。如果任何一个资源无法被添加到缓存中（例如请求失败），这个 Promise 就会被拒绝（reject）。

4. ##### 实现请求过程（fetch）

   在 Service Worker 中，`fetch` 事件是用来拦截和处理网络请求的。当页面发起网络请求时，如果 Service Worker 已经激活并控制了这个页面，那么 `fetch` 事件就会被触发。

   在 `fetch` 事件的回调函数中，可以使用 `event.respondWith()` 来控制如何响应这个请求。例如，你可以从缓存中获取资源，或者发起一个新的网络请求，或者生成一个自定义的响应。

   ```javascript
   // serviceWorker.js
   self.addEventListener('fetch', event => {
     const request = event.request;
     const url = request.url.split('?')[0];
     event.respondWith(
       caches.match(url).then(response => {
         // 缓存存在，使用缓存
         if(response) return response;
         // 缓存不存在利用fetch发起网络请求
         const fetchRequest = event.request.clone();
         return fetch(fetchRequest).then(response => {
           // 这些类型的请求直接返回网络请求结果，不需要被缓存
           if(!response || response.status !== 200 || response.type !== 'basic' || !shouldCache(request)) return response;
           // 请求成功的资源直接放入缓存中
           const responseToCache = response.clone();
           caches.open(CACHE_NAME)
             .then(cache => {
               // 添加新的缓存
               cache.put(url, responseToCache)
             });
           return response;
         })
       })
     );
   });
   const shouldCacheUrl = (url) => {
     const excludeSource = ['chrome-extension'];
     for(let i = 0; i < excludeSource.length; i++) {
       const exclude = excludeSource[i];
       if(~url.indexOf(exclude)) return false;
     }
     return true;
   }
   
   const shouldCacheDestination = (destination) => {
     const cacheList = ['image', 'document', 'script'];
     return cacheList.includes(destination);
   }
   
   const shouldCache = ({ url, destination }) => shouldCacheUrl(url) && shouldCacheDestination(destination);
   ```
   
   在 fetch 事件中我们可以自由控制需要缓存哪些资源，哪些资源不需要缓存处理。刷新页面后可以看到静态资源已经是从 Service Worker 缓存中读取的了。
   
   ![image-20231122173408491](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231122173408491.png)
   
   不过有些开发者可能在本地开发的时候会遇到一些问题，当二次刷新浏览器或者更新本地代码后刷新浏览器会出现报错或者浏览器无限刷新的情况。
   
   ![image-20231123094641237](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231123094641237.png)
   
   ![chrome-capture-2023-10-23_副本](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-10-23_%E5%89%AF%E6%9C%AC.gif)
   
   出现这些问题的原因还是缓存文件的问题，比如无限刷新的原因是，当代码更新后会触发 vue 的热更新机制，此时会生成一份新的带 hash 前缀的 hot-update.json 文件。浏览器刷新后会读取热更新文件，但是由于读取逻辑是老的（这个读取文件从缓存中获取），导致一直在读取老的 hot-update.json 文件，如果没读到就会触发浏览的 reload，一直循环如此就会造成无限刷新的问题。代码报错也是类似的问题，有兴趣可以自行调试看看原因。
   
   
   
5. ##### 实现激活过程（Activation）

   当资源更新通常需要构建新的缓存并清除老的缓存，新的缓存构建依旧是在 `install` 事件中执行，也就是之前讲的预构建。旧缓存的清除则需要放到 `activate` 事件执行，不能再 `install` 事件中执行。因为旧的 Service Worker 可能仍然在控制一些页面，并且依赖这些缓存来正常工作。如果在这个时候清理缓存，可能会导致这些页面出现问题。

   在 Service Worker 的生命周期中，当新的 Service Worker 安装完成后，它会进入等待状态，直到所有的页面都不再被旧的 Service Worker 控制。只有当所有的页面都开始使用新的 Service Worker，新的 Service Worker 才会激活，`activate` 事件才会被触发。

   因此，当 `activate` 事件被触发时，可以确保所有的页面都已经开始使用新的 Service Worker，旧的 Service Worker 已经不再控制任何页面。这就是为什么可以在 `activate` 事件中安全地清理老的缓存，而不会影响旧的 Service Worker 的工作。

   但是，如果应用允许多个版本的 Service Worker 同时运行，或者 Service Worker 使用了 `self.skipWaiting()` 来跳过等待状态，那么需要更小心地处理缓存的清理，以避免影响正在运行的 Service Worker。

   ```javascript
   // serviceWorker.js
   const CACHE_NAME = 'cache';
   self.addEventListener('activate', function (event) {
     // 资源有更新时删除旧缓存名称对应的缓存资源
     event.waitUntil(
       caches.keys().then(cacheNames => {
         return Promise.all(
           cacheNames.map(cacheName => {
             if(cacheName !== CACHE_NAME) {
               return caches.delete(cacheName);
             }
           })
         );
       })
     );
   });
   ```

   这里还需要考虑一个点，当资源更新后 `CACHE_NAME` 也需要更新。如果不更新 `CACHE_NAME`，那么新的 Service Worker 将会使用同一个缓存，这可能会导致以下问题：

   1. 如果新的 Service Worker 尝试缓存新的资源，但是老的资源仍然在缓存中，那么用户可能会看到老的或者不一致的内容。
   2. 如果新的 Service Worker 删除了一些资源，但是这些资源仍然被老的 Service Worker 使用，那么这可能会导致错误或者应用无法正常工作。

   因此，在清除缓存时，需要获取所有的缓存名，然后删除老的缓存名对应的缓存，只保留新的。考虑到每次发布更新时需要创建新的 `CACHE_NAME`，这里采用打包构建时动态注入缓存名的方式。

   

   在配置文件中使用`DefinePlugin` 在环境变量中增加 `CACHE_NAME` 变量，值为 cache 加时间戳的方式，这样每次构建时都会生成一个全新的名字。

   ```javascript
   // vue.config.js
   config.plugins.push(new webpack.DefinePlugin({
     'process.env.CACHE_NAME': JSON.stringify('cache' + Date.now())
   }));
   ```

   在 serviceWorker.js 中使用该环境变量

   ```javascript
   // serviceWorker.js
   const CACHE_NAME = process.env.CACHE_NAME;
   ```

   这样就完成了整个资源更新的逻辑。

6. ##### 实现通知功能

   当资源更新时，第一次打开浏览器可能会加载到老的资源，由于老的 Service Worker 还在工作，新的 Service Worker 还在注册中，需要二次刷新页面才能取到新的资源。为了能及时让用户获取到新的资源，还需要实现一个通知的功能。新的 Service Worker 更新激活时提示用户有资源更新，需要重新刷新页面。

   这个功能可以利用 `controllerchange` 事件实现：

   ```javascript
   // pwa/index.js
   import { MessageBox } from "element-ui";
   if('serviceWorker' in navigator) {
     let newWorker;
     const showUpdateNotification = () => {
       MessageBox.confirm('检测到新版本，是否更新？', '提示').then(() => {
         // 执行更新操作
         window.location.reload();
       });
     }
     // 当 Service Worker 状态变化时，检查是否有新的 Service Worker 可用
     navigator.serviceWorker.addEventListener('controllerchange', function () {
       if(newWorker && newWorker.state === 'activating') {
         // 新的 Service Worker 已经激活，显示更新提示
         showUpdateNotification();
       }
     });
   
     window.addEventListener('load', () => {
       navigator.serviceWorker.register('./serviceWorker.js').then((registration) => {
         registration.addEventListener('updatefound', () => {
           newWorker = registration.installing;
         });
       }).catch(err => {
         console.log('sw registered error：', err);
       })
     });
   }
   ```

   当重新打开页面时就会出现提示，点击确定后会再次刷新页面，此时就能获取到新的资源。

   ![image-20231123115909189](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20231123115909189.png)



### 四、Service Worker 的缺点

Service Worker 的确有很多优点，比如离线访问、资源缓存、后台同步等，但是也有一些潜在的缺点：

1. 兼容性问题：虽然大部分现代浏览器都支持 Service Worker，但是一些老的或者不常见的浏览器可能不支持。
2. 复杂性：Service Worker 的生命周期和工作机制相对复杂，需要花费一些时间和精力去理解和管理。
3. 缓存问题：如果不正确地使用 Service Worker 的缓存，可能会导致用户看到的是过期的或者不一致的内容。
4. 安全性：Service Worker 只能在 HTTPS 环境中工作，这是因为 Service Worker 有能力拦截和修改网络请求，如果在不安全的环境中使用，可能会被攻击者利用。
5. 更新延迟：由于 Service Worker 的更新机制，新发布的资源可能不会立即被用户看到，需要等到下次访问或者刷新页面时才能看到。
6. 资源消耗：Service Worker 运行在后台，即使在页面关闭后也能继续运行，这可能会消耗用户的系统资源和电量。
