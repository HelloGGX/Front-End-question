本文的一些伪代码，都是基于我的上一篇文章《vue3 如何监听每个属性的变化》的实现来的，因此，建议阅读之前先了解 vue3 的依赖收集原理，方便理解。

# 1. 理解对象

在 Javascript 中一切皆对象，那么到底什么是对象呢？查阅 ECMAScript 规范（基于 ECMA-262）在 4.4.10 中，会看到这样一段话：

> NOTE Any object that is not an ordinary object is an exotic object.

**翻译为：任何不属于常规对象的对象都是异质对象**

我们可知在 JavaScript 中有两种对象：

1. 常规对象（ordinary object）
2. 异质对象（exotic object）

常规对象：该对象具有所有对象必须支持的基本内部方法的默认行为。
异质对象：如果不完全具备所有对象拥有的基本内部方法，就是异质对象。JS 里的对象不是普通对象就是异质对象。

在 ES6 时代的规范中，针对对象的划分其实变成了四种，除了我们前面说的两种之外，还有两种对象：

3. 标准对象（standard object）: ECMAScript6 中定义的对象， 比如：Array、Date，**标准对象均为内置对象**,
4. 内置对象（build-in object）: 在引擎初始化阶段就被创建，不需要 New,每个内置对象都是原生对象，内置对象是原生对象的子集，比如：Global、Math、JSON、

**以上两种对象均在常规对象和异质对象的范围内。**

以上 4 种对象是根据对象的内部方法作为条件划分的。所谓内部方法，指的是当我们对一个对象进行操作时在引擎内部调用的方法，这些方法对于 Javascript 使用者来说是不可见的。举个例子，当我们访问对象属性时：

```js
obj.foo;
```

引擎内部会调用[[Get]]这个内部方法来读取属性，在 ECMAScript 规范中使用[[xxx]]来代表内部方法或者内部槽。

我们假设 **O 为普通对象，P 为属性键值，V 为任意 ECMAScript 语言值，而 Desc 是一个属性描述符记录**，那么对象必要的内部方法如下：

1. [[GetPrototypeOf]] ( )
2. [[SetPrototypeOf]] ( V )
3. [[IsExtensible]] ( )
4. [[PreventExtensions]] ( )
5. [[GetOwnProperty]] ( P )
6. [[DefineOwnProperty]] ( P, Desc )
7. [[HasProperty]] ( P )
8. [[Get]] ( P, Receiver )
9. [[Set]] ( P, V, Receiver )
10. [[Delete]] ( P )
11. [[OwnPropertyKeys]] ( )

以上内部方法均使用 ECMA 规范文档 10.1.X 节给出的定义实现。

在 JavaScript 中函数是可调用的对象，在 ECMAScript 标准中称为：**ECMAScript Function Objects**

函数对象也是一个普通对象，具有与其他普通对象相同的内部槽和内部方法，除此之外它还定义了两个额外的内部方法：

1. [[Call]] ( thisArgument, argumentsList )
2. [[Construct]] ( argumentsList, newTarget )

也就是说，如果一个对象需要作为函数调用，那么这个对象就必须部署内部方法[[Call]]。

因此我们可以区分什么是普通对象，什么是异质对象

| 对象类型 | 满足条件                                                                                                                                                                                             |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 普通对象 | 1. 对于必要的内部方法，必须使用 ECMA 规范 10.1.x 节给出的定义实现 2. 对于内部方法[[Call]] 必须使用 ECMA 规范 10.2.1 节给出的规范实现 3. 对于[[Construct]],必须使用 ECMA 规范 10.2.2 节给出的定义实现 |
| 异质对象 | 不符合以上三点要求的都是异质对象                                                                                                                                                                     |

打个比方：Proxy 和常规的对象内部都有[[Get]]方法，但是 Proxy 内部的[[Get]]方法没有使用 ECMA 规范 10.1.8 节给出的定义实现，因此 Proxy 是一个异质对象

