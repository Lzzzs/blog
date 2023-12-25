---
external: false
draft: false
title: Vue 原理剖析-异步组件
date: 2023-05-26
---

### 前言

在我们平时开发中，为了减少首屏代码的体积，我们会采用异步组件的方法将一些非首屏的组件变成异步组件，进行按需加载

本文我们来研究下`Vue`的异步组件是如何实现的

<!--more-->

### 写法
- 普通函数写法
```js
Vue.component("async-component", function(resolve, reject) {
  setTimeout(() => {
    resolve({
      template: '<div></div>'
    })
  }, 2000)
})
```
- Promise写法
```js
Vue.component(
  'async-webpack-example',
  // 这个动态导入会返回一个 `Promise` 对象。
  () => import('./my-async-component')
)
```
- 高级异步组件写法
```js
Vue.component('async-webpack-example', () => ({
  // 需要加载的组件 (应该是一个 `Promise` 对象)
  component: import('./MyComponent.vue'),
  // 异步组件加载时使用的组件
  loading: LoadingComponent,
  // 加载失败时使用的组件
  error: ErrorComponent,
  // 展示加载时组件的延时时间。默认值是 200 (毫秒)
  delay: 200,
  // 如果提供了超时时间且组件加载也超时了，
  // 则使用加载失败时使用的组件。默认值是：`Infinity`
  timeout: 3000
}))
```
接下来我们通过这三种写法来研究异步组件是如何实现的

### createComponent
**在我们生成组件VNode的时候，此时才会对异步组件进行处理，我们直接看这个逻辑**

> src/core/vdom/create-component.js
```js
export function createComponent (
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
  }
  
  // ...

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }
}
```
我们省略了部分逻辑，留下了主要逻辑。Ctor也就是我们Vue.component的第二个参数，那么正常的话就是传入的一个对象，也就是会进入到`if (isObject(Ctor))`判断里面，将Ctor扩展成Vue子类。那我们作为异步组件传入的都是函数，那么就会走下面`if (isUndef(Ctor.cid))`，因为我们没有cid。

之后会调用`resolveAsyncComponent`

#### resolveAsyncComponent
> src/core/vdom/helpers/resolve-async-component.js
```js
export function resolveAsyncComponent(
  factory: Function,
  baseCtor: Class<Component>
): Class<Component> | void {
  if (isTrue(factory.error) && isDef(factory.errorComp)) {
    return factory.errorComp;
  }

  if (isDef(factory.resolved)) {
    return factory.resolved;
  }

  const owner = currentRenderingInstance;
  if (owner && isDef(factory.owners) && factory.owners.indexOf(owner) === -1) {
    // already pending
    factory.owners.push(owner);
  }

  if (isTrue(factory.loading) && isDef(factory.loadingComp)) {
    return factory.loadingComp;
  }

  if (owner && !isDef(factory.owners)) {
    const owners = (factory.owners = [owner]);
    let sync = true;
    let timerLoading = null;
    let timerTimeout = null;

    (owner: any).$on("hook:destroyed", () => remove(owners, owner));

    const forceRender = (renderCompleted: boolean) => {
      for (let i = 0, l = owners.length; i < l; i++) {
        (owners[i]: any).$forceUpdate();
      }

      if (renderCompleted) {
        owners.length = 0;
        if (timerLoading !== null) {
          clearTimeout(timerLoading);
          timerLoading = null;
        }
        if (timerTimeout !== null) {
          clearTimeout(timerTimeout);
          timerTimeout = null;
        }
      }
    };

    const resolve = once((res: Object | Class<Component>) => {
      // cache resolved
      factory.resolved = ensureCtor(res, baseCtor);
      // invoke callbacks only if this is not a synchronous resolve
      // (async resolves are shimmed as synchronous during SSR)
      if (!sync) {
        forceRender(true);
      } else {
        owners.length = 0;
      }
    });

    const reject = once((reason) => {
      process.env.NODE_ENV !== "production" &&
        warn(
          `Failed to resolve async component: ${String(factory)}` +
            (reason ? `\nReason: ${reason}` : "")
        );
      if (isDef(factory.errorComp)) {
        factory.error = true;
        forceRender(true);
      }
    });

    const res = factory(resolve, reject);

    if (isObject(res)) {
      if (isPromise(res)) {
        // Promise异步组件走这
        // Vue.component(
        //   'async-webpack-example',
        //   // 该 `import` 函数返回一个 `Promise` 对象。
        //   () => import('./my-async-component')
        // )
        if (isUndef(factory.resolved)) {
          res.then(resolve, reject);
        }
      } else if (isPromise(res.component)) {
        // 高级异步组件初始化走这
        // const AsyncComp = () => ({
        //   // 需要加载的组件。应当是一个 Promise
        //   component: import("./MyComp.vue"),
        //   // 加载中应当渲染的组件
        //   loading: LoadingComp,
        //   // 出错时渲染的组件
        //   error: ErrorComp,
        //   // 渲染加载中组件前的等待时间。默认：200ms。
        //   delay: 200,
        //   // 最长等待时间。超出此时间则渲染错误组件。默认：Infinity
        //   timeout: 3000,
        // });
        // Vue.component("async-example", AsyncComp);

        res.component.then(resolve, reject);

        if (isDef(res.error)) {
          factory.errorComp = ensureCtor(res.error, baseCtor);
        }

        if (isDef(res.loading)) {
          factory.loadingComp = ensureCtor(res.loading, baseCtor);
          if (res.delay === 0) {
            factory.loading = true;
          } else {
            timerLoading = setTimeout(() => {
              timerLoading = null;
              if (isUndef(factory.resolved) && isUndef(factory.error)) {
                factory.loading = true;
                forceRender(false);
              }
            }, res.delay || 200);
          }
        }

        if (isDef(res.timeout)) {
          timerTimeout = setTimeout(() => {
            timerTimeout = null;
            if (isUndef(factory.resolved)) {
              reject(
                process.env.NODE_ENV !== "production"
                  ? `timeout (${res.timeout}ms)`
                  : null
              );
            }
          }, res.timeout);
        }
      }
    }

    sync = false;
    // return in case resolved synchronously
    return factory.loading ? factory.loadingComp : factory.resolved;
  }
}
```

