# vue3 如何监听每个属性的变化

首先我们看一个简单的 vue 应用：

```html
<body>
  <div id="app">
    <h1>{{ message }}</h1>
    <p>{{name}}</p>
    <p>{{age}}</p>
    <p>{{skill}}</p>
    <input type="text" v-model="skill" />
  </div>
</body>
<script>
  const app = Vue.createApp({
    data() {
      return {
        age: 20,
        name: "ggx",
        skill: "react",
      };
    },
    computed: {
      message() {
        return this.age > 18 ? "成年人" : "未成年人";
      },
    },
  });
  const vm = app.mount("#app");
  vm.age++;
</script>
```

html 的 input 标签与 data 中的 skill 变量建立了双向数据绑定，当我们改变 input 框中的值，此时 p 标签中的值会实时变化，当我们给变量 age 自增 1，那 p 标签中 age 的值渲染出来是 21。

那 vue 是如何知道某个变量改变，我需要去重新更新这个变量，重新去渲染该变量所在的 dom 呢？

下面，我会逐步实现 vue3 的响应式系统，并逐步贴近 vue3 的真实表现。

# Proxy 代理

ECMAScript 6 新增的代理和反射为开发者提供了拦截并向基本操作嵌入额外行为的能力。具体地
说，可以给目标对象定义一个关联的代理对象，而这个代理对象可以作为抽象的目标对象来使用。在对
目标对象的各种操作影响目标对象之前，可以在代理对象中对这些操作加以控制。

它可以用作目标对象的替身，但又完全对立于目标对象。

vue3 便使用代理对象的方式，监听对象的访问，赋值等操作。

```js
const data = {
  age: 20,
  name: "ggx",
  skill: "react",
  ok: true,
};
const obj = new Proxy(data, {
  get(target, key) {
    console.log("我被访问了");
    return target[key];
  },
  set(target, key, value) {
    console.log("我被赋值了");
    target[key] = value;
    return true;
  },
});
```

obj 作为 data 的代理对象，每当访问 obj 或者给 obj 的属性赋值时，都会触发 get 和 set 的回调，我们后面的一系列实现都是基于 obj，而不是 data。

当 obj.age++;我们需要触发页面的重新渲染，这里我们使用一个 effect 函数模拟页面的渲染，你可以想象成 effect 的回调返回的是一个模板组件：

```js
effect(() => <Component />);
```

**需要解决的问题 1: 当 age 属性自增 1，effect 函数内部的回调会重新执行。那如何实现呢？**

```js
effect(() => {
  console.log(obj.age);
});
obj.age++;
```

**实现思路: 首先，我们需要一个全局变量 activeEffect,存储被注册的副作用函数，effect 函数接收一个参数 fn,即要注册的副作用函数，当代理对象被赋值，触发 set 回调，重新执行全局变量中存储的 effect 回调**

```js
const data = {
  age: 20,
  name: "ggx",
  skill: "react",
  ok: true,
};

let activeEffect;

const obj = new Proxy(data, {
  get(target, key) {
    return target[key];
  },
  set(target, key, value) {
    target[key] = value;
    activeEffect();
    return true;
  },
});

function effect(fn) {
  activeEffect = fn;
}

effect(() => {
  console.log(obj.age);
});

obj.age++;
```

目前的实现暂时解决了问题 1，但是过于粗糙，因为，当我们执行 obj.skil = 'vue'的时候，依然会执行 console.log(obj.age);，哪怕这个 effect 回调跟 skill 这个变量无关。

**需要解决的问题 2: 我们如何更精细的知道哪个属性“注册”过哪个 effect，当这个属性被赋值时，我们需要执行“依赖“于这个属性的 effect 回调呢？也就是说：如何将副作用函数与被操作的目标字段之间建立明确的联系？**

<img width=600 src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/vue-map0.png"/>

**实现思路：我们需要实现上图这样的树型结构，需要有一个数据结构存储 target（代理对象）与它的所有 key 的联系，并且还需要有一个数据结构存储 key 与副作用函数的联系**

<img width=600 src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/vue-map.png"/>

接下来我们尝试用代码实现这样的数据结构，首先需要使用 WeakMap 存储 target，至于为什么要用 WeakMap，下节会讲：

