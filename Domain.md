# Domain（域）（即将弃用）

> 这个模块即将被弃用。一旦替代的 API 确定下来，这个模块就会被弃用。本文是基于 Node.js v6.9.4 总结而来，目前最新 Node.js 的版本是 7.4.0，该版本中 Domain
依然是即将弃用的状态，所以最快也是在 8.x 的版本中才会弃用。

这个模块是 Node.js 中重要的组成部分，它能作为单个组处理多种不同的 I/O 操作。任何注册到 domain 的 `EventEimitter`（后面简称为 `EE`） 或者回调函数发射一个错误事件或者抛出一个错误，这个 domain 对象都能够被通知，而不是像 `process.on('uncaughtException')` 处理函数那样丢失错误的上下文，或者导致程序立即退出。

## 警告：不要忽略错误

Domain 错误处理不能代替在错误发生时关闭进程这件事（也就是说，系统发生位置错误的时候还是要关闭进程的）。

就 JavaScript 中 `throw` 的工作机制的本质来说，几乎不存在任何一种在不带来引用泄漏的情况下安全的从错误产生的地方继续运行或者创建一些未定义的脆弱状态的方法 ( `uncertain` )。

最安全的解决抛出错误的方法就是关闭进程。当然，通常在 web server 中会有很多打开状态的连接，突然关闭这些连接是不合理的，因为错误是其他操作触发的。

更好的处理方式是给触发错误的请求发送一个错误的响应，让其他已经存在的请求正常的结束，并且在那个工作线程中停止监听新的请求。

这样，`domain` 的使用和 `cluster` 模块紧密联系在一起，因为工作线程在遇到错误时，master 可以创建新的工作线程。对于部署到多机的 Node.js 程序，终端代理或服务注册者能够知道错误的产生并做出适当的反应。下面给出的例子能很好的实现这种处理方式。

```js
const cluster = require('cluster');
const cpus = require('os').cpus().length;
const PORT = +process.env.PORT || 1337;

if (cluster.isMaster) {
  // You can also of course get a bit fancier about logging,
  // and implement whatever custom logic you need to prevent
  // DoS attacks and other bad behaviour.
  //
  // The important thing is that the master does very little,
  // increasing our resilience to unexpected errors.
  for(let i = 0; i < cpus; i++) {
    cluster.fork();
    cluster.on('disconnect', (worker) => {
      console.error('disconnect');
      cluster.fork();
    });
  }
} else {
  const domain = require('domain');

  const server = require('http').createServer((req, res) => {
    const d = domain.create();
    d.on('error', (err) => {
      console.error('error', err.stack);

      try {
        // make sure we close down within 30 seconds
        // and don't keep the process open just for that!
        setTimeout(() => {
          process.exit(1);
        }, 30000).unref();

        // stop taking new requests.
        server.close();

        // let the master know we're dead.
        // This will trigger a 'disconnect'
        // in the cluster master, and then
        // it will fork a new worker.
        cluster.worker.disconnect();

        // try to send an error message to
        // the request that triggered the problem.
        res.status(500).send('Oops, there was a problem!\n');
      } catch (err2) {
        // oh well, not much we can do at this point.
        console.error('Error sending 500', err2.stack);
      }
    });

    // Because req and res were created before this domain existed,
    // we need to explicitly add them.
    // See the explanation of implicit vs explicit binding below.
    d.add(req);
    d.add(res);

    d.run(() => {
      handleRequest(req, res);
    });
  });

  server.listen(PORT);
}

// This part isn't important. Just an example routing thing.
// You'd put your fancy application logic here.
function handleRequest(req, res) {
  switch (req.url) {
    case '/error':
      setTimeout(() => {
        flerk.bark();
      });
    default:
      res.end('ok');
  }
}
```

## 像 Error 对象添加额外信息

一个错误对象路由到 domain 时，一些额外的字段会被添加到其上。

- `error.domain`: 第一个处理 error 的域
- `error.domainEimitter`: 发射这个 error 对象的 `EE`
- `error.domainBound`: 绑定到这个域的回调函数，其接收一个 error 作为第一个参数
- `error.domainThrown`: 布尔值，表示这个错误是否被抛出、发射或传递到绑定的回调函数中。

## 隐式绑定

如果域先创建并使用，那么其下的所有新创建的 `EE` 对象都会在被创建的时候隐式的绑定到当前的活动域上。

另外，传递给低级事件循环请求（例如 `fs.open` 或其他接收 callback 作为参数的方法 `uncertain`）的回调函数会自动绑定到当前活动域。如果这些方法抛出异常，活动域会 catch 到这个错误。

为了防止过多的使用内存，domain 对象不会隐式的添加为活动域的孩子。一旦隐式添加，则会组织 request 和 response 对象的垃圾回收。如果想将 domain 对象添加为另一个 domain 的孩子，需要显示添加。

隐式绑定会将抛出的错误和错误事件路由到 domain 的 `error` 事件，但不会在该 domain 上注册 `EE`，所以 `domain.dispose()` 不会关闭这个 `EE`（这里的意思应该是不会阻止事件继续冒泡 `uncertain`）。隐式绑定只关注抛出的错误和 `error` 事件。

## 显式绑定

需要使用显示绑定的情况有三种：

