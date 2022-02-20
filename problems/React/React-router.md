# React-router 路由机制

本文以 React-Router 最新的 v6 版本为例，从路由起源、基本构成、渲染机制、匹配机制几个方面系统介绍 React-Router 原理。

众所周知，用 React 或者 Vue 构建的应用都是单页面应用，单页面应用是使用一个 html 前提下，一次性加载 js ， css 等资源，所有页面都在一个容器页面下，页面切换实质是组件的切换。
那么这种路由方式是如何做到 URL 变化，页面无刷新更新内容的呢？
我们可以从以上问题中提炼两个关键词：URL、无刷新。要追本溯源的话，我们需要探讨两个问题：

1. URL 的作用是什么？
2. 如何做到无刷新更新页面内容？

## URL 的作用是什么

URL 英文全称：Uniform Resource Locator，表示“统一资源定位符”，是因特网上标准的资源的地址，它标志一个唯一的网络资源。
你可以把它作为书签；搜索引擎可以索引它；你可以复制和粘贴它，并通过电子邮件发给其他人，后者可以点击它，并最终看到你最初看到的相同资源。这些都是 URL 独特且优秀的品质。
但与此同时，浏览器一直有一个基本的限制：如果你改变了 URL，即使是通过脚本，也会引发对远程网络服务器的请求操作和整个页面的刷新。
这需要花费一定的等待时间和网络资源，而且当你浏览到一个与当前页面基本相似的页面时，这似乎特别浪费。
新页面上的所有内容都会被下载，甚至包括与当前页面完全相同的部分。我们没有办法告诉浏览器改变 URL 但只下载半个页面。

那有没有办法在切换 url 的时候，浏览器只下载用户需要的脚本资源，页面只更新某一部分而不刷新页面呢？还真有办法，下面我们讨论第二个问题。

## 如何做到无刷新更新页面内容

HTML5 History 新的 API 提供了这样的操作，它包括了一种向浏览器历史记录添加状态的方法，可以改变浏览器搜索栏中的 URL，而无需触发页面刷新，以及当用户按下浏览器的返回按钮，历史记录从历史堆栈中弹出时触发的监听事件。它们分别为：

```js
window.history.pushState(state, title, url);
window.history.replaceState(state, title, url);
window.addEventListener("popstate", function (e) {
  /* 监听改变 */
});
```

以上 History API 已经很好的兼容了大部分浏览器(支持到 IE10),但只是做到了第一步：更改 URL 状态，不刷新页面。那么如何让 dom 在不刷新页面的情况下也改变呢？

举个经典的 🌰

假设你有两个页面，A 页和 B 页。这两个页面 90%是相同的，只有 10%的页面内容是不同的。用户导航到 A 页，然后试图导航到 B 页。但你没有触发全页面刷新，而是中断了这个导航，并手动进行了以下步骤。

1. 从 B 页面加载与 A 页面不同的 10%的页面（可能使用 XMLHttpRequest）
2. 把改变的内容换进来（使用 innerHTML 或其他 DOM 方法）
3. 通过 pushState 将浏览器地址改变为 B 页面的 URL

浏览器最终会出现一个与 B 页相同的 DOM。浏览器的地址栏最后显示的是一个与 B 页相同的 URL，与直接导航到 B 页没有任何体验上的区别。但其实用户从来没有真正浏览过 B 页，也没有进行过全页面刷新。
这是个假象，但是因为 "编译 "的页面看起来和 B 页一样，并且有和 B 页一样的 URL，这样的过程对用户来说是无感知的。

以下网站为我们提供了无刷新更新页面内容最基本的实现，思路基本跟上面的步骤一致：
http://diveintohtml5.info/examples/history/fer.html

核心代码如下：

```js
function swapPhoto(href) {
  var req = new XMLHttpRequest();
  req.open(
    "GET",
    "http://diveintohtml5.info/examples/history/gallery/" +
      href.split("/").pop(),
    false
  );
  req.send(null);
  if (req.status == 200) {
    document.getElementById("gallery").innerHTML = req.responseText;
    setupHistoryClicks();
    return true;
  }
  return false;
}

function addClicker(link) {
  link.addEventListener(
    "click",
    function (e) {
      if (swapPhoto(link.href)) {
        history.pushState(null, null, link.href);
        e.preventDefault();
      }
    },
    true
  );
}
```

