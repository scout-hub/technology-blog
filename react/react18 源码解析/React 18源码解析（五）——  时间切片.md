# React 18源码解析（五）——  时间切片（上）



### 该部分解析基于我们实现的简单版react18中的代码，是react18源码的阉割版，希望用最简洁的代码来了解react的核心原理。其中大部分逻辑和结构都和源码保持一致，方便阅读源代码。



### 一、什么是时间切片

`react`时间切片是一种技术，它允许`react`将渲染工作切分成多个片段。浏览器可以在这些片段之间去执行其它浏览器任务，例如用户输入以及动画，从而提高程序的响应能力，使得页面交互更为流畅。



### 二、什么是长任务

长任务是指长时间独占主线程，导致界面卡顿的 JavaScript 代码。

在同步调度模式下，`react`是这样调度更新的（如下图）。所有`fiber`的调度逻辑都在一个宏任务中执行，整个任务的耗时跟页面`DOM`量级正相关。下图中圈出部分的调度任务已经被标红，意味着这是一个长任务（long task），需要优化。

为什么需要优化长任务？首先`JavaScript`是单线程的，浏览器渲染工作、IO处理、 JS 的执行等等都在主线程上，会互相阻塞。目前大多数显示器的刷新频率都在60HZ，也就是每`16.6ms`左右刷新一次。如果 JS 任务执行时间过长，假设超过 100ms，那么中间将有好几帧都无法进行浏览器渲染更新任务，用户就会明显感受到页面卡顿（掉帧，这几帧的画面丢了）。

![image-20230706172555410](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230706172555410.png)

这种长任务也会导致一些`IO`操作得不到及时响应，如下图所示，我们在输入框中不停地输入内容，在长任务触发之前内容可以正常呈现。当长任务触发时，这些输入事件会被阻塞。虽然我们还在输入内容，但是内容并不会呈现直到长任务结束。作为用户的直观感受就是页面卡住了，这是一个非常不好的体验。

![image-20230706170636146](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230706170636146.png)

 由于长任务会带来种种性能和体验上的问题，因此消除长任务是前端性能优化中的一块重头戏。



### 三、优化长任务

