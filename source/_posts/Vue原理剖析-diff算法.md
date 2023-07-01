---
title: Vue原理剖析-diff算法
date: 2023-06-10 16:24:54
tags:
---

### 前言

在模版编译过程中会产生render函数，之后就可以通过调用render函数生成VNode，再通过patch函数根据VNode去修改真实的DOM节点，那我们怎么确定哪些地方需要修改呢，以及如何用性能最高的方式去修改呢，这就是diff算法要做的事情了。

这一节主要看下Vue中diff算法怎么实现的，以及做了哪些优化。

<!--more-->

### createPatchFunction

createPatchFunction是一个工厂函数，可以接收配置，之后会返回patch函数。这里用了**函数柯里化**的技巧，主要是为了适应多平台的一种优雅解决方案。

```js
export function createPatchFunction(backend) {
  return function patch(oldVnode, vnode, hydrating, removeOnly) {}
}

export const patch: Function = createPatchFunction({ nodeOps, modules });
```

### patch 

我们来重点观察patch函数

```js
return function patch(oldVnode, vnode, hydrating, removeOnly) {
  // 如果新节点不存在，老节点存在，则调用 destroy，销毁老节点
  if (isUndef(vnode)) {
    if (isDef(oldVnode)) invokeDestroyHook(oldVnode);
    return;
  }

  let isInitialPatch = false;
  const insertedVnodeQueue = [];

  if (isUndef(oldVnode)) {
    // 新的 VNode 存在，老的 VNode 不存在，这种情况会在一个组件初次渲染的时候出现，比如：
    // <div id="app"><comp></comp></div>
    // 这里的 comp 组件初次渲染时就会走这儿
    isInitialPatch = true;
    createElm(vnode, insertedVnodeQueue);
  } else {
    // 判断 oldVnode 是否为真实元素
    const isRealElement = isDef(oldVnode.nodeType);
    if (!isRealElement && sameVnode(oldVnode, vnode)) {
      // 不是真实元素，但是老节点和新节点是同一个节点，则是更新阶段，执行 patchVnode 更新节点
      patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly);
    } else {
      // 是真实元素，则表示初次渲染
      if (isRealElement) {
        // 挂载到真实元素以及处理服务端渲染的情况
        if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
          oldVnode.removeAttribute(SSR_ATTR);
          hydrating = true;
        }
        if (isTrue(hydrating)) {
          if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
            invokeInsertHook(vnode, insertedVnodeQueue, true);
            return oldVnode;
          } else if (process.env.NODE_ENV !== "production") {
            warn(
              "The client-side rendered virtual DOM tree is not matching " +
                "server-rendered content. This is likely caused by incorrect " +
                "HTML markup, for example nesting block-level elements inside " +
                "<p>, or missing <tbody>. Bailing hydration and performing " +
                "full client-side render."
            );
          }
        }
        // 走到这儿说明不是服务端渲染，或者 hydration 失败，则根据 oldVnode 创建一个 vnode 节点
        oldVnode = emptyNodeAt(oldVnode);
      }

      // 拿到老节点的真实元素
      const oldElm = oldVnode.elm;
      // 获取老节点的父元素，即 body
      const parentElm = nodeOps.parentNode(oldElm);

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

      // update parent placeholder node element, recursively
      // 递归更新父占位符节点元素
      if (isDef(vnode.parent)) {
        let ancestor = vnode.parent;
        const patchable = isPatchable(vnode);
        while (ancestor) {
          for (let i = 0; i < cbs.destroy.length; ++i) {
            cbs.destroy[i](ancestor);
          }
          ancestor.elm = vnode.elm;
          if (patchable) {
            for (let i = 0; i < cbs.create.length; ++i) {
              cbs.create[i](emptyNode, ancestor);
            }
            // #6513
            // invoke insert hooks that may have been merged by create hooks.
            // e.g. for directives that uses the "inserted" hook.
            const insert = ancestor.data.hook.insert;
            if (insert.merged) {
              // start at index 1 to avoid re-invoking component mounted hook
              for (let i = 1; i < insert.fns.length; i++) {
                insert.fns[i]();
              }
            }
          } else {
            registerRef(ancestor);
          }
          ancestor = ancestor.parent;
        }
      }

      // destroy old node
      if (isDef(parentElm)) {
        removeVnodes([oldVnode], 0, 0);
      } else if (isDef(oldVnode.tag)) {
        invokeDestroyHook(oldVnode);
      }
    }
  }

  invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch);
  return vnode.elm;
}
```

