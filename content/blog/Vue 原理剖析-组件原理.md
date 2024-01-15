---
external: false
draft: false
title: Vue 原理剖析-组件原理
date: 2023-05-26
---

### 前言

Vue的核心思想就是组件化思想，将复杂的模块拆成一个个组件再拼装起来。

> **那么组件化的原理是什么呢，我们写的组件又是怎么搭建起来的呢？**

带着这个问题，我们去研究下源码如何实现的

首先，我们观察下面代码，有一个`Hello-World`组件，那我们这次主要目的就是探索这个组件如何渲染出来的
```html
<body>
  <div id="app">
    <Hello-World></Hello-World>
    <div>123</div>
  </div>

  <script>
    new Vue({
      components: {
        "hello-world": {
          template: "<div>hello world</div>",
        },
      },
      el: "#app",
    });
  </script>
</body>
```
首先给出代码渲染到浏览器的格式，可以发现就是将组件内部的元素替换了下
```html
  <div id="app">
    <div>hello world</div>
    <div>123</div>
  </div>
```

### 初始化
在我们`new Vue`的时候会执行内部`_init`方法，会进行各种初始化的工作。

之后会进行挂载
```js
if (vm.$options.el) {
  vm.$mount(vm.$options.el);
}
```

### 挂载
因为我们采用的是`CDN`引用的`Vue.js`，所以这个时候我们要的版本就是全量的版本，什么意思呢。就是内部包含了编译时和运行时的代码，如果是我们脚手架搭的代码就不需要编译时的代码，因为`vue-loader`会帮我们编译。

因为我们采用的是全量的版本，所以我们`$mount`的定义也不一样
> src/platforms/web/entry-runtime-with-compiler.js
```js
import Vue from "./runtime/index";
const mount = Vue.prototype.$mount;

Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // 得到挂载点
  el = el && query(el);
  /**
   * 如果用户提供了 render 配置项，则直接跳过编译阶段，否则进入编译阶段
   *   解析 template 和 el，并转换为 render 函数
   *   优先级：render > template > el
   */
  const options = this.$options;
  if (!options.render) {
    let template = options.template;
    if (template) {
      // 处理 template 选项
      if (typeof template === "string") {
        if (template.charAt(0) === "#") {
          // { template: '#app' }，template 是一个 id 选择器，则获取该元素的 innerHtml 作为模版
          template = idToTemplate(template);
        }
      } else if (template.nodeType) {
        // template 是一个正常的元素，获取其 innerHtml 作为模版
        template = template.innerHTML;
      }
    } else if (el) {
      // 设置了 el 选项，获取 el 选择器的 outerHtml 作为模版
      template = getOuterHTML(el);
    }

    if (template) {
      // 编译模版，得到 动态渲染函数和静态渲染函数
      const { render, staticRenderFns } = compileToFunctions(
        template,
        {
          // 在非生产环境下，编译时记录标签属性在模版字符串中开始和结束的位置索引
          outputSourceRange: process.env.NODE_ENV !== "production",
          shouldDecodeNewlines,
          shouldDecodeNewlinesForHref,
          // 界定符，默认 {{}}
          delimiters: options.delimiters,
          // 是否保留注释
          comments: options.comments,
        },
        this
      );

      // 将两个渲染函数放到 this.$options 上
      options.render = render;
      options.staticRenderFns = staticRenderFns;
    }
  }

  // 执行挂载
  return mount.call(this, el, hydrating);
};
```
从代码逻辑可以看出，先缓存了runtime下的mount，然后定义了另外一个mount，内部做了编译相关的处理，将`render`函数挂到$options上，然后执行缓存的mount

这里的代码主要是拿到`render`函数，然后挂载


> src/platforms/web/runtime/index.js
```js
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```
内部调用了`mountComponent`去做挂载

#### mountComponent
> src/core/instance/lifecycle.js
```js
export function mountComponent(
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el;

  callHook(vm, "beforeMount");

  let updateComponent;
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== "production" && config.performance && mark) {
    ...
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating);
    };
  }

  new Watcher(
    vm,
    updateComponent,
    noop,
    {
      before() {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, "beforeUpdate");
        }
      },
    },
    true /* isRenderWatcher */
  );
  hydrating = false;

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true;
    callHook(vm, "mounted");
  }
  return vm;
}
```
**这个函数非常的重要，可以算的上是说Vue内部的非常核心的函数**

