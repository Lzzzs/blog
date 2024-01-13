---
external: false
draft: false
title: Tapable 源码解析
date: 2024-01-12
---

# 前言
`Webpack` 是一个非常著名的打包工具，甚至可以被称作为前端的“基石”。那么这样一个工具，它内部的东西非常多，但是它也非常的灵活，特别是插件系统，用户可以通过编写插件，在 `Webpack` 的不同执行时机去实现一些自定义功能。这个能力，是借助于 `Tapable` 来实现的。

# Tapable 使用方法
Tapable 不需要把它想的多复杂，它其实就是一个**发布订阅者**的模式。

接下来我们来看看怎么用它，再通过源码去研究下它是怎么实现的

首先我们需要安装 Tapable，这里我们使用 `pnpm` 去安装

```bash
pnpm add tapable
```

然后，我们就可以在项目中使用它

```js
const { SyncHook } = require("tapable");

// 初始化同步钩子
const hook = new SyncHook(["arg1", "arg2", "arg3"]);

// 注册事件
hook.tap('flag1', (arg1,arg2,arg3) => {
    console.log('flag1:',arg1,arg2,arg3)
})

hook.tap('flag2', (arg1,arg2,arg3) => {
    console.log('flag2:',arg1,arg2,arg3)
})

// 调用事件并传递执行参数
hook.call('lzzzs','abc','cde')

// 打印结果
flag1: lzzzs abc cde
flag2: lzzzs abc cde
```
这是关于 tapable 最基本的用法。这里我们使用的 SyncHook 这个钩子，关于 tapable 它一共提供了九种钩子

```js
const {
	SyncHook,
	SyncBailHook,
	SyncWaterfallHook,
	SyncLoopHook,
	AsyncParallelHook,
	AsyncParallelBailHook,
	AsyncSeriesHook,
	AsyncSeriesBailHook,
	AsyncSeriesWaterfallHook
 } = require("tapable");
```
我们可以从两个维度去看钩子的含义

## 按照同步和异步分类
以 Sync 开头的钩子都是同步钩子，以 Async 开头的钩子都是异步钩子。
同步钩子只能通过 tap 去注册事件，并且只能通过 call 调用。
但是异步钩子可以通过 tap、tapAsync、tapPromise 去注册事件，同时通过 call、callAsync、promise 去调用。

同时异步钩子还可以进一步划分
- 串行（Series）：按照顺序调用
- 并行（Parallel）：并发调用

## 按照执行机制分类

### basic hook
最基本的钩子，它仅只关注注册的事件，不关注被事件函数的返回值。

### Waterfall
瀑布类型的钩子，它关注每个事件函数的返回值，前一个事件函数的返回值，会被当作参数传递给下一个事件函数

### Bail
保险类型钩子，只要在执行事件函数过程中，有一个函数没有返回 undefined，那么就不会继续执行下去了，也可以说叫熔断

### Loop
循环类型钩子，只要有一个事件函数没有返回 undefined，它就会重新执行所有的事件函数，直到所有的事件函数都返回 undefined

# 拦截器
Tapable 中的所有 Hook 都支持注入拦截器，例如

```js
const hook = new SyncHook(['arg1', 'arg2', 'arg3']);

hook.intercept({
  // 每次调用 hook 实例的 tap() 方法注册回调函数时, 都会调用该方法,
  // 并且接受 tap 作为参数, 还可以对 tap 进行修改;
  register: (tapInfo) => {
    console.log(`${tapInfo.name} is doing its job`);
    return tapInfo; // may return a new tapInfo object
  },
  // 通过hook实例对象上的call方法时候触发拦截器
  call: (arg1, arg2, arg3) => {
    console.log('Starting to calculate routes');
  },
  // 在调用被注册的每一个事件函数之前执行
  tap: (tap) => {
    console.log(tap, 'tap');
  },
  // loop类型钩子中 每个事件函数被调用前触发该拦截器方法
  loop: (...args) => {
    console.log(args, 'loop');
  },
});
```

# Before 和 Stage
在使用 tap 注册事件的时候，第一个参数之前传的都是字符串，其实它可以传入一个对象。

## Before
传入 before 字段，可以指定当前事件函数，在指定的事件之前进行执行

```js
const { SyncHook } = require('tapable');

const hooks = new SyncHook();

hooks.tap(
  {
    name: 'flag1',
  },
  () => {
    console.log('This is flag1 function.');
  }
);

hooks.tap(
  {
    name: 'flag2',
    // flag2 事件函数会在flag1之前进行执行
    before: 'flag1',
  },
  () => {
    console.log('This is flag2 function.');
  }
);

hooks.call();

// result
This is flag2 function.
This is flag1 function.
```
## Stage
stage 是一个数字类型，数字越大事件回调执行越晚，支持传入负数，不传时默认为 0。

