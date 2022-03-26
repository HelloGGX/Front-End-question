# React 基于可视化建模设计解决复杂应用的开发方式

在前端应用中，不管我们使用哪些框架，都会遇到和使用各种 state，这些 state 分布在应用的各个位置，最典型的是在 react 项目，state 被用到 dom 的判断和函数中。随着测试的不断深入，前端会修复很多: 因为边界情况未考虑到而引发的 bug。随着需求的增加，我们有非常多的场景需要满足，于是我们引入新的变量，用更好的命名方式，同时可能还需要引入新的条件判断，应用的复杂度也会随之增加，前端需要维护大量的 state，针对 state 的改变而引发的问题，我们也无法预知：我们的“修复”会触发哪些额外的问题，因此测试会一遍一遍的走全量测试，耗费大量的人力资源。

一个简单的 demo,看看我们的应用是如何从简单到复杂的：

我们有一个 switch 开关：

<img width="500" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-machine.png"/>

代码如下：

```jsx
const ScratchApp = () => {
  const [isActive, setIsActive] = useState(false);

  return (
    <div className="scratch">
      <div className="alarm">
        <div
          className="alarmToggle"
          data-active={isActive || undefined}
          onClick={() => setIsActive((v) => !v)}
        ></div>
      </div>
    </div>
  );
};
```

针对激活和关闭，我们通常的做法是给一个 boolean 类型的 state，false 表示关闭，true 表示激活，目前来说状况良好。
很显然，当我们需要与接口联调，需要根据接口返回的状态来触发关闭和激活的状态时，我们需要新增一个"pending"的状态，当处于 pending 状态时，switch 组件是处于 loading 状态的。那目前这个 isActive 显然无法满足目前的情况，于是我们一般会定义三个状态：

1. inactive
2. pending
3. active

代码如下：

```jsx
const ScratchApp = () => {
  // 1、inactive 2、pending 3、active
  const [status, setStatus] = useState("inactive");
  // 模拟接口请求
  const fetchWeb = () => {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve("active");
      }, 2000);
    });
  };
  useEffect(() => {
    // 当为请求状态时，请求接口
    if (status === "pending") {
      fetchWeb().then((res) => {
        setStatus(res);
      });
    }
  }, [status]);

  return (
    <div className="scratch">
      <div className="alarm">
        <div
          className="alarmToggle"
          data-active={status === "active" || undefined}
          style={{
            opacity: status === "pending" ? 0.5 : 1,
          }}
          onClick={() => setStatus("pending")}
        ></div>
      </div>
    </div>
  );
};
```

此时在激活的状态下点击按钮，按钮处于 pending 的状态，2 秒过后，变成了 active 状态，但是当我们再次点击按钮，按钮再次处于 pending，2 秒后，处于 active 状态，这显然不符合我们的预期，我们期望在 active 的状态下，点击按钮，切换到 inactive 的状态。状态顺序如下图：

<img width="500" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-machine1.png"/>

于是我们添加判断条件：

```jsx
const switchApp = () => {
  // 1、inactive 2、pending 3、active
  const [status, setStatus] = useState("inactive");
  // 模拟接口请求
  const fetchWeb = () => {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        resolve("active");
      }, 2000);
    });
  };
  useEffect(() => {
    if (status === "pending") {
      fetchWeb().then((res) => {
        setStatus(res);
      });
    }
  }, [status]);

  // 新增条件判断
  const handleClick = () => {
    // 根据状态判断切换动作
    if (status === "inactive") {
      setStatus("pending");
    } else if (status === "active") {
      setStatus("inactive");
    }
  };

  return (
    <div className="scratch">
      <div className="alarm">
        <div
          className="alarmToggle"
          data-active={status === "active" || undefined}
          style={{
            opacity: status === "pending" ? 0.5 : 1,
          }}
          onClick={handleClick}
        ></div>
      </div>
    </div>
  );
};
```

现在我们的逻辑开始稍微有点复杂了，我们的状态判断散布在 dom 中，函数的判断中，短短的三个状态在面对这样一个小小的需求和修复问题，就已经有 40 行的代码，显然在小应用中我们还能掌控，稍微大一点的应用，其实代码可维护性的问题就会渐渐显现出来。

我们稍微优化一下目前的逻辑：使用 useReducer 把状态判断都丢到 statusReducer 里面，外部我们只关心 dispatch 的派发事件，这样的好处在于，state 的逻辑我们集中在了一个地方，甚至我们可以在组件之间复用这个 reducer，逻辑变的更有条理，而不是分散开来。

```jsx
const statusReducer = (state, event) => {
  switch (state) {
    case "inactive":
      if (event.type === "TOGGLE") {
        return "pending";
      }
      return state;
    case "pending":
      if (event.type === "SUCCESS") {
        return "active";
      }
      return state;
    case "active":
      if (event.type === "TOGGLE") {
        return "inactive";
      }
      return state;
    default:
      return state;
  }
};
```