# 2. 普通对象与数组的区别

根据上文我们针对对象的详细区分，那么数组是什么对象呢？
我们可以看到数组的内部方法 [[DefineOwnProperty]]并不是由 10.1.6 节给出的定义实现的，而是由 10.4.2.1 节给出的定义实现，因此数组是一个异质对象。数组的其他内部方法基于 **ArrayCreate ( length [ , proto ] )** 方法创建，它的内部实现如下：

```js
1. If length > 2^32 - 1, throw a RangeError exception.
2. If proto is not present, set proto to %Array.prototype%.
3. Let A be ! MakeBasicObject(« [[Prototype]], [[Extensible]] »).
4. Set A.[[Prototype]] to proto.
5. Set A.[[DefineOwnProperty]] as specified in 10.4.2.1.
6. Perform ! OrdinaryDefineOwnProperty(A, "length", PropertyDescriptor { [[Value]]: 픽(length), [[Writable]]: true, [[Enumerable]]: false, [[Configurable]]: false }).
7. Return A.
```

我们可以看到第三步将对象的必要内部方法作为数组的[[Prototype]]，第五步单独设置 10.4.2.1 节定义的[[DefineOwnProperty]]作为实现，因此除了 [[DefineOwnProperty]]不一致，其他内部方法与普通对象的内部方法的逻辑保持一致。

因此我们说普通对象与数组最本质的区别是：内部方法 [[DefineOwnProperty]]的实现不一致。（具体逻辑区别对比 ECMA-262 中 10.1.6 与 10.4.2.1 的逻辑）

普通对象 O 的[[DefineOwnProperty]]内部方法接受参数 P，其实它的内部直接返回了 OrdinaryDefineOwnProperty

```
Return ? OrdinaryDefineOwnProperty(O, P, Desc).
```

我们可以看到普通对象的内部实现非常简单，直接返回 OrdinaryDefineOwnProperty 方法的执行结果。
而 OrdinaryDefineOwnProperty 接收参数 O（一个对象）、P（一个属性键）和 Desc（一个 Property Descriptor），它被调用时执行以下步骤：

```
1.  Let current be ? O.[[GetOwnProperty]](P).
2.  Let extensible be ? IsExtensible(O).
3.  Return ValidateAndApplyPropertyDescriptor(O, P, extensible, Desc, current)
```

可以看到普通对象的[[DefineOwnProperty]]内部方法会调用 GetOwnProperty 获取对象属性，并检查是否允许该对象添加其他属性之后，实际会执行 ValidateAndApplyPropertyDescriptor 这个抽象操作，其中的逻辑我们不必深究，感兴趣的同学可以查阅 10.1.6.3 小结。

而我们的数组的的内部方法[[DefineOwnProperty]]实现如下：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/vue-对象的代理细节1.png"/>

我们可以看到数组的内部方法[[DefineOwnProperty]]是基于普通对象的 OrdinaryDefineOwnProperty(O, P, Desc).方法实现的，只不过 P 换成了数组的 length 属性和索引。

**基于上面的陈述，我们可以说，针对普通对象的代理，其实基本都可以用到数组上来，因为它们的内部方法基本一样。**

# 3. 普通对象的代理细节

### 3.1 代理的细节

VUE3 的代理方式是通过 Proxy 去拦截对应的操作，并返回一个代理对象

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/vue-对象的代理细节1.png2"/>

例如以上代码，当我们访问 proxyObj 的属性时，引擎会调用部署在代理对象上的内部方法[[Get]], 而这个内部方法会调用对应的处理函数，也就是我们目前定义的 get 处理函数，但是当我们没有写 get 处理函数时，这个代理对象的内部方法[[Get]]会调用目标对象的内部方法[[Get]]来获取属性值。

**总结来说在代理对象上调用一个内部方法会导致调用对应的处理函数，如果没有写对应的处理程序，那么目标对象上相应的内部方法就会被调用。**