```js
const hooks = new SyncHook();

hooks.tap(
  {
    name: 'flag1',
    stage: 1,
  },
  () => {
    console.log('This is flag1 function.');
  }
);

hooks.tap(
  {
    name: 'flag2',
    // 默认为stage: 0,
  },
  () => {
    console.log('This is flag2 function.');
  }
);

hooks.call();

// result
This is flag2 function.
This is flag1 function.
```
> 如果同时存在 before 和 stage，优先处理 before

# Tapable 源码实现
Tapable 的源码仓库在：https://github.com/webpack/tapable

源码的 case 很多，我们重点看 SyncHook 这个钩子的 case，去了解源码内部的大概流程是怎么样的

先看一段和 SyncHook 有关的代码
![SyncHook](/images/tapable-one.png)

按照之前看过的发布订阅的套路，应该是执行 tap，将回调函数推入队列，然后执行 call 的时候将队列的函数遍历执行一遍。但是 Tapable 有所区别，**在使用 call 执行的时候，执行的其实是一个动态更新的函数代码块，在我们每次调用 call 的时候这个函数体会动态更新，也可以叫做懒执行**

例如上面的代码，调用 call 的时候，其实在执行下面这段代码

```js
function fn(arg1, arg2) {
    "use strict";
    var _context;
    var _x = this._x;
    var _fn0 = _x[0];
    _fn0(arg1, arg2);
    var _fn1 = _x[1];
    _fn1(arg1, arg2);
}
```
从这段代码，看得出从 _x 是一个数组，然后根据下标分别执行 _x 中的函数并且传入参数。这一步就是在调用我们通过 tap 注册的事件函数。

这个代码看起来很死板，**但神奇的点就在于，每次执行 call 的时候，函数体的代码就会生成一次，保证满足需求**

## 入口文件
接下来我们根据 SyncHook 这个 hook 从入口文件，一步步看，在下面贴出来的代码，会删减一些无关的 case，让大家把流程看的更清晰。

通过 Tapable 的 package.json，我们可以看到入口文件的位置在 `lib/index.js`

```json
{
  "main": "lib/index.js",
}
```

> lib/index.js
```js
"use strict";

exports.__esModule = true;
exports.SyncHook = require("./SyncHook");
exports.SyncBailHook = require("./SyncBailHook");
exports.SyncWaterfallHook = require("./SyncWaterfallHook");
exports.SyncLoopHook = require("./SyncLoopHook");
exports.AsyncParallelHook = require("./AsyncParallelHook");
exports.AsyncParallelBailHook = require("./AsyncParallelBailHook");
exports.AsyncSeriesHook = require("./AsyncSeriesHook");
exports.AsyncSeriesBailHook = require("./AsyncSeriesBailHook");
exports.AsyncSeriesLoopHook = require("./AsyncSeriesLoopHook");
exports.AsyncSeriesWaterfallHook = require("./AsyncSeriesWaterfallHook");
exports.HookMap = require("./HookMap");
exports.MultiHook = require("./MultiHook");
```
可以看出我们使用的 SyncHook 是 lib 下的 SyncHook 导出的

## SyncHook.js
> lib/SyncHook.js
```js
"use strict";

const Hook = require("./Hook");
const HookCodeFactory = require("./HookCodeFactory");

class SyncHookCodeFactory extends HookCodeFactory {
	content({ onError, onDone, rethrowIfPossible }) {
		return this.callTapsSeries({
			onError: (i, err) => onError(err),
			onDone,
			rethrowIfPossible
		});
	}
}

const factory = new SyncHookCodeFactory();

const TAP_ASYNC = () => {
	throw new Error("tapAsync is not supported on a SyncHook");
};

const TAP_PROMISE = () => {
	throw new Error("tapPromise is not supported on a SyncHook");
};

const COMPILE = function(options) {
	factory.setup(this, options);
	return factory.create(options);
};

function SyncHook(args = [], name = undefined) {
	const hook = new Hook(args, name);
	hook.constructor = SyncHook;
	hook.tapAsync = TAP_ASYNC;
	hook.tapPromise = TAP_PROMISE;
	hook.compile = COMPILE;
	return hook;
}

SyncHook.prototype = null;

module.exports = SyncHook;
```
代码不多，但我们重点看导出的这个 SyncHook 是一个函数，也就是说我们使用的时候 new SyncHook 会执行一遍 SyncHook 函数体的内容。

