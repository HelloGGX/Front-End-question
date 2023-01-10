## React服务端渲染演进

> 概述：本文是服务端渲染架构演进的梳理，以react为例子，详细说明了从服务端渲染到目前的最新岛群架构，设计思路和所要解决的问题。

### 1. 什么是服务端渲染
常见的单页应用会采用**客户端渲染**的方式：提供一个渲染的容器, 而页面的加载与更新完全交由js控制处理。因此页面的展示完全受控于js。
```html
<div id="root"></div>
```
从用户首次打开应用等待本地的js包加载完成，到执行渲染函数，再到通过特定事件（这个事件可能是自动的，比如useEffect）触发http请求，服务器解析该请求返回响应，并渲染出真正有意义的内容，在等待期间用户可能看到的都是空白页面。

![](https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-SSR1.png)

而js的包体积较大的话，用户等待的时间会更长一些。而这直接会影响用户体验的指标之一FCP（首次内容渲染）。

当然我们会有如下的优化方案，比如：

* 代码拆分
* 预加载js
* 按需加载js包
* 打包的时候压缩js包的体积
* 使用tree-shaking删除未使用的代码

以上的方案都是为了缩短上图中FP的时间，减少首屏渲染时间，但并不能解决用户在首次访问应用等待js加载时看到白屏的问题。而在客户端渲染的情况下，因为大量的js请求和瀑布式的网络请求（例如API响应）可能会导致有意义的内容无法快速渲染，从而使爬虫无法对其进行索引，导致SEO不友好。

而服务端渲染（SSR）在服务器上把你的React组件渲染成HTML，并把它发送给用户。虽然HTML的互动性不强（除了简单的原生自带的互动，如链接和表单输入）。但它至少可以让用户在JavaScript仍在加载时看到页面内容。

React中的SSR的步骤如下：

1. 服务器：请求获取整个页面的数据
2. 服务器：将整个应用渲染成HTML并在响应中发送
3. 客户端：加载整个应用程序的JavaScript代码
4. 客户端：将JavaScript逻辑与服务器生成的HTML连接起来（hydration）

于CSR相比, SSR有相对较少的js加载，因此FCP到TTI之间的时间更短，用户体验相对更好。

![](https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-SSR2.png)

由于首屏能快速返回html页面, 搜索引擎更容易爬取生成快照，因此SEO更友好。
SSR只是做到了让用户能更快看到页面，但是用户并不能马上交互，给用户一种页面卡顿的错觉。

### 2. 服务端渲染存在的问题

从上一节的React SSR的渲染步骤可以看出，每一步都必须在下一步开始之前一次性完成。这样会导致当某个模块较慢时（这里的慢指：请求接口，加载js, 注水等），会拖慢整个页面SSR渲染，总结下来会有有如下三个问题：

1. 必须先收集服务器上的所有数据，然后才能开始向客户端发送任何HTML
2. 必须在客户端上加载所有组件的JavaScript，然后才能开始对模块中的任何一个进行hydration
3. 必须等待所有模块都注水完成后才能与它们中的任何一个交互

由此看出SSR的渲染不够灵活，并且低效，优化空间巨大。

### 3. 如何解决以上问题
在react中的传统的服务端渲染方案采用提供的API：
```js
const htmlString = ReactDOMServer.renderToString(element)
```
输入 React 组件（准确来说是ReactElement），输出 HTML 字符串, 期间将等待服务端fetch到所需所有数据后才能开始输出HTML，之后由客户端 hydrate API 对服务端返回的视图结构附加上交互行为，完成页面渲染。为了加快这个速度，我们需要减少对HTML输出的阻塞时间。

react提供了renderToNodeStream方案：
```js
const htmlStream = ReactDOMServer.renderToNodeStream(element)
```
他将HTML输出为nodejs中的可读流，并将可读流输送到一个可写流中（如HTTP response对象）具体流程如下：

1. 服务器端将流的头部发送到客户端，流的头部通常包含HTML的头部信息、样式表等内容，一般长这样：
```js
response.write('
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <link rel="stylesheet" href="/css/main.css" />
  </head>
  <body>
    <div id="root">
')
```
2. 客户端接收到流的头部后，开始渲染页面，
3. 服务器端不断将流的其他部分发送到客户端，客户端继续渲染，
4. 当服务器端发送完所有的流后，客户端渲染完成。

完整示例代码如下：

```js
import { renderToNodeStream } from 'react-dom/server';
import Frontend from '../client';

app.use('*', (request, response) => {
  response.write('<html><head><title>Page</title></head><body><div id="root">');
  const stream = renderToNodeStream(<Frontend />);
  //将流的数据发送到HTTP响应中，发送完流的所有数据后不关闭管道。
  stream.pipe(response, { end: 'false' });
  // 显式地关闭管道，并执行回调
  stream.on('end', () => {
    response.end('</div></body></html>');
  });
});
```
这样我们在请求接口时，就能在第一时间收到响应数据，浏览器会开始渐进渲染，我们的FCP时间会提前：
![](https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-SSR3.png)

