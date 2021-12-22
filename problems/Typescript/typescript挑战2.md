<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/typescript-banner.png"/>

# 1. Typescript挑战-2 <img src="https://img.shields.io/badge/typescript-%E4%B8%AD%E7%BA%A7-blue"/>

今天的挑战是实现扁平化数组Flat, 例子如下：

## 只扁平化一层的情况

```typescript
type NaiveFlat<T extends any[]> = 这里写代码
// test case
type Naive = [['a'], ['b', 'c'], ['d']];
type NaiveResult = NaiveFlat<[['a'], ['b', 'c'], ['d']]>
// should evaluate to "a" | "b" | "c" | "d"
```
先思考几分钟。。。。

***

## 思路分析

1. 首先我们需要遍历数组
2. 判断每个元素是否是数组
3. 如果是数组就解构，如果不是数组那么就直接返回

那么在Typescript我们如何遍历数组呢，很遗憾是没有提供类似for循环的API的，那么我们如何取到数组中的每个元素呢？

### 第一种方案
我们知道**数组也是对象**，元素的索引为键，元素为值。因此我们可以用对象的形式表达数组，如下表示：

```typescript
type NaiveFlat<T extends any[]> = {
    [K in keyof T]: T[K]
}
type NaiveResult = NaiveFlat<[['a'], ['b', 'c'], ['d']]> //[['a'], ['b', 'c'], ['d']]
```

用 [K in keyof T]的形式可以拿到对象的键,这里的**K**打印出来分别为 0，1，2，那么自然T[K]打印出来为数组的每个值，可能为['a'] 或者 ['b', 'c'] 或者['d']。NaiveFlat打印出来的结果要求是联合类型，那么我们如何将数组中的每个元素表达为联合类型呢？

typescript为我们提供了一种方式：['b', 'c'][number] 表示：**以某个索引取数组中某个元素的所有可能，因此会输出 'b' | 'c'**。

那么我们可以判断T[K]如果是数组就直接返回其联合类型，因为我们目前假定元素是一个只有一层的数组，不存在嵌套数组的情况。

```typescript
type NaiveFlat<T extends any[]> = {
    [K in keyof T]: T[K] extends any? T[K][number]: T[K]
}
type NaiveResult = NaiveFlat<[['a'], ['b', 'c'], ['d']]> //["a", "b" | "c", "d"]
```
["a", "b" | "c", "d"] 需要转换成 "a"| "b" | "c"| "d"，那么我们再次利用刚才的特性，["a", "b" | "c", "d"][number]

最终代码如下：

```typescript
type NaiveFlat<T extends any[]> = {
    [K in keyof T]: T[K] extends any? T[K][number]: T[K]
}[number]
type NaiveResult = NaiveFlat<[['a'], ['b', 'c'], ['d']]> // "a"| "b" | "c" | "d"
```

### 第二种方案
typescript中也可以用数组解构赋值和infer相结合的方式拿到,（想了解infer的后续我会出文章补充）

```typescript
T extends [infer First, ...infer Rest]
```
上面的First 和 Rest分别代表第一个元素和剩下的元素。
接着我们需要采用递归的方式去遍历数组每个元素, 总是判断如果第一个元素是数组就解构，不是数组就保留，并递归剩余的部分：

```typescript
type NaiveFlatHelper<T extends any[]> = T extends [infer First, ...infer Last]? 
First extends any[]? [...First, ...NaiveFlatHelper<Last>]: [First, ...NaiveFlatHelper<Last>]: T;

type TEST = NaiveFlatHelper<[['a'], ['b', 'c'], ['d']]> // ['a', 'b', 'c', 'd']
```
最后要得到打平后的数组的联合类型，采用 [XXX][number]的方式，代码如下：

```typescript
type NaiveFlatHelper<T extends any[]> = T extends [infer First, ...infer Last]? 
First extends any[]? [...First, ...NaiveFlatHelper<Last>]: [First, ...NaiveFlatHelper<Last>]: T;

type TEST = NaiveFlatHelper<[['a'], ['b', 'c'], ['d']]> // ['a', 'b', 'c', 'd']

type NaiveFlat<T> = NaiveFlatHelper<T>[number] // 'a'| 'b' | 'c' | 'd'
```

## 扁平化最深的情况

有了以上的扁平化一层的基础，那么想要扁平化最深，那就太好办了，给大家思考3分钟：............

其实我们只需要动一个地方，要想真正的深度遍历，我们需要把解构数组的地方转为递归调用

#### 针对方案一: T[k][number] => DeepFlat<T[K]>

```typescript
type DeepFlat<T extends any[]> = {
    [K in keyof T]: T[K] extends any? DeepFlat<T[K]>: T[K]
}
type Deep = [['a'], ['b', 'c'], [['d']], [[[['e']]]]];
type NaiveResult = DeepFlat<Deep> // "a" | "b" | "c" | "d" | "e"
```

#### 针对方案二:  ...First => ...DeepFlatHelper<First>

```typescript
type DeepFlatHelper<T extends any[]> = T extends [infer First, ...infer Last]? 
First extends any[]? [...DeepFlatHelper<First>, ...DeepFlatHelper<Last>]: [First, ...DeepFlatHelper<Last>]: T;
type Deep = [['a'], ['b', 'c'], [['d']], [[[['e']]]]];
type TEST = DeepFlatHelper<Deep> // ['a', 'b', 'c', 'd', 'e']

type DeepFlat<T> = DeepFlatHelper<T>[number] // 'a'| 'b' | 'c' | 'd' | 'e'
```

# 关于作者

大家好，我是程序员高翔的猫，MonkeyDesign用户体验部前端研发。

**加我的微信，备注：「个人简单介绍」+「组队讨论前端」**， 拉你进群，每周两道前端讨论分析，题目是由浅入深的，把复杂的东西讲简单的，把枯燥的知识讲有趣的，实用的，深入的，系统的前端知识。


<a name="微信"></a>
<img width="400" src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/weixin.jpg"/>

# 我的公众号

更多精彩文章持续更新，微信搜索：「高翔的猫」第一时间围观


<a name="公众号"></a>

<img src="https://mp.weixin.qq.com/mp/qrcode?scene=10000004&size=102&__biz=MzkxNjMxNDU0MQ==&mid=2247483692&idx=1&sn=2d2baccebfd92fbf6d0506d3c75b3ade&send_time=" data-img="1" width="200" height="200">