函数 statusReducer 与 app 逻辑隔离开来，现在 switch 组件的逻辑，似乎少了很多：

```jsx
import { statusReducer } from "./statusReducer";

const switchApp = () => {
  // 1、inactive 2、pending 3、active
  const [status, dispatch] = useReducer(statusReducer, initialState);

  useEffect(() => {
    if (status === "pending") {
      fetchWeb().then((res) => {
        dispatch({ type: "SUCCESS" });
      });
    }
  }, [status]);

  return (
    <div className="scratch">
      <div className="alarm">
        <div
          className="alarmToggle"
          data-active={status === "active" || undefined}
          style={{
            opacity: status === "pending" ? 0.5 : 1,
          }}
          onClick={() => dispatch({ type: "TOGGLE" })}
        ></div>
      </div>
    </div>
  );
};
```

以上的逻辑优化或者说重构，我们通过 switch case 这种方式来连接状态，其实并不能清楚的看到所有状态之间的关系，随着状态的增多，reducer 函数逻辑逐渐庞大，这种情况更加突出。

其实我们有更好的方式，去解决上面的问题。

### 有限状态机（finite state machine）

基于上面的流程图，我们可以用可视化的方式编写状态之间的联系，其中就不得不引入一个概念：**有限状态机**

有限状态机是一种数学计算模型，它描述了在任何给定时间只能处于一种状态的系统的行为。机器可以响应一些称为事件的输入，从一种状态转换到另一种状态。

您可以通过其状态列表、初始状态和触发每个状态转换的事件来定义有限状态机。

形式上，有限状态机有五个部分：

- 有限的 状态 数
- 有限的 事件 数
- 一个 初始状态
- 在给定当前状态和事件的情况下，确定下一个状态的 转换函数
- 一组（可能是空的）最终状态

> 状态 是指，由状态机建模的系统中某种有限的、定性 的“模式”或“状态”，并不描述与该系统相关的所有（可能是无限的）数据。

例如，水可以处于以下 4 种状态中的一种：冰、液体、气体或等离子体。然而，水的温度可以变化，所以其测量值是 定量的 和无限的。

### 状态图

