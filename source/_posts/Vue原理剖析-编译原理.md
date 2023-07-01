---
title: Vue原理剖析-编译原理
date: 2023-06-10 16:22:08
tags:
---
### 前言

关于Vue编译相关内容应该算的上是Vue中最枯燥的部分了，其中有很多分支逻辑，以及各种正则匹配。

那这次我们带着这两个问题去看源码
- Vue编译的流程是怎么样的
- render函数是怎么生成VNode

<!--more-->

### Vue编译的流程是怎么样的

首先，Vue会先将模块编译成AST语法树，在 [AST explorer](https://astexplorer.net/) 中可以去看看Vue模版转成AST是怎么样的。获取到AST之后，会进行下一步**静态节点标记**，这一步主要是将静态节点标记一次，也就是不会变化的节点，因为不会变了所以之后做diff就没必要重新创建，这一步是做性能优化的一步。最后一步就是将优化后的AST生成render函数。

总的来说Vue编译流程有三步：**解析 -> 优化 -> 代码生成**

```js
export const createCompiler = createCompilerCreator(
  function baseCompile(
    template: string,
    options: CompilerOptions
  ): CompiledResult {
    // 1、将 html 模版解析成 ast
    const ast = parse(template.trim(), options);

    // 2、对 ast 树进行静态标记
    if (options.optimize !== false) {
      optimize(ast, options);
    }

    // 3、将 ast 生成渲染函数
    const code = generate(ast, options);

    return {
      ast,
      render: code.render,
      staticRenderFns: code.staticRenderFns,
    };
  }
);

const { compile, compileToFunctions } = createCompiler(baseOptions)
```
从上面的代码就能看出编译模版的全部过程，但是仔细看发现并不是在这里执行这些代码的，这里只是将`baseCompile`做为参数传入了`createCompilerCreator`函数中。`createCompilerCreator`的返回值并不是一个普通值，而是一个函数。其实这里就是**函数柯里化**的一个体现了。那为什么要这样去设计？

```js
export function createCompilerCreator(baseCompile: Function): Function {
  return function createCompiler(baseOptions: CompilerOptions) {
    function compile(
      template: string,
      options?: CompilerOptions
    ): CompiledResult {
      // 以平台特有的编译配置为原型创建编译选项对象
      const finalOptions = Object.create(baseOptions);
      ...
    }

    return {
      compile,
      compileToFunctions: createCompileToFunctionFn(compile),
    };
  };
}
```
因为Vue不单单针对于web平台，它还可以编译weex，也就是说有两个平台。但是两个平台的编译配置可能是不一样的，但是核心流程是不变的，于是采用了**函数柯里化**的方式去实现。先将`baseCompile`确定，然后再根据不同的平台去传入不同的配置，这样就做到了代码的复用，以一种相对优雅的方法去解决平台差异问题，这个思想很值得我们去学习。

那这样的话，如果还有一个平台，那么我们一样直接拿到createCompiler然后传入这个平台的配置就可以很快速的兼容另外一个平台，因为编译过程是不变的，变的只是对于不同的情况做不同的处理。

其实在之后创建 **patch** 函数的地方也用到了**函数柯里化**的技巧和这里的`createCompilerCreator`类似。之后提到 **patch** 的时候会单独拿出来说。

#### 解析

> 这一步是将模版编译成AST语法树

从上面提供的代码我们可以看到第一步就只调用了一个parse函数，将template和平台特定配置传入，之后就直接把AST返回了。所以说我们这一步的重点是parse函数。

> src/compiler/parser/index.js
```js
export function parse(
  template: string,
  options: CompilerOptions
): ASTElement | void {
  let root;

  parseHTML(template, {
    ...
    start(tag, attrs, unary, start, end) {
    },
    end(tag, start, end) {
    },
    chars(text: string, start: number, end: number) {
    },
    comment(text: string, start, end) {
    },
  })

  return root
}
```
parse函数的代码逻辑非常多，但是重点就在于parseHTML。

> src/compiler/parser/html-parser.js
```js
export function parseHTML(html, options) {
  const stack = [];

  while(html) {
    ...
  }
}
```
parseHTML的代码也非常多。所以我就直接讲逻辑了。

首先，parseHTML根据很多正则关系去匹配标签或者文本等等，例如匹配到开始标签时，会将该标签压入栈中，并且会调用在parse函数中定义的start钩子函数去创建AST，然后解析标签上的各种属性添加到AST上。

如果遇到结束标签，弹出栈顶的元素，这样做的好处是可以很容易的构建父子关系，并且还会调用end钩子函数。

如果遇到文本内部，就会调用chars钩子做相关逻辑处理。

遇到注释，就会调用comment钩子。

而且在每次处理完当前匹配的内容时，都会裁剪html，这样当html为空字符时，解析完成。

##### 总结
parse函数用于生成AST。生成AST的过程主要是通过HTML解析器完成的，通过触发不同的钩子函数可以构建不同的节点，而且在其中我们运用了栈维护层级关系。当HTML解析器运行完之后，我们就可以得到一个带DOM层级的AST。


#### 优化

接下来，我们看第二步优化。这一步主要是将静态节点标记。

```js
// 2、对 ast 树进行静态标记
if (options.optimize !== false) {
  optimize(ast, options);
}
```

编译器中是通过optimize函数标记我们上一步生成的AST

> src/compiler/optimizer.js
```js
export function optimize(root: ?ASTElement, options: CompilerOptions) {
  if (!root) return;

  isStaticKey = genStaticKeysCached(options.staticKeys || "");

  // 遍历所有节点，给每个节点设置 static 属性，标识其是否为静态节点
  markStatic(root);

  // 进一步标记静态根
  markStaticRoots(root, false);
}
```

首先深度遍历所有节点去标记静态节点，之后再去标记静态根节点

```js
function markStatic(node: ASTNode) {
  // 通过 node.static 来标识节点是否为 静态节点
  node.static = isStatic(node);
  if (node.type === 1) {
    /**
     * 不要将组件的插槽内容设置为静态节点，这样可以避免：
     *   1、组件不能改变插槽节点
     *   2、静态插槽内容在热重载时失败
     */
    if (
      !isPlatformReservedTag(node.tag) &&
      node.tag !== "slot" &&
      node.attrsMap["inline-template"] == null
    ) {
      // 递归终止条件，如果节点不是平台保留标签  && 也不是 slot 标签 && 也不是内联模版，则直接结束
      return;
    }
    // 遍历子节点，递归调用 markStatic 来标记这些子节点的 static 属性
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i];
      // 递归标记节点
      markStatic(child);

      // 如果子元素是动态节点的话，父元素肯定是动态节点
      if (!child.static) {
        node.static = false;
      }
    }

    // 如果节点存在 v-if、v-else-if、v-else 这些指令，则依次标记 block 中节点的 static
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        const block = node.ifConditions[i].block;
        markStatic(block);
        if (!block.static) {
          node.static = false;
        }
      }
    }
  }
}

function isStatic(node: ASTNode): boolean {
  if (node.type === 2) {
    // expression
    // 比如：{{ msg }}
    return false;
  }
  if (node.type === 3) {
    // text
    return true;
  }
  return !!(
    node.pre ||
    (!node.hasBindings && // no dynamic bindings
      !node.if &&
      !node.for && // not v-if or v-for or v-else
      !isBuiltInTag(node.tag) && // not a built-in
      isPlatformReservedTag(node.tag) && // not a component
      !isDirectChildOfTemplateFor(node) &&
      Object.keys(node).every(isStaticKey))
  );
}
```
node上有一个type属性，1表示元素节点，2表示动态文本节点，3表示纯文本节点

markStatic首先通过isStatic去判断node是否为静态节点，根据type判断。之后单独判断type=1的情况。

如果type=1，会去递归其子节点，也就是去深度遍历子节点，之后还会判断如果子节点已经是动态节点了，那父节点应该改成动态节点。


```js
function markStaticRoots(node: ASTNode, isInFor: boolean) {
  if (node.type === 1) {
    if (node.static || node.once) {
      // 节点是静态的 或者 节点上有 v-once 指令，标记 node.staticInFor = true or false
      node.staticInFor = isInFor;
    }
    if (
      node.static &&
      node.children.length &&
      !(node.children.length === 1 && node.children[0].type === 3)
    ) {
      // 节点本身是静态节点，而且有子节点，而且子节点不只是一个文本节点，则标记为静态根 => node.staticRoot = true，否则为非静态根
      node.staticRoot = true;
      return;
    } else {
      // 优化成本大于收益，所以这种情况直接不标记
      node.staticRoot = false;
    }

    // 当前节点不是静态根节点的时候，递归遍历其子节点，标记静态根
    if (node.children) {
      for (let i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for);
      }
    }

    // 如果节点存在 v-if、v-else-if、v-else 指令，则为 block 节点标记静态根
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        markStaticRoots(node.ifConditions[i].block, isInFor);
      }
    }
  }
}
```
标注静态根节点直接是对type=1进行判断，其中有一个判断是，当前节点是静态，而且有子元素，并且这个子元素不能是一个单独的文本节点，那么就设置成静态根节点，否则就不算。为什么要这样？因为这种情况就存在**优化成本大于收益**，所以索性不优化。从这里能看出Vue团队做了很多考虑。

之后和标记静态节点一样，深度遍历子节点去标记静态根节点。

##### 总结
优化部分主要分两步，首先标记所有静态节点，之后再标记所有静态根节点，而且标记静态根节点的过程中，对于子元素是一个单独的文本节点这种情况不标记，因为**优化成本大于收益**。

#### 代码生成
这一步是编译过程的最后一步，它的作用就是将AST转换成代码字符串然后包装在函数中去执行，再返回这个函数，这个函数也就是渲染函数，也叫render函数。那我们之后拿到render函数再调用就可以直接生成VNode，然后做patch了。

看到这我相信肯定有疑惑，那就是代码字符串是什么？

例如有这样的一个模版
```html
<div id="el">Hello {{name}}</div>
```

它首先会生成AST，然后被优化，最后的AST长这样
```json
{
  "type": 1,
  "ns": 0,
  "tag": "div",
  "tagType": 0,
  "props": [
    {
      "type": 6,
      "name": "id",
      "value": {
        "type": 2,
        "content": "el",
      },
    }
  ],
  "isSelfClosing": false,
  "children": [
    {
      "type": 2,
      "content": "Hello ",
    },
    {
      "type": 5,
      "content": {
        "type": 4,
        "isStatic": false,
        "isConstant": false,
        "content": "name",
      },
    }
  ],
}
```
之后到了我们的最后一步代码生成，生成的代码字符串是
```js
with(this) {
  return _c("div", {
    attrs: {"id": "el"},
    [
      _v("Hello " + _s(name))
    ]
  })
}
```
但是我们最后是得到一个函数，也就是render函数，其实就是代码字符串外面包了一层函数，执行这个函数也就会执行代码字符串中的内容。
```js
new Function(code)
```

那像代码字符串中的_c、_v这些是什么。这些其实是某些函数的简写，例如_c是createElement函数。那createElement函数主要是通过传入的参数生成VNode，所以渲染函数可以生成VNode的原因是因为执行了代码字符串中的函数。

生成代码字符串是通过generate函数生成
```js
// 3、将 ast 生成渲染函数
const code = generate(ast, options);

export function generate(
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  // 实例化 CodegenState 对象，生成代码的时候需要用到其中的一些东西
  const state = new CodegenState(options);

  const code = ast ? genElement(ast, state) : '_c("div")';
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns,
  };
}
```
可以发现如果AST为空，会生成一个空div，否则通过genElement生成code

```js
export function genElement(el: ASTElement, state: CodegenState): string {
  if (el.parent) {
    el.pre = el.pre || el.parent.pre;
  }

  if (el.staticRoot && !el.staticProcessed) {
    /**
     * 处理静态根节点，生成节点的渲染函数
     *   1、将当前静态节点的渲染函数放到 staticRenderFns 数组中
     *   2、返回一个可执行函数 _m(idx, true or '')
     */
    return genStatic(el, state);
  } else if (el.once && !el.onceProcessed) {
    /**
     * 处理带有 v-once 指令的节点，结果会有三种：
     *   1、当前节点存在 v-if 指令，得到一个三元表达式，condition ? render1 : render2
     *   2、当前节点是一个包含在 v-for 指令内部的静态节点，得到 `_o(_c(tag, data, children), number, key)`
     *   3、当前节点就是一个单纯的 v-once 节点，得到 `_m(idx, true of '')`
     */
    return genOnce(el, state);
  } else if (el.for && !el.forProcessed) {
    /**
     * 处理节点上的 v-for 指令
     * 得到 `_l(exp, function(alias, iterator1, iterator2){return _c(tag, data, children)})`
     */
    return genFor(el, state);
  } else if (el.if && !el.ifProcessed) {
    /**
     * 处理带有 v-if 指令的节点，最终得到一个三元表达式：condition ? render1 : render2
     */
    return genIf(el, state);
  } else if (el.tag === "template" && !el.slotTarget && !state.pre) {
    /**
     * 当前节点不是 template 标签也不是插槽和带有 v-pre 指令的节点时走这里
     * 生成所有子节点的渲染函数，返回一个数组，格式如：
     * [_c(tag, data, children, normalizationType), ...]
     */
    return genChildren(el, state) || "void 0";
  } else if (el.tag === "slot") {
    /**
     * 生成插槽的渲染函数，得到
     * _t(slotName, children, attrs, bind)
     */
    return genSlot(el, state);
  } else {
    // component or element
    // 处理动态组件和普通元素（自定义组件、原生标签）
    let code;
    if (el.component) {
      /**
       * 处理动态组件，生成动态组件的渲染函数
       * 得到 `_c(compName, data, children)`
       */
      code = genComponent(el.component, el, state);
    } else {
      // 自定义组件和原生标签走这里
      let data;
      if (!el.plain || (el.pre && state.maybeComponent(el))) {
        // 非普通元素或者带有 v-pre 指令的组件走这里，处理节点的所有属性，返回一个 JSON 字符串，
        // 比如 '{ key: xx, ref: xx, ... }'
        data = genData(el, state);
      }

      // 处理子节点，得到所有子节点字符串格式的代码组成的数组，格式：
      // `['_c(tag, data, children)', ...],normalizationType`，
      // 最后的 normalization 表示节点的规范化类型，不是重点，不需要关注
      const children = el.inlineTemplate ? null : genChildren(el, state, true);
      // 得到最终的字符串格式的代码，格式：
      // '_c(tag, data, children, normalizationType)'
      code = `_c('${el.tag}'${
        data ? `,${data}` : "" // data
      }${
        children ? `,${children}` : "" // children
      })`;
    }

    // 如果提供了 transformCode 方法，
    // 则最终的 code 会经过各个模块（module）的该方法处理，
    // 不过框架没提供这个方法，不过即使处理了，最终的格式也是 _c(tag, data, children)
    for (let i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code);
    }
    return code;
  }
}
```

生成code的过程中有很多逻辑，先是生成根节点，之后递归子节点生成子节点字符串再拼在根节点的参数中，其实根本过程就是字符串拼接，然后再返回code。

到这里，我们对Vue编译模版的过程已经有了一个大概的了解。


### render函数是怎么生成VNode

在我们对编译的最后一个流程中提到过render函数中其实是代码字符串，然后字符串中有很多函数的别名。例如_c其实就是createElement。通过运行时执行这些别名函数来生成的VNode。

那类似于这些别名有哪些呢？

> src/core/instance/render-helpers/index.js
```js
export function installRenderHelpers(target: any) {
  /**
   * v-once 指令的运行时帮助程序，为 VNode 加上打上静态标记
   * 有点多余，因为含有 v-once 指令的节点都被当作静态节点处理了，所以也不会走这儿
   */
  target._o = markOnce;
  // 将值转换为数字
  target._n = toNumber;
  /**
   * 将值转换为字符串形式，普通值 => String(val)，对象 => JSON.stringify(val)
   */
  target._s = toString;
  /**
   * 运行时渲染 v-for 列表的帮助函数，循环遍历 val 值，依次为每一项执行 render 方法生成 VNode，最终返回一个 VNode 数组
   */
  target._l = renderList;

  // 解析插槽
  target._t = renderSlot;

  /**
   * 判断两个值是否相等
   */
  target._q = looseEqual;
  /**
   * 相当于 indexOf 方法
   */
  target._i = looseIndexOf;

  /**
   * 运行时负责生成静态树的 VNode 的帮助程序，完成了以下两件事
   *   1、执行 staticRenderFns 数组中指定下标的渲染函数，生成静态树的 VNode 并缓存，下次在渲染时从缓存中直接读取（isInFor 必须为 true）
   *   2、为静态树的 VNode 打静态标记
   */
  target._m = renderStatic;

  target._f = resolveFilter;
  target._k = checkKeyCodes;
  target._b = bindObjectProps;

  /**
   * 为文本节点创建 VNode
   */
  target._v = createTextVNode;
  /**
   * 为空节点创建 VNode
   */
  target._e = createEmptyVNode;
  // 解析作用域插槽
  target._u = resolveScopedSlots;

  target._g = bindObjectListeners;
  target._d = bindDynamicKeys;
  target._p = prependModifier;
}
```
installRenderHelpers函数在Vue初始化的之后会被执行，也就是说这些别名函数会被安装到Vue的实例上，之后调用render函数的时候，with中的this是Vue实例，那么就可以生成VNode了。

_c没有在这定义，而是在`src/core/instance/render.js`中定义。

如果之后想了解对应的别名函数，直接去看它对应的函数就能明白到底是想生成怎么样的VNode了。
