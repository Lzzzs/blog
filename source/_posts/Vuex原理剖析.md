---
title: Vuex原理剖析
date: 2023-06-25 16:46:06
tags:
---

### 前言
版本：`Vuex 4.1.0`

本文我们来研究Vuex内部原理

**带着如下问题去看源码：**
- store对象怎么被注入到Vue实例中的？
- 模块化怎么实现的？为什么在模块中调用mutations中取到的state是模块内部的？
- 为什么我们在组件中使用mapState等辅助函数，就可以直接使用store中的数据？
- mapState等辅助函数是怎么实现的？
- Vuex内部是怎么区分用户是通过mutations修改state，还是手动修改state？

<!--more-->
### store对象怎么被注入到Vue实例中的？
首先，回顾下在Vue中使用Vuex的方式
```js
import { createApp } from 'vue'
import { createStore } from 'vuex'

const moduleA = {
  state () {
    return {
      count: 0
    }
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
}

const moduleB = {
  state () {
    return {
      count: 0
    }
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
}

// 创建一个新的 store 实例
const store = createStore({
  modules: {
    a: moduleA,
    b: moduleB
  },
  state () {
    return {
      count: 0
    }
  },
  mutations: {
    increment (state) {
      state.count++
    }
  }
})

const app = createApp({ /* 根组件 */ })

// 将 store 实例作为插件安装
app.use(store)
```

我们可以看到，刚开始通过createStore传入配置返回store，之后再通过app.use(store)将store作为插件安装到Vue实例中

我们先看createStore的实现。其实就是返回一个Store实例
> src/store.js
```js
export function createStore(options) {
  return new Store(options);
}
```

那么我们通过use调用将store传入其中，其实本质就是调用Store中的install方法
> src/store.js
```js
export class Store {
  install(app, injectKey) {
    app.provide(injectKey || storeKey, this);
    app.config.globalProperties.$store = this;

    const useDevtools =
      this._devtools !== undefined
        ? this._devtools
        : __DEV__ || __VUE_PROD_DEVTOOLS__;

    if (useDevtools) {
      addDevtools(app, this);
    }
  }
}
```
在install内部我们重点关注`app.config.globalProperties.$store`，这里就是将我们创建的Store实例绑定到了Vue实例的`$store`上

### 模块化怎么实现的？为什么在模块中调用mutations中取到的state是模块内部的？
我们回到调用createStore的地方，刚开始new了Store实例，那么就会调用Store的构造方法
```js
class Store {
  constructor(options = {}) {
    if (__DEV__) {
      assert(
        typeof Promise !== 'undefined',
        `vuex requires a Promise polyfill in this browser.`
      );
      assert(
        this instanceof Store,
        `store must be called with the new operator.`
      );
    }

    const { plugins = [], strict = false, devtools } = options;

    // store internal state
    this._committing = false;
    this._actions = Object.create(null);
    this._actionSubscribers = [];
    this._mutations = Object.create(null);
    this._wrappedGetters = Object.create(null);
    // 注册module
    this._modules = new ModuleCollection(options);

    this._modulesNamespaceMap = Object.create(null);
    this._subscribers = [];
    this._makeLocalGettersCache = Object.create(null);

    // EffectScope instance. when registering new getters, we wrap them inside
    // EffectScope so that getters (computed) would not be destroyed on
    // component unmount.
    this._scope = null;

    this._devtools = devtools;

    // bind commit and dispatch to self
    // 重写commit、dispatch，绑定this
    const store = this;
    const { dispatch, commit } = this;
    this.dispatch = function boundDispatch(type, payload) {
      return dispatch.call(store, type, payload);
    };
    this.commit = function boundCommit(type, payload, options) {
      return commit.call(store, type, payload, options);
    };

    // strict mode
    this.strict = strict;

    // 拿root state
    const state = this._modules.root.state;

    // 初始化root module
    // init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    installModule(this, state, [], this._modules.root);

    // initialize the store state, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    resetStoreState(this, state);

    // apply plugins
    plugins.forEach((plugin) => plugin(this));
  }
}
```
重点看`this._modules = new ModuleCollection(options)`

> src/module/module-collection.js
```js
export default class ModuleCollection {
  constructor (rawRootModule) {
    // register root module (Vuex.Store options)
    this.register([], rawRootModule, false)
  }

  register (path, rawModule, runtime = true) {
    if (__DEV__) {
      assertRawModule(path, rawModule)
    }

    const newModule = new Module(rawModule, runtime)
    if (path.length === 0) {
      this.root = newModule
    } else {
      const parent = this.get(path.slice(0, -1))
      parent.addChild(path[path.length - 1], newModule)
    }

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }
}
```
ModuleCollection初始化会调用register，然后在register中path.length = 0。因为我们传入的是根module，所以会将根module赋值给this.root。如果root module有modules，那么就会递归调用register，将子module也注册到this.root中。
模块的区分主要是根据path数组来的，如果path.length = 0，那么就是根module，否则就是子module。

store._modules如下，是根据模块的关系构建的树状结构
![Vuex_modules](images/Vuex_modules.png)

到这，我们有模块结构有了初步的认识，也明白了模块是如何构建起来的。

回到Store初始化部分，其中调用了installModule方法，这个方法主要是将模块的state、getters、mutations、actions等注册到store中
> src/store-util.js
```js
export function installModule(store, rootState, path, module, hot) {
  const isRoot = !path.length;
  const namespace = store._modules.getNamespace(path);

  // register in namespace map
  if (module.namespaced) {
    if (store._modulesNamespaceMap[namespace] && __DEV__) {
      console.error(
        `[vuex] duplicate namespace ${namespace} for the namespaced module ${path.join(
          '/'
        )}`
      );
    }
    store._modulesNamespaceMap[namespace] = module;
  }

  // set state
  if (!isRoot && !hot) {
    const parentState = getNestedState(rootState, path.slice(0, -1));
    const moduleName = path[path.length - 1];
    store._withCommit(() => {
      if (__DEV__) {
        if (moduleName in parentState) {
          console.warn(
            `[vuex] state field "${moduleName}" was overridden by a module with the same name at "${path.join(
              '.'
            )}"`
          );
        }
      }
      parentState[moduleName] = module.state;
    });
  }

  const local = (module.context = makeLocalContext(store, namespace, path));

  // 注册mutation
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key;
    registerMutation(store, namespacedType, mutation, local);
  });

  // 注册action
  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key;
    const handler = action.handler || action;
    registerAction(store, type, handler, local);
  });

  // 注册getter
  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key;
    registerGetter(store, namespacedType, getter, local);
  });

  // 递归初始化子module
  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot);
  });
}
```

