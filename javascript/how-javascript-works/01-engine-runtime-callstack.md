## javascript是如何工作的：引擎、运行时和调用栈的概述

> 原文： [How JavaScript works: an overview of the engine, the runtime, and the call stack](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf)

随着 JavaScript 的越来越流行，促使团队在多个栈上都需要它支持 - 前端、后端、混合应用程序、嵌入式设备等等。

这篇文章是这个系列的第一篇，目的是深入研究 JavaScript 以及它真正是如何工作的：我们认为了解 JavaScript 的构建块，并且知道它是如何一起运行的，那么你将会写出更好的代码和应用程序。我们也将分享一些我们在使用构建 [SessionStatck](https://www.sessionstack.com/?utm_source=medium&utm_medium=source&utm_content=javascript-series-post1-intro)时候的一些经验规则，一个轻量的应用程序，为了保持竞争力，必须是健壮和高性能的。

正如`GitHut stats`[GitHut - Programming Languages and GitHub](http://githut.info/)统计数据展示， JavaScript 是 GitHub 上 Repositories 最活跃和 push 最多的语言。在其他的类别中，它也没有落后。

![](https://cdn-images-1.medium.com/max/800/1*Zf4reZZJ9DCKsXf5CSXghg.png)

[获取最新的GitHub language stats]([GitHut 2.0](https://madnight.github.io/githut/#/pull_requests/2018/1))

如果项目越来越多的依赖  JavaScript，这意味着开发者为了构建一个令人惊叹的软件，必须利用语言提供的一切，并且对于生态系统内部有着深入的理解。

事实证明， 很多开发者每天都在使用 JavaScript，但是并不知道它们内部发生了什么。

### 概述
几乎每个人都已经听说过V8引擎的概念，并且很多人知道 JavaScript 是单线程的或者它是使用回调队列的。

在这篇文章中，我们将会详细讲述所有的概念，并且解释 JavaScript 是如何真正运行的。在知道这些字节之后，你将会适当的利用提供的APIS写出更好的，非阻塞的应用程序。

如果你是 JavaScript 新手，这篇文章会帮助你理解为什么 JavaScript 和其他语言比较起来如此“怪异”。

如果你是一个有经验的 JavaScript 开发者，希望它可以让你对每天都在使用的 JavaScript 运行时是如何真正关注的有一些新的理解。

### JavaScript 引擎

最流行的 JavaScript 引擎的例子是谷歌的 V8 引擎。V8被用于 Chrome 和 Node.js 内部。下面是一个简单的视图示例：
![](https://cdn-images-1.medium.com/max/800/1*OnH_DlbNAPvB9KLxUCyMsA.png)

引擎主要由两个部分组成：
* 内存堆 — 这是内存分配的地方
* 调用栈 — 这是代码执行的栈


### 运行时
有很多浏览器的APIS被 JavaScript  开发者使用过（例如：setTimeout）。然而这些APIS，并不是引擎提供的。

那么，他们从哪里来？

事实证明这真的有一些复杂。

![](https://cdn-images-1.medium.com/max/800/1*4lHHyfEhVB0LnQ3HlhSs8g.png)

因此，我们有引擎，但实际上有更多。我们有浏览器提供的 Web APIs，例如：DOM，AJAX，setTimeout等。

接下来，我们有非常流行的 **event loop** 和 **callback queue**。

### 调用栈

JavaScript 是一个单线程编程语言，这意味着它指引一个单一的调用栈。因此在同一时间它只能做一件事情。

调用栈是一种记录我们正在程序什么地方的数据结构。如果我们进入一个函数，我们就将这个函数放在栈的顶部。如果我们从一个函数中返回，我们将从这个栈顶弹出。这就是这个栈做的所有事情。

我们看个例子。请看下面的代码：

```javascript
function multiply(x, y) {
    return x * y;
}
function printSquare(x) {
    var s = multiply(x, x);
    console.log(s);
}
printSquare(5);
```


当引擎开始执行这些代码的时候，调用栈是空的。然后，按照下面的步骤进行：
![](https://cdn-images-1.medium.com/max/800/1*Yp1KOt_UJ47HChmS9y7KXw.png)

调用栈的每一项被称为**栈帧(Stack Frame)**。

而且这是实际上当抛出一个异常的时候栈追踪是如何构建的 — 这基本就是异常发生时候调用栈的状态。请看下面的代码：
```javascript
function foo() {
    throw new Error('SessionStack will help you resolve crashes :)');
}
function bar() {
    foo();
}
function start() {
    bar();
}
start();
```


如果这是在Chrome中运行（假设这段代码在一个叫作foo.js的文件中），接下来的栈追踪将会产生：
![](https://cdn-images-1.medium.com/max/800/1*T-W_ihvl-9rG4dn18kP3Qw.png)

“**爆栈**” —— 当调用栈达到最大的时候就会发生。并且这种情况很容易发生，尤其是你没有对你的代码做全面的测试。看下面简单的代码：
```javascript
function foo() {
    foo();
}
foo();
```

当引擎开始执行这段代码，它一开始就调用函数”foo“。然后这个函数递归调用本身没有任何中止的条件。因此在每一次执行的时候，同样的函数一次次的被添加到调用栈。看起来就像下面这样：
![](https://cdn-images-1.medium.com/max/800/1*AycFMDy9tlDmNoc5LXd9-g.png)

然而，在某个点上，调用栈中函数的调用书超过了调用栈实际的大小，那么浏览器决定采取行动，抛出一个异常，看起来就是这样：
![](https://cdn-images-1.medium.com/max/800/1*e0nEd59RPKz9coyY8FX-uw.png)

在单线程中运行代码可能比较容易，因为你不需要处理多线程环境中的负责场景 —— 例如： 死锁。

同样在单线程中运行比较容易受到限制。由于 JavaScript 有一个单线程调用栈，**当运行变得缓慢的时候发生了什么？**

### 并发 & 事件循环
当你在调用栈中，有函数调用需要花费大量的时间才能够被处理，它到底发生了什么？例如，想象下你需要在浏览器中使用 JavaScript 进行复杂图片的转换。

你可能会问 — 为什么这会是一个问题？问题是调用栈有函数在运行，浏览器实际不能做任何事情 — 它被阻塞了。这意味着浏览器不能渲染，它不能运行任何代码，它挂了。如果你希望你的app有一个流畅的UI那么问题就产生了。

而且这不是唯一的问题。一旦你的浏览器开始处理调用栈中的很多任务，它将在很长时间内无法响应。大多数浏览器通过抛出一个错误来才去行动，问你是否需要中止网页。

![](https://cdn-images-1.medium.com/max/800/1*WlMXK3rs_scqKTRV41au7g.jpeg)

现在，这并不是一个好的用户体验，是不是？

因此，如何在不阻塞UI，并且浏览器有响应的情况下执行大量代码？解决方案就是**异步回调**。

这个将会在“浏览器是如何工作的”第二部分“[V8引擎和5个如何优化代码的tips](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)”中详细介绍。

与此同时，如果你在你的应用程序中有很难重现或者很难理解的问题的时候，看下[SessionStack](https://www.sessionstack.com/?utm_source=medium&utm_medium=blog&utm_content=Post-1-overview-outro) 。SessionStack 记录你的web应用程序的所有东西：所有的DOM变化，用户交互，JavaScript异常，栈追踪，失败的网络请求和调试信息。

使用 SessionStack ，你可以重现在你的web应用中的一切问题就像视频一样，并且可以看到你的用户交互信息。

这里有一个免费的计划，不需要信用卡。[现在开始就开始吧]([Record and Reproduce Errors in JavaScript Apps | SessionStack](https://www.sessionstack.com/?utm_source=medium&utm_medium=blog&utm_content=Post-1-overview-getStarted)。

![](https://cdn-images-1.medium.com/max/800/1*kEQmoMuNBDfZKNSBh0tvRA.png)







