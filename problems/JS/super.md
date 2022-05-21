# 为什么不能在 super 之前使用 this

来看一个经典的例子：

```js
class MyObject {
  constructor() {
    this.name = "ggx";
  }
}
class MyObjectEx extends MyObject {
  constructor() {
    console.log(this);
    super();
    this.age = 20;
  }
}
const myObjectInstance = new MyObjectEx();
```

我在 super 前打印了 this, 运行上面的代码，会报下面这段错误：

> Uncaught ReferenceError: Must call super constructor in derived class before accessing 'this' or returning from derived constructor

大概意思是说：在访问'this'或从派生构造函数返回之前必须调用派生类中的 super();

上面的派生类指：**MyObjectEx**。

这是一个经典的关于继承的问题，我的疑问是：为何在 super 之前不能访问 this, super 之后却可以访问呢？

声明一个类本质上就是声明一个构造函数，其基本语法为：

```js
class ClassName [extends ParentClass] {
    constructor(){
       console.log('这里是构造方法')
    }
}
```

而我们在将一般函数用作构造器时，函数体就是构造过程本身。而在使用 class 关键字时，该构造过程就被独立出来用特定的方法名来声明：constructor;

因此下面两段代码是类似的：

```js
class MyObject {
  constructor() {
    console.log("这里是构造方法");
  }
}
/***********与如下效果类似**************/
function MyObject() {
  console.log("这里是构造方法");
}
```

方法名 constructor 会告诉解释器在使用 new 操作符创建类的新实例时，应该调用这个函数，在这个函数内部，可以为新创建的实例（this）添加自有属性。而这些属性除了该 class 自身的，还包含所继承自父级 class 的属性和方法。那么我们是如何拿到父类的属性和方法的呢？

这些都要归功于 ES6 时代, 与 class 一起出现的关键字: super.

super 的出现填补了原型继承的不足：无法有效调用父类方法。但 super 的使用基于一个前提：即明确地知道类继承关系。例如利用 extends 来声明这种关系：

```js
class MyObject {
  constructor() {
    this.name = "ggx";
  }
  myMethods() {
    console.log(this.name);
  }
}
class MyObjectEx extends MyObject {
  exMethods() {
    super.myMethods(); //可以明确super指向MyObject类
  }
}
```

显然 super 关键字引用派生类的原型, 即：父类，我们可以通过 super 拿到父类的方法，也可以使用 super 调用父类的构造器拿到 this 实例:

```js
...
constructor() {
    super(); // 相当于 super.constructor()
}
...
```

而在继承关系中，派生类通过 super 关键字来调用父类的构造器（constructor）并将返回的实例赋值给 this。

我们知道，当通过 new 运算符对类进行实例化时候，将使用它的基类来构造实例。更准确的说，new 运算符回溯它的继承链并使用顶端的原生构造器来构造实例。举个例子：

```js
class OriginObject {
  constructor() {
    this.skill = "react";
  }
}
class MyObject extends OriginObject {
  constructor() {
    super();
    this.name = "ggx";
  }
}
class MyObjectEx extends MyObject {
  // 这里会隐式的加上
  // constructor() {
  //super();
  //}
}
```

当执行 new MyObjectEx();时 this 的创建过程顺序如下:

```js
var thisObj = new Object();
OriginObject.call(thisObj);
MyObject.call(thisObj);
MyObjectEx.call(thisObj);
```

当 new 的时候,会先调用 class 的 constructor, 从而执行内部的 super 关键字, 当执行 super 时, 又会调用父类的 constructor,又会执行父类的 super 关键字, 最终让所有构造方法都先调用 super()以回溯整个原型链,才能确保基类(MyObject\OriginObject\Object)最先创建实例.

> 这也是没有在类中声明 constructor()方法时,Javascript 会为它默认添加一个构造方法,并且在其中调用 super()的原因.更确切地说: 如果子类是派生的,那么它就是必须在 constructor()中调用 super, 而非派生类就不能调用它.

因此要想出现 super()之前不能访问 this 这样的报错,我们需要有两个前提条件:

1.  使用 new 运算符构造实例
2.  有 extends 继承关系

因为 new 运算符的调用才会导致让 super 回溯整个原型链, 在还没初始化完父类的实例之前, 我们在派生类中的 super 之前访问 this 是访问不了的.

本质上**继承**的目的就是为了复用父类的代码, 因此我们要**先**拿到父类构造函数的实例, 才能愉快的用到子类的 this 中. 而这个"先"的动作,就是使用 super 关键字.

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
