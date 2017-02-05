# Events（事件）

> Events 是 Node.js 的基础且重要的组成部分，本文是基于 Node.js v6.9.4 总结而来的。

很多 Node.js 核心 API 是基于一种惯用的异步事件驱动架构构建的，该架构被成为“发射器”，会定期的发射命名事件，这些事件会触发函数对象（监听者）的调用。

例如，`net.Server` 在每次有连接被创建时会发射事件；`fs.ReadStream` 在文件被打开时会发射事件；`stream` 流每当有数据可以被读取的时候也会发射事件。

所有能够发射事件的对象都是 `EventEmitter`（后文简称 `EE`）类的实例。典型地，事件名采用驼峰命名规则，但是任何有效的 JavaScript 属性键值也可以使用。

当 `EE` 发射一个事件，所有关联到该事件的函数都会同步被调用。这些函数的任何返回值都会被忽略和舍弃。

下面的例子展示了一个简单的 `EE` 的使用方法。

```js
const EE = require('events');

class MyEmitter extends EE {}

const myE = new MyEmitter();
myE.on('event', () => {
  console.log('an event occurred!');
});

myE.emit('event');
```

## 给监听者传递参数和 this

`emit` 方法允许传递一组任意参数集给监听函数。当一个普通的监听函数被 `EE` 调用的时候，其内的 `this` 关键字指向调用这个函数的 `EE`。

```js
const myE = new MyEmitter();
myE.on('event', (a, b) => {
  console.log(a, b, this);
  // Prints:
  // a b MyEmitter {
  //   domain: null,
  //   _events: { event: [Function] },
  //   _eventsCount: 1,
  //   _maxListeners: undefined }
});

myE.emit('event', 'a', 'b');
```

使用 ES6 的箭头函数作为监听函数也可以，但是此时的 `this` 关键字就不在指向 `EE` 实例了。

## 异步 vs 同步

事件监听函数会根据注册的顺序被同步调用。保证事件的正确顺序以及避免竞争条件或者逻辑错误是真重要的。如果需要异步操作，可以使用 `setImmediate()` 或者 `process.nextTick()` 方法。

## 只处理事件一次

`on()` 方法注册的监听函数在每次命名事件被发射的时候都会调用。如果只想监听函数最多被调用一次，可以使用 `once()` 方法。

## 错误事件

当一个错误在 `EE` 实例内部发生的时候，一个典型的行为是发射一个 `error` 事件。这在 Node.js 中被视为特殊情况。

如果没有监听函数向 `EE` 监听错误，那么当错误事件发射时，这个错误会被抛出，堆栈跟踪也会被打印，并且 Node.js 进行退出。

为了防止对 Node.js 进程造成奔溃，可以在 process 对象上注册 `uncaughtException` 事件或者使用 domain 模块（具体查看 domain 章节）。

```js
const myE = new MyEmitter();

process.on('uncaughtException', (err) => {
  console.log('whoops! there was an error');
});

myE.emit('error', new Error('whoops!'));
```

*作为最佳事件，错误事件总是应该添加监听函数。*

## EventEmitter 类

`EE` 类通过 `events` 模块被定义和暴露接口。

```js
const EE = require('events');
```

所有 `EE` 都会在新监听函数被添加时发射 `newListener` 事件，在已注册监听函数被移除时发射 `removeListener` 事件。

### newListener 事件

- `eventName` &lt;String&gt; | &lt;Symbol&gt; 即将被监听的事件名
- `listener` &lt;Function&gt; 事件处理函数

`EE` 实例会在一个监听函数被添加到监听器函数内部队列之前发射 `newListener` 事件。被监听的事件名和监听器函数的引用作为参数传递给 `newListener` 事件的处理函数。

`newListener` 事件在监听函数被添加到队列之前触发有一个微妙但很重要的副作用：任何在 `newListener` 事件处理函数中被注册到同一个事件中的其他监听函数都会先于当前要被添加到队列中的监听函数被添加进去。下面给出一个例子：

```js
const myE = new MyEmitter();
// Only do this once so we don't loop forever
myE.once('newListener', (event, listener) => {
  if (event === 'event') {
    // Insert a new listener in front
    myE.on('event', () => {
      console.log('B');
    });
  }
});
myE.on('event', () => {
  console.log('A');
});
myE.emit('event');
// Prints:
// B
// A
```

### removeListener 事件

`removeListener` 事件的回调函数参数与 `newListener` 一样。该事件在监听函数被移除后触发。

### EventEmitter.listenerCount(emitter, eventName)（已弃用）

该函数已弃用，请使用 `emitter.listenerCount(eventName)`。

### EventEmitter.defaultMaxListeners

默认单个事件的最大监听函数是 `10`。这个阈值可以通过实例方法  `emitter.setMaxListeners(n)` 来修改。如果要修改所有实例的默认阈值，可以直接修改类属性  `EventEmitter.defaultMaxListeners`。

*使用这个方式修改需要注意一点：这种修改不仅对之后创建的 `EE` 实例有效，对修改前创建的实例也有效果。不过即便这样，使用 `emitter.setMaxListeners(n)` 方法会有更高的优先级。*

注意：这个并不是硬约束。`EE` 实例允许添加超过限制的监听函数，但同时也会向 `stderr` 输出一个“EventEmitter 可能会内存泄漏”的跟踪警告。对于单个 `EE`，可以使用 `emitter.getMaxListeners()` 和 `emitter.setMaxListeners()` 方法临时避免这个警告：

```js
emitter.setMaxListeners(emitter.getMaxListeners() + 1);
emitter.once('event', () => {
  // do stuff
  emitter.setMaxListeners(Max.max(emitter.getMaxListeners() - 1, 0));
});
```

命令行标志 `--trace-warnings` 可以用来显示类似警告的堆栈跟踪信息。警告信息可以使用 `process.on('warning')` 检测到，该信息还会包含额外的 `emitter`，`type` 和 `count` 属性，分别是指向 `EE` 的实例引用，事件名称以及相关监听函数的数量。

### emitter.addListener(eventName, listener)

`emitter.on(eventName, listener)` 方法的别称。

### emitter.emit(eventName[, ...args])

发射指定事件名的事件，并按照监听函数注册的顺序顺序调用它们，并将提供的参数传递给它们。

如果事件存在监听函数则返回 true，否则返回 false。

### emitter.eventNames()

返回一个数组，里面包含当前发射器中注册的监听事件列表。这些值的类型包括字符串和 Symbol。

```js
const EE = require('events');
const myE = new EE();

myE.on('foo', () => {});
myE.on('bar', () => {});

const sym = Symbol('symbol');
myE.on(sym, () => {});

console.log(myE.eventNames());
// Prints: [ 'foo', 'bar', Symbol(symbol) ]
```

### emitter.getMaxListeners()

返回 `emitter` 当前的最大监听函数的个数。这个数值是 `emitter.setMaxListeners(n)` 或者等于 `EventEmitter.defaultMaxListeners`。

### emitter.listenerCount(eventName)

返回事件名是 eventName 的监听函数的个数。eventName 的类型是字符串或者 Symbol。

### emitter.listeners(eventName)

返回 eventName 事件的监听函数的拷贝数组。

```js
server.on('connection', (stream) => {
  console.log('someone connected!')；
});
console.log(util.inspect(server.listeners('connection')));
// Prints: [ [Function] ]
```
