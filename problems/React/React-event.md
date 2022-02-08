# React 事件机制

本文从源码角度，深入剖析 react（17.0.2 版本）的事件机制，但并不详细介绍源码实现，只介绍源码思路， 因为这对我们理解 react 事件机制，解释 react 事件现象已经足够。

为了更好的理解 React 事件机制，我们需要回答下面三个问题:

1. React为什么有自己的事件系统？
2. 什么是事件合成？
3. 事件系统如何模拟冒泡和捕获阶段？

## 1、React 为什么有自己的事件系统？

首先，不同的浏览器对事件存在不同的兼容性，为了让自定义事件尽量不去依赖于浏览器，就需要创建一个兼容全浏览器的事件系统，以此抹平不同浏览器的差异，让用户能灵活且一致性地处理事件。

再比如： 有些鼠标事件（mouse ente）其实还未得到浏览器的广泛支持， react 在底层帮用户做了 pollyfill, 可以帮用户暂时的过渡。

其次，v17 之前 React 事件都是绑定在 document 上，v17 之后 React 把事件绑定在应用对应的容器上（根 root 上），将事件绑定在同一容器统一管理， **有利于微前端的实施和多版本 react 共存**，因为微前端都是一个 document 下面多个容器应用， 将事件绑定在容器这一层级，防止微应用的所有事件都直接绑定在原生的 document 元素上，造成一些不可控的情况。

由于不是绑定在真实的 DOM 上，所以 React 需要模拟一套事件流：事件捕获-> 事件源 -> 事件冒泡，也包括重写一下事件源对象 event。

以下面的简单例子为例: 在 button 和 parent 节点上分别绑定两个自定义点击事件（我们称为：合成事件），在 container 节点上绑定一个原生事件，并观察浏览器的事件监听器：

```typescript
function App() {
  const handleBubbleParentClick = () => {
    console.log("parent的冒泡合成事件");
  };
  const handleBubbleBtnClick = () => {
    console.log("button的冒泡合成事件");
  };
  return (
    <div className="container">
      <div className="parent" onClick={handleBubbleParentClick}>
        <button onClick={handleBubbleBtnClick}>按钮</button>
      </div>
    </div>
  );
}
ReactDOM.render(<App />, document.querySelector("#app"));
```

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react_event1.png"/>

我们可以看到在 button 按钮和 parent 节点的事件处理函数 handler 是一个空函数 noop，并没有绑定我们自定义的事件。

```typescript
function noop() {}
function trapClickOnNonInteractiveElement(node) {
  node.onclick = noop;
}
```

> 绑定空函数的原因是 React 为了解决 iPhone Safari 上，事件委托不适用于 click 事件的问题。 造成 Iphone 这样的问题的原因可能是：用户在 iphone 这样的触控设备上随机点击页面元素， 并通过捕获、目标、冒泡阶段传播点击事件的性能开销可能会很大,于是 iphone 工程师关闭了这样的功能。

而在`#app`的容器位置分别绑定了两个事件处理函数，他们分别处理捕获事件和冒泡事件，通过事件处理函数**dispathDiscreteEvent触发**。

由以上现象我们验证了 React 的事件绑定方式：

<span style="background-color: #fff199;color: black">
所有合成事件均注册到了root节点上并由统一的事件处理函数处理，所有原生事件均注册到各自目标元素本身上。</span>

结论：

1. 所有合成事件均注册到了根 root 上并由统一的事件处理函数处理（方便多版本 react 共存）
2. 由于在真实 DOM 上不用绑定事件，从而节省了内存
3. 所有原生事件均注册到各自目标元素本身上
4. react 实现自己的事件系统抹平了浏览器之间的差异，简化了事件逻辑

## 2、初始化事件系统

在搞清楚事件合成之前，我们需要知道 React 在初始化事件系统时做了哪些工作。
通过源码可知， 在初始化时, react-dom 上下文首先通过下面几个方法把 react 事件名称对应的原生事件依赖存入变量 registrationNameDependencies 中:

```typescript
SimpleEventPlugin.registerEvents(); //处理基础事件，比如：cancel, click, input...
EnterLeaveEventPlugin.registerEvents(); //处理事件：onMouseEnter, onMouseLeave, onPointerEnter, onPointerLeave
ChangeEventPlugin.registerEvents(); //处理事件：onChange
SelectEventPlugin.registerEvents(); //处理事件：onSelect
BeforeInputEventPlugin.registerEvents(); //  处理事件：onBeforeInput, onCompositionEnd, onCompositionStart, onCompositionUpdate
```