从patch中的代码可以看出，首先判断oldVNode是否存在，不存在的情况就只有组件初始化的情况。如果存在的话进一步去判断，如果不是真实元素并且oldVNode和vnode是同一个元素的话，就执行patchVNode函数进行调用，这里其实算是diff算法的入口。然后不满足这个条件的话，就有可能是首次渲染，就直接拿到oldVNode的真实元素，然后通过createElm根据VNode创建真实DOM元素插入到父节点下，最后将oldVNode对应的DOM删除。


之后我们重点看patchVNode，对于两个相同的节点如何实现高效更新的。

### patchVNode
  
```js
function patchVnode(
  oldVnode,
  vnode,
  insertedVnodeQueue,
  ownerArray,
  index,
  removeOnly
) {
  // 如果新老节点相同，直接结束
  if (oldVnode === vnode) {
    return;
  }

  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // clone reused vnode
    vnode = ownerArray[index] = cloneVNode(vnode);
  }

  const elm = (vnode.elm = oldVnode.elm);

  // 跳过静态节点的更新
  if (
    isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    // 新旧节点都是静态的而且两个节点的 key 一样，并且新节点被 clone 了 或者 新节点有 v-once指令，则重用这部分节点
    vnode.componentInstance = oldVnode.componentInstance;
    return;
  }

  // 执行组件的 prepatch 钩子
  let i;
  const data = vnode.data;
  if (isDef(data) && isDef((i = data.hook)) && isDef((i = i.prepatch))) {
    i(oldVnode, vnode);
  }

  // 老节点的孩子
  const oldCh = oldVnode.children;
  // 新节点的孩子
  const ch = vnode.children;
  // 全量更新新节点的属性，Vue 3.0 在这里做了很多的优化
  if (isDef(data) && isPatchable(vnode)) {
    // 执行新节点所有的属性更新
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode);
    if (isDef((i = data.hook)) && isDef((i = i.update))) i(oldVnode, vnode);
  }

  if (isUndef(vnode.text)) {
    // 新节点不是文本节点
    if (isDef(oldCh) && isDef(ch)) {
      // 如果新老节点都有孩子，则递归执行 diff 过程
      if (oldCh !== ch)
        updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly);
    } else if (isDef(ch)) {
      // 老孩子不存在，新孩子存在，则创建这些新孩子节点
      if (process.env.NODE_ENV !== "production") {
        checkDuplicateKeys(ch);
      }
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, "");
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
    } else if (isDef(oldCh)) {
      // 老孩子存在，新孩子不存在，则移除这些老孩子节点
      removeVnodes(oldCh, 0, oldCh.length - 1);
    } else if (isDef(oldVnode.text)) {
      // 老节点是文本节点，则将文本内容置空
      nodeOps.setTextContent(elm, "");
    }
  } else if (oldVnode.text !== vnode.text) {
    // 新节点是文本节点，则更新文本节点
    nodeOps.setTextContent(elm, vnode.text);
  }
  if (isDef(data)) {
    if (isDef((i = data.hook)) && isDef((i = i.prepatch)))
      i(oldVnode, vnode);
  }
}
```
在patchVNode中会做如下的事情
- 先会比较新老节点是否相同，如果相同则直接返回，不需要更新。
- 然后会判断新节点是否是静态节点，如果是静态节点则直接返回，不需要更新。
- 如果VNode是组件VNode，就会执行组件的prepatch钩子去更新组件。
- 更新节点上的属性
  - 如果是普通节点更新的话，会调用之前在创建patch函数中传入的配置项中的update钩子函数去更新节点的属性。
  - 如果是组件更新的话，会调用组件的update钩子函数去更新组件。
- 如果VNode没有文本属性，此时有很多种情况
  - 新节点和老节点都有孩子节点，就会执行updateChildren去更新孩子节点。这个函数之后讲
  - 如果只有新孩子节点有，老孩子节点没有，就直接创建新孩子节点去插入
  - 如果只有老孩子节点有，新孩子节点没有，就直接删除老孩子节点
  - 如果老节点是文本节点，就将文本内容置空
- 如果VNode有文本属性，也就是说是文本节点，并且新老节点的文本不相同，就直接更新文本节点的内容即可。
- 最后如何VNode是组件的话，还会执行组件的prepatch钩子函数。


### updateChildren
updateChildren内部就是diff算法了。