这个函数代码比较多，因为它包含了我们之前说的三种写法的判断，我们每次分析单独分析一种写法。


##### 普通函数写法
作为普通函数写法，我们肯定会进入到`if (owner && !isDef(factory.owners))`这个if中，owner其实就是组件实例，不用太多关注。进入其中，肯定会执行`factory(resolve, reject)`，那么此时的factory也就是我们Vue.component的第二个参数，那么也就是会调用我们传入的参数，然后将内部的resolve和reject作为参数传入给我们定义的函数，我们的函数内部是通过定时器设置为两秒后调用了resolve，那么我们看下resolve的实现

```js
const resolve = once((res: Object | Class<Component>) => {
  // cache resolved
  factory.resolved = ensureCtor(res, baseCtor);
  // invoke callbacks only if this is not a synchronous resolve
  // (async resolves are shimmed as synchronous during SSR)
  if (!sync) {
    forceRender(true);
  } else {
    owners.length = 0;
  }
});

// src/shared/util.js
export function once (fn: Function): Function {
  let called = false
  return function () {
    if (!called) {
      called = true
      fn.apply(this, arguments)
    }
  }
}
```
调用resolve，也就是调用了once函数传入了一个函数作为其参数。那么once函数其实就是让通过闭包保证我们的resolve函数只会执行一次。那么传入的函数被调用，首先执行`ensureCtor`。
```js
function ensureCtor(comp: any, base) {
  if (comp.__esModule || (hasSymbol && comp[Symbol.toStringTag] === "Module")) {
    comp = comp.default;
  }
  return isObject(comp) ? base.extend(comp) : comp;
}
```
我们执行ensureCtor传入的res，就是我们外部调用resolve传入的参数。那么ensureCtor内部会判断传入的是不是对象，是对象就扩展成Vue实例，也就成为了一个组件。那此时，这个组件被赋值到了`factory.resolved`。接着往下走sync肯定是false，所以会执行forceRender。forceRender就是拿到所有与自己有关联的组件Watcher然后update一下，这个时候又会重新更新，那么又会重新走到resolveAsyncComponent这个函数内部，那么在前面的判断中有一个`if (isDef(factory.resolved))`，因为我们刚才将组件赋值给了`factory.resolved`，那么此时就存在，也就直接return出去了，这就是我们普通函数写法的逻辑。

当然其中两种写法也类似，只不过多了一些参数

##### Promise写法
Promise写法和普通函数写法非常类似，只不过调用factory返回的是Promise，那么会走到`if (isPromise(res))`中，内部`res.then(resolve, reject)`，在异步拿到结果的时候会回调resolve，这个时候和我们普通函数写法的过程就一样了，就不重复了。

##### 高级异步组件写法
高级异步组件写法执行factory返回的res.component是一个Promise，所以会走到`else if (isPromise(res.component))`。之后会调用`res.component.then(resolve, reject)`等待异步之后完之后会调用resolve或reject。因为这是异步的过程，不会立即调用。先会走后面逻辑，如果定义了res.error就会把res.error扩展成Vue组件赋值到factory.errorComp。如果定义了`res.loading`同样会这样做，但是内部对delay是否为0做了一个判断，如果是0，那么factory.loading = true。那这样有啥作用呢，我们直接看`return factory.loading ? factory.loadingComp : factory.resolved;`，如果为true，那么直接返回的是loading的组件，也就是我们外面设置的加载组件。

如果delay不为0，会调用setTimeout默认延时200毫秒，内部调用forceRender重新渲染，之后过200毫秒才会生成loading组件，也就是说延迟显示loading组件

那么如果有res.timeout，和刚才分析的逻辑一样，也是利用setTimeout，等超过了这个时间就会reject抛出错误

### createAsyncPlaceholder
在我们内部调用`resolveAsyncComponent`生成Ctor之后会对此判断，如果是undefined，就会调用`createAsyncPlaceholder`生成一个注释节点，也可以称之为占位节点，那么在什么情况下Ctor不是undefined呢。在我们定义了loading组件的情况下，我们回想下`resolveAsyncComponent`最后的三元判断符，如果有loading comp那么就会返回，否则返回factory.resolved，那因为异步的原因，此时`factory.resolved`还没东西，所以就用占位符替代，等异步组件加载完成之后再重新渲染

```js
  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }
```

### 总结
异步组件其实就是一个多次渲染的过程，刚开始还没拿到组件时，渲染一个注释节点作为占位。之后拿到组件时，调用内部`resolve`函数，通过`forceRender`重新渲染，再走到之前的逻辑，此时就有组件了，然后再渲染这个组件。