```js
const bucket = new WeakMap();
```

然后修改 get/set 拦截器代码：

```js
const data = { name: "ggx", skill: "react", ok: true, age: 20 };
let activeEffect;
const bucket = new WeakMap();
const obj = new Proxy(data, {
  get(target, key) {
    if (!activeEffect) return;
    let depsMap = bucket.get(target);
    // 建立target与depsMap的关联
    if (!depsMap) bucket.set(target, (depsMap = new Map()));
    let deps = depsMap.get(key);
    // 建立depsMap与key的关联
    if (!deps) depsMap.set(key, (deps = new Set()));
    deps.add(activeEffect);
    return target[key];
  },
  set(target, key, value) {
    target[key] = value;
    // 根据target从WeakMap中取得depsMap, 它是 key --> effects
    let depsMap = bucket.get(target);
    if (!depsMap) return;
    // 根据key取得所有副作用函数的effects
    let deps = depsMap.get(key);
    // 依次执行依赖的副作用函数
    deps && deps.forEach((dep) => dep());
    return true;
  },
});
function effect(fn) {
  activeEffect = fn;
  fn();
}
effect(() => {
  console.log(obj.age);
});
obj.age++;
obj.name = "super";
```

我们可以看到：effect 的 fn 中，访问了 obj.age, 于是触发代理的 get 回调，收集 activeEffect 作为 age 的副作用函数。当 age++时，触发代理的 set 回调，然后取出 age 所对应的依赖副作用函数执行，于是打印：20，21。当执行 obj.name = 'super'时，由于还没有触发 name 的依赖收集，于是不会在 name 的名下找到所依赖的副作用函数，更不会执行。

## WeakMap

WeakMap 对象是一组健/值对的集合，其中的健是弱引用，健 key 只能是 Object 类型(不能包含无引用的对象，否则会被自动清除回收)，原始数据类型（string，number，bigint，boolean，null，undefined，symbol)

> 在计算机程序设计中，弱引用与强引用相对，是指不能确保其引用的对象不会被垃圾回收器回收的引用。一个对象若只被弱引用所引用，则被认为是不可访问（或弱可访问）的，并因此可能在任何时刻被回收。

如果我们持有对一个对象的引用，那么这个对象就不会被垃圾回收。这里的引用，指的是强引用。
一个对象若只被弱引用所引用，则被认为是不可访问（或弱可访问）的，并因此可能在任何时刻被回收。

### WeakMap 和 Map 的区别

#### Map 的内部实现

Map api 共用了两个数组（一个存放 key,一个存放 value）。给 Map set 值时会同时将 key 和 value 添加到这两个数组的末尾。从而使得 key 和 value 的索引在两个数组中相对应。当从 Map 取值时，需要遍历所有的 key，然后使用索引从存储值的数组中检索出相应的 value。这个实现的缺点很大，首先是赋值和搜索的时间复杂度为 O(n)；其次是可能导致内存溢出，因为数组会一直保存每个键值引用，即便是引用早已离开作用域，垃圾回收器也无法回收这些内存。

```js
const map = new Map();
const weakMap = new WeakMap();
(function () {
  const foo = { foo: 1 };
  const bar = { bar: 2 }; // 强引用
  map.set(foo, 1); // 弱引用，
  weakMap.set(bar, 2);
})();
```

让立即执行函数执行完毕之后，对于对象 foo 来说，他任然作为 map 的 key 被引用着，因此垃圾回收器不会把他从内存中移除，我们仍然可以通过 map.keys 打印出对象 foo。而 WeakMap 在立即执行函数执行完毕之后，垃圾回收器就会把对象 bar 从内存移除。

所以 WeakMap 经常用于存储那些只有当 key 所引用的对象存在时（没有被回收）才有价值的信息。WeakMap 结构有助于防止内存泄漏。

然后修改 get/set 拦截器代码中的逻辑抽离出来，封装为两个函数一个负责收集副作用依赖名为：track， 一个负责触发执行副作用依赖名为：trigger;

