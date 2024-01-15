---
external: false
draft: false
title: Vue-Router 原理剖析-导航守卫
date: 2023-06-15
---

### 前言
本文我们来研究下 `Vue-Router` 中**导航守卫**的实现以及流程

为什么这一篇要单独拿出来讲呢，因为其中实现比较复杂以及巧妙，所以我觉得完全可以抽出来

上一篇讲到`confirmTransition`这个函数的时候没有进去看，因为我说这里面是关于导航守卫的逻辑，那我们就直接从这个函数开始

### 导航守卫执行顺序
在开始之前，我们先要知道导航守卫的执行顺序，我直接把官网的顺序贴在下面了

- 导航被触发。
- 在失活的组件里调用 beforeRouteLeave 守卫。
- 调用全局的 beforeEach 守卫。
- 在重用的组件里调用 beforeRouteUpdate 守卫 (2.2+)。
- 在路由配置里调用 beforeEnter。
- 解析异步路由组件。
- 在被激活的组件里调用 beforeRouteEnter。
- 调用全局的 beforeResolve 守卫 (2.5+)。
- 导航被确认。
- 调用全局的 afterEach 钩子。
- 触发 DOM 更新。
- 调用 beforeRouteEnter 守卫中传给 next 的回调函数，创建好的组件实例会作为回调函数的参数传入。

### confirmTransition
```js
confirmTransition (route: Route, onComplete: Function, onAbort?: Function) {
  const current = this.current
  this.pending = route

  const abort = err => {
    // changed after adding errors with
    // https://github.com/vuejs/vue-router/pull/3047 before that change,
    // redirect and aborted navigation would produce an err == null
    if (!isNavigationFailure(err) && isError(err)) {
      if (this.errorCbs.length) {
        this.errorCbs.forEach(cb => {
          cb(err)
        })
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn(false, 'uncaught error during route navigation:')
        }
        console.error(err)
      }
    }
    onAbort && onAbort(err)
  }
  const lastRouteIndex = route.matched.length - 1
  const lastCurrentIndex = current.matched.length - 1

  if (
    isSameRoute(route, current) &&
    // in the case the route map has been dynamically appended to
    lastRouteIndex === lastCurrentIndex &&
    route.matched[lastRouteIndex] === current.matched[lastCurrentIndex]
  ) {
    this.ensureURL()
    if (route.hash) {
      handleScroll(this.router, current, route, false)
    }
    return abort(createNavigationDuplicatedError(current, route))
  }

  const { updated, deactivated, activated } = resolveQueue(
    this.current.matched,
    route.matched
  )

  const queue: Array<?NavigationGuard> = [].concat(
    // in-component leave guards
    extractLeaveGuards(deactivated),
    // global before hooks
    this.router.beforeHooks,
    // in-component update hooks
    extractUpdateHooks(updated),
    // in-config enter guards
    activated.map(m => m.beforeEnter),
    // async components
    resolveAsyncComponents(activated)
  )

  const iterator = (hook: NavigationGuard, next) => {
    if (this.pending !== route) {
      return abort(createNavigationCancelledError(current, route))
    }
    try {
      hook(route, current, (to: any) => {
        if (to === false) {
          // next(false) -> abort navigation, ensure current URL
          this.ensureURL(true)
          abort(createNavigationAbortedError(current, route))
        } else if (isError(to)) {
          this.ensureURL(true)
          abort(to)
        } else if (
          typeof to === 'string' ||
          (typeof to === 'object' &&
            (typeof to.path === 'string' || typeof to.name === 'string'))
        ) {
          // next('/') or next({ path: '/' }) -> redirect
          abort(createNavigationRedirectedError(current, route))
          if (typeof to === 'object' && to.replace) {
            this.replace(to)
          } else {
            this.push(to)
          }
        } else {
          // confirm transition and pass on the value
          next(to)
        }
      })
    } catch (e) {
      abort(e)
    }
  }

  runQueue(queue, iterator, () => {
    // wait until async components are resolved before
    // extracting in-component enter guards
    const enterGuards = extractEnterGuards(activated)
    const queue = enterGuards.concat(this.router.resolveHooks)
    runQueue(queue, iterator, () => {
      if (this.pending !== route) {
        return abort(createNavigationCancelledError(current, route))
      }
      this.pending = null
      onComplete(route)
      if (this.router.app) {
        this.router.app.$nextTick(() => {
          handleRouteEntered(route)
        })
      }
    })
  })
}
```
我们先重点看这个函数的执行逻辑`resolveQueue`，它传入`current.matched`和`route.matched`，然后函数返回了三个值