以上逻辑是最原生的路由实现，也是目前众多单页应用路由切换的实现思路。下面我们会以 React-Router 为例子，深入介绍 React-Router 的生态以及实现方式。

## React-Router 的路由原理

弄清楚 Router 原理之前，用一幅图表示 History ，React-Router ， React-Router-Dom 三者的关系。

下面是 React-Router-dom V5 版本的关系图, 与 V6 相比，除了一些 API 有变动之外，其他的一致：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-router1.jpg"/>

- **history**： history 是整个 React-router 的核心，里面包括两种路由模式下改变路由的方法，和监听路由变化方法等。
- **react-router**：既然有了 history 路由监听/改变的核心，那么需要调度组件负责派发这些路由的更新，也需要容器组件通过路由更新，来渲染视图。所以说 React-router 在 history 核心基础上，增加了 Router ，Routes ，Route 等组件来处理视图渲染。
- **react-router-dom**：  在 react-router 基础上，增加了一些 UI 层面的拓展比如 Link ，NavLink 。以及两种模式的根部路由 BrowserRouter ，HashRouter 。

### 两种路由主要方式

路由主要分为两种方式，一种是 history 模式，另一种是 Hash 模式。History 库对于两种模式下的监听和处理方法不同，稍后会讲到。 两种模式的样子：

- history 模式下：http://www.xxx.com/home
- hash 模式下： http://www.xxx.com/#/home

而两种方式的引用也不同：

开启 history 模式

```jsx
import { BrowserRouter as Router } from "react-router-dom";

function Index() {
  return <Router> {/* ...开启history模式 */} </Router>;
}
```

开启 hash 模式

```jsx
import { HashRouter as Router } from "react-router-dom";

function Index() {
  return <Router> {/* ...开启hash模式 */} </Router>;
}
```

两种路由方式的区别分别如下：

原理区别：

- hash 原理：hash 通过监听浏览器的 onhashchange()事件变化，查找对应的路由规则
- history 原理： 利用 HTML5 History 中新增的两个 API pushState() 和 replaceState() 和一个事件 onpopstate 监听 URL 变化

请求区别：

- hash  模式下: 仅 hash 符号#之前的内容会被包含在请求中，如  https://music.163.com/#/my/，浏览器实际发出的请求是https://music.163.com，因此不会引起页面重新加载，对于后端来说，即使没有做到对路由的全覆盖，也不会返回 404 错误。
- history  模式下: 前端的 URL 必须和实际向后端发起请求的 URL 一致，如http://www.abc.com/book/id。如果后端缺少对 /book/id  的路由处理，将返回 404 错误，需要服务端的支持。所以呢，你要在服务端增加一个覆盖所有情况的候选资源：如果 URL 匹配不到任何静态资源，则应该返回同一个 index.html 页面，这个页面就是你 app 依赖的页面。”

下面我们深入'react-router-dom 和 history 源码，讨论 BrowserRouter 组件和 HashRouter 组件的实现原理

## React-Router 基本构成

下面分别是 BrowserRouter 和 HashRouter 组件的源码：

```jsx
import { createBrowserHistory, createHashHistory } from "history";

/***BrowserRouter组件***/
export function BrowserRouter({
  basename,
  children,
  window
}: BrowserRouterProps) {
  let historyRef = React.useRef<BrowserHistory>();
  if (historyRef.current == null) {
    historyRef.current = createBrowserHistory({ window });
  }

  let history = historyRef.current;
  let [state, setState] = React.useState({
    action: history.action,
    location: history.location
  });

  React.useLayoutEffect(() => history.listen(setState), [history]);

  return (
    <Router
      basename={basename}
      children={children}
      location={state.location}
      navigationType={state.action}
      navigator={history}
    />
  );
}

/***HashRouter组件***/
export function HashRouter({ basename, children, window }: HashRouterProps) {
  let historyRef = React.useRef<HashHistory>();
  if (historyRef.current == null) {
    historyRef.current = createHashHistory({ window });
  }

  let history = historyRef.current;
  let [state, setState] = React.useState({
    action: history.action,
    location: history.location
  });

  React.useLayoutEffect(() => history.listen(setState), [history]);

  return (
    <Router
      basename={basename}
      children={children}
      location={state.location}
      navigationType={state.action}
      navigator={history}
    />
  );
}
```