```js
function track(target, key) {
  if (!activeEffect) return;
  let depsMap = bucket.get(target);
  if (!depsMap) bucket.set(target, (depsMap = new Map()));
  let deps = depsMap.get(key);
  if (!deps) depsMap.set(key, (deps = new Set()));
  deps.add(activeEffect);
}
function trigger(target, key) {
  let depsMap = bucket.get(target);
  if (!depsMap) return;
  let deps = depsMap.get(key);
  deps && deps.forEach((dep) => dep());
}
```

# 完善响应式系统

目前来看，我们的响应式只是实现了基本的功能，其实还有很多场景没有考虑到：

## 需要解决的问题 1：

```js
effect(() => {
  document.body.innerText = obj.ok ? obj.name : "渲染";
  console.log("执行了");
});
obj.ok = false;
obj.name = "ggx1";
obj.name = "ggx2";
```

目前 ok 已经被赋值为 false 了，无论设置多少次 obj.name，effect 渲染的 UI 依然是"渲染"这个文案, 因此理想情况是：无论 obj.name 被赋值多少次都不会造成其副作用函数的执行；

> 解决思路： 收集的时候建立连接，赋值触发执行的时候，断开连接。那么就需要拿到 key 对应的所有副作用依赖，并在每次执行的时候，清除所有的依赖。

```js
function track(target, key) {
  if (!activeEffect) return;
  let depsMap = bucket.get(target);
  if (!depsMap) bucket.set(target, (depsMap = new Map()));
  let deps = depsMap.get(key);
  if (!deps) depsMap.set(key, (deps = new Set()));
  deps.add(activeEffect);
  // deps是一个与当前副作用函数存在联系的依赖集合，并将其添加到activeEffect.deps中
  activeEffect.deps.push(deps);
}
function trigger(target, key) {
  let depsMap = bucket.get(target);
  if (!depsMap) return;
  let deps = depsMap.get(key);
  // 新增逻辑：解决无限forEach执行的问题
  const depsToRun = new Set(deps);
  depsToRun.forEach((dep) => dep());
}
function cleanup(effectFn) {
  for (let i = 0; i < effectFn.deps.length; i++) {
    const deps = effectFn.deps[i];
    // 将effectFn从依赖集合中移除
    deps.delete(effectFn);
  }
  effectFn.deps.length = 0;
}
function effect(fn) {
  function effectFn() {
    // 新增cleanup清空与当前副作用函数存在联系的依赖集合
    cleanup(effectFn);
    activeEffect = effectFn;
    fn();
  }
  // 用来存储所有与该副作用函数相关联的依赖集合
  effectFn.deps = [];
  effectFn();
}
effect(function effectFn() {
  document.body.innerText = obj.ok ? obj.name : "渲染";
  console.log("执行了");
});

obj.ok = false;
obj.name = "ggx1";
```

我们用一张图表示上面代码的依赖关系：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/vue-map1.png"/>

当 obj.ok 被赋值时，会触发 trigger 执行所有副作用依赖：effectFn， 该函数内部会执行 cleanup, 依次断开 ok 和 name 的副作用依赖，并清空数组 deps， 紧接着会触发传入 effect 的函数，又会进行依赖收集，而此时 obj.ok 早已变成了 false, 因此根据三目运算逻辑，只会触发对 obj.ok 的依赖收集，不会触发 obj.name 的依赖收集。因此后面针对 obj.name 的赋值操作，name 没有可以执行的副作用函数。

额外收获：我们会注意到，在 trigger 函数里面新增了这样一段逻辑：

```js
let deps = depsMap.get(key);
// 新增逻辑：解决无限forEach执行的问题
const depsToRun = new Set(deps);
depsToRun.forEach((dep) => dep());
```

如果不加这段逻辑，deps.forEach 会一直执行下去，因为我们在执行 cleanup 的时候，会删除 deps 中的元素，当执行 effectFn 的时候，又会触发依赖收集，又会增加 deps 中的元素，导致 forEach 遍历一直不结束。

> 语言规范中对此有明确的说明：在调用 forEach 遍历 Set 集合时，如果一个值已经被访问过，但该值被删除并重新添加到集合，如果此时 forEach 遍历还没有结束，那么该值会被重新访问。

## 需要解决的问题 2：

effect 嵌套问题，当给 name 赋值时，理想情况下，应该执行外层和内层的副作用函数，但现在只执行了内层的

