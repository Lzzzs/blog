---
title: Vue原理剖析-对象响应式
date: 2023-05-23 15:15:23
tags:
---

### 前言
版本：`Vue 2.6.12`

之前看 `Vue` 源码一直只是看，没有总结文章，所以这一次把之前看过的都总结一遍。

我们先思考一个问题？
> **为什么我们定义在data中的数据，会随着我们的改变和导致页面的更新？**

接下来，伴随着这个问题，我会从源码一步一步去分析，`Vue` 内部是怎么做到的
下面会给出源码路径以及核心代码

### 首先，我们先来看一下 `Vue` 的初始化过程

> src/core/instance/index.js
```js
// Vue构造函数
function Vue(options) {
  // 开发环境的提示
  if (process.env.NODE_ENV !== "production" && !(this instanceof Vue)) {
    warn("Vue is a constructor and should be called with the `new` keyword");
  }

  // 调用 Vue.prototype._init 方法，该方法是在 initMixin 中定义的
  this._init(options);
}
```
在上面路径的文件下可以看到`Vue`本质是一个构造函数，在其中调用了`this._init`方法，该方法是在 `initMixin` 中定义的

> src/core/instance/init.js
```js
export function initMixin(Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {

    // 数据响应式的重点，处理 props、methods、data、computed、watch
    initState(vm);

  };
}
```
可以看到`_init`的方法里面调用了`initState`方法，那我们找下`initState`方法在哪里定义的

> src/core/instance/state.js
```js
export function initState(vm: Component) {
  const opts = vm.$options;

  if (opts.data) {
    initData(vm);
  } else {
    observe((vm._data = {}), true /* asRootData */);
  }

}
```
在`initState`方法中首先从`$options`上拿到了`data`，这个`data`就是我们定义的data对象，如果我们没有定义`data`，其实内部会给它一个默认的空对象，如果我们定义了`data`，就调用`initData`，那我们看下`initData`的逻辑

```js
function initData(vm: Component) {
  let data = vm.$options.data;

  // observe data
  observe(data, true /* asRootData */);
}
```
从上面的代码，能很明显的看出，就能拿到`data`调用了`observe`这个方法，这个方法也就是让data拥有响应式的方法。那可能有同学会想了这代码逻辑这么简单吗，其实不是的，我只是截取了核心代码，因为我们本质是要看data是如何有响应式的，源码内部还有很多边界情况的判断，有兴趣的同学可以去看看。

**看源码最忌讳的就是忘记我们本来看源码的目的，也就是忘了我们的主线。源码内部有非常多的边界情况需要判断，如果看看这看看那，我们脑袋就只会越来越乱，所以我们要时刻牢记我们这次看源码的目的。**

> src/core/observer/index.js
```js
export function observe(value: any, asRootData: ?boolean): Observer | void {
  let ob: Observer | void;

  if (hasOwn(value, "__ob__") && value.__ob__ instanceof Observer) {
    // 如果 value 对象上存在 __ob__ 属性，则表示已经做过观察了，直接返回 __ob__ 属性
    ob = value.__ob__;
  } else if (
    shouldObserve &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    // 创建观察者实例
    ob = new Observer(value);
  }
  return ob;
}

```
这一部分就开始和响应式有关了，我们调用observe函数传入的data其实就是这里的形参value。开始之前会做判断，如果value上存在 \_\_ob\_\_ 属性，就返回，如果没有就将value传入Observer，返回新的ob。

### \_\_ob\_\_ 是什么？
只要数据是响应式的话，都会被设置这个属性，有了这个属性我们就可以之前该数据已经是响应式了，当然不单单只有判断是否是响应式这个作用，我们还可以从中拿到dep，这是后话了。

### Observer
> src/core/observer/index.js
```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number;

  constructor(value: any) {
    this.value = value;

    // 在 value 对象上设置 __ob__ 属性
    def(value, "__ob__", this);

    this.walk(value);
  }

  walk(obj: Object) {
    const keys = Object.keys(obj);
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i]);
    }
  }
}
```
从上面的代码能看到在构造函数中调用了`walk`函数，然后内部循环我们的`data`，再调用`defineReactive`