我们可以看到以上两个组件其实就是根据 history 提供的 createBrowserHistory 或者 createHashHistory 创建出不同的 history 对象，传入到 Router 组件中，而这些 history 对象分别包含了 history 和 hash 模式下用到的方法和变量，比如：push、go、replace、back、listen 等。

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-router2.jpg"/>

## React-Router 更新机制

下面是 Router 组件源码，我删除了部分代码，只留下关键的部分：

```jsx
export function Router({
  basename: basenameProp = "/",
  children = null,
  location: locationProp,
  navigationType = NavigationType.Pop,
  navigator,
  static: staticProp = false,
}: RouterProps): React.ReactElement | null {
  let basename = normalizePathname(basenameProp);
  let navigationContext = React.useMemo(
    () => ({ basename, navigator, static: staticProp }),
    [basename, navigator, staticProp]
  );

  let location = React.useMemo(() => {
    return { pathname, search, hash, state, key };
  }, [pathname, search, hash, state, key]);

  return (
    <NavigationContext.Provider value={navigationContext}>
         
      <LocationContext.Provider
        children={children}
        value={{ location, navigationType }}
      /> 
    </NavigationContext.Provider>
  );
}
```

通过 router 组件不难看出： **首先 React-Router 是通过 context 上下文方式传递的路由信息。context 改变，会使消费 context 的组件更新**，这就能合理解释了，当开发者触发路由改变，为什么能够重新渲染匹配组件。

我们也注意到，在 BrowserRouter 和 HashRouter 组件内部都有这样一段逻辑：

```jsx
let [state, setState] = React.useState({
  action: history.action,
  location: history.location,
});

// 当路由改变，会触发 listen 方法，传递新生成的 location
React.useLayoutEffect(() => history.listen(setState), [history]);
```

history 是一个对象，里面除了暴露的函数外，还有两个 get 属性

```js
{
    get action() {
      return action;
    },
    get location() {
      return location;
    },
}
```

**因此 react 会在 DOM 更新过后，监听 history 对象中 action 和 location 的变化，从而触发 history.listen 的订阅收集，而 location 其实就是 window.location**

```js
function push(to, state) {
    let nextAction = Action.Push;
    let nextLocation = getNextLocation(to, state);
    // 用try catch的原因是因为ios限制了100次pushState的调用
      try {
        globalHistory.pushState(historyState, '', url);
      } catch (error) {
        window.location.assign(url);
      }
      applyTx(nextAction);
    }
  }
```

> 调用 history.pushState()或 history.replaceState()不会触发 popstate 事件。只有在做出浏览器动作时，才会触发该事件，如用户点击浏览器的回退按钮

当我们调用 history.push 方法时, 其实执行了 window.history.pushState(historyState, '', url),改变 url,（window.location 改变会触发 listen 方法订阅更新），紧接着执行方法`applyTx(nextAction);` ，遍历每个订阅函数，触发 setState, 然后通过 setState 来改变 context 中的 value, 触发组件更新，所以改变路由，本质上是 location 改变带来的更新作用。

```js
function applyTx(nextAction) {
  action = nextAction;
  [index, location] = getIndexAndLocation();
  listeners.call({
    action,
    location,
  });
}
```

改变路由，子组件更新，那 react-router 如何知道展示哪一个组件呢？下面我们深入 React-router 的路由匹配机制。

## React-Router 路由匹配机制

以最新的 V6 版本为背景，我们日常写路由一般是这样的：Route 一定要包裹在 Routes 中：

