# VUE 模板解析原理

在 vue3 的官方文档里面有这样一段话：

> Vue 使用一种基于 HTML 的模板语法，使我们能够声明式地将其组件实例的数据绑定到呈现的 DOM 上。所有的 Vue 模板都是语法上合法的 HTML，可以被符合规范的浏览器和 HTML 解析器解析。
>
> 在底层机制中，Vue 会将模板编译成高度优化的 JavaScript 代码。结合响应式系统，当应用状态变更时，Vue 能够智能地推导出需要重新渲染的组件的最少数量，并应用最少的 DOM 操作。
> 这段话高度概括了 vue 的编译原理：**将模板语法编译为 JavaScript 代码**
> 我们平时写的 vue 代码如下所示：

```html
<script>
  export default {
    data() {
      return {
        counter: 0,
      };
    },
    methods: {
      handler() {
        this.counter++;
      },
    },
  };
</script>
<template>
   
  <div @click="handler">ggx点击 {{ counter }}</div>
</template>
```

其中`<template>`标签里的内容就是模板内容，编译器会把模板内容编译成渲染函数并添加到`<script>`标签块的组件对象上，所以最终在浏览器里运行的代码就是：

```html
<script>
  export default {
    data() {
      return {
        counter: 0,
      };
    },
    methods: {
      handler() {
        this.counter++;
      },
    },
    render() {
      return h("div", { onClick: handler }, "ggx点击 " + counter, 1 /* TEXT */);
    },
  };
</script>
```

而这个 h 函数在 vue 中用来创建 vnode,也就是虚拟 DOM, 它是一个纯 JavaScipt 对象用来描述我们创建的实际元素所需的所有信息，而我们写的 HTML 模板一般会存在嵌套和并列关系，于是这些子节点会形成一个树型结构的虚拟 DOM 树。上面的 h 函数会返回大致如下虚拟 DOM:

```
{
  type: "div",
  props: { onClick: ...},
  children: "ggx点击0",
}
```

vue 会启动一个运行时渲染器遍历整个虚拟 DOM 树，并据此构建真实的 DOM 树。这个过程被称为挂载 (mount)。
以上就是 vue 从模板、渲染函数、真实 DOM 的整个过程，下面我们来详细剖析 vue 解析模板语法的过程。

## 1. 编译简介

编译器其实只是一段程序，我们将所写的`template`称为**源代码**， 将生成的 render 渲染函数称为**目标代码**。

编译器将源代码翻译为目标代码的过程叫做**编译**。