那么代理对象的内部方法有哪些呢？

表 3.1.1
| 内部方法 | 对应的处理函数 |
| --- | --- |
| [[GetPrototypeOf]] | getPrototypeOf |
| [[SetPrototypeOf]] | setPrototypeOf |
| [[IsExtensible]] | isExtensible |
| [[PreventExtensions]] | preventExtensions |
| [[GetOwnProperty]] | getOwnPropertyDescriptor |
| [[DefineOwnProperty]] | defineProperty |
| [[HasProperty]] | has |
| [[Get]] | get |
| [[Set]] | set |
| [[Delete]] | deleteProperty |
| [[OwnPropertyKeys]] | ownKeys |
| [[Call]] | apply |
| [[Construct]] | construct |

后面的内容都是围绕这个表来进行的，我们会根据内部方法，去查找对应的处理函数。

首先我们先讨论普通对象的读取操作，在响应式系统中，“读取”是一个很宽泛的概念，例如：使用 in 操作符检查对象上是否具有给定的 key 也属于读取操作，for...in, Object.keys 都是“读取”操作。

下面列出了一个普通对象所有可能的**读取**操作：

1. 访问属性：obj.foo
2. 判断对象或原型上是否存在给定义的 key: key in obj
3. 使用 for...in 循环遍历对象
4. 读取对象所有 keys: Object.keys
5. 读取对象所有值：Object.values

### 3.2 访问属性 obj.xxx 的拦截

首先对于属性的读取，例如 obj.foo 我们可以知道这可以通过 get 拦截操作

```js
const p = new Proxy(obj, {
  // 拦截读取操作
  get(target, key) {
    // 收集属性对应的副作用依赖
    track(target, key); // 返回属性值
    return Reflect.get(target, key, receiver);
  },
});
```

### 3.3 in 操作符的拦截

在 ECMA-262 规范的 13.10.1 节中，明确定义了 in 操作符的运行时逻辑：

```js
1. Let lref be the result of evaluating RelationalExpression.
2. Let lval be ? GetValue(lref).
3. Let rref be the result of evaluating ShiftExpression.
4. Let rval be ? GetValue(rref).
5. If Type(rval) is not Object, throw a TypeError exception.
6. Return ? HasProperty(rval, ? ToPropertyKey(lval)).
```

我们可以看到第 6 步，in 操作符的运算结果是通过调用一个 HasProperty 的抽象方法得到的，我们查看 HasProperty 的逻辑，在 ECMA-262 规范的 7.3.11 节中找到：

```js
1. Assert: Type(O) is Object.
2. Assert: IsPropertyKey(P) is true.
3. Return ? O.[[HasProperty]](P).
```

可以看到 HasProperty 抽象方法的返回值是通过调用对象的内部方法 **[[HasProperty]]** 。而[[HasProperty]]内部方法对应的拦截函数从表 3.1.1 中可以查到：has

因此针对 in 操作符的拦截为：

```js
new Proxy(obj, {
  has(target, key) {
    track(target, key);
    return Reflect.has(target, key);
  },
});
```

### 3.4 针对 for...in、Object.keys、Object.values 的代理

for...in、Object.keys、Object.values 这三个操作，通过查阅 ECMA-262 规范可知，其实它们三个的内部实现，都会调用抽象方法

```js
EnumerateObjectProperties(obj).
```

以下是该方法的内部实现：

```js
function* EnumerateObjectProperties(obj) {
  const visited = new Set();
  for (const key of Reflect.ownKeys(obj)) {
    if (typeof key === "symbol") continue;
    const desc = Reflect.getOwnPropertyDescriptor(obj, key);
    if (desc) {
      visited.add(key);
      if (desc.enumerable) yield key;
    }
  }
  const proto = Reflect.getPrototypeOf(obj);
  if (proto === null) return;
  for (const protoKey of EnumerateObjectProperties(proto)) {
    if (!visited.has(protoKey)) yield protoKey;
  }
}
```

