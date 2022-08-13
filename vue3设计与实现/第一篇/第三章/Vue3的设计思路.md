<!--
 * @Description: Vue.js 3的设计思路
 * @version: 1.0
 * @Author: renyong
 * @Date: 2022-07-02 17:24:02
 * @LastEditors: renyong
 * @LastEditTime: 2022-07-03 12:50:03
-->
### 第三章 Vue.js 3的设计思路
##### 本篇围绕5个章节：  
- 声明式地描述UI  
- 初识渲染器  
- 组件的本质  
- 模板的工作原理  

#### 3.1 声明式地描述UI
```javascript
  <h1 @click="handler">
    <span></span>
  </h1>
  <script>
    // h 标签的级别
    let h_hevel = 3
    // 使用javascript对象来描述UI(虚拟DOM)
    const title = {
      // 标签名称
      tag: `h${h_hevel}`,
      // 标签属性
      props: {
        onClick: handler
      },
      // 子节点
      children: [
        {
          tag: 'span'
        }
      ]
    }
  </script>
```
也可以手写渲染函数也就是使用虚拟DOM来描述UI
```javascript
 import { h } from 'vue'
  export default {
    render() {
      return h('h1', { onClick: handler }) // 虚拟dom
    }
  }
```
h 函数的返回值就是一个对象，其作用是让我们编写虚拟DOM变得更轻松。
```javascript
  export default {
    render() {
      return {
        tag: 'h1',
        props: {
          onClick: handler
        }
      }
    }
  }
```
一个组件要渲染的内容是通过渲染函数来描述的，也就是上面代码中的render函数，Vue.js会根据组件的render函数的返回值拿到虚拟DOM，然后就把组件的内容渲染出来了

