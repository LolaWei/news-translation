> * 原文地址：[Async Generators as an alternative to State Management](https://www.freecodecamp.org/news/async-generators-as-an-alternative-to-state-management/?fbclid=IwAR2Py7k7WayAE_zq4tkd99pj3oBP7scsKp9mZbPCtv_zJqvhN4eOVAef6M8)
> * 原文作者：[Vitalii Akimov](https://www.freecodecamp.org/news/author/vitalii/)
> * 译者：luyc
> * 校对者：
  
![Async Generators as an alternative to State Management](https://www.freecodecamp.org/news/content/images/size/w2000/2019/07/async-state.png)

`Async Generators`（异步生成器） 是一个简单但功能强大的特性，它现在已经是 `JavaScript` 的一部分。它解锁了许多新的工具来改进软件结构，使其更加灵活、易扩展、更易组合。


#### TL;DR

-   使用 `Async Generators`，将不再需要组件状态、状态管理工具、生命周期方法，甚至是最近的 `React Context`、`Hooks` 、`Suspense APIs`。它将简化开发，管理和测试。
-   与状态管理方法不同的是，异步生成器将异步转换变得更加可控而无害（它只在生成器作用域里有效）。
-   这个思路有函数式编程的背景
-   状态持久化工具，像时间旅行器，通用应用程序也是可用的。
-   这篇文章使用了 `React` 和 `JavaScript`，但是这项技术在任何其他框架或拥有生成器（协程）的编程语言都是适用的。
-   _我只在最后简短的宣传我的工具。本文的大部分内容都是关于异步生成器的，没有其他依赖内容_

![](https://cdn-media-1.freecodecamp.org/images/1*KlSEFFBTjyZKovSoQ0NnEw.png)

![](https://cdn-media-1.freecodecamp.org/images/1*ZrJKJqBsksWd-8uKM9OvgA.png)

让我们从 [Redux动机页面][1] 的声明开始:

> _这种复杂性很难处理，因为****我们混淆了两个概念****，这些概念对于人类的思维来说非常难以理解：****突变和异步性。**** 我称之[曼妥思和可乐现象][2]。两者分开时各自都运行得很好，但放在一起使用，他们就会造成混乱_

Redux和其他的状态管理工具主要侧重于约束和控制数据的突变。异步生成器可以处理异步。如果变异仅在特定的生成器范围内可见，则这使得突变更安全。

所有常见的状态管理技术可以分为两大类。

第一类是维护数据关系图，通过处理器传播改变——React组件状态、MobX、RxJS。维护这些关系是一项复杂的任务。底层库通过管理订阅，优化处理器执行顺序，对它们进行批处理来负责部分复杂的任务，但有时使用起来仍然令人困惑，通常需要进行硬微调，例如，使用 `shouldComponentUpdate` 方法。

另一种方法是将突变限制为仅有单个单元（storage）（例如，Redux）。这需要更小的并且带有更少魔法功能的库。说它是库，其实它更像是一种模式。不幸的是，这种程序会更加冗长，这破坏了数据的封装。虽然有很多模式或包装器可以解决这个问题，但是它们使用单个单元的方法更类似于基于图的方法。

这个故事的技术和 Redux 都是基于事件源模式，它们有许多相似之处。它也会为具有副作用的操作提供封装数据和使用同步来确定执行顺序。

这种方法也可以抽象地视为依赖图，但是变化是反向传播的，从它的根节点朝它生成树的子节点。在每一个节点处，我们需要检查传播是否应该传给子节点。这样使调度算法非常轻量级并且易于控制。它不需要引入任何库，仅基于JavaScript内置功能。

让我们首先针对 [Redux VanillaJS counters][3] 的例子来说明这个想法。

原版的 reducer 是用异步生成器函数替代的。该函数计算并将其状态存储在一个局部变量中。它还会产生计算值，新的值被存储在单例存储中，并且它可以从事件处理器中访问。我将在下一步中删除那个单例存储。

这个版本与 Redux 中的例子没有什么不同。异步生成器可以是 Redux 的存储中间件。这样违反了其中一条 Redux [原则][4] 原则，即仅将所有应用程序状态存储在存储器中。即使生成器没有任何的局部变量，它仍然具有执行状态————在 `yield` 或 `await` 中暂停执行的代码的位置。

#### 从内到外转换组件

生成器函数是返回迭代器的函数。我们可以用普通函数完成我们所做的一切。例如，通过组合生成器函数，我们可以将计算分成几个独立的阶段，每个阶段有自己的封装状态。每个阶段接收前一阶段产生的消息，处理它们时会产生另一个消息，将这些消息传递到下一个阶段。

消息的有效负载可以包含 VDOM 元素。我们不是使用一个单独的组件树，而是将它的一部分发出去并且发送到下一个阶段，在那里它们可以被组装或转换。 这与React的计数器示例相同。

`pipe` 函数是一个函数组合。函数接收两个参数。第一个是来自前一阶段消息的异步迭代。第二个是将消息发送到管道的起始。它应该只从事件处理器调用。 使用 JavaScript 嵌入式管道运算符可以很快替换此函数。

当我们编写普通函数时，链中的下一个函数仅在前一个函数完成之后开始执行。对于生成器（实际上是任何协程），执行操作都可以与其他函数交叉执行或暂停。这使得将不同部分组合起来会更加容易。

![](https://www.freecodecamp.org/news/content/images/2019/07/lanes--1--1.svg)

上面的示例通过将一些菜单按钮从根组件分离到一个单独的阶段，简要地展示了可扩展性。它不是将菜单按钮抽象到一个单独的组件中，而是维护一个占位符，它会在 “MENU\_ITEM” 类型的消息中注入它接收的组件。

#### 扩展

这项技术令人兴奋的一点是，你不需要预先设计什么即可实现代码重用和解耦。如今过早抽象的害处可能远大于过早优化。差不多可以肯定的是，它会因为过度设计而导致混乱而无法使用。使用抽象生成器，很容易保持稳定并实现所需的功能，在需要时进行拆分，而不用考虑将来的扩展，在更多细节可用之后易于重构或抽象一些公共部分。

Redux 以使程序更易于扩展和重用而闻名。这个故事中的方法也是基于事件溯源的，但是运行异步操作要简单得多，而且没有单个存储的瓶颈，所以不应该过早设计任何东西。

许多开发人员喜欢单一存储，因为它易于控制。虽然控制不是免费的。事件源模式被广泛认同的优点是不存在中央数据库，因此修改某个部分的时候，不会破坏其他部分，操作起来也更加简单并且风险更低。下面的“持久性”部分讨论了单个存储的另一个问题。

这里有篇文章 [解耦业务逻辑][5]，有更多详细的案例研究。在某个步骤里面，我添加了一个多选功能来拖放，而不会改变单个元素处理中的任何内容。使用单一存储，这意味着将其模型从存储单个当前拖动的元素更改为列表。

在 Redux 里面，有相似的解决方案，叫做应用性高阶 reducer。它能够让 reducer 与一个单独的元素工作并且转化为一个 reducer 工作列表。生成器解决方案使用更高阶的异步生成器替代，为单个的元素提供函数并且为列表生成一个函数。它很类似但不那么冗长，因为生成器封装了数据和隐式控制状态。

作为例子，让我们做一个计数器列表。“解耦业务逻辑”一文中介绍了此步骤，我这里将不会提供很多细节。`fork` 函数是异步迭代器转换函数，在每一项线程中运行其参数。它虽然不简单，但很通用，在许多情景下都可用。比如，我会在下面的部分使用它递归获取一个树状图。

#### 性能

异步生成器开销比状态管理库小得多。但是也有许多方法会导致性能问题，例如，消息过度泛滥。但是也有许多毫不费力的方法来提高性能。

在前一个例子中，对 `ReactDom.render` 进行了无效调用。这显然是个效率问题，并且有一个简单的解决方案。在每一次调度事件之后，通过发送另一个类型为 “FLUSH” 的消息快速解决它。React 渲染只会在它收到这条消息后运行。中间步骤可以产生他们之间需要的任何东西。

这种方法的另一个令人敬畏的方面是，在问题出现之前，你可能不需要担心性能问题。一切都是在小型自治阶段构建的。它们很容易重构，甚至没有重构 - 许多性能问题可以通过在步骤管道中添加另一个通用状态来解决，例如批处理，确定优先级，保存中间数据等。 

例如，在构建的演示中，React 元素被保存在局部变量中，React 可以重用它们。 变化从根向叶传播，因此不需要重写 `shouldComponentUpdate` 来优化。

#### 测试

与 Redux reducer 测试相比，生成器适合暗箱测试策略。测试无法访问当前状态。尽管如此，它们写起来非常简单。 使用 Jest 快照，测试可以是使用快照比较输出，输入内容列表。

```javascript
test("counterControl", async () => {
  expect.assertions(3)
  for await(const i of Counter.mainControl([
         {type:"MENU", value:<span>Menu</span>},
         {type:"VALUE", value:10},
         {type:"CONTROL", value:<span>Control</span>},
         {type:"FLUSH"},
         {type:"VALUE", value: 11},
         {type:"FLUSH"}]))
    if (i.type === "CONTROL")
      expect(renderer.create(i.value).toJSON()).toMatchSnapshot()
})
```

如果你更喜欢将单元测试作为文档策略，那么有很多方法可以创建用于测试的自记录 API 。 比方说，函数 \`eventually\`/ \`until\` 作为传统 BDD 表达式的补充。

#### 持久化状态

Dan Abramov 在 [你可能不需要Redux][6] 文章中描述了 Redux 的另一个动机 - 即提供对状态的访问，它可以被序列化、克隆、差异化、修复等。这可以用于时间旅行、热重装、通用应用程序等。

为此，整个应用程序状态都应该保存在 Redux 存储中。 许多 Redux 应用程序（甚至是 Redux 例子）都将某些状态存储在其存储之外。 这些是组件状态，闭包，生成器或异步函数状态。基于 Redux 的工具无法保持此状态。

当然，单一存储 Redux（即单一数据来源）使程序更简单。 不幸的是，这通常是不可能的。考虑例如分布式应用程序，例如，数据在前端和后端之间共享。

> "哦，你想\*增加一个计数器\*？！祝你好运！" -- 分布式系统文献
> 
> — 林赛·库珀（@lindsey） [2015年5月9日][7]

事件源模式非常适合分布式应用程序。使用生成器，我们可以编写一个代理，将所有传入的消息发送到远端并挂起所有收到的消息。 每个对等体上都可以有单独的管道，或者它们可以是相同的应用程序，但也可以是一些正在运行的进程。 许多配置易于设置，使用和重复使用。

例如 `pipe(task1，remoteTask2，task3)`。这里 `remoteTask2` 可以是代理，也可以在这里定义，比如说，用于调试目的。

每个部分都保持自己的状态，不需要持久化。假设每个任务都是由一个单独的团队实施的，对于状态，他们可以自由地使用任何模型，随时更改它而不必担心破坏其他团队的工作。

这非常适合服务器端渲染。 比如，根据后端的输入，可以有一个特定的高阶函数来缓存结果值。

```javascript
const backend = pipe(
    commonTask1,    
    memo(pipe(         
        renderTask1,         
        renderTask2)),
    commonTask2)
```

这里的 `memo` 高阶函数检查传入的消息，也可能会发现一些计算会被重用。 这可能是服务器端呈现的字符串，下一个阶段使用它构建HTTP响应。

渲染任务可以运行异步操作，请求远程操作。为了更好的用户体验，我们希望页面加载速度更快。为了加快初始页面加载速度，应用程序可以在加载组件的同时，延迟显示一些加载占位符而不是组件，直到它准备就绪。在页面上有一些具有不同加载时间的组件会导致页面重新布局，从而导致用户体验变差。

React 团队最近宣布了 Suspense API 来解决这个问题。它是嵌入到渲染器中的 React 的扩展。使用本文中的反向组件，不需要 Suspense API，解决方案更简单，它不是UI框架的一部分。

假设应用程序使用动态导入来延迟加载控件，这可以通过以下方式完成：

```javascript
yield {type:”LAZY_CONTROL”}
yield {type:”CONTROL”, value: await import(“./lazy_component”)}
```

还有另一个通用的下一个阶段。 它收集所有 “LAZY_CONTROL” 消息，等待在接收到所有 “CONTROL” 消息或阈值时间间隔之后。 之后，它会使用加载的控件或加载指示器占位符发出 “CONTROL” 消息。 所有下一次更新也可以使用一些特定的超时进行批处理，以最大限度地减少重新布局。

某些生成器还可以对消息进行重新排序，以赋予动画比服务器数据更新且更高的优先级。 我甚至不确定是否需要服务器端框架。 微型生成器可以根据URL，身份验证会话等将初始HTTP请求转换为消息或线程。

#### 函数式编程

常用的状态管理工具具有FP背景。 由于强制性的 `for-of/switch/break` 声明，本文中的代码与 JavaScript 中的 FP 看起来不一样。 FP 中也有相应的概念。 这就是所谓的 `单子符号`。 例如，它们在 Haskell 中的用途之一就是解决诸如 React 组件属性钻孔之类的问题。

为了使这个故事实用，在这里我不偏离主题，还有另一篇文章 [使用Generators作为副作用的语法糖][8]。

#### Effectful.js

[Effectful.js][9] 是一个 babel 预设，不使用任何 JavaScript 语法扩展，实现对任何函数式编程的无符号工作。它还通过 [es-persist][10] 库中的参考实现来支持状态持久性。例如，这可以用于将上述所有异步生成器示例转换为纯函数。

状态持久性不是该工具的主要目标。它用于更高级别的业务逻辑描述。但是，该工具是抽象的，具有许多用途。我会尽快写关于他们的文章。

这是 GitHub上 的 [摘要样本][11]，具有上述所有功能以及自动撤消/重做功能，并将其完整状态存储在 `localStorage` 中。 这是 [运行编译][12] 版本（它写入浏览器的本地存储，但没有信息发送到服务器端）。我不会在本文中提供很多细节，它是关于没有依赖的异步生成器的，但是我认为代码很容易阅读。例如，查看 [undoredo.js][13]，以简单了解时间旅行实现细节。

原始样本几乎不需要任何更改，我只替换了不可序列化的 Promises，使用了来自 “es-persist” 的相应函数，并使用了来自同一库的 `R.bind` 函数的调用来代替了闭包。EffectfulJS 工具链还有另一个编译器，可以使所有功能（包括闭包）序列化，但在本示例中为了使其更简单并未使用它。

故事只是对该技术的简要说明。我已经使用了几年了，很高兴它提供了改进。试试吧，我相信你也会喜欢它。有很多东西需要深入描述。敬请关注！

[1]: https://redux.js.org/introduction/motivation
[2]: https://zh.wikipedia.org/zh-cn/%E5%8F%AF%E6%A8%82%E5%8A%A0%E6%9B%BC%E9%99%80%E7%8F%A0%E5%99%B4%E7%99%BC%E7%8F%BE%E8%B1%A1
[3]: https://github.com/reduxjs/redux/blob/master/examples/counter-vanilla/index.html
[4]: https://redux.js.org/introduction/three-principles
[5]: https://medium.com/dailyjs/decoupling-business-logic-using-async-generators-cc257f80ab33
[6]: https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367
[7]: https://twitter.com/lindsey/status/575006945213485056?ref_src=twsrc%5Etfw
[8]: https://medium.com/@vitaliy.akimov/using-generators-as-monads-do-notation-8600c53648cf
[9]: https://effectful.js.org/
[10]: https://github.com/awto/effectfuljs/tree/master/packages/es-persist
[11]: https://github.com/awto/effectfuljs
[12]: https://effectful.js.org/demo/alternative/
[13]: https://github.com/awto/effectfuljs/blob/master/samples/persist-counters/undoredo.js
