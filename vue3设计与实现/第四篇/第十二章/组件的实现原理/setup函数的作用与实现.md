#### setup函数的作用与实现

setup函数主要用于配合组合式API，为用户提供一个地方，用于建立组合逻辑、创建响应式数据、创建通用函数、注册生命周期钩子等能力。在组件的整个生命周期中，setup函数只会被挂载执行一次，它的返回值可以有两种情况。

```js
  const Comp = {
    setup() {
      // setup函数可以已返回一个函数，改函数将作为组件的渲染函数
      return () => {
        return {
          type: 'div',
          children: 'hello'
        }
      }
    }
  }
```

这种方式常用于组件不是以模板来表达其渲染内容的情况。如果组件以模版来表达其渲染的内容，那么setup函数不可以再返回函数，否则会与模板编译生成的渲染函数产生冲突。

返回一个对象，该对象中包含的数据将暴露给模板使用：

```js
  const Comp = {
    setup() {
      const count = ref(0)

      // 返回一个对象，对象中的数据会暴露到渲染函数中
      return {
        count
      }
    }，
    render() {
      // 通过this可以访问setup暴露出来的响应式数据
      return {
        type: 'div',
        children: `count is ${this.count}`
      }
    }
  }
```

另外，setup函数接收两个参数，第一个参数是props数据对象，第二个参数也是一个对象，通常称为setupContext, 如下所示：

```js
  const Comp = {
    props: {
      foo: String
    },
    setup(props, setupContext) {
      props.foo // 访问传入的props数据
      // setupContext 中包含与组件接口相关的重要数据
      const { slots, emit, attrs, expose } = setupContext

      // ...
    }
  }
```