可以看到该方法是一个 generator 函数，接收一个参数 obj。实际上，obj 就是被 for...in 循环遍历的对象，其关键点是在于使用 Reflect.ownKeys(obj)来获取只属于对象自身拥有的键。因为 Reflect 和 Proxy 都有相同的方法对应，所以我们可以使用 ownKeys 来拦截 Reflet.ownKeys 操作：

```js
new Proxy(obj, {
  ownKeys(target) {
    return Reflect.ownKeys(target);
  },
});
```

### 3.5. 删除对象属性的代理

当我们调用 delete proxyObj.xxx 时，会调用内部方法[[Delete]],与之对应的处理函数为：deleteProperty

```js
new Proxy(obj, { 
      deleteProperty(target, key) {
        ...
      }
 })
```

# 4. 数组的代理细节

在第二小节我们可以知道，其实普通对象和数组除了内部方法：[[DefineOwnProperty]] 不一样之外，其他内部方法的逻辑都是一样的，因此针对数组的代理，其实代理普通对象的的大部分代码都已经实现。

因此当我们访问 arr[0],或者给 arr[0]赋值，都可以正确的触发 get/set 的拦截。

但是对数组的操作与普通对象仍然存在不同，下面是对数组元素或属性的“读取”操作：

1. 通过索引访问数组的元素值：arr[0]
2. 访问数组的长度：arr.length
3. 使用 for...in 遍历数组
4. 使用 for...of 遍历数组
5. 数组的原型方法，如：concat/join/every/some/find/findIndex/includes 等，以及其他所有不改变原数组的原型方法

可以看到对数组的读取操作要比普通对象丰富的多。我们再来看看对数组元素或属性的设置操作：

1. 通过索引修改数组元素值：arr[1] = 3
2. 修改数组的长度 arr.length = 0
3. 数组栈方法：push/pop/shift/unshift
4. 修改原数组的原型方法：splice/fill/sort

那么 vue 是如何正确的响应数组的读取和设置操作呢？

```js
//对数组['foo']建立响应，返回代理数组arr
const arr = reactive(["foo"]);
// 副作用函数可以理解为render
effect(() => {
  console.log(arr.length);
});
arr[1] = "bar";
```

以上代码设置数组的索引 1 的值，从表现来看：会导致数组的长度变为 2，

实际上，当我们通过索引设置数组元素值时，会执行数组对象内部方法[[Set]],而根据规范，内部方法[[Set]]其实依赖于[[DefineOwnProperty]],而数组的内部方法[[DefineOwnProperty]]内部实现中提到：

**如果设置的索引值大于数组当前的长度，那么需要更新数组的 length 属性。所以当通过索引设置元素值时，可能会隐式地修改 length 的属性值**。因此会触发与 length 属性相关联的副作用函数的重新执行。

因此在 Vue3 中，我们需要判断代理目标是否是数组，当检测到被设置的索引值小于数组长度时，那么为 set 操作，大于数组长度时，为 add 操作。

```js
// 拦截设置操作
    set(target, key, newVal, receiver) {
     ...
      // 如果属性不存在，则说明是在添加新的属性，否则是设置已存在的属性
      const type = Array.isArray(target)
        ? Number(key) < target.length ? 'SET' : 'ADD'
        : Object.prototype.hasOwnProperty.call(target, key) ? 'SET' : 'ADD'
      // 设置属性值
      const res = Reflect.set(target, key, newVal, receiver)
     trigger(target, key, type)
      return res
    },
```

而 trigger 函数内部根据代理对象是否是数组和操作的类型，来取出 length 相关的副作用依赖执行

