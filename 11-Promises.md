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

#### 使用Promise.reject()
你也可以使用`Promise.resolve()`方法创建rejected promise。这和`Promise.resolve()`效果一样，除了创建的promise属于rejected状态，如下：

```js
let promise = Promise.reject(42);

promise.catch(function(value) {
    console.log(value);         // 42
});
```

添加到此promise的任何rejection handler都会被调用，但是fulfillment handlers不会。

如果你传递一个promise给`Promise.resolve()`或者`Promise.reject()`方法，该promise会毫无修改的返回。

#### 非Promise Thenables
`Promise.resolve()`和`Promise.reject()`也接受非promise thenables作为参数。当传递一个非promise thenable，这些方法创建一个新的promise，在`then()`函数之后调用。

当一个对象有`then()`方法时，一个非promise thenable就被创建了，`then()`方法接受`resolve`和`reject`参数，比如：

```js
let thenable = {
    then: function(resolve, reject) {
        resolve(42);
    }
};
```

这个示例中的`thenable`对象除了`then()`方法之外，没有与promise相关联的特性。你可以调用`Promise.resolve()`把`thenable`转换为一个fulfilled promise：

```js
let thenable = {
    then: function(resolve, reject) {
        resolve(42);
    }
};

let p1 = Promise.resolve(thenable);
p1.then(function(value) {
    console.log(value);     // 42
});
```

在这个实例中，`Promise.resolve()`调用`thenable.then()`，以便promise状态可以检测到。`thenable`的promise状态是fulfilled，因为在`then()`方法的内部调用了`resolve(42)`。在fulfilled状态下创建的新promise为p1，它的值来自`thenable`(它为42)，而且p1的fulfillment handler接受42作为参数值。

同样的流程可以用于`Promise.resolve()`从thenable中创建rejected promise。

```js
let thenable = {
    then: function(resolve, reject) {
        reject(42);
    }
};

let p1 = Promise.resolve(thenable);
p1.catch(function(value) {
    console.log(value);     // 42
});
```

这个示例与上一个示例类似，除了`thenable`是rejected。当执行`thenable.then()`，在rejected状态创建一个值为42的新promise。这个值接着会传递给`p1`的rejection handler。

`Promise.resolve()`和`Promise.reject()`这种机制，允许你轻松地结合非promise thenable使用。许多第三方库在ECMAScript 6引入promises之前就使用了thenables，所以将thenables转化为正式的promise的能力对于向后兼容之前存在的库很重要。当你不确定对象是否是promise时，把这个对象传递给`Promise.resolve()`或者`Promise.reject()`（取决于你的预期结果）是最好的判断方法，因为promise只会无变化的传递。

#### 执行器错误
如果在执行器中报错，然后promise的rejection handler就会被调用。比如：

```js
let promise = new Promise(function(resolve, reject) {
    throw new Error("Explosion!");
});

promise.catch(function(error) {
    console.log(error.message);     // "Explosion!"
});
```

在这段代码中，这个执行器试图报错。在每个执行器中，有一个隐性的`try-catch`，因此错误会被捕获，然后传给rejection handler。上一个示例等效于：

```js
let promise = new Promise(function(resolve, reject) {
    try {
        throw new Error("Explosion!");
    } catch (ex) {
        reject(ex);
    }
});

promise.catch(function(error) {
    console.log(error.message);     // "Explosion!"
});
```

这个执行器处理捕获任何抛出的异常，用来简化通用的用例，但是执行器里抛出的异常只有当rejection handler存在时才会报告。否则，错误会被抑制。
这在开发者早期使用promises就是一个问题，而且JavaScript环境通过提供捕获rejected promise的钩子来解决这个问题。

### 全局Promise Rejection处理

promises最有争议的一方面是，当promise在没有rejection handler时rejected之后，它只会隐式失败。一些人认为这是规范中最大的缺陷，因为它是JavaScript语言中唯一没让错误显式暴露的部分。由于promise的特性，确定promise rejection是否处理并不简单。比如，看这个示例：

```js
let rejected = Promise.reject(42);

// 此刻，rejected没有处理

// 一段时间之后...
rejected.catch(function(value) {
    // 现在rejected被处理了
    console.log(value);
});
```