在函数体中做的事情也非常简单
- new 了一个 Hook，最后将这个 hook 返回出去
- 重写 hook 上的异步方法，因为 SyncHook 没有异步的方法
- 重写了 hook 上的 compile

那我们实际调用 tap、call 方法，实际上是来自于 Hook 这个类的身上，接下来就去看看这个类

## Hook.js
> lib/Hook.js
```js
/*
	MIT License http://www.opensource.org/licenses/mit-license.php
	Author Tobias Koppers @sokra
*/
"use strict";

const util = require("util");

const deprecateContext = util.deprecate(() => {},
"Hook.context is deprecated and will be removed");

const CALL_DELEGATE = function(...args) {
	this.call = this._createCall("sync");
	return this.call(...args);
};
const CALL_ASYNC_DELEGATE = function(...args) {
	this.callAsync = this._createCall("async");
	return this.callAsync(...args);
};
const PROMISE_DELEGATE = function(...args) {
	this.promise = this._createCall("promise");
	return this.promise(...args);
};

class Hook {
	constructor(args = [], name = undefined) {
		this._args = args;
		this.name = name;
		this.taps = [];
		this.interceptors = [];
		this._call = CALL_DELEGATE;
		this.call = CALL_DELEGATE;
		this._callAsync = CALL_ASYNC_DELEGATE;
		this.callAsync = CALL_ASYNC_DELEGATE;
		this._promise = PROMISE_DELEGATE;
		this.promise = PROMISE_DELEGATE;
		this._x = undefined;

		this.compile = this.compile;
		this.tap = this.tap;
		this.tapAsync = this.tapAsync;
		this.tapPromise = this.tapPromise;
	}

	compile(options) {
		throw new Error("Abstract: should be overridden");
	}

	_createCall(type) {
		return this.compile({
			taps: this.taps,
			interceptors: this.interceptors,
			args: this._args,
			type: type
		});
	}

	_tap(type, options, fn) {
		if (typeof options === "string") {
			options = {
				name: options.trim()
			};
		} else if (typeof options !== "object" || options === null) {
			throw new Error("Invalid tap options");
		}
		if (typeof options.name !== "string" || options.name === "") {
			throw new Error("Missing name for tap");
		}
		if (typeof options.context !== "undefined") {
			deprecateContext();
		}
		options = Object.assign({ type, fn }, options);
		options = this._runRegisterInterceptors(options);
		this._insert(options);
	}

	tap(options, fn) {
		this._tap("sync", options, fn);
	}

	tapAsync(options, fn) {
		this._tap("async", options, fn);
	}

	tapPromise(options, fn) {
		this._tap("promise", options, fn);
	}

	_resetCompilation() {
		this.call = this._call;
		this.callAsync = this._callAsync;
		this.promise = this._promise;
	}

	_insert(item) {
		this._resetCompilation();
    ...
		this.taps[i] = item;
	}
}

Object.setPrototypeOf(Hook.prototype, null);

module.exports = Hook;
```

在初始化 Hook 的时候，会被带上很多属性，比如_x、taps等等，这些属性用到了再聊。

在我们实际调用 tap 注册事件函数的时候，实际调用的是 _tap。当然，像 tapAsync，tapPromise 也是调用的 _tap，只不过传入的 type 不一样。

```js
_tap(type, options, fn) {
  if (typeof options === "string") {
    options = {
      name: options.trim()
    };
  } else if (typeof options !== "object" || options === null) {
    throw new Error("Invalid tap options");
  }
  if (typeof options.name !== "string" || options.name === "") {
    throw new Error("Missing name for tap");
  }
  if (typeof options.context !== "undefined") {
    deprecateContext();
  }
  options = Object.assign({ type, fn }, options);
  options = this._runRegisterInterceptors(options);
  this._insert(options);
}
```
重点看_tap，会将 options 组装一次，变成符合 _insert 要求的格式，最后调用了_insert。

```js
_insert(item) {
  this._resetCompilation();
  ...
  this.taps.push(item)
}
```
_insert我省去了一部分逻辑，重点看最后就是将 options push 进 taps 中了，还有一个逻辑是 _resetCompilation，这个之后会提。

那我们看到这里，tap 做的事情其实非常的简单，就是将组装好的事件 options 推入一个队列中。

接下来我们看调用 call 发生了什么事情。

