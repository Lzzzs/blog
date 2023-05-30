---
title: Vue-Router原理剖析-工作流程
date: 2023-05-29 16:19:08
tags:
---
### **前言**

本文的目的是从源码角度下去探究`Vue-Router`实际工作的流程

**Vue-Router版本：** `3.6.5`
**GitHub** 仓库地址：https://github.com/vuejs/vue-router

<!--more-->

### Vue 与 Vue-Router
首先我们研究`Vue-Router`的流程，要先明白 `Vue` 与 `Vue-Router`是如何结合起来的

> `Vue-Router`以插件的形式注入到`Vue`中

Vue 中的插件注册都会调用`Vue.use`，我们先看下`Vue.use`的内部实现
```js
  Vue.use = function (plugin: Function | Object) {
    // 判断插件是否重复安装
    const installedPlugins =
      this._installedPlugins || (this._installedPlugins = []);
    if (installedPlugins.indexOf(plugin) > -1) {
      return this;
    }

    // 将 Vue 构造函数放到第一个参数位置，然后将这些参数传递给 install 方法
    const args = toArray(arguments, 1);
    args.unshift(this);

    if (typeof plugin.install === "function") {
      // plugin 是一个对象，则执行其 install 方法安装插件
      plugin.install.apply(plugin, args);
    } else if (typeof plugin === "function") {
      // 执行直接 plugin 方法安装插件
      plugin.apply(null, args);
    }

    // 在 插件列表中 添加新安装的插件
    installedPlugins.push(plugin);
    return this;
  };
```
从上面的代码我们可以很清晰的看到，我们执行 `Vue.use` 无非是执行插件内部提供的 **install** 方法


### install
> src/install.js
```js
export function install (Vue) {
  if (install.installed && _Vue === Vue) return
  install.installed = true
  _Vue = Vue

  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (
      isDef(i) &&
      isDef((i = i.data)) &&
      isDef((i = i.registerRouteInstance))
    ) {
      i(vm, callVal)
    }
  }

  // 向根Vue混入beforeCreate和destroyed
  Vue.mixin({
    beforeCreate () {
      // 判断new Vue的时候有没有传router这个参数
      if (isDef(this.$options.router)) {
        this._routerRoot = this
        this._router = this.$options.router
        this._router.init(this)

        // 将this._route 设置为响应式数据，这也是为什么改变路径页面可以重新渲染的原因
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  // 访问Vue.prototype.$router 其实是访问 this._routerRoot._router
  Object.defineProperty(Vue.prototype, '$router', {
    get () {
      return this._routerRoot._router
    }
  })

  // 访问Vue.prototype.$route 其实是访问 this._routerRoot._route
  Object.defineProperty(Vue.prototype, '$route', {
    get () {
      return this._routerRoot._route
    }
  })

  // 注册RouterView和RouterLink 两个全局组件
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  // 对于beforeRouteEnter beforeRouteLeave beforeRouteUpdate 这三个router hook都使用与created相同的合并策略
  const strats = Vue.config.optionMergeStrategies
  // use the same hook merging strategy for route hooks
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate =
    strats.created
}
```
我们重点放在 **Vue.mixin** 混入的那两个生命周期上。也就是说，我们之后创建的每一个组件都会有这两个生命周期函数被调用。

##### beforeCreate
函数内部首先会判断 this.$options.router 存不存在，这个this指向的是组件实例，那么当前指向的也就是 Vue的根实例。我们在new Vue 根实例的时候会传入router参数，所以这个定义是生效的
```js
new Vue({
  router
})
```
那么接下来会将Vue根实例赋值给根实例上的`_routerRoot`，然后将传入的router赋值给根实例上的`_router`。接着执行router上的`init`方法，然后将每个Vue实例上的`_route`设置成响应式数据