你可以在任何时候调用`then()`或者`catch()`，无论这个promise是否处理，它们都能正常工作。在这个例子中，这个promise会立即rejcted，但是之后才会处理。

虽然ECMAScript的下个版本可能解决这个问题，当时浏览器和Node.js都已经实现这个变化去解决这个开发者痛点。它们都不是ECMAScript 6规范的一部分，但在使用promise时是很有用的工具。

#### Node.js Rejection处理
在Node.js，`process`对象上有两个事件关于promise rejection处理：

 - `unhandledRejection`：当promise被rejected，并且在事件循环的一回合内没有调用rejection handler时触发。
 - `rejectionHandled`：当promise被rejected，并且在事件循环的一回合之后调用rejection handler时触发。

这些事件目的是帮助识别rejected和未处理的promise。

`unhandledRejection`事件handler传递rejection原因（通常是一个错误对象）和rejected的promise作为参数。下面这段代码显示实际应用的`unhandledRejection`：

```js
let rejected;

process.on("unhandledRejection", function(reason, promise) {
    console.log(reason.message);            // "Explosion!"
    console.log(rejected === promise);      // true
});

rejected = Promise.reject(new Error("Explosion!"));
```

这个示例创建了一个带有error对象的rejected promise，并且监听`unhandledRejection`时间。这个事件handler接受error对象作为第一个参数，以及promise作为第二参数。

这个`rejectionHandled`事件handler只有一个参数，它是一个rejected的promise。例如：

```js
let rejected;

process.on("rejectionHandled", function(promise) {
    console.log(rejected === promise);              // true
});

rejected = Promise.reject(new Error("Explosion!"));

// 等待添加rejection handler
setTimeout(function() {
    rejected.catch(function(value) {
        console.log(value.message);     // "Explosion!"
    });
}, 1000);
```

这里，当最后调用rejection handler时，触发`rejectionHandled`事件。创建`rejected`之后，如果`rejected`调用rejection handler，那么这个事件不会触发。在创建`rejected`所在的事件循环中，调用rejection handler，这是无用的。

为了更好地追踪可能未处理的rejection，使用`rejectionHandled`和`unhandledRejection`事件去维持可能未处理的rejection列表。然后等待一段时间去检查这个列表。比如：

```js
let possiblyUnhandledRejections = new Map();

// 当rejection未处理，把它添加到map
process.on("unhandledRejection", function(reason, promise) {
    possiblyUnhandledRejections.set(promise, reason);
});

process.on("rejectionHandled", function(promise) {
    possiblyUnhandledRejections.delete(promise);
});

setInterval(function() {

    possiblyUnhandledRejections.forEach(function(reason, promise) {
        console.log(reason.message ? reason.message : reason);

        // 处理这些rejections
        handleRejection(promise, reason);
    });

    possiblyUnhandledRejections.clear();

}, 60000);
```

这是一个简单的未处理的rejection追踪器。它使用map去存储promise和它们rejection的原因。每个promise是一个key，promise的原因是关联值。每次触发`unhandledRejection`，就添加promise和rejection到map。每次触发`rejectionHandled`事件，就把处理的promise从map中移除。结果是，随着事件被调用，`possiblyUnhandledRejections` 增长和缩减。定期调用`setInterval()`检查可能未处理的rejection列表，然后想控制台输出信息（实际上，你可能想做一些其他的日志或者其他方式处理rejection）。
这个实例中使用了map，而不是weak map，因为你需要定期检查这个map，确认哪个promises存在，使用weak map是无法做到的。

虽然这个示例只适应于Node.js，但是浏览器也实现了类似的机制去告知开发者关于未处理的rejections。

#### 浏览器Rejection处理

浏览器也触发了两个事件帮助识别未处理的rejections。这些事件由`window`对象触发，而且和他们的Node.js等效。

- `unhandledrejection`：当promise被rejected，而且在一轮的事件循环内没有rejection handler时触发。
- `rejectionhandled`：当promise被rejected，而且在第一轮事件循环内调用了rejection handler。

Node.js实现了给事件handler传递单个参数，然而浏览器事件的事件handler只接收一个带有如下参数的事件对象：