### resolveQueue
```js
function resolveQueue (
  current: Array<RouteRecord>,
  next: Array<RouteRecord>
): {
  updated: Array<RouteRecord>,
  activated: Array<RouteRecord>,
  deactivated: Array<RouteRecord>
} {
  let i
  const max = Math.max(current.length, next.length)
  for (i = 0; i < max; i++) {
    if (current[i] !== next[i]) {
      break
    }
  }
  return {
    updated: next.slice(0, i),
    activated: next.slice(i),
    deactivated: current.slice(i)
  }
}
```
`resolveQueue`函数逻辑很简单，因为我们是从current切换到next路径上，所以它们先匹配不一样的位置i，i前面就是一样的，后面就是不一样的。所以说，updated就可以是一样的位置，因为页面切换一样的话就是更新。那么activated就只属于新的路径，所以取i后面，也就是要激活的路径。deactivated也就是失活的，那切换路径，只有之前的路径而且不相同的地方会失活，那么就取之前的路径i后面的元素。

这样我们就能拿到`updated`、`activated`、 `deactivated` 这三个数组

### queue
confirmTransition 内部有这么一段逻辑，是通过我们之前的那三个数组生成一个队列
```js
  const queue: Array<?NavigationGuard> = [].concat(
    // in-component leave guards
    extractLeaveGuards(deactivated),
    // global before hooks
    this.router.beforeHooks,
    // in-component update hooks
    extractUpdateHooks(updated),
    // in-config enter guards
    activated.map(m => m.beforeEnter),
    // async components
    resolveAsyncComponents(activated)
  )
```
我们先看`extractLeaveGuards`，内部调用了`extractGuards`
```js
function extractLeaveGuards (deactivated: Array<RouteRecord>): Array<?Function> {
  return extractGuards(deactivated, 'beforeRouteLeave', bindGuard, true)
}

function bindGuard (guard: NavigationGuard, instance: ?_Vue): ?NavigationGuard {
  if (instance) {
    return function boundRouteGuard () {
      return guard.apply(instance, arguments)
    }
  }
}
```

关于`extractGuards`函数的代码就比较复杂了，我们一步步拆开来看
```js
function extractGuards (
  records: Array<RouteRecord>,
  name: string,
  bind: Function,
  reverse?: boolean
): Array<?Function> {
  const guards = flatMapComponents(records, (def, instance, match, key) => {
    const guard = extractGuard(def, name)
    if (guard) {
      return Array.isArray(guard)
        ? guard.map(guard => bind(guard, instance, match, key))
        : bind(guard, instance, match, key)
    }
  })
  return flatten(reverse ? guards.reverse() : guards)
}

function extractGuard (
  def: Object | Function,
  key: string
): NavigationGuard | Array<NavigationGuard> {
  if (typeof def !== 'function') {
    // extend now so that global mixins are applied.
    def = _Vue.extend(def)
  }
  return def.options[key]
}

export function flatMapComponents (
  matched: Array<RouteRecord>,
  fn: Function
): Array<?Function> {
  return flatten(matched.map(m => {
    return Object.keys(m.components).map(key => fn(
      m.components[key],
      m.instances[key],
      m, key
    ))
  }))
}

export function flatten (arr: Array<any>): Array<any> {
  return Array.prototype.concat.apply([], arr)
}

```
首先调用了`flatMapComponents`，传入了records，以及一个函数。在`flatMapComponents`内部，对传入的matched遍历，取出内部的components的key，然后循环调用fn函数，这个fn函数就是我们外面传进来的。

在fn中执行了`extractGuard`，`extractGuard`内部会extend生成def，从def中拿到beforeRouteLeave函数，然后在fn内部会判断返回的guard是不是一个数组，是数组的话就给每个guard生成一个新函数，绑定this，如果不是，那么就生成单个的。

然后此时在fn执行的返回值，有两种可能，第一种是数组，第二种就是函数，然后作为map的返回值返回回去，之后还会调用flatten，对数组做降维，也就是拍平数组。

这个时候`flatMapComponents`执行完了，返回guards，如果外部传入的参数是需要翻转的，那会先翻转，否则拍平之后返回出去

那么到这`extractGuards`执行完了，被当做`extractLeaveGuards`的返回值返回出去
所以`extractLeaveGuards`的返回值可能是一个一维数组，里面都是函数，函数内部又调用了导航守卫的钩子

之后继续看`this.router.beforeHooks`，这个是全局的钩子，也就是全局注册的时候会被注入

然后看`extractUpdateHooks`，可以发现和`extractLeaveGuards`逻辑是一样的，只不过这次拿的是`beforeRouteUpdate`。

