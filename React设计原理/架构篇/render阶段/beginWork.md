#### beginWork

首选判断当前流程属于 mounted 还是 updated，判断依据为”Current fiberNode“ 是否存在：

```js
if (current !== null) {
  // update流程
}
```

如果当前流程是 update 流程，则 wip fiberNode 存在对应的 Current fiberNode。如果本次更新不影响 fiberNode.child，则可以复用对应的 Current fiberNode,这是一条 render 阶段的优化路径。

!['xxx'](../images/beginWork工作流程.drawio.svg "beginWork流程图")

如果无法复用 Current fiberNode, 则 mount 与 update 的流程大体一致，包括：
(1) 根据 wip.tag 进入”不同类型元素的处理分支“
(2) 使用 reconcile 算法生成下一级 fiberNode
两个流程的区别在于”最终是否会为 '生成的子 fiberNode' 标记 ’副作用 flags‘“。

beginWork 方法的结构如下：

```js
function beginWork(current, workInProgress, renderLanes) {
  if (current !== null) {
    // 可复用
  } else {
    // 不可复用
  }

  // 根据tag的不同，进入不同的处理逻辑
  swtich(workInProgress.tag) {
    // 省略
    case InterminateComponent;
    case LazyComponent;
    case FunctionComponent;
    case ClassComponent;
    case HostRoot;
    case HostComponent;
    case HostText;
    // ...省略代码
  }
}
```

不同类型的 fiberNode 会进入不同的处理逻辑，其中：

- HostComponent 代表原生 Element 类型(比如 DIV、SPAN)
- IndeterminateComponent 是 FC mount 时进入的分支，update 时则进入 FunctionComponent 分支
- HostText 代表文本元素类型

如果常见类型(FunctionComponent、ClassComponent、HostComponent)没有命中优化策略，它们最终会进入 reconcileChildren 方法。从该函数名可以发现这是 Reconcile 模板核心的部分，代码如下：

```js
export function reconcileChildren(
  current,
  workInProgress,
  nextChildren,
  renderLanes
) {
  if (current === null) {
    // mount时
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes
    );
  } else {
    // update时
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes
    );
  }
}
```

mountChildFibers 与 reconcileChildFibers 方法都是 ChildReconciler 方法的返回值：

```js
// 两者的区别只是传参不同
const reconcileChildrenFibers = ChildReconciler(true);
const mountChildFibers = ChildReconciler(false);

function ChildReconciler(shouldTrackSideEffects) {
  //...
}
```

传参 shouldTrackSideEffect 代表”是否追踪副作用“，即是否标记”flag“。在执行 mountChildFibers 时，以其内部的 placeChild 方法举例：

```js
function placeChild(newFiber, lastPlacedIndex, newIndex) {
  newFiber.index = newIndex;

  if (!shouldTrackSideEffects) {
    // 不标记Placement
    newFiber.flags |= Forked;
    return lastPlacedIndex;
  }

  // ...

  // 标记Placement
  newFiber.flags |= Placement;
}
```

由于 shouldTrackSideEffects 为 false,即使该 fiberNode 执行了 placeChild, 也不会被标记 Placement,因此在 Renderer 中不会对”该 fiberNode 对应 DOM 元素“执行”插入页面“的操作。reconcileChildFibers 中会标记 flags 主要与元素位置相关，包括：

- 标记 ChildDeletion, 代表删除操作。
- 标记 Placement, 代表插入或者移动操作。