- `type`：事件的名称（`"unhandledrejection"` 或者`"rejectionhandled"`)。
- `promise`：被rejected的promise对象。
- `reason`：来自promise的rejection值

在浏览器实现中的其他差异是，对于这两个事件都可以访问rejection value（`reason`）。比如：

```js
let rejected;

window.onunhandledrejection = function(event) {
    console.log(event.type);                    // "unhandledrejection"
    console.log(event.reason.message);          // "Explosion!"
    console.log(rejected === event.promise);    // true
});

window.onrejectionhandled = function(event) {
    console.log(event.type);                    // "rejectionhandled"
    console.log(event.reason.message);          // "Explosion!"
    console.log(rejected === event.promise);    // true
});

rejected = Promise.reject(new Error("Explosion!"));

```

这段代码使用`onunhandledrejection` 和`onrejectionhandled`DOM 0 级表示法赋值事件handlers。（如果你喜欢，你也可使用`addEventListener("unhandledrejection")` 和 `addEventListener("rejectionhandled")`）每个事件handler接收一个事件对象，它包含关于rejected promise的信息。`type`，`promise`和`reason`属性在事件handler都是可访问的。

在浏览器中的追踪未处理的rejections的代码和Node.js中的代码也十分相似：

```js
let possiblyUnhandledRejections = new Map();

// 当rejection未处理，把它添加到map
window.onunhandledrejection = function(event) {
    possiblyUnhandledRejections.set(event.promise, event.reason);
};

window.onrejectionhandled = function(event) {
    possiblyUnhandledRejections.delete(event.promise);
};

setInterval(function() {

    possiblyUnhandledRejections.forEach(function(reason, promise) {
        console.log(reason.message ? reason.message : reason);

        // 处理这些rejections
        handleRejection(promise, reason);
    });

    possiblyUnhandledRejections.clear();

}, 60000);
```

这个实现几乎和Node.js的实现完全一样。它使用相同的方法在maps存储promises和它们的rejection值，然后之后检查它们。唯一的区别在于事件handler中在哪提取信息。

虽然处理promise rejection很棘手，但是你才刚刚开始看到promises的功能有多强大。是时候采取下一步，把多个promises链接起来。

### 链接Promises

在这点上，promises可能只比使用回调和`setTimeout`函数多了些改进，但是还有更多，而不只是满足眼睛。进一步讲，有很多方法实现链式调用promise，去完成更多的异步行为。

`then()`或者`catch()`的每次调用实际上创建和返回另外一个promise。仅当第一个promise被fulfilled或rejected之后，第二个promise才会resolved。请看这个示例：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

p1.then(function(value) {
    console.log(value);
}).then(function() {
    console.log("Finished");
});
```

这段代码输出：

```js
42
Finished
```

`p1.then()`在调用`then()`上返回第二个promise。第二个`then()`的fulfilment handler在第一个promise已经resolved之后才会被调用，如果你不链式这个示例，它看起来如下：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = p1.then(function(value) {
    console.log(value);
})

p2.then(function() {
    console.log("Finished");
});
```

在这个非链式的版本中，`p1.then()`的结果存储在`p2`，然后调用`p2.then()`添加最后fulfillment handler。你可能已经猜到，`p2.then()`也返回了一个promise。这个示例只是没有使用那个promise。

#### 捕获错误

链式Promises允许你捕获异常，这个异常可能发生在前一个promise的fulfillment或者rejection handler中。比如：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

p1.then(function(value) {
    throw new Error("Boom!");
}).catch(function(error) {
    console.log(error.message);     // "Boom!"
});
```

在这段代码中，`p1`的fulfilment handler抛出一个错误。在第二promise上链式调用`catch`方法，能够通过它的rejection handler接收那个错误。如果rejection handler抛出错误也一样：

```js
let p1 = new Promise(function(resolve, reject) {
    throw new Error("Explosion!");
});

