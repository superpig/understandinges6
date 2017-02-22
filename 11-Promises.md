# Promises和异步编程
JavaScript最强大的一方面是很容易处理异步编程。作为为web创建的一门编程语言，JavaScript需要能够响应异步的用户交互，比如从一开始的点击和按键操作。Node.js通过使用回调作为事件的替代方案，进一步普及JavaScript的异步编程。随着越来越多的程序开始使用异步编程，事件和回调不再足够强大去支持开发者想要的每件事。*Promise* 是这个问题的解决方案。

Promise是异步编程的另外一种选择，它们像其他语言中的futures和deferreds一样工作。Promise指定一些代码之后执行(就像使用事件和回调)，而且明确表示代码在它的运行上是成功还是失败。你可以基于成功或者失败链式使用promises，通过这种方式使得你的代码更容易理解和调试。

为了很好地理解promises如何工作，然而理解一些关于构建它们的基本概念是重要的。

## 异步编程的背景
JavaScript引擎建立在单线程事件循环的概念上。*单线程* 表示每次只有一段代码在执行。和其它语言相反，比如 Java 或者 C++，其中线程可以允许多段不同的代码在同一时刻执行。当多段代码可以访问和改变状态时，维持和保护状态是个难题，而且在基于线程的软件中，这经常是bug的来源。

JavaScript引擎每次只能执行一段代码，所以他们需要追踪要运行的代码。这些代码保存在一个*作业队列*中。无论什么时候一段代码准备去执行，它会被添加到这个作业队列中。每当JavaScript引擎完成执行代码，事件循环会执行队列中的下一项作业。*事件循环* 是指在JavaScript引擎中的一个进程，它监控代码执行和管理作业队列。请记住，作为一个队列，作业执行运行的顺序是从队列的第一项作业到最后一项作业。


### 事件模型
当一个用户点击一个按钮或者按下键盘上的按钮，像`onclick`的事件就会触发。这个事件通过添加一项新的作业到作业队列尾部来响应交互。这就是JavaScript最基本的异步编程形式。直到事件触发，事件handler代码才会执行，而且当它执行时，它有适当的上下文。比如：


```js
let button = document.getElementById("my-btn");
button.onclick = function(event) {
    console.log("Clicked");
};
```

在这段代码中，在点击`button`之前， `console.log("Clicked")` 不会执行。点击`button`时，分配给`onclick`的函数添加到作业队列的尾部，而且当它之前的其他作业完成后，它将会执行。

事件适应于简单的交互，但是把多个隔离的异步调用链接一起更加复杂，因为你必须追踪每个事件的事件目标(event target)（之前例子中的`button`）。另外，你需要确保在第一次发生事件之前，添加了所有对应的事件handler。比如，如果在`onclick`赋值之前点击`button`，什么事情都不会发生。虽然事件响应适用于用户交互以及类似不常用的功能，但是它们对于更复杂的需求不是很灵活。

### 回调模式
当创建Node.js时，它通过普及编程的回调模式推广异步编程模型。回调模式和事件模型类似，因为异步代码直到之后的事件点才会执行。它们是有差异的，因为调用的函数是作为参数传递，比如这里展示的代码：

```js
readFile("example.txt", function(err, contents) {
    if (err) {
        throw err;
    }

    console.log(contents);
});
console.log("Hi!");
```

这个例子使用了传统的Node.js错误第一的回调方式。`readFile()`是为在磁盘中读取文件。这表示调用`readFile()`之后，`console.log(contents)` 打印任何内容之前， `console.log("Hi!")  ` 会立即输出。当`readFile()`完成时，它会带着回调函数及它参数添加一个新的作业到作业队列末尾。在这个作业之前的其他作业完成之后，接着它会执行。

回调模式比事件更加灵活，因为使用回调把多个调用链接在一起更容易。比如：

```js
readFile("example.txt", function(err, contents) {
    if (err) {
        throw err;
    }

    writeFile("example.txt", function(err) {
        if (err) {
            throw err;
        }

        console.log("File was written!");
    });
});
```

就像这个例子，嵌套多个方法调用创建的代码错综复杂，很难去理解和调式。当你想实现更复杂的功能时，回调也存在问题。如果你想两个异步操作并行运行，而且当它们都完成时通知你，该怎么办呢？如果你想同时开启两个异步操作，但是只取的第一个完成的结果，该怎么办？

在这些场景中，你需要追踪多个回调和清除操作，然而promises很大程度上改善了这种场景。

## Promise基础
promise是异步操作结果的占位符。不是订阅事件或者给函数传递回调，函数可以返回一个promise，像这样：

```js
// readFile promises 在未来的某个时间点完成
let promise = readFile("example.txt");
```

在这段代码中，`readFile()`实际上不会立即开始读取文件；它在之后发生。
相反，这个函数返回一个表示异步读取操作的promise对象，所以你可以在之后使用它。确切地说，当你能够使用这个结果完全取决于你promise的生命周期如何运行。

### Promise声明周期
每个promise都会经历一个短暂的生命周期，从*pending* 状态开始，这表示异步操作还没完成。pending状态的promise被视为未处理(unsettled)。只要`readFile()`函数返回它，最后一个例子中的promise就处于pending状态。一旦异步操作完成，promise就视为已处理(settled)，并进入以下两种可能的状态之一：

 1. Fulfilled: promise的异步操作已经成功完成。
 2. Rejected: 由于错误或者一些其他的原因，promise的异步操作没有成功地完成。

内置[[PromiseState]]属性设置为`"pending"`，"`fulfilled`"，或者"`rejected`"来反映promise的状态。这个属性没有在promise对象上暴露，所以你不能在程序中判断promise是在哪个状态。但是当promise使用`then()`方法改变状态时，你可以采取特殊的操作。

`then()`方法存在于所有的promise上，而且接收两个参数。第一个参数是promise处于fulfilled时调用的函数。与异步操作相关的任何其他数据将传递给这个fulfillment函数。第二个参数是promise为rejected时调用的函数。与filfillment函数相似，与rejection相关的任何其他数据将传递给rejection函数。

任何以这种方式实现`then()`方法称之为 *thenable* 。所有的promises都是
thenables，但是不是所有的thenables都是promises。

`then()`的两个参数都是可选的，所以你可以监听fulfillment和rejection的任意组合。比如，思考这组`then()`调用：

```js
let promise = readFile("example.txt");

promise.then(function(contents) {
    // fulfillment
    console.log(contents);
}, function(err) {
    // rejection
    console.error(err.message);
});

promise.then(function(contents) {
    // fulfillment
    console.log(contents);
});

promise.then(null, function(err) {
    // rejection
    console.error(err.message);
});
```

三个`then()`调用都是操作同一个promise。第一个调用监听了fulfillment和rejection。第二调用只监听了fulfillment；不会报告错误。第三个调用只监听rejection，并不报告错误。

Promise也有`catch()`方法，当`then()`只传递rejection handler时，它和`then()`的行为一致。比如，下面的`catch()`和`then()`调用功能上是等效的：

```js
promise.catch(function(err) {
    // rejection
    console.error(err.message);
});

// 等同于:

promise.then(null, function(err) {
    // rejection
    console.error(err.message);
});
```

`then()`和`then()`背后的意图是，让你组合使用它们去正确处理异步操作的结果。这个系统比事件和回调更好，因为它使操作成功或者失败完全明确。（当有错误时事件不会触发，在回调里，你必须始终记住检验错误参数。）只要知道，如果你不把rejection handler附加给promise，所有的错误将默默地发生。即使handler只是打印失败日志，通常也要传递一个rejection handler。
