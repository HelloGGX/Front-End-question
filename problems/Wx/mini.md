# 小程序和开发者工具架构解析

最近在做小程序相关的项目，使用 Taro 这样一个开放式跨端框架，可以轻松的用 react 开发小程序，并在过程中使用到了微信开发者工具进行调试，在开发过程中产生一些疑问：

- 小程序的底层是如何设计的
- 微信开发者工具底层是如何让开发者能调试小程序的

## 小程序的技术选型

目前主流的 APP 主要有三种：Native App、Web App、Hybird App。它们对应了三种渲染模式：

- Native(纯客户端原生技术渲染)
- WebView(纯 Web 技术)
- WebView+原生组件（Hybird 技术）

WebView 就是一个 WebKit 内核的浏览器，用于嵌入 web 内容到原生应用，一般来说，WebView 和 Native 的区别如下：

| 对比项   | Native                       | WebView           |
| -------- | ---------------------------- | ----------------- |
| 开发门槛 | 高                           | 低                |
| 用户体验 | 好                           | 白屏、交互反馈差  |
| 版本更新 | 需审核，迭代慢               | 在线更新+动态加载 |
| 管控性   | 平台可管控，不合规内容可下架 | 内容管控难        |

小程序选择了 WebView+原生组件的方式，结合了 Native 和 WebView 的一些优势，让开发者既可以享受 WebView 页面的低门槛和在线更新，又可以使用部分流畅的 Native 原生组件，同时通过代码包上传、审核、发布的方式来对内容进行管控。

## 小程序为何不提供 DOM 和 BOM 相关 API？

在开发原生小程序时，你会发现小程序并没有提供 DOM 和 BOM API，用一张图简单回忆一下 DOM 和 BOM:

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/dom&bom.png"/>

从上图可以看出 document 是 DOM 的核心对象，window 则是 BOM 的核心对象，

```js
console.log(window.document === document); //true
```

- **BOM**：浏览器对象模型(Brower Object Model)，是用于操作浏览器而出现的 API，BOM 对象则是 Javascript 对 BOM 接口的实现。
- **DOM**：文档对象模型（Document Object Model），是 W3C 定义的一套用于处理 HTML 和 XML 文档内容的标准编程接口 API。javascript 实现 DOM 接口的对象对应的是 document 对象，JS 通过该对象来对 HTML/XML 文档进行增删改查。

我们可以通过以上两个接口，随意的操作浏览器的行为和网页的展示内容，这也随之带来了一些问题：

1. 开发者可以随意操作浏览器跳转
2. 开发者可以随意获取敏感数据，改变展示内容
3. XSS(注入 js 脚本)和 CSRF(利用 cookie)安全漏洞

由于小程序所承载的微信客户端是用户人数巨大的应用，安全和环境稳定是首先要考虑的问题，因此小程序团队采用了类似”沙箱“的解决方案。提供一个纯 JavaScript 的解释执行环境，其中没有浏览器相关的接口，彻底与浏览器接口隔离。

这样一个纯的 JavaScript 环境，在小程序中负责**逻辑层**。

js 处理事件交互，页面需要响应更新，我们知道小程序的渲染模式是 Hybird 的方式，因此渲染层交给 WebView 线程处理。小程序的一个页面就是一个 WebView, **因此在渲染层一般会有多个页面，多个 WebView 线程,但是逻辑层只有一个 JavaScript 线程**。

这样一种将渲染层和逻辑层隔离开，分别交给两个线程处理的设计，称为小程序的**双线程设计**：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/mini-双线程.png"/>

这样设计的好处在于：

- 避免了 Js 的执行，阻塞页面的渲染
- js 纯沙箱环境，排除了上面提到的安全问题

> 注意点：前面提到一个小程序只有一个逻辑线程，也就意味着，多个页面共用一个逻辑线程，小程序会将开发者编写的业务代码打包成一份 JavaScript 文件，在小程序启动的时候从后台下载；小程序被销毁的时候，停止运行。因此页面关闭后又打开，整个 JS 文件不会重新加载，而右上角的关闭按钮只是指小程序切换到了后台，并不会真正关闭小程序。只有用户在快捷入口手动删除小程序，或是缓存被清理之后，才会重新加载环境。

## Virtual DOM 与双线程通信