p1.catch(function(error) {
    console.log(error.message);     // "Explosion!"
    throw new Error("Boom!");
}).catch(function(error) {
    console.log(error.message);     // "Boom!"
});
```

这里，执行器抛出了一个错误，然后触发了`p1`promise的rejection handler。这个handler然后抛出了另外一个错误，它被第二个promise的rejection handler捕获。这链式的promise调用已经知道这条链中其他promise的错误。

通常在promise链的结尾有一个rejection handler ，确保你可以正确地处理可能发生的任何错误。

### 在promise链中返回值

promise链中另一个重要的方面是，能够把数据从一个promise传递到下一个promise。你已经看到在执行器内传递给`resolve()`的值被传递给该promise的fulfillment handler。你可以通过从fulfilment handler指定返回值，继续沿着promise链传递数据。例如：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

p1.then(function(value) {
    console.log(value);         // "42"
    return value + 1;
}).then(function(value) {
    console.log(value);         // "43"
});
```

在执行时，`p1`的fulfilment handler返回`value + 1`。因为`value`是42（来自这个执行器），这个fulfilment返回43。该值然后被传递给第二个promise的fulfilment handler，把它输出到控制台。

你可以使用rejection handler做同样的事。当调用rejection handler时，它可能返回一个值。如果返回一个值，这个值用于fulfill 链中下一个promise，比如：

```js
let p1 = new Promise(function(resolve, reject) {
    reject(42);
});

p1.catch(function(value) {
    // 第一个 fulfillment handler
    console.log(value);         // "42"
    return value + 1;
}).then(function(value) {
    // 第二个 fulfillment handler
    console.log(value);         // "43"
});
```

这里，执行器用42调用`reject()`。这个值传入promise的rejection handler，在这里返回`value + 1`。即使这个返回值来自rejection handler，它仍然在链中下一个promise的fulfilment handler中使用，如果需要，一个promise的失败可以允许整条链恢复。

#### 在Promise链中返回Promises

从fulfillment和rejection handler里返回基本类型值，这允许在promises之间传递数据，但如果你想返回一个对象该怎么办？如果这个对象是一个promise，然后需要采取额外的步骤判断如何处理。看如下示例：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = new Promise(function(resolve, reject) {
    resolve(43);
});

p1.then(function(value) {
    // 第一个 fulfillment handler
    console.log(value);     // 42
    return p2;
}).then(function(value) {
    // 第二个 fulfillment handler
    console.log(value);     // 43
});
```

在这段代码中，`p1`调度工作resolves为42。`p1`的fulfillment handler返回`p2`，一个已经处于resolved状态的promise。调用第二个fulfilment handler，因为`p2`已经被fulfilled。如果`p2`被rejected，将会调用rejection handler（如果存在），而不是第二个fulfilment handler。

重要的是认识这个模式，第二个fulfillment handler没有添加到`p2`，而是第三个promise。所以，第二个fulfilment handler附属于第三个promise，让前一个示例等效于这个：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = new Promise(function(resolve, reject) {
    resolve(43);
});

let p3 = p1.then(function(value) {
    // 第一个 fulfillment handler
    console.log(value);     // 42
    return p2;
});

p3.then(function(value) {
    // 第二个 fulfillment handler
    console.log(value);     // 43
});
```

这里，很明显第二个fulfilment handler附属于`p3`而不是`p2`。这是一个细节，但也是重要的区别。如果`p2`被rejected，将不会调用第二个fulfilment handler。例如：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = new Promise(function(resolve, reject) {
    reject(43);
});

p1.then(function(value) {
    // 第一个 fulfillment handler
    console.log(value);     // 42
    return p2;
}).then(function(value) {
    // 第二个 fulfillment handler
    console.log(value);     // never called
});
```

在这个示例中，第二个fulfillment handler永远不会被调用，因为`p2`被rejected。然后，你可以附加一个rejection handler：
 
```js
 let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = new Promise(function(resolve, reject) {
    reject(43);
});

p1.then(function(value) {
    // 第一个 fulfillment handler
    console.log(value);     // 42
    return p2;
}).catch(function(value) {
    // rejection handler
    console.log(value);     // 43
});
```

这里，rejection handler会被调用，作为`p2`被rejected的结果。来自`p2`的rejected值43传入该rejection handler。

当promise执行器执行时，filfillment或者rejection handlers返回的thenables不会改变。第一个定义的promise首先会运行它的执行器，然后将会运行第二个promise执行器等等。返回的thenables允许你定义这个promise结果的其他响应。你可以通过在fulfillment handler中创建新的promise，来延迟执行fulfillment handler。比如：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

p1.then(function(value) {
    console.log(value);     // 42

    // 创建新的promise
    let p2 = new Promise(function(resolve, reject) {
        resolve(43);
    });

    return p2
}).then(function(value) {
    console.log(value);     // 43
});
```
在这个示例中，`p1`的fulfillment handler中创建一个新的promise。这意味着，第二个fulfillment handler不会执行，直到`p2`fulfilled之后。在你想等前一个promise处理之后触发另外一个promise时，这种模式很有用。

