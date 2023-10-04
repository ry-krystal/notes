#### schedule 阶段

Schedule(调度器)为“Time Slice 分割出的一个个短宏任务”提供了执行的驱动力。

![并发更新的体系架构](../images/并发更新的体系架构.drawio.svg "并发更新的体系架构")

为了灵活地控制宏任务的执行时机，React 实现了一套基于**lane 模型的优先级算法，并基于这套算法实现了 Batched Update(批量更新)、任务打断/恢复机制等低级特性。** 这些特性不适合开发者直接控制，一般由 React 统一挂历。基于低级特性，React 实现了“面向开发者”的高级特性(或叫作并发特性)，比如 Concurrent Suspense、useTransition。

本章主要关注 Schedule、lane 模型、低级特性。
