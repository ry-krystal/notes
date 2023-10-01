#### commit 阶段

Render 工作的阶段被称为 commit 阶段。在 commit 阶段，会将各种副作用(flags 表示) commit(提交)到宿主环境 UI 中，可以类比“在使用 Git 管理代码时，编写完毕后 commit 代码”这一过程。

render 阶段流程可能被打断，而 commit 阶段一旦开始就会同步执行直到完成。整个阶段可以分为三个子阶段：

- BeforeMutation 阶段
- Mutatiion 阶段
- Layout 阶段

![commit流程](../images/commit阶段流程.drawio.svg "commit阶段流程图")