走了这么多初始化的过程，最后我们终于来到了挂载的地方了

可以看到，我们定义了`updateComponent`，给它赋值了一个函数。然后传入`Watcher`中，`Watcher`的构造函数会执行传入的这个函数，也就是执行`updateComponent`

#### updateComponent
```js
updateComponent = () => {
  vm._update(vm._render(), hydrating);
};
```
`_render`中的代码其实就是拿到我们的`render`函数，执行一下生成`VNode`，之后返回出来

#### vm._update
> src/core/instance/lifecycle.js
```js
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this;
  const prevEl = vm.$el;
  const prevVnode = vm._vnode;
  const restoreActiveInstance = setActiveInstance(vm);

  vm._vnode = vnode;
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode);
  }

  ...
};
```
内部实现也很简单主要是调用了`__patch__`

#### \_\_patch\_\_

> src/platforms/web/runtime/index.js
```js
import { patch } from './patch'

// install platform patch function
Vue.prototype.__patch__ = inBrowser ? patch : noop
```

> src/platforms/web/runtime/patch.js
```js
import { createPatchFunction } from 'core/vdom/patch'

export const patch: Function = createPatchFunction({ nodeOps, modules })
```

> src/core/vdom/patch.js
```js
export function createPatchFunction(backend) {
  return function patch(oldVnode, vnode, hydrating, removeOnly) {
    ...
  }
}
```

其实最后是调用到了`src/core/vdom/patch.js`下`createPatchFunction`函数返回的`patch`函数，这个函数也是我们经常说的`patch`函数，非常的重要

### patch
> src/core/vdom/patch.js
```js
export function createPatchFunction(backend) {
  return function patch(oldVnode, vnode, hydrating, removeOnly) {
     // 如果新节点不存在，老节点存在，则调用 destroy，销毁老节点
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode);
      return;
    }

    let isInitialPatch = false;
    const insertedVnodeQueue = [];

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      // 新的 VNode 存在，老的 VNode 不存在，这种情况会在一个组件初次渲染的时候出现，比如：
      // <div id="app"><comp></comp></div>
      // 这里的 comp 组件初次渲染时就会走这儿
      isInitialPatch = true;
      createElm(vnode, insertedVnodeQueue);
    } else {
      // 判断 oldVnode 是否为真实元素
      const isRealElement = isDef(oldVnode.nodeType);
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // 不是真实元素，但是老节点和新节点是同一个节点，则是更新阶段，执行 patch 更新节点
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly);
      } else {
        // 是真实元素，则表示初次渲染
        if (isRealElement) {
          // 挂载到真实元素以及处理服务端渲染的情况
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR);
            hydrating = true;
          }
          oldVnode = emptyNodeAt(oldVnode);
        }

        // replacing existing element
        // 拿到老节点的真实元素
        const oldElm = oldVnode.elm;
        // 获取老节点的父元素，即 body
        const parentElm = nodeOps.parentNode(oldElm);

        // create new node
        // 基于新 vnode 创建整棵 DOM 树并插入到 body 元素下
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        );
      }
    }
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch);
    return vnode.elm;
  }
}
```
按照我们开始的逻辑我们会走到`createElm`这个函数的执行，这个函数也很重要。

#### createElm
```js
function createElm(
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return;
  }

  // 获取 data 对象
  const data = vnode.data;
  // 所有的孩子节点
  const children = vnode.children;
  const tag = vnode.tag;
  if (isDef(tag)) {
    // 创建新节点
    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode);
    setScope(vnode);

    if (__WEEX__) {
      ...
    } else {
      // 递归创建所有子节点（普通元素、组件）
      createChildren(vnode, children, insertedVnodeQueue);
      if (isDef(data)) {
        invokeCreateHooks(vnode, insertedVnodeQueue);
      }
      // 将节点插入父节点
      insert(parentElm, vnode.elm, refElm);
    }

  } else if (isTrue(vnode.isComment)) {
    // 注释节点，创建注释节点并插入父节点
    vnode.elm = nodeOps.createComment(vnode.text);
    insert(parentElm, vnode.elm, refElm);
  } else {
    // 文本节点，创建文本节点并插入父节点
    vnode.elm = nodeOps.createTextNode(vnode.text);
    insert(parentElm, vnode.elm, refElm);
  }
}
```
首先，会执行`createComponent`函数，我们当前的VNode肯定不是组件VNode，是元素VNode，所以继续往下执行。主要看`createChildren`这个函数

