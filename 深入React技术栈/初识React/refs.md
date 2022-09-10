<!--
 * @Description: refs
-->
#### refs
我们利用render方法得到了App组件的实例，然后就可以对它做一些操作。但在组价内，JSX是不会返回一个组件的实例的！它只是一个ReactElement,只是告诉React被挂载的组件应该长什么样：
```javascript
  const myApp = <App />
```
refs就为此而生，它是React组件中非常特殊的prop,可以附加到任何一个组件上。从字面意思来看，__refs即reference,组件被调用时会创建一个该组件的实例，而refs就会指向这个实例。__

它可以是一个回调函数，这个回调函数会在组件被挂载后立即执行。例如：
```javascript
  import React, { Component } from 'react'

  class App extends Component {
    constructor(props) {
      super(props)
      this.handleClick = this.handleClick.bind(this)
    }

    handleClick() {
      if (this.myTextInput !== null) {
        this.myTextInput.focus()
      }
    }

    render() {
      return (
        <div>
          <input 
            type="text" 
            ref={(ref) => this.myTextInput = ref} 
          />
          <input
            type="button"
            value="Focus the text input"
            onClick={this.handleClick}
          />
        </div>
      )
    }
  }
```
这个例子里，我们得到input组件的真正实例，所以可以在按钮被按下后调用输入框的focus()方法。这个例子把 __refs放到原生的DOM组件input中，我们可以通过refs得到DOM节点；而如果把refs放到React组件，比如\<TextInput />，我们获得的就是TextInput的实例，因此就可以调用TextInput的实例方法。__

refs同样支持字符串。对于DOM操作，不仅可以使用findDOMNode获得该组件DOM，还可以使用refs获得组件内部的DOM。比如：
```javascript
  import React, { Component } from 'react'
  import ReactDOM from 'react-dom'

  class App extends Component {
    componentDidMount() {
      // myComp 是 Comp的一个实例，因此需要用 findDOMNode 转换为相应的DOM
      const myComp = this.refs.myComp
      const dom = ReactDOM.findDOMNode(myComp)
    }

    render() {
      return (
        <div>
          <Comp ref="myComp" />
        </div>
      )
    }
  }
```

```javascript
  this.refs.myProtal.openProtal()
```
这种命令式调用的方式，尽管说并不是React推崇的，但我们仍然可以使用。原则上，在组件状态维护中不建议用这种方式。

为防止内存泄露，当卸载一个组件的时候，组件里所有的refs就会变成null。

__值得注意的是，findDOMNode和refs都无法用于无状态组件中，原因是无状态组件挂载时只是方法调用，没有新建实例。__

对于React组件来说，refs指向一个组件类的实例，所以可以调用该类定义的任何方法。如果需要访问该组件的真实DOM，可以用React.findDOMNode来找到DOM节点，但我们并不推荐这样做。因为这在大部分情况下都打破了封装性，而且通常都能用更清晰的办法在React中构建代码。