在上面的 SimpleEventPlugin.registerEvents 函数中，react 对不同类别的原生事件进行了优先级定义，最终以 Map 的形式存在变量 eventPriorities（后面事件注册会用到）中：

```typescript
export const DiscreteEvent = 0; // 离散事件，cancel、click、mousedown 这类单点触发不持续的事件，优先级最低
export const UserBlockingEvent = 1; // 用户阻塞事件，drag、mousemove、wheel 这类持续触发的事件，优先级相对较高
export const ContinuousEvent = 2; // 连续事件，load、error、waiting 这类大多与媒体相关的事件为主的事件需要及时响应，所以优先级最高

const eventPriorities = new Map();
// 0: {"cancel" => 0}
// 1: {"click" => 0}
// 2: {"close" => 0}
// 3: {"mousedown" => 0}
// 4: {"copy" => 0}
// ...
```

而对象 registrationNameDependencies 打印出来如下所示：

```typescript
}
onChange:  ['change', 'click', 'focusin', 'focusout', 'input', 'keydown', 'keyup', 'selectionchange']
onChangeCapture:  ['change', 'click', 'focusin', 'focusout', 'input', 'keydown', 'keyup', 'selectionchange']
onClick: ['click']
onClickCapture: ['click']
onMouseDownCapture: ['mousedown']
onMouseEnter:  ['mouseout', 'mouseover']
onMouseLeave:  ['mouseout', 'mouseover']
onSelect:  ['focusout', 'contextmenu', 'dragend', 'focusin', 'keydown', 'keyup', 'mousedown', 'mouseup', 'selectionchange']
onSelectCapture: ['focusout', 'contextmenu', 'dragend', 'focusin', 'keydown', 'keyup', 'mousedown', 'mouseup', 'selectionchange']
...
}
```

可以看到，我们平常使用的 react 事件其实对应了一个或者多个原生事件，我们称这样的 React 事件为**合成事件**。

除此之外，还有些重要的变量也简单介绍一下：

| 变量               | 数据类型 | 意义                                                   |
| ------------------ | -------- | ------------------------------------------------------ |
| allNativeEvents    | Set      | 所有有意义的原生事件名称集合                           |
| nonDelegatedEvents | Set      | 不需要在冒泡阶段进行事件代理（委托）的原生事件名称集合 |

## 3、注册事件代理

在我们调用 ReactDOM.render 在提供的 container 里初始化渲染根组件时：

```typescript
ReactDOM.render(<App />, document.querySelector("#app"));
```

实际会依次调用下面函数:

legacyRenderSubtreeIntoContainer => legacyCreateRootFromDOMContainer => createLegacyRoot => new ReactDOMBlockingRoot => createRootImpl => listenToAllSupportedEvents

### 3.1 listenToAllSupportedEvents

**而 listenToAllSupportedEvents 会遍历 allNativeEvents 在所有可以监听的原生事件上添加监听事件：**

```typescript
function listenToAllSupportedEvents(rootContainerElement) {
  allNativeEvents.forEach(function (domEventName) {
    // 需要在冒泡阶段添加事件代理的原生事件
    if (!nonDelegatedEvents.has(domEventName)) {
      listenToNativeEvent(domEventName, false, rootContainerElement, null);
    } //不需要在冒泡阶段添加事件代理的原生事件
    listenToNativeEvent(domEventName, true, rootContainerElement, null);
  });
}
```

### 3.2 addTrappedEventListener

而 listenToNativeEvent 会调用 addTrappedEventListener 函数，根据前文的变量 eventPriorities 获取原生事件对应的优先级， 根据不同的优先级提供不同的监听函数，并绑定到 target（root 结点）上。

```typescript
function addTrappedEventListener(
    targetContainer,
    domEventName,
    eventSystemFlags,
    isCapturePhaseListener,
    isDeferredListenerForLegacyFBSupport
) {
  // 根据domEventName创建带有优先级的事件监听器
  var listener = createEventListenerWrapperWithPriority(targetContainer, domEventName, eventSystemFlags);
  targetContainer =  targetContainer;   
  var unsubscribeListener;
  // 不需要在冒泡阶段添加事件代理的原生事件，都在捕获阶段添加事件代理
  if (isCapturePhaseListener) {      
      unsubscribeListener = addEventCaptureListener(targetContainer, domEventName, listener);     
  }
  else {
    // 需要在冒泡阶段添加事件代理的原生事件，都在冒泡阶段添加事件代理
      unsubscribeListener = addEventBubbleListener(targetContainer, domEventName, listener);   
 } 
```

