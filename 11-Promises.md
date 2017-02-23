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

即使在promise已经处理之后，如果fullfillment或者rejection添加到作业队列，它仍然会被执行。这允许你在任意时候添加新的fullfillment和rejection，保证它们将被调用。比如：

```js
let promise = readFile("example.txt");

// 原始的fulfillment handler
promise.then(function(contents) {
    console.log(contents);

    // 另外一个
    promise.then(function(contents) {
        console.log(contents);
    });
});
```

在这段代码中，fulfillment handler给同一个promise添加了另外一个fulfillment。这个promise在这个时候已经是fulfilled，所以新的fulfillment handler被添加到作业队列，并且条件满足就会调用。Rejection handlers也是同样的原理。

当promise resolved之后，每个`then()`或者`catch()`调用将创建以各新的作业队列去执行。但是这些作业最终在一个单独的作业队列中，这作业队列只保存promises。第二个作业队列的精确的细节对于理解如何使用promise不是很重要，只要你理解一般情况下作业队列如何工作。

### 创建未处理的(unsettled) Promises
使用`Promise`构造函数创建新的promise。这个构造函数接受一个参数：一个称为 *executor* 的函数，它包含初始化promise的代码。executor传递两个命名为`resolve()`和`reject()`的函数作为参数。当executor已经成功完成示意promise准备resolved时，调用`resolve()`函数，`reject()`函数表示executor已经失败。

```js
// Node.js 示例

let fs = require("fs");

function readFile(filename) {
    return new Promise(function(resolve, reject) {

        // 触发异步操作
        fs.readFile(filename, { encoding: "utf8" }, function(err, contents) {

            // 检查错误
            if (err) {
                reject(err);
                return;
            }

            // 读取成功
            resolve(contents);

        });
    });
}

let promise = readFile("example.txt");

// 监听 fulfillment 和 rejection
promise.then(function(contents) {
    // fulfillment
    console.log(contents);
}, function(err) {
    // rejection
    console.error(err.message);
});
```

在这个示例中，原生的Node.js `fs.readFile()` 异步调用包裹在promise中。excutor要么传递错误对象到`reject()`函数，要么传递文件内容到`resolve()`函数。

请记住，当`readFile()`调用后，executor将立即执行。在executor内，调用`resolve()`或者`reject()`时，添加一个作业到作业队列来resolve promise。这称之为 *作业调度* (job scheduling)，而且如果你曾经使用过`setTimeout()`或者
`setInterval()`，你应该已经很熟悉它。在作业调度中，你给作业队列添加一个新的作业，然后说“现在不要执行它，但是之后要执行它。”比如，`setTimeout()`函数让你在作业添加到队列之前指定延时时间：

```js
// 500毫秒之后添加此函数到作业队列中。
setTimeout(function() {
    console.log("Timeout");
}, 500);

console.log("Hi!");
```

这段代码调度作业在500毫秒之后添加到作业队列。这两个`console.log()`调用产生如下输出：

```
Hi!
Timeout
```

由于500毫秒延时，传入`setTimeout()`的函数的输出显示在`console.log("Hi!")`调用的输出之后。

Promises的工作原理相似。promise的executor会立即执行，先于在这段源码之后的任何内容。比如：

```js
let promise = new Promise(function(resolve, reject) {
    console.log("Promise");
    resolve();
});

console.log("Hi!");
```

这段代码的输出是：

```
Promise
Hi!
```

调用`resolve()`将触发异步操作。传入`then()`和`catch()`的函数将被异步执行，这些函数也将添加到作业队列。这有一个示例：

```js
let promise = new Promise(function(resolve, reject) {
    console.log("Promise");
    resolve();
});

promise.then(function() {
    console.log("Resolved.");
});

console.log("Hi!");
```

这个示例的输出是：

```
Promise
Hi!
Resolved
```

注意到计时`then()`调用出现在`console.log("Hi!")`之前，它实际上之后才执行（不像executor）。这是因为fulfillment和rejection的handlers通常是在executor完成之后，添加到工作队列的末尾。

### 创建处理的(settled)Promises
`Promise`的构造函数是创建未处理的promise的最好方式，因为promise executor执行的动态性质。但是如果你想promise只是表示一个简单值，这样调度作业没有意义，它只是简单地传递值给`resolve()`函数。取而代之，有两种方法可以创建处理的promises并给予指定的值。

#### 使用Promise.resolve()
`Promise.resolve()`方法接受一个参数并且返回fulfilled状态的promise。这表示没有作业调度发生，而且你需要添加一个或者多个fulfillment handler到promise去提取值。比如：

```js
let promise = Promise.resolve(42);

promise.then(function(value) {
    console.log(value);         // 42
});
```

这段代码创建一个fulfilled状态的promise，所以fulfillment handler接受的value为42。如果rejection handler也添加到这个promise，这个rejection handler将不会被调用，因为这promise绝不会处于rejected状态。
