#### DOM复用与key的作用

通过减少DOM操作的次数，提升了更新性能。但这种方式仍然存在可优化的空间。举个例子，假设新旧两组子节点的内容如下：

```javascript
  // oldChildren
  [
    { type: 'p' },
    { type: 'div' },
    { type: 'span' }
  ]
  // newChildren
  [
    { type: 'span' },
    { type: 'p' },
    { type: 'div' }
  ]
```

按上次的算法来完成上述两组子节点的更新，则需要6次DOM操作。

- 调用patch函数在旧子节点{ type: 'p' }与新子节点{ type: 'span' } 之间打补丁，由于两者是不同的标签，所以patch函数会卸载 { type: 'p' }, 然后再挂载{ type: 'span' },这需要执行2次DOM操作。

- 与第1步类似，卸载旧子节点{ type: 'div' }, 然后再挂载新子节点{ type: 'p' },这也需要执行2次DOM操作。
  
- 与第1步类似，卸载旧子节点{ type: 'div' }, 然后再挂载新子节点{ type: 'div' }, 同样需要执行2次DOM操作。
  
最优的处理方式是，__通过DOM的移动来完成子节点的更新，这要比不断地执行子节点的卸载和挂载性能更好。但是，想要通过DOM的移动来完成更新，必须保证一个前提：新旧两组子节点中的确存在可复用的节点。__ 这个很好理解，如果新的子节点没有在旧的一组子节点中出现，就无法通过移动节点的方式完成更新。所以现在问题变成了：应该如何确定新的子节点是否出现在旧的一组子节点中呢？

一种解决方案是，通过vnode.type判断，只要vnode.type的值相同，我们就认为两者是相同的节点。但这种方式并不靠谱。例如：

```javascript
  // oldChildren
  [
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

  // newChildren
  [
    {
      type: 'p', children: '3'
    },
    {
      type: 'p', children: '1'
    },
    {
      type: 'p', children: '2'
    }
  ]
```

通过上面两组子节点，我们发现，这个案例可以通过移动DOM的方式来完成更新。__但是所有节点的vnode.type属性值都相同，这导致我们无法确定新旧子节点中节点的对应关系，也就无法得知应该进行怎样的DOM移动才能完成更新。这时，就需要引入额外的key来作为vnode的标识.__

```javascript
  [
    {
      type: 'p', children: '1', key: 1
    },
    {
      type: 'p', children: '2', key: 2
    },
    {
      type: 'p', children: '3', key: 3
    }
  ]

  // newChildren
  [
    {
      type: 'p', children: '3', key: 3
    },
    {
      type: 'p', children: '1', key: 1
    },
    {
      type: 'p', children: '2', key: 2
    }
  ]
```

__key属性就像虚拟节点的"身份证"号，只要两个虚拟节点的type属性值和key属性值都相同，那么我们就认为它们是相同的，即可以进行DOM的服用。__

如果没有key,我们无法知道新子节点与旧子节点间的映射关系，也就无法知道应该如何移动节点。__有key的话情况则不同，我们根据子节点的key属性，能够明确知道新子节点在旧子节点中的位置，这样就可以进行相应的DOM移动操作了。__

有必要强调一点是，DOM可复用并不意味着不需要更新，如下面的两个虚拟节点所示：

```javascript
  const oldVNode = {
    type: 'p',
    key: 1,
    children: 'text 1'
  }

  const newVNode = {
    type: 'p',
    key: 1,
    children: 'text 2'
  }
```

这两个虚拟节点拥有相同的key值和vnode.type属性值。这意味着，在更新时可以复用DOM元素，即只需要通过移动操作来完成更新。但仍需要对这个虚拟节点进行打补丁操作，因为新的虚拟节点(newVNode)的文本子节点的内容已经改变了。因此，在讨论如何移动DOM之前，我们需要先完成打补丁操作，如下面patchChildren函数的代码所示：

```javascript
  function patchChildren(n1, n2, container) {
    if (typeof n2.children === 'string') {
      // 省略部分代码
    } else if (Array.isArray(n2.children)) {
      const oldChildren = n1.children
      const newChildren = n2.children

      // 遍历新的children
      for(let i = 0, newLen = newChildren.length; i < newLen; i++) {
        const newVNode = newChildren[i]
        // 遍历旧的children
        for(let j = 0, oldLen = oldChildren.length; j < oldLen; j++) {
          const oldVNode = oldChildren[j]
          // 如果找到了具有相同key值的两个节点，说明可以复用，但仍然需要调用patch函数更新
          if (newVNode.key === oldVNode.key) {
            patch(oldVNode, newVNode, container)
            break
          }
        }
      }
    } else {
      // ...
    }
  }
```

在上面的这段代码中，我们重新实现了新旧两组子节点的更新逻辑。可以看到，我们使用了两层for循环，外层循环用于遍历新的一组子节点，内层循环则遍历旧的一组子节点。在内层循环中，我们逐个对比新旧子节点的key值，试图在旧的子节点中找到可复用的节点。一旦找到，则调用patch函数进行打补丁。经过这一步操作之后，我们能够保证所有可复用的节点本身都已经更新完毕了。下面以新旧两组子节点为例：

```javascript
  const oldVNode = {
    type: 'div',
    children: [
      {
        type: 'p', children: '1', key: 1
      },
      {
        type: 'p', children: '2', key: 2
      },
      {
        type: 'p', children: '3', key: 3
      }
    ]
  }

  const newVNode = {
    type: 'div',
    children: [
      {
        type: 'p', children: 'world', key: 3
      },
      {
        type: 'p', children: '1', key: 1
      },
      {
        type: 'p', children: '2', key: 2
      }
    ]
  }

  // 首次挂载
  renderer.render(oldVNode, document.querySelector('#app'))
  setTimeout(() => {
    // 1 秒钟后更新
    renderer.render(newVNode, document.querySelector('#app'))
  })
```

经过上述更新操作后，所有节点对应的真实DOM元素都更新完毕了。但真实DOM仍然保持旧的一组子节点的顺序，即key值为3的节点对应的真实DOM仍然是最后一个子节点。由于在新的一组子节点中，key值为3的节点已经变为第一个子节点了，因此我们还需要通过移动节点来完成真实DOM顺序的更新。