### 3.3 createEventListenerWrapperWithPriority

该函数会根据每个原生事件对应的优先级，返回不同的事件监听器， 它们都将作为 addEventListener 函数的 listener 绑定到 root 节点上。

```typescript
function createEventListenerWrapperWithPriority(
  targetContainer,
  domEventName,
  eventSystemFlags
) {
  // 根据前文的eventPriorities变量获取原生事件的优先级
  var eventPriority = getEventPriorityForPluginSystem(domEventName);
  var listenerWrapper;
  // 根据不同的优先级提供不同的监听函数listenerWrapper
  switch (
    eventPriority // 离散事件，cancel、click、mousedown 这类单点触发不持续的事件，优先级最低
  ) {
    case DiscreteEvent:
      listenerWrapper = dispatchDiscreteEvent;
      break;
    // 用户阻塞事件，drag、mousemove、wheel 这类持续触发的事件，优先级相对较高
    case UserBlockingEvent:
      listenerWrapper = dispatchUserBlockingUpdate;
      break;
    // 连续事件，load、error、waiting 这类大多与媒体相关的事件为主的事件需要及时响应，优先级最高
    case ContinuousEvent:
    default:
      listenerWrapper = dispatchEvent;
      break;
  }
  return listenerWrapper.bind(
    null,
    domEventName,
    eventSystemFlags,
    targetContainer
  );
}
```

### 3.4 addEventCaptureListener/addEventBubbleListener
这里真正调用了原生的addEventListener方法
```typescript
// 调用原生的监听函数addEventListener
function addEventBubbleListener(target, eventType, listener) {
  target.addEventListener(eventType, listener, false);
  return listener;
}
function addEventCaptureListener(target, eventType, listener) {
  target.addEventListener(eventType, listener, true);
  return listener;
}
```

最终你会看到在 root 节点上每个原生事件都绑定了其对应优先级所对应的监听器。

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react_event2.png"/>

> UI 产生交互的根本原因是各种事件，这也就意味着事件与更新有着直接关系。不同事件产生的更新，它们的优先级是有差异的，所以更新优先级的根源在于事件的优先级。一个更新的产生可直接导致 react 生成一个更新任务，最终这个任务被 Scheduler 调度。

所以 react 将事件划分等级的最终目的是决定调度任务的轻重缓急。关于 React 的优先级任务调度，由于篇幅过长这里就不过多赘述，后面将会有专门的篇幅来深入优先级问题。

总结：

1. 初始化跟组件时，react 会为所有可以监听的原生事件上添加监听器
2. React 会根据不同的优先级提供不同的监听器分别是：
    - 离散事件（DiscreteEvent）：click、keydown、focusin 等，这些事件的触发不是连续的，优先级为 0。
    - 用户阻塞事件（UserBlockingEvent）：drag、scroll、mouseover 等，特点是连续触发，阻塞渲染，优先级为 1。
    - 连续事件（ContinuousEvent）：canplay、error、audio 标签的 timeupdate 和 canplay，优先级最高，为 2。
3. React 将事件划分优先级的目的是决定调度任务的轻重缓急，提供更好的用户体验。

# 4. 触发事件流程

首先我们的 demo 如下：

```typescript
function App() {
  const handleBubbleParentClick = () => {
    console.log("parent的冒泡合成事件");
  };
  const handleBubbleBtnClick = () => {
    console.log("button的冒泡合成事件");
  };
  React.useEffect(() => {
    document.querySelector(".container").addEventListener("click", () => {
      console.log("container的原生事件");
    });
    return () => {
      document.querySelector(".container").removeEventListener("click");
    };
  }, []);
  return (
    <div className="container">
      <div className="parent" onClick={handleBubbleParentClick}>
        <button onClick={handleBubbleBtnClick}>按钮</button>     
      </div>
    </div>
  );
}

ReactDOM.render(<App />, document.querySelector("#app"));
```

