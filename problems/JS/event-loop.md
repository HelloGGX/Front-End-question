# 对事件循环与浏览器渲染时机的探讨

> (当我们用时间作为行为的基准时), 循环不再是相同状态的重复，而是在不同时间经历相似的状态。而且，测量时间让我们能区分以不同速率进行的相似循环。
> —— 《系统化思维导论》，杰拉尔德·温伯格

## JavaScript的异步错觉

早期的javascript并不是具有并行特性，它的并行特性是在随着应用需要而逐渐加入的。

由于早期的JavaScript主要应用与网页浏览器，因此有宿主（浏览器）通过宿主全局对象（例如window）提供的超时回调或定时器（setTimeout或setInterval）成为实现并行的主要手段。

而这事实上并非JavaScript的内置特性，也并未在语言层面上加以规范。

在ECMAScript中没有约定任何与调度时间相关的运行期机制，也就是说没有进程、线程、也没有单线程/多线程这样的调度模型。

因此仅使用ECMAScript约定的标准库，事实上是无法“写出一个并行过程”的。

这也是几乎所有展示Promise特性的示例代码都要是使用setTimeout的原因——这样才能建一个并行的任务。

之所以我们可以“并行”地执行任务，是因为浏览器是多线程的。当JS需要执行并行任务时，浏览器会帮我们启动另外的线程执行任务。

我们说JS是单线程的，并不是说JS这门语言就是单线程的，而是说**浏览器为JS提供的执行线程只有一个**，我们一般称为：主线程**Main Thread**

> The main thread is where a browser processes user events and paints. By default, the browser uses a single thread to run all the JavaScript in your page, as well as to perform layout, reflows, and garbage collection.

除了为JS引擎提供的线程外，浏览器的**渲染进程**还有其他线程，如：

1. GUI 渲染线程
2. 定时触发器线程
3. 事件触发线程
4. 异步http 请求线程

举个例子：
当JS主线程中的某个事件发起AJAX请求，就会把该任务交给另一个浏览器线程—— **异步http请求线程**发送请求，当请求数据就绪，就会将callback中需要执行的JS回调交给JS引擎的主线程执行。

也就是说浏览器才是真正发送请求的角色，而JS只是负责执行最后的回调处理。

所以如前文所说，异步特性并不是ECMAScript约定的标准实现，而是浏览器为JS提供的能力。

## 事件循环

js在解析一段代码时，会将同步代码，我们称为：**可执行上下文**按照约定顺序放入栈中依次执行（**当一个新的上下文入栈时，那么新的上下文必然是活动上下文**），js在执行一个任务时，会生成属于这个任务的执行环境，也就**执行上下文**。

每当栈顶的同步代码执行完成后，销毁当前的执行环境，从栈顶中弹出。

当遇到异步任务时，就交给浏览器其他线程来处理,等待请求返回时，将异步回调依次放入一个队列中。

等待当前执行栈中所有的同步代码执行完后，检测任务队列是否有新任务，如果有，取出队列中的回调，加入执行栈中执行。

当执行栈空了，又去检测任务队列是否有新任务，重复上面的过程，我们称这样的一次过程为：**事件循环**或者一次**tick**（每个宏任务需要等待下一轮的事件循环）

<img WIDTH="600" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/event-loop1.png"/>


## 老生常谈的宏任务和微任务

根据任务的种类不同，可以分为微任务队列和宏任务队列。

- 微任务: process.nextTick, Promises, queueMicrotask, MutationObserver
- 宏任务：setTimeout, setInterval, setImmediate, I/O

> 注意一个点：当执行Promise.then时，V8引擎不会将异步任务交给浏览器其他线程处理，而是将回调存在自己的一个队列中，这和setTimeout有本质的区别，Promise.then这个微任务并没有多线程的参与。

在事件循环的过程中，执行栈在同步代码执行完成后，优先会去检查微任务队列，等待微任务队列彻底执行完成之后，才执行宏任务队列，进而才能正常走完一个事件循环。

也就意味着在开发中要注意三点：

1. 当微任务中有一个死循环正在执行，那么宏任务队列会一直挂起等待，主线程代码不会执行，更不会进入UI渲染流程，导致页面卡住
```js
function loop(){
    Promise.resolve().then(loop);
};
loop();
```
2. 当宏任务中有一个死循环, 主线程依然会执行，不断把回调放入宏任务队列，并不会阻塞事件循环的进行，也就意味着会正常进入UI渲染流程，页面不会卡住

```js
function loop(){
    setTimeout(()=> {
        loop()
    })
};
loop();
```
3. 主线程的同步任务，如果有阻塞，执行栈没有清空，那么会导致事件队列中的任务挂起等待。
最典型的问题就是**倒计时**，如果我们用setTimeout或者setInterval来实现倒计时，那么有可能会导致倒计时的不准确。

## 浏览器渲染时机

由于浏览器有很多进程，这里我们只讨论负责渲染的GUI渲染进程。

> 渲染进程的核心工作是将 HTML，CSS 和 JavaScript 转换为用户可与之交互的网页。

渲染进程中包含三个线程：
1. Compositor Thread: 合成器线程
2. Tile Worker：由合成器线程产生的一个或多个工作线程处理光栅化任务
3. Main Thread：主线程，执行JavaScript、样式、布局和绘制页面

一次事件循环结束之后（一个宏任务执行完），浏览器会判断当前的document是否需要渲染，因为浏览器会尽可能的保持60fps的稳定帧率，1秒最多出现60次渲染机会，也就是说，只需要保持16ms左右渲染一次就可以了。