```js
function trigger(target, key, type, newVal) {
  const depsMap = bucket.get(target)
  if (!depsMap) return
  const effects = depsMap.get(key)
  const effectsToRun = new Set()

  effects && effects.forEach(effectFn => {
    if (effectFn !== activeEffect) {
      effectsToRun.add(effectFn)
    }
  })
 // 当操作类型为ADD并且目标对象是数组时，应该取出并执行那些与length属性相关的副作用函数，放入待执行队列
  if (type === 'ADD' && Array.isArray(target)) {
    const lengthEffects = depsMap.get('length')
    lengthEffects && lengthEffects.forEach(effectFn => {
      if (effectFn !== activeEffect) {
        effectsToRun.add(effectFn)
      }
    })
  }

  effectsToRun.forEach(effectFn => {
    if (effectFn.options.scheduler) {
      effectFn.options.scheduler(effectFn)
    } else {
      effectFn()
    }
  }
```

那么反过来，修改数组的 length 属性也会隐式的影响数组元素,比如：

```js
//对数组['foo']建立响应，返回代理数组arr
const arr = reactive(["foo"]);
// 副作用函数可以理解为render
effect(() => {
  console.log(arr[0]);
});
arr.length = 0;
```

将数组长度修改为 0，会导致数组元素被删除，因此应该触发页面的重新渲染，也就是这里的 effect 函数内部的回调。

那我们如何解决呢？上面的问题我们可以总结为：当修改 length 属性时，只有那些索引值大于或等于新的 length 值的元素才会受影响，才应该触发页面的重新渲染。

因此我们需要给触发函数 trigger 传入新的 length 的值作为判断依据:

```js
function trigger(target, key, type, length) {
  ...
  if (Array.isArray(target) && key === 'length') {
    depsMap.forEach((effects, key) => {
    // 对于索引大于或等于的新length的值的元素，
    // 需要把所有相关联的副作用函数取出来添加到待执行队列中
      if (key >= length) {
        effects.forEach(effectFn => {
          if (effectFn !== activeEffect) {
            effectsToRun.add(effectFn)
          }
        })
      }
    })
  }
 ...
}
```

## 4.1 遍历数组的代理细节

### 4.1.1 for...in 遍历数组

数组也是对象，也就意味着同样可以使用 for...in 循环遍历数组：

```js
const arr = reactive(["foo"]);
effect(() => {
  for (const key in arr) {
    console.log(key);
  }
});
```

因此跟对象一样，我们依然可以通过 ownKeys 拦截。当数组的 length 发生改变，也就意味着会影响 for...in 的循环次数，进而影响渲染结果，那么在 vue3 中，针对数组 length 的改变，vue3 会去收集 length 属性的副作用依赖，当 length 属性改变时，触发 trigger 函数, 取出 length 的副作用依赖依次执行。

因此，针对 for...in 的拦截中，我们需要去判断目标对象如果是数组，那么我们就要收集 length 作为依赖，供后续执行。

```js
new Proxy(obj, {
  ownKeys(target) {
    // 判断目标对象是数组的话，就收集length属性的依赖
    track(target, Array.isArray(target) ? "length" : ITERATE_KEY);
    return Reflect.ownKeys(target);
  },
});
```

### 4.1.2 for...of 遍历数组

for...of 是用来遍历可迭代对象的，如果一个对象内部实现了 Symbol.iterator 方法，那么这个对象是可以迭代的，数组内建了 Symbol.iterator 方法的实现。我们查阅 ECMA-262 规范中，关于数组可迭代对象的逻辑实现。在 23.1.5.1 小节 CreateArrayIterator 方法的实现中可以知道：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/vue-对象的代理细节3.png"/>

我们可以从上面的逻辑知道，for...of 遍历数组会访问数组的 length 属性和数组的索引，因此我们可以通过数组的 length 属性和索引与副作用函数建立响应联系，就可以实现响应式的 for...of 迭代。

显然，之前几个小节中我们已经讨论了针对 length 和索引的相关处理。

这里不得不提一点的是，数组的 values\keys 方法返回的值实际上就是数组的内建的迭代器，从 ECMA-262 规范，23.1.3.32 和 23.1.3.16 小结中，我们可以找到答案：