```js
  function updateChildren(
    parentElm,
    oldCh,
    newCh,
    insertedVnodeQueue,
    removeOnly
  ) {
    // 老节点的开始索引
    let oldStartIdx = 0;
    // 新节点的开始索引
    let newStartIdx = 0;
    // 老节点的结束索引
    let oldEndIdx = oldCh.length - 1;
    // 第一个老节点
    let oldStartVnode = oldCh[0];
    // 最后一个老节点
    let oldEndVnode = oldCh[oldEndIdx];
    // 新节点的结束索引
    let newEndIdx = newCh.length - 1;
    // 第一个新节点
    let newStartVnode = newCh[0];
    // 最后一个新节点
    let newEndVnode = newCh[newEndIdx];
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm;

    // removeOnly是一个特殊的标志，仅由 <transition-group> 使用，以确保被移除的元素在离开转换期间保持在正确的相对位置
    const canMove = !removeOnly;

    if (process.env.NODE_ENV !== "production") {
      // 检查新节点的 key 是否重复
      checkDuplicateKeys(newCh);
    }

    // 遍历新老两组节点，只要有一组遍历完（开始索引超过结束索引）则跳出循环
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        // 如果节点被移动，在当前索引上可能不存在，检测这种情况，如果节点不存在则调整索引
        oldStartVnode = oldCh[++oldStartIdx]; // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx];
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        // 老开始节点和新开始节点是同一个节点，执行 patch
        patchVnode(
          oldStartVnode,
          newStartVnode,
          insertedVnodeQueue,
          newCh,
          newStartIdx
        );
        oldStartVnode = oldCh[++oldStartIdx];
        newStartVnode = newCh[++newStartIdx];
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        // 老结束和新结束是同一个节点，执行 patch
        patchVnode(
          oldEndVnode,
          newEndVnode,
          insertedVnodeQueue,
          newCh,
          newEndIdx
        );
        oldEndVnode = oldCh[--oldEndIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldStartVnode, newEndVnode)) {
        // 老开始和新结束是同一个节点，执行 patch
        // Vnode moved right
        patchVnode(
          oldStartVnode,
          newEndVnode,
          insertedVnodeQueue,
          newCh,
          newEndIdx
        );
        canMove &&
          nodeOps.insertBefore(
            parentElm,
            oldStartVnode.elm,
            nodeOps.nextSibling(oldEndVnode.elm)
          );
        oldStartVnode = oldCh[++oldStartIdx];
        newEndVnode = newCh[--newEndIdx];
      } else if (sameVnode(oldEndVnode, newStartVnode)) {
        // 老结束和新开始是同一个节点，执行 patch
        // Vnode moved left
        patchVnode(
          oldEndVnode,
          newStartVnode,
          insertedVnodeQueue,
          newCh,
          newStartIdx
        );
        canMove &&
          nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
        oldEndVnode = oldCh[--oldEndIdx];
        newStartVnode = newCh[++newStartIdx];
      } else {
        // 如果上面的四种假设都不成立，则通过遍历找到新开始节点在老节点中的位置索引

        // 找到老节点中每个节点 key 和 索引之间的关系映射 => oldKeyToIdx = { key1: idx1, ... }
        if (isUndef(oldKeyToIdx))
          oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx);

        // 在映射中找到新开始节点在老节点中的位置索引
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);

        if (isUndef(idxInOld)) {
          // New element
          // 在老节点中没找到新开始节点，则说明是新创建的元素，执行创建
          createElm(
            newStartVnode,
            insertedVnodeQueue,
            parentElm,
            oldStartVnode.elm,
            false,
            newCh,
            newStartIdx
          );
        } else {
          // 在老节点中找到新开始节点了
          vnodeToMove = oldCh[idxInOld];
          if (sameVnode(vnodeToMove, newStartVnode)) {
            // 如果这两个节点是同一个，则执行 patch
            patchVnode(
              vnodeToMove,
              newStartVnode,
              insertedVnodeQueue,
              newCh,
              newStartIdx
            );
            // patch 结束后将该老节点置为 undefined
            oldCh[idxInOld] = undefined;
            canMove &&
              nodeOps.insertBefore(
                parentElm,
                vnodeToMove.elm,
                oldStartVnode.elm
              );
          } else {
            // same key but different element. treat as new element
            // 最后这种情况是，找到节点了，但是发现两个节点不是同一个节点，则视为新元素，执行创建
            createElm(
              newStartVnode,
              insertedVnodeQueue,
              parentElm,
              oldStartVnode.elm,
              false,
              newCh,
              newStartIdx
            );
          }
        }

        // 新节点向后移动一个
        newStartVnode = newCh[++newStartIdx];
      }
    }

    // 走到这里，说明老节点或者新节点被遍历完了
    if (oldStartIdx > oldEndIdx) {
      // 说明老节点被遍历完了，新节点有剩余，则说明这部分剩余的节点是新增的节点，然后添加这些节点
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm;
      addVnodes(
        parentElm,
        refElm,
        newCh,
        newStartIdx,
        newEndIdx,
        insertedVnodeQueue
      );
    } else if (newStartIdx > newEndIdx) {
      // 说明新节点被遍历完了，老节点有剩余，说明这部分的节点被删掉了，则移除这些节点
      removeVnodes(oldCh, oldStartIdx, oldEndIdx);
    }
  }

export function insertBefore (parentNode: Node, newNode: Node, referenceNode: Node) {
  parentNode.insertBefore(newNode, referenceNode)
}
```
可以看出这里面代码还算复杂，但是我们一个个case去看，就会发现其实也很好理解。