```js
let temp = "";
effect(() => {
  console.log("外层执行");
  effect(() => {
    console.log("内层执行");
    temp = obj.name;
  });
  temp = obj.skill;
});
obj.skill = "vue";
```

原因分析：activeEffect 只有一个，当执行内层的 effect 函数时，activeEffect 指向内层的 effectFn

解决思路：使用一个栈，存储每个 effectFn, 当副作用函数执行完毕就抛出，把内部的 effectFn 置为 activeEffect

```js
const effectStack = [];
function effect(fn) {
  function effectFn() {
    cleanup(effectFn);
    // 在调用副作用函数之前将当前副作用函数压入栈中
    effectStack.push(effectFn);
    activeEffect = effectFn;
    fn();
    // 执行完副作用函数后就弹出当前的副作用函数effectFn
    effectStack.pop();
    activeEffect = effectStack[effectStack.length - 1];
  }
  effectFn.deps = [];
  effectFn();
}
```

## 需要解决的问题 3：

effect 内部当是 obj.age++时，会导致无限循环，栈溢出。

```js
effect(() => {
  obj.age++;
});
```

原因分析：obj.age = obj.age + 1; 会触发依赖收集，然后触发副作用函数执行，又回触发收集，导致无限循环
解决思路：在 trigger 做个判断，当 age 依赖的副作用函数就是当前的 activeEffect 时，那就过滤掉，也就不会执行，不会再次触发依赖收集

```js
function trigger(target, key) {
  let depsMap = bucket.get(target);
  if (!depsMap) return;
  let deps = depsMap.get(key);
  const depsToRun = new Set();
  deps &&
    deps.forEach((effectFn) => {
      // 如果trigger触发执行的副作用函数与当前正在执行的副作用函数相同，则不触发执行
      if (effectFn !== activeEffect) {
        depsToRun.add(effectFn);
      }
    });
  depsToRun && depsToRun.forEach((dep) => dep());
}
```

## 需要解决的问题 4：

```js
effect(() => {
  console.log(obj.age);
});
obj.age++;
obj.age++;
```

当 obj.age 自增 2 次， 20 -> 21 -> 22,   其实页面最终渲染出来是 22，age 是 21 的情况只是个中间过程，无需渲染，如何做到中间过程不执行副作用函数，只在最终的结果执行副作用函数？

这时就需要响应系统支持调度了。

> 可调度性是响应式系统非常重要的特性。首先我们需要明确什么是可调度性。所谓可调度，指的是当 trigger 动作触发副作用函数重新执行时，有能力决定副作用函数执行的时机，次数以及方式

因此上面的问题实现的思路是：我们需要一个在触发 trigger 执行时的一个回调 scheduler，拿到副作用函数，并决定执行时机。利用调度器 scheduler,将 fn 放入微任务队列中，当所有同步任务执行完毕，再执行微任务中的 fn，无论 key 被赋值多少次，由于目前只有一个 effectFn, 利用 Set 的的去重能力，最终在微任务中的副作用函数只执行一次。

```js
function trigger(target, key) {
  let depsMap = bucket.get(target);
  if (!depsMap) return;
  let deps = depsMap.get(key);
  const depsToRun = new Set();
  deps &&
    deps.forEach((effectFn) => {
      if (effectFn !== activeEffect) {
        depsToRun.add(effectFn);
      }
    });
  depsToRun &&
    depsToRun.forEach((dep) => {
      // 如果一个副作用函数存在调度器，则调用该调度器，并将副作用函数作为参数传递
      if (dep.option.scheduler) {
        dep.option.scheduler(dep);
      } else {
        dep();
      }
    });
}

// 给effect添加一个option选项
function effect(fn, option) {
  function effectFn() {
    cleanup(effectFn);
    effectStack.push(effectFn);
    activeEffect = effectFn;
    fn();
    effectStack.pop();
    activeEffect = effectStack[effectStack.length - 1];
  }
  effectFn.deps = [];
  // 让effectFn的option属性指向option选项
  effectFn.option = option;
  effectFn();
}
effect(
  () => {
    console.log(obj.age);
  },
  {
    scheduler: () => {},
  }
);
```