因此并不是，也没必要每轮事件循环完成之后就马上渲染，一般每帧都会有多次事件循环。

> 浏览器会根据屏幕刷新率、页面性能、页面是否在后台运行来共同决定渲染时机。

同时对于一些比较卡顿的已经不能维持60fps的页面，若再在此时执行界面渲染会雪上加霜，所以浏览器可能会下调至document能获益的频率为（比如说）30fps的更新速率。

> 多提一句：浏览器上下文不可见时，页面的帧速率会降低至4fps


在浏览器判断可以渲染之后，整个渲染细节如下：

### 第一阶段. Vsync

垂直同步信号Vsync被触发（Vsync，水平同步表示画出一行屏幕线，垂直同步就表示从屏幕顶部到底部的绘制已经完成，指示着前一帧的结束，和新一帧的开始），开始渲染（一帧开始）

<img width="380px" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/event-loop3.png"/>


### 第二阶段. Input event handlers

1. [run the resize steps](https://drafts.csswg.org/cssom-view/#run-the-resize-steps): 若浏览器resize过，那么这里会在Window上触发’resize’事件。
2. [run the scroll steps](https://drafts.csswg.org/cssom-view/#run-the-scroll-steps): 每当我们在某个target上滚动时(target可以是某个可滚动元素也可能就是document)会触发

由于 JavaScript 是在主线程上运行，因此在合成页面时，合成线程会将页面上具有事件处理程序的区域标记为“非快速滚动区域”。

有了这些信息，如果事件发生在该区域中，则合成线程可以确保将输入事件发送到主线程。

但是我们一般会利用事件委托的方式将事件绑定到根节点，react就是个典型，但是，如果从浏览器的角度查看此代码，现在整个页面都被标记为不可快速滚动的区域了。

这意味着，即使您的页面的某些部分不关心输入事件，合成线程也必须与主线程通信，并在每次输入事件发生时等待主线程。此时，合成线程就丧失了使页面平滑滚动的能力。

为了避免这种情况的发生，您可以在事件监听时设置 passive: true 。这会告知浏览器您仍要在主线程中监听事件，但是合成器也可以并行继续合成新的帧。

```js
document.body.addEventListener('touchstart', event => {
    if (event.target === area) {
        event.preventDefault()
    }
 }, { passive: true });
```

如果输入事件来自该区域之外，则合成线程将在不等待主线程的情况下进行新帧的合成。

对于用户输入，常用的触摸屏设备每秒发送 60-120次触摸事件，而常用的鼠标则每秒发送 100次事件。输入事件的频率高于我们的屏幕刷新能力。

如果诸如 mousemove 这样的连续事件被以每秒钟 120次的频率传递给渲染进程的主线程，与屏幕低刷新率相比，会触发过量的命中测试和 JavaScript 的回调执行。

为了尽量减少对主线程过度调用，Chrome聚合了连续事件（如 wheel，mousewheel，mousemove，pointermove， touchmove，scroll）并延迟调度，直到下一次 requestAnimationFrame 执行前。

因此也就意味着，这些连续触发的事件自带节流效果。因为它们都固定每帧执行一次。

### 第三阶段: requestAnimationFrame

这里你可以在页面渲染前，提前获取最新的输入数据，然后针对输入的数据进行样式计算。因为requestAnimationFrame的回调会放在每帧的开始执行，所以大量的样式计算，会被规律的分为一帧一帧的处理，这样就不会存在过度绘制的问题，动画不会掉帧，自然流畅。

在MDN中，还有这样一句话：**但在大多数遵循W3C建议的浏览器中，回调函数执行次数通常与浏览器屏幕刷新次数相匹配**。

但目前大多数设备的刷新率已经远远超过了60fps, 在Chrome76之前，无论多高的刷新率，通过requestAnimationFrame 回调函数执行次数依然是每秒30-60次。这是之前的一个bug。

Chrome76之后官方进行了修复，详情见[高刷显示器下的 requestAnimationFrame](https://juejin.cn/post/6953541785217925151)

但需要注意的一点：当你在回调中访问计算样式和布局属性时，也就是我们说的访问了scrollWidth、clientHeight、ComputedStyle等触发了强制重排，导致Recalc Styles和Layout前移到代码执行过程当中。

之后的阶段就不一一赘述。他们依次为：

解析HTML(Parse HTML)-> 样式计算(Recalc Styles)-> 布局（Layout) -> 更新层叠树（update layer tree）-> 绘制（Paint）-> 合成图层（Composite）-> 提交到GPU线程呈现页面 -> 一帧结束

想要了解以上阶段其中的细节，[这篇文章《inside-browser-part3》讲的很好](https://github.com/xitu/gold-miner/blob/master/TODO1/inside-browser-part3.md)

## 总结

1: 渲染进程下各个线程的流程：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/anatomy-of-a-frame.jpg"/>


2: 事件循环与渲染进程的关系：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/微信图片_20220221224634.jpg"/>




---

## 参考资料

- [The Anatomy of a Frame](https://aerotwist.com/blog/the-anatomy-of-a-frame/)
- 《JavaScript 语言精髓与编程实践》
- [The JavaScript Event Loop: Explained](https://towardsdev.com/event-loop-in-javascript-672c07618dc9)
- [inside-browser-part3](https://github.com/xitu/gold-miner/blob/master/TODO1/inside-browser-part3.md)


