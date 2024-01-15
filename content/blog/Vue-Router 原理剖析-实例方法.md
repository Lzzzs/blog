---
external: false
draft: false
title: Vue-Router 原理剖析-实例方法
date: 2023-06-16
---

### 前言

本文我们来研究下 `Vue-Router` 中的实例方法

这也是我们对 `Vue-Router` 的最后一篇文章

### 路径切换实例方法

在初始化路由时，我们可以选择mode。其实Vue-Router有三种mode。分别是hash、history、abstract。其中hash和history是我们常用的两种。而abstract是在非浏览器环境下使用的。比如说在node环境下使用Vue-Router。这里我们只讨论hash和history两种mode。

我们都知道Vue-Router中路径切换有push、replace、go这三种方法。那关于这三种实例方法是如何实现的呢？

#### VueRouter.prototype.push / VueRouter.prototype.replace 

> src/router.js
```js
class VueRouter {
  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    // $flow-disable-line
    if (!onComplete && !onAbort && typeof Promise !== 'undefined') {
      return new Promise((resolve, reject) => {
        this.history.push(location, resolve, reject)
      })
    } else {
      this.history.push(location, onComplete, onAbort)
    }
  }
}
```
push的代码逻辑很简单，就是判断下是否支持Promise而且没有传onComplete和onAbort的情况下，返回一个Promise，内部调用的是VueRouter.history.push方法，将promise的resolve作为第二个参数传入。否则直接执行VueRouter.history.push方法。

history其实在初始化VueRouter时就已经创建了。
```js
class VueRouter {
  constructor (options: RouterOptions = {}) {
    if (process.env.NODE_ENV !== 'production') {
      warn(
        this instanceof VueRouter,
        `Router must be called with the new operator.`
      )
    }
    this.app = null
    this.apps = []
    this.options = options
    this.beforeHooks = []
    this.resolveHooks = []
    this.afterHooks = []
    this.matcher = createMatcher(options.routes || [], this)

    // 不指定mode 默认是hash模式
    let mode = options.mode || 'hash'
    this.fallback =
      mode === 'history' && !supportsPushState && options.fallback !== false
    if (this.fallback) {
      mode = 'hash'
    }
    if (!inBrowser) {
      mode = 'abstract'
    }
    this.mode = mode

    switch (mode) {
      case 'history':
        this.history = new HTML5History(this, options.base)
        break
      case 'hash':
        this.history = new HashHistory(this, options.base, this.fallback)
        break
      case 'abstract':
        this.history = new AbstractHistory(this, options.base)
        break
      default:
        if (process.env.NODE_ENV !== 'production') {
          assert(false, `invalid mode: ${mode}`)
        }
    }
  }
}
```
通过传入不同的配置new 不同的history实例。这里我们以hash模式为例。
```js
export class HashHistory extends History {
  constructor (router: Router, base: ?string, fallback: boolean) {
    super(router, base)
    // check history fallback deeplinking
    if (fallback && checkFallback(this.base)) {
      return
    }
    ensureSlash()
  }

  push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
    const { current: fromRoute } = this
    this.transitionTo(
      location,
      route => {
        pushHash(route.fullPath)
        handleScroll(this.router, route, fromRoute, false)
        onComplete && onComplete(route)
      },
      onAbort
    )
  }
}
```
从HashHistory中我们可以发现是继承History的，当然HTML5History、AbstractHistory都是继承History。

HashHistory中的push方法先是通过transitionTo调用，我们重点关注第二个参数，这个是所有前置事情完成之后成功的回调。其中可以发现调用了pushHash

```js
function pushHash (path) {
  if (supportsPushState) {
    pushState(getUrl(path))
  } else {
    window.location.hash = path
  }
}

export function pushState (url?: string, replace?: boolean) {
  saveScrollPosition()
  // try...catch the pushState call to get around Safari
  // DOM Exception 18 where it limits to 100 pushState calls
  const history = window.history
  try {
    if (replace) {
      // preserve existing history state as it could be overriden by the user
      const stateCopy = extend({}, history.state)
      stateCopy.key = getStateKey()
      history.replaceState(stateCopy, '', url)
    } else {
      history.pushState({ key: setStateKey(genStateKey()) }, '', url)
    }
  } catch (e) {
    window.location[replace ? 'replace' : 'assign'](url)
  }
}
```
pushHash内部通过supportsPushState，判断当前环境是否支持浏览器 History API。如果支持调用pushState方法，否则直接通过window.location.hash = path的方式改变hash值。

pushState函数内部是通过window.history.pushState。当然如果此时第二个参数replace是true，就是replaceState

所以，此时我们应该能明白VueRouter上的实例方法push和replace就是通过History API中的pushState和replaceState实现的。


#### VueRouter.prototype.go
关于go方法直接给出答案，因为和我们上面分析的代码都是一样。

go方法也是通过History API中的go方法实现的。

### 用户回退的时候怎么处理的？
在Vue-Router中，不管是hash还是history这两种mode都会在路由初始化阶段对浏览器回退时间做监听。

对于HashHistory中有setupListeners实例方法，在初始化路由切换成功之后会被调用
```js
setupListeners () {
  if (this.listeners.length > 0) {
    return
  }

  const router = this.router
  const expectScroll = router.options.scrollBehavior
  const supportsScroll = supportsPushState && expectScroll

  if (supportsScroll) {
    this.listeners.push(setupScroll())
  }

  // 监听回退的函数
  const handleRoutingEvent = () => {
    const current = this.current
    if (!ensureSlash()) {
      return
    }
    this.transitionTo(getHash(), route => {
      if (supportsScroll) {
        handleScroll(this.router, route, current, true)
      }
      if (!supportsPushState) {
        replaceHash(route.fullPath)
      }
    })
  }

  // 设置监听器，这里是监听回退
  const eventType = supportsPushState ? 'popstate' : 'hashchange'
  window.addEventListener(eventType, handleRoutingEvent)
  this.listeners.push(() => {
    window.removeEventListener(eventType, handleRoutingEvent)
  })
}
```
重点关注最后的eventType，这里通过supportsPushState判断当前环境是否支持浏览器History 
然后再监听这个事件，也就是说当用户按下回退按钮之后，事件被触发，handleRoutingEvent函数触发，又开始执行一次跳转，页面也因此更新。

> 对于HTML5History直接就是监听popstate事件

### 总结
关于路径切换的实例方法，本质都是通过History API中的pushState、replaceState、go方法实现的。如果浏览器不支持的情况下，就直接window.location上的方法去修改路径。

对于用户回退，Vue-Router在支持History API的情况下，会监听popstate事件，否则监听hashchange事件，然后内部相当于重新做一次路由跳转更新页面。