首先，我们定义一个任务队列 jobQueue,它是一个 Set 数据结构，目的是利用 Set 数据结构的去重能力。接着利用调度器 scheduler，每次调度执行时，先将当前副作用函数添加到 jobQueue 队列中，再调用 flushJob 函数刷新队列，而 flushJob 内通过 p.then 将一个函数添加到微任务队列，在微任务队列内完成对 jobQueue 的遍历执行。因此无论调用多少次 flushJob 函数，jobQueue 的遍历只会执行一次，要等到该微任务结束之后，才能进行下一次。

```js
const p = Promise.resolve();
let isFlushing = false;
const jobQueue = new Set();
function flushJob() {
  if (isFlushing) return;
  isFlushing = true;
  p.then(() => {
    jobQueue.forEach((effectFn) => effectFn());
  }).finally(() => {
    isFlushing = false;
  });
}
effect(
  () => {
    console.log(obj.age);
  },
  {
    scheduler: (effectFn) => {
      jobQueue.add(effectFn);
      flushJob();
    },
  }
);
obj.age++;
obj.age++;
```

因此上面的代码打印出来 20，22，不会打印 21。

# 懒执行：

**如何实现懒执行？**

实现思路：我们依然利用 effect 函数中的 option 这个选项，新增一个属性：lazy。在 effect 函数中判断，当存在 lazy 选项时，直接返回 effectFn 的函数引用，暴露给外部，如果不存在 lazy 选项时，就直接执行。

```js
function effect(fn, option) {
  function effectFn() {
    cleanup(effectFn);
    effectStack.push(effectFn);
    activeEffect = effectFn;
    fn();
    effectStack.pop();
    activeEffect = effectStack[effectStack.length - 1];
  }
  effectFn.deps = [];
  effectFn.option = option;
  // 新增逻辑
  if (option?.lazy) {
    return effectFn;
  }
  effectFn();
}
const lazyEffect = effect(
  () => {
    console.log(obj.name);
  },
  {
    lazy: true,
  }
);
lazyEffect();
```

# computed 实现及完善

实现思路：依然利用调度器 scheduler, lazy 的特性，只返回 effectFn 的引用，利用对象的 get 属性，拿到 effectFn 的执行结果

```js
function effect(fn, option) {
  function effectFn() {
    cleanup(effectFn);
    effectStack.push(effectFn);
    activeEffect = effectFn;
    // 拿到副作用函数的执行结果
    const res = fn();
    effectStack.pop();
    activeEffect = effectStack[effectStack.length - 1];
    // 返回执行结果，供computed返回
    return res;
  }
  effectFn.deps = [];
  effectFn.option = option;
  if (option?.lazy) {
    return effectFn;
  }
  effectFn();
}

function computed(callback) {
  const lazyEffect = effect(callback, {
    lazy: true,
  });
  const obj = {
    get value() {
      return lazyEffect();
    },
  };
  return obj;
}
const result = computed(() => obj.name + obj.skill);
obj.skill = "vue";
console.log(result.value);
```

## 缓存计算属性的结果

我们发现多次访问 computed 返回的值，相当于执行了副作用函数，每次访问，都会导致副作用函数的计算， 即使值本身没有任何变化，而理想情况是：缓存 computed 的返回结果，多次访问，不会导致 computed 的副作用函数的重新执行。

实现思路：观察 computed 函数的逻辑实现，发现当访问 obj 属性的时候，其实会触发内部 obj 对象的 get 方法的执行，这里其实可以利用闭包的特性在 get 方法内部去保留外层的变量，达到多次访问，根据变量判断是否执行副作用函数。

```js
function computed(callback) {
  // dirty标识，用来标识是否需要重新计算值，为true则意味着"脏",需要计算
  let dirty = true;
  // value用来缓存上一次的计算值
  let value;
  const lazyEffect = effect(callback, {
    lazy: true,
  });
  const obj = {
    get value() {
      // 只有"脏"时才计算值，并将得到的值缓存到value中
      if (!dirty) return value;
      // 设置为false,下一次访问直接使用缓存到value中的值
      dirty = false;
      value = lazyEffect();
      return value;
    },
  };
  return obj;
}
```

## 解决计算属性无法拿到改变后的值的问题

```js
const result = computed(() => {
  console.log("执行了");
  return obj.name + obj.age;
});
console.log(result.value); //ggx20
obj.age++;
console.log(result.value); //ggx20
```