在底层设计中，渲染层和逻辑层是相互独立的，没法直接通信。小程序则使用微信客户端 Native 进行直接通信。而渲染层与客户端，逻辑层与客户端都采用**在渲染层和逻辑层的全局对象中注入一个原生方法用于通信**，封装成 WeiXinJSBridge 这样一个兼容层。那么小程序如何将逻辑层的数据通信，转化成渲染层页面的展示的呢？

在小程序中，双线程通信时会将需要传输的数据转换为字符串的形式。通过 setData API 将数据同步到渲染层：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/mini-双线程通信.png"/>

在渲染层里，小程序会把 WXML 转换成 Virtual DOM 的 Js 对象，在收到来自逻辑层的数据传输后，对比差异，将差异应用到原来的 Virtual DOM 树上，并更新页面。
那 Virtual DOM 是如何进行更新的呢？这就要说一下 DOM Diff 的过程了。

1. WXML 转换成 Virtual DOM 的 Js 对象，模拟 DOM 树。
2. 当状态变更的时候，重新构造一个新的对象树，然后比较新旧树，记录两棵树的差异。通常需要记录的差异包括：需要替换的原来的节点，移动、删除、新增子节点，修改节点的属性，文本节点的文本内容改变。
3. 把差异应用到真正的 DOM 树上。
4. 渲染 UI 界面

以上的过程可以看出，逻辑层与渲染层需要经过微信客户端中转通信，并不能实时的到达渲染层，所以 setData 函数将数据从逻辑层发送到渲染层是异步的。

在实际开发中我们可能会多次调用 setData,那么逻辑层与渲染层会产生频繁的通信，产生性能损耗，因此你会在小程序的官方文档里面找到这样的优化建议：

> setData 调用频率
> setData 接口的调用涉及逻辑层与渲染层间的线程通信，通信过于频繁可能导致处理队列阻塞，界面渲染不及时而导致卡顿，应避免无用的频繁调用。

> 得分条件：每秒调用 setData 的次数不超过 20 次

> setData 数据大小
> 由于小程序运行逻辑线程与渲染线程之上，setData 的调用会把数据从逻辑层传到渲染层，数据太大会增加通信时间。

> 得分条件：setData 的数据在 JSON.stringify 后不超过 256KB

面对以上问题，小程序也提供了三种解决方案：

1. 自定义组件
2. 原生组件
3. WXS

## 自定义组件

在小程序中，不管是内置组件还是自定义组件的实现，都需要与 Shadow DOM 模型结合起来理解。Shadow DOM 模型是一个 HTML 新规范，允许开发人员封装 HTML 组件。
我们会在开发者工具中，看到我们的自定义组件引用的位置：

```jsx
<my-component is="components/my-component/my-component">
    #shadow-root
  <my-component is="components/my-component/my-component">
      #shadow-root
  </my-component>
</my-component>
```

#shadow-root 称为影子根。自定义组件之间可以相互嵌套，形成影子树（Shadow Tree）的拼接，shadow dom 有一个天然的优势，dom 和组件样式隔离，外部 js 无法直接访问改变组件的 DOM Tree，基于 html 原生支持，不需要重新解析模板，浏览器可以直接渲染出结果,性能自然更好。

对于开发者来说，逻辑层需要知道组件创建、销毁等生命周期和状态，如果我们依然将 Virtual DOM 信息维护在渲染层中，那么每个自定义组件的所有状态和生命周期的变更都需要通过系统层转发通知到逻辑层，通信就会过于频繁和密集。这时，我们可以在逻辑层也维护一份 Virtual DOM 信息。所以，在双线程下，两个线程都需要保存一份节点信息。

因此当我们调用 setData 的过后，小程序会在逻辑层进行 DOM Diff，然后将 Diff 结果传到渲染层（注意，此处与页面的渲染流程不一致）。此时渲染层只需要拿到 Diff 信息就可以更新 Virtual DOM 节点信息，同时更新页面了。这样同时避免了通信频繁和传输数据量过大的问题。

具体渲染流程如下：

(1) 新建页面在渲染层进行：渲染层 WXML 生成一个 Virtual DOM 的 JS 对象，拼接 Shadow Tree，注入初始数据进行渲染。

(2) 逻辑层调用 setData，更新数据到渲染层：逻辑层开始执行逻辑，调用 setData 之后，会把 setData 的数据通过 Native 传递到渲染层。

(3) 渲染层页面更新：渲染层对需要更新的数据进行 Diff，得到差异，然后把差异应用到真实的 DOM 中，从而更新页面。

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/mini-自定义组件.png"/>

