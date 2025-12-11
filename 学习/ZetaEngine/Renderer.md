> [(99+ 封私信 / 80 条消息) Render Graph 全网最细介绍（一） - 知乎](https://zhuanlan.zhihu.com/p/582098945)

## 基本知识
### GPU中的渲染管线
![[Pasted image 20251204191309.png]]
### pass

我们首先来看看GPU中的渲染管线（采用OpenGL中的命名），我们将GPU的一次完整的执行流程叫做一个**pass。**
### resource

**我们能够在GPU执行一次之后，就将整个游戏场景给渲染出来吗？答案显然是不可能的**，那么这些渲染结果被存储在哪里呢？我们在学习OpenGL的时候，会学习到一个概念叫做FrameBuffer的缓冲，我们通常将GPU的渲染结果存储在各种Buffer中。

我们将搭载GPU的输出信息的Buffer，称作**Resource**（这是一种狭隘的理解，但是RenderGraph中，Resource大部分都是指的是这些Buffer）

## RenderGraph的目标

我们希望RenderGraph能够实现以下几点目标：

1. 解耦合
2. 自动的资源管理
3. 简化的多线程渲染
4. 良好的可扩展性
5. 容易debug

## 什么是RenderGraph

从数据结构上来讲，它是一个有向无环图。它长这样：
![[Pasted image 20251204191511.png]]
RenderGraph的图中，主要有两种节点：**1.pass节点**。**2.resource节点**。

在上图中：

- 橙色边框：pass节点
- 蓝色边框：resource节点
- 红色箭头：pass节点指向resource节点，表示该pass对该resource进行写操作。
- 绿色节点：resource节点指向pass节点，表示该resource被该pass读取。

### pass节点

我们可以狭义地将pass理解为一个函数，它可以接受一个或多个输入，并产生一个或多个输出。这里的输入与输出只能是resource节点。pass的在拿到resource之后，调用自己内部的执行函数，执行函数一般是调用一次或若干次GPU，产生一个或多个结果，并将这些结果输出。

### resource节点

它就是资源节点，每一个资源都有它自己的状态（clear or discard/undefined）。

### **再次理解RenderGraph**

在理解RenderGraph的时候，需要注意以下几点：

- RenderGraph不是一个抽象图、概念图，它是实实在在的一种数据结构，它的每一部分都是能够找到它的实现代码的。
- 各个pass之间是**完全独立**的存在，pass之间**不存在相互调用**，每一个pass就根据自己的输入调用GPU，产生若干输出。
- resource节点就是各个pass之间的纽带，但是pass并不关心自己的输入到底是哪一个pass产生的。

这样的抽象，很好解决了以前渲染流程存在的强耦合的问题。

## Render Graph是怎样实现的

RenderGraph是一个对于渲染流程的一个高层次抽象，它实现了对整帧信息的整体把握，并对整帧的渲染进行整体优化。

在理解RenderGraph的时候，需要注意以下几点：
- RenderGraph不是一个抽象图、概念图，它是实实在在的一种数据结构，它的每一部分都是能够找到它的实现代码的。
- 各个pass之间是**完全独立**的存在，pass之间**不存在相互调用**，每一个pass就根据自己的输入调用GPU，产生若干输出。
- resource节点就是各个pass之间的纽带，但是pass并不关心自己的输入到底是哪一个pass产生的。

RenderGraph的三个阶段：
- Setup
- Compile
- Execute
### Setup 阶段

**RenderGraph需要掌握整帧的所有信息。**

pass需要声明：1. 需要操作哪些resource。 2. 要对resource进行哪种操作（read，write，create）。

**pass在执行之前，都是需要向RenderGraph进行注册，提前进行声明，pass不能读或写没有声明的resource。**

setup阶段主要干了以下几件事：
1. pass声明注册自己用到的资源，需要确定资源的状态
2. 未实际分配GPU资源
3. 将持久资源导入内存

### Compile 阶段

在Compile阶段，RenderGraph中未被引用的resource 和 pass将会被裁减掉，只留下真正执行的部分，相当于是做一次优化。

同时，Compile阶段会对resource的声明周期进行计算，确定resource应该在何时分配资源、何时被回收。

### Execute 阶段

在Execute阶段，各个pass开始执行自己的Execute函数（在这个时候才会真正用到GPU）。Execute阶段最大的特点就是，它真正的需要GPU为它分配资源了。

