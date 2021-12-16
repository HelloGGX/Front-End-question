<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/typescript-banner.png"/>

# 1. Typescript挑战-1 <img src="https://img.shields.io/badge/typescript-%E4%B8%AD%E7%BA%A7-blue"/>

今天的挑战是关于declare定义函数：如何根据Config类型，返回对应的User键值对类型，实例代码如下：

```typescript
type Config = {
    name: boolean;
    lastname: boolean;
};
type User = {
    name?: string;
    lastname?: string;
};

// 修改下面代码
declare function getUser(config: Config): User

// 测试用例
const user = getUser({ name: true, lastname: false })
user.name // 这里会提示name为可选类型
user.lastname // 这里会报错说没有该类型

const user2 = getUser({ name: true, lastname: true })
user2.name // 这里会提示name为可选类型
user2.lastname // 这里会提示lastname为可选类型

const user3 = getUser({ name: false, lastname: true })
user3.name // 这里会报错说没有该类型
user3.lastname // 这里会提示lastname为可选类型

const user4 = getUser({ name: false, lastname: false })
user4 // 返回空对象 {}

```
**我们如何从类型层面去约束函数的返回值呢？**

给各位思考10分钟。。。。。。

***
#### 1、第一种思路分析

- 我们需要根据true/false来找到对应的key
- 可能key都是为true的，那么获取到的为true的key应该是联合类型的
- 我们拿到了值为true的key的联合，然后通过Pick函数来取User

#### 2、代码实施

因为需要根据对应的输入返回对应的输出，因此我们首先改造一下getUser函数定义，加入泛型

```typescript
declare function getUser<T extends Config, U extends User>(config: T): User
```
我们需要工具函数来拿到泛型T中的值为true的键，因此定义一个type工具函数：KeysOfTrueValues

我们要拿到Config对象的每一个key的类型，首先通过**keyof Config** 拿到key的联合，然后通过关键字 in 映射到一个K类型中，具体代码为 **{ [K in keyof T] }: T[K]**, 这里的K表示对象T中的某个键，继而也可以拿到键值：T[K], 此时我们可以通过条件类型extends来判断 T[K]是否属于true,如果是true那么表示需要拿到该K, 否则返回never 表示什么都没有。

因此我们可以根据 { name: true, lastname: false } 来得到 { name: 'name', lastname: never }, 再通过 { name: 'name', lastname: never }[keyof  { name: 'name', lastname: never }] 拿到值的联合：name| never 也就是 name。代码如下所示：

```typescript
type KeysOfTrueValues<T extends Config> = {
    [K in keyof T]: T[K] extends true ? K : never
}[keyof T];
```

此时我们就可以通过Pick拿到User中对应的键值对了，理想是这样的：

```typescript
Pick<User, KeysOfTrueValues<Config>>
```
但是这里我们用的是泛型，User和 config属于两个不同的泛型条件，因此如果我们这样写的话就会出现报错：大致意思是KeysOfTrueValues<T> 不等于 keyof U,

```typescript
Pick<U, KeysOfTrueValues<T>>
```
因此这里我们需要做条件类型判断：`KeysOfTrueValues<T> 可分配给 keyof U` 才可以用Pick

```typescript
KeysOfTrueValues<T> extends keyof U? Pick<U, KeysOfTrueValues<T>>
```
再加上边界条件的判断，当每个key都是false的时候，返回{}，得到下面的代码：

```typescript
type ReturnUser1<T extends Config, U extends User> = KeysOfTrueValues<T> extends never? {}:  KeysOfTrueValues<T> extends keyof U? Pick<U, KeysOfTrueValues<T>>: {}
```

***
#### 1、第二种思路分析

- 在User的键上通过断言来判断该键是否是true,是就返回键，否则返回never, 我们知道如果键为never，那么表示这个键根本不存在


#### 2、代码实施

这也是我一开始能想到的办法

```typescript
type ReturnUser<T extends Config, U extends User> = {
    [P in keyof T as  T[P] extends true ? P : never]?: P extends keyof U? U[P]: never
}
```