## 响应多个Promises
到目前为止，本章的每个例子都是针对一个响应对应一个promise。然而，有时候你想监控多个promise的进程，以此确定下一个行为。ECMAScript 6提供了两个方法去监控多个promises: `Promise.all()`和`Promise.race()`。

### Promise.all()方法
这个`Promise.all()`方法接受一个参数，这是一个可以监控promises的迭代（例如数组），而且仅当迭代中每个promise resolved后，返回一个resolved promise。当迭代中的每个promise都是fulfilled，返回的promise是fulfilled，比如这个示例：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = new Promise(function(resolve, reject) {
    resolve(43);
});

let p3 = new Promise(function(resolve, reject) {
    resolve(44);
});

let p4 = Promise.all([p1, p2, p3]);

p4.then(function(value) {
    console.log(Array.isArray(value));  // true
    console.log(value[0]);              // 42
    console.log(value[1]);              // 43
    console.log(value[2]);              // 44
});
```

这里每个promise用数字resolves。调用`Promise.all()`创建promise`p4`，当promises `p1`，`p2`和`p3`都fulfilled之后，`p4`最后fulfilled。传入`p4` fulfillment handler的结果是一个包含每个resolved值的数组：42，43和44。这些值按传入`Promise.all`的promises顺序存储，所以你可以根据resolved它们的promises匹配promise的结果。

如果传入`Promise.all()`的任何一个promise被rejected，返回的promise会立即rejected，不会等待其他promises完成：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = new Promise(function(resolve, reject) {
    reject(43);
});

let p3 = new Promise(function(resolve, reject) {
    resolve(44);
});

let p4 = Promise.all([p1, p2, p3]);

p4.catch(function(value) {
    console.log(Array.isArray(value))   // false
    console.log(value);                 // 43
});
```

在这个例子中，`p2`的rejected值为43。`p4`的rejection handler会立即调用，不会等待`p1`和`p3`执行（它们仍然会执行；只是`p4`不会等待）。

rejection handler总是接受单个值而不是数组，而且这个值是来自rejected promise的rejection值。在这个例子中，rejection handler传入43对应`p2`的rejection。

### Promise.race()方法
`Promise.race()`方法提供一个稍微不同的方法监听多个promises。这个方法也接受promises的迭代去监听，并且返回一个promise，但是只要第一个promise被处理，返回的promise就会被处理。不像`Promise.all()`方法等所有promises都被fulfilled，只要数组中的任何一个promise被fulfilled，`Promise.race()`方法就会返回一个合适的promise。比如：

```js
let p1 = Promise.resolve(42);

let p2 = new Promise(function(resolve, reject) {
    resolve(43);
});

let p3 = new Promise(function(resolve, reject) {
    resolve(44);
});

let p4 = Promise.race([p1, p2, p3]);

p4.then(function(value) {
    console.log(value);     // 42
});
```

在这段代码中，创建`p1`作为fullfilled promise，然而其他promise调度工作。然后`p4`的fulfillment handler就会被调用，值为42，并且忽略其他promises。传入`Promise.race()`的promises真是一场比赛，看哪个promise首先处理。如果处理的第一个promise是fulfilled，那么返回的promise也是fulfilled；如果处理的第一个promise是rejected，那么返回的promise是rejected。这里有一个用rejection的例子：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = Promise.reject(43);

let p3 = new Promise(function(resolve, reject) {
    resolve(44);
});

let p4 = Promise.race([p1, p2, p3]);