> 实际上小程序的页面，也可以视作自定义组件，因此在实际开发中，如果页面某个数据数据的 setData 十分频繁，就可以使用 Component 来包装一层。

## 原生组件

为了应对这种强交互的场景，小程序引入了原生组件。小程序是 Hybrid 应用，除了 Web 组件的渲染体系，还有由客户端原生参与组件（原生组件）的渲染。原生组件可以直接与逻辑层通信，减少了逻辑层和渲染层的中转通信和计算。

这些组件有：camera、canvas、video、map、live-player、live-pusher、textarea、input（仅在 focus 时表现为原生组件）。

需要强调的是原生组件脱离在 WebView 渲染流程外。因此页面中的其他组件无论设置 z-index 为多少，都无法覆盖在原生组件上。小程序给出了同层渲染的解决方案，现在原生组件都已经支持同层渲染。

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/mini-原生组件.png"/>

具体渲染流程如下：

(1) 首先，渲染层会根据逻辑层传入的数据创建该组件。

(2) 然后，该组件会被插入页面，同时根据样式和属性设计，WebView 会渲染布局，可以得到组件的位置和宽高。

(3) 最后，我们将计算得到的数据告诉客户端，客户端就可以将原生组件渲染到具体的位置上。

## WXS

WXS 可以看作是精简版的 JavaScript, 它直接运行在渲染层，规避了频繁地进行逻辑层和渲染层的通信和计算，性能自然更好。

根据官方描述，在 iOS 上 WXS 比 JavaScript 代码快 2~20 倍，而在 Android 上基本没什么差异。例如，要实现左滑的功能，如果要频繁地进行逻辑层和渲染层的通信和计算，当左滑列表过长的时候，在 iOS 上就会有不顺畅的情况，对于这种情况，使用 WXS 会有明显的改善

### 总结

显然频繁的调用 setDate,会导致逻辑层和渲染层频繁的通信，导致页面卡顿，这是开发者和小程序不希望的，小程序为我们提供了三种开发方式和建议

1. 自定义组件通过在渲染层和逻辑层都维护一个 Virtual DOM 树，数据更新，先在逻辑层计算过后，只传 Diff 结果到渲染层，减少了逻辑层与渲染层过多的通信。
2. 原生组件直接与逻辑层通信，节省交互过程的中转通信次数，同时减少 WebView 的计算和渲染工作。
3. WXS 干脆直接在渲染层进行计算，不与逻辑层通信，解决双线程频繁通信计算，导致的卡顿问题。

## 开发者工具原理设计

既然小程序是双线程的，微信开发者工具是如何去为开发者呈现逻辑层和渲染层供开发者调试的呢？

微信开发者工具基于 NW.js,使用 Node.js、Chromium 和系统 API 来实现底层模块，使用 React、Redux 等前端技术框架来搭建用户交互层，实现同一套代码跨 macOS 和 Windows 平台使用。

### 逻辑层模拟

小程序的逻辑层在 iOS 中是运行在 JavaScriptCore 中的，在 Android 中是运行在 X5 JSCore/V8 解析引擎中的。而在开发者工具里，我们可以使用 WebView 来模拟小程序的 JSCore 环境。

WebView 是一个 Chrome 的 <webview/> 标签，它是采用独立的线程运行的。因为它是一个逻辑层，而不是渲染层，所以对用户是不可见的。

同时，我们可以复用 WebView 的调试工具，使用 Chrome Devtools 的 Sources 面板来调试逻辑层的 JS 代码。需要注意的是，小程序中的逻辑层是一个纯 JavaScript 的解释执行环境，这个环境没有浏览器相关的接口，如果开发者在代码中写了相关的逻辑，虽然在工具中跑得顺畅，但在真机上很可能会报错白屏。所以，在开发者工具中还需要对 WebView 的逻辑层中不支持的对象和接口进行隔离处理，通过局部变量化，使开发者无法在小程序代码中正常使用这些对象，从而避免不必要的错误。

我们可以在 console 面板上输入 document,看到开发者工具的逻辑层代码：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/mini-开发者工具.png"/>

可以清楚的看到，里面包含了框架，渲染相关和每个页面的业务逻辑代码。

我们还可以从逻辑层的代码中看到这样一段代码：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/mini-开发者工具1.png"/>

小程序监听 babel 依赖是否加载完毕，会设置一个 babellock 锁，只有当 bebel 依赖包都加载完，才加载主包和开发者的业务代码包。

