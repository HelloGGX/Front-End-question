# 什么是 Fiber

在 react 中通过数据映射到界面的概念，通常表示为：

```js
view = f(data)
```

因此渲染一个 React 组件相当于调用一个函数，该函数的主体包含对其他函数的调用，而我们的计算机会通过**调用堆栈**的结构记录函数的执行顺序。当一个函数被执行的时候，一个新的栈帧被添加到栈顶，而该栈帧表示该函数执行的工作。

在 react 中，一个 Fiber 对应一个堆栈帧， 而具体到 Fiber, 它是一个 javascript 对象，对应某一个组件实例，dom 节点。

我们写的 jsx 语法首先会通过`babel-plugin-transform-react-jsx` 将 jsx 转译为 React.createElement 的形式，接着转换为 Fiber 树来映射真实的 dom 结构，每一个节点都对应一个 Fiber，而 fiber 对象中的 tag 属性用 0-24 表示该 fiber 所关联的标签类型：

```js
export const FunctionComponent = 0;
export const IndeterminateComponent = 2;
export const HostRoot = 3;
export const HostPortal = 4;
export const HostComponent = 5;
export const HostText = 6;
export const Fragment = 7;
export const Mode = 8;
export const ContextProvider = 10;
export const ForwardRef = 11;
export const Profiler = 12;
export const Suspensecomponent = 13
;export const MemoComponent = 14;
export const SimpleMemoComponent = 15;
export const LazyComponent = 16;
export const IncompleteclassComponent = 17;
export const DehydratedFragment = 18;
export const SuspenseListcomponent = 19;
export const ScopeComponent = 21;
export const OffscreenComponent = 22;
export const LegacyHiddenComponent = 23;
export const CacheComponent = 24;
```

在旧版本的 react 中，react 在渲染 ui 时，需要同步递归遍历整个组件树并为每个组件执行任务，每个组件的执行都会放到堆栈顶部，直到堆栈为空才给出 js 主线程。如果组件树很大，就会长时间占用 js 主线程，如果浏览器需要做一些高优先级的事情，比如处理用户的输入或者保持动画的流畅性，它就必须等待渲染完成，可能超过 16ms 导致视觉上的不流畅。

于是 react 需要实现一种机制：能中断递归遍历造成的堆栈增长，把 js 主线程交给浏览器渲染和优先级较高的任务，并在浏览器空闲时间恢复渲染。将一个长任务拆分为多个时间片分开执行的方式，避免 js 线程长时间被占用，优化用户的操作体验。

效果如下图所示：

