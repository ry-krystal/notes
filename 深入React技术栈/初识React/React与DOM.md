<!--
 * @Description: 
-->
#### React 与 DOM
ReactDOM中的API非常少，只有findDOMNode、unmountComponentAtNode和render。

1. findDOMNode
DOM真正被添加到HTML中的生命周期方法是componentDidMount和componentDidUpdate方法。在这两个方法中，我们可以获取真正的DOM元素。React提供的获取DOM元素的方法有两种，其中一种就是ReactDOM提供的findDDOMNode:
```javascript
  import React, { Component } from 'react'
  import ReactDOM from 'react-dom'

  class App extends Component {
    componentDidMount() {
      // this 为当前组件的实例
      const dom = ReactDOM.findDOMNode(this)
    }

    render()
  }
```
如果在render中返回null,那么findDOMNode也返回null。findDOMNode 只对已经挂载的组件有效。
涉及到复杂操作时，还有非常多的原生DOM API可以用。但是需要严格限制场景，在使用之前多问自己为什么要操作DOM。

2. render
  为什么说只有在顶层组件我们才不得不使用ReactDOM呢？这是因为要把React渲染的Virtual DOM渲染到浏览器的DOM当中，就要使用render方法了：
  ```javascript
    ReactComponent render(
      ReactElement element,
      DOMelement container,
      [function callback]
    )
  ```
  该方法把元素挂载到container中，并且返回元素的实例。当然，如果是无状态组件，render会返回null。当组件装载完毕时，callback就会被调用。
  当组件初次渲染之后再次更新时，React不会把整个组件重新渲染一次，而会用它高效的DOM diff算法做局部的更新。这也是React最大的亮点之一。