当我们访问了计算属性的值，打印出来为："ggx20"，更改 obj.age 的值，当再次访问计算属性的值的时候，发现打印出来依然为："ggx20"。预期结果为："ggx21"

问题分析：当第一次访问 result.value 的值之后，变量 dirty 会设置为 false, 代表不需要计算。即使我们修改了 obj.age, 但只要 dirty 的值为 false, 就不会重新计算，所以导致我们得到了错误的值。

解决方案：当值发生变化时，只要把 dirty 的值重置为 true 就可以了，因为 trigger 的触发，会导致 scheduler 的重新执行，我们可以利用这个时机，去把 dirty 设置为 true.

```js
function computed(callback) {
  let dirty = true;
  let value;
  const lazyEffect = effect(callback, {
    lazy: true,
    // 新增代码
    scheduler: () => {
      dirty = true;
    },
  });
  const obj = {
    get value() {
      if (!dirty) return value;
      dirty = false;
      value = lazyEffect();
      return value;
    },
  };
  return obj;
}
```

## 解决计算属性嵌套在 effect 中的问题

接着上面的代码:

```js
const result = computed(() => {
  console.log("执行了");
  return obj.name + obj.age;
});
effect(() => {
  console.log(result.value); //ggx20
});

obj.age++;
```

上面的代码将 computed 返回值放入副作用函数中，obj.age 值自增，预期是：会导致 result.value 的改变，从而导致副作用函数的重新执行，但是实际情况是：副作用函数并没有随着 result.value 的改变而重新执行。

问题分析：本质上是一个 effect 嵌套的问题，外层的 effect 无法收集计算属性的返回值的依赖，也就意味着无法在赋值的时候触发副作用函数的执行。

解决方案：当 computed 的返回值被访问的时候，手动触发针对该返回值的依赖收集，当 obj.age 值改变的时候，手动触发副作用函数的执行。

```js
function computed(getter) {
  let dirty = true;
  let value;
  const lazyEffect = effect(getter, {
    lazy: true,
    scheduler: () => {
      dirty = true;
      // 手动触发副作用函数的执行
      trigger(obj, "value");
    },
  });
  const obj = {
    get value() {
      if (!dirty) return value;
      dirty = false;
      value = lazyEffect();
      // 手动触发副作用函数的依赖收集
      track(obj, "value");
      return value;
    },
  };
  return obj;
}
const result = computed(() => {
  console.log("执行了");
  return obj.name + obj.age;
});
effect(() => {
  console.log(result.value); //ggx20
});
obj.age++;
```

# watch 实现和完善

从 vue3 的文档中可知，watch 这个 api 的用法主要有两种调用方式：

第一种：直接侦听对象

```js
watch(obj, () => {
  console.log("执行了", obj.age);
});
obj.age++;
```

第二种：侦听一个 getter

```js
watch(
  () => obj.age,
  () => {
    console.log("执行了", obj.age);
  }
);
obj.age++;
```

所谓 watch,其本质就是观测一个响应式数据，当数据发生变化时通知并执行相应的回调函数。
要想实现侦听的内容改变，触发回调函数的重新执行，依然会充分利用到 effect 的 scheduler 调度器的执行时机，粗略的实现如下：

```js
function watch(source, callback) {
  let getter;
  if (typeof source === "function") {
    getter = source;
  } else {
    getter = () => source;
  }
  effect(getter, {
    scheduler() {
      callback();
    },
  });
}
```

watch 的逻辑可以分为两步：

1. 触发针对 watch 第一个参数中数据的依赖收集，建立联系
2. getter 中的数据变化--->触发 scheduler 函数的执行 --> 触发 callback 执行回调

现在我们来实现第一步。考虑到 watch 第一个参数，可能是一个深层对象，我们需要读取一个对象上的任意属性，从而当任意属性发生变化时都能够触发回调函数的执行。因此我们需要一个方法去递归地读取对象属性，收集依赖。

```js
function traverse(value, seen = new Set()) {
  // 如果要读取的数据是原始值，或者已经被读取过，那么什么都不做
  if (typeof value !== "object" || value === null || seen.has(value)) return; // 将数据添加到seen中，代表遍历地读取过了，避免循环引用引起的死循环
  seen.add(value); // 暂不考虑数组等其他数据结构 // 假设value就是一个对象，使用 for in 读取对象的每一个属性，并递归地调用traverse进行处理
  for (const k in value) {
    traverse(value[k], seen);
  }
  return value;
}
```