p4.catch(function(value) {
    console.log(value);     // 43
});
```

这里，`p4`是rejected，因为在调用`Promise.race()`时，`p2`已经处于rejected状态。即使`p1`和`p3`是fulfilled，这些结果会被忽略，因为它们在`p2`被rejected之后发生。

## 继承Promises

就像其他的内置类型，你可以使用promise作为派生类的基础。这可以让你定义你的自己的promises种类去扩展内置promises的功能。比如，假设你想一个promise，除了通常的`then()`和`catch()`方法，你还可以使用命名为`success()`和`failure()`的方法。你可以按照一下方式创建该promise的类型：

```js
class MyPromise extends Promise {

    // 使用默认的constructor

    success(resolve, reject) {
        return this.then(resolve, reject);
    }

    failure(reject) {
        return this.catch(reject);
    }

}

let promise = new MyPromise(function(resolve, reject) {
    resolve(42);
});

promise.success(function(value) {
    console.log(value);             // 42
}).failure(function(value) {
    console.log(value);
});
```

在这例子中，`MyPromise`源自`Promise`，而且有两个额外的方法。`success()`方法模拟`resolve()`，`failure()`模拟`reject()`方法。

每个添加的方法使用`this`去调用它模拟的方法。衍生的promise和内置的promise功能相同，除了现在你可以调用`success()`和`failure()`，如果你想。

因为静态方法是继承的，`MyPromise.resolve()`方法，`MyPromise.reject()`，`MyPromise.race()`和`MyPromise.all()`在派生的promises上也是存在的。最后两个方法和内置的方法表现的一致，但是头两个方法稍微有点区别。

不管传入什么值，`MyPromise.resolve()`和`MyPromise.reject()`都会返回一个`MyPromise`的实例，因为这些方法使用`Symbol.species`属性（涵盖在第九章）去判断返回的promise类型。如果内置的promise出入其中一个方法，这个promise将会被resolved或者rejected，而且这个方法将返回一个新的`MyPromise`，所以你可以指定fulfillment和rejection handler。比如：

```js
let p1 = new Promise(function(resolve, reject) {
    resolve(42);
});

let p2 = MyPromise.resolve(p1);
p2.success(function(value) {
    console.log(value);         // 42
});

console.log(p2 instanceof MyPromise);   // true
```

这里，`p1`是一个内置的promise，将他传入`MyPromise.resolve()`方法。所得到的结果P2是`MyPromise`的实例，其中从`p1`中resolved的值传入fulfillment handler。

如果`MyPromise`的实例传入`MyPromise.resolve()`或者`MyPromise.reject()`方法，它将直接返回，不需要resolved。在所有其他方式，这两个方法的行为与`MyPromise.resolve()`和`MyPromise.reject()`一样。

## 异步任务运行

在第八章，我介绍了generators，而且向你们展示如何使用它们运行异步任务，比如：

```js
let fs = require("fs");

function run(taskDef) {

    // 创建迭代器, 在其他地方也可以访问
    let task = taskDef();

    // 启动任务
    let result = task.next();

    // 递归函数保持调用next()
    function step() {

        // 如果还有更多的事要做
        if (!result.done) {
            if (typeof result.value === "function") {
                result.value(function(err, data) {
                    if (err) {
                        result = task.throw(err);
                        return;
                    }

                    result = task.next(data);
                    step();
                });
            } else {
                result = task.next(result.value);
                step();
            }

        }
    }

    // 启动程序
    step();

}

// 定义和任务运行程序一起使用的函数

function readFile(filename) {
    return function(callback) {
        fs.readFile(filename, callback);
    };
}

// 运行任务

run(function*() {
    let contents = yield readFile("config.json");
    doSomethingWith(contents);
    console.log("Done");
});
```

这个实现有一些痛点。首先，再返回函数的函数包装每个函数有点混乱（甚至这个语句也是混乱的）。其次，没有办法去区分作为任务运行程序的回调函数的返回值和不是回调的返回值。

使用promise，你可以通过确保每个异步操作返回promise，来大大简化和概括这个流程。通用接口意味着可以大量简化一步代码。这里有一种方式，你可以简化任务运行程序：

```js
let fs = require("fs");

function run(taskDef) {

    // 创建迭代器
    let task = taskDef();

    // 启动任务
    let result = task.next();

    // 递归函数进行遍历
    (function step() {

        // 如果还有更多的事情要做
        if (!result.done) {

            // resolve一个promise，�使其更容易
            let promise = Promise.resolve(result.value);
            promise.then(function(value) {
                result = task.next(value);
                step();
            }).catch(function(error) {
                result = task.throw(error);
                step();
            });
        }
    }());
}