```js
// 23.1.3.16 Array.prototype.keys ( )
1. Let O be ? ToObject(this value).
2. Return CreateArrayIterator(O, key).

// 23.1.3.32 Array.prototype.values ( )
1. Let O be ? ToObject(this value).
2. Return CreateArrayIterator(O, value)
```

我们可以看到它们都返回了抽象方法 CreateArrayIterator 的执行结果，因此无论我们使用 for...of 循环，还是调用 values，keys 等方法，它们都会读取数组的 Symbol.iterator 属性。

## 4.2 数组的查找方法

针对数组的查找，我们知道一般有四种操作：

1. includes
2. find
3. indexOf
4. lastIndexOf

例如：当执行 arr.includes 方法时，实际上会去先访问 includes 这个属性，进而触发 get 拦截操作，收集 includes 属性相关的依赖。在 vue3 的 get 的拦截中，会重写这些方法的逻辑，indexOf, lastIndexOf 同理，伪代码如下：

```js
const arrayInstrumentations = {}
;['includes', 'indexOf', 'lastIndexOf'].forEach(method => {
  const originMethod = Array.prototype[method]
  arrayInstrumentations[method] = function(...args) {
    ...
  }
})

return new Proxy(obj, {
    // 拦截读取操作
    get(target, key, receiver) {
    ...
    // 拦截'includes', 'indexOf', 'lastIndexOf'等操作
      if (Array.isArray(target) && arrayInstrumentations.hasOwnProperty(key)) {
        return Reflect.get(arrayInstrumentations, key, receiver)
      }
      ...
      return res
    },

  })
```

## 4.2.1 includes 需要解决的问题

```js
const obj = {};
const arr = reactive([obj, "ss"]);
console.log(arr.includes(arr[0]));
```

上面的代码应该打印 true，我们来看看 **arr.includes(arr[0])** 的属性访问顺序：

```js
1、依赖收集第一个 arr的 includes,
2、依赖收集第二个arr的 0 索引 检测到是对象，执行reactive(该对象)返回 new Proxy
3、依赖收集第一个arr的 length
4、依赖收集第一个arr的 0 索引 监测到是对象，执行reactive(该对象)返回 new Proxy
5、依赖收集第一个arr的 1 索引
```

我们可以看到第 2 步和第 4 步，其实都是返回了新的 Proxy 实例, 其实本来不应该相等的。那么 vue 是如何处理的呢？

reactive 的函数实现如下：

```js
const reactiveMap = new WeakMap();
function reactive(obj) {
  const proxy = createReactive(obj);
  const objReceiver = reactiveMap.get(obj);
  if (objReceiver) return objReceiver;
  reactiveMap.set(obj, proxy);
  return proxy;
}
```

我们可以看到，当检测到数组的元素依然是一个对象时，说明它应该也是一个响应式的对象，因此又会通过 reactive 方法再次创建一个响应式对象，关键点就在这里，在返回响应式对象之前，我们创建一个 WeakMap 集合去收集原始对象和响应式对象的依赖关系，当发现之前已经收集过某个原始对象的映射时，那么就直接返回该对象对映的代理对象，不去返回 new Proxy。

这样就避免了上面的 demo 可能存在的问题。

## 4.3 隐式修改数组长度的原型方法

这里主要指的是数组的栈方法，比如：push/pop/shift/unshift。除此之外 splice 方法也会隐式的修改数组的长度。

基于我之前的文章《vue3 如何监听每个属性的变化》的代码。我们看看如下场景：

```js
// 针对空数组返回proxy代理数组对象
const arr = reactive([]);

// 第一个副作用函数
effect(() => {
  arr.push(1);
});

//第二个副作用函数
effect(() => {
  arr.push(1);
});
```

上面的代码会报栈溢出的错误，我们分析一下执行过程：