### defineReactive
> src/core/observer/index.js
```js
export function defineReactive(
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  // 实例化dep类，一个dep对应一个对象的key
  const dep = new Dep();

  // 递归调用，处理 val 即 obj[key] 的值为对象的情况，保证对象中的所有 key 都被观察
  let childOb = !shallow && observe(val);

  // 响应式核心
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    // get 拦截对 obj[key] 的读取操作
    get: function reactiveGetter() {
      const value = getter ? getter.call(obj) : val;
        // 依赖收集，在 dep 中添加 watcher，也在 watcher 中添加 dep
        dep.depend();
        // childOb 表示对象中嵌套对象的观察者对象，如果存在也对其进行依赖收集
        if (childOb) {
          childOb.dep.depend();
        }
      }
      return value;
    },
  });
}
```
终于，我们走到了响应式的真正核心代码。
能看到Vue2内部是调用`Object.defineProperty`来拦截get操作
我们之前循环调用这个函数，内部会判断obj[key]是不是对象，如果是那么递归。如果不是，会拦截obj上的key，定义get方法。在我们获取这个值的时候，get方法会被回调，此时dep.depend会收集依赖。

#### 那什么时候会开始收集依赖呢？
举个例子
如果我们在template中用了data中的某个key，那么在模版编译的时候会访问到这个值，就会触发依赖收集的过程。

### Dep
> src/core/observer/dep.js
```js
export default class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor() {
    this.id = uid++;
    this.subs = [];
  }

  // 向 watcher 中添加 dep
  depend() {
    if (Dep.target) {
      Dep.target.addDep(this);
    }
  }
}
```
刚才在`defineReactive`中调用了`dep.depend`可以看到内部执行了`Dep.target.addDep(this)`。这段代码的含义是往`Watcher`中添加`dep`。那肯定有同学疑惑，`Dep.target`为什么就是`Watcher`了？其实在初始化Watcher时`Dep.target` 就被设置成`Watcher`了，然后调用`Watcher`内部的`addDep`方法，向 `Watcher` 中添加 `dep`

我们简单看下`Watcher`中的`addDep`方法
```js
  addDep(dep: Dep) {
    // 判重，如果 dep 已经存在则不重复添加
    const id = dep.id;
    if (!this.newDepIds.has(id)) {
      // 缓存 dep.id，用于判重
      this.newDepIds.add(id);
      // 添加 dep
      this.newDeps.push(dep);

      // 避免在 dep 中重复添加 watcher，this.depIds 的设置在 cleanupDeps 方法中
      if (!this.depIds.has(id)) {
        // 添加 watcher 自己到 dep
        dep.addSub(this);
      }
    }
  }
```
从其中能发现，`Watcher`收集`Dep`，但`Dep`又收集`Watcher`，我们也就是可以得出一个结论，他们是双向收集的。

那有同学可能好奇，为什么要收集`Watcher`?
其实`Watcher`分很多种，渲染`Watcher`，以及用户自己写的`Watcher`。具体代码就不带大家看了，我直接说结论吧。
那我们收集到了对应要更新的`Watcher`，假设我们数据此时变化了，会触发set，set内部会执行dep.notify()，去通知dep中的所有`Watcher`更新，那如果我们收集了渲染`Watcher`，那页面就会更新，这就是为什么我们修改了数据，对应数据的地方就会更新。

那渲染`Watcher`是什么呢，简单看下里面的函数，之后在编译和渲染的时候会重点讲

```js
updateComponent = () => {
  vm._update(vm._render(), hydrating);
};
```
如上，看起来很简单的函数，但是它内部做的事情可多了，生成虚拟DOM，然后做diff，做完diff就会开始渲染页面。
所以说，我们就是收集这个函数，可以让我们页面进行更新。


### 总结
在初始化的时候`data`中的数据就会在`defineReactive`函数中定义`set`和`get`拦截器，之后在某一个节点，比如模版编译的时候触发到`get`，这个时候会开始收集依赖，也就是收集`Watcher`。之后数据发生变化了，触发`set`，`set`中会通知每一个`Watcher`进行更新，如果之前收集过渲染`Watcher`，那么页面就会更新。这也就是为什么，我们在`data`中的数据变化了，页面就会立马更新的原因。