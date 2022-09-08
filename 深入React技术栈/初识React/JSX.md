<!--
 * @Author: renyong
 * @Date: 2022-08-27 18:18:59
 * @LastEditors: Please set LastEditors
 * @LastEditTime: 2022-08-27 23:12:06
 * @Description: 
-->
### 1.JSX基本语法
示例：
```javascript
const List = () => (
  <div>
    <Title>this is title</Title>
  </div>
)
```
##### 注意以下两点：
- __1.定义标签时，只允许被一个标签包裹__
  以下错误示例：
  ```javascript
    const component = (
      <span>name</span>
      <span>value</>
    )
  ```
  为什么？
  原因：一个标签会被转译成对应的React.createElement调用方法，最外层没有包裹，显然无法转译成方法调用。
- __2.标签一定要闭合__

##### 展开属性
```javascript
  const component = <Component name={name} value={ value }>
  
  // 这里可以用es6 rest/spread 特性提高效率
  const data = {
    name: 'foo',
    value: 'bar'
  }
  const components = <Component { ...data } />
```
##### 自定义HTML属性
如果在JSX中往DOM元素中传入自定义属性，React是不会渲染的：
```html
  <div d="xxx">Content</div>
```
如果要使用HTML自定义属性，要使用data-前缀，这与HTML标准也是一致的：
```html
  <div data-d="xxx"></div>
```
然而，在自定义标签中任意的属性都是被支持的：
```html
  <x-my-component custom-attr="foo">
```
以aria- 开头的网络无障碍属性同样可以正常使用：
```html
  <div aria-hidden={true}></div>
```
##### Javascript表达式
```javascript
  // 输入
  const person = <Person name={window.isLoggedIn ? window.name : ''} />
  // 输出
  const person = React.createElement(
    Person,
    {
      name: window.isLoggedIn ? window.name : ''
    }
  )

  // 子组件也可以作为表达式使用：
  // 输入
  const content = <Container>
    {
      window.isLoggedIn ? <Nav /> : <Login />
    }
  </Container>

  // 输出
  const content = React.createElement(
    Container,
    null,
    window.isLoggedIn ? React.createElement(Nav) : React.createElement(Login)
  )
```
##### HTML转义
React 会将所有要显示到DOM的字符串转义，防止XSS。所以，如果JSX中含有转义后的实体字符，比如\&copy;(&copy;), 则最后DOM中不会正确显示，因为React自动把@copy;中的特殊符转义了。有几种解决办法：

  - 直接使用utf-8字符&copy;
  - 使用对应字符的Unicode编码查询编码
  - 使用数组组装\<div>{['cc', \<span>\&copy;\</span>, '2022']}\</div>
  - 直接插入原始的HTML

此外，React提供了dangerouslySetInnerHTML属性，正如其名，它的作用就是避免React转义字符，在确定必要的情况下可以使用它：
```html
  <div dangerouslySetInnerHTML={{__html: 'cc &copy; 2022'}}></div>
```