通常一个长任务的耗时应该在`50ms`以内，在[RAIL 模型](https://web.dev/rail/)中建议在[50 毫秒](https://web.dev/rail/#response-process-events-in-under-50ms)内处理用户输入事件，以确保在 100 毫秒内做出可见响应，并且这些任务需要在合适的时间和地点运行。合适时间指的是执行任务的时机，合适的地点指的是执行任务的位置。

对于合适的时间点，我们通常会选择`setTimeout`、`requestAnimationFrame`、`requestIdleCallback`等 API。对于合适的地点其实就是考虑是否需要在非主线程上执行，常见的就是`web worker`技术。



我们以先来看一个例子：

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
    <div id="div">0</div>
    <button id="btn">更新</button>
    <script>
        const fib = (n) => {
            if(n <= 1) return n;
            return fib(n - 1) + fib(n - 2);
        };
        let i = 0;
        document.getElementById('btn').addEventListener('click', () => {
            i++;
            document.getElementById('div').innerText = i;
            if(i === 2) {
                fib(10);
            }
        });        
    </script>
</body>

</html>
```

在这个例子中有一个`div`和`button`，`div`用来显示内容，点击`button`的时候会更新内容。当我们传入一个较小值（10）给`fib`函数并连续点击按钮时可以看到，页面上的数字是连续变化的。

![chrome-capture-2023-6-7](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2023-6-7.gif)

当我们传入一个较大值（40）时可以明显看到中间缺失了一段数字变化的过程。

![chrome-capture-3](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-3.gif)

从 performance 性能面板中，我们可以看到当`fib`传入的参数为 40 时，这个方法执行了大约 1s 的时间。当方法结束之后，页面从原先的数字突变到了 7，这也就意味着当方法执行完之后浏览器才有时间去执行渲染，此时对应的变量`i`已经是 7 了。

![image-20230707140037104](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230707140037104.png)

![image-20230707140158496](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230707140158496.png)

因此我们要做的优化就是，即使传给`fib`的参数是 40，页面上的数字也能呈现连续变化，至少不应该缺失那么多画面。

通常对于长任务来说，可以做任务拆分，将一个长任务切分成多个小任务执行，每一个小任务的耗时也需要尽可能的少。但是上述例子中的`fib`是一个递归函数，我们无法通过简单的方式去做任务拆分。因此，对于这种无法拆分的耗时任务，我们可以考虑在合适的地点进行处理，也就是`web worker`。

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <div id="div">0</div>
    <button id="btn">更新</button>
    <script>
        const worker = new Worker('./worker.js');
        worker.addEventListener('message', (e) => {
            document.getElementById('div').innerText = `fib的计算结果是: ${e.data}`;
        });
        let i = 0;
        document.getElementById('btn').addEventListener('click', () => {
            i++;
            document.getElementById('div').innerText = i;
            if(i === 2) {
                worker.postMessage(40);
            }
        });        
    </script>
</body>

</html>
<!-- worker.js -->
const fib = (n) => {
    if(n <= 1) return n;
    return fib(n - 1) + fib(n - 2);
};

self.addEventListener('message', (event) => {
    const n = event.data;
    const result = fib(n);
    self.postMessage(result);
});
```

我们可以看到页面数字依旧呈现连续变化的过程，当`worker`子线程计算完后通过`postMessage`通知主线程上监听的`message`消息，将计算结果渲染到页面上。

![chrome-capture-2](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-2.gif)

从`performance`面板上也可以看到主线程上已经没有耗时任务了，对应的多出了一个`worker`线程，这个线程执行的就是我们定义的`fib`方法。

![image-20230707144022433](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230707144022433.png)

当然，`web worker`也不是万能的，它有以下几个问题：

1.  `DOM`限制：`worker`无法读取DOM对象，一些文档对象 api 无法调用，只能访问`navigator`和`location`对象。

2. 文件读取限制：`worker`子线程无法访问文件系统，因此所有文件需要来自网络。

3. 通信限制：主线程和`worker`子线程处于不同上下文，因此数据通信需要借助`postMessage`。

4. 脚本执行限制：`worker`不能使用`alert`和`confirm`提示。

5. 同源限制：`worker`代码文件和主线程执行的代码文件需要同源。
5. `worker`子线程创建后不会主动销毁，并且会一直处于执行状态，因此比较耗资源，切不可不能滥用，在执行完线程任务后也需要手动销毁。



我们再来看一个例子：

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
    <button id="btn">更新</button>
    <div id="container"></div>
    <script>
        const addChildren = () => {
            const fragment = document.createDocumentFragment();
            for(let i = 0; i < 200000; i++) {
                const div = document.createElement('div');
                div.innerHTML = i;
                fragment.appendChild(div);
            }
            document.getElementById('container').appendChild(fragment);
        }
        document.getElementById('btn').addEventListener('click', () => {
            addChildren();
        });
    </script>
</body>

</html>
```

在这个例子中，我们点击按钮会向页面添加 20 万个`div`。当我们点击按钮后可以看到页面过了大概 2-3s 才显示出内容。

![chrome-capture-4](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-4.gif)

从`performance`性能面板中我们可以看到整个任务的执行大概花了 3s 时间，并且后续的浏览器渲染也花了大概 500ms 的时间。如此长时间的响应对用户来说肯定是不有好的，接下去我们需要对这部分内容做优化。

![image-20230707164003508](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230707164003508.png)

在这个例子中，我们采用一次性循环的方式创建元素，最后再一起添加到页面中。既然一次性创建元素再添加到页面会出现长任务，造成页面卡顿，那我们是否可以通过任务拆分的方式，将一次大循环分割成多个小循环的方式去处理？我们尝试将任务进行拆分，每一个任务执行 100 次循环，看看效果如何。

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
    <button id="btn">更新</button>
    <div id="container"></div>
    <script>
        const addChildren = (begin, limit) => {
            const fragment = document.createDocumentFragment();
            for(let i = begin; i < begin + limit; i++) {
                const div = document.createElement('div');
                div.innerHTML = i + 1;
                fragment.appendChild(div);
            }
            document.getElementById('container').appendChild(fragment);
        }
        document.getElementById('btn').addEventListener('click', () => {
            for(let i = 0; i < 1000; i++) {
                addChildren(i * 100, 100);
            }
        });
    </script>
</body>

</html>
```

从结果上看好像并没有多大的区别，依旧是一个比较耗时的长任务。之前说了通过任务拆分的方式可以优化长任务，那是哪里还存在问题？

我们从下图中可以看到，虽然我们通过任务拆分的方式执行`addChildren`，但是所有的`addChildren`依旧是在一个宏任务里面执行的，这个宏任务依旧是耗时任务，从事件循环的角度讲依旧会阻塞后续的渲染任务。我们知道，一次事件循环中会先执行一个宏任务，如果任务执行过程中持续产生微任务，那么这些微任务会被放入微任务队列中，等待后续执行，最后浏览器会判断是否需要重新渲染页面，如果需要则执行渲染任务。因此，我们只是单纯拆分任务还不够，还需要将拆分的任务放到合适的时间点去执行。

![image-20230707170924746](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230707170924746.png)

 那么在什么时间点去执行我们的任务才能保证不阻塞渲染，也就是说我们需要在每一帧为浏览器预留一定的时间，将主线程的控制权交还给浏览器，让浏览器能去做自己的事情。答案是：宏任务。

在浏览器事件循环中，渲染任务（paint、layout）会在宏任务间隙之间执行，每一次宏任务的执行都代表新一轮的事件循环。因此，我们只需要把每一次`addChildren`放到宏任务中去执行，这样就能保证浏览器渲染任务能够在其之间执行。

那为什么不用微任务呢？因为微任务也是本次事件循环中，宏任务之后，渲染之前执行，它依旧会阻塞渲染。



我们尝用宏任务去优化上面的例子：

1. `setTimeout`

它是一个定时器，用来指定回调任务在多少毫秒之后执。当我们调用这个API时，该回调任务会被放入延迟队列中，等到超时时间后将其从延迟队列中取出，放入宏任务队列中执行。在之前的例子中，我们在每一次`addChildren`外都包一层定时器：

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
    <button id="btn">更新</button>
    <div id="container"></div>
    <script>
        const addChildren = (begin, limit) => {
            const fragment = document.createDocumentFragment();
            for(let i = begin; i < begin + limit; i++) {
                const div = document.createElement('div');
                div.innerHTML = i + 1;
                fragment.appendChild(div);
            }
            document.getElementById('container').appendChild(fragment);
        }
        document.getElementById('btn').addEventListener('click', () => {
            for(let i = 0; i < 1000; i++) {
                setTimeout(() => {
                    addChildren(i * 100, 100);
                });
            }
        });
        document.getElementById('btn').addEventListener('click', click);   
    </script>
</body>

</html>
```

 从交互结果上来看确实要比之前快很多

![chrome-capture-x](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-x.gif)

我们再看看`performance`面板。在面板上看可以到，先前的一整块大任务被切分成了一段一段，红色和绿色部分交替执行。绿色部分是多个定时器任务`addChildren`，这些任务几乎看不到长任务标记；红色部分则是浏览器的渲染任务，这部分有长任务标记，当然我们不考虑结合`DOM`的优化。

![image-20230710101814096](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230710101814096.png)

`setTimeout`虽然能够解决问题，但是它存在着一定的缺陷：

- 执行时机：`setTimeout`只是在达到延迟时间后将回调任务放入宏任务队列中，并不是直接执行回调。假设宏任务队列中有其它任务排在前面，那么依旧需要等待前面的任务执行完成。
- 最小延迟时间：当`setTimeout`的嵌套层级过多时（没记错的话应该是 5 层），如果延迟时间小于 4ms，那么默认延迟会变成 4ms。



2. `requestAnimationFrame`

这是一个用于在下一次浏览器重绘之前调用函数的方法，回调的执行跟屏幕刷新频率有关。一般用于优化动画效果，让动画变得更加流畅。在这里我们也可以用这个 API 来优化上面的例子，核心就是保证页面流畅不卡顿。

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
    <button id="btn">更新</button>
    <div id="container"></div>
    <script>
        const addChildren = (begin = 0, limit = 100) => {
            const fragment = document.createDocumentFragment();
            for(let i = begin; i < begin + limit; i++) {
                const div = document.createElement('div');
                div.innerHTML = i + 1;
                fragment.appendChild(div);
            }
            document.getElementById('container').appendChild(fragment);
        }
        const render = (i) => {
            addChildren(i * 100, 100);
            if(i < 999) {
                requestAnimationFrame(() => {
                    render(i + 1);
                });
            }
        }
        document.getElementById('btn').addEventListener('click', () => {
            requestAnimationFrame(() => render(0));
        });
    </script>
</body>

</html>
```

我们依旧可以看到页面渲染非常流畅。从`peformance`面板中也可以看到，任务被切分成了一个个 task，这些 task 都是 animation frame 触发，并且回调中的`addChildren`逻辑都在渲染之前执行。

![chrome-capture-xxxx](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/chrome-capture-xxxx.gif)

![image-20230710144121915](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230710144121915.png)

虽然`requestAnimationFrame`能够让页面渲染、动画更加流畅，但是也存在着一些限制或者前提：

- 触发时机：与`setTimeout`不同的是，我们无法控制`requestAnimationFrame`回调的执行时机。它总是在下一次浏览器重绘之前触发，受屏幕刷新频率影响，执行频率并不高。
- 回调任务耗时：如果`requestAnimationFrame`的回调任务执行时间过长，可能会导致帧率下降。这也就是我们之前提到过的，切分的任务执行耗时也要尽可能得少。
- 页面状态：`requestAnimationFrame`只会在当前页面处于激活状态时运行，如果页面被隐藏或最小化，回调函数将不会运行。



3. `requestIdleCallback`

该函数将在浏览器空闲期间被调用。这使得开发者能够在主事件上循环执行后台和低优先级工作，而不会影响延迟按键事件，如动画和输入响应。

我们换一个新的例子：

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
    <button id="btn">更新</button>
    <div id="container"></div>
    <script>
        const fib = (n) => {
            if(n < 2) {
                return n;
            }
            return fib(n - 1) + fib(n - 2);
        }

        const addChildren = (begin = 0, limit = 100) => {
            const fragment = document.createDocumentFragment();
            for(let i = begin; i < begin + limit; i++) {
                const div = document.createElement('div');
                div.innerHTML = i + 1;
                fragment.appendChild(div);
            }
            document.getElementById('container').appendChild(fragment);
        }
        const render = () => {
            for(let i = 0; i < 3; i++) {
                addChildren(i * 100, 100);
                fib(35);
            }
        }

        document.getElementById('btn').addEventListener('click', render);

    </script>
</body>

</html>
```

在上面这个例子中，每次执行`addChildren`之前都会调用`fib`计算数据，根据之前的经验我们知道这个`fib(35)`的执行会阻塞渲染，现在我们尝试用`requestIdleCallback`进行优化：

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
    <button id="btn">更新</button>
    <div id="container"></div>
    <script>
        const fib = (n) => {
            if(n < 2) {
                return n;
            }
            return fib(n - 1) + fib(n - 2);
        }

        const addChildren = (begin = 0, limit = 100) => {
            const fragment = document.createDocumentFragment();
            for(let i = begin; i < begin + limit; i++) {
                const div = document.createElement('div');
                div.innerHTML = i + 1;
                fragment.appendChild(div);
            }
            document.getElementById('container').appendChild(fragment);
        }
        const render = () => {
            for(let i = 0; i < 3; i++) {
                addChildren(i * 100, 100);
                requestIdleCallback(function (deadline) {
                    // 如果还有时间，继续执行任务
                    if(deadline.timeRemaining() > 0) {
                        fib(35);
                    }
                });
            }
        }

        document.getElementById('btn').addEventListener('click', render);

    </script>
</body>

</html>
```

从结果上看`fib(35)`的执行已经被放到 idle callback中执行了，虽然这个 idle callback 依然是一个长任务，但是它在页面渲染之后执行，并不会阻塞渲染。

![image-20230710161754277](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230710161754277.png)

`requestIdleCallback`的问题：

- 兼容性问题：这个 API 目前还没有被所有浏览器支持。
- 执行时机：`requestIdleCallback`只会在当前帧有剩余时间才会执行，因此它的执行频率也不高，有可能一直不会触发。
- 场景：`requestIdleCallback`的设计初衷是处理低优先级的任务，将其放到浏览器空闲时间再执行，不让其影响到高优先级任务的执行，因此并不适合所有任务。



### 四、React 时间切片

上面提到的三个优化调度的 API 都有一定的缺陷，因此`react`并没有优先考虑。其实`setTimeout`已经是接近可以使用的方案，但是由于其存在最小 4ms 延迟的问题，使得`react`不得不将其作为备胎。在 60 FPS下，每帧间隔不能超过 16.6ms，不容这 4ms 的浪费。

`react`最终采用的是`MessageChannel`，它也是一个宏任务，并且它的触发时机要比`setTimeout`更加提前。`MessageChannel`接口允许创建一个新的消息通道，并通过它的两个[`MessagePort`](https://developer.mozilla.org/zh-CN/docs/Web/API/MessagePort)属性发送数据。

我们采用`MessageChannel`的方式优化一下之前的例子：

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
    <button id="btn">更新</button>
    <div id="container"></div>
    <script>
        const addChildren = (begin = 0, limit = 100) => {
            const fragment = document.createDocumentFragment();
            for(let i = begin; i < begin + limit; i++) {
                const div = document.createElement('div');
                div.innerHTML = i + 1;
                fragment.appendChild(div);
            }
            document.getElementById('container').appendChild(fragment);
        }
        const channel = new MessageChannel();
        const port = channel.port2;
        let beginTime = performance.now();
        let i = 0;

        const shouldYield = () => performance.now() - beginTime >= 5;

        channel.port1.onmessage = () => {
            while(i < 300 && !shouldYield()) {
                addChildren(i * 100, 100);
                i++;
            }
            // 如果 i 没到 300 说明之前任务被中断了，这里需要继续执行
            if(i !== 300) {
                render();
            }
        };
        const render = () => {
            beginTime = performance.now();
            port.postMessage(null);
        };
        document.getElementById('btn').addEventListener('click', render);
    </script>
</body>

</html>
```

在上面这个例子中，我们加入了中断机制。当整个`render`任务执行总时间超过 5ms 时就会中断执行，后续通过 i 是否达到预定值来判断是否需要恢复执行。如果 i 没有达到预定值则继续调用`render`。这里用 5ms 是为了跟`react`保持一致，当屏幕刷新频率为 60 HZ时，一帧执行时间为 16.6ms，去掉浏览器各种任务的执行，留给 js 执行的逻辑大致为 5ms左右。从这里也能看出`setTimeout`的 4ms 延迟还是有不小的影响。

从`performance`面板上也能看到任务被切分了，并且单个任务的执行时间大概在 5ms 左右。

![image-20230710173454450](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230710173454450.png)

上面这个优化例子已经接近了`react`分片的思想，我们看一下`react`切片的效果：

![image-20230706171536310](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230706171536310.png)

![image-20230706170449777](https://raw.githubusercontent.com/scout-hub/picgo-bed/dev/image-20230706170449777.png)

这一章我们大致了解了切片相关的概念及其优化措施，下一章我们将深入源码，探究`react`如何实现时间切片。