- 正在使用的 domain 不是应该被绑定到指定 `EE` 上的那个。
- `EE` 在一个 domain 上下文内部创建，但应该被绑定到其他 domain 上。
- `EE` 不在活动域上下文被创建，需要显示绑定到某个 domain（译者添加）

下面的例子正是第二种情况的场景。

```js
const domain = require('domain');
const http = require('http');
const serverDomain = domain.create();

serverDomain.run(() => {
  http.createServer((req, res) => {
    var reqd = domain.create();
    reqd.add(req);
    reqd.add(res);
    reqd.on('error', (err) => {
      console.error('Error', err, req.url);
      try {
        res.writeHead(500);
        res.end('Error occured, sorry.');
      } catch (error) {
        console.error('Error sending 500', error, req.url);
      }
    });
  }).listen(1337);
});
```

## domain.create()

创建一个新的域对象

## Domain 类

Domain 类封装了路由错误和未捕获异常到当前活动域对象的功能。是 `EE` 的子类。要处理其捕获的错误，需要监听 `error` 事件。

### domain.run(fn[, ...args])

在 domain 的上下文中运行指定的函数，并隐式的绑定在其内创建的所有的 `EE`、`timers` 以及低等级的请求。可以选择给运行的函数传递参数。这是 domain 的基本用法。

一个例子：

```js
const domain = require('domain');
const fs = require('fs');
const d = domain.create();

d.on('error', (err) => {
  console.error('Caught error!', err);
});

d.run(() => {
  process.nextTick(() => {
    setTimeout(() => { // simulating some various async stuff
      fs.open('non-existent file', 'r', (err, fd) => {
        if (err) throw err;
        // proceed ...
      });
    }, 100);
  });
});
```

这个例子中，`d.on('error')` 处理函数会被触发，而不是导致程序崩溃。

### domain.members

返回一个数组，包括被显示添加到 domain 的定时器和 `EE`。

### domain.add(<EventEimitter> | <Timer>)

显式添加 emitter （包括 `EE` 和 `Timer`）到 domain。如果 emitter 的任何处理函数抛出错误，或者该 emitter 发射一个错误事件，这个错误都会被路由到 domain 的错误事件，这一行为与隐式绑定相同。这一行为对于定时器也有相同的表现。

如果 emitter 已经绑定到某个 domain，那么调用此接口，该 emitter 会从前一个 domain 中移除，然后绑定到调用的 domain 上。

### domain.remove(<EventEimitter> | <Timer>)

从 domain 中移除指定 emitter，功能与 `domain.add(emitter)` 相对。

### domain.bind(callback)

返回一个绑定后的函数，该函数对 `callback` 进行了包裹。返回的函数被调用时，其中如果抛出任何错误都会被路由到 domain 的 `error` 事件。

一个例子：

```js
const d = domain.create();

function readSomeFile(filename, cb) {
   fs.readFile(filename, 'utf8', d.bind((err, data) => {
     if (err) throw err; // this will be passed to the domain

     // if error occurs in JSON.parse， it will also be passed to the domain
     return cb(data ? JSON.parse(data) : null);
   }));
}

d.on('error', (err) => {
  // proceed ...
});
```

### domain.intercept(callback)

该方法与 `domain.bind(callback)` 基本一致。然而此方法还能够捕获抛出的错误。它会拦截作为 `callback` 第一个实参的 `Error` 对象。通过这个功能，较常见的 `if (err) return callback(err);` 模式可以用一个单一的错误处理函数集中到一个地方进行错误处理。

一个例子：

```js
const d = domain.crate();

function readSomeFile(filename, cb) {
  fs.readFile(filename, 'utf8', d.intercept((data) => {
    return cb(data ? JSON.parse(data) : null);
  }));
}

d.on('error', (err) => {
  // proceed ...
});
```

### domain.enter()

该方法被 `run`、`bind` 和 `intercept` 根据情况调用用来设置活动域。该方法会为该 domain 设置 `domain.active` 和 `process.domain`，并隐式将该 domain 压入 domain 栈，该栈由 domain 模块管理（详见 `domain.exit()` 一节查看 domain 栈的详情）。对 `enter` 的调用限定了绑定到该 domain 的异步调用和 I/O 操作链的开始。

调用 `enter` 只改变当前的活动域，不改变域本身。`enter` 和 `exit` 可以在一个 domain 中成对调用任意次。

如果调用 `enter` 的 domain 已经被释放了，那么 `enter` 会立即结束而不做任何事。

### domain.exit()

该方法会退出当前 domain，并将该 domain 从 domain 栈中弹出。任何时候如果要切换一个不同的异步调用链的上下文，需要确保当前 domain 是已退出状态。调用 `exit` 限定了绑定到该 domain 的异步调用和 I/O 操作链的结束或者中断。

如果当前执行上下文中绑定了多个嵌套的 domain，`exit` 方法会退出当前 domain 下的所有 domain。

调用 `exit` 只会改变当前活动域，不会改变域本身。`enter` 和 `exit` 可以在一个 domain 中成对调用任意次。

如果调用 `exit` 的 domain 已经被释放了，那么 `exit` 会立即结束而不退出该 domain。

### domain.dispose()（已经被废弃）

请通过在域上设置的错误事件处理程序明确地从失败的IO操作恢复（不太理解 `uncertain`）。
