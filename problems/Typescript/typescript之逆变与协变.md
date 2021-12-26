# Typescript之逆变与协变

## 协变、逆变、双向协变的概念
在Typescript文档中， 我们经常会遇见两个单词：co-variant 和 contra-variant ，分别表示协变和逆变。

可以看到他们都有一个共同的单词 variant, 其名词形式为：variance，那么什么是variance?Typescript想要表达什么？

从本质来讲，variance是一种描述子类型之间关系的概念， 具体为当两个通用类型作为参数，那么其子类型的关系是如何的描述。

假设我们有两种类型，分别为`Animal`和`Dog`, `Dog`是`Animal`的子类型， 还有一个泛型类型List, 接收一个类型参数来分别实例化Animal和Dog: `List<Animal>`、`List<Dog>`，那么`List<Animal>`和`List<Dog>`之间的关系是怎么的呢？

<img width="500" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/variance.png"/>

### 1、协变概念(Covariant)

当Dog是Animal的子类型，那么`List<Dog>`是`List<Animal>`的子类型，我们称其为类型协变，箭头保持一致：

<img width="500" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/co-variant.png"/>


```typescript
let dogList: List<Dog> = v as List<Animal> //报错
let animalList: List<Animal> = v as List<Dog> //类型安全
```

### 2、逆变概念(Contravariant)

当Dog是Animal的子类型，那么`List<Animal>`是`List<Dog>`的子类型，我们称其为类型逆变，箭头相反：

<img width="500" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/contra-variant.png"/>

```typescript
let dogList: List<Dog> = v as List<Animal> //类型安全
let animalList: List<Animal> = v as List<Dog> //报错
```

### 2、双向协变概念（Bivariant）

当Dog是Animal的子类型，那么`List<Animal>`是`List<Dog>`的子类型, `List<Dog>`也是`List<Animal>`的子类型, 我们称为类型双变，箭头双向：

<img width="500" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/varicnce.png"/>

```typescript
let dogList: List<Dog> = v as List<Animal> //类型安全
let animalList: List<Animal> = v as List<Dog> //类型安全
```

## 深入Variance

以上介绍了协变、逆变、双变的概念之后，我们由此引出了另外的问题: 
- 什么情况下是协变的，什么情况下是逆变的？
- 该特性的意义在哪里？

首先我们需要了解数学中子集的概念：(取自维基百科)

> In mathematics, set A is a subset of a set B if all elements of A are also elements of B; B is then a superset of A. It is possible for A and B to be equal; if they are unequal, then A is a proper subset of B. The relationship of one set being a subset of another is called inclusion (or sometimes containment). A is a subset of B may also be expressed as B includes (or contains) A or A is included (or contained) in B.

由概念可知，当A是B的子类，我们说A中的每个实例在某种程度上都是B的实例，即使子类A中扩展了超类B更多的属性，而超类B中的实例不都是子类A中的实例，子类A定义了内圈，超类B定义了外圈。

更进一步说每个属于子类型的元素其实也是属于超类型的，而不是每个属于超类型的元素都是属于子类型的，举个实际的例子：

考虑一个超类型的动物和一个子类型的猫狗，我们可以肯定的说，每只猫都是动物，但不是每只动物都是猫。我们可以把每只猫放在内圈，其余的动物放在外圈（与猫分开）。我们也可以说 "外圈的每个元素都是动物，内圈的每个元素都是猫。还有一些外圈的元素（动物）不在内圈（不是猫）比如狗。

<img width="700" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/A-B.png"/>

### 1、协变 Covariant

我们根据上图定义三个基础类：

```typescript
class Animal {
  public weight: number = 0;
}

class Dog extends Animal {
 public isGoodBoy: boolean = true
 public bark() { console.log("Wolf!") }
}

class Cat extends Animal {
  public Muia: "坨子"
  public play() { console.log("Miau!") }
}
```

接着我们定义泛型接口

```typescript
interface Cage<T> {
    readonly animal: T
}
```
通过该接口我们可以定义猫狗基类的实例

```typescript
let dogCage: Cage<Dog> = { animal: new Dog() }

let cage: Cage<Animal>

let catCage: Cage<Cat> 
```

当我们将dogCage赋值给cage变量是安全的，表现为**类型收敛**，但是我们把Animal赋值给catCage就会报错