但这仍然解决不了，需要较大数据量的组件请求，阻塞其他组件的问题。
而React提供了一个内置组件：**Suspense**

比如下面的代码：

```jsx
<Layout>
  <NavBar />
  <Sidebar />
  <Suspense fallback={<Spinner />}>
     <Comments />
  </Suspense>
</Layout>
```
通过将<Suspense>包裹<Comments>组件，告诉React：**Suspense之外的其他组件不需要等待Comments组件的评论数据，而在该组件获取到完整数据之前，用一个Spinner组件占位**。

当Comments组件获取到完整评论数据之后，React会将额外的HTML发送到同一个流中，并附加一个最小的内联`<script>`标签，将HTML替换到正确的地方, 示例代码如下：

```html
<div hidden id="comments">
  <!-- Comments -->
  <p>First comment</p>
  <p>Second comment</p>
</div>
<script>
  document.getElementById('sections-spinner').replaceChildren(
    document.getElementById('comments')
  );
</script>
```
**这就解决我们前面说的第一个问题：必须先收集服务器上的所有数据，然后才能开始向客户端发送任何HTML。大数据量的组件不会阻塞应用其他模块的渲染。**

总结下上面采用的技术方案：

* 我们采用Streaming HTML传输的方式（renderToNodeStream），逐步发送数据块到客户端，让客户端渐进式的渲染HTML, 缩短FCP的时间
* 采用Suspense组件包裹耗时的组件，让其在渲染期间用fallback占位，而不阻塞其他部分的渲染

在HTML渲染完毕之后，用户是不能马上交互的，就像枯萎的植物，没有生机，我们需要“注水”让它活过来。
我们需要等待js包加载完毕才能交互，如果加载的js包很大，我们等待的时间会更长一些，而前面提到了很多方案可以减小包的大小，比如：代码拆分

```jsx
import { lazy } from 'react';

const Comments = lazy(() => import('./Comments.js'));

<Suspense fallback={<Spinner />}>
  <Comments />
</Suspense>
```
事实上，上面的Suspense除了告诉React: **其他模块不需要等待Comments组件的数据请求，还告诉React：其他模块不需要等待Comments组件的js加载完成，而是在Comments组件的js加载期间用fallback占位，其他组件正常走注水流程，当Comments组件的js加载完成之后，其他组件可能已经可以交互了**

如果Comments组件是一个特别“expensive”的组件，那么该组件不会阻塞其他模块的Hydration

**这就解决了上面提到的第二个问题：必须在客户端上加载所有组件的JavaScript，然后才能开始对模块中的任何一个进行hydration**

在上面的例子中，只有评论组件被包裹在Suspense中，所以对于页面的其他部分进行注水是一次性完成的。然而，我们可以通过在更多的地方使用Suspense来解决这个问题。例如，让我们把侧边栏也包起来：

```jsx
<Layout>
  <NavBar />
  <Suspense fallback={<Spinner />}>
    <Sidebar />
  </Suspense>
  <Suspense fallback={<Spinner />}>
    <Comments />
  </Suspense>
</Layout>
```
那问题来了，用户去点击了评论组件，希望看到评论组件的交互结果，但是Sidebar和Comments组件都在注水过程中，还无法交互，理想情况下Comments组件先注水完成当然是最好的，那React如何处理这里的优先级呢？

![](https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-SSR4.png)

React会在事件捕获阶段，同步的对Comments组件进行注水，通俗的讲：**交互哪里，优先注水哪里**
最终，评论组件将被及时注水，以处理点击和响应的互动。然后，React没有什么紧急的事情要做了，便开始给侧边栏注水。
**而这刚好解决了前面的第三个问题：必须等待所有模块都注水完成后才能与它们中的任何一个交互**

我们不必等待所有模块都注水完成后，才对其中任何一个进行交互，React尽可能早地开始给所有东西注水，它根据用户的交互情况，优先考虑屏幕上最紧急的部分。
随着你在整个应用中采用Suspense，渲染边界将变得更加细化，那么**Selective Hydration**的好处就更加明显。

> 在遇到嵌套Suspense的场景，React总是以父级优先的顺序进行hydrates，

以上便是前端服务端渲染的优化演进过程，而目前最新的服务端渲染的架构模式是由Katie Sylor-Miller和Jason Miller推广的 [Islands architecture](https://jasonformat.com/islands-architecture/)，它是一种基于组件的架构风格，结合了不同渲染技术的理念，如服务器端渲染、静态网站生成和部分水化。来实现极致的加载性能和用户体验。

Islands architecture渲染理念总结为：**静态组件只有服务器渲染的HTML，交互式组件才需要脚本**

它用静态和动态的岛屿对页面进行分割，页面的静态区域是纯粹的非交互式HTML，不需要
hydration 。动态区域是HTML和脚本的组合，能够在渲染后自我注水，而每个岛屿都是一块独立的应用，可以独立交付和维护。
Islands architecture目前实现的框架较少，目前比较出名的有Astro和Marko，有兴趣可以官网了解。
