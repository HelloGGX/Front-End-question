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

在React Fiber中，每个 Fiber 节点的处理称为一个工作单元（unit of work），这些工作单元是由React的调度器管理的。每个工作单元代表一个需要处理的更新或任务，比如组件的渲染或状态更新。React Fiber的调度器会根据不同的优先级来安排这些工作单元的执行顺序。

## 如何给任务分配优先级

React Fiber的优先级分配主要基于以下几个因素：

1. **任务的类型**：用户交互相关的任务通常会被分配较高的优先级，而后台任务则分配较低的优先级。
2. **任务的紧急程度**：某些任务需要立即响应用户操作，比如输入框的输入事件，这类任务会被分配最高优先级。
3. **任务的影响范围**：涉及到界面大范围更新的任务可能会被分配较高的优先级，以确保用户看到的界面尽快更新。
4. **任务的时间需求**：一些任务可以延迟执行而不影响用户体验，这些任务会被分配较低的优先级。

### 分配优先级的机制

React Fiber通过一个调度器来管理和分配任务的优先级。调度器会根据任务的紧急程度、类型和影响范围，使用不同的优先级类别来调度这些任务。以下是具体的分配过程：

1. **任务分类**：当任务进入调度器时，首先会根据任务的类型和上下文进行分类。
2. **优先级赋值**：每类任务会被赋予对应的优先级。例如，用户输入事件会被赋予立即优先级，而后台数据同步任务会被赋予低优先级。
3. **任务调度**：调度器根据任务的优先级，将高优先级的任务安排在前面执行。如果当前正在执行低优先级任务，调度器可以中断它以执行更高优先级的任务。

### 示例代码

以下是一个示例代码，展示了如何在React中利用不同优先级来调度任务：

```javascript
import { unstable_scheduleCallback as scheduleCallback, unstable_LowPriority as LowPriority, unstable_UserBlockingPriority as UserBlockingPriority } from 'scheduler';

// 高优先级任务
scheduleCallback(UserBlockingPriority, () => {
  console.log('高优先级任务执行');
});

// 低优先级任务
scheduleCallback(LowPriority, () => {
  console.log('低优先级任务执行');
});
```

在这个示例中，`scheduleCallback`函数用于调度任务，并为任务分配不同的优先级。`UserBlockingPriority`表示用户阻塞优先级，而`LowPriority`表示低优先级。调度器会首先执行高优先级任务，然后在浏览器空闲时执行低优先级任务。

通过这样的优先级调度机制，React Fiber能够更高效地管理任务，提升应用的性能和响应速度。

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

### 1. React Fiber 的 Render 阶段

在 React Fiber 架构中，Render 阶段通过 React 自己实现的 `requestIdleCallback` 版本来调用 `workLoop`，其目的是在浏览器有空闲时间时处理 Fiber 节点。以下是 `workLoop` 函数的基本工作原理：

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

### Render 阶段的工作流程

1. **处理 Fiber 节点**：
   - 在 `workLoop` 中，通过 `while` 循环判断浏览器主线程是否有空闲时间，每次迭代处理一个 Fiber 节点，每个 Fiber 节点的处理称为一个工作单元（unit of work）。这个工作单元包括计算新的状态、生成新的子节点、准备更新等。

2. **时间片切换**：
   - 如果没有空余时间，React 会暂停当前的工作单元，将控制权交还给主线程，并让浏览器渲染此时完成的任何内容。在下一帧中，React 会从上次停止的 Fiber 节点继续构建树。

3. **遍历 Fiber 树**：
   - 当有空余时间时，React 会遍历 Fiber 树的子节点。期间主要做以下三件事：
     1. 构建 Fiber 的子级（每个 Fiber 都有一个 `effectTag` 表示该 Fiber 当前处于创建、更新还是删除状态）。
     2. 返回下一个 Fiber。
     3. 将对应的 DOM 节点添加到 Fiber 中，使每个 Fiber 映射到真实的 DOM 节点。

### 向下调和（beginWork）和向上调和（completeUnitOfWork）