## call
在调用call的时候，会调用 Hook 这个类的call 方法，在 Hook 被初始化的时候，call 方法就会被初始化，并且赋值为CALL_DELEGATE
```js
const CALL_DELEGATE = function(...args) {
	this.call = this._createCall("sync");
	return this.call(...args);
};

class Hook {
	constructor(args = [], name = undefined) {
		this._call = CALL_DELEGATE;
		this.call = CALL_DELEGATE;
	}
}
```
那么我们实际调用的是CALL_DELEGATE这个函数，内部通过_createCall创建一个 call 函数，然后赋值给 this.call ，最后调用这个 this.call，**_createCall这个函数做的事情，就是我们之前说过的动态生成函数**，那我们看看它是怎么个动态生成的。

```js
_createCall(type) {
  return this.compile({
    taps: this.taps,
    interceptors: this.interceptors,
    args: this._args,
    type: type
  });
}
```
调用 Hook 中的_createCall，它会调用 compile，因为我们之前初始化 SyncHook 重写过 compile，那么我们实际调用的是 SyncHook 中的 compile

```js
class SyncHookCodeFactory extends HookCodeFactory {
	content({ onError, onDone, rethrowIfPossible }) {
		return this.callTapsSeries({
			onError: (i, err) => onError(err),
			onDone,
			rethrowIfPossible
		});
	}
}

const factory = new SyncHookCodeFactory();

const COMPILE = function(options) {
	factory.setup(this, options);
	return factory.create(options);
};
```
在调用 compile 中，它内部会通过 setup 去初始化，然后通过 create 创建函数，并且返回。

> 为什么要把 compile 在子类单独去实现呢？
其实答案很明显了，为了解耦。Tapable 将重复性代码提到父类中，然后一些不同的代码，让子类去实现，这个思想值得我们学习。

**在上面的代码通过 SyncHookCodeFactory 创建了 factory，但是 SyncHookCodeFactory 又继承于 HookCodeFactory，这里为什么这样做，其实和 SyncHook 与 Hook 的关系是一样的。都是为了解耦。**

那么也就调用到了 HookCodeFactory 中的 setup，并且通过 create 创建函数

## HookCodeFactory
```js
class HookCodeFactory {
	constructor(config) {
		this.config = config;
		this.options = undefined;
		this._args = undefined;
	}

	create(options) {
		this.init(options);
		let fn;
		switch (this.options.type) {
			case "sync":
				fn = new Function(
					this.args(),
					'"use strict";\n' +
						this.header() +
						this.contentWithInterceptors({
							onError: err => `throw ${err};\n`,
							onResult: result => `return ${result};\n`,
							resultReturns: true,
							onDone: () => "",
							rethrowIfPossible: true
						})
				);
				break;
			case "async":
        ...
				break;
			case "promise":
        ...
				break;
		}
		this.deinit();
		return fn;
	}

	setup(instance, options) {
		instance._x = options.taps.map(t => t.fn);
	}

  init(options) {
		this.options = options;
		this._args = options.args.slice();
	}

	deinit() {
		this.options = undefined;
		this._args = undefined;
	}
}
```
在 setup 中，只是将每个事件函数取了出来，放在了_x这个数据中。

然后调用 create 方法，首先调用 init 方法，在 this 上初始化一些数据，最后在通过 deinit 将初始化的数据清空。

之后走 sync 这个 case
```js
case "sync":
  fn = new Function(
    this.args(),
    '"use strict";\n' +
      this.header() +
      this.contentWithInterceptors({
        onError: err => `throw ${err};\n`,
        onResult: result => `return ${result};\n`,
        resultReturns: true,
        onDone: () => "",
        rethrowIfPossible: true
      })
  );
  break;
```
new Function第一个参数是参数，第二个是函数体。重点看函数体，函数体前面有一个`'"use strict";\n'`，然后拼接 this.header，以及 this.contentWithInterceptors 这两个函数的返回值，就形成了最终的函数

### header
```js
header() {
  let code = "";
  if (this.needContext()) {
    code += "var _context = {};\n";
  } else {
    code += "var _context;\n";
  }
  code += "var _x = this._x;\n";
  if (this.options.interceptors.length > 0) {
    code += "var _taps = this.taps;\n";
    code += "var _interceptors = this.interceptors;\n";
  }
  return code;
}
```
因为我们没有设计上下文和拦截器，这里返回的 code 就是`var _context;\n var _x = this._x;\n`

### contentWithInterceptors

```js
contentWithInterceptors(options) {
  if (this.options.interceptors.length > 0) {
    ...
  } else {
    return this.content(options);
  }
}
```
因为我们没有拦截器，所以直接执行 content 方法，但是这个 content 已经被子类重写了，所以我们调用的是子类的 content 的方法，这也就将子类的逻辑抽离了出去。