```jsx
<Routes>
  <Route path="/" element={<Dashboard />}>
    <Route path="msg" element={<DashboardMessages />} />
    <Route path="tasks" element={<DashboardTasks />} />
  </Route>
  <Route path="about" element={<AboutPage />} />
</Routes>
```

那么问题来了：

1. react-router 是如何判断 Routes 的子组件不是 Route 就会报错的呢？
2. react-router 是如何匹配对应的 Route 组件展示的？

上源码：

```js
export function Routes({
  children,
  location,
}: RoutesProps): React.ReactElement | null {
  return useRoutes(createRoutesFromChildren(children), location);
}
```

可以看到 Routes 实际上调用了 useRoutes 方法，而它的作用就是使用路由对象而不是<Route>元素来定义你的路由，你甚至可以传入**嵌套路由数组**来定义路由：

```jsx
import * as React from "react";
import { useRoutes } from "react-router-dom";
function App() {
  let element = useRoutes([
    {
      path: "/",
      element: <Dashboard />,
      children: [
        {
          path: "messages",
          element: <DashboardMessages />,
        },
        { path: "tasks", element: <DashboardTasks /> },
      ],
    },
    { path: "team", element: <AboutPage /> },
  ]);
  return element;
}
```

那么不难看出`createRoutesFromChildren(children)`方法就是创建这样配置形式的 children 对象，并构成数组的形式。

因为存在 Route 嵌套 Route 的情况，`createRoutesFromChildren`方法以递归的方式生成 routes 数组，并判断 element.type 是否等于 Route, 如果不是，就抛出异常：

```js
invariant(
  element.type === Route,
  `[${
    typeof element.type === "string" ? element.type : element.type.name
  }] is not a <Route> component. All component children of <Routes> must be a <Route> or <React.Fragment>`
);
```

这就解释了一开始的第一个疑问：react-router 是如何判断 Routes 的子组件不是 Route 的？

紧接着我们需要回答第二个问题：react-router 是如何匹配对应的 Route 组件展示的？

useRoutes 函数的简化版如下

```jsx
export function useRoutes(
  routes: RouteObject[],
  locationArg?: Partial<Location> | string
): React.ReactElement | null {

  let matches = matchRoutes(routes, { pathname: remainingPathname });

  return _renderMatches(
    matches &&
      matches.map(match =>
        Object.assign({}, match, {...} ),
    parentMatches
  );
}
```

以上的 matchRoute 函数就是 React Router 匹配算法的核心，它会返回匹配的路由并由函数`_renderMatches`渲染出来：

```jsx
export function matchRoutes(
  routes: RouteObject[],
  locationArg: Partial<Location> | string,
  basename = "/"
): RouteMatch[] | null {
  let location =
    typeof locationArg === "string" ? parsePath(locationArg) : locationArg;
  let pathname = stripBasename(location.pathname || "/", basename);
  if (pathname == null) {
    return null;
  }
  let branches = flattenRoutes(routes);
  rankRouteBranches(branches);
  let matches = null;
  for (let i = 0; matches == null && i < branches.length; ++i) {
    matches = matchRouteBranch(branches[i], pathname);
  }
  return matches;
}
```

由于路由可能存在多层嵌套的情况，那么 routes 可能是多维数组，那么**flattenRoutes**将其打平，在 flatten 的过程中会收集每个 route 的 props 作为 routeMeta，并采用了深入优先遍历的方式，优先添加子路由，因此打平后的数组，总是子路由在前，父路由在后。
以下面的多层嵌套的路由为例：

```jsx
<Routes>
     
  <Route path="/" element={<Layout />}>
    <Route index element={<Home />} />
    <Route path="about" element={<About />} />
    <Route path="dashboard" element={<Dashboard />} />
    <Route path="skills">
      <Route path="ggx" element={<Ggx />} />
    </Route>
    <Route path="user" element={<User />} />
  </Route>
        <Route path="*" element={<NoMatch />} />
</Routes>
```

打印出来的`branches`是这样的：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-router3.png"/>

**最后将每个 branch 和 pathname 传入 matchRouteBranch: `matches = matchRouteBranch(branches[i], pathname);` , 判断 branches 中是否有与 pathname 匹配的项，把匹配的都返回，与筛选数组类似。**
而 matchRouteBranch 会遍历每个 branch 的 routesMeta。以上面的 demo 为例子，每个 branch 下的 routeMeta 如下图所示：

