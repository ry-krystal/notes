<!-- slide -->
#### HTML Attributes 与 DOM Properties

理解HTML Attributes 和 DOM Properties之间的差异和关联非常重要，这能够帮助我们合理地设计虚拟节点的结构，更是正确地为元素设置属性的关键。
我们从最基本的HTML说起，给出如下HTML代码：

```javascript
  <input id="my-input" type="text" value="foo" />
```

HTML Attributes 指的就是定义在HTML标签上的属性，这里指的就是id="my-input"、type="text"和value="foo"。当浏览器解析这段HTML代码后，会创建一个与之相符的DOM元素对象，我们可以通过JavaScript代码来读取该DOM对象：

```javascript
  const el = document.querySelector('#my-input')
```

<!-- slide -->

这个DOM对象包含很多属性。这些属性就是所谓的DOM Properties。很多HTML Attributes 在DOM对象上有与之同名的DOM Properties,例如id="my-input"对应el.id，type="text"对应el.type， value=“foo”对应el.value等。但DOM Properties 与 HTML Attributes 的名字不总是一模一样的，例如：

```html
  <div class="foo"></div>
```

class="foo"对应的DOM Properties则是el.className。另外，并不是所有HTML Properties都有与之对应的DOM Properties,例如：

```html
  <div aria-valuenow="75"></div>
```
<!-- slide -->

aria-*类的HTML Attributes就没有与之对应的DOM Properties。
类似地，也不是所有DOM Properties都有与之对应的HTML Attributes,例如可以用el.textContent来设置元素的文本内容，但并没有与之对应的HTML Attributes来完成同样的工作。

HTML Attributes 的值与DOM Properties的值之间是有关联的，例如：

```html
  <div id="foo"></div>
```

这个片段描述了一个具有id属性的div标签。其中，id="foo"对应的DOM Properties 是 el.id，并且值为字符串“foo”。 我们把这种HTML Attributes 与 DOM Properties具有相同名称(即 id)的属性看作直接映射。但这并不是所有HTML Attributes 与 DOM Properties 之间都是直接映射的关系， 例如：

```html
  <input value="foo" />
```
<!-- slide -->

这里一个具有value属性的input标签。如果用户没有修改文本框的内容，那么通过el.value读取对应的DOM Properties 的值就是字符串“foo”。而如果用户修改了文本框的值，那么el.value的值就是当前文本框的值。例如，用户将文本框的内容修改为“bar”，那么：

```javascript
  console.log(el.value) // ‘bar’
```

如果运行下面的代码，会发生“奇怪”的现象：

```javascript
  console.log(el.getAttribute('value')) // 仍然是‘foo’
  console.log(el.value) // ‘bar’
```

<!-- slide -->

可以发现，用户对文本框的内容的修改并不会影响el.getAttribute('value')的返回值，这个现象蕴含着HTML Attributes 所代表的意义。实际上，__HTML Attributes的作用是设置与之对应的DOM Properties的初始值。一旦值改变，那么DOM Properties 始终存储着当前值，而通过getAttribute 函数得到的仍然是初始值。__

但我们仍然可以通过el.defaultValue来访问初始值，如下代码所示：

```javascript
  console.log(el.getAttribute('value')) // 仍然是‘foo’
  console.log(el.value) // ‘bar’
  el.defaultValue // ‘foo’
```

<!-- slide -->

这说明一个HTML Attributes可能关联多个DOM Properties。例如上例中，value="foo"与el.value和el.defaultValue都有关联。

虽然我们可以认为HTML Attributes是用来设置与之对应的DOM Properties的初始值的，但有些值是受限制的，就好像浏览器内部做了默认值校验。如果你通过HTML Attributes 提供的默认值不合法，那么浏览器会使用内建的合法值作为对应DOM Properties的默认值，例如：

```html
  <input type="foo" />
```

<!-- slide -->

我们知道，为\<input />标签的type属性指定字符串‘foo’是不合法的，因此浏览器会矫正这个不合法的值。所以我们尝试读取el.type时，得到的其实是矫正后的值，即字符串“text”，而非字符串“foo”:

```javascript
  console.log(el.type) // ‘text’
```

其实我们只需要记住一个核心原则即可：__HTML Attributes的作用是设置与之对应的DOM Properties的初始值。__