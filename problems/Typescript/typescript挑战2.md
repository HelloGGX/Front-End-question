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
2. 判断数组每个元素是否是数组
3. 如果是数组就取出其中的元素，如果不是数组那么就直接返回

那么在Typescript我们如何遍历数组呢，很遗憾是没有提供类似for循环的API的，那么我们如何取到数组中的每个元素呢？

### 第一种方案
我们知道数组也是对象，元素的索引为键，元素为值。因此我们可以用对象的形式表达数组，如下表示：

```typescript
type NaiveFlat<T extends any[]> = {
    [K in keyof T]: T[K]
}
type NaiveResult = NaiveFlat<[['a'], ['b', 'c'], ['d']]> //[['a'], ['b', 'c'], ['d']]
```

用 [K in keyof T]的形式可以拿到对象的键,这里的K打印出来分别为 0，1，2，那么自然T[K]打印出来为数组的每个值，可能为['a'] 或者 ['b', 'c'] 或者['d']。NaiveFlat打印出来的结果要求是联合类型，那么我们如何将数组中的每个元素表达为联合类型呢？

typescript为我们提供了一种方式：['b', 'c'][number] 表示以某个索引取数组中某个元素，因此会输出 'b' | 'c'。

那么我们可以判断T[K]如果是数组就直接返回其联合类型，因为我们目前假定元素是一个只有一层的数组，不存在嵌套。

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