有了状态机，我们需要一种方式对其建模表达，计算机科学家 David Harel 在他 1987 年的论文 状态图: [一个复杂系统的可视化表达 \(opens new window\)](https://www.sciencedirect.com/science/article/pii/0167642387900359/pdf)中提出了这种表达作为状态机的扩展。一些扩展包括：

- 守卫（Guard）转换
- 动作（Action）（进入、退出、转换）
- 扩展状态（上下文）
- 并行状态
- 分层（嵌套）状态
- 历史

基于以上的概念和社区的推动，2015 年 4 月 30 日，W3C 的语音浏览器工作组（Voice Browser Working Group）发布了状态图可扩展标记语言：控制抽象的状态机标注（State Chart XML / SCXML: State Machine Notation for Control Abstraction）的提案推荐标准（Proposed Recommendation）。该文档定义了状态机可扩展标记语言（SCXML），在 CCXML 和 Harel State Table 的基础上，提供通用的状态机模型和执行环境。

查看地址 [https://www.w3.org/TR/scxml/](https://www.w3.org/TR/scxml/)

### XState

基于以上的模型概念和 W3C 的 SCXML 规范, 社区孵化出了能在前端创建、解释和执行有限状态机和状态图的库：**[XState](https://xstate.js.org/docs/)**
他提供了一种[可视化的工具平台](https://stately.ai/viz#)，为我们创建状态图，你能实时的看到自己的逻辑走向和状态关系。

XState 的 API 就不一一赘述，[官网更详细](https://xstate.js.org/docs/)。

### 开发步骤

#### 第一步：

根据前文的 switch 组件，我们用流程图的方式表示 switch 组件内部的状态流转，一般的项目中产品也会给我们类似的业务员流程图：

<img width="600" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-machine2.png"/>

这里为了更好的理解逻辑关系，我们用了流程图，在实际项目中，我们可能会用到**业务流程时序图**
下面是 switch 组件内部的流程时序图：

<img width="600" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-machine6.png"/>

#### 第二步：

基于以上的流程图，我们使用可视化工具平台进行模型的开发：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-machine4.png"/>

可以看到我们的状态机基于 xstate 库提供的方法：createMachine 进行创建，以对象的形式描述了状态与状态之间的关系，以对象的形式的一个最大的好处是：完全序列化的 JSON，给了我们可以跨框架，甚至跨语言的复用状态机可能性。
并且我们的状态图与代码实时同步，能清楚地看到整个状态机的状态关系走向，在哪种状态下，具有哪些事件一目了然。

状态机代码如下：

```jsx
// 模拟接口请求
const fetchWeb = () => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve("active");
    }, 2000);
  });
};

const switchMachine = createMachine({
  id: "switch",
  initial: "inactive",
  states: {
    inactive: {
      on: {
        TOGGLE: {
          target: "pending",
        },
      },
    },
    pending: {
      invoke: {
        src: (ctx, event) => fetchWeb(),
        onDone: {
          target: "active",
        },
        onError: {
          target: "inactive",
        },
      },
      on: {
        TOGGLE: {
          target: "inactive",
        },
      },
    },
    active: {
      on: {
        TOGGLE: {
          target: "inactive",
        },
      },
    },
  },
});
```

基于我们刚才写的状态机，我们改造一下前面的 switch 组件:

```jsx
import React from "react";
import { switchMachine } from "./machine";
import { useMachine } from "@xstate/react";

export const switchApp = () => {
  // 1、inactive 2、pending 3、active
  const [state, send] = useMachine(switchMachine);

  return (
    <div className="scratch">
      <div className="alarm">
        <div
          className="alarmToggle"
          data-active={state.value === "active" || undefined}
          style={{
            opacity: state.value === "pending" ? 0.5 : 1,
          }}
          onClick={() => send("TOGGLE")}
        ></div>
      </div>
    </div>
  );
};
```

useMachine 接收一个状态机，返回一个状态 state 和事件发送器 send，我们只关心事件的发送，state 会根据状态机返回正确的状态。在解决问题的过程中我们能很容易的查到某个状态下，出了哪些问题。因为**任何给定时间状态机只能处于一种状态**。

### 面对大型的应用的建议

我们以上的 demo，只是一个玩具，在真正的大型应用中，其实情况比上面复杂得多，如果我们在面对业务逻辑写状态机的时候，可能会存在上百行的状态机，我们称为“状态爆炸”。
解决这样的情况，我们需要了解一个概念：**Actor**

演员模型是另一个非常古老的数学模型，它与状态机配合得很好。 它指出，一切都是一个“演员”，可以做三件事：

- 接收 消息
- 发送 消息到其他 Actor
- 用它收到的消息做一些事情( 行为)，比如：
  - 改变它的本地状态
  - 发送消息到其他演员
  - 产生 新的演员

因此从建模角度我们的一个状态机可以抽象为一个演员，这个演员可以**孵化出**跟多的演员，为其他的状态机提供能力。
xstate 提供了强大的 api，spawn 方法，

```js
const actor = spawn(machine);
```

拿 todoList 为例：每个新增的 todoItem 都可以视为一个演员实体，因为它具有选中和未选中的状态，也具有派发 delete，change 等事件的能力，每个演员实体各自独立。整个 todo 应用可以视为一个大状态机：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-machine3.png"/>

同时 xstate 也提供了 **sendParent**方法，让子状态机与父状态就之间通信。甚至提供了 withConfig 的选项，扩展状态机方法和状态。

因此面对“状态爆炸”,我们需要从建模角度去拆分流程图，并抽象出演员模型来复用状态机的能力。让我们的状态机更小，更容易管理。

### debug 调试状态机

为了方便的调试模拟状态机的走向，xstate 提供了调试库：[@xstate/inspect](https://xstate.js.org/docs/zh/packages/xstate-inspect/#%E5%AE%89%E8%A3%85)可以可视化的调试状态机：

```jsx
import React from "react";
import { switchMachine } from "./machine";
import { useMachine } from "@xstate/react";
import { inspect } from "@xstate/inspect";
import { interpret } from "xstate";

inspect({
  url: "https://statecharts.io/inspect",
  iframe: false,
});
export const ScratchApp = () => {
  // 1、inactive 2、pending 3、active
  const [state, send] = useMachine(switchMachine);

  const service = interpret(switchMachine, { devTools: true }).start();

  service.subscribe((state) => {
    console.log(state);
  });

  return (
    <div className="scratch">
      <div className="alarm">
        <div
          className="alarmToggle"
          data-active={state.value === "active" || undefined}
          style={{
            opacity: state.value === "pending" ? 0.5 : 1,
          }}
          onClick={() => send("TOGGLE")}
        ></div>
      </div>
    </div>
  );
};
```

当我们使用方法：

```js
inspect({
  url: "https://statecharts.io/inspect",
  iframe: false,
});
```

页面渲染时，会额外打开一个联调界面，来展示当前的状态，如下：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-machine5.png"/>

我们可以操作点击流程图中的事件，service.subscribe 可以实时的监听状态的改变，并执行回调：

```js
service.subscribe((state) => {});
```

xstate 也会实时为我们生成时序图。

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
