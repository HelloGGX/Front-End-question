# 关于 NodeJS 模块系统的思考

在探索 NodeJS 模块系统时，我们需要思考下面几个问题：

1. Node.js 平台为什么一定要有模块，该平台总共有几种模块系统
   - 把代码库划分成为多个文件。让代码更有条理、更容易理解、更容易测试
   - 可以通用的功能，并复用
   - 实现逻辑的封装，隐藏复杂的实现，暴露简单的接口
   - 更好的管理依赖，让开发者根据现有模块，轻松的构建其他模块
   - 一共有两种模块系统，CommonJS 和 ESM
2. CommonJS 的内部原理以及相关的模块模式
3. Node.js 平台的 ES module（ESM）模块系统
4. CommonJS 与 ESM 的区别

# 1、CommonJS 的内部原理以及相关的模块模式

CommonJS 内部实现的基本思路参考了 revealing module 模式：一种利用了立即调用函数表达式子的方式（IIFE）,它可以创建一个私有的作用域，该作用域内部的变量不受函数以外的影响，通过 return 暴露需要公开的 API。
当我们通过 require 加载一个模块时， require 内部会将模块的源码通过一个立即调用函数包裹:

```js
(function (exports, require, module, __filename, __dirname) {
  // 你的代码在这里
})(module.exports, require, module, __filename, __dirname);
```

因此你的源码内部，可以顺利地使用`__filename`、`__dirname`等变量的信息，并且你的代码在 require 函数创建的私有作用域里面，不受外界的影响。
通过我们也需要注意三个细节：

_细节一： exports 变量其实是 module.exports 初始值的引用，这个初始值仅仅是加载模块之前创建的对象字面量： 一个空对象{}。_
伪代码实现如下：

```js
function Module(id) { 
  this.id = id;  // 读取到的文件内容会放在exports中 
  this.exports = {};
}；
const module = new Module(模块地址);
(function (exports, require, module, __filename, __dirname) {
  // 你的代码在这里
})(module.exports, require, module, __filename, __dirname);
```

也就意味着我们在使用 exports 时，可以这样用：

```js
// 给空对象添加属性
exports.hello = () => {
  console.log("hello world");
};
```

但不能这样用：

```js
exports = () => {
  console.log("hello world");
};
```

exports 指向 module.exports 的内存地址，上面的赋值方式只是让 exports 变量重新指向了一个新的函数，而不会改变 module.exports 的内容，因此如果想通过直接赋值，而不是通过给对象字面量里添加新属性的办法，来公布某个函数、实例或字符串，那么赋值的对象应该是：**module.exports**

> 这里容易存在一个误解：CommonJS 其实只允许 exports 这种方式，module.exports 这种方式其实是 Nodejs 做的补充。

```js
module.exports = () => {
  console.log("hello world");
};
```

_细节二： 系统只会在程序首次索要某个模块的时候，加载并执行该模块中的代码，如果以后还有别的语句也调用 require()函数来索要这个模块，那么系统只会把早前缓存的那一份返回给调用放_。伪代码实现如下：

```js
// 定义导入类，参数为模块路径
function Require(modulePath) {
  // 获取当前要加载的绝对路径
  let absPathname = resolve(modulePath); // 从缓存中读取，如果存在，直接返回结果
  if (Module._cache[absPathname]) {
    return Module._cache[absPathname].exports;
  } // 创建模块，新建Module实例
  const module = new Module(absPathname); // 添加缓存
  Module._cache[absPathname] = module; // 加载当前模块
  tryModuleLoad(module); // 返回exports对象
  return module.exports;
}
```

_细节三： require（）函数是一个同步的函数，它仅仅是直接将模块内容返回而已，并没有用到回调机制，所以针对 module.exports 的赋值操作，也必须是同步的。_

# 2、commonjs 模块的解析规则

我们可以从上一节细节二的伪代码中看到，require 函数需要根据传入的 modulePath 获取该模块的完整路径， 那么 resolve 函数用到的解析算法是如何查找模块的呢？ 一般会有三种常见的情况：