```js
class SyncHookCodeFactory extends HookCodeFactory {
	content({ onError, onDone, rethrowIfPossible }) {
		return this.callTapsSeries({
			onError: (i, err) => onError(err),
			onDone,
			rethrowIfPossible
		});
	}
}
```
在子类又调用了父类的callTapsSeries方法，这里是有点绕的，需要好好理解下。

### callTapsSeries
```js
class HookCodeFactory {
  // 根据this._x生成整体函数内容
  callTapsSeries({ onDone }) {
    let code = '';
    let current = onDone;
    // 没有注册的事件则直接返回
    if (this.options.taps.length === 0) return onDone();
    // 遍历taps注册的函数 编译生成需要执行的函数
    for (let i = this.options.taps.length - 1; i >= 0; i--) {
      const done = current;
      // 一个一个创建对应的函数调用
      const content = this.callTap(i, {
        onDone: done,
      });
      current = () => content;
    }
    code += current();
    return code;
  }

  // 编译生成单个的事件函数并且调用 比如 fn1 = this._x[0]; fn1(...args)
  callTap(tapIndex, { onDone }) {
    let code = '';
    // 无论什么类型的都要通过下标先获得内容
    // 比如这一步生成 var _fn[1] = this._x[1]
    code += `var _fn${tapIndex} = ${this.getTapFn(tapIndex)};\n`;
    // 不同类型的调用方式不同
    // 生成调用代码 fn1(arg1,arg2,...)
    const tap = this.options.taps[tapIndex];
    switch (tap.type) {
      case 'sync':
        code += `_fn${tapIndex}(${this.args()});\n`;
        break;
      // 其他类型不考虑
      default:
        break;
    }
    if (onDone) {
      code += onDone();
    }
    return code;
  }
  
  // 从this._x中获取函数内容 this._x[index]
  getTapFn(idx) {
    return `_x[${idx}]`;
  }
}
```
这一部分就是生成函数体的核心代码了，代码还是比较复杂的，但是我们这个 case 生成的代码是 easy 的。最后执行通过执行这个函数生成的代码是

```js
var _fn0 = _x[0];
_fn0(arg1, arg2);
var _fn1 = _x[1];
_fn1(arg1, arg2);
```
到这，整个函数就被动态创建完成了，最后执行的就是前面之前的那个函数，也其实就是将 taps 一个个拿出来去执行，函数逻辑还是比较简单的，但是难就难在这个动态生成。

在前文我们留有一个悬念，在执行 tap 最后调用 _insert的时候会执行 _resetCompilation 这个函数，那么这个函数是为了干什么呢？
```js
_insert(item) {
  this._resetCompilation();
  ...
  this.taps.push(item)
}
```
我们先看一个例子
```js
const { SyncHook } = require('tapable')

const hooks = new SyncHook(['arg1','arg2'])

hooks.tap('flag1', () => {
    console.log(1)
})

hooks.tap('flag',() => {
    console.log(2)
})

hooks.call('arg1', 'arg2')

// 再次添加一个tap事件函数
hooks.tap('flag3', () => {
  console.log(3);
});

// 同时再次调用
hooks.call('arg1', 'arg2');
```
如果没有 _resetCompilation 这个函数，我们再次调用的 call 函数其实是之前那次 call 函数，也就是说 call 并没有动态生成，会出现这个 bug。那为什么会出现这个 bug 呢？因为在我们生成函数的时候，我们会将 最新生成的函数放到 call 上，那么之后的代码也就是直接执行之前的 call 了。我们看看 _resetCompilation 做了什么事情帮我们避免了这个问题。

```js
const CALL_DELEGATE = function(...args) {
	this.call = this._createCall("sync");
	return this.call(...args);
};

class Hook {
  constructor(args = [], name = undefined) {
    this._call = CALL_DELEGATE;
    this.call = CALL_DELEGATE;
  }

  _resetCompilation() {
    this.call = this._call;
    this.callAsync = this._callAsync;
    this.promise = this._promise;
  }
}
```
**_resetCompilation 做的事情也非常简单，将 call 重置为之前的默认函数，这样就可以解决了。在我们每一次通过 tap 注册事件的时候，就会重置一次，那么我们每次调用 call 的时候，函数体都是当前最完美的函数。**

# 总结
所以到这，我们总结下，关于 Tapable 的流程。

**本质上 Tapable 就是一个发布订阅的模型，但在通知发布的时候，是通过动态编译生成一个 Function，然后根据具体的行为去执行当前这个事件的所有回调函数**

> reference: 
> - https://github.com/webpack/tapable
> - https://juejin.cn/post/7040982789650382855