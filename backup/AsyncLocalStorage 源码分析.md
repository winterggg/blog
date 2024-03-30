# AsyncLocalStorage 源码分析

## TL; DR

本文概述了 `AsyncLocalStorage` 的概念和它在 Node.js 中的历史发展。随后简要探讨了 `AsyncLocalStorage` 的部分源码，以阐明其在异步编程中如何管理上下文状态。最后简单分析了 `AsyncLocalStorage` 在 `fx-code` 项目中的实际应用场景。

## What, Why & History

### What & Why?

`AsyncLocalStorage` （后文简写为 `ALS`）是 `Node.js` 中的一个 API，它提供了一种在异步调用链中存储和传递数据的方法。

这个类属于 `async_hooks` 模块，类似于 `Python` 的 `contextvars`、`Java` 的 `ThreadLocal` ，能够让程序在一个异步上下文中存储数据。

`ALS` 的典型应用场景包括：请求范围内跟踪用户 auth、日志记录、事务追踪、Metrics 等等。

### ALS 的历史

在 `ALS` 之前，Node 社区曾涌现过各种解决异步上下文跟踪的方案，包括 `domain`、`CLS` 和 `CLS-hooked`：

- （2012年左右）`domain` 是 Node 古早时期用来解决异步代码错误处理的 api，也可以用来维持异步上下文。它的实现涉及到对 Node 的一些底层 API 的魔改，主要是涉及到事件循环的部分，例如 `nextTick`、`setTimeout` 以及大部分异步 IO 的操作，以确保上下文的维持和拦截其中的错误。随之而来的缺点是，由于该模块魔改了全局函数，很容易出现奇奇怪怪的副作用，并且有非常严重的性能问题。目前已被废弃，不再推荐使用。
- （2013 年）`Continuation-Local Storage (CLS)` 是一个社区维护的库，其实原理类似于 `domain` ，大概就是给 Node.js 的核心异步代码打猴子补丁，依赖的应该是 `process.addAsyncListener` 这个古老（node v0.11）的API，后续通过 polyfill 来兼容。更深入一点其实就是维护了一个全局的栈结构，用来实现一些嵌套的上下文切换。
- （2017 年）2017年，node 官方释出了 `async_hooks` API，给出了一个 hook 异步调用生命周期各阶段的官方实现。热心网友立马就 fork 了 `CLS` 并基于 `async_hooks` 重写，也就是`CLS-Hooked`库。不过由于 `async_hooks` 的很多 api 其实一直处于 stage-1 实验阶段，依赖于它的 `CLS-Hooked` 也并不推荐在生产环境使用。
- (2019年，Node 13.10, 12.17) 由于需求太多，官方终于在 19 年随 Node 13.10 发布了 `ALS`，并随后加入到了 12.17 LTS 里。不过刚出来的时候，`ALS` 其实有比较严重的[性能问题](https://github.com/nodejs/node/issues/34493)的。2020 年底，Qard 使用新的 `v8::Context PromiseHook` API [重构了 async_hooks](https://github.com/nodejs/node/pull/36394)，随后的 benchmarks 显示，性能问题已经得到了大幅度的缓解。

简单来说，`domain` 和 `CLS`的实现方式是给 node 打猴子补丁（运行时补丁），而 `CLS-hooked` 和 `ALS` 则是依赖官方的 `async_hooks` API。经过一系列的优化，`ALS` 的性能损耗已经足够低，官方也推荐使用 `ALS` 来追踪异步执行上下文。

## 使用方式

根据[官方文档](https://nodejs.org/docs/latest-v18.x/api/async_context.html)，`ALS` 的主要 API 如下：

- new AsyncLocalStorage()：创建一个 ALS 实例；
- asyncLocalStorage.run(store, callback[, ...args]): 在 store 的上下文中运行指定的回调函数。回调函数之外无法访问 store，回调内的**任何异步操作**都能访问 store。并且如果回调出错，run 也会抛出错误，且上下文正常退出，stack trace 不会被污染；
- asyncLocalStorage.getStore(): 获得当前上下文 store，如果没有调用过 run 或者 enterwith，将返回 undefined；
- 【stage 1 - 实验中】asyncLocalStorage.enterWith(store): 从当前行开始进入 store 的上下文，直到当前同步执行结束；
- 【stage 1 - 实验中】asyncLocalStorage.disable(): 清理 store。

官网的一个例子：

```js
import http from 'node:http';
import { AsyncLocalStorage } from 'node:async_hooks';

const asyncLocalStorage = new AsyncLocalStorage();

function logWithId(msg) {
  const id = asyncLocalStorage.getStore();
  console.log(`${id !== undefined ? id : '-'}:`, msg);
}

let idSeq = 0;
http.createServer((req, res) => {
  asyncLocalStorage.run(idSeq++, () => {
    logWithId('start');
    // Imagine any chain of async operations here
    setImmediate(() => {
      logWithId('finish');
      res.end();
    });
  });
}).listen(8080);

http.get('http://localhost:8080');
http.get('http://localhost:8080');
// Prints:
//   0: start
//   1: start
//   0: finish
//   1: finish
```

在这个例子中，我们创建了一个 `AsyncLocalStorage` 实例，并使用中间件来为每个请求分配一个唯一 ID。使用 `run` 方法可以在当前异步执行上下文中绑定请求 ID，这样，在这个异步执行上下文创建的任何异步事件中，我们都可以通过 `getStore` 方法得到同一个 ID，而无需显式地将请求 ID 透传给每个函数。

## 实现原理

`ALS` 的原理其实非常简单，下面以 `Node 18.13` 源码为例进行简要分析。

`ALS` 的源码中 js 部分在代码仓库的 `lib/async_hooks.js` 文件里，首先看 `run` 的实现：

```js
run(store, callback, ...args) {
    // Avoid creation of an AsyncResource if store is already active
    if (ObjectIs(store, this.getStore())) {
      return ReflectApply(callback, null, args);
    }

    this._enable();

    const resource = executionAsyncResource();
    const oldStore = resource[this.kResourceStore];

    resource[this.kResourceStore] = store;

    try {
      return ReflectApply(callback, null, args);
    } finally {
      resource[this.kResourceStore] = oldStore;
    }
}
```

其中, 

- `this._enable()` 用于启动 `ALS` 的一个 init hook，这个稍后再聊。

- `executionAsyncResource()`: 这是 `async_hooks` 模块的一个函数，它返回代表当前执行异步资源的对象。在 Node.js 中，**每个**异步操作都与一个 resource 对象相关联，它代表了操作的上下文。
- `this.kResourceStore`: 这个其实是 ALS 构造函数里创建的一个 Symbol，用来往 resource 里存 store 时的 key。

可以看出，`run` 主要做了这样一件事情：备份当前的 `store`，然后替换为 run 函数入参提供的新 `store`，然后执行入参里的回调函数，这样，后续的异步操作都可以通过这个 resource 访问到新的 `store`。

当然，回调函数里也能继续嵌套调用 run，而这里其实是借用函数的调用栈结构，实现了一个 store 的切换与重置。



到这里，回调函数里的所有同步函数都能正确的通过 `getStore` 获得对应的存储：

```js
  getStore() {
    if (this.enabled) {
      const resource = executionAsyncResource();
      return resource[this.kResourceStore];
    }
  }
```

不过，万一回调里开启了新的异步操作呢？这时executionAsyncResource() 会得到一个全新的 resource。因此，我们需要一个机制去在调用处传递 resource 里的 store。

这其实是通过 `async_hook` API 来实现的，具体来说，node 在全局维护了一个 store 列表：

```js
const storageList = [];
const storageHook = createHook({
  init(asyncId, type, triggerAsyncId, resource) {
    const currentResource = executionAsyncResource();
    // Value of currentResource is always a non null object
    for (let i = 0; i < storageList.length; ++i) {
      storageList[i]._propagate(resource, currentResource, type);
    }
  }
});

// als._enable()
  _enable() {
    if (!this.enabled) {
      this.enabled = true;
      ArrayPrototypePush(storageList, this);
      storageHook.enable();
    }
  }
```

如上面的代码所示，`ALS` 创建了一个 `init` hook，首次调用 `run` 时，会把当前 store 对象存入全局 store 列表并启动 hook。这样，每次进入新的异步操作前，会将旧 resource 中的所有 store 实例传递给新的 resource，这样在异步操作内，也能正常通过 `getStore` 获取 store 啦！

至于剩下两个函数 `enterWith` 和 `disable`，实现比较简单。扫一眼代码就大概知道是在干嘛了：

```js
enterWith(store) {
  this._enable();
  const resource = executionAsyncResource();
  resource[this.kResourceStore] = store;
}


disable() {
    if (this.enabled) {
      this.enabled = false;
      // If this.enabled, the instance must be in storageList
      ArrayPrototypeSplice(storageList,
                           ArrayPrototypeIndexOf(storageList, this), 1);
      if (storageList.length === 0) {
        storageHook.disable();
      }
    }
  }
```

## FX-CODE 里的应用

fx-code 工程里的 nstarter-core 依赖里使用 `ALS` 封装了一个异步上下文管理器，主要用来维护一个请求的 traceId。另外 james 之前做 metrics 的时候也用它来存储一些请求范围的 trace 信息。

实现上来说，在 `node_modules/nstarter-core/src/context/provider.ts` 里定义了 `ContextProvider` 类，其中 `getMiddleware` 返回一个中间件，将 next 函数包括在了 als 的上下文里。这样每一个新的请求，都能通过静态方法 `ContextProvider.getContext()` 获取请求上下文了：

``` ts
private getMiddleware(options: IContextMiddlewareOptions): RequestHandler {
    const opts: IContextMiddlewareOptions = {
        idGenerator: defaultIdGenerator,
        ...options
    };
    return (req: Request, res: Response, next: NextFunction) =>
        this._localStorage.run(new this._Context(), () => {
            const context = this.context;
            const traceId = opts.idGenerator?.(req);
            if (context && traceId) {
                context.setTraceId(traceId);
            }
            return next();
        });
}

/**
 * 获取上下文对象
 */
private get context(): T | undefined {
    return this._localStorage.getStore();
}

public static getContext<T extends BaseContext>(): T | undefined {
    try {
        return ContextProvider.getInstance<T>().context;
    } catch (err) {
        return;
    }
}
```



## 上下文丢失的问题

在大多数情况下， `AsyncLocalStorage` 可以正常工作，对于基于回调的函数，只需要使用 [`util.promisify()`](https://nodejs.org/docs/latest-v18.x/api/util.html#utilpromisifyoriginal) 将其转换为 `Promise` 就可以了。

在少数情况下可能会发生上下文丢失，这时只需要通过 `AsyncResource` API 恰当地绑定好上下文即可。当你发现上下文丢失时，可以通过在适当的位置执行 `getStore` 来推理造出上下文丢失的函数调用并解决修复。



例如，由 `EventEmitter` 触发的事件监听器可能在调用 `eventEmitter.on()` 时的执行上下文中运行。下面展示了一个问题场景以及对应的修复方式，其他类似的基于事件驱动的场景也可以用同样的方式修复：

```js
import { createServer } from 'node:http';
import { AsyncResource, executionAsyncId } from 'node:async_hooks';

const server = createServer((req, res) => {
  req.on('close', AsyncResource.bind(() => {
    // Execution context is bound to the current outer scope.
  }));
  req.on('close', () => {
    // Execution context is bound to the scope that caused 'close' to emit.
  });
  res.end();
}).listen(3000);
```