##### init
> src/router.js
```js
export default class VueRouter {
  static install: () => void
  static version: string
  static isNavigationFailure: Function
  static NavigationFailureType: any
  static START_LOCATION: Route

  app: any
  apps: Array<any>
  ready: boolean
  readyCbs: Array<Function>
  options: RouterOptions
  mode: string
  history: HashHistory | HTML5History | AbstractHistory
  matcher: Matcher
  fallback: boolean
  beforeHooks: Array<?NavigationGuard>
  resolveHooks: Array<?NavigationGuard>
  afterHooks: Array<?AfterNavigationHook>

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

  init (app: any /* Vue component instance */) {
    process.env.NODE_ENV !== 'production' &&
      assert(
        install.installed,
        `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
          `before creating root instance.`
      )

    this.apps.push(app)

    // set up app destroyed handler
    // https://github.com/vuejs/vue-router/issues/2639
    app.$once('hook:destroyed', () => {
      // clean out app from this.apps array once destroyed
      const index = this.apps.indexOf(app)
      if (index > -1) this.apps.splice(index, 1)
      // ensure we still have a main app or null if no apps
      // we do not release the router so it can be reused
      if (this.app === app) this.app = this.apps[0] || null

      if (!this.app) this.history.teardown()
    })

    // main app previously initialized
    // return as we don't need to set up new history listener
    if (this.app) {
      return
    }

    this.app = app

    const history = this.history

    if (history instanceof HTML5History || history instanceof HashHistory) {
      const handleInitialScroll = routeOrError => {
        const from = history.current
        const expectScroll = this.options.scrollBehavior
        const supportsScroll = supportsPushState && expectScroll

        if (supportsScroll && 'fullPath' in routeOrError) {
          handleScroll(this, routeOrError, from, false)
        }
      }
      // transitionTo成功或者失败的回调
      const setupListeners = routeOrError => {
        // 设置监听器
        history.setupListeners()
        handleInitialScroll(routeOrError)
      }
      // 切换路径
      history.transitionTo(
        history.getCurrentLocation(),
        setupListeners,
        setupListeners
      )
    }

    history.listen(route => {
      this.apps.forEach(app => {
        // 这里修改会触发set 导致页面更新
        app._route = route
      })
    })
  }

  ...
}
```

###### 构造函数
我们之前在beforeCreate中会调用init，也就是调用VueRouter中的init方法。那么其实在beforeCreate之前我们其实会先`new VueRouter`创建router然后传入Vue中的配置中，所以我们先看VueRouter构造函数初始化过程

构造函数内部初始化重点关注`this.matcher = createMatcher(options.routes || [], this)`,以及初始化mode的过程

`createMatcher`是`Vue-Router`中比较重要的一环。具体不进去看了，直接给出结论。createMatcher会返回`match`、`addRoute`、`getRoutes`、`addRoutes`四个函数，比如match就是路径匹配的函数，addRoute可以动态添加路由

在`createMatcher`内部有三个很重要的变量`pathList`, `pathMap`, `nameMap`，这三个变量存放着我们写的所有路径的记录，也就是说我们写的routes都记录在这三个变量中

之后还有对mode进行选择以及降级处理，然后根据对应的mode初始化对应的History类

###### init 方法
init方法内部我们重点看`history.transitionTo`
```js
history.transitionTo(
  history.getCurrentLocation(),
  setupListeners,
  setupListeners
)
```
`transitionTo`是Vue-Router中非常重要的一个函数，目的是做路径跳转的，当前我们初始化的话，我们路径的跳转肯定是往根路径跳，那我们看看它是如何实现路径跳转的

###### transitionTo
> src/history/base.js
```js
transitionTo (
  location: RawLocation,
  onComplete?: Function,
  onAbort?: Function
) {
  let route
  // catch redirect option https://github.com/vuejs/vue-router/issues/3201
  try {
    route = this.router.match(location, this.current)
  } catch (e) {
    this.errorCbs.forEach(cb => {
      cb(e)
    })
    // Exception should still be thrown
    throw e
  }
  const prev = this.current
  this.confirmTransition(
    route,
    () => {
      this.updateRoute(route)
      onComplete && onComplete(route)
      this.ensureURL()
      this.router.afterHooks.forEach(hook => {
        hook && hook(route, prev)
      })

      // fire ready cbs once
      if (!this.ready) {
        this.ready = true
        this.readyCbs.forEach(cb => {
          cb(route)
        })
      }
    },
    err => {
      if (onAbort) {
        onAbort(err)
      }
      if (err && !this.ready) {
        // Initial redirection should not mark the history as ready yet
        // because it's triggered by the redirection instead
        // https://github.com/vuejs/vue-router/issues/3225
        // https://github.com/vuejs/vue-router/issues/3331
        if (
          !isNavigationFailure(err, NavigationFailureType.redirected) ||
          prev !== START
        ) {
          this.ready = true
          this.readyErrorCbs.forEach(cb => {
            cb(err)
          })
        }
      }
    }
  )
}
```
首先执行match匹配location与current，然后返回我们需要跳转的route，之后再执行`confirmTransition`

```js
confirmTransition (route: Route, onComplete: Function, onAbort?: Function) 
```
confirmTransition的第一个参数是route，第二个是完成也就是成功的毁掉，第三个是失败的回调。在confirmTransition内部其实没有跳转的代码，内部主要是导航守卫的代码，我们在下一节重点分析其中逻辑。
那跳转逻辑在哪呢？其实是在第二个参数，也就是成功的回调函数中
```js
() => {
  this.updateRoute(route)
  onComplete && onComplete(route)
  this.ensureURL()
  this.router.afterHooks.forEach(hook => {
    hook && hook(route, prev)
  })

  // fire ready cbs once
  if (!this.ready) {
    this.ready = true
    this.readyCbs.forEach(cb => {
      cb(route)
    })
  }
}
```
我们重点看this.updateRoute这个函数，这个函数可以触发视图的更新
```js
updateRoute (route: Route) {
  this.current = route
  this.cb && this.cb(route)
}
```
这个函数看起来非常简单，重点就在于this.cb(route)。在我们进行路径切换的过程中，cb已经被赋值过了，那cb到底是什么？其实它就是一个回调函数，在我们VueRouter.init中会被注入到cb中

VueRouter.init 最后一部分调用了history.listen
```js
history.listen(route => {
  this.apps.forEach(app => {
    // 这里修改会触发set 导致页面更新
    app._route = route
  })
})
```
history.listen的实现更简单了，就是将listen传入的函数赋值给cb
```js
listen (cb: Function) {
  this.cb = cb
}
```
于是我们updateRoute调用的其实是app._route = route，这一段代码就可以实现页面更新。根本原因在于，Vue.mixin混入的beforeCreate其中有这么一行代码
```js
Vue.util.defineReactive(this, '_route', this._router.history.current)
```
会将_route变成响应式，于是我们只要对它赋值，就会调用到set，那么就会执行dep.notify，在之前dep已经收集了渲染Watcher，所以就开始Vue的更新过程

那我们思考一个问题更新的组件放哪了？也就是说我们对应的路径的组件放哪里？**当然是放在router-view上！**

#### router-view
我们路由的渲染和`router-view`分不开关系，如果没有`router-view`我们组件连渲染的位置都没有。所以我们组件中肯定有`router-view`这个组件的存在，那么接着上面的流程，假如App.vue中有`router-view`组件，那么我们调用了`updateRoute`，Vue开始更新视图，那我们的`router-view`作为组件是不是要重新渲染，在组件内部会对路径做一些匹配，匹配到当前路径应该展示什么组件，然后渲染该组件，那么我们页面就能看到东西了！

那我们来看看`router-view`具体是怎么匹配对应的组件的
> src/components/view.js
```js
export default {
  name: 'RouterView',
  functional: true,
  props: {
    name: {
      type: String,
      default: 'default'
    }
  },
  render (_, { props, children, parent, data }) {
    // used by devtools to display a router-view badge
    data.routerView = true

    // directly use parent context's createElement() function
    // so that components rendered by router-view can resolve named slots
    const h = parent.$createElement
    const name = props.name
    const route = parent.$route
    const cache = parent._routerViewCache || (parent._routerViewCache = {})

    // determine current view depth, also check to see if the tree
    // has been toggled inactive but kept-alive.
    let depth = 0
    let inactive = false
    while (parent && parent._routerRoot !== parent) {
      const vnodeData = parent.$vnode ? parent.$vnode.data : {}
      if (vnodeData.routerView) {
        depth++
      }
      if (vnodeData.keepAlive && parent._directInactive && parent._inactive) {
        inactive = true
      }
      parent = parent.$parent
    }
    data.routerViewDepth = depth

    // render previous view if the tree is inactive and kept-alive
    if (inactive) {
      const cachedData = cache[name]
      const cachedComponent = cachedData && cachedData.component
      if (cachedComponent) {
        // #2301
        // pass props
        if (cachedData.configProps) {
          fillPropsinData(cachedComponent, data, cachedData.route, cachedData.configProps)
        }
        return h(cachedComponent, data, children)
      } else {
        // render previous empty view
        return h()
      }
    }

    const matched = route.matched[depth]
    const component = matched && matched.components[name]

    // render empty node if no matched route or no config component
    if (!matched || !component) {
      cache[name] = null
      return h()
    }

    // cache component
    cache[name] = { component }

    // attach instance registration hook
    // this will be called in the instance's injected lifecycle hooks
    data.registerRouteInstance = (vm, val) => {
      // val could be undefined for unregistration
      const current = matched.instances[name]
      if (
        (val && current !== vm) ||
        (!val && current === vm)
      ) {
        matched.instances[name] = val
      }
    }

    // also register instance in prepatch hook
    // in case the same component instance is reused across different routes
    ;(data.hook || (data.hook = {})).prepatch = (_, vnode) => {
      matched.instances[name] = vnode.componentInstance
    }

    // register instance in init hook
    // in case kept-alive component be actived when routes changed
    data.hook.init = (vnode) => {
      if (vnode.data.keepAlive &&
        vnode.componentInstance &&
        vnode.componentInstance !== matched.instances[name]
      ) {
        matched.instances[name] = vnode.componentInstance
      }

      // if the route transition has already been confirmed then we weren't
      // able to call the cbs during confirmation as the component was not
      // registered yet, so we call it here.
      handleRouteEntered(route)
    }

    const configProps = matched.props && matched.props[name]
    // save route and configProps in cache
    if (configProps) {
      extend(cache[name], {
        route,
        configProps
      })
      fillPropsinData(component, data, route, configProps)
    }

    return h(component, data, children)
  }
}
```
`router-view`中的代码逻辑还是挺多的，我们重点看它是如何确认当前路径下该渲染什么组件的。

`router-view`本质是一个函数式组件，之后渲染是调用h函数做渲染，熟悉Vue源码的同学应该很明白这个函数对应的什么，就不展开讲了

我们重点看其中求 **depth** 的过程，这个过程是为了之后匹配组件做准备
```js
let depth = 0
let inactive = false
while (parent && parent._routerRoot !== parent) {
  const vnodeData = parent.$vnode ? parent.$vnode.data : {}
  if (vnodeData.routerView) {
    depth++
  }
  if (vnodeData.keepAlive && parent._directInactive && parent._inactive) {
    inactive = true
  }
  parent = parent.$parent
}
data.routerViewDepth = depth
```
在这个while中它会往上找，如果它的父节点存在routerView，那么depth加一。举个例子，如果是`/foo`，那么它的深度就为0，如果是`/foo/child`，那么它的深度就为1。那我们知道深度之后，它是如何去匹配组件的呢？

```js
const matched = route.matched[depth]
const component = matched && matched.components[name]
```
从第一行可以看到拿depth取route.matched中的元素，那route.matched是什么。route中的matched就对应的当前route的父子层级关系，matched的顺序是从大到小。也就是说我们只需要知道它的深度，我们就可以知道当前我们匹配的matched，那么拿到matched之后，其中有components，这也是我们之前配置routes中会配置的。那么至此，我们已经匹配到了我们想要的组件了

之后调用h函数把component传入，就能正常渲染了

### 总结
Vue-Router作为插件传入到Vue实例中，挂载了两个生命周期函数。在我们每次进行路径切换的时候都会调用到`transitionTo`这个函数来帮助我们做路由切换，其中会涉及到很多路由钩子的执行，以及会调用`updateRoute`，会触发history上的cb函数，这个函数对_route进行赋值，触发set方法，然后触发Vue的异步更新视图过程。组件中的router-view组件被重新渲染，会根据depth正确的选择该路径上的组件，于是页面上就会更新出我们想要的内容。