- **要加载的是文件模块**：以./或者/开头的路径，直接返回
- **要加载的是 node 核心模块**：如果 moduleName 既不是以/开头，也不是以./开头，那么首先尝试寻找 Node.js 核心模块
- **要加载的是包模块**：如果没有找到与 moduleName 相匹配的核心模块，那就从发出加载请求的这个模块开始，逐层向上搜寻名为 node_module 的目录，如果有就加载，如果没有就沿着目录树继续向上走，在相应的 node_module 目录中搜索。举个例子：

如果 **/home/ggx/projects/foo.js** 中的文件调用了 `require('bar.js')` ，node 将在下面的位置进行搜索：

1. /home/ggx/projects/node_modules/bar.js
2. /home/ggx/node_modules/bar.js
3. /home/node_modules/bar.js
4. /node_modules/bar.js

在解析文件模块和包模块的时候，解析算法会按照下面的顺序匹配:

1. 看主文件名是 moduleName 且后缀名为.js 的文件，也就是`<moduleName>.js`
2. 其次看有没有名为 moduleName 的目录，且该目录下有没有名为 index.js 的文件，也就是 `<moduleName>/index.js`
3. 最后有没有名为 moduleName 的目录，且该目录下有没有 package.json 的文件，如有就采用其中的 main 属性所指定的目录或文件

# 3、ESM 模块的加载过程

node.js 平台默认会把所有的 js 文件，都当成采用 CommonJS 语法所写的文件，因此如果你直接在.js 文件里面采用 ESM 语法来写代码，那么解释器会报错。有几种办法让 node 识别 ES 模块：

1. 把模块的后缀名写成.mjs
2. 给最近的上级 package.json 文件添加名为”type“的字段，并将字段值设为”module“

node 解释器启动的时候，会从一个 JavaScript 文件开始解析，这份文件就是解析依赖关系的出发点，也叫做“入口”。
解释器会从入口点开始，寻找所有的 import 语句，如果在寻找过程中又遇到了 import 语句，那么就会以“深度优先的方式递归”，直到把所有代码解析执行完毕。
这个过程细分为三个阶段：

1. **构造（Construction）**: 找到所有的 import 语句，递归地从相关文件里加载每个模块的内容
2. **实例化（Instantiation）**：在有依赖关系的点之间创建路径
3. **执行（Evaluation）**： 按照正确的顺序遍历这些路径

以上过程必须是静态的，在编译阶段就先把依赖图完整的构造出来，然后才能开始执行代码，因此 ESM 的引入操作都必须在内容顶部。

这与 CommonJS 的加载方式有本质的区别，require 语句在代码执行期才开始执行，因此它可以根据条件语句动态引入，出现在内容的任何地方。

ESM 模块还有一项重要的特征：**它在引进来的模块与该模块所导出的值之间，建立了一种 live 绑定关系，然而这种绑定关系在引入方这一端是只能读不能写的。**

举个例子：

```js
// counter.js
export let count = 0;
export function increment() {
  count++;
}

// main.js
import { count, increment } from "./counter.js";
console.log(count); //打印0
increment();
console.log(count); //打印1
count++; // 报错TypeError: Assignment to constant variable
```

引入方 main.js 中，引入的 count 是只读的，不能写， 但我们可以通过 counter.js 暴露的函数来修改 count, 并且可以实时的拿到最新的值，因此我们称：在 main.js 引入的 count 和 counter.js 导出的 count 建立了 live 绑定关系。

这与 CommonJS 系统所用的方法有本质的区别，我们用 CommonJS 的方式实现上面的例子:

```js
// counter.js
let count = 0;
function increment() {
  count++;
}
module.exports = { count, increment };

// main.js
const counter = require("./counter");
console.log(counter.count); //打印0
counter.increment();
console.log(counter.count); //依然打印0
counter.count++; //不报错
```

我们可以发现区别。如果有某个模块要引入 CommonJS 模块，那么系统会对后者的整个 exports 对象做浅拷贝，从而将内容复制到当前模块里面。