<img src="https://cdn.jsdelivr.net/gh/HelloGGX/Front-End-question@master/pics/react-router4.png"/>

例如：当切换到路由 http://localhost:3000/skills/ggx ,此时有几个关键变量要了解：

```js
pathname: "/skills/ggx" //当前要匹配的路径
matchedPathname："/" //目前已经匹配的路径
remainingPathname: "/skills/ggx" //剩余要匹配的路径
```

由以下关键代码可以看出， matchRouteBranch 会遍历数组 routesMeta，每一项 routeMeta 都会通过 matchPath 函数看看是否匹配到，其会根据 routeMeta 的 relativePath(即我们在 Route 中写的 path，如 path = 'skills'，path='ggx'；)，生成对应的正则匹配，只有所有 routeMeta 都匹配上了，才真正渲染页面，只要中途有一个匹配不上，就会直接跳出 for 循环，中断后面的渲染。（matchPath 函数涉及的正则算法不必深究）

> caseSensitive(即根据 relativePath 生成的正则是否忽略大小写)
> end(是否是最后一项 routeMeta，最后一项表示是该 route 自己的路由信息，同时也意味着匹配到最后了)

```js
function matchRouteBranch<ParamKey extends string = string>(
  branch: RouteBranch,
  pathname: string
): RouteMatch<ParamKey>[] | null {
  let { routesMeta } = branch;

  let matchedParams = {};
  let matchedPathname = "/";
  let matches: RouteMatch[] = [];
  for (let i = 0; i < routesMeta.length; ++i) {
    let meta = routesMeta[i];
    let end = i === routesMeta.length - 1;
    // 剩余要匹配的路径, 用slice方法，总路径除去已经匹配到的，就是还剩的路径
    let remainingPathname =
      matchedPathname === "/"
        ? pathname
        : pathname.slice(matchedPathname.length) || "/";
    let match = matchPath(
      { path: meta.relativePath, caseSensitive: meta.caseSensitive, end },
      remainingPathname
    );

    if (!match) return null;

    Object.assign(matchedParams, match.params);

    let route = meta.route;

    matches.push({
      params: matchedParams,
      // 之前已经匹配到的路径+当前匹配成功的relativePath => 已经匹配完的路径
      pathname: joinPaths([matchedPathname, match.pathname]),
      pathnameBase: joinPaths([matchedPathname, match.pathnameBase]),
      route
    });

    if (match.pathnameBase !== "/") {
      matchedPathname = joinPaths([matchedPathname, match.pathnameBase]);
    }
  }

  return matches;
}
```

例如：路由 http://localhost:3000/skills/ggx 的匹配的顺序为：

第一次匹配：

```js
pathname: "/skills/ggx" //当前要匹配的路径
matchedPathname："/" //目前已经匹配的路径
remainingPathname: "/skills/ggx" //剩余要匹配的路径
```

第二次匹配：

```js
pathname: "/skills/ggx" //当前要匹配的路径
matchedPathname："/skills" //目前已经匹配的路径
remainingPathname: "/ggx" //剩余要匹配的路径
```

第三次匹配：

```js
pathname: "/skills/ggx" //当前要匹配的路径
matchedPathname："" //目前已经匹配的路径
remainingPathname: "/skills/ggx" //剩余要匹配的路径
```

最后当所有的 routeMeta 都匹配成功，matchRouteBranch 会返回如下结构供`_renderMatches`函数渲染：

```js
[
{params: {…}, pathname: '/', pathnameBase: '/', route: {…}}
{params: {…}, pathname: '/skills', pathnameBase: '/skills', route: {…}}
{params: {…}, pathname: '/skills/ggx', pathnameBase: '/skills/ggx', route: {…}}
]
```

而\_renderMatches  会根据匹配项和父级匹配项  parentMatches，执行方法 `matches.reduceRight` 从右到左遍历以上数组，即从 child --> parent 渲染  RouteContext.Provider。

