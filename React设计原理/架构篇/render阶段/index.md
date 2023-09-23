#### render 阶段

Reconciler 工作的阶段在 React 内部被称为 render 阶段，ClassComponent 的 render 函数、Function Component 函数本身都在该阶段被调用。

根据 Scheduler 调用的结果不同，render 阶段可能开始于 performSyncWorkOnRoot(同步更新流程)或 performConcurrentWorkOnRoot(并发更新流程)方法。

```js
// performSyncWorkOnRoot 会执行该方法
function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// performConcurrentWorkOnRoot 会执行该方法
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

workInProgress 变量(简称‘wip’)代表“生成 Fiber Tree 工作已经进行到的 wip fiberNode”。performUnitOfWork 方法会创建一个 fiberNode 并赋值给 wip，并将 wip 与已创建的 fiberNode 连接起来构成 Fiber Tree。wip === null 代表“Fiber Tree 的构建工作结束”。上述两个方法的唯一区别是“是否调用 shouldYield(是否可中断)”。

Fiber Reconciler 是从 Stack Reconciler 重构而来，通过遍历的方式实现可中断的递归，因此 performUnitOfWork 的工作可以分为两部分：‘递’和'归'。

'递'阶段会从 HostRootFiber 开始向下以 DFS 的方式遍历，为‘遍历到的每个 fiberNode’ 执行 beginWork 方法。该方法会根据传入的 fiberNode 创建下一级 fiberNode,有以下两种情况。
（1）下一级只有一个元素，这时 beginWork 方法会创建子 fiberNode，并与 wip 连接。考虑到如下 JSX，如果 wip 为“UL 对应 fiberNode”,则会创建“LI 对应 fiberNode”, 同时两者产生连接：

```js
// jsx情况
<ul>
  <li></li>
</ul>;

// 子fiberNode与父fiberNode通过return连接
LIFiber.return = ULFiber;
```

(2) 下一级只有一个元素，这时 beginWork 方法会创建所有子 fiberNode 并连接在一起，为首的子 fiberNode 会与 wip 连接。

```js
// jsx情况
```