![](https://files.mdnice.com/user/23305/306d1660-9b23-49f8-aaab-ed0a8408fcf5.png)

可以看到 vue.js 模板编译器的目标代码就是渲染函数 render。vue 模板编译器首先对模板进行词法分析和语法分析，得到一个抽象语法树 AST，它是一个用来描述模板内容具有层级结构的对象。接着将 AST 转换成 JavaScript AST，最后根据 JavaScript AST 通过字符串拼接的方式生成渲染函数 render。

以上工作流程，它主要有三个部分组成：

1. 用来将模板字符串解析为模板 AST 的解析器（parser）
2. 用来将模板 AST 转换为 JavaScript AST 的转换器 （transformer）
3. 用来根据 JavaScript AST 生成渲染函数代码的生成器 （generator）

## 2. 解析器-parser

首先，vue 针对模板字符串的解析是根据 WHATWG 关于 HTML 的解析规范来的，其中定义了完整的错误处理和状态机的状态迁移流程，还提及了一些特殊的状态，例如 DATA、CDATA、RCDATA、RAWTEXT 等。而 parse 函数正是对 WHATWG 状态机规范的实现，下面的 parse 函数是 vue.js 解析器核心原理的简单实现：

```js
// 解析器函数，接收模板作为参数
function parse(str) {
  // 定义上下文对象
  const context = {
    // suorce 是模板内容，用于在解析的过程中进行消费
    source: str,
    // 解析器当前处于文本模式，初始化模式为DATA
    mode: TextModes.DATA,
    // 消费字符串的工具函数
    advanceBy(num) {
      context.source = context.source.slice(num);
    },
    // 消费空格的工具函数
    advanceSpaces() {
      const match = /^[\t\r\n\f ]+/.exec(context.source);
      if (match) {
        context.advanceBy(match[0].length);
      }
    },
  };
  // 调用parseChildren函数开始进行解析，它返回解析后得到的子节点
  // parseChildren函数接收两个参数
  // 第一个参数是上下文对象context
  // 第二个参数是由父代节点构成的节点栈，初始时栈为空
  const nodes = parseChildren(context, []);
  // 解析器返回Root根节点
  return {
    type: "Root",
    // 使用nodes作为根节点的children
    children: nodes,
  };
}
```

> 后文中我们所说的"消费"指：通过上下文中的工具函数 advanceBy， advanceSpaces 来截取掉字符串或者空格的部分内容。

我们可以看到**parseChildren**函数是整个解析器的核心。而它本质上是一个状态机，该状态机有多少种状态取决于子节点的类型数量，我们所写的模板中，元素的子节点可以是以下几种：

1. 标签节点，例如<div>
2. 文本插值节点， 例如 {{val}}
3. 普通文本节点， 例如：text
4. 注释节点，例如：<!---->
5. CDATA 节点，例如 <![CDATA[xxx]]>

在标准的 HTML 中，节点的类型将会更多，例如 DOCTYPE 节点等，我们目前只考虑比较典型的节点，以下是节点状态的迁移过程：

![](https://files.mdnice.com/user/23305/349a2f12-ab2a-4e9b-8062-2cff3f73f49e.png)

在 parseChildren 函数中会开启一个 while 循环，根据节点的类型不同，走不同的状态处理函数，每次 while 循环都会解析一个或多个节点，这些节点会被添加到 nodes 数组中，并最终返回子节点组成的数组。

以下是 parseChildren 核心原理的基本实现：

```js
function parseChildren(context, ancestors) {
  let nodes = [];
  const { mode } = context;
  while (!isEnd(context, ancestors)) {
    let node;
    if (mode === TextModes.DATA || mode === TextModes.RCDATA) {
      if (mode === TextModes.DATA && context.source[0] === "<") {
        if (context.source[1] === "!") {
          if (context.source.startsWith("<!--")) {
            // 注释
            node = parseComment(context);
          } else if (context.source.startsWith("<![CDATA[")) {
            // CDATA
            node = parseCDATA(context, ancestors);
          }
        } else if (context.source[1] === "/") {
          // 结束标签
        } else if (/[a-z]/i.test(context.source[1])) {
          // 标签
          node = parseElement(context, ancestors);
        }
      } else if (context.source.startsWith("{{")) {
        // 解析插值
        node = parseInterpolation(context);
      }
    }
    if (!node) {
      node = parseText(context);
    }
    nodes.push(node);
  }
  return nodes;
}
```

我们可以看到，parseChildren 本质上是一个状态机。而 vue 根据 WHATWG 规范给出的解析 HTML 的工作流程，会判断当前解析处于什么模式下，比如：在默认的 DATA 模式下，解析器在遇到字符`<`时，会切换到 **标签开始状态**，在遇到 `<script>` 标签时就会进入 RAWTEXT 模式，这时会把`<script>`标签内的内容全部作为普通文本处理。

我们再看看 parseChildren 和 parseElement 函数之间的关系，以下是 parseElement 核心原理的简单实现：

```js
function parseElement(context, ancestors) {
  //解析开始标签
  const element = parseTag(context);
  if (element.isSelfClosing) return element;
  ancestors.push(element);
  if (element.tag === "textarea" || element.tag === "title") {
    context.mode = TextModes.RCDATA;
  } else if (/style|xmp|iframe|noembed|noframes|noscript/.test(element.tag)) {
    context.mode = TextModes.RAWTEXT;
  } else {
    context.mode = TextModes.DATA;
  }
  // 解析子节点
  element.children = parseChildren(context, ancestors);
  ancestors.pop();
  if (context.source.startsWith(`</${element.tag}`)) {
    // 解析结束标签
    parseTag(context, "end");
  } else {
    console.error(`${element.tag} 标签缺少闭合标签`);
  }
  return element;
}
```

我们可以看到 parseElement 函数会做三件事：**解析开始标签，解析子节点，解析结束标签。** 其中，解析子节点又会调用 parseChildren 函数，进而开启一个新的状态机。

举个例子：

```js
const template = `<div>
    <!-- 我是注释 -->
    <p> 
        Text1<span>ggx</span>
    </p>
    <img src="xxx" />
</div>
`;
```

**需要强调的是，在解析模板是，我们不能忽略空白符。这些空白符包括：换行符（\n）、回车符（\r）、空格（' '）、制表符（\t）以及换页符（\f）**。

解析步骤为：

1.  将整个模板传入 parse 函数，内部会执行 parseChildren，开启一个新的状态机，我们称为**状态机 1**
2.  解析器遇到第一个字符为<. 并且第二个字符能够匹配正则表达式/a-z/i, 所以解析器会进入标签节点状态，调用 parseElement 函数。parseElement 函数会解析开始标签，template 的开头`<div>`标签会执行 parseTag 函数被消费掉，并将解析出的 div 开始节点存进 ancestors 数组中([div])
    > 此时剩下模板：` <!-- 我是注释 --><p> Text1<span>ggx</span></p><img src="xxx" /></div>`
3.  parseElement 函数解析 div 的子节点，这里又会递归调用 parseChildren 函数，创建一个新的状态机，我们这里称为**状态机 2**，在这个过程中，parseChildren 函数会消费掉模板内容：`<!-- 我是注释 -->`,
    > 此时模板为：`<p> Text1<span>ggx</span></p><img src="xxx" /></div>`
4.  在状态机 2 的基础上，遇到`<p>`标签节点，此时会走 parseElement 的函数处理逻辑，于是又开始解析 p 标签的开始标签，子节点，结束标签。在消费掉 p 标签的开始标签之后，并将解析出的 p 开始节点存进 ancestors 数组中([div, p])，
    > 此时剩下模板：` Text1<span>ggx</span></p><img src="xxx" /></div>`
5.  接着解析 p 标签的子节点，这里又会递归调用 parseChildren 函数，创建一个新的状态机，我们这里称为**状态机 3**， 在这个过程中，parseChildren 函数会消费掉模板内容：` Text1`,
    > 此时模板为：`<span>ggx</span></p><img src="xxx" /></div>`
6.  在状态机 3 的基础上，遇到 span 标签，此时会走 parseElement 的函数处理逻辑，于是又开始解析 span 标签的开始标签，子节点，结束标签。在消费掉 span 标签的开始标签之后，并将解析出的 span 开始节点存进 ancestors 数组中([div, p, span])，
    > 此时剩下模板：` ggx</span></p><img src="xxx" /></div>`
7.  接着解析 span 标签的子节点, 这里又会递归调用 parseChildren 函数，创建一个新的状态机，我们这里称为**状态机 4**， 在这个过程中，parseChildren 函数会消费掉模板内容：` ggx`,
    > 此时模板为：`</span></p><img src="xxx" /></div>`
8.  因为所有状态机都是在 while 循环中进行的，那么什么时候结束循环呢？我们判断当模板开头以 **</** 开始的标签与 ancestors 节点栈中存在同名的节点时，说明遇到了对应的闭合标签，那么跳出 while 循环（意味着结束了状态机 4），从 ancestors 栈中将 span 节点 pop 出去, 紧接着消费掉该结束标签，
    > 此时模板为：`</p><img src="xxx" /></div>`
9.  同理，遇到 p 标签的闭合标签，结束状态机 3，从 ancestors 栈中将 p 节点 pop 出去, 紧接着消费掉该结束标签，
    > 此时模板为：`<img src="xxx" /></div>`
10. 此时在状态机 2 的循环中，遇到标签 img, 此时会走 parseElement 的函数处理逻辑，于是又执行 parseTag 开始解析 img 标签的开始标签：`<img`, 由于 img 标签是一个自闭合标签，因此它没有子节点，直接消费掉，并返回节点信息，
    > 此时模板为：`</div>`
11. 接着在状态机 2 的循环中，继续判断：遇到 div 的结束标签，跳出状态机 2 的循环，并将 div 从 ancestors 栈中 pop 出去，紧接着消费掉该结束标签，
    > 此时模板为：''

以上就是模板的整个解析过程，可以用下图表示：

![](https://files.mdnice.com/user/23305/0a579592-db54-4640-b0de-9eb687abf183.png)

通过上述过程我们可以认识到，parseChildren 解析函数是整个状态机的核心，状态迁移的操作都在该函数内部完成。在 parseChildren 函数运行过程中，为了处理标签节点，会调用 parseElement 解析函数，这会间接地调用 parseChildren 函数，并产生一个新的状态机。随着标签嵌套层次的增加，新的状态机会随着 parseChildren 函数被递归地调用而不断创建，这种递归构造上下级模板节点的算法我们称为：**递归下降算法**

最终以上 demo 转换成的 AST 模板为：

```js
{
  type: "Root",
  children: [
    {
      type: "Element",
      tag: "div",
      props: [],
      children: [
        { type: "Text", content: " " },
        { type: "Comment", content: " 我是注释 " },
        { type: "Text", content: " " },
        {
          type: "Element",
          tag: "p",
          props: [],
          children: [
            { type: "Text", content: " Text1" },
            {
              type: "Element",
              tag: "span",
              props: [],
              children: [{ type: "Text", content: "ggx" }],
              isSelfClosing: false,
            },
            { type: "Text", content: " " },
          ],
          isSelfClosing: false,
        },
        { type: "Text", content: " " },
        {
          type: "Element",
          tag: "img",
          props: [{ type: "Attribute", name: "src", value: "xxx" }],
          children: [],
          isSelfClosing: true,
        },
        { type: "Text", content: " " },
      ],
      isSelfClosing: false,
    },
    { type: "Text", content: " " },
  ],
}
```

除了解析标签节点以外，vue 也需要解析模板中的属性和指令。例如：

```html
<div id="foo" v-show="display"></div>
```

上面模板中的 div 标签存在一个 id 属性和一个 v-show 指令。那这些属性和指令在什么阶段处理呢？我们注意到，parseElement 函数会通过 parseTag 函数消费开始标签：`<div`,
紧接着又会消费闭合标签 `>`， 而属性和指令存在于前面两次处理的中间，那么正确顺序为：

1. `<div`
2. `id="foo" v-show="display"`
3. `>`

第二个步骤我们通过 parseAttributes 函数处理，该函数只用来解析属性和指令，因此它内部会开启循环，不断地消费上面第二步的模板内容，直到遇到标签的闭合部分`>`或者结束部分`</`，就结束循环。

如下图所示：

![](https://files.mdnice.com/user/23305/d89f9f4c-3aff-4871-8f95-b19b9f59ea18.png)

parseAttributes 会通过正则表达式匹配属性和指令：

```js
const match = /^[^\t\r\n\f />][^\t\r\n\f />=]*/.exec(
  'id="foo" v-show="display"></div>'
);
const name = match[0]; // id
```

该正则表达式主要由反向字符集组成，匹配两个位置，分别是第一个字符和后面等号（=）之前的 0 个或多个：

![](https://files.mdnice.com/user/23305/57a9567f-e800-43bb-ab31-b1f7d49f05e1.png)

以上正则实现了匹配标签中的属性和指令。

parseAttributes 函数会按照从左到右的顺序不断的消费字符串，解析过程如下：

1. 通过正则拿到第一个属性名称 id，并消费掉字符串'id'。

   > 此时剩余模板为：`="foo" v-show="display"`

2. 消费掉属性 id 后面的空格, =。

   > 此时剩余模板为: `"foo" v-show="display"></div>`

3. 遇到单引号或者双引号，消费掉属性值的第一个单双引号。

   > 此时剩余模板为: `foo" v-show="display"></div>`

4. 通过 indexOf 找到第一个单双引号的索引位置，那么从最左到该索引位置的内容就是属性的值，拿到值后，消费掉内容、单双引号以及后面的空格

   > 此时剩余模板为: `v-show="display"></div>`

将以上步骤拿到的属性和值存入数组中，结束第一次循环。后面解析 v-show 的过程同理，最终遇到闭合标签`>`结束循环。此时 div 标签的 props 属性收集完毕。

最终我们收集到的内容为：

```js
[
  { type: "Attribute", name: "id", value: "foo" },
  { type: "Attribute", name: "v-show", value: "display" },
];
```

以下是 parseAttributes 核心原理的简单实现：

```js
function parseAttributes(context) {
  const { advanceBy, advanceSpaces } = context;
  const props = [];
  while (!context.source.startsWith(">") && !context.source.startsWith("/>")) {
    const match = /^[^\t\r\n\f />][^\t\r\n\f />=]*/.exec(context.source);
    const name = match[0];
    advanceBy(name.length);
    advanceSpaces();
    advanceBy(1);
    advanceSpaces();
    let value = "";
    const quote = context.source[0];
    const isQuoted = quote === '"' || quote === "'";
    if (isQuoted) {
      advanceBy(1);
      const endQuoteIndex = context.source.indexOf(quote);
      if (endQuoteIndex > -1) {
        value = context.source.slice(0, endQuoteIndex);
        advanceBy(value.length);
        advanceBy(1);
      } else {
        console.error("缺少引号");
      }
    } else {
      const match = /^[^\t\r\n\f >]+/.exec(context.source);
      value = match[0];
      advanceBy(value.length);
    }
    advanceSpaces();
    props.push({
      type: "Attribute",
      name,
      value,
    });
  }
  return props;
}
```

除了以上介绍的解析标签以及标签属性的以外，其实我们还需要解析**HTML 实体**，它分为两类：

1. 命名字符引用（named character reference）
2. 数字字符引用 (numeric character reference)

举个例子，用户编写了如下模板，最终预期渲染到页面的内容为：'1<2'

```js
const template = "<div>1&lt;2<div>";
```

这里的`&lt;`我们就称为命名字符引用。

这里我们需要知道的重点并不是 html 实体的解析过程，我们需要知道一个原则：

首先拿一个简单的命名字符引用表为例，内容如下：

```js
 {
    "lt;": "<",
    "ltcc;": "⪦"
  }
```

举个例子：

```js
const template1 = '<div>1&ltcc2<div>'；
const template2 = '<div>1&ltcc;2<div>'
```

以上两者的区别在于一个没有用分号区分，一个用了分号区分。但解析策略完全不一样。

- 当没有用分号隔开的命名字符引用，浏览器的解析规则是：**最短匹配原则**，template1 会被解析为：<div>1<cc2<div>
- 当用分号隔开的命名字符引用，浏览器的解析规则是：**完整匹配原则**，template2 会被解析为：<div>1&ltcc;2<div>

### 总结

我们详细介绍了 vue3 的模板解析中针对标签节点以及属性的解析过程，本质上是一个状态机，针对标签类型做了状态的迁移。每层嵌套的子级标签都会导致开启一个新的状态机去解析当前层级的节点，最终构建出每层节点的 children，最终构建出一个完整的 AST 对象，我们称这样的算法为**递归下降算法**。我们还介绍了 vue3 在解析文本节点中针对 HTML 实体的匹配原则：**当存在分号时, 执行完整匹配，当不存在分号时，执行最短匹配。**
