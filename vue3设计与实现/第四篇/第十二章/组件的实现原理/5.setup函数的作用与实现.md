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

```js
  function mountComponent(vnode, container, anchor) {
    const componentOptions = vnode.type
    // 从组件选项中取出setup函数
    let { render, data, setup, /* 省略其他选项 */} = componentOptions

    beforeCreate && beforeCreate()

    const state =  data ?  reactive(data) : null
    const [props, attrs] = resolveProps(propsOption, vnode.props)

    // 组件实例
    const instance = {
      state,
      props: shallowReactive(props),
      isMounted: false,
      subTree: null
    }
    
    // setupContext， 由于我们还没讲解emit和slots, 所以暂时只需要attrs
    const setupContext = { attrs }
    // 调用setup函数，将只读版本的props作为第一个参数传递，避免用户意外地修改props的值,
    // 将setupContext作为第二个参数传递
    const setupResult = setup(shallowReadonly(instance.props), setupContext)
    // setupState用来存储由setup返回的数据
    let setupState = null
    if (typeof setupResult === 'function') {
      // 报告冲突
      if (render) console.error('setup 函数返回渲染函数， render选项将被忽略')
      render = setupResult
    } else {
      // 如果setup 的返回值不是函数，则作为数据状态赋值给setupState
      setupState = setupResult
    }

    vnode.component = instance

    const renderContext = new Proxy(instance, {
      get(t, k, r) {
        const { state, props } = t
        if (state && k in state) {
          return state[k]
        } else if(k in props) {
          return props[k]
        } else if (setupState && k in setupState) {
          // 渲染上下文需要增加对setupState的支持
          return setupState[k]
        } else {
          console.error('不存在')
        }
      },
      set(t, k, v, r) {
        const { state, props } = t
        if (state && k in state) {
          state[k] = v
        } else if(k in props) {
          props[k] = v
        } else if (setupState && k in setupState) {
          // 渲染上下文需要增加对setupState的支持
          setupState[k] = v
        } else {
          console.error('不存在')
        }
      }
    })
    // ...
  }
```

上面是setup函数的最小实现，这里注意几点：

- setupContext是一个对象，由于我们还没有讲解关于emit和slots的内容，因此setupContext暂时只包含attrs.
- 如果setup函数的返回类型为函数，则直接将其作为组件的渲染函数，这需要注意的是，为了避免产生歧义，我们需要检查组件选项中是否已经存在render选项，如果存在，则需要打印警告信息。
- 渲染上下文renderContext应该正确地处理setupState，因为setup函数返回的数据状态也应该暴露到渲染环境。