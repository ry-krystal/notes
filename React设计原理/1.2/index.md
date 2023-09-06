@@ -1,89 +0,0 @@
#### 编程：细粒度更新

首先，实现useState,用来定义自变量：

```js
  function useState(value) {
    const getter = () => value
    const setter = (newValue) => value = newValue

    return [getter, setter] 
  }
```
useState接收初始值value为参数，形成闭包。调用getter取值会返回闭包中的value, 调用setter赋值会修改闭包中的value。与React不同，返回数组[0]并不是value，而是getter。这样做的意义是什么呢？

使用方式如下：
```js
  const [count, setCount] = useState(0)

  console.log(count()) // 0
  setCount(1)
  console.log(count()) // 1
```

接下来实现”有副作用因变量“ --- useEffect，期望的行为是：

1. useEffect执行后，回调函数立即执行；
2. 依赖的自变量变化后，回调函数立即执行；
3. 不需要显示指明依赖；

```js
  const [count, setCount] = useState(0)

  // effect1
  useEffect(() => {
    console.log('count is:', count()) // count is: 0
  })

  // effect2
  useEffect(() => {
    console.log('没有什么事儿') // 没有什么事儿
  })

  setCount(2) // count is: 2
```

实现的关键在于建立useState与useEffect的订阅发布关系：

1. 在useEffect回调中执行useState的getter时，该effect会订阅”该state的变化“。
2. useState的setter在执行时，会向所有”订阅了该state变化“的effect发布通知。
   
state内部的集合subs用来保存”订阅该state变化的effect“。effect是每个useEffect对应的数据结构：

```js
  const effect = {
    // 用于执行useEffect回调函数
    execute,
    // 保存该useEffect依赖的state对应subs的集合
    dep: new Set()
  }
```

通过遍历state.subs, 可以找到所有”订阅该state变化的effect“。通过遍历effect.deps,可以找到所有”该effect依赖的state.subs“。完整实现如下：

```js
  function useEffect(callback) {
    const execute = () => {
      // 重置依赖
      cleanup(effect);
      // 将当前effect推入栈顶。
      effectStack.push(effect);
    }

    try {
      // 执行回调
      callback()
    } finally {
      // effect出栈
      effectStack.pop()
    }

    const effect = {
      execute,
      deps: new Set()
    }

    // 立即执行一次，建立订阅发布关系
    execute()
  }
```