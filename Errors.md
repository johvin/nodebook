# Errors（错误）

> Errors 是 Node.js 中基础且重要的组成部分，本文是基于 Node.js v6.9.4 总结而来的。

Node.js 应用中存在四种错误类型：

- 标准 JavaScript 错误：
    + EvalError
    + SyntaxError
    + RangeError
    + ReferenceError
    + TypeError
    + URIError
- 系统错误
- 用户指定的错误
- Assertion 错误（assert 生成）

## 错误冒泡和拦截

Node.js 有多种错误冒泡和处理的机制，所有的错误被作为异常来处理。错误机制根据 API 可以分为两类：同步 APIs 错误机制和异步 APIs 错误机制。

### 同步 APIs 错误机制

同步 APIs 的错误机制比较简单，API 内会使用 `throw` 抛出错误，然后使用 `try/catch` 来捕获处理，如果不捕获，那么 Node.js 的进程会触发 `error` 事件并会立即退出。

### 异步 APIs 错误机制

异步 APIs 的错误机制一共有三种：

- 异步方法接收一个 `callback` 回调函数，该函数接收一个 `Error` 对象作为第一个参数，错误处理逻辑应该在 `callback` 回调中处理。
- 异步方法被 `EventEmitter`(后文简称 `EE`) 对象调用，此时 `EE` 会触发 `error` 事件。如果不存在 `error` 事件处理函数，则这个错误会被 `throw`。最终的结果跟同步的情况一致。
- Node.js API 中的一小部分典型的异步方法仍会使用 `throw` 机制来抛出异常，这类异常必须使用 `try/catch` 来处理。暂时没有这类方法的列表，需要查看具体文档来找到他们并使用适当的错误处理机制。

对于 `EE` 对象或继承自 `EE` 的对象，如果不存在 `error` 事件处理函数，那么错误就会被抛出，从而导致 Node.js 进程退出。解决方案是使用 `domain` 模块注册错误处理函数来捕获错误或者使用 `process.on('uncaughtException')` 进行错误拦截。具体逻辑请查看相应章节。

## Error 类

Node.js 产生的错误对象都是 `Error` 的实例或者继承自 `Error`。`Error` 对象会捕获错误生成时刻的代码“堆栈跟踪”的详细信息。

### new Error(message)

该方法生成一个新的 `Error` 对象，如果 `message` 的一个对象，则会使用 `message.toString()` 的结果作为 `error.message`。“堆栈跟踪”是基于 `V8` 的堆栈跟踪 API 的。堆栈信息会添加到 `.stack` 属性上，其值第一行格式为 `ErrorType: message`，之后跟着函数调用的堆栈信息。

### Error.captureStackTrace(targetObject[, constructorOpt])

该方法在 `targetObject` 上添加 `.stack` 属性，其值为代码中调用 `Error.captureStackTrace` 时的堆栈跟踪信息。如果 `targetObject` 中存在 `.message`，那么 `.stack` 值的第一行中的 `message` 也会确定。可选参数 `constructorOpt` 的类型是函数。如果指定了这个参数，那么堆栈跟踪中这个函数之后的函数调用信息会被忽略（即 `.stack` 中不包括这部分信息）。这一功能可用于隐藏错误的产生细节。

*注意：`targetObject` 上生成 `stack` 信息后 delete，然后改变 `message` 属性和调用关系，再次在 `targetObject` 上生成堆栈跟踪信息，则该信息不会改变，还是第一次生成的信息。*

### Error.stackTraceLimit

该属性指定了最大堆栈跟踪层数，即调用堆栈层数超过该值时，`.stack` 中的堆栈层数就是这个值。默认值为 10，可以更改为任意数值。更改后生效。

*如果该值设置为非数字或负数，则堆栈跟踪不会捕获任何调用信息*

### error.message

使用 `new Error(message)` 创建的 `Error` 新对象的 `message` 属性可以更改，但是 `stack` 属性第一行中的 `message` 值是不会更改的。

### error.stack

第一行格式为 `<error 类名>: <error message>`，第二行起是一系列堆栈帧。每一站的格式为 `at <名字> (<对应的调用位置信息>)`，其中名字是变量名、函数名或者对象方法名。如果找不到合适的名字则省略。

*另外需要注意的是，堆栈帧只包括 JavaScript 的函数，并不包括 C++ 函数。*

`error.stack` 的值不是一开始就生成的，而是在访问的时候才生成的。

## RangeError 类

Node.js 会立即抛出 `RangeError` 实例作为参数验证的一种形式。

## ReferenceError

这类错误通常表示代码中存在打字错误，或者其他巨大错误。如果这种错误不是动态生成的，那么就认为这是代码或者依赖包中的一个 bug。

`error.arguments` 属性是只包含一个字符串的数组，该字符串即未定义变量的名字。

## SyntaxError

这类错误出现在代码评估（code evaluation）中，代码评估包括 `eval`、 `Function`、`require` 或者 `vm`。语法错误说明代码存在巨大问题，且这类错误只能在其他的上下文捕获，生成错误的上下文中是不可恢复的。一个正确的例子：

```js
try {
  require('vm').runInThisContext('binary ! isNotOk');
} catch(err) {
  // err will be a SyntaxError
}
```

## TypeError

类型错误和 `RangeError` 类似，也是作为参数验证的一种形式。

## 异常 vs 错误

虽然没有要求错误是 `Error` 或者继承自 `Error` 的类的实例，但所有 Node.js 或者 JavaScript 运行时抛出的异常都是 `Error` 的实例。

某些异常在 JavaScript 层是不可恢复的。这类异常总是会导致 Node.js 进行崩溃。例如 C++ 层的 `assert()` 检查或者 `abort()` 调用。

## 系统错误

系统错误是程序在运行环境中发生异常时产生的。Node.js 中，系统错误是增强型的 `Error` 对象，这些对象具有额外添加的一些属性。错误代码及意义可以运行 `man 2 intro` 或者 `man 3 errno` 或者[点这里](http://man7.org/linux/man-pages/man3/errno.3.html)。

### System Error 类

`System Error` 添加的属性有三个：

- `error.code`: 例如 `ENOENT`、`EPIPE`
- `error.errno`: 返回 error code 的负值，例如 `ENOENT` 错误的 `code` 是 2，所以 `errno` 为 -2
- `error.syscall`: 返回一个描述系统调用失败的字符串
