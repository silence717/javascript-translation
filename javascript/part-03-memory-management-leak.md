##  javascript是如何工作的：内存管理和如何处理4种常见的内存泄漏

>原文： [How JavaScript works: memory management + how to handle 4 common memory leaks](https://blog.sessionstack.com/how-javascript-works-memory-management-how-to-handle-4-common-memory-leaks-3f28b94cfbec)

（忽略前面关于第一第二篇文章的介绍）

在这第3篇文章中，我们将会讨论另一个逐渐被忽视的关键主题 - 内存管理，因为每天使用的编程语言越来越成熟和复杂。
我们也会提供一些在 JavaScript 中如何处理内存泄漏的技巧，就是我们在 SessionStack 中确保 SessionStack 不会内存泄漏，或者不会增加继承在web app中的内存消耗。

### 概述

像C语言这种具有低级内存管理的原始语言，例如`malloc()`和`free()`。开发人员使用这些原始语言明确地给操作系统分配和释放内存。

同时，JavaScript 在创建事物（对象，字符串，等等）的时候分配内存，并且在不再使用的时候“自动”释放内存，这是一个*垃圾回收*的过程。释放资源的这种看起来“自动”的特性是混乱的来源，给 JavaScript （和其他高级语言）开发者一种他们可以不再关心内存管理的错误印象。**这是一个大错误**。

即使在使用高级语言工作的时候，开发者应该理解内存管理（或者最少是基本知识）。有时，自动内存管理存在问题（例如垃圾回收中的bug或者实现限制，等），开发人员必须了解这些问题才能正确处理它们（或者找到适当的解决办法，同时尽量减少成本和代码负担）。

### 内存生命周期

无论你使用哪种编程语言，内存生命周期几乎都是一样的：

![](https://cdn-images-1.medium.com/max/800/1*slxXgq_TO38TgtoKpWa_jQ.png)

以下是对周期中的每一步发生的情况概述：

* **分配内存** - 内存是由操作系统分配的，操作系统允许你的程序使用它。在低级别语言（例如C）中，这是你作为一个开发者应该处理的明确操作。然而，在高级别语言，这是你应该处理的。
* **使用内存** - 这实际上是程序使用之前分配的内存的时候。**读**和**写**操作在代码中分配内存的时候发生。
* **释放内存** - 现在是时候释放你不再需要的整个内存了，这样它可以重新释放并且可再次使用。与**分配内存**一样，在低级别语言中这是明确的。

要快速了解调用栈和内存堆的概念，可以[阅读这个主题的第一篇文章](https://github.com/silence717/javascript-translation/blob/master/javascript/part-01-engine-runtime-callstack.md)。

### 什么是内存？

在直接进入 JavaScript 内存之前，我们将简单地讨论下什么是一般内存，以及它是如何工作的。

在硬件层面，计算机内存是由大量的[触发器](https://en.wikipedia.org/wiki/Flip-flop_%28electronics%29)组成。
每个触发器包含几个晶体管，并且可以存储一个比特。每个触发器通过一个**唯一的标识符**寻址，因此我们可以读取和复写它们。因此，从概念上讲，我们可以认为整个计算机的内存是一组比特数组，我们可以读写他们。

但是作为人类，我们并不擅长使用比特来完成我们所有的思考和算数，我们将它们组成更大的组，它们可以一起来表示数字。8比特被称为1字节。除了字节，还有单词（有时是16，有时是32位）。

内存中存储大量的东西：
1. 所有的变量和所有程序使用的数据。
2. 编程的代码，包括操作系统的。

编译器和操作系统一起工作，为你处理大量的内存管理，但是我们建议你查看一下背后发生的事情。

当你编译你的代码的时候，编译器可以检查原始数据类型，并且提前计算它们需要多少内存。然后在调用**堆栈空间**中将需要的内存分配给程序。分配这些变量的空间叫做堆栈空间，因为在调用函数的时候，他们的内存会添加到现有的内存之上。一旦他们中止，它们将以LIFO（后入先出）的顺序移除。例如，考虑下面的声明：

```javascript
init n; // 4bytes
int x[4]; // array of 4 elements, each 4 bytes
double m; // 8 bytes
```

编译器可以立马看到这些代码需要多大内存
```
4 + 4 * 4 + 8 = 28 bytes.
```

> 这就是它如何处理当前整型和double的大小。大概20年前，整型通常是2bytes，和4字节。你的代码不应该依赖当前基本数据类型的大小。

编译器将插入与操作系统交互的代码，它需要请求当前栈上所需存储变量的大小的字节数。

在上面的例子中，编译器确切地知道每个变量的内存地址。实际上，不管你什么时候书写变量`n`，这将会在内部被翻译成类似于“内存地址 4127963”。

注意到，如果我们试图在这里访问`x[4]`，我们将需要访问与之关联的m。这是因为我们访问了数组上不存在的元素 - 它比实际上分配给数组的最后一个元素`x[3]`多了4字节，并且可能将会最终读取（或者复写）一些m的比特。这将对剩下的程序产生非常不期望的后果。

![](https://cdn-images-1.medium.com/max/800/1*5aBou4onl1B8xlgwoGTDOg.png)

当一个函数调用另一个函数时，每个函数在调用堆栈时都会得到自己的堆栈块。它将其所有的本地变量都保存在这里，并且有一个程序计数器去记住执行过程中的位置。当这个函数调用完成，它的内存块将会用于其他用途。


### 动态分配

不幸的是，当我们不知道一个变量在编译的时候需要多少内存，事情就变得不简单了，假如我们要做下面的事情：

```javascript
int n = readInput(); // reads input from the user
...
// create an array with "n" elements
```

这里，在编译的时候，编译器不知道这个数组需要读书内存，因为它是由user提供的值决定的。

因此，它不能为堆栈上的变量分配空间。相反，我们的程序需要在运行的时候向操作系统申请适当的空间。这个内存是从**堆空间**分配的。静态和动态分配内存的不同可以总结为如下的表格：

![](https://cdn-images-1.medium.com/max/800/1*qY-yRQWGI-DLS3zRHYHm9A.png)

要完全理解动态内存分配是如何工作的，我们需要花费一些时间在**指针**上，这可能有点偏离了本文的主体。如果你有兴趣了解更多，请在评论区让我知道，我们可以在以后的文章中再去深入了解这些细节。

### 在 JavaScript 中分配

现在我们将解释第一步在 JavaScript 中内存分配是如何工作的。

JavaScript 减轻了开发者内存分配的责任 - JavaScript 除了声明之外，自己做了这部分工作。

```javascript
var n = 374; // allocates memory for a number
var s = 'sessionstack'; // allocates memory for a string 

var o = {
  a: 1,
  b: null
}; // allocates memory for an object and its contained values

var a = [1, null, 'str'];  // (like object) allocates memory for the  array and its contained values
                           
function f(a) {
  return a + 3;
} // allocates a function (which is a callable object)

// function expressions also allocate an object
someElement.addEventListener('click', function() {
  someElement.style.backgroundColor = 'blue';
}, false);
```

一些对象里的函数调用也会导致内存分配：
```javascript
var d = new Date(); // allocates a Date object

var e = document.createElement('div'); // allocates a DOM element
```

方法可以分配新的值或者对象：
```javascript
var s1 = 'sessionstack';
var s2 = s1.substr(0, 3); // s2 is a new string
// Since strings are immutable, 
// JavaScript may decide to not allocate memory, 
// but just store the [0, 3] range.

var a1 = ['str1', 'str2'];
var a2 = ['str3', 'str4'];
var a3 = a1.concat(a2); 
// new array with 4 elements being
// the concatenation of a1 and a2 elements
```

### 在 JavaScript 中使用内存

一般的，在 JavaScript 中使用分配内存，就是读写。

这可以通过给一个变量或者对象赋值，甚至是给函数传递一个参数来完成读或写。

### 当内存不再需要的时候释放

大多数的内存管理问题都出在这个阶段。

这里最困难的任务就是计算出分配的内存不再需要。它通常依赖开发者决定再程序中哪块内存不再需要并且释放它。

高级语言把嵌入一块叫作**垃圾回收**的软件，它的工作就是追踪内存分配，并且使用它找到哪块分配的内存不再被需要，在这种情况下自动释放。

不幸的是，这个过程是不准确的，因为一般的问题是知道哪块内存被需要是不可判断的（使用算法解决不了）。

大多数垃圾回收工作是通过收集哪块内存是不会再被访问的，例如：指向它的所有变量都超过了作用域。然而，它可以收集到一些内存空间也是不准确的，因为在作用域中很可能一直有一个变量指向它的内存地址，虽然它从未被再次访问。

## 垃圾回收

由于找出“内存不再需要”是不可判断的，垃圾回收实现了对一般问题的解决方案的限制。这个部分将解释垃圾回收的主要算法和他们的限制的一些必要概念。

### 内存引用

垃圾回收算法主要的概念依赖之一就是**引用**。

在内存管理的上下文中，如果一个对象可以访问后面的对象（隐式或者显式）则可以说是一个对象引用另一个对象。例如，一个 JavaScript 对象对[原型](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Inheritance_and_the_prototype_chain)（隐式引用）和对它属性值（显式引用）有一个引用。

在这种情况下，“对象”的范围扩展到比常规的 JavaScript 对象更广泛的内容，并且包含函数作用域（或者全局**词法作用域**）。

> 词法作用域定义了如何在嵌套函数中解析变量名：内部函数包含父函数的作用域，即使父函数已经返回。

### 引用 - 计算垃圾回收

这是最简单的垃圾回收算法。一个对象如果没有一个引用指向它，那么认为可以进行“垃圾回收”。

看下面的代码：
```javascript
var o1 = {
  o2: {
    x: 1
  }
};
// 2 objects are created. 
// 'o2' is referenced by 'o1' object as one of its properties.
// None can be garbage-collected

var o3 = o1; // the 'o3' variable is the second thing that 
            // has a reference to the object pointed by 'o1'. 
                                                       
o1 = 1;      // now, the object that was originally in 'o1' has a         
            // single reference, embodied by the 'o3' variable

var o4 = o3.o2; // reference to 'o2' property of the object.
                // This object has now 2 references: one as
                // a property. 
                // The other as the 'o4' variable

o3 = '374'; // The object that was originally in 'o1' has now zero
            // references to it. 
            // It can be garbage-collected.
            // However, what was its 'o2' property is still
            // referenced by the 'o4' variable, so it cannot be
            // freed.

o4 = null; // what was the 'o2' property of the object originally in
           // 'o1' has zero references to it. 
           // It can be garbage collected.
```


### 循环正在制造问题

当出现循环的时候有一个限制。在下面的例子中，创建两个对象并且引用另一个，这样产生了一个循环。函数被调用后，它们将超出作用域，因此他们是无用的，可以被释放的。然而，引用计算算法认为由于两个对象至少被引用一次，因此不能被垃圾回收。

```javascript
function f() {
  var o1 = {};
  var o2 = {};
  o1.p = o2; // o1 references o2
  o2.p = o1; // o2 references o1. This creates a cycle.
}

f();
```
![](https://cdn-images-1.medium.com/max/800/1*GF3p99CQPZkX3UkgyVKSHw.png)

### 标记扫描算法

为了确定对象是否需要，这个算法决定对象是否可访问。

标记扫描算法需要经过这3步：
1. Roots：通常，roots是代码中引用的全局变量。例如在 JavaScript 中，一个可以充当root的全局变量是"window"对象。Node.js中相同的对象是“global”。垃圾回收生成所有roots完整的列表。
2. 算法将会检查roots的所有变量已经他们的children并且标记为活动(意思是，他们不是垃圾)。root访问不到的东西将会被标记为垃圾。
3. 最后，垃圾回收将会释放没有被标记为活动的内存片段，并且将内存返回给操作系统。

![](https://cdn-images-1.medium.com/max/800/1*WVtok3BV0NgU95mpxk9CNg.gif)

这个算法比之前那个好，因为“一个对象零引用”导致这个对象不可访问。相反的情况并不像我们看到的循环引用那样。

截止到2012年，所有的现代浏览器都实现了一个标记扫描垃圾回收。过去几年在JavaScript垃圾回收领域所做的所有优化都是对该算法（标记扫描）的改进实现，而不是哟花垃圾回收算法本身，也不是一个对象是否可访问的目标。

[在这篇文章](https://en.wikipedia.org/wiki/Tracing_garbage_collection),你可以读到关于追踪垃圾回收非常详细的介绍，并且包括标记扫描的优化。

### 循环不再是问题

在上面的第一个例子中，当函数调用返回后，两个对象不再被全局对象的任何东西访问。因此，垃圾回收将会找到无法访问的他们。

![](https://cdn-images-1.medium.com/max/800/1*FbbOG9mcqWZtNajjDO6SaA.png)

即使在两个对象之间有引用，但是从root开始再也无法访问。

### 垃圾回收器的反直觉行为

虽然垃圾回收器非常方便有着自己的一套推导方案。其中之一是非决定的。换句话说，CGs是不可预测的。你无法真正判断何时执行回收。这意味着在某些情况下，程序会使用比它实际需要的更多的内存。在其他情况下，在一些非常敏感的应用程序中短暂停顿是非常明显的。虽然非确定性意味着不能确定什么时候执行回收，大多数GC在分配内存期间实现执行集合传递的通用模式。如果不执行分配，大多数GCs处于空闲状态。考虑以下情况：

1. 执行一组相当大的分配。
2. 大多数这些元素（或者所有）都被标记为无法访问（假设我们将不再需要的引用指向缓存）。
3. 没有更多的执行分配。

在这种情况下，大多数GCs将不会进一步执行回收。换句话说，对于收集器就算有不可访问的变量引用，收集器也不会清理这些。这些严格意义上来说不是引用，但是会导致比通常更高的内存。

### 什么是内存泄漏？

正如内存表示，内存泄漏是应用程序过去使用，但是未来不再需要的内存片段，并且没有返回给操作系统或者空闲内存池。

![](https://cdn-images-1.medium.com/max/800/1*0B-dAUOH7NrcCDP6GhKHQw.jpeg)

编程语言倾向于不同的内存管理方式。然而，确定一个内存片段使用与否是一个[难以确定的问题](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management#Release_when_the_memory_is_not_needed_anymore)。换句话说，只有开发者可以清楚地确定一个内存片段是否可以返回给操作系统。

某些编程语言提供了帮助开发者实现这个的功能。其他则期望开发者清楚的知道一段内存未被使用。维基百科对于内存管理的[手动](https://en.wikipedia.org/wiki/Manual_memory_management)和[自动](https://en.wikipedia.org/wiki/Garbage_collection_%28computer_science%29)有着非常好的文章。

### 4种常见的 JavaScript 内存泄漏

#### 1: 全局变量

JavaScript使用一种有趣的方式处理未声明变量：当一个未声明的变量被引用，那么就在*全局*对象中创建一个新变量。在浏览器，全局对象可能就是`window`，就是这个意思

```javascript
function foo(arg) {
    bar = "some text";
}
```

和下面的效果是一样的

```javascript
function foo(arg) {
    window.bar = "some text";
}
```

假设`bar`的目的是仅引用在foo函数中的变量。然而，如果你不使用`var`来声明它，一个多余的全局变量就被创建。在上面的例子中，这不会造成很大的危害。不过你肯定可以想出一个更具破坏力的场景。

你也可以意外地使用`this`创建全局变量：

```javascript
function foo() {
    this.var1 = "potential accidental global";
}

// Foo called on its own, this points to the global object (window)
// rather than being undefined.
foo();
```

> 你可通过在你的 JavaScript 文件开始添加`'use strict';`去避免这些，它将切换为更加严格的模式去解析 JavaScript,可以阻止意外的创建全局变量。

然而，全局变量确实是一个问题，你的代码通常可能会出现显式的全局变量定义，这是垃圾回收无法收集的。需要特别注意用于零时存储和处理大量信息的全局变量。如果你必须这么使用全局变量来存储数据，当你这么做的时候，必须确保`给它赋值为null或者重新分配`当它完成工作的时候。

#### 2: 被遗忘的定时器和回调

我们使用经常在 JavaScript 中使用的 `setInterval`作为例子。

提供观察器和其他接受回调的工具库通常要确保所有的引用和他们的实例都变成不可访问。不过，下面的代码并不少见：

```javascript
var serverData = loadData();
setInterval(function() {
    var renderer = document.getElementById('renderer');
    if(renderer) {
        renderer.innerHTML = JSON.stringify(serverData);
    }
}, 5000); //This will be executed every ~5 seconds.
```

上面的片段展示了使用不再需要的引用节点和数据的结果。

`renderer`对象可以在某个点被替换或者删除，这使得使用间隔快处理变得冗余。如果这种情况发生，不管是处理器，还是依赖都需要被回收，首先需要停止间隔（记住，它仍然是活动的）。归根结底，`serverData`肯定不会被存储和处理加载数据，也不会被回收。

在使用观察者时，你需要确保使用完他们之后，明确的来删除它们（不管是不再需要观察者，或者对象变成无法访问）。

幸运的是，大多数的现代浏览器都会为你完成这项工作：即使你忘记删除监听，一旦观察对象编程不可访问，他们将会自动回收观察处理程序。在过去，一些浏览器不能处理这些情况（很好的旧IE6）。

尽管如此，一旦对象变的过期了，最佳实践是移除这些观察者。参见下面的例子：

```javascript
var element = document.getElementById('launch-button');
var counter = 0;

function onClick(event) {
   counter++;
   element.innerHtml = 'text ' + counter;
}

element.addEventListener('click', onClick);

// Do stuff

element.removeEventListener('click', onClick);
element.parentNode.removeChild(element);

// Now when element goes out of scope,
// both element and onClick will be collected even in old browsers 
// that don't handle cycles well.
```

你不再需要在使得节点不可访问之前调用`removeEventListener`，因为现代浏览器的垃圾回收可以检测这些循环并且适当的处理他们。

如果你使用`jQuery`的APIs（其他类库和框架也支持），你可以在节点过期之前移除掉这些监听。类库需要确保应用程序即使在低版本的浏览器下运行也不会出现内存泄漏。

#### 3：闭包

JavaScript 开发的一个关键方面是闭包：一个可以访问外部（包围）函数变量的内部函数。由于 JavaScript 运行的实现细节，以下方式很可能会导致内存泄漏：

```javascript

var theThing = null;

var replaceThing = function () {

  var originalThing = theThing;
  var unused = function () {
    if (originalThing) // a reference to 'originalThing'
      console.log("hi");
  };
  
  theThing = {
    longStr: new Array(1000000).join('*'),
    someMethod: function () {
      console.log("message");
    }
  };
};

setInterval(replaceThing, 1000);
```

一旦`replaceThing`被调用，`theThing`得到一个由很大数组和新闭包（`someMethod`）组成的新对象。然而，`originalThing`是由`unused`变量持有的闭包引用的（它是从前一次调用`replaceThing`替换的`theThing`变量）。需要记住的就是一旦为同一父作用域的闭包创建了作用域，那么这个作用域是共享的。


在这种情况下，为闭包`someMethod`创建的作用域与`unused`是共享的。即使`unused`从未被使用，`someMethod`可以通过`replaceThing`外部的作用域`theThing`来使用。并且，由于`someMethod`和`unused`共享闭包作用域，`unused`引用必须强制`originalThing`保持活动（两个闭包之间共享整个作用域）。这就阻止了回收。

在上面的例子中，为闭包`someMethod`创建的作用域与`unused`是共享的，同时`unused`引用`originalThing`。`someMethod`可以通过`replaceThing`外部的作用域`theThing`来使用，尽管实际上`unused`从未被使用。实际上未使用的引用`originalThing`需要保持active状态，因为`someMethod`与`unused`共享作用域。

所有的这些会导致相当大的内存泄漏。当上面的代码一遍又一遍的运行时，你可能会看到内存使用量暴增。当垃圾回收运行的时候它的大小不会缩小。一个有序的闭包列表被创建（在本例中root是`theThing`变量），并且每个闭包作用域都窜第了一个队大数组的间接引用。

这个问题是 Meteor 团队发现的，他们有一个很好的[文章](https://blog.meteor.com/an-interesting-kind-of-javascript-memory-leak-8b47d2e7f156)详细的描述了这个问题。

#### 4：脱离DOM引用

有些情况下，开发者在数据结构中存储DOM节点。假设你希望快速更新表格中的几行内容。如果你在一个字典或者数组中存储了每个DOM的引用，那么将有两个引用指向同一个DOM元素：一个在DOM树中，另一个在字典中。如果你决定删除这些行，你需要记住让这两个引用都不可访问。

```javascript

var elements = {
    button: document.getElementById('button'),
    image: document.getElementById('image')
};

function doStuff() {
    elements.image.src = 'http://example.com/image_name.png';
}

function removeImage() {
    // The image is a direct child of the body element.
    document.body.removeChild(document.getElementById('image'));
    // At this point, we still have a reference to #button in the
    //global elements object. In other words, the button element is
    //still in memory and cannot be collected by the GC.
}
```

此外需要考虑DOM树中的内部或者子节点的引用问题。如果在代码中保留着对单元格（td标签）的引用，并且决定从DOM中删除这个表格，但需要保留对特定单元格的引用，则可能会导致大的内存泄漏。你可能会认为垃圾回收会释放除了那个单元格以外的任何东西。然而实际上并非如此。由于单元格是表格的一个子节点，并且子节点保留对父节点的引用，**这个对表格单元格的单个引用将整个表格保持在内存中**。
