## 理解 node.js 中的 module.exports 与 exports

> 原文链接: [https://www.sitepoint.com/understanding-module-exports-exports-node-js/](https://www.sitepoint.com/understanding-module-exports-exports-node-js/)

作为一个开发者，我们经常会遇到需要使用不熟悉的代码的情况。
在这个过程中遇到一个问题：我需要花费多少时间去理解这些代码，明白如何使用?
一个典型的回答就是：先让我可以开始coding，等到时间允许再去做深入研究。
接下来我们将对 module.exports 和 exports 在 node.js中的使用有一个更好地了解。

Note: 这篇文章包括了 node 中 module 的使用。如果你想了解浏览器内部 modules 的使用，可以参考这面这篇文章：
[Understanding JavaScript Modules: Bundling & Transpiling](https://www.sitepoint.com/javascript-modules-bundling-transpiling/)

<!-- more -->

#### What is a Module？

一个模块就是将文件中相关的代码封装为一个代码块。
创建一个module,可以理解为将所有相关的方法挪到一个文件中。
我们使用一个node.js的应用程序来说明一下这个观点。
创建一个名叫 greetings.js 的文件，其中包含下面两个方法：
```javascript
// greetings.js
sayHelloInEnglish = function() {
  return "Hello";
};

sayHelloInSpanish = function() {
  return "Hola";
};
```
#### Exporting a Module

为了 greetings.js 公共逻辑增加的时候，其封装的代码可以在其他文件中使用。所以我们
重构一下 greetings.js 来达到这个目的。为了更好地理解这个过程，我们分为3步：

1） 想象一下有这么一行代码在 greetings.js 的第一行：

```javascript
// greetings.js
var exports = module.exports = {};
```

2） 把greetings.js中的方法赋值给exports对象在其他文件中使用：

```javascript
// greetings.js
// var exports = module.exports = {};

exports.sayHelloInEnglish = function() {
  return "HELLO";
};

exports.sayHelloInSpanish = function() {
  return "Hola";
};
```
在上面的代码中，我们可以使用 module.exports 替换 exports达到相同的结果。
这看起来似乎有些困惑，请记住：exports  和 module.exports引用的是同一对象。

3） 此时 module.exports 是这样的：

```javascript
module.exports = {
  sayHelloInEnglish: function() {
    return "HELLO";
  },

  sayHelloInSpanish: function() {
    return "Hola";
  }
};
```

#### Importing a Module

我们在 main.js 中 require greetings.js 的公开接口。这个过程有以下三个步骤：

1）关键词 require 在 node.js 中用于导入模块，即所获取模块的 exports 对象。
我们可以想到它是这么定义的：

```javascript
var require = function(path) {

  // ...

  return module.exports;
};
```
2) 在 main.js 中 require greetings.js
```javascript
// main.js
var greetings = require("./greetings.js");
```

上面的代码等同于：
```javascript
// main.js
var greetings = {
  sayHelloInEnglish: function() {
    return "HELLO";
  },

  sayHelloInSpanish: function() {
    return "Hola";
  }
};
```

3) 现在我们可以在 main.js 中使用greetings访问 greetings.js 中公开的方法就像获取它的属性一样。

```javascript
// main.js
var greetings = require("./greetings.js");

// "Hello"
greetings.sayHelloInEnglish();

// "Hola"
greetings.sayHelloInSpanish();

```

#### Salient Points 重点

require 返回一个 object ，该对象引用了 module.exports 的值。
如果开发者无意或有意的将 module.exports 赋值给另外一个对象，
或者赋予不同的数据结构，这样会导致原来的 module.exports 对象
所包含的属性失效。

看一个复杂的示例去说明这个观点。

```javascript
// greetings.js
// var exports = module.exports = {};

exports.sayHelloInEnglish = function() {
  return "HELLO";
};

exports.sayHelloInSpanish = function() {
  return "Hola";
};

/*
 * this line of code re-assigns
 * module.exports
 */
module.exports = "Bonjour";
```

在 main.js 中require greetings.js

```javascript
// main.js
var greetings = require("./greetings.js");
```

此时，和之前并没有任何变化。我们将greetings.js中公开的方法
赋值给greetings变量。

当我们试图调用sayHelloInEnglish和sayHelloInSpanish结果显示为
module.exports 重新赋值给一个新的不同于默认值的数据格式。

```javascript
// main.js
// var greetings = require("./greetings.js");

/*
 * TypeError: object Bonjour has no
 * method 'sayHelloInEnglish'
 */
greetings.sayHelloInEnglish();

/*
 * TypeError: object Bonjour has no
 * method 'sayHelloInSpanish'
 */
greetings.sayHelloInSpanish();

```

为了清楚地知道这个错误原因，我们将greetings的结果打印出来：

```javascript
// "Bonjour"
console.log(greetings);
```

在这个点上，我们试着在 module.exports 抛出来的字符串"Bonjour" 去调用 sayHelloInEnglish 和 sayHelloInSpanish
方法，换句话说，我们永远也不会引用到 module.exports 默认输出object里面的方法。
#### Conclusion 总结

importing 和 exporting 模块在 node.js 中是一个随处可见的任务。
我希望 exports 和 module.exports之间的差异更加清晰。
此外，如果将来你遇到调用公共方法错误的时候，我希望你可以对这些
错误的原因有一个更好地理解。