```js
function _renderMatches(
  matches: RouteMatch[] | null,
  parentMatches: RouteMatch[] = []
): React.ReactElement | null {
  if (matches == null) return null;
  return matches.reduceRight((outlet, match, index) => {
    return (
      <RouteContext.Provider
        children={
          match.route.element !== undefined ? match.route.element : <Outlet />
        }
        value={{
          outlet,
          matches: parentMatches.concat(matches.slice(0, index + 1))
        }}
      />
    );
  }, null as React.ReactElement | null);
}
```

最终我们的多层嵌套的路由 demo:

```jsx
<Routes>
     
  <Route path="/" element={<Layout />}>
    ...
    <Route path="skills">
      <Route path="ggx" element={<Ggx />} />
    </Route>
    ...
  </Route>
  ...
</Routes>
```

渲染出来如下:

```jsx
<RouteContext.Provider
  value={{
    outlet: (
      <RouteContext.Provider
        value={{
          outlet: (
            <RouteContext.Provider value={{ outlet: null }}>
              {<Ggx /> || <Outlet />}
            </RouteContext.Provider>
          ),
        }}
      >
        /** 由于路由/skills没有组件渲染，它的element: undefined，所以渲染{" "}
        <Outlet /> **/
        {<Outlet />}         
      </RouteContext.Provider>
    ),
  }}
>
  {<Layout /> || <Outlet />}
</RouteContext.Provider>
```

你会发现渲染出来是与路由结构对应的用`RouteContext.Provider`包裹的层层嵌套的结构(注意：是嵌套在了 value 内部)。
这种结构的好处是：react-router 可以在内部自定义很多 hook，通过`let { matches， outlet } = React.useContext(RouteContext);` 就能方便的拿到子路由的相关信息，非常典型的例子，react-router 暴露出了 useOutlet 方法，可以直接
拿到此路由层次结构级别的子路由的元素

```jsx
export function useOutlet(context?: unknown): React.ReactElement | null {
  let outlet = React.useContext(RouteContext).outlet;
  if (outlet) {
    return (
      <OutletContext.Provider value={context}>{outlet}</OutletContext.Provider>
    );
  }
  return outlet;
}

export function useParams<
  ParamsOrKey extends string | Record<string, string | undefined> = string
>(): Readonly<
  [ParamsOrKey] extends [string] ? Params<ParamsOrKey> : Partial<ParamsOrKey>
> {
  let { matches } = React.useContext(RouteContext);
  let routeMatch = matches[matches.length - 1];
  return routeMatch ? (routeMatch.params as any) : {};
}

```

因此方便了开发者从父路由拿到当前展示的深层级子路由的信息。

### 额外提一嘴：Outlet

我们会发现，Route 如果没有定义 element，或者组件不渲染的时候，react-router 会用 `<Outlet/>`组件兜底。

```jsx
export function Outlet(props: OutletProps): React.ReactElement | null {
  return useOutlet(props.context);
}

export function useOutlet(context?: unknown): React.ReactElement | null {
  //RouteContext.Provider value中的outlet
  let outlet = React.useContext(RouteContext).outlet;
  if (outlet) {
    return (
      <OutletContext.Provider value={context}>{outlet}</OutletContext.Provider>
    );
  }
  return outlet;
}
```

从上面的 if 判断可以看出，只要父级路由有  <Outlet />  就能拿到最近一层 RouteContext 的  outlet  了，所以我们常常在嵌套路由的 parent route 的 element 写上一个<Outlet />，相当于插槽的作用，因此我们常常这样写：

```jsx
<div>
  <nav>
    <ul>
             
      <li>
          <Link to="/">Home</Link>       
      </li>
      <li>
          <Link to="/about">About</Link>       
      </li>       <li>
           <Link to="/dashboard">Dashboard</Link>       
      </li>         <li>
        {" "}
             <Link to="/skills">
            <Link to="/skills/ggx">ggx</Link>         
        </Link>      
      </li>    <li>
           <Link to="/user">User</Link>       
      </li>
    </ul>
  </nav>
  <hr />
  <Outlet /> 
</div>
```