### 渲染层模拟

同样地，开发者工具使用 Chrome 的 <webview/> 标签来加载渲染层页面，每个页面启动一个 WebView，通过基础库维护和管理页面栈。

小程序的渲染层的运行环境是一个 WebView，而小程序构建页面使用 WXML，显然 WebView 无法直接理解 WXML 标签。那渲染层的调试工具又是怎么来的呢？Chrome Devtools 自带的 Element 面板调试的是逻辑层 WebView 的节点（HTML 节点），并不能调试当前渲染层页面的节点。所以开发者工具将默认的 Element 面板隐藏，开发了 WXML 面板插件给开发者进行调试辅助。开发者工具会在每个渲染层的 WebView 中注入界面调试的脚本代码，这些节点的调试信息最终会通过底层通信系统转发给 WXML 面板进行处理。

小程序的 WXML 生成 HTML，中间会经过 AST 转成 JavaScript 函数，在小程序运行时再根据该函数生成页面结构的 JSON，最后通过小程序组件系统，在虚拟树对比后将结果渲染到页面上。

### 渲染层与逻辑层的通信模拟

开发者工具有一个消息中心底层模块，维持着一个 WebSocket 服务器，通过该 WebSocket 服务，开发者工具与逻辑层的 WebView、渲染层页面的 WebView 建立长连，同时使用 WebSocket 的 protocol 字段来区分 Socket 的来源。

具体包括：

(1) 逻辑层和渲染层分别通过各自的 WebView 与消息中心进行通信，从而模拟双线程通信。

(2) WebView 的调试信息（如 DOM 树、节点样式、节点变化、选中节点的高亮处理、界面调试命令处理等）会通过 WebSocket 经由开发者工具转发给 WXML 面板进行处理。

(3) 借助开发者工具的 BOM 对象，wx.request 使用 XMLHttpRequest 模拟，wx.connectSocket 使用 WebSocket，来做到对外三方服务的通信调试。

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/mini-微信开发者工具2.png"/>

> 我们在调试代码的时候会发现，虽然我们在开发的时候创建了许多 JS 文件，但最终运行在小程序逻辑层的只有 app-service.js 这一个文件。这是因为在代码上传之前，微信开发者工具会在服务器编译过程中将每个 JS 文件的内容进行处理和隔离，再按一定的顺序合并成 app-service.js，达到最终加载单个 JS 文件的效果。

### 小程序执行代码

在小程序运行时，逻辑层直接加载 app-service.js，渲染层使用 WebView 加载 page-frame.html，在确定页面路径之后，通过动态注入 script 的方式调用 WXML 文件和 WXSS 文件生成的对应页面的 JS 代码，再结合逻辑层的页面数据，最终渲染出指定的页面。

## 总结：

1. 开发者工具通过 WebView 分别模拟了渲染层和逻辑层
2. 开发者的调试面板，其实是用的逻辑层中 WebView 这样一个内嵌浏览器中的调试面板，只不过 WebView 被开发者工具隐藏了
3. 由于调试面板是逻辑层的，自然无法调试渲染层页面的节点，于是隐藏了逻辑层调试面板中的 Element, 自研了 WXML 面板，开发者工具在每个渲染层的 WebView 注入界面调试的脚本代码，这些节点的调试信息，最终会通过底层通信系统转发给 WXML 面板处理。
4. 逻辑层和渲染层都通过微信客户端这样一个系统层通信，采用 webSocket 长连接的方式通信

---

## 参考资料

- 《小程序开发原理与实战》

---

# 关于作者

大家好，我是程序员高翔的猫，MonkeyDesign 用户体验部前端研发。

**加我的微信，备注：「个人简单介绍」+「组队讨论前端」**， 拉你进群，每周两道前端讨论分析，题目是由浅入深的，把复杂的东西讲简单的，把枯燥的知识讲有趣的，实用的，深入的，系统的前端知识。

<a name="微信"></a>
<img width="400" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/weixin.jpg"/>

# 我的公众号

更多精彩文章持续更新，微信搜索：「高翔的猫」第一时间围观

<a name="公众号"></a>

<img src="https://mp.weixin.qq.com/mp/qrcode?scene=10000004&size=102&__biz=MzkxNjMxNDU0MQ==&mid=2247483692&idx=1&sn=2d2baccebfd92fbf6d0506d3c75b3ade&send_time=" data-img="1" width="200" height="200">