当我们点击按钮触发事件监听时，根据浏览器生成的性能分析报告，我们能清楚的看到调用栈：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react_event3.png"/>

我们可以看到，首先触发了函数 dispatchUserBlockingUpdate,该函数是处理类似 mousemove 这样的持续触发的事件，那是因为前文的**函数 createEventListenerWrapperWithPriority 在事件注册阶段为每个原生事件创建了带有优先级的事件监听**。

当我们的鼠划过 id 为 app 的容器时，就会触发原生事件 pointermove 和 mousemove，它们对应的优先级为 1， 从而触发监听器 dispatchUserBlockingUpdate。

同理，当我们点击按钮的时候，因为事件 click 为单点类触发不持续的事件，对应的优先级为 0，会触发函数 dispatchDiscreteEvent。

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react_event4.png"/>

最终，不管是 dispatchUserBlockingUpdate 还是 dispatchDiscreteEvent，都会执行 dispatchEvent。

实际上 dispathEvent 中最核心的内容就是调用 dispatchEventsForPlugins，因为正是这个函数触发了**事件收集、事件执行**。

```typescript
function dispatchEventsForPlugins(
  domEventName,
  eventSystemFlags,
  nativeEvent,
  targetInst,
  targetContainer
) {
  // event.target
  const nativeEventTarget = getEventTarget(nativeEvent);
  // 事件队列, 收集的事件都会存储在这里
  const dispatchQueue = [];
  // 找到所有绑定了相应事件的节点，生成合成事件实例，并放入dispatchQueue队列中（收集事件）
  extractEvents(
    dispatchQueue,
    domEventName,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags
  );
  // 执行dispatchQueue队列中的绑定的方法
  processDispatchQueue(dispatchQueue, eventSystemFlags);
}
```

下面我们重点讲事件的收集和事件的执行。

## 4.1 事件的收集

事件收集函数 extractEvents 如下：

```typescript
function extractEvents(
  dispatchQueue: DispatchQueue,
  domEventName: DOMEventName,
  targetInst: null | Fiber,
  nativeEvent: AnyNativeEvent,
  nativeEventTarget: null | EventTarget,
  eventSystemFlags: EventSystemFlags,
  targetContainer: EventTarget
) {
  //收集主要事件 比如：click,keydown,keyup
  SimpleEventPlugin.extractEvents(
    dispatchQueue,
    domEventName,
    targetInst,
    nativeEvent,
    nativeEventTarget,
    eventSystemFlags,
    targetContainer
  );
  const shouldProcessPolyfillPlugins =
    (eventSystemFlags & SHOULD_NOT_PROCESS_POLYFILL_EVENT_PLUGINS) === 0;
  if (shouldProcessPolyfillPlugins) {
    //收集mouseover、pointerover、mouseout、pointerout事件
    EnterLeaveEventPlugin.extractEvents(
      dispatchQueue,
      domEventName,
      targetInst,
      nativeEvent,
      nativeEventTarget,
      eventSystemFlags,
      targetContainer
    ); // 收集onChange事件
    ChangeEventPlugin.extractEvents(
      dispatchQueue,
      domEventName,
      targetInst,
      nativeEvent,
      nativeEventTarget,
      eventSystemFlags,
      targetContainer
    ); // 收集onSelect事件，处理select事件的兼容性问题
    SelectEventPlugin.extractEvents(
      dispatchQueue,
      domEventName,
      targetInst,
      nativeEvent,
      nativeEventTarget,
      eventSystemFlags,
      targetContainer
    ); // 收集onBeforeInput事件，处理beforeInput和composition事件的兼容性问题
    BeforeInputEventPlugin.extractEvents(
      dispatchQueue,
      domEventName,
      targetInst,
      nativeEvent,
      nativeEventTarget,
      eventSystemFlags,
      targetContainer
    );
  }
}
```

我们可以看到 react 把事件收集以插件的形式，针对不同的事件类型由不同的插件处理并收集。但是他们内部逻辑都做了同一件事：**收集事件处理函数和创建合成事件实例存入 dispatchQueue 中**。

以 packages/react-dom/src/events/plugins/ 路径下的 SimpleEventPlugin.js 为例：