```js
function extractUpdateHooks (updated: Array<RouteRecord>): Array<?Function> {
  return extractGuards(updated, 'beforeRouteUpdate', bindGuard)
}
```

接着看`activated.map(m => m.beforeEnter)`，也就是将activated中所有的beforeEnter取出返回

然后是`resolveAsyncComponents`这一部分是异步组件，本质还是返回一个数组，里面有函数

之后再通过`concat`拼接数组，也就是queue中全是我们需要执行的函数

### runQueue
我们组装好的queue，通过`runQueue`去执行队列，第一个参数是queue，第二个参数是iterator，第三个参数是成功的回调
```js
runQueue(queue, iterator, () => {
  // wait until async components are resolved before
  // extracting in-component enter guards
  const enterGuards = extractEnterGuards(activated)
  const queue = enterGuards.concat(this.router.resolveHooks)
  runQueue(queue, iterator, () => {
    if (this.pending !== route) {
      return abort(createNavigationCancelledError(current, route))
    }
    this.pending = null
    onComplete(route)
    if (this.router.app) {
      this.router.app.$nextTick(() => {
        handleRouteEntered(route)
      })
    }
  })
})

const iterator = (hook: NavigationGuard, next) => {
  if (this.pending !== route) {
    return abort(createNavigationCancelledError(current, route))
  }
  try {
    hook(route, current, (to: any) => {
      if (to === false) {
        // next(false) -> abort navigation, ensure current URL
        this.ensureURL(true)
        abort(createNavigationAbortedError(current, route))
      } else if (isError(to)) {
        this.ensureURL(true)
        abort(to)
      } else if (
        typeof to === 'string' ||
        (typeof to === 'object' &&
          (typeof to.path === 'string' || typeof to.name === 'string'))
      ) {
        // next('/') or next({ path: '/' }) -> redirect
        abort(createNavigationRedirectedError(current, route))
        if (typeof to === 'object' && to.replace) {
          this.replace(to)
        } else {
          this.push(to)
        }
      } else {
        // confirm transition and pass on the value
        next(to)
      }
    })
  } catch (e) {
    abort(e)
  }
}
```
runQueue的实现
```js
export function runQueue (queue: Array<?NavigationGuard>, fn: Function, cb: Function) {
  const step = index => {
    if (index >= queue.length) {
      cb()
    } else {
      if (queue[index]) {
        fn(queue[index], () => {
          step(index + 1)
        })
      } else {
        step(index + 1)
      }
    }
  }
  step(0)
}
```
首先，我们执行runQueue，先调用`step(0)`，然后执行step函数，进入else逻辑，从队列里面取出函数，然后把函数当做fn的第一个参数传入，这个fn也就是我们之前传入的iterator。第二个参数也传入一个函数，这个函数是fn执行成功会回调的。

在`iterator`会调用hook函数，传入`route`、`current`以及一个函数，到这里就真正意义上执行了导航守卫的函数，那我们回顾下导航守卫中的参数(to, from, next)，这三个参数不就对应着`route`、`current`和函数。

那我们思考一个问题，我们之前写导航守卫的时候，为什么一定要调用下next函数才能继续往下走呢，其实答案就在`iterator`中，这个时候next函数就是hook中的第三个参数。第三个参数前面做了很多逻辑判断，我们主要看`next(to)`，最后调用了next，那这个next是什么呢，这个next是iterator函数的第二个参数，那这个参数又是我们在执行step函数中的fn函数的第二个参数，第二个参数是`() => { step(index + 1) }`，**那也就是说我们外部调用next()，其实内部就是在调用`step(index + 1)`，也就是开始执行下一个队列中的函数**，这个时候我们可能会恍然大悟，简直太巧妙了，用这么多回调函数来让外部控制内部去执行，真的是巧妙

那当我们队列中的都执行完了，这个时候会调用runQueue成功回调，也就是第三个参数。
第三个参数内部做的事其实和我们之前一样，也是构建queue，然后继续runQueue
然后刚构建的queue执行完了，就执行第三个参数的回调，然后内部调用`onComplete`

那`onComplete`是什么呢，其实就是我们在`transitionTo`中调用`confirmTransition`传入的第二个参数，现在发现了没有，我们又走到了我们第一篇中讲的那一部分了，只不过当时把我们的导航守卫逻辑跳过了。


### 总结
导航守卫的执行是通过构造queue队列来进行执行的，通过updated, deactivated, activated 这三个数组来获取我们对应导航守卫钩子，然后都推入到queue中。再通过runQueue进行执行，特别是runQueue中的实现，只能用一个字来形容：“巧妙”。