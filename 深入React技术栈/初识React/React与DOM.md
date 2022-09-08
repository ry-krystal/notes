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
