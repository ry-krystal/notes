<!--
 * @Description: 
-->
#### React之外的DOM操作
React声明式渲染机制把复杂的DOM操作对象抽象为简单的state和props操作，因此避免了很多直接的DOM操作。不过，仍然有一些DOM操作是React无法避免或者正在努力避免的。

HTML5 audio/video的play方法和input的focus方法，React就无能为力了，这时只能使用相应的DOM方法来实现。

React提供了事件绑定的功能，但是仍然有一些特殊情况需要自行绑定事件，例如Popup等组件，当点击组件其他区域时可以收缩此类组件。这就要求我们对组件以外的区域(一般指document和body)进行事件绑定。例如:
```javascript
  componentDidUpdate(prevProps, prevState) {
    const { isActive: isStateActive } = this.state
    const { isActive: isPrevActive } = prevState
    if (!isStateActive && isPrevActive) { // 未激活
      document.removeEventListener('click', this.hidePopup)
    }

    if (isStateActive && !isPrevActive) { // 已激活
      document.addEventListener('click', this.hidePopup)
    }
  }

  componetWillUnmount() {
    document.removeEventListener('click', this.hidePopup)
  }

  hidePopup(e) {
    if (!this.isMounted()) return false

    // 获取当前组件的所有dom节点
    const node = ReactDOM.findDOMNode(this)
    // 获取点击事件的目标元素
    const target = e.target || e.srcElement
    // 判断这个目标元素有没有在该组件的dom节点中
    const isInside = node.contains(target)

    // 被激活但是没有在当前组件dom里面
    if (isStateActive && !isInside) {
      this.setState({
        isActive: false
      })
    }
  }
```
React中使用DOM最多的还是计算DOM的尺寸(即位置信息)。我们可以提供像width或height这样的工具函数：
```javascript
  function width(el) {
    const styles = el.ownerDocument.defaultView.getComputedStyle(el, null)
    const width = parseFloat(styles.width.indexOf('px') !== -1 ? style.width : 0)

    const boxSizing = styles.boxSizing || 'content-box'
    if (boxSizing === 'border-box') return width

    const { borderLeftWidth, borderRigthWidth, paddingLeft, paddingRight } = styles
    const sborderLeftWidth = parseFloat(borderLeftWidth)
    const sborderRigthWidth = parseFloat(borderRigthWidth)
    const sPaddingLeft = parseFloat(paddingLeft)
    const sPaddingRight = parseFloat(paddingRight)

    return width - sborderLeftWidth - sborderRigthWidth - sPaddingLeft - sPaddingRight
  }
```
上述计算方法并不能完全覆盖所有情况，这需要付出不少的成本去实现。值得高兴的是，React正在自己构建一个DOM排列模型，来努力避免这些React之外的DOM操作。