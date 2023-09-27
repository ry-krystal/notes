#### completeWork

与 beginWork 类似，completeWork 也会根据 wip.tag 区分对待，流程大体包括两个步骤：

1. 创建或者标记元素更新；
2. flags 冒泡；

beginWork 的 reconcileChildFibers 方法用来”标记 fiberNode 的插入、删除、移动“。completeWork 方法的步骤 1 会完成”更新的标记“。当步骤 1 完成后，该 fiberNode 在本次更新中的增、删、改操作均已标记完成。至此，Reconciler 中”标记 flags“相关工作基本完成。但是距离在 Renderer 中”解析 flags“还有一项很重要的工作要做，这就是 flags 冒泡。

3.4.1 flags 冒泡
当更新流程经过 Reconciler 后，会得到一棵 Wip Fiber Tree, 其中部分 fiberNode 被标记 flags。Renderer 需要对”被标记的 fiberNode 对应的 DOM 元素执行 flags 对应的 DOM 操作。“

completeWork 属于”归“阶段，从叶子元素开始，整体流程是”自下而上“的。fiberNode.subTreeFlags 记录了该 fiberNode 的”所有子孙 fiberNode 上被标记的 flags“。每个 fiberNode 经由如下操作，可以将子孙 fiberNode 中”标记 flags“向上冒泡一层。

```js
let subtreeFlags = NoFlags;

// 收集子fiberNode的子孙fiberNode中标记的flags
subTreeFlags |= child.subTreeFlags;
// 收集子fiberNode标记的flags
subTreeFlags |= child.flags;
// 附加在当前fiberNode的subtreeFlags上
completeWork.subtreeFlags |= subTreeFlags;
```

当 HostRootFiber 完成 completeWork，整棵 Wip Fiber Tree 中所有”被标记的 flags“都在 HostRootFiber.subtreeFlags 中定义。在 Renderer 中，通过任意一级 fiberNode.subtreeFlags 都可以快速确定”该 fiberNode 所在子树是否存在副作用需要执行“。

3.4.2 mount 概览

completeWork 流程如下：

!['completeWork'](../images/completeWork流程图.drawio.svg "completeWork流程图")

与 beginWork 相同，completeWork 通过 current !== null 判断是否处于 update 流程，在 mount 流程中，其首先通过 createInstance 方法创建”fiberNode 对应的 DOM 元素“：

```js
function createInstance(
  type,
  props,
  rootContainerInstance,
  hostContext,
  intervalInstanceHandle
) {
  // 省略代码
  if (
    typeof props.children === "string" ||
    typeof props.children === "number"
  ) {
    // 省略chidlren是String、Number时的处理逻辑
  }

  // 创建dom element
  const domElement = createElement(
    type,
    props,
    rootContainerInstance,
    parentNameSpace
  );

  // ...

  return domElement;
}
```

然后执行 appendAllChildren 方法，将下一层 DOM 元素插入“createInstance 方法创建的 DOM 元素中”，具体逻辑为：

1. 从当前 fiberNode 向下遍历，将遍历到的第一层 DOM 元素类型(HostComponent、HostText)通过 appendChild 方法插入 parent 末尾。
2. 对兄弟 fiberNode 执行步骤 1
3. 如果没有兄弟 fiberNode，则对父 fiberNode 的兄弟执行步骤 1
4. 当遍历流程回到最初执行步骤 1 所在层或者 parent 所在层时终止。

相关代码如下：

```js
function appendAllChildren(parent, workInProgress /* 省略相关参数 */) {
  let node = workInProgress.child;

  while (node !== null) {
    // 步骤1， 对第一层dom元素执行appendChild
    if (node.tag === HostComponent || node.tag === HostText) {
      // 对HostComponent、HostText执行appendChild
      appendInitialChild(parent, node.stateNode);
    } else if (node.child !== null) {
      // 继续向下遍历，直到找到第一层dom元素类型
      node.child.return = node;
      node = node.child;
      continue;
    }

    // 终止情况1：遍历到parent对应fiberNode
    if (node === workInProgress) {
      return;
    }

    // 如果没有兄弟fiberNode, 则向父fiberNode遍历
    while (node.sibling === null) {
      if (node.return === null || node.return === workInProgress) {
        return;
      }
      node = node.return;
    }

    // 对兄弟fiberNode执行步骤1
    node.sibling.return = node.return;
    node = node.sibling;
  }
}
```

可以发现，appendAllChildren 方法与 flags 冒泡类似，都是”处理某个元素的下一级元素“。flags 冒泡处理的是”下一级 flags“，appendAllChildren 处理的是”下一级 DOM 元素类型“。appendAllChildren 方法中存在复杂的遍历过程，是因为 fiberNode 的层级与 DOM 元素的层级可能不是一一对应的。举例说明：

```js
function World() {
  return <span>World</span>;
}
```

使用 World 组件场景如下：

```html
<div>
  Hello
  <World />
</div>
```

从 fiberNode 的角度看，”hello“ fiberNode 与 World fibeNode 同级。

