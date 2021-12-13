<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/typescript-banner.png"/>

# 1. what？！这不是TypeScript的bug吗

事情是这样的，今天一个朋友发来一个有趣的问题,代码如下：

```typescript
type User = {
  id: number;
  kind: string;
};

function getCustomer<T extends User>(u: T): T {
  return {
    id: u.id,
    kind: 'customer'
  }
}
```
这里的TS会报如下错误:

>不能将类型“{ id: number; kind: string; }”分配给类型“T”。
>  "{ id: number; kind: string; }" 可赋给 "T" 类型的约束，但可以使用约束 "User" 的其他子类型实例化 "T"

这里报错说的很清楚了，函数返回值{id: number; kind: string} 可赋给“T”类型的约束，建议使用User的其他子类型，比如 User。
乍一看，由getCustomer返回的对象是有效的User类型，因为它有User中定义的两个必要字段id和kind。但要注意的是，**这里的[extends](https://www.tslang.cn/docs/release-notes/typescript-2.8.html)表示条件类型。我们在这里使用的是类型变量T，它从User扩展而来，但这并不意味着它就是User。T是可以分配给User的，所以它需要拥有User所拥有的所有字段，但是，它可以拥有更多的字段**。如下：

```typescript
getCustomer({id: 1, kind: 'admin', other: 2})
```

这正是问题所在，返回的对象是一个User，并传递了它的所有约束，但没有传递T的所有约束，而T可以有额外的字段。我们不知道这些字段是什么，所以为了解决该类型问题，我们应该返回一个拥有T的所有字段的对象，并且我们知道T的所有字段都在参数u中。理解了问题产生的原因，我们会自然的产生如下几种解决方案：

方案一：我们可以使用解构操作符，将所有未知的字段分散到新创建的对象中。如下:

```typescript
type User = {
  id: number;
  kind: string;
};

function getCustomer<T extends User>(u: T): T {
  return {
    ...u,
    id: u.id,
    kind: 'customer'
  }
}
```

方案二：根据报错提示我们直接指定返回的类型为User的子类型，可以是User

```typescript
type User = {
  id: number;
  kind: string;
};

function getCustomer<T extends User>(u: T): User {
  return {
    id: u.id,
    kind: 'customer'
  }
}
```

方案三：甚至可以强制类型断言为T（不推荐）

```typescript
type User = {
  id: number;
  kind: string;
};

function getCustomer<T extends User>(u: T): T {
  return {
    id: u.id,
    kind: 'customer'
  } as T
}
```

事情看来已经得到圆满解决了，但是我们可以看到返回的kind是一个值为'customer'的string类型，那假如我定义传入的User子类型中kind：'admin'，会发生什么呢？如下面所示，我们定义一个别名为Admin的类型：

```typescript
type Admin = User & {
  kind: 'admin';
}
```
显然 Admin是User扩展而来的，不信我们试试：

```typescript
type IsAdminUser = Admin extends User ? true : false
```
最终IsAdminUser打印出来是true,那我可以放心这样写了（也没有报错）：

```typescript
const admin = getCustomer({ id: 1, kind: 'admin' } as Admin);
```

那么根据前面的定义，getCustomer返回的类型为Admin, 期望是返回 {id:1, kind: 'admin'} 但是最终admin打印出来是下面这样：

```javascript
{ id: 1, kind: 'customer' }
```
显然违背了我之前的逻辑意愿，ts也诡异的正常运行了，这个问题希望TS早日解决，从这点我们可以看出，以后项目中尽量少用类型断言，毫无根据的断言是危险的！多说一句我的TS版本是4.4.3。


