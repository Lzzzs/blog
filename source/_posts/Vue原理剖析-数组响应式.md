---
title: Vue原理剖析-数组响应式
date: 2023-05-24 10:09:47
tags:
---

上一篇我们分析了对象的响应式处理，这一篇我们分析数组响应式。

同理，在开始之前，我们先思考一个问题
> **为什么我们在data中定义的数组一变页面就会自动刷新？**

接下来我们带着这个问题去看源码是如何实现的

<!--more-->

#### 初始化
看过上一篇的同学应该能知道初始化`data`数据是调用了`initData`，然后其中调用了`observe`这个函数去对`data`中的数组做响应式处理，那我们重点去看`observe`之后对data的处理。

> observe
```js
export function observe(value: any, asRootData: ?boolean): Observer | void {
  // 非对象和 VNode 实例不做响应式处理
  if (!isObject(value) || value instanceof VNode) {
    return;
  }
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
  if (asRootData && ob) {
    ob.vmCount++;
  }
  return ob;
}
```

上面代码逻辑比较多，我们重点看`ob = new Observer(value)`。这里将我们定义的data数据传入了构造函数的参数中，那我们继续看构造函数的实现。

#### Observer 
```js
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that have this object as root $data

  constructor(value: any) {
    this.value = value;
    /**
     * 这里定义dep是为了收集数组的依赖，因为数组收集依赖是在defineReactive函数的getter中收集的
     * 但是触发依赖是在对函数拦截的部分进行拦截的，dep放这的原因是因为这样defineReactive和函数拦截部分都能拿到dep
     */
    this.dep = new Dep();
    this.vmCount = 0;

    // 在 value 对象上设置 __ob__ 属性
    def(value, "__ob__", this);

    if (Array.isArray(value)) {
      // 处理数组响应式
      if (hasProto) {
        // 有 __proto__
        protoAugment(value, arrayMethods);
      } else {
        copyAugment(value, arrayMethods, arrayKeys);
      }
      this.observeArray(value);
    } else {
      // 处理对象响应式
      // value 为对象，为对象的每个属性（包括嵌套对象）设置响应式
      this.walk(value);
    }
  }

  /**
   * Walk through all properties and convert them into
   * getter/setters. This method should only be called when
   * value type is Object.
   */
  walk(obj: Object) {
    const keys = Object.keys(obj);
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i]);
    }
  }

  /**
   * Observe a list of Array items.
   */
  observeArray(items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i]);
    }
  }
}


function protoAugment(target, src: Object) {
  target.__proto__ = src;
}

function copyAugment(target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i];
    def(target, key, src[key]);
  }
}

```
这里我是贴出了`Observer`类的完整代码，在上一篇我删减了这里有关数组的代码，是为了更直观的看到对象的响应式，那么我们从这里继续看数组的响应式，在`constructor`时会判断传入的值是不是数组。

是数组的话，会判断当前环境下有没有`__proto__`。因为在有些环境下是没有这个属性的。如果有执行`protoAugment`，其实内部就是将`value`的`__proto__`指向`arrayMethods`了。没有的话就执行`copyAugment`，内部比较粗暴的直接将`arrayMethods`的方法赋值到数组上，而不是通过原型链的方法。

我们重点看有`__proto__`这种情况。那有的同学可能好奇`arrayMethods`具体是一个什么东西，我们继续往下走。

#### arrayMethods
```js
import { def } from "../util/index";

const arrayProto = Array.prototype;
export const arrayMethods = Object.create(arrayProto);

const methodsToPatch = [
  "push",
  "pop",
  "shift",
  "unshift",
  "splice",
  "sort",
  "reverse",
];

/**
 * Intercept mutating methods and emit events
 */
methodsToPatch.forEach(function (method) {
  // cache original method
  const original = arrayProto[method];
  def(arrayMethods, method, function mutator(...args) {
    const result = original.apply(this, args);
    const ob = this.__ob__;

    let inserted;
    // 如果 method 是以下三个之一，说明是新插入了元素
    switch (method) {
      case "push":
      case "unshift":
        inserted = args;
        break;
      case "splice":
        inserted = args.slice(2);
        break;
    }

    // 对新插入的元素做响应式处理
    if (inserted) ob.observeArray(inserted);

    // notify change
    ob.dep.notify();
    return result;
  });
});
```
这部分代码就是有关`arrayMethods`的实现，我们可以看到`arrayMethods`刚开始就是一个空对象，然后`__proto__`指向`arrayProto`，也就是指向`Array.prototype`。看到这里大家可能觉得为什么要这样，不是多此一举吗？

再继续看下去有一个循环，循环内部遍历了`methodsToPatch`数组中的七个方法。刚开始缓存了原方法，然后开始定义`arrayMethods`这个空对象，将这七个方法在`arrayMethods`中定义一遍。

看到这里大家应该明白了，其实这里就是做了一个方法的拦截，我们每个数组如果要进行响应式处理的话，都会将本身的原型替换掉，在原来的基础上加入自己拦截方法，然后再继续在拦截的方法里面调用原方法。这样是为什么干什么呢？这样我们就可以在我们自己的拦截方法里面做一些事情了，比如触发依赖更新。因为`Object.defineProperty`无法监听到数组的数据改变，所以我们只能这样去操作。

拦截方法内部做的一些操作就是，有新增元素的情况，那么新增元素也需要做响应式处理。之后调用了`ob.dep.notify()`，去做依赖更新，那为什么要更新？在回答这个问题之前，大家先考虑另外一个问题。为什么只处理数组的这七种方法？细心的同学应该很快就能得出答案。因为这七种方法都会改变数组的数据，那么我们响应式的初衷不就是为了数据改变然后更新视图吗，那我们缓存不会更改数据的方法干啥。

所以回到刚才那个问题，那为什么要更新？在我们调用这七个方法的时候，数据是会变动的，那我们自然而然就肯定需要通知视图去更新。

#### observeArray
在我们处理完数组的方法之后，我们还会调用observeArray这个方法
```js
  observeArray(items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i]);
    }
  }
```
这个方法本质是拿出数组中的每一个元素，然后调用`observe`。又回到`observe`这个方法了。然后又是一样的步骤，就不细说了。

那到这里，我们回想下我们刚开始的那个问题
> 为什么我们在data中定义的数组一变页面就会自动刷新?

现在我们已经可以回答了，如果我们通过push，pop的方法改变数组的话，其实会走到Vue内部定义的拦截器中，其中会通知依赖更新，视图也就自然就会更新。但是如果我们不是通过这些方法去更新数组的呢？例如`this.arr[0] = 2`,那这种情况Vue其实不知道你更新的数据，所以这种情况就不会更新。在日常开发尽量避免这种修改数据的方法，实在需要修改的话可以使用那七个api，如果满足不了的话，就实现Vue提供的`$set`，也可以及时更新。