于是我们在 main.js 中访问的 counter.count 和 counter.increment 是 counter.js 这个模块所导出的副本，而不会与 counter.js 这个模块中的变量联动。
因此用户拿到的 counter.count 一直是拷贝过来的副本，不会是新的值。

# 4、CommonJS 与 ESM 模块在循环引用上处理的不同

我们在 2 和 3 小节分析了两种模式的解析过程，了解到 ESM 模块是在静态编译阶段就已经构造好了依赖图，它不是运行时执行的。

而 CommonJS 则是在运行时才执行模块的加载的，并且 CommonJS 模块的引入是对该模块的 exports 出的对象的浅拷贝，而 ESM 模块的引入和导出是建立了实时绑定关系，模块之间可以拿到最新的状态。

这种本质的区别决定了以上俩 种方式在处理循环依赖时的不同。

## 4.1 CommonJS 处理循环依赖

下面是一个典型的循环依赖的例子：

![](https://files.mdnice.com/user/23305/a1505a6e-ab10-47df-888b-f1ccd59213c6.png)

```js
// a.js
exports.loaded = false;
const b = require("./b");
module.exports = {  b,  loaded: true};

//b.js
exports.loaded = false;
const a = require("./a");
module.exports = {  a,  loaded: true};

//main.js
const a = require("./a");
const b = require("./b");
console.log("a ->", JSON.stringify(a, null, 2));
console.log("b ->", JSON.stringify(b, null, 2));

//打印结果
a -> {
  "b": {
    "a": {
      "loaded": false
    },
    "loaded": true
  },
  "loaded": true
}
b -> {
  "a": {
    "loaded": false
  },
  "loaded": true
}
```

我们会发现在 CommonJS 的模式下，a 和 b 打印出来的结果不同，我们用时序图来描述内部的过程：

![](https://files.mdnice.com/user/23305/ed2664e1-c8f1-4c87-9e27-cc30514f4b7c.png)

我把步骤标为了序号，重点解释一些重要步骤：

1. 第 5 步的时候，就已经形成了循环依赖，此时在 b 模块中，由于系统已经开始处理 a 模块了，那么 b 模块会把 a.js 导出的内容复制一份到本模块范围内
2. 第 6 步的时候，在 b 模块中，require 完 a 模块之后，把 loaded 更新为 true,此时 b 模块内容为第 7 步的内容
3. 第 8 步和第 9 步： 由于 b.js 已经彻底执行完了，因此控制权交给了 a.js，此时会把 b.js 模块当前的状态拷贝一份，放到 a 模块中。也就意味着，在 a.js 的 require（'./b'）执行完之后，a 模块拿到的是当前 b 模块状态的副本。紧接着更新 loaded: true，覆盖该副本。
4. 第 10 步： a.js 执行完之后，控制权交给 main.js，此时会把 a.js 模块的当前状态浅拷贝一份，放到自己这里
5. 第 11 步和 12 步： 在 main.js 中，执行完了 require('./a'), 紧接着执行 require('./b'),由于该模块已经载入，因此 require 函数内部立即返回缓存中的该模块，也就是第 7 步中的最新状态。最终打印结果为第 12 步所示。

我们会发现出现 CommonJS 这样的情况，正是由于 require 是在运行期执行的，导入 require 的内容其实就是一次浅拷贝， 模块始终拿到的导出内容的副本， 再加上 require 函数内部针对模块做了缓存，因此只要某个模块加载过， 就不会重新加载了，而是直接拿缓存中最新的状态作为浅拷贝对象。
以上出现的情况对大型项目来说是致命的，会造成不可预知的依赖问题。

## 4.2 ESM 处理循环依赖

还是上面的例子，我们改成 ESM 模式：

```js
//a.js
import * as bModule from './b.js';
export let loaded = false;
export const b = bModule;
loaded = true;

//b.js
import * as aModule from './a.js';
export let loaded = false;
export const a = aModule;
loaded = true;

//main.js
import * as a from './a.js';
import * as b from './b.js';

console.log('a->', a);
console.log('b->', b);

//打印出来
a-> <ref *1> [Module: null prototype] {
  b: [Module: null prototype] { a: [Circular *1], loaded: true },
  loaded: true
}
b-> <ref *1> [Module: null prototype] {
  a: [Module: null prototype] { b: [Circular *1], loaded: true },
  loaded: true
}
```

我们发现 a 和 b 这两种模块可以完整的获取对方的实时状态，它们的 loaded 都是 true。
因为在 ESM 模式中，a 模块拿到的 b 和当前范围之中的 b,其实是同一个实例，反之亦然。

ESM 的解析过程如下：

**第一阶段： 构造**

从 main.js 这个文件入口开始，系统会搜索模块中的 import 语句，并把这些语句想要引入的模块所对应的源代码加载进来。解释器以深度优先的方式搜索依赖关系图，而且只把图中的每个模块访问一次

![](https://files.mdnice.com/user/23305/be049dc6-c1ba-4b5c-9eb7-e31ca61e4801.png)

**第二阶段：实例化**

在这一阶段，解释器会从树状结构的底部开始，逐渐向顶部走。每走到一个模块，就寻找该模块所要导出的全部属性，并在内存中构建一张映射表，以存放此模块所要导出的属性名称与该属性即将拥有的取值（此时这些值还没有初始化）。

![](https://files.mdnice.com/user/23305/b3223c29-e6d7-4806-aaaf-e6e1636e2ca3.png)

紧接着，解释器还要再过一遍，这次，它会把各个模块所导出的名称与引入这些名称的模块链接起来：

![](https://files.mdnice.com/user/23305/37ce2b9b-14ce-48a9-bb41-27259f0c6486.png)

这个阶段只是建立了链接，让这些链接能够指向相应的值，至于值本身，则要等到下一个阶段结束才能确定。

**第三阶段：执行**

这个阶段，系统会执行每份文件里面的代码。它按照后序深度优先顺序， 由下而上地访问最初那张依赖图，并逐个执行访问的文件。因此本例中的 main.js 文件会在最后执行。

这种执行顺序能够保证程序开始运行主业务逻辑的时候，各模块所导出的那些值，全部已经做了初始化。

下面是模块的执行顺序：
![](https://files.mdnice.com/user/23305/9bca4884-fc41-4a08-bc49-c0e504f14492.png)

最终系统是通过引用而不是复制来引入模块的，因此就算模块之间有循环依赖关系，每个模块也还是能够完整地“看”到对方的最终状态。

# 总结

NodeJS 平台有两种模块管理方式： CommonJS 和 ESM， nodeJS 平台默认是 commonJS 加载模块的方式，该方式的基本实现思路来源于： **revealing module 模式**，也就是利用了自执行函数能够创建私有作用域的特点。

而 ESM 模式需要 package.json 中的配置**type: module**，来让 node 平台识别。CommonJS 通过 require 引入模块，这种引入方式在代码运行时进行，因此 require 函数可以写在模块的任何位置（条件语句中也行）。 require 内部实现利用了自执行函数，将模块源码包裹到函数内部，并传入参数如： `module.exports, require, module, __filename, __dirname`等， 我们可以从中了解到， exports 其实就是 module.exports 的初始值(一个对象字面量)的引用，因此我们可以给 exports 赋值属性导出，但是不能直接给 exports 赋值，因为这样只是改变了 exoprts 的指向的地址，没有改变 module.exports 的指向。

CommonJS 加载方式与 ESM 加载方式有本质的区别，前者针对模块的 exports 做了一层浅拷贝，当我们 require 模块时，其实拿到的是该模块 exports 出的副本。 模块只要加载过了就会存入缓存中，下次加载会直接从缓存中获取。而 ESM 模块会通过构造、实例化、执行三个阶段，采用深度优先的方式在代码预编译（静态分析）时就确定了依赖有向图关系，而这些关系本质上是实例的引用而不是复制，因此在 ESM 的循环依赖的模块中，我们依然能相互拿到最新的值。