但是从 DOM 元素的角度看，”Hello“ TextNode 与 HTMLSpanElement 同级。所以从 fiberNode 中查找同级的 DOM 元素，可能需要跨 fiberNode 层级查找。

completeWork 流程接下来会执行 finalizeInitialChildren 方法完成属性的初始化，包括如下几类属性：

styles, 对应 setValueForStyles 方法；
innerHTML, 对应 setInnerHTML 方法
文本类型 children, 对应 setTextContent 方法
不会在 DOM 中冒泡的事件，包括 cancel、close、invalid、load、scroll、toggle,
对应 listenToNonDelegatedEvent 方法； 其他属性，对应 setValueForProperty 方法。

最后，执行 bubbleProperties 方法将 flags 冒泡。completeWork 在 mount 时的流程总结如下：

```js
  1. 根据wip.tag进入不同处理分支
  2. 根据current !== null 区分mount和update流程
  3. 对于HostComponent，首先执行createInstance方法创建对应的DOM元素
  4. 执行appendAllChildren将下一级DOM元素挂载在步骤3创建的DOM元素下；
  5. 执行finalizeInitialChildren完成属性初始化
  6. 执行bubbleProperties完成flags冒泡
```

由于 appendAllChildren 方法的存在，当 completeWork 执行到 HostRootFiber 时，已经形成一棵完成的离屏 DOM Tree。mount 时构建 Wip Fiber Tree, 并不是所有 fiberNode 都不存在 alternate。由于 HostRootFiber 存在 alternate(即 HostRootFiber.current !== null), 因此 HostRootFiber 在 beginWork 时会进入 reconcileChildFibers 而不是 mountChildFibers, 它的子 fiberNode 会被标记 Placement。

Renderer 发现本次更新流程中存在一个 Placement flags,执行一次 parentNode.appendChild(或 parentNode.insertBefore)，即可将已经构建好的离屏 DOM Tree 插入页面。试想，如果 mountChildFibers 也会标记 flags,那么对于”首屏渲染“这样初始化场景，就需要执行大量 parentNode.appendChild(或 parentNode.insertBefore)方法。相比之下，离屏构建后再执行一次插入的性能更佳。

3.4.3 update 概览

既然 mount 流程完成了属性的初始化，那么 update 流程显然将完成标记属性的更新。updateHostComponent 的主要逻辑在 diffProperties 方法中，该方法包括两次遍历：

- 第一次遍历，标记删除”更新前有，更新后没有“的属性
- 第二次遍历，标记更新”update 流程前后发生改变“的属性

相关代码如下：

```js
function diffProperties(
  domElement,
  tag,
  lastRawProps,
  nextRawProps,
  rootContainerElement
) {
  // 保存变化属性的key、value
  let updatePayload = null;
  // 更新前的属性
  let lastProps;
  // 更新后的属性
  let nextProps;

  // ...

  // 标记删除”更新前有，更新后没有“的属性
  for (propKey in lastProps) {
    if (
      nextProps.hasOwnProperty(propKey) ||
      !lastProps.hasOwnProperty(propKey) ||
      lastProps[propKey] == null
    ) {
      continue;
    }

    // 处理style
    if (propKey === STYLE) {
      // ...
    } else {
      // 其他属性
      (updatePayload = updatePayload || []).push(proKey, null);
    }
  }
  // 标记更新”update流程前后发生变化“的属性
    for (propKey in nextProps) {
      let nextProp = nextProps[propKey];
      let lastProp = lastProps != null ? lastProps[propKey] : undefined;

      if (
        nextProps.hasOwnProperty(propKey) ||
        nextProps === lastProps ||
        (nextProp == null && lastProps == null)
      ) {
        continue;
      }

      if (propKey === STYLE) {
        // 处理style
      } else if (propKey === DANGEROUSLY_SET_INNER_HTML) {
        // 处理单一文本类型的children
      } else if (propKey === CHILDREN) {
        // 处理单一文本类型的children
      } else if (registrationNameDependencies.hasOwnProperty(proKey)) {
        if (nextProp != null) {
          // 处理onScroll事件
        } else {
          // 处理其他属性
        }
      }
    return updatePayload;
}
```

所有变化属性的 key、value 会保存在 fiberNode.updateQueue 中。同时，该 fiberNode 会标记 Update:

```js
workInProgress.flags |= Update;
```

在 fiberNode.updateQueue 中，数据以 key、value 作为数组的相邻两项。举例说明，点击 div 元素触发更新，此时 style、title 属性发生变化：

```js
export default () => {
  const [num, updateNum] = useState(0);
  return (
    <div
      onClick={() => updateNum(num + 1)}
      style={{ color: `#${num}${num}${num}` }}
      title={num + ""}
    ></div>
  );
};
```

此时 fiberNode.updateQueue 保存的数据如下，代表”title 属性变为‘1’“, "style 属性中的 color 变为‘#111’"：

```js
["title", "1", "style", { color: "#111" }];
```
