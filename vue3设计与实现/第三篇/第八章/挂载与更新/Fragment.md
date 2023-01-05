<!-- slide -->
#### Fragment

Fragment(片断)是Vue3中新增的一个vnode类型，在具体讨论Fragment的实现之前，我们有必要先了解为什么需要Fragment。请思考这样的场景，假设我们需要封装一组列表组件：

```html
  <List>
    <Items></Items>
  </List>
```
<!-- slide -->

整体是两个组件构成，即\<List>组件和\<Items>组件。其中\<List>组件会渲染一个\<ul>标签作为包裹层：

```html
  <!-- List.vue -->
  <template>
    <ul>
      <slot></slot>
    </ul>
  </template>
```

<!-- slide -->

而\<Items>组件负责渲染一组\<li>列表：

```html
  <!-- Items.vue -->
  <template>
    <li>1</li>
    <li>2</li>
    <li>3</li>
  </template>
```

<!-- slide -->

这在Vue2中是无法实现的，Vue2中，组件模板不允许存在多个根节点。这意味着，一个\<Items>组件最多只能渲染一个\<li>标签：

```html
  <!-- Item.vue -->
  <template>
    <li>1</li>
  </template>
```

<!-- slide -->

因此，在Vue2中，我们通常需要配合v-for指令来达到目的：

```html
  <List>
    <items v-for="item in list"></items>
  </List>
```

<!-- slide -->

那么，Vue3是如何用vnode来描述多根节点模板的呢？答案是，使用Fragment,如下所示：

```javascript
  const Fragment = Symbol()
  const vnode = {
    type: Fragment,
    children: [
      {
        type: 'li', children: 'text1'
      },
      {
        type: 'li', children: 'text2'
      },
      {
        type: 'li', children: 'text3'
      }
    ]
  }
```
<!-- slide -->

有了Fragment之后，我们就可以用它来描述Items.vue组件中的模板了:

```html
  <template>
    <li>1</li>
    <li>2</li>
    <li>3</li>
  </template>
```

<!-- slide -->

这段模板对应的虚拟节点是：

```javascript
  const vnode = {
    type: 'Fragment',
    children: [
      {
        type: 'li', children: '1'
      },
      {
        type: 'li', children: '2'
      },
       {
        type: 'li', children: '3'
      }
    ]
  }
```

<!-- slide -->

类似地，如下模板：

```html
  <List>
    <Items></Items>
  </List>
```

<!-- slide -->

可以用虚拟节点来描述它：

```javascript
  const vnode = {
    type: 'ul',
    children: [
      {
        type: 'Fragment',
        children: [
          {
            type: 'li', children: '1'
          },
          {
            type: 'li', children: '2'
          },
           {
            type: 'li', children: '3'
          }
        ]
      }
    ]
  }
```

<!-- slide -->

当渲染器渲染Fragment类型的虚拟节点时，由于Fragment本身并不会渲染任何内容，所以渲染器只会渲染Fragment的子节点：
<!-- slide -->
```javascript
  function patch(n1, n2, container) {
    if (n1 && n1.type !== n2.type) {
      unmount(n1)
      n1 = null
    }
    const { type } = n2
    if (typeof type === Fragment) {
      if (!n1) {
        // 如果旧vnode不存在，则只需要将Fragment的children逐个挂载即可
        n2.children.forEach(c => patch(null, c, container))
      } else {
        // 如果旧vnode存在，则只需要更新Fragment的children即可
        patchChildren(n1, n2, container)
      }
    }
  }
```
<!-- slide -->

从本质上来说，渲染Fragment与渲染普通元素的区别在于，Fragment本身并不渲染任何内容，所以只需要处理它的子节点即可。但仍然需要注意一点，unmount函数也需要支持Fragment类型的虚拟节点的卸载：

```javascript
  function unmount(vnode) {
    // 在卸载时，如果卸载的vnode的类型为Fragment,则需要卸载其children
    if (vnode.type === Fragment) {
      vnode.children.forEach(v => unmount(c))
      return
    }
    const parent = vnode.el.parentNode
    if (parent) {
      parent.removeChild(vnode.el)
    }
  }
```