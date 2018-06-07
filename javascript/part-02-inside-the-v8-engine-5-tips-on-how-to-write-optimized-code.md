## javascript是如何工作的：V8引擎内部机制及如何编写优化代码的5个诀窍

>原文：[https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e](https://blog.sessionstack.com/how-javascript-works-inside-the-v8-engine-5-tips-on-how-to-write-optimized-code-ac089e62b12e)

几周以前我开始了一个系列，目的是深入研究 JavaScript 以及它真正是如何工作的：我们认为了解 JavaScript 的构建块，并且知道它是如何一起运行的，那么你将会写出更好的代码和应用程序。

系列的[第一篇文章](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf)重点介绍了引擎、运行时和调用栈。第二篇文章将深入谷歌的V8 JavaScript 引擎的内部部分。我们还将提供一些关于如何编写更好的 JavaScript 代码的技巧 -- 在我们的 SessionStack  开发团队构建产品时候的最佳实践。

### 概述

一个 **JavaScript 引擎**是一个执行 JavaScript  代码的程序或者解释器。一个JavaScript 引擎可以实现成一个标准的解释器，也可以是一个实时编译器，它以某种像是将 JavaScript  编译为字节码。

这是一个正在实现 JavaScript 引擎的热门项目列表：
* **[V8]([Chrome V8 - Wikipedia](https://en.wikipedia.org/wiki/V8_%28JavaScript_engine%29))** — 开源，谷歌开发，使用C++编写
* **[Rhino]([Rhino (JavaScript engine) - Wikipedia](https://en.wikipedia.org/wiki/Rhino_%28JavaScript_engine%29))** — 由Mozilla基金会管理，开源，完全使用Java开发
* **[SpiderMonkey]([SpiderMonkey - Wikipedia](https://en.wikipedia.org/wiki/SpiderMonkey))** — 第一个JavaScript引擎，在当时支持  Netscape Navigator， 今天支持Firefox
* **[JavaScriptCore](https://en.wikipedia.org/wiki/WebKit#JavaScriptCore)** -- 开源，作为Nitro销售，由苹果公司为Safari开发
* **[KJS]([KJS (software) - Wikipedia](https://en.wikipedia.org/wiki/KJS_(software)))** — KDE 引擎起初由 Harri Porten 为KDE Konqueror web 浏览器项目而开发
* **[Chakra (JScript9)]([Chakra (JScript engine) - Wikipedia](https://en.wikipedia.org/wiki/Chakra_%28JScript_engine%29))** — IE浏览器
* **[Chakra (JavaScript)]([Chakra (JavaScript engine) - Wikipedia](https://en.wikipedia.org/wiki/Chakra_%28JavaScript_engine%29))** — 微软 Edge
* **[Nashorn]([Nashorn (JavaScript engine) - Wikipedia](https://en.wikipedia.org/wiki/Nashorn_%28JavaScript_engine%29))** — 
作为JDK的一部分开源，由Oracle的Java语言和工具组编写
* **[JerryScript]([JerryScript - Wikipedia](https://en.wikipedia.org/wiki/JerryScript))** — 物联网的轻量级引擎

### 为什么要创建V8引擎？

V8引擎是由谷歌使用C++编写创建的开源引擎。这个引擎被用于谷歌的Chrome内部。然而，与其他引擎不同的是，V8页用于流行的Node.js运行时。

![](https://cdn-images-1.medium.com/max/800/1*AKKvE3QmN_ZQmEzSj16oXg.png)

V8是第一个为了提高 JavaScript 在 web 浏览器的执行的性能优化而设计的。为了提高速度，V8将  JavaScript 代码转换为效率更改的机器代码，而不是使用内部解释器。它通过实现**JIT(Just-In-Time) compiler** 将 JavaScript 代码在运行的时候转为机器代码，就像许多现代的JavaScript引起做的一样，例如 SpiderMonkey 或者 Rhino（Mozilla）。这里的主要区别是V8不产生字节码或者任何其他中间代码。

### V8以前有两个编译器

在V8 5.9版本发布之前（今年早些时候发布），该引擎使用了两个编译器：

* full-codegen — 一个简单的并且非常快速的编译器，它生成相对缓慢的机器代码。
* Crankshaft — 一个更复杂（Just-In-Time）优化编译器，产生高性能的代码。

V8引擎内部使用几个线程：

* 主线程执行您期望的任务：获取代码，编译它，并且执行它。
* 还有一个单独的线程进行编译，因此主线程可以在它优化代码的时候持续执行
* 一个Profiler线程将告诉运行时，那些方法花费了比较大量的时间，因此 Crankshaft可以优化他们
* 有几个线程处理垃圾回收扫描

当第一次执行 JavaScript 代码，V8 利用**full-codegen** 直接将解析的JavaScript代码翻译成机器代码而不进行任何转换。这使得开始运行机器代码**非常快**。注意，V8不适用任何中间字节码表示，这样就省去了解释器的需要。

当你的代码运行了一段时间，Profiler进程以及收集到了足够多的数据知道哪个方法需要优化。

下一步，**Crankshaft**优化在另一个线程开始。它将 JavaScript抽象语法树翻译成高级经验单元分配（SSA）称之为**Hydrogen**，并且尝试去优化 Hydrogen 结构。大多数优化都在在这个级别完成的。


### 内联

第一个优化就是尽可能多的内联代码。内联是被调用的函数使用函数体替换调用的地方（调用函数所在的代码行数）。这个简单的步骤让下面的优化变的更加有意义。
![](https://cdn-images-1.medium.com/max/800/0*RRgTDdRfLGEhuR7U.png)

### 隐藏类

JavaScript 是一个基于原型的语言：他们没有**类**，并且对象是使用克隆程序创建的。JavaScript 也是一种动态编程语言，这也意味着在对象实例化之后可以很容易的给对象添加或者删除属性。

大多数的 JavaScript 解释器都是会用类似字典的结构（基于哈希函数），将对象属性值的位置存储在内存中。这种结构使得在 JavaScript 中获取属性的值比在非定要编程语言想 Java 或者 C# 中更加昂贵。在Java中，所有的对象属性都是在编译钱由一个固定的对象决定的，并且在运行时不能动态的添加或者删除（当然，C#有动态类型，这是另外的一个话题）。作为结果，这些属性的值（或者指向属性的指针）可以被存储为连续的缓冲，在内存中有一个固定的偏移。基于属性的类型，属性偏移的长度可以被轻松的确定，而在运行时可以更改属性类型的JavaScript中是不可能的。

由于使用字典在内存中查找对象属性的效率非常低，V8使用另一个不同的方法代替：**隐藏类**。隐藏类和在Java中使用的固定对象布局的工作非常相似，此外他们是在运行时创建的。现在，我们看看他到底是什么样子：
```javascript
function Point(x, y) {
    this.x = x;
    this.y = y;
}
var p1 = new Point(1, 2);
```

一旦 “new Point(1, 2)”被调用，V8将会创建一个叫做“C0”的隐藏类。
![](https://cdn-images-1.medium.com/max/800/1*pVnIrMZiB9iAz5sW28AixA.png)

没有为Point定义任何属性，所以“C0”是空的。

一旦第一句“this.x = x;”被执行（在“Point”函数中），V8将会创建第二个隐藏类叫做“C1”，它是基于“C0”来创建的。“C1”表示在内存中 x 属性可以被找到的位置（关联到对象的指针）。在这个示例中，“x”存储在偏移量0的位置，这也意味着在内存中将指针对象看做连续的缓冲器，那么第一个偏移也会指到对应的属性“x”。V8也会使用“类转换”来更新“C0”状态，如果一个“x”属性被添加到指针对象，隐藏类将会从“C0”切换到“C1”。指针对象的隐藏类现在是“C1”。

![](https://cdn-images-1.medium.com/max/800/1*QsVUE3snZD9abYXccg6Sgw.png)

*每次给对象添加新的属性，旧的hidden class就会更新转换路径到新的hidden class。Hidden class的转换是非常重要的，因为他们允许在同样创建方式的对象中共享。如果两个对象共享一个hidden class，那么相同的属性会添加到他们，转换会确保两个对象接收相同的hidden class和它所有优化过的代码。*

这个过程在语句“this.y = y”被执行的时候会重复（再一次，在Point函数内部，在”this.x = x”语句后面）。

一个叫作“C2”的hidden class被创建，一个类转换被添加到“C1”，如果属性“y”被添加到Point对象（它已经包含属性“x”），然后hidden class需要被变成“C2”，并且指针对象的hidden class会被更新到“C2”.


![](https://cdn-images-1.medium.com/max/800/1*spJ8v7GWivxZZzTAzqVPtA.png)

Hidden class的转换依赖于给对象添加属性的顺序。看下面的代码片段：

```javascript
function Point(x, y) {
    this.x = x;
    this.y = y;
}
var p1 = new Point(1, 2);
p1.a = 5;
p1.b = 6;
var p2 = new Point(3, 4);
p2.b = 7;
p2.a = 8;
```


现在，你可能会假设P1和P2有相同的hidden class并且转换已经被使用。当然，不是真的。对于“p1”，首先添加属性”a“，然后是属性”b“。然而，对于”p2“，首先是”b“，然后是”a“。因此，”p1“和”p2“结果是有着不用的hidden class不同的转换路径。在这个情况下，最好以相同的殊勋初始化动态属性，以便于hidden class可以被重复利用。

### 内联缓存

V8利用另一种优化动态类型语言的技术，叫做内联缓存。内联缓存依赖于观察在同一类型的对象上重复调用同一方法。想深入了解内联缓存可以点击[这里](https://github.com/sq/JSIL/wiki/Optimizing-dynamic-JavaScript-with-inline-caches).

我们去了解一些内联缓存的常规性概念（如果你没有时间去深入上面的解释）。

那么它是如何工作的？V8维护在最近方法调用中作为参数传递的对象类型的缓存，并且利用这些信息判断未来防范调用参数的对象类型。如果V8能够很好地假定传递给方法的对象类型，它可以绕过如何访问对象属性的计算过程，而是使用以前从hideden class中查找之前对象存储的信息。

那么hidden class和inline caching的概念是如何关联起来的？每次调用特定对象方法的时候，V8引擎必须从对象的hidden class中查找，以便访问特定属性的偏移。在同一hidden class的相同方法调用成功两次以后，V8忽略hidden class的查找，并且简单地将偏移量添加到对象指针本身。对于未来该方法的调用，V8引擎假设hidden class没有发生改变，并且使用之前查找存储的偏移直接跳到指定属性的内存地址。这极大地提高了执行速度。

内联缓存也是同一类型对象共享hidden class的重要原因。如果你创建两个相同类型的对象有着不同的hidden class（就像我们之前的例子），V8不会使用内联缓存，即使两个对象有着相同的类型，相应的hidden class给不同的属性分配的偏移量不一样。

![](https://cdn-images-1.medium.com/max/1600/1*iHfI6MQ-YKQvWvo51J-P0w.png)

*这两个底线基本是一样的，但是创建“a”和“b”属性的顺序是不一样的。*

### 编译为机器码

一旦Hydrogen的结构被优化，曲轴就会将它降到一个低层的表示叫做Lithium。大多数Lithium的实现都有着特殊的结构。寄存器分配发生在这一层级。

最后，Lithium被编译成机器码。然后发生的事情叫做OSR：堆栈上的替换。在我们开始编译和优化一个明显运行时间长的方法，我们可能会运行它。V8不会忘记它缓慢的执行了什么，而是从优化的版本重新开始。相反，它会转换我们所有的上下文（栈，寄存器）以便于我们在执行的过程中切换为优化版本。这是一个非常复杂的任务，考虑到在其他优化中，V8最初已经将代码内联起来。V8并不是唯一能够做到这一点的引擎。

有一种保护操作叫做去优化，做相反的转换，并且返回非优化的代码，以防引擎做的假设不再成立。

### 垃圾回收

对于垃圾回收，V8使用传统的分代方法标记和扫来且清理老一代的代码。标记阶段应该停止 JavaScript 的执行。为了控制GC的成本和让运行更加稳定，V8使用增量标记：它不是遍历所有的堆，尝试标记每个可能的对象，它只是遍历堆的一部分，然后恢复正常运行。下一次GC停止将会从上一次堆遍历停止的地方开始。这允许在正常的执行中进行非常短的暂停。如之前提到的一样，扫描阶段将由独立的线程去处理。

### Ignition 和 TurboFan

随着2017早些时候V8 5.9版本的发布，引入了一个新的执行管道。这个新的管道在真实的JavaScript应用程序中不仅提高了性能而且节省了内存。

新的管道建立在V8解释器[Ignition](https://github.com/v8/v8/wiki)和V8最新优化编译[TurboFan](https://github.com/v8/v8/wiki/TurboFan)之上。

你可以在[这里](https://v8project.blogspot.bg/2017/05/launching-ignition-and-turbofan.html)查看V8团队关于这个主题的博客。

由于5.9版本V8的出现，V8不再使用full-codegen 和 Crankshaft（自2010年以来V8所用的技术）来执行JavaScript，因为V8团队一直在努力跟着新的JavaScript语言的新特性，并且这些特性需要优化。

这意味着在未来V8将会有更加简单的和更稳定的架构。

![](https://cdn-images-1.medium.com/max/1600/0*pohqKvj9psTPRlOv.png)

这些提升只是开始。新 Ignition 和 TurboFan 管道为下一步优化铺平了路，这将在未来几年内促进 JavaScript 性能提升，并且缩小V8在Chrome和Node.js中的比重。

最后，这里有一些关于如何编写更优化的，更好的JavaScript的建议和技巧。当然，从上面的内容不难得出这些，但是这里为了方便还是给出一个总结：

### 如何编写更优化的JavaScript

1. 对象属性的顺序：始终以相同的顺序去吃实话对象，以便于hidden class，和随后的优化代码，可以共享。  
2. 动态属性：对象初始化以后添加属性会强制改变hidden class，并且减慢为上一个hidden class优化的任何方法。相反，在构造函数中分配对象的所有属性。  
3. 方法：重复执行相同方法的代码比只执行一次的不同方法（在内联缓存中）快。  
4. 数组：避免key值不是自增增长数字的稀疏数组。元素不全的稀疏数组是一个**哈希表**。这样数组的元素访问起来更加昂贵。另外，避免预分配大数组。最好随着发展就增加。最后，不要删除数组中的元素。这会导致key值稀疏。  
5. 标记值：V8使用32位标识对象和数字。它使用1位去标记是否是对象（flag=1）或者是一个整数（flag=0）叫做SMI(小整数)因为它是31位。然后，如果一个数字值大于31位，V8将会包装数字，将它转为double，并且创建一个新对象把数字放在里面。尽可能的使用31位有标记的数字去避免昂贵的JS对象包装操作。