#### createChildren
```js
function createChildren(vnode, children, insertedVnodeQueue) {
  if (Array.isArray(children)) {
    // 遍历这组节点，依次创建这些节点然后插入父节点，形成一棵 DOM 树
    for (let i = 0; i < children.length; ++i) {
      createElm(
        children[i],
        insertedVnodeQueue,
        vnode.elm,
        null,
        true,
        children,
        i
      );
    }
  } else if (isPrimitive(vnode.text)) {
    ...
  }
}
```
其实`createChildren`就是把我们的子元素递归的调用`createElm`去创建元素，那此时我们的`children`有两个元素，一个是组件，一个是普通的元素，这个时候如果是作为组件VNode去调用`createElm`是怎么样的场景呢

通过上面给出的createElm相关的代码，我们肯定是会进入到createComponent这个函数内部的

#### createComponent

```js
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data;
  if (isDef(i)) {
    if (isDef((i = i.hook)) && isDef((i = i.init))) {
      i(vnode, false /* hydrating */);
    }
    if (isDef(vnode.componentInstance)) {
      initComponent(vnode, insertedVnodeQueue);
      insert(parentElm, vnode.elm, refElm);
      if (isTrue(isReactivated)) {
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
      }
      return true;
    }
  }
}
```
在`createComponent`内部调用了i.hook.init上的函数，这个函数是哪来的呢。直接给出答案吧，这个函数是在我们生成组件VNode给我们挂载的，我们直接看函数内部实现吧。
```js
init(vnode: VNodeWithData, hydrating: boolean): ?boolean {
  if (
    vnode.componentInstance &&
    !vnode.componentInstance._isDestroyed &&
    vnode.data.keepAlive
  ) {
    ...
  } else {
    const child = (vnode.componentInstance = createComponentInstanceForVnode(
      vnode,
      activeInstance
    ));
    // 执行子组件的挂载方法
    child.$mount(hydrating ? vnode.elm : undefined, hydrating);
  }
}
```
重点部分在于`createComponentInstanceForVnode`和`$mount`

#### createComponentInstanceForVnode
```js
export function createComponentInstanceForVnode(
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent,
  };
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate;
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render;
    options.staticRenderFns = inlineTemplate.staticRenderFns;
  }
  // 执行构造函数
  return new vnode.componentOptions.Ctor(options);
}
```
我们也主要看`return new vnode.componentOptions.Ctor(options)`这一部分代码

`vnode.componentOptions.Ctor`这个就是Vue实例，只不过它是从根Vue上继承下来的，可以是说子类。那它为什么存在呢，还是因为我们在创建组件VNode的时候挂载上去的。我们只需要记住它是Vue实例，当前这个Vue实例那肯定就是我们定义的组件`Hello-World`的构造函数，那这个时候new了这个构造函数，不就执行了内部的_init了吗

这个时候有同学可能会突然醒悟了，组件初始化不就是执行了我们刚才开始的一样的逻辑吗。所以我们这个时候可以得出一个结论，组件都是一个个Vue实例。

那组件一样会走到patch，一样会走到createElm，那样也就会去创建元素，然后插入。这个时候如果组件内部还有组件，一样也会走createComponent，那么又会走我们刚才的步骤。

### 结论
在Vue初始化过程中，有`patch`过程，其中遇到组件就会重新走一遍初始化以及挂载流程，遇到普通节点，就直接创建，这个过程就是一个深度遍历的过程。就像我们的`Hello-World`组件，如果里面还有组件，就继续初始化，直到内部都是普通节点，然后进行插入，这个过程就很像递归了。所以，这也就回答了，我们开始的问题，组件是如何搭建起来的。