// 定义与任务运行程序一起使用的函数

function readFile(filename) {
    return new Promise(function(resolve, reject) {
        fs.readFile(filename, function(err, contents) {
            if (err) {
                reject(err);
            } else {
                resolve(contents);
            }
        });
    });
}

// 运行任务

run(function*() {
    let contents = yield readFile("config.json");
    doSomethingWith(contents);
    console.log("Done");
});
```

这个版本的代码，通用的`run()`函数执行生成器（generator）创建迭代器。它调用`task.next()`去启动任务，并且递归调用`step()`直到迭代器完成。

在`step()`函数内，如果有更多的事情要做，那么`result.done`是`false`。在这点上，`result.value`应该是一个promise，但是调用`Promise.resolve()`，以防万一有问题的函数没有返回promise。（记住，`Promise.resolve()`通过任何传入的promise，并且把任何非promise包裹在promise里。）然后，添加一个fulfillment handler，它提取promise的值，并且把值传回迭代器。之后，在`step()`调用自身之前，`result`分配给为下一个yield结果。

rejection handler把任何rejection结果存储在一个错误对象中。`task.throw()`方法把错误对象传回迭代器，如果在任务中存在错误，则将`result`分配给下一个yield结果。最后，在`catch()`中调用`step()`继续执行。

这个`run()`函数可以运行任何生成器，它使用`yield`去异步代码，不需要暴露promises（或者回调函数）给开发者。事实上，因为函数调用的返回值总是转换为promise，所以这个函数甚至可以返回除了promise之外的内容。这表示，当使用`yield`调用时，同步和异步方法都可以正常地工作，而且你不需要判断返回的值是否是一个promise。

唯一的担心是确保异步函数比如`readFile()`返回能够一个正确标识其状态的promise。对于Node.js内置方法，意味着你必须把这些方法转换为返回promises而不是使用回调函数。

### 未来的异步任务运行
在我写作的时候，有一个即将到来的解决方案给JavaScript中异步任务运行带来更简单的语法。`await`语法的工作正在进行，它将进一步反映先前部分基于promise的例子。基本思想是使用标记为`async`而不是generator的函数，在调用函数时，使用`await`而不是`yield`，比如：

```js
(async function() { 
    let contents = await readFile("config.json");  
    doSomethingWith(contents);
    console.log("Done");
});
```

`function`之前的`async`关键字指示，这个函数以异步的方式运行。`await`关键字表示`readFile("config.json")`函数调用应该返回promise。响应应该包裹在promise中。就像之前部分的`run()`的实现，如果promise是rejected `await` 将会抛出异常，否则从promise中返回值。最终结果是你可以编写异步代码，就像它是同步的，而不需要管理基于迭代器的状态机的开销。`await`语法预计将在ECMAScript 2017 （ECMAScript 8）实现。

## 概括

Promises旨在异步操作给予你比时间和回调更多的控制和组合，来改善JavaScript中的异步编程。Promises将作业添加到JavaScript引擎的作业队列中，以便稍后执行，而第二个作业队列追踪promise fulfillment 和rejection handler，以确保正确执行。

Promises有三种状态：pending，fulfilled和rejected。promise开始为pending状态，成功执行后为变为fulfilled状态，或者失败后为rejected状态。在任何一种情况下，当promise处理时，可以添加handler来执行。`then`方法允许你分配一个fulfillment和rejection handler，以及`catch()`方法只允许你分配一个rejection handler。

你可以用各种方式链式调用promises，并在它们之间传递信息。当之前的promise被resolved，每次调用`then()`会创建和返回一个新的resolved promise。这样可以使用promise链触发一系列异步事件的响应。你也可以使用`Promise.race()`和`Promise.all()`来监控多个promises的进展，并做出相应的回应。

当你组合使用generator和promises时，异步任务运行更简单，因为promises提供了异步操作符可以返回的通用接口。然后你可以使用generator和`yield`操作符去等待异步响应并进行适当响应。

大部分新的web APIs正在建立在promises之上，你可以期待更多的后续。