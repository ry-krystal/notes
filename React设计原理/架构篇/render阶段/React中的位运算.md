#### React 中的位运算

基本的三种位运算

1. 按位与(&), 如果两个二进制操作数的每个 bit 都为 1，则结果为 1，否则为 0。
2. 按位或(|), 如果两个二进制操作数的每个 bit 都为 0，则结果为 0，否则为 1。
3. 按位非(~)，对于一个二进制操作数逐位进行取反操作(0、1 互换)。

位运算在”标记状态“中的应用

```js
  react源码内有多个上下文环境，在执行方法时经常需要判断”当前处于哪个上下文环境中“
  // 未处于React上下文
  const NoContext = 0b000;
  // 处于batchedUpdates上下文
  const BatchedContext = 0b001;
  // 处于render阶段
  const RenderContext = 0b0010;
  // 处于commit阶段
  const CommitContext = 0b0100;
```

当执行流程进入 render 阶段，会使用按位或标记进入对应上下文：

```js
let executionContext = NoContext;
executionContext |= 0b0010;
```

此时可以结合按位与和 NoContext 来判断”是否处在某一上下文中“：

```js
// 是否处在RenderContext上下文中，结果为true
(executionContext & RenderContext) !== NoContext;

// 是否处于CommitContext上下文中，结果为false
(executionContext & CommitContext) !== NoContext;
```

离开 RenderContext 上下文后，结合按位与、按位非移除标记：

```js
// 从当前上下文中移除RenderContext上下文
executionContext &= ~RenderContext;

// 是否处于在RenderContext上下文中，结果为false
(executionContext & RenderContext) !== NoContext;
```