```typescript
function extractEvents(
  dispatchQueue: DispatchQueue,
  domEventName: DOMEventName,
  targetInst: null | Fiber,
  nativeEvent: AnyNativeEvent,
  nativeEventTarget: null | EventTarget,
  eventSystemFlags: EventSystemFlags,
  targetContainer: EventTarget,
): void {
  // 根据原生事件名称获取合成事件名称
  // 效果: onClick = topLevelEventsToReactNames.get('click')
  const reactName = topLevelEventsToReactNames.get(domEventName);
  if (reactName === undefined) {
    return;
  }
  let SyntheticEventCtor = SyntheticEvent;
  let reactEventType: string = domEventName;
  switch (domEventName) {
    //按照原生事件名称来获取对应的合成事件构造函数
    ...
    case 'click':
      if (nativeEvent.button === 2) {
        return;
      }
    case 'auxclick':
    case 'dblclick':
    case 'mousedown':
    case 'mousemove':
    case 'mouseup':
    case 'mouseout':
    case 'mouseover':
    case 'contextmenu':
      SyntheticEventCtor = SyntheticMouseEvent;
      break;
    ...
    default:
      break;
  }
// 是否是捕获阶段
  const inCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0;
  const accumulateTargetOnly =!inCapturePhase && domEventName === 'scroll';
 //从事件源target到root节过程中收集与事件源类型相同的事件回调
  const listeners = accumulateSinglePhaseListeners(
      targetInst,
      reactName,
      nativeEvent.type,
      inCapturePhase,
      accumulateTargetOnly,
      nativeEvent,
    );
    if (listeners.length > 0) {
      // 生成合成事件的 Event 对象
      const event = new SyntheticEventCtor(
        reactName,
        reactEventType,
        null,
        nativeEvent,
        nativeEventTarget,
      );
      //放入队列
      dispatchQueue.push({event, listeners});
    }
}
```

以上最核心的逻辑分别为：

- 收集事件函数 accumulateSinglePhaseListeners
- 生成合成事件对象 new SyntheticEventCtor

### 1、事件回调的收集

函数 accumulateSinglePhaseListeners 通过 while 的方式从 target 节点遍历到 root 节点,收集 Fiber 节点树中对应的事件：

```typescript
function accumulateSinglePhaseListeners(
  targetFiber,
  reactName,
  nativeEventType,
  inCapturePhase,
  accumulateTargetOnly
) {
  // 捕获阶段合成事件名称
  var captureName = reactName !== null ? reactName + "Capture" : null;
  // 最终合成事件名称
  var reactEventName = inCapturePhase ? captureName : reactName;
  var listeners = [];
  var instance = targetFiber;
  var lastHostComponent = null;
  // 收集从事件源Fiber节点到root节点的事件
  while (instance !== null) {
    var _instance2 = instance,
      stateNode = _instance2.stateNode,
      tag = _instance2.tag;
    // 如果时有效节点则获取其事件
    if (tag === HostComponent && stateNode !== null) {
      lastHostComponent = stateNode;
      if (reactEventName !== null) {
        // 获取存储在 Fiber 节点上 Props 里的对应事件（如果存在）
        var listener = getListener(instance, reactEventName);
        if (listener != null) {
          listeners.push(
            // 简单返回一个 {instance, listener, lastHostComponent} 对象
            createDispatchListener(instance, listener, lastHostComponent)
          );
        }
      }
    } // scroll 不会冒泡，获取一次就结束了
    if (accumulateTargetOnly) {
      break;
    }
    // 其父级 Fiber 节点，向上递归
    instance = instance.return;
  }
  return listeners;
}
```

下面是 getListener 核心逻辑：沿途收集 Fiber 节点上的 props 对象中与事件源相同的类型事件回调。

```typescript
var props = getFiberCurrentPropsFromNode(stateNode);
if (props === null) {
  // Work in progress.
  return null;
}
var listener = props[registrationName];
return listener;
```

然后通过 createDispathListener 函数返回一个{instance, listener, lastHostComponent} 对象，存入 listeners 数组中：

```typescript
function createDispatchListener(instance, listener, currentTarget) {
  return {
    instance: instance, // 事件源的Fiber对象
    listener: listener, //事件回调
    currentTarget: currentTarget, //事件源的原生dom节点
  };
}
```

于是根据前文的 demo,当我们点击按钮的时候，打印 listeners 数组是这样的:

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react_event5.png"/>

可以看到从 button 到 root 节点，沿途收集了两个 click 事件，正是我们自定义的两个函数 ：handleBubbleBtnClick、handleBubbleParentClick。