将 watch 的第一个参数，用 traverse 包裹：

```js
function watch(source, callback) {
  let getter;
  // 如果source是函数，说明用户传递的是getter,所以直接把source赋值给getter
  if (typeof source === "function") {
    getter = source;
  } else {
    // 否则按照原来的实现调用traverse递归地读取
    getter = () => traverse(source);
  }
  effect(getter, {
    scheduler() {
      callback();
    },
  });
}
```

## 获取新旧值

我们注意到 watch 的回调函数中拿不到旧值和新值。通常我们在使用 Vue.js 中的 watch 函数时，能够在回调函数中得到变化前后的值

解决思路：这里需要充分利用 effect 的 lazy 选项，手动执行返回的副作用函数，拿到新值。

```js
function watch(source, callback) {
  let getter;
  if (typeof source === "function") {
    getter = source;
  } else {
    getter = () => traverse(source);
  }
  let oldVal;
  let newVal;
  const lazyEffect = effect(getter, {
    lazy: true,
    scheduler() {
      newVal = lazyEffect();
      callback(oldVal, newVal);
      oldVal = newVal;
    },
  });
  oldVal = lazyEffect();
}

watch(
  () => obj.age,
  (oldVal, newVal) => {
    console.log("执行了", obj.age); //打印：执行了，20
    console.log(oldVal, newVal); // 打印：20, 21
  }
);
obj.age++;
```

## 实现 immediate 特性

在 vue.js 中使用 watch 函数，会有第三个参数的选项，该参数选项中有三个选项：

1. deep
2. immediate
3. flush

下面我们实现 immediate 这个开发的特性：

在选项参数中指定  immediate: true  将立即以表达式的当前值触发回调：

因此上面的代码如果开启 immediate 选项，在 watch 创建的时候，会先执行 watch 的回调。实现起来非常简单：

```js
function watch(source, callback, option = {}) {
  let getter;
  if (typeof source === "function") {
    getter = source;
  } else {
    getter = () => traverse(source);
  }
  let oldVal;
  let newVal;
  // 提取scheduler调度函数为一个独立的job函数
  function job() {
    newVal = lazyEffect();
    callback(oldVal, newVal);
    oldVal = newVal;
  }
  const lazyEffect = effect(getter, {
    lazy: true,
    // 使用job函数作为调度器函数
    scheduler: job,
  });
  if (option?.immediate) {
    // 当immediate为true时立即执行job,从而触发回调执行
    job();
  } else {
    oldVal = lazyEffect();
  }
}
```

## 实现 flush 特性

除了指定回调函数为立即执行之外，还可以通过其他选项参数来指定回调函数的执行时机，例如在 Vue3.js 中使用 flush 选项指定：pre，post，sync。

> 默认值是  'pre'，指定的回调应该在渲染前被调用。它允许回调在模板运行前更新了其他值。'post'  值是可以用来将回调推迟到渲染之后的。如果回调需要通过  $refs  访问更新的 DOM 或子组件，那么则使用该值。

flush 本质上是指定调度函数的执行时机。前面讲过如何在为任务队列中执行调度函数 scheduler,这与 flush 的功能相同：

```js
function watch(source, callback, option = {}) {
  let getter;
  if (typeof source === "function") {
    getter = source;
  } else {
    getter = () => traverse(source);
  }
  let oldVal;
  let newVal;
  function job() {
    newVal = lazyEffect();
    callback(oldVal, newVal);
    oldVal = newVal;
  }
  const lazyEffect = effect(getter, {
    lazy: true,
    scheduler() {
      // 在调度函数中判断flush是否为‘post’，如果是，将其放到微任务队列中执行
      if (option?.flush === "post") {
        const p = Promise.resolve();
        p.then(job);
      } else {
        job();
      }
    },
  });
  if (option?.immediate) {
    job();
  } else {
    oldVal = lazyEffect();
  }
}
```

将 job 函数放到微任务队列中，从而实现异步延迟执行。

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
