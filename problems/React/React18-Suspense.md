# 深入React18的Suspense特性
## 1.1 Suspense在异步请求中的妙用
说到react的异步请求我们通常会这样写异步请求代码：

```typescript
function List({pageId}) {
    const [items, isLoading] = useData(pageId);

    if (isLoading) return <Spinner />;

    return items[pageId].map(item =>
        <li>{item}</li>
    );
}
```
但以上的代码存在两个问题：
1. 每当我们删除、更新数据或者类似其他API请求的时候， 页面可能会有不同的反馈，可能不需要Spinner, 也可能需要其他类型的loading，那么我们就需要时刻关心: 当isLoading为true的时候到底需要返回什么。
2. 如果我们需要改变加载状态的显示方式，我们必须修改获取数据的代码，可能涉及移动大量的代码， 比如： 我们需要将两个模块的独立的loading合并到一起loading, 那么我们就需要移动和合并这些数据查询，比如PromiseAll来得到两个模块获取数据的状态。

以上的两个问题给开发者造成了不必要的麻烦，仅仅是为了调整加载数据的视觉外观。针对以上两种问题，我的解决方案是：将 数据请求和指定loading状态进行关注点分离：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/GIF 2021-12-9 22-49-14.gif"/>

我们回到回到获取数据本身， useData只需要负责获取数据，不需要关心loadig:

```typescript
function List({pageId}) {
    const items = useData(pageId);
  
    if (isLoading) return <Spinner />;

    return items[pageId].map(item =>
        <li>{item}</li>
     );
}
```
那么Loading怎么办呢？ 我认为Loading的正确位置应该在JSX中,，如下：

```typescript
<Suspense fallback = { <Spinner /> }>
	<List pageId = { pageId } />  
</Suspense>
```
用一个特殊的父组件告诉React,: 这里用一个Spinner来作为组件还没渲染出来的占位。 这不是一个编程概念，这个类似于设计师在UI设计中使用骨架和加载状态时用到的方式:  你有一个UI和当这个UI不可用时的备用方案。

因此JSX 才是Loading加载状态设置的正确位置，我们完全不需要考虑加载状态什么时候需要true和什么时候需要false。举个例子： 有两个兄弟级别的组件 Header和 List。

```typescript
<Suspense fallback = { <Skeleton /> }> 
	<Header />  
  	<List pageId = { pageId } /> 
</Suspense>
```

假如设计变了加载的视觉外观， Header组件需要直接渲染出来， 而List 组件有自己独有的加载视觉外观， 我们不需要考虑动任何请求相关的函数， 就能达到目的：

```typescript
<Suspense fallback = { <Skeleton /> }> 
      <Header />
      <Suspense fallback = { <ListPlaceholder /> }>
            <List pageId = { pageId } />
       </Suspense>
 </Suspense>
```
这样的一个简单的设计，解耦了数据请求和加载状态的关系，使局部Loading有更多的可能性。

## 1.2 Suspense与Server Components
不仅如此Suspense还可以用于代码拆分中。 我们知道在React16-17中已经可以通过React.lazy可以实现组件的动态import， 当还在下载某些组件或者页面的javascript时，可以通过Suspense来占位显示加载状态，这些动作都是在客户端执行的，但是到了React18, Suspense可以在服务端运行了，这意味着在服务器上的代码拆分，也支持Suspence占位， 我们称之位“streaming server rendering”， 在从服务器中下载HTML过程中，如果某个位置的JSX还没有准备好，React会先发送HTML加载占位符渲染，直到实际的内容可用。

基于Suspense这样的特性，React18提出了一个新的概念： “Server Components”, 把Suspense上升到了一个新的高度， 再次扩展了我们对组件的定义：

>组件不仅可以读取服务端返回的数据，还可以在服务端就先请求数据再返回客户端

### 1.2.1 服务端渲染
首先我们回忆一下什么是“服务端渲染”：

>SSR（服务端渲染）是一种关注何处渲染HTML页面的模式，代表在服务器端完成数据和模板转换成HTML并返回给客户端

传统的客户端渲染中，页面需要等待JS加载完成才能渲染出来，如下图所示：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/jsrender.png"/>

网站在首次加载JS的时候可能有较长的白屏时间， 用户体验不是很好。而服务端渲染直接从接口返回HTML, 用户不需要等待所有JS的加载完，最先看到的是静态的HTML，但是这时的HTML是不可交互的，我们需要javascript来实现交互， 那么下一步我们需要在客户端未该HTML片段注入javascript, 将交互逻辑连接到HTML上，这样的过程我们称之为： **Hydration**。

>Hydration发生在整个应用程序都加载完成之后，一次性注入

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/hydrate.png"/>

服务端渲染的好处是在单页应用的情况下，很好的支持了SEO,并且加快了页面的显示（在javascript chunk 还在加载时就能渲染页面），但是也随之带来了一个弊端，比如： 一个服务端渲染的页面中，只有某个特定的组件加载数据很慢（要等带接口返回或者javascript包很大下载完成才能交互），其他模块加载都很快，这会导致要等待该模块加载出来，整个HTML才返回，从而拉低了整个应用程序的速度。这是因为服务端渲染返回的是HTML, 是一个要么都渲染出来，要么都不渲染出来的过程。 举个实际的例子：

```typescript
<Layout>
  	<NavBar />
  	<Sidebar />
  	<RightPane>
  		<Post />
  		<Comments />
  	</RightPane>
</Layout>
```
以上的代码在浏览器呈现的样子为：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/temp.png"/>

在这个例子中，评论列表是应用程序的一部分，它涉及到昂贵的API请求和大量的javascript代码，因此可以肯定的说，在服务端渲染中评论模块会阻塞其他模块的展示。

到了React18中，支持了服务器端的Suspense, 这意味着你可以用Suspense包裹住某个加载拖后腿的模块，告诉React18延迟加载这个模块，并用加载Loading占位, 其他模块正常渲染。而这些过程都发生在页面加载javascrip或者React之前，从而显著改善了用户体验 。

```typescript
<Layout>   
  <NavBar />   
  <Sidebar />   
  <RightPane>   
  	<Post />
  	<Suspense fallback={<Spinner />}>
  		<Comments />
    </Suspense>
  </RightPane> 
</Layout>
```
<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/tempLoading.gif"/>

在React18中，整个应用不用等待Comments组件加载，就可以完成Hydration, 也就意味着当Comments还在加载时，其他组件已经可以交互。