#### 3.2 初始渲染器
虚拟DOM：其实就是用javascript对象来描述真实的DOM结构
```javascript
const vnode = {
  tag: 'div',
  props: {
    onClick: () => alert('hello')
  },
  children: 'click me'
}
```
编写一个渲染器，把上面的虚拟DOM渲染为真实DOM:
```javascript
/**
 * @Description: 
 * vnode:虚拟DOM对象
 * container: 一个真实的DOM元素，作为挂载点，渲染器会把虚拟DOM渲染到该挂载点下
 * @return {*}
 * @author: renyong
 */
const renderer = (vnode, container) => {
  const tag = vnode.tag
  // 使用vnode.tag 作为标签名称创建DOM元素
  const el = document.createElement(tag)
  // 遍历vnode.props，将属性、事件添加到DOM元素
  const props = Object.keys(vnode.props)
  props.forEach(key => {
    const reg = /^on/
    if (reg.test(key)) {
      // 如果key 以on开头，说明它是事件
      el.addEventListener(
        key.substr(2).toLowerCase(), // 事件名称 onClick ---> click
        vnode.props[key] // 事件处理函数
      )
    }
  })

  // 处理children
  const children = vnode.children
  if(typeof children === 'string') {
    // 如果children是字符串，说明它是元素的文本子节点
    el.appendChild(document.createTextNode(children))
  } else if (Array.isArray(children)) {
    // 递归地调用renderer函数渲染子节点，使用当前元素el作为挂载点
    children.forEach(child => renderer(child, el))
  }

  // 将元素添加到挂载点下
  container.appendChild(el)
}
renderer(vnode, document.body) // body作为挂载点
```
#### 3.3  组件的本质
__组件是一组DOM元素的封装__，这组DOM元素就是组件要渲染的内容
```javascript
  const MyComponent = () => {
    return {
      tag: 'div',
      props: {
        onClick: () => alert('hello')
      },
      children: 'click me'
    }
  }
  // 让虚拟DOM对象的tag属性存储组件函数
  const vnode = {
    tag: MyComponent
  }
```
修改renderer函数:
```javascript
  const renderer = (vnode, container) => {
    const tag = vnode.tag
    if (typeof tag === 'string') {
      // 说明vnode描述的是标签元素
      mountElement(vnode, container)
    } else if(typeof tag === 'function') {
      // 说明vnode描述的是组件
      mountComponent(vnode, container)
    }
  }
```
mountElement函数：
```javascript
  const mountElement = (vnode, container) => {
    // 使用vnode.tag作为标签名称创建DOM元素
    const tag = vnode.tag
    const el = document.createElement(tag)
    // 遍历vnode.props, 将属性、事件添加到DOM元素
    const props = Object.keys(vnode.props)
    props.forEach(key => {
      const reg = /^on/
      if (reg.test(key)) {
        // 如果key是以字符串on 开头，说明它是事件
        el.addEventListener(
          key.substr(2).toLowerCase(), // 事件名称 onClick ---> click
          vnode.props[key] // 事件处理函数
        )
      }
    })

    // 处理children
    const children = vnode.children
    if (typeof children === 'string') {
      // 如果chidlren是字符串，说明它是元素的文本子节点
      el.appendChild(document.createTextNode(children))
    } else if(Array.isArray(children)) {
      // 递归地调用renderer函数渲染子节点，使用当前元素el作为挂载点
      children.forEach(child => renderer(child, el))
    }

    // 将元素添加到挂载点下
    container.appendChild(el)
  }
```
再看mountComponent是如果实现的：
```javascript
  const mountComponent = (vnode, container) => {
    // 调用组件函数，获取组件要渲染的内容(虚拟DOM)
    const subtree = vnode.tag()
    // 递归调用renderer渲染subtree
    renderer(subtree, container)
  }
```
我们也可以用一个javascript对象来表达一个组件，其返回值代表组件要渲染的内容
```javascript
  // MyComponent是一个对象
  const MyComponent = {
    render() {
      return {
        tag: 'div',
        props: {
          onClick: () => alert('hello')
        },
        children: 'click me'
      }
    }
  }
```
修改渲染器条件：
```javascript
  const renderer = (vnode, container) => {
    const tag = vnode.tag
    if (typeof tag === 'string') {
      mountElement(vnode, container)
    } else if (typeof tag === 'function') {
      mountComponent(vnode, container)
    } else if(typeof tag === 'object') {
      mountComponent(vnode, container)
    }
  }
```
接下来修改mountComponent:
```javascript
  const mountComponent = (vnode, container) => {
    // vnode.tag 是组件对象，调用它的render函数就得到组件要渲染的内容（虚拟DOM）
    const subtree = vnode.tag.render()
    // 递归调用renderer渲染subtree
    renderer(subtree, container)
  }
```
#### 3.4 模板工作原理
Vue.js框架中另一个重要的组成部分：编译器，其作用就是将模板编译成渲染函数：
```javascript
  <div @click="handler">
    click me
  </div>
```
对于编译器来说，模板就是一个普通的字符串，它会分析该字符串并生成一个功能与之相同的渲染函数“
```javascript
  render() {
    return h('div', { onClick: handler }, 'click me')
  }
```
以我们熟悉的.vue文件为例，一个.vue文件就是一个组件，如下所示：
```javascript
  <template>
    <div @click="handler">
      click me
    </div>
  </template>

  <script>
    export default {
      data() {/*  */},
      methods: {
        handler() {/*  */}
      }
    }
  </script>
```
其中tempalte标签里面的内容就是模板内容，编译器会把模板内容编译成渲染函数并添加到<script\>标签块的组件对象上，所以最终在浏览器里运行的代码就是：
```javascript
   export default {
      data() {/*  */},
      methods: {
        handler() {/*  */}
      },
      render() {
        return h('div', { onClick: handler }, 'click me')
      }
    }
```
所以，无论是使用模板还是直接手写渲染函数，对于一个组件来说，它要渲染的内容最终都是通过渲染函数产生的，然后渲染器再把渲染函数返回的虚拟DOM渲染成真实DOM，这就是模板工作原理，也就是Vue.js渲染页面的流程

模板编译流程：模块---->编译器---->渲染函数(render)--->渲染器(renderer) ----> 真实DOM