- **beginWork**：
  1. 对于组件，执行部分生命周期方法，执行 `render` 方法，得到最新的子节点。
  2. 向下遍历调和子节点，复用 `oldFiber`（diff 算法）。
  3. 标记不同的副作用标签 `effectTag`，例如类组件的生命周期方法，或者元素的增加、删除、更新。

- **completeUnitOfWork**：
  1. 当某个节点不存在子节点时，执行 `completeUnitOfWork`。`completeUnitOfWork` 内部有一个 `do while` 循环，从当前节点沿着 Fiber 树往上爬。每次循环经过一个节点，都会将 `effectTag` 的 Fiber 节点向上合并到 `effectList`。`effectList` 是一个单向链表，包含了所有需要更新的 DOM 操作。在提交（commit）阶段，React 会一次性应用这些变更，确保 DOM 操作尽可能高效。
  2. `completeWork` 阶段处理组件的上下文，对于元素标签会初始化并创建真实的 DOM 节点，将子孙 DOM 节点插入刚生成的 DOM 节点中，触发 `diffProperties` 处理属性，例如事件收集、`style`、`className` 等。

### 构建 WorkInProgress Tree

- React 以深度优先遍历的方式构建 WorkInProgress Tree。如果没有子节点，就返回兄弟节点，对兄弟节点进行深度优先遍历，直到没有兄弟节点为止，然后开始回溯到根节点。这个过程使用 `while` 循环而不是递归，因此调用堆栈不会增长，顶部的堆栈就是当前的 Fiber 节点。

通过上述过程，React Fiber 能够高效地调度和执行渲染工作，使得应用在响应速度和性能上都有显著提升。

![](https://files.mdnice.com/user/23305/eeee7485-6c04-489b-b418-f6de17efbf4f.gif)

### 2.commit阶段

提交阶段是 React Fiber 渲染过程的第二个阶段。它在调和阶段完成后执行，主要负责将更新的 DOM 应用到页面上。提交阶段可以分为三个子阶段：

1. **Before Mutation Phase**（突变前阶段）
2. **Mutation Phase**（突变阶段）
3. **Layout Phase**（布局阶段）

### 1. 突变前阶段（Before Mutation Phase）

在突变前阶段，React Fiber 会执行一些预处理操作，这些操作不会对 DOM 进行修改。主要包括：

- 触发生命周期方法 `getSnapshotBeforeUpdate`：在组件更新前调用，允许组件捕获 DOM 状态。
- 执行 `useEffect` 和 `useLayoutEffect` 清理函数：在实际突变发生之前清理副作用。

### 2. 突变阶段（Mutation Phase）

在突变阶段，React Fiber 会将需要的变化实际应用到 DOM 中。主要的操作包括：

- **DOM 更新**：React Fiber 会根据调和阶段生成的 Fiber 树，执行相应的 DOM 更新操作。这包括插入、删除和更新 DOM 元素。
- **Ref 更新**：在突变阶段，React Fiber 会更新 `ref` 属性，将最新的 DOM 节点或组件实例赋值给 `ref`。
- **组件的生命周期方法**：调用组件的 `componentDidMount` 和 `componentDidUpdate` 生命周期方法。

### 3. 布局阶段（Layout Phase）

在布局阶段，React Fiber 会处理那些需要在 DOM 更新后立即执行的操作。主要包括：

- **useLayoutEffect**：执行布局副作用。在突变阶段完成后，所有 `useLayoutEffect` 的回调函数会被调用。这些副作用通常用于读取布局信息或者在 DOM 更新后执行同步操作。
- **生命周期方法**：调用 `componentDidMount` 和 `componentDidUpdate` 生命周期方法。这些方法在 DOM 变化已经应用后执行，可以确保 DOM 处于最新状态。

### 提交阶段的执行顺序

1. **开始提交阶段**：确定 Fiber 树中的哪些节点需要更新。
2. **Before Mutation Phase**：执行突变前阶段的生命周期方法和副作用清理。
3. **Mutation Phase**：应用 DOM 更新，处理 `ref` 和生命周期方法。
4. **Layout Phase**：执行布局副作用和相关的生命周期方法。
5. **结束提交阶段**：完成所有更新，准备进入下一个渲染循环。
