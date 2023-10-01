#### reactdom-renderer

react-reconciler 是独立的包。使用这个包构建一个简易的 ReactDOM Renderer。了解其原理后，可以在“任何可以绘制 UI 的环境”中利用 react-reconciler 实现该环境下的 React。

```js
  // 创建一个新项目
  npx create-react-app reactdom-app
```

新建 customRender.js, 引入 react-reconciler 并完成初始化。其中 hostConfig 是宿主环境的配置项，后续我们将完善配置项：

```js
import ReactReconciler from "react-reconciler";
// 宿主环境的配置项
const hostConfig = {};
// 初始化ReactReconciler
const ReactReconcilerInst = ReactReconciler(hostConfig);
```

执行 customRenderer.js 导出一个“包含 render 方法的对象”。

```js
export default {
  // render 方法
  render: (reactElement, domElement, callback) => {
    // 创建根节点
    if (!domElement._rootContainer) {
      domElement._rootContainer = ReactReconcilerInst.createContainer(
        domElement,
        false
      );
    }

    // 更新容器
    return ReactReconcilerInst.updateContainer(
      reactElement,
      domElement._rootContainer,
      null,
      callback
    );
  },
};
```

在项目入口文件中，将 ReactDOM 替换成 CustomRenderer:

```js
import ReactDOM from "react-dom";
import CustomRenderer from "./customRenderer";

// 替换ReactDOM
CustomRenderer.render(<App />, document.getElementById("root"));
```

然后，实现 hostConfig 配置，填充空函数，避免应用因缺失必要函数而报错：

```js
const hostConfig = {
  supportsMutation: true,
  getRootHostContext() {},
  getChildHostContext() {},
  prepareForCommit() {},
  resetAfterCommit() {},
  shouldSetTextContent() {},
  createInstance() {},
  createTextInstance() {},
  appendInitialChildren() {},
  finalizeInitialChildren() {},
  clearContainer() {},
  appendChild(),
  appendChildToContainer() {},
  prepareUpdate() {},
  commitUpdate() {},
  commitTextUpdate() {},
  removeChild() {}
};
```

注意唯一的 Boolean 类型的配置项 supportsMutation 表示“宿主环境的 API 是否支持 Mutation”。DOM API 的工作方式属于 Mutation,比如 element.appendChild、 element.removeChild。Native 环境的 API 则不支持这种工作方式。

这里将 API 分为四类：

- 初始化环境信息
  比如 getHostContext、getChildHostContext，用于初始化上下文信息。
- 创建 DOM Tree
  比如 createInstance 用于创建 DOM 元素。createTextInstacne 用于创建 DOM TextNode，其实现如下：

  ```js
  const createTextInstance = (text) => {
    return docuemnt.createTextNode(text);
  };
  ```

- 关键逻辑的判断
  shouldSetTextContent 用于判断”组件的 children 是否为文本节点“，实现如下：

  ```js
  const shouldSetTextContent = (_, props) => {
    return (
      typeof props.children === "string" || typeof props.children === "number"
    );
  };
  ```

  - DOM 操作
    appendInitialChild 用于”插入 DOM 元素“，对应 Placement flag,实现如下：

  ```js
  const appendInitialChild = (parent, child) => {
    parent.appendChild(child);
  };
  ```

  removeChild 用于”删除子 DOM 元素“，对应 ChildDeletion flag, 实现如下：

  ```js
  const removeChild = (parentInstance, child) => {
    parentInstance.removeChild(child);
  };
  ```

  当实现所有 API 后，页面即可正常渲染。customRenderer.js 的完成实现如下：

  ```js
  const createInstance = (
    type,
    newProps,
    rootContainerInstacne,
    _currentHostText,
    workInProgress
  ) => {
    const domElement = document.createElement(type);
    Object.keys(newProps).forEach((propName) => {
      const propValue = newProps[propName];
      if (proName === "children") {
        if (typeof propValue === "string" || typeof propValue === "number") {
          domElement.textContext = propValue;
        }
      } else if (propName === "onClick") {
        domElement.addEventListener("click", propValue);
      } else if (propName === "className") {
        domElement.setAttribute("class", propValue);
      } else {
        const propValue = newProps[propName];
        domElement.setAttribute(propName, propValue);
      }
    });

    return domElement;
  };

  const commitUpdate = (
    domElement,
    updatePayload,
    type,
    oldProps,
    newProps
  ) => {
    Object.keys(newProps).forEach((propName) => {
      const propValue = newProps[propName];
      if (proName === "children") {
        if (typeof propValue === "string" || typeof propValue === "number") {
          domElement.textContext = propValue;
        }
      } else {
        const propValue = newProps[propName];
        domElement.setAttribute(propName, propValue);
      }
    });
  };

  const commitTextUpdate = (textInstance, oldText, newText) => {
    textInstance.text = newText;
  };
  ```
