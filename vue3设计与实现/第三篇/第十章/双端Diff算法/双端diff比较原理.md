#### 双端diff比较原理

双端diff算法是一种同时对新旧两组子节点的两个端点进行比较的算法。因此，我们需要四个索引值，分别指向新旧两组子节点的端点，用代码来表达四个端点，如下patchChildren和patchKeyedChildren函数代码所示：

```js
  function patchChildren(n1, n2, container) {
    if (typeof n2.children === 'string') {
      // ...
    } else if (Array.isArray(n2.children)) {
      // 封装 patchKeyedChildren函数处理两组子节点
      patchKeyedChildren(n1, n2, children)
    } else {
      // ...
    }
  }

  function patchKeyedChildren(n1, n2, container) {
    const oldChildren = n1.children
    const newChildren = n2.children
    // 四个索引值
    let oldStartIdx = 0
    let oldEndIdx = oldChildren.length - 1
    let newStartIdx = 0
    let newEndIdx = newChildren.length - 1
  }
```

有了索引之后，就可以找到它所指向的虚拟节点了：

```js
  function patchKeyedChildren(n1, n2, container) {
    const oldChildren = n1.children
    const newChildren = n2.children
    // 四个索引值
    let oldStartIdx = 0
    let oldEndIdx = oldChildren.length - 1
    let newStartIdx = 0
    let newEndIdx = newChildren.length - 1
    // 四个索引指向的vnode节点
    let oldStartVNode = oldChildren[oldStartIdx]
    let oldEndVNode = oldChildren[oldEndIdx]
    let newStartVNode = newChildren[newStartIdx]
    let newEndVNode = newChildren[newEndIdx]
  }
```

有了这些信息就可以进行双端比较了。怎么比较呢？这里暂时没有画图

第一步：

```js
 function patchKeyedChildren(n1, n2, container) {
    const oldChildren = n1.children
    const newChildren = n2.children
    // 四个索引值
    let oldStartIdx = 0
    let oldEndIdx = oldChildren.length - 1
    let newStartIdx = 0
    let newEndIdx = newChildren.length - 1
    // 四个索引指向的vnode节点
    let oldStartVNode = oldChildren[oldStartIdx]
    let oldEndVNode = oldChildren[oldEndIdx]
    let newStartVNode = newChildren[newStartIdx]
    let newEndVNode = newChildren[newEndIdx]

    // 比较key是否相同
    if (oldStartVNode.key === newStartVNode.key) {
      // 第一步： oldStartVNode和newStartVNode比较
    } else if (oldEndVNode.key === newEndVNode.key ) {
      // 第二步：oldEndVNode和newEndVNode比较
    } else if (oldStartVNode.key === newEndVNode.key) {
      // 第三步： oldStartVNode和newEndVNode比较
    } else if (oldEndVNode.key === newStartVNode.key) {
      // 第四步：oldEndVNode和newStartVNode比较
    } {
      // 仍然需要调用patch打补丁
      patch(oldEndVNode, newStartVNode, container)
      // 移动DOM操作
      // oldEndVNode.el移动到oldStartVNode.el前面
      insert(oldEndVNode.el, container, oldStartVNode.el)

      // 移动完成后，更新索引值，并指向下一个位置
      oldEndVNode = oldChildren[--oldEndIdx]
      newStartVNode = newChildren[++newStartIdx]
    }
  }
```

这时diff算法还没结束，还需要进行下一轮的更新。因此，我们需要将更新逻辑封装到一个while循环中，while循环的条件是： __头部索引值要小于等于尾部索引值。__

第二步：

```js
  while(oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
     // 比较key是否相同
    if (oldStartVNode.key === newStartVNode.key) {
      // 第一步： oldStartVNode和newStartVNode比较
    } else if (oldEndVNode.key === newEndVNode.key ) {
      // 第二步：oldEndVNode和newEndVNode比较
      // 节点在新的顺序中仍然处于尾部，不需要移动，仍然需要打补丁
      patch(oldEndVNode, newEndVNode, container)
      // 更新索引和头尾部节点变量
      oldEndVNode = oldChildren[--oldEndIdx]
      newEndVNode = newChildren[--newEndIdx]
    } else if (oldStartVNode.key === newEndVNode.key) {
      // 第三步： oldStartVNode和newEndVNode比较
    } else if (oldEndVNode.key === newStartVNode.key) {
      // 第四步：oldEndVNode和newStartVNode比较
    } {
      // 仍然需要调用patch打补丁
      patch(oldEndVNode, newStartVNode, container)
      // 移动DOM操作
      // oldEndVNode.el移动到oldStartVNode.el前面
      insert(oldEndVNode.el, container, oldStartVNode.el)

      // 移动完成后，更新索引值，并指向下一个位置
      oldEndVNode = oldChildren[--oldEndIdx]
      newStartVNode = newChildren[++newStartIdx]
    }
  }
```

