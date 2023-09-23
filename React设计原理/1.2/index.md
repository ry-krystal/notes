#### 编程：细粒度更新

首先，实现 useState,用来定义自变量：

```js
function useState(value) {
  const getter = () => value;
  const setter = (newValue) => (value = newValue);

  return [getter, setter];
}
```

useState 接收初始值 value 为参数，形成闭包。调用 getter 取值会返回闭包中的 value, 调用 setter 赋值会修改闭包中的 value。与 React 不同，返回数组[0]并不是 value，而是 getter。这样做的意义是什么呢？

使用方式如下：

```js
const [count, setCount] = useState(0);

console.log(count()); // 0
setCount(1);
console.log(count()); // 1
```

接下来实现”有副作用因变量“ --- useEffect，期望的行为是：

1. useEffect 执行后，回调函数立即执行；
2. 依赖的自变量变化后，回调函数立即执行；
3. 不需要显示指明依赖；

```js
const [count, setCount] = useState(0);

// effect1
useEffect(() => {
  console.log("count is:", count()); // count is: 0
});

// effect2
useEffect(() => {
  console.log("没有什么事儿"); // 没有什么事儿
});

setCount(2); // count is: 2
```

实现的关键在于建立 useState 与 useEffect 的订阅发布关系：

1. 在 useEffect 回调中执行 useState 的 getter 时，该 effect 会订阅”该 state 的变化“。
2. useState 的 setter 在执行时，会向所有”订阅了该 state 变化“的 effect 发布通知。

state 内部的集合 subs 用来保存”订阅该 state 变化的 effect“。effect 是每个 useEffect 对应的数据结构：

```js
const effect = {
  // 用于执行useEffect回调函数
  execute,
  // 保存该useEffect依赖的state对应subs的集合
  dep: new Set(),
};
```

通过遍历 state.subs, 可以找到所有”订阅该 state 变化的 effect“。通过遍历 effect.deps,可以找到所有”该 effect 依赖的 state.subs“。完整实现如下：

```js
function useEffect(callback) {
  const execute = () => {
    // 重置依赖
    cleanup(effect);
    // 将当前effect推入栈顶。
    effectStack.push(effect);
    try {
      // 执行回调
      callback();
    } finally {
      // effect出栈
      effectStack.pop();
    }
  };

  const effect = {
    execute,
    deps: new Set(),
  };
  // 立即执行一次，建立订阅发布关系
  execute();
}
```

cleanup 的实现：

```js
function cleanup(effect) {
  // 从该effect订阅的所有state对应subs中移除该effect
  for (const subs of effect.deps) {
    subs.delete(effect);
  }
  // 将该effect依赖的所有state对应subs移除
  effect.deps.clear();
}
```

```js
function useState(value) {
  // 保存订阅该state变化的effect
  const subs = new Set();

  const getter = () => {
    // 获取当前上下文的effect
    const effect = effectStack[effectStack.length - 1];
    if (effect) {
      // 建立订阅发布关系
      subscribe(effect, subs);
    }
    return value;
  };

  const setter = (nextValue) => {
    value = nextValue;

    // 通知所有订阅该state变化的effect执行
    for (const effect of [...subs]) {
      effect.execute();
    }
  };

  return [getter, setter];
}
```

```js
function subscribe(effect, subs) {
  // 订阅关系建立
  subs.add(effect);
  // 依赖关系建立
  effect.deps.add(subs);
}
```

实现 useMemo:

```js
function useMemo(callback) {
  const [s, set] = useState();
  // 首次执行callback，建立回调中state的订阅发布关系
  useEffect(() => set(callback()));
  return s;
}
```
