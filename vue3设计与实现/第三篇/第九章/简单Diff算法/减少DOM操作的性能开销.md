#### 减少DOM操作的性能开销

核心Diff只关心新旧虚拟节点都存在一组子节点的情况。

```javascript
  // 旧vnode
  const oldVNode = {
    type: 'div',
    children: [
      {
        type: 'p', children: '1'
      },
      {
        type: 'p', children: '2'
      },
      {
        type: 'p', children: '3'
      }
    ]
  }
  // 新的vnode
  const newVNode = {
    type: 'div',
    children: [
      {
        type: 'p', children: '4'
      },
      {
        type: 'p', children: '5'
      },
      {
        type: 'p', children: '6'
      }
    ]
  }
```

按照之前的做法，当更新子节点时，我们需要执行6次DOM操作：

- 卸载所有旧子节点，需要3次DOM删除操作
- 挂载所有新子节点，需要3次DOM添加操作
  
但是观察上面新旧vnode的子节点，可以发现：

- 更新前后的所有子节点都是标签，即标签元素不变
- 只有p标签的子节点（文本节点）会发生变化

最理想的更新方式是，直接更新这个p标签的文本节点的内容。一共只需要3次DOM操作就可以完成全部节点的更新。

按照这个思路，我么可以重新实现两组子节点的更新逻辑，下面patchChildren函数的代码所示：

```javascript
  function patchChildren(n1, n2, container) {
    if (typeof n2.children === 'string') {
      // 省略部分代码
    } else if (Array.isArray(n2.children)) {
      // 重新实现两组子节点的更新方式
      // 新旧children
      const oldChildren = n1.children
      const newChildren = n2.children
      // 遍历旧的children
      for(let i = 0, len = oldChildren.length; i < len; i++) {
        // 调用patch函数逐个更新子节点
        patch(oldChildren[i], newChildren[i])
      }
    }
  }
```

patch函数在执行更新时，发现新旧子节点只有文本内容不同，因此只会更新其文本节点的内容。

__这样做法虽然能够减少DOM操作次数，但问题很明显。在上面的代码中，我们通过遍历旧的一组子节点，并假设新的一组子节点的数量与之相同，只有在这种情况下，这段代码才能正确地工作。但是，新旧两组子节点的数量未必相同。当新的一组子节点的数量少于旧的一组子节点的数量时，意味着有些节点在更新后应该被卸载。__

在进行新旧两组子节点的更新时，不应该总是遍历旧的一组子节点或遍历新的一组子节点，而是应该遍历其中长度较短的那一组。这样，我们才能够尽可能多地调用patch函数进行更新。接着，再对比新旧两组子节点的长度。如果新的一组子节点更长，则说明有新子节点需要挂载，否则说明有旧子节点需要卸载。最终实现如下：

```javascript
  function patchChildren(n1, n2, container) {
    if (typeof n2.children === 'string') {

    } else if (Array.isArray(n2.children)) {
      const oldChildren = n1.children
      const newChildren = n2.children
      // 旧的一组子节点的长度
      const oldLen = oldChildren.length
      // 新的一组子节点的长度
      const newLen = newChildren.length
      // 两组子节点的公共长度，即两者中较短的那一组子节点的长度
      const commonLen = Math.min(oldLen, newLen)
      // 遍历commonLen次
      for(let i = 0; i < commonLen; i++) {
        // 更新操作
        patch(oldChildren[i], newChildren[i], container)
      }
      // 如果newLen > oldLen,说明有新子节点需要挂载
      if (newLen > oldLen) {
        for (let i = commonLen; i < newLen; i++) {
          patch(null, newChildren[i], container)
        }
      } else if (newLen < oldLen) {
        // 如果newLen < oldLen,说明有旧子节点需要卸载
        for (let i = commonLen; i < oldLen; i++) {
          unmount(oldChildren[i])
        }
      }
    } else {
      // ...
    }
  }
```

这样，无论新旧两组子节点的数量关系如何，渲染器都能够正确地挂载或卸载它们。