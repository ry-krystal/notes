<!-- slide -->
#### class的处理

在Vue.js，仍然需要一些特殊处理，比如class属性。为什么要做特殊处理呢？这是因为Vue.js对class属性做了增强。在Vue.js中为元素设置类名有以下几种方式。

方式一：指定class为一个字符串值

```html
  <p className="foo bar"></p>
```
<!-- slide -->

这段模板对应的vnode是：

```javascript
  const vnode = {
    type: 'p',
    props: {
      class: 'foo bar'
    }
  }
```
<!-- slide -->

方式二： 指定class为一个对象值。

```html
  <p :class="cls"></p>
```
<!-- slide -->

假设对象cls的内容如下：

```javascript
  const cls = {
    foo: true,
    bar: false
  }
```
<!-- slide -->

那么，这段模板对应的vnode是：

```javascript
  const vnode = {
    type: 'p',
    props: {
      class: {
        foo: true,
        bar: false
      }
    }
  }
```
<!-- slide -->

方式三：class是包含上述两种类型的数组。

```html
  <p :class="arr"></p>
```

这个数组可以是字符串与对象值的组合：

```javascript
  const arr = [
    // 字符串
    'foo bar',
    {
      baz: true
    }
  ]
```
<!-- slide -->

那么，这段模板对应的vnode是：

```javascript
  const vnode = {
    type: 'p',
    props: {
      class: [
        'foo bar',
        { baz: true }
      ]
    }
  }
```
<!-- slide -->

可以看到，因为class的值可以是多种类型，所以我们必须在设置元素的class之前将值归一化为统一的字符串形式，再把该字符串作为元素的class值去设置。因此，我们需要封装normalizeClass函数，用它来将不同类型的class值正常化为字符串，例如：

```javascript
  const vnode = {
    type: 'p',
    props: {
      // 使用normalizeClass函数对值进行序列化
      class: normalizeClass([
        'foo bar',
        {
          baz: true
        }
      ])
    }
  }
```
<!-- slide -->

最后结果等价于：

```javascript
  const vnode = {
    type: 'p',
    props: {
      // 序列化的结果
      class: 'foo bar baz'
    }
  }
```

<!-- slide -->

在浏览器中为一个元素设置class有三种方式，即使用setAttribute、el.className或el.classList。那么哪一种性能更好呢？对比三种方式为元素设置1000次class的性能。
可以看到，el.className的性能最优。因此我们需要调整patchProps函数的实现，如下：
<!-- slide -->

```javascript
  const renderer = createRenderer({
    // 省略其他项

    patchProps(el, key, preValue, nextValue) {
      // 对class进行特殊处理
      if (key === 'class') {
        el.className = nextValue || ''
      } else if (shouldSetAsProps(el, key, nextValue)) {
        const type = typeof el[key]
        if (type === 'boolean' && value === '') {
          el[key] = true
        } else {
          el[key] = nextValue
        }
      } else {
        el.setAttribute(key, nextValue)
      }
    }
  })
```
<!-- slide -->

通过对class的处理，我们能够意识到，vnode.props对象中定义的属性值的类型并不总是与DOM元素属性的数据结构保持一致，这取决于上层API的设计。Vue.js允许对象类型的值作为class是为了方便开发者，在底层实现上，必须需要对值进行正常化后再使用。