![](https://files.mdnice.com/user/23305/1e243b35-7317-4449-b86a-88e00ffcca4f.png)

基于以上的背景，react 团队需要重新实现堆栈，于是就有了开头讲到的 Fiber 的概念，对比之前的堆栈的形式，可以将一个 Fiber 理解为一个**virtual stack frame.**

而一个 Fiber 只包含一个组件实例和 dom 节点的相关信息是远远不够的，它还需要表达节点之间的关系，于是它还需要保存父节点的，子节点，兄弟节点的信息，每一个 fiber 通过 return，child, sibling 三个属性建立联系：

- return: 指向父级 Fiber 节点
- child：指向子级 Fiber 节点
- sibling: 指向兄弟 fiber 节点

![](https://files.mdnice.com/user/23305/4c60e960-ba2f-4ac6-af92-4a32c6c9016d.png)

我们可以看到，fiber 之间是链表和指针的形式建立联系的，而 fiber 用双向链表的优势有：

1. 调整节点位置灵活，只需要调整指针指向
2. 使得插入，删除，时间复杂度为 o(1)
3. 更方便地查找下一个 fiber
4. 可以随时从某个节点还原整课树

## react 更新机制

那么从组件的挂载和更新，react 内部是怎样运作的呢？

react 首先会 babel-plugin-transform-react-jsx 将 JSX 转译为 React.createElement 的形式， 紧接着调用 render 函数：

```js
ReactDOM.render(<App/>, document.getElementbyId('app'));
```

会创建一个 rootFiber 对象作为 workInProgress root，表示正在构建的 fiber 树的根，该对象有个 alternate 属性，指向该 fiber 节点对应的当前视图层渲染的节点 current root，而初次构建，这个属性为 null, react 采用两棵树来表示构建的视图：workInProgress 树（在内存中构建的树）和 current 树（渲染视图），两课树用 alternate 指针相互指向，在下一次渲染时，直接复用 workInProgress tree 作为 current tree 渲染视图，上一次的 current tree 作为 workInProgress tree 在内存中继续构建，这样可以防止只用一棵树更新状态的丢失情况，又加快了 dom 节点的替换和更新

![](https://files.mdnice.com/user/23305/dc1b77d9-8034-4d62-8a7d-84f083dee551.png)

> canvas 绘制动画的时候，如果上一帧计算量比较大，导致清除上一帧画面到绘制当前帧画面之间有较长间隙，就会出现白屏。为了解决这个问题，canvas 在内存中绘制当前动画，绘制完毕后直接用当前帧替换上一帧画面，由于省去了两帧替换间的计算时间，不会出现从白屏到出现画面的闪烁情况。这种在内存中构建并直接替换的技术叫做双缓存

接下来我们会深度调和这个 rootFiber 的子节点，以深度优先遍历的方式完成整个 fiber 树的遍历，包括 fiber 节点的创建。

在调和的过程中分为两个阶段：render 和 commit

### 1. render 阶段

该阶段通过 react 自己实现的 requestIdleCallback 版本调用 workLoop， 而在 workLoop 中会通过 while 循环不断判断浏览器主线程是否有空余时间。

```js
function workLoop(deadline) {
      let shouldYield = false;
      while (nextUnitOfWork && !shouldYield) {
        nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
        shouldYield = deadline.timeRemaining() < 1;
      }
      requestIdleCallback(workLoop);
 }
```

如果没有空余时间，React 会暂停当前的工作单元，将控制权交还给主线程，并让浏览器渲染此时完成的任何内容， 然后，在下一帧中，React 从它停止的 fiber 节点开始继续构建树。

当它有空余时间时，会遍历它的子节点， 期间会做 3 件事：

1. 构建 Fiber 的子级（每个 Fiber 都有一个 effectTag 表示该 fiber 当前处于创建、更新还是删除）
2. 返回下一个 Fiber
3. Fiber 添加对应的 dom 节点，让每个 fiber 映射真实的 dom 节点

而遍历的过程又分为两个过程：向下调和 beginWork() 和向上调和 completeUnitOfWork()

**beginWork：**

1. 对于组件，执行部分生命周期，执行 render ，得到最新的 children 。
2. 向下遍历调和 children ，复用 oldFiber ( diff 算法)
3. 打不同的副作用标签 effectTag ，比如类组件的生命周期，或者元素的增加，删除，更新。

**completeUnitOfWork：**

1. 当某个节点不存在子节点，就要从这个节点离开了，改执行 completeUnitOfWork。completeUnitOfWork 有个内层 do while 循环，从当前节点沿着 Fiber 树往上爬。每次循环经过一个节点，都会将 effectTag 的 Fiber 节点向上合并到 effectList，直至根节点。effectList 是个单向链表。在 commit 阶段，将不再需要遍历每一个 fiber ，只需要执行更新 effectList 就可以了。

2. completeWork 阶段对于组件处理 context ；对于元素标签初始化，会创建真实 DOM ，将子孙 DOM 节点插入刚生成的 DOM 节点中；会触发 diffProperties 处理 props ，比如事件收集，style，className 的处理

以上的构建顺序以深度优先遍历的方式构建 WorkInProgress Tree: 如果没有子节点，那么就返回兄弟节点，对兄弟节点进行深度优先遍历，直到没有兄弟节点为止，紧接着开始回溯到 root 节点。 重要的是这些都不是递归的，而是使用了一个 while 循环来完成， 因此我们的调用堆栈不会增长，顶部的堆栈就是当前的 fiber 节点，动图如下：

![](https://files.mdnice.com/user/23305/eeee7485-6c04-489b-b418-f6de17efbf4f.gif)

### 2. commit 阶段

commit 逻辑就会用到我们在构建 Fiber 节点时，在 Fiber 中添加的 dom 节点。深度优先递归地去操作 dom 节点并渲染到页面上，并且 currentRoot 指向当前的 WorkInProgress tree, 后面它将作为 fiber 节点的 alternate 属性， 后面会作为 diff 的依据，该阶段还会做下面的一些事情：

1. 一方面是对一些生命周期和副作用钩子的处理，比如 componentDidMount ，函数组件的 useEffect ，useLayoutEffect；
2. 另一方面就是在一次更新中，添加节点（ Placement ），更新节点（ Update ），删除节点（ Deletion ），还有就是一些细节的处理，比如 ref 的处理。

需要注意的是：该阶段是同步一次性执行的，不会被中断。

### 3. 调用 setState

当 setState 时，会重新执行 render 函数， 构建一个新的 rootFiber 对象，作为 workInProgress Tree，将当前的 current tree 作为 workInProgress Tree 的 alternate 属性值。
伪代码如下：

```js
function render(element, container) {
    wipRoot = {
        dom: container,
        props: {
            children: [element],
        },
        alternate: currentRoot,
    };
    nextUnitOfWork = wipRoot;
}
```

后面该属性会作为 oldFiber 与新的 workInProgress 进行 diff 比较，该过程会在第一步调和子节点构建新的 Fiber 树的时候进行，比较条件如下：

- 如果旧的 fiber 和新的 element 有同样的 type，我们可以保留 DOM 节点，并使用新的属性进行更新
- 如果 type 不同，且新的 element 存在，意味着我们需要创建一个新的 DOM 节点
- 如果 type 不同，且旧的 fiber 存在，我们需要删除这个旧的节点

通过 diff 比较，我们会在 fiber 对象上添加 effectTag 属性，表示该 fiber 当前处于创建、更新还是删除状态， 以便在后面的 commit 阶段根据 effectTag 对 dom 进行操作。

在前面讲过：completeUnitOfWork 有个内层 do while 循环，从当前节点沿着 Fiber 树往上爬。每次循环经过一个节点，都会将 effectTag 的 Fiber 节点向上合并到 effectList，直至根节点。effectList 是个单向链表。在 commit 阶段，将不再需要遍历每一个 fiber ，只需要执行更新 effectList 就可以了。