```typescript
let dogCage: Cage<Dog> = { animal: new Dog() }

let cage: Cage<Animal> = dogCage; //安全

let catCage: Cage<Cat> = cage; // 报错：类型“Animal”缺少类型“Cat”中的以下属性: Muia, play
```
通过上面的圆圈图类比就是：外圈Animal的元素比内圈Dog的元素多得多，所以当我们期待外圈的某个元素时，我们可以通过内圈的任何元素来满足需求，相反当我们期待内圈的某个元素时，外圈的元素类型可能满足不了，表现为类型不安全。

由上可知：接口Cage的泛型**赋值**的用法，让Dog是Animal的子类型可以得出 `Cage<Dog>`是 `Cage<Animal>`的子类型，表现为**类型协变**，

紧接着上面的例子，我们定义三个包含函数返回的对象：

```typescript
interface Mother<T> { create(): T }

let dogMother: Mother<Dog> = { create: ()=> new Dog() }

let animalMother: Mother<Animal>;

let catMother: Mother<Cat>;
```
当我们dogMother赋值给animalMother时，类型安全，表现为**类型收敛**，因为animalMother的类型只能访问weight:

```typescript
let animalMother: Mother<Animal> = dogMother;

animalMother.create().weight
```
但当把animalMother赋值给catMother就会报错：类型“Animal”缺少类型“Cat”中的以下属性: Muia, play；因为根据定义，catMother可能访问play,这是animalMother不具有的属性：

```typescript
let animalMother: Mother<Animal>;

let catMother: Mother<Cat> = animalMother; //报错
```
协变的结论：

- 将一个变量赋给另一个变量时表现为协变
- 将一个函数赋给另一个函数变量时，函数返回值类型表现为协变


### 2、逆变 Contravariant

接下来再看一个例子：

```typescript
type getWeight<T> = (animal: T) => string;

const weightAnimals: getWeight<Animal> = ({weight}) => weight.toString()
const weightCats: getWeight<Cat> =  ({Muia, play, weight }) => Muia

const expectweightAnimals: getWeight<Animal> = weightCats;  // 报错

const expectWeightCats: getWeight<Cat> = weightAnimals;     // 类型安全
```

上面的例子可能有点反直觉，怎么理解呢？

我们知道，函数接收输入，执行操作，返回输出, 而对于一个具体的类型Cat作为参数，函数不仅仅可以把它当成Animal，来执行一些操作；还可以访问其作为Cat独有的一些属性和方法，来执行另一部分操作。因此`getWeight<Cat>`的操作可能性肯定比`getWeight<Animal>`要多。

更进一步说：为一个函数提供的形参属性越多，函数可以使用的可能操作就越多，我们可以考虑的总体可能的函数就越多。

还是拿上面的例子：函数expectweightAnimals期望的是拿到形参属性weight的值，但是赋值的函数weightCats由于能使用的参数很多，因此操作的可能性就很多，可能只需要用到属性Muia，并不会用到属性weight,因此类型不安全。

但是反过来，函数expectWeightCats的形参属性很多，因此拿这些属性操作的可能性就有很多，而赋值的函数weightAnimals只需要其中一种操作：拿到其属性weight，函数expectWeightCats的形参属性完全覆盖，类型安全


<img width="600" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/cotr.png"/>

由上可以得出：**Cat是Animal的子类型，但是`getWeight<Cat>`却是`getWeight<Animal>`的父类型，表现为类型逆变**

逆变的结论：
- 函数类型中，参数是逆变的


## 协变、逆变概念的意义

- 我们根据基础类型的位置以及关系，来判断出复杂类型的关系
- 从前面的圆圈图中，我们可以得出一个共性规律：协变、逆变的本质都是让类型收敛，缩小类型范围，保证类型安全

## 简短总结：

- 变量赋值和函数返回值类型表现为协变
- 函数类型中，参数类型表现为逆变
- 本质都是让类型收敛，缩小类型范围，保证类型安全


# 关于作者

大家好，我是程序员高翔的猫，MonkeyDesign用户体验部前端研发。

**加我的微信，备注：「个人简单介绍」+「组队讨论前端」**， 拉你进群，每周两道前端讨论分析，题目是由浅入深的，把复杂的东西讲简单的，把枯燥的知识讲有趣的，实用的，深入的，系统的前端知识。


<a name="微信"></a>
<img width="400" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/weixin.jpg"/>

# 我的公众号

更多精彩文章持续更新，微信搜索：「高翔的猫」第一时间围观


<a name="公众号"></a>

<img src="https://mp.weixin.qq.com/mp/qrcode?scene=10000004&size=102&__biz=MzkxNjMxNDU0MQ==&mid=2247483692&idx=1&sn=2d2baccebfd92fbf6d0506d3c75b3ade&send_time=" data-img="1" width="200" height="200">