在开始之前定义了很多变量，这些是在之后会用上的。之后调用了checkDuplicateKeys去检查key是否有重复。

之后就开始循环了，而且采用的是while循环。`while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx)`，从循环的条件来看，只要新老开始索引有一个大于结束索引就会跳出循环。

在循环里面有很多if else，我们一个个来看。
- isUndef(oldStartVnode)
  - 如果老开始节点未定义，就将oldStartIdx++，并且oldStartVnode更新一下。这样做的目的是因为在遍历过程中，可能会遇到undefined的情况，所以需要跳过。
- isUndef(oldEndVnode)
  - 如果老结束节未定义，就将oldEndIdx--，并且oldEndVnode更新一下。这样做的目的和上一个是一样的意思。
- sameVnode(oldStartVnode, newStartVnode)
  - 这种情况就是老开始和新开始是同一个节点。就直接执行patchVnode去深层次继续比较，然后将新老开始索引都++。
- sameVnode(oldEndVnode, newEndVnode)
  - 这种情况就是老结束和新结束是同一个节点。就直接执行patchVnode去深层次继续比较，然后将新老结束索引都--。
- sameVnode(oldStartVnode, newEndVnode)
  - 这种情况就是老开始和新结束是同一个节点。就直接执行patchVnode去深层次继续比较，然后将老开始索引++，新结束索引--。
  - 因为两个节点的位置不一样，所以涉及到节点的移动，之后就将**老开始节点移动到老结束节点的后面**。
- sameVnode(oldEndVnode, newStartVnode)
  - 这种情况就是老结束和新开始是同一个节点。就直接执行patchVnode去深层次继续比较，然后将老结束索引--，新开始索引++。
  - 因为两个节点的位置不一样，所以涉及到节点的移动，之后就将**老结束节点移动到未处理节点的前面**。
- 最后这种情况就是以上情况都没命中
  - 如果没有定义老节点中key对应idx的对象，就先通过createKeyToOldIdx生成一个
  - 如果新节点有key属性
    - 通过当前新节点中的key去上面那个对象中找到对应的索引
    - 如果没有key，通过findIdxInOld去找对应的索引
  - 之后对idxInOld去判断，也就是对新节点对应在老节点中的索引进行判断
    - 如果压根没在老节点中找到对应的新节点，那么这个新节点就是新创建的，执行createElm去创建，并且之后**插入到未处理节点的前面**
    - 如果找到了，并且新老节点都是一样的，就直接执行patchVnode去深层次继续比较
      - 因为两个节点的位置不一样，所以涉及到节点的移动，之后就将**老结束节点移动到未处理节点的前面**。
    - 新老节点不一样的话，就说明新节点需要创建，执行createElm去创建，并且之后**插入到未处理节点的前面**
- 之后将新开始索引++

可以看出逻辑非常多，但是需要我们注意的情况有几个。
- 第一个就是我们如果有需要移动的节点或者新增的节点我们都选择插入到**未处理的节点前面**，除了老开始和新结束这种情况，可以确认这个节点是在最后面的，所以我们选择插入到老结束节点的后面。那其它情况为什么要选择插入到**未处理的节点前面**。**这样是为了保证顺序是正确的**

- 第二个就是我们光在里面移动位置了，没有删除的情况。那之后是怎么删除的呢？其实在while执行完之后，有判断是老节点被遍历完了还是新节点被遍历完了。如何是老节点先被遍历完，那新节点还有剩余，说明这部分剩余的节点是新增的节点，然后添加这些节点。如果是新节点被遍历完了，那老节点还有剩余，说明老节点这部分已经不需要了，则移除这些节点。

- 第三个就是在while中不是先用循环的方式去找节点的对应位置，而是多了四个判断，如果是四个判断没有命中，才会用这种循环找的这种方法。**这其实是diff算法的优化部分**。通过预测的方法，如果命中那么就有优化，而且命中一般都比较高。这四种查找方法分别是：**新前与旧前、新后与旧后、新前与旧后、新后与旧前**。

- 第四个就是，就算我们定义的四种情况没有命中，那么**我们也会通过key去查对应的元素**，最后才会用效率最低的循环去遍历查找。所以说，我们在日常写的时候，尽量加上key，这样可以在数据量多的时候很大的提高效率。