1. 首先第一个副作用函数执行，执行 arr.push 会导致针对 push 这个属性的依赖收集，从 ECMA-262 规范的 23.1.3.20 Array.prototype.push 小结，我们可知，push 的执行也会读取数组的 length 属性，因此也会对 length 进行依赖收集。
2. 接着第二个函数执行，同样，它也会读取 length 属性建立响应联系。但不要忘了，调用 arr.push 方法不仅会间接读取数组的 length 属性，还会设置数组的 length 属性的值
3. 第二个函数内的 arr.push 方法的调用设置了 length 属性的值，于是响应式系统会尝试把与 length 属性相关联的副作用依赖全部取出执行，其中就包括第一个副作用函数，问题就出现在这里，可以发现，第二个副作用函数还未执行完毕，就要执行第一个副作用函数
4. 第一个副作用函数再次执行，又会设置 length 属性，响应式系统又会把与 length 有关的副作用依赖全部取出执行，导致第一个还没执行完毕，就要执行第二个。
5. 如此循环往复，最终导致调用栈溢出。

那么 vue 如何解决这样的问题呢？跟我们之前拦截查找数组的方法一样，我们需要针对 push 方法进行拦截重写：

接着 4.2 小节的代码，我们利用 arrayInstrumentations 对象：

```js
  const arrayInstrumentations = {};

    ;['includes', 'indexOf', 'lastIndexOf'].forEach(method => {
      const originMethod = Array.prototype[method]
      arrayInstrumentations[method] = function (...args) {
      ...
        return res
      }
    })
 

    ;['push'].forEach(method => {
      const originMethod = Array.prototype[method]
      arrayInstrumentations[method] = function (...args) {
      ...
        return res
      }
    })

return new Proxy(obj, {
    // 拦截读取操作
    get(target, key, receiver) {
    ...
    // 拦截'includes', 'indexOf', 'lastIndexOf', push等操作
      if (Array.isArray(target) && arrayInstrumentations.hasOwnProperty(key)) {
        return Reflect.get(arrayInstrumentations, key, receiver)
      }
      ...
      return res
    },

  })
```

问题的原因是 push 方法的调用会间接读取 length 属性。因此我们需要屏蔽针对 length 属性的收集，因为数组的 push 方法在语义上是修改操作，不是读取操作。因此我们设置一个标记变量，代表是否需要收集。默认为 true，表示允许收集

```js
let shouldTrack = true;
["push"].forEach((method) => {
  const originMethod = Array.prototype[method];
  arrayInstrumentations[method] = function (...args) {
    // 在执行数组的方法之前不允许收集
    shouldTrack = false;
    let res = originMethod.apply(this, args);
    // 在执行数组的方法之后允许收集
    shouldTrack = true;
    return res;
  };
});
```

于是我们可以根据这个 shouldTrack 变量，判断是否需要进行依赖收集：

```js
function track(target, key) {
  if (!activeEffect || !shouldTrack) return
 ...
}
```

那么在调用 originMethod.apply(this, args)时，因为 shouldTrack 为 false,那么就不会对 length 属性进行依赖收集，也就不会触发 length 相关依赖的执行。

---

# 关于作者

大家好，我是程序员高翔的猫, 用户体验部前端研发。

**加我的微信，备注：「个人简单介绍」+「组队讨论前端」**， 拉你进群，每周两道前端讨论分析，题目是由浅入深的，把复杂的东西讲简单的，把枯燥的知识讲有趣的，实用的，深入的，系统的前端知识。

<a name="微信"></a>
<img width="400" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/weixin.jpg"/>

# 我的公众号

更多精彩文章持续更新，微信搜索：「高翔的猫」第一时间围观

<a name="公众号"></a>

<img src="https://mp.weixin.qq.com/mp/qrcode?scene=10000004&size=102&__biz=MzkxNjMxNDU0MQ==&mid=2247483692&idx=1&sn=2d2baccebfd92fbf6d0506d3c75b3ade&send_time=" data-img="1" width="200" height="200">