第三步：

```js
  while(oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
     // 比较key是否相同
    if (oldStartVNode.key === newStartVNode.key) {
      // 第一步： oldStartVNode和newStartVNode比较
    } else if (oldEndVNode.key === newEndVNode.key ) {
      // 第二步：oldEndVNode和newEndVNode比较
      // 节点在新的顺序中仍然处于尾部，不需要移动，仍然需要打补丁
      patch(oldEndVNode, newEndVNode, container)
      // 更新索引和头尾部节点变量
      oldEndVNode = oldChildren[--oldEndIdx]
      newEndVNode = newChildren[--newEndIdx]
    } else if (oldStartVNode.key === newEndVNode.key) {
      // 第三步： oldStartVNode和newEndVNode比较
      // 调用patch函数在oldStartVNode和newEndVNode之间打补丁
      patch(oldStartVNode, newEndVNode, container)
      // 将旧的一组子节点的头部节点对应的真实DOM节点oldStartVNode.el移动到
      // 旧的一组子节点的尾部节点对应的真实DOM节点后面
      insert(oldStartVNode.el, container, oldEndVNode.el.nextSibling)
      // 更新相关索引到下一个位置
      oldStartVNode = oldChildren[++oldStartIdx]
      newEndVnode = newChildren[--newEndIdx]
    } else if (oldEndVNode.key === newStartVNode.key) {
      // 第四步：oldEndVNode和newStartVNode比较
    } {
      // 仍然需要调用patch打补丁
      patch(oldEndVNode, newStartVNode, container)
      // 移动DOM操作
      // oldEndVNode.el移动到oldStartVNode.el前面
      insert(oldEndVNode.el, container, oldStartVNode.el)

      // 移动完成后，更新索引值，并指向下一个位置
      oldEndVNode = oldChildren[--oldEndIdx]
      newStartVNode = newChildren[++newStartIdx]
    }
  }
```

如果旧的一组子节点的头部节点与新的一组子节点的尾部节点匹配，则说明该旧节点所对应的真实DOM节点需要移动到尾部。因此我们需要获取当前尾部节点的下一个兄弟节点作为锚点，即oldEndVNode.el.nextSibling。

最后一步：

```js
  while(oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
     // 比较key是否相同
    if (oldStartVNode.key === newStartVNode.key) {
      // 第一步： oldStartVNode和newStartVNode比较
      // 调用patch函数在oldStartVNode和newStartVNode之间打补丁
      patch(oldStartVNode, newStartVNode, container)
      // 更新相关索引，指向下一个位置
      oldStartVNode = oldChildren[++oldStartIdx]
      newStartVNode = newChildren[++newStartIdx]
    } else if (oldEndVNode.key === newEndVNode.key ) {
      // 第二步：oldEndVNode和newEndVNode比较
      // 节点在新的顺序中仍然处于尾部，不需要移动，仍然需要打补丁
      patch(oldEndVNode, newEndVNode, container)
      // 更新索引和头尾部节点变量
      oldEndVNode = oldChildren[--oldEndIdx]
      newEndVNode = newChildren[--newEndIdx]
    } else if (oldStartVNode.key === newEndVNode.key) {
      // 第三步： oldStartVNode和newEndVNode比较
      // 调用patch函数在oldStartVNode和newEndVNode之间打补丁
      patch(oldStartVNode, newEndVNode, container)
      // 将旧的一组子节点的头部节点对应的真实DOM节点oldStartVNode.el移动到
      // 旧的一组子节点的尾部节点对应的真实DOM节点后面
      insert(oldStartVNode.el, container, oldEndVNode.el.nextSibling)
      // 更新相关索引到下一个位置
      oldStartVNode = oldChildren[++oldStartIdx]
      newEndVnode = newChildren[--newEndIdx]
    } else if (oldEndVNode.key === newStartVNode.key) {
      // 第四步：oldEndVNode和newStartVNode比较
    } {
      // 仍然需要调用patch打补丁
      patch(oldEndVNode, newStartVNode, container)
      // 移动DOM操作
      // oldEndVNode.el移动到oldStartVNode.el前面
      insert(oldEndVNode.el, container, oldStartVNode.el)

      // 移动完成后，更新索引值，并指向下一个位置
      oldEndVNode = oldChildren[--oldEndIdx]
      newStartVNode = newChildren[++newStartIdx]
    }
  }
```