但当我们的在 button 和 parent 节点上分别绑定 onClickCapture 捕获事件，如下：

```html
<div className="parent" onClick={handleBubbleParentClick} onClickCapture={handleCaptureParentClick}>
  <button onClick={handleBubbleBtnClick} onClickCapture={handleCaptureBtnClick}>
      按钮
  </button>
</div>
```

点击按钮打印listeners,我们会发现无论事件是在冒泡阶段执行，还是捕获阶段执行，都以同样的顺序 push 到 dispatchQueue 的 listeners 中。

思考一下： 那 react 是如何模拟冒泡和捕获这两种不同的执行顺序呢？

### 2、生成合成事件

从 extractEvents 这一函数的代码中可以看出，react 会根据不同的事件， 选择不同的构造函数，创建一个构造函数 event 实例。
这些构造函数都是由函数**createSyntheticEvent**创建，该函数的主要目的是根据传入的不同事件接口，实现了[浏览器的 3 级 DOM 事件 API](https://www.w3.org/TR/DOM-Level-3-Events/)，规避了浏览器的一些兼容性问题。

## 4.2 事件的执行

紧接着前文留下的思考问题：react 是如何模拟冒泡和捕获这两种不同的执行顺序呢？

我们在函数 processDispatchQueue 里打印数组 dispatchQueue,然后点击按钮查看打印信息：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react_event6.png"/>

发现 processDispatchQueue 执行了两次, 现象为：先收集捕获事件，然后执行捕获回调完之后，再收集冒泡事件，再执行冒泡回调。

<span style="background-color: #fff199;color: black">从注册事件代理阶段可知，react 在创建 root 阶段调用 listenToAllSupportedEvents 函数， 将所有可以监听的原生事件添加监听代理绑定到 root 节点上，其中就包含了捕获和冒泡的监听代理。</span>

<span style="background-color: #fff199;color: black">在浏览器的环境中，若父子元素绑定了相同类型的事件，除非手动干预，那么这些事件都会按照冒泡或者捕获的顺序触发。</span>

React 也是如此，点击（click）按钮，事件捕获最先发生，触发 root 上绑定的 dispatchDiscreteEvent 函数,并触发事件的收集和执行：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react_event7.png"/>

依然是从按钮开始遍历到 root 节点收集 onClickCapture 事件, 存储到 listeners 数组中，这里的逻辑不管是捕获还是冒泡都是一样的，然后执行函数 processDispatchQueue：

```typescript
function processDispatchQueue(dispatchQueue, eventSystemFlags) {
  // 通过 eventSystemFlags 判断当前事件阶段是捕获还是冒泡
  const inCapturePhase = (eventSystemFlags & IS_CAPTURE_PHASE) !== 0;
  for (let i = 0; i < dispatchQueue.length; i++) {
    // 从dispatchQueue中取出事件对象和事件监听数组
    const { event, listeners } = dispatchQueue[i];
    //将事件监听交由processDispatchQueueItemsInOrder去触发，同时传入事件对象供事件监听使用
    processDispatchQueueItemsInOrder(event, listeners, inCapturePhase);
  }
  rethrowCaughtError();
}
```

重点看 processDispatchQueueItemsInOrder 这个函数，它模拟了事件执行的捕获和冒泡流程,如果是捕获阶段的事件则倒序调用收集的事件，反之为正序调用:

```typescript
function processDispatchQueueItemsInOrder(
  event: ReactSyntheticEvent,
  dispatchListeners: Array<DispatchListener>,
  inCapturePhase: boolean
): void {
  let previousInstance;
  if (inCapturePhase) {
    // 事件捕获倒序循环
    for (let i = dispatchListeners.length - 1; i >= 0; i--) {
      const { instance, currentTarget, listener } = dispatchListeners[i];
      // 如果事件对象阻止了冒泡，则跳出循环
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return;
      }
      // 执行事件，传入event对象，和currentTarget
      executeDispatch(event, listener, currentTarget);
      previousInstance = instance;
    }
  } else {
    // 事件冒泡正序循环
    for (let i = 0; i < dispatchListeners.length; i++) {
      const { instance, currentTarget, listener } = dispatchListeners[i];
      // 如果事件对象阻止了冒泡，则跳出循环
      if (instance !== previousInstance && event.isPropagationStopped()) {
        return;
      }
      // 执行事件，传入event对象，和currentTarget
      executeDispatch(event, listener, currentTarget);
      previousInstance = instance;
    }
  }
}
```

捕获或者冒泡阶段过程中，如果浏览器中途发现有原生事件绑定，那么执行对应事件阶段的原生事件的回调， 因为原生事件绑定并不会触发事件注册，更不会触发事件收集，属于浏览器的原生行为。

参考如下代码，在 container 节点绑定了 click 原生事件：

```jsx
function App() {
  const handleBubbleParentClick = () => {
    console.log("parent的冒泡合成事件");
  };
  const handleBubbleBtnClick = () => {
    console.log("button的冒泡合成事件");
  };
  const handleCaptureParentClick = () => {
    console.log("parent的捕获合成事件");
  };
  const handleCaptureBtnClick = () => {
    console.log("button的捕获合成事件");
  };
  React.useEffect(() => {
    document.querySelector(".container").addEventListener("click", () => {
      console.log("container的原生事件");
    });
    return () => {
      document.querySelector(".container").removeEventListener("click");
    };
  }, []);
  return (
    <div className="container">
      <div
        className="parent"
        onClick={handleBubbleParentClick}
        onClickCapture={handleCaptureParentClick}
      >
        <button
          onClick={handleBubbleBtnClick}
          onClickCapture={handleCaptureBtnClick}
        >
          按钮
        </button>
      </div>
    </div>
  );
}
```
点击按钮打印如下:

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react_event8.png"/>

我们根据浏览器的事件流行为来解释以上现象：
捕获阶段=> 事件源阶段 => 冒泡阶段

点击按钮时：
1. 从root节点开始就已经触发了负责捕获事件的dispatchDiscreteEvent监听代理：从事件源节点到root节点收集onClick事件，最后**倒序**执行。（打印捕获回调）
2. 事件捕获阶段从root到事件源的过程中，如果遇到绑定的**原生捕获事件**，执行原生事件回调，一直到事件源阶段。
3. 从事件源到root节点的冒泡阶段，途中如果遇到绑定的**原生冒泡事件**，正常执行浏览器的原生行为。（打印原生事件回调）
4. 当事件冒泡到root节点，触发绑定在root上负责冒泡的dispatchDiscreteEvent监听代理：从事件源节点到root节点沿途收集onClick事件，最后**顺序**执行（打印冒泡事件回调）

因此我们可以发现，原生的禁止冒泡方法是可以阻止触发root节点的冒泡监听，但反过来，react的isPropagationStopped api只能阻止react事件的执行，不能阻止原生方法的冒泡执行，因为react并不能识别原生事件的绑定，不会进行事件的注册，更别提收集和执行了。

总结：

1. React针对不同的事件，调用不同的插件处理事件收集
2. React收集事件的方式是从事件源开始向上遍历，直到root节点为止，沿途收集Fiber点的Props属性中与事件源类型相同的回调
3. createSyntheticEvent函数根据传入的不同事件接口，实现了浏览器的3级DOM事件API，规避了浏览器的一些兼容性问题，并返回合成事件对象
4. React遍历事件队列dispatchQueue数组，并遍历其中的listeners，捕获阶段，倒序遍历执行回调，冒泡阶段，正序遍历执行回调合成事件和原生事件存在的情况下，执行顺序为 捕获合成事件->原生事件->冒泡合成事件
5. 原生的禁止冒泡方法可以阻止react冒泡，react的isPropagationStopped无法阻止原生事件



***

# 关于作者

大家好，我是程序员高翔的猫，MonkeyDesign用户体验部前端研发。

**加我的微信，备注：「个人简单介绍」+「组队讨论前端」**， 拉你进群，每周两道前端讨论分析，题目是由浅入深的，把复杂的东西讲简单的，把枯燥的知识讲有趣的，实用的，深入的，系统的前端知识。


<a name="微信"></a>
<img width="400" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/weixin.jpg"/>

# 我的公众号

更多精彩文章持续更新，微信搜索：「高翔的猫」第一时间围观


<a name="公众号"></a>

<img src="https://mp.weixin.qq.com/mp/qrcode?scene=10000004&size=102&__biz=MzkxNjMxNDU0MQ==&mid=2247483692&idx=1&sn=2d2baccebfd92fbf6d0506d3c75b3ade&send_time=" data-img="1" width="200" height="200">