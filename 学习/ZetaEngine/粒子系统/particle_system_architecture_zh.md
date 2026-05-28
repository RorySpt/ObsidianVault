# 粒子系统架构设计

## 背景与目标

ZetaEngine 当前的粒子系统并不是照搬 Niagara 的脚本 VM，也不是传统“固定字段 + 固定更新器”的单一路线，而是一个更偏运行时组合式的系统：

- 资产侧用 `ParticleSystem` / `ParticleEmitter` 描述系统与发射器
- 运行时用模块栈驱动不同阶段
- 粒子数据按模块需求动态拼装
- 渲染和材质输入通过桥接层接入引擎现有渲染管线

这套设计的核心目标有四个：

- 让粒子能力以“模块”而不是硬编码逻辑块的形式扩展
- 让粒子数据布局按需生成，而不是永远携带一大套固定字段
- 让系统、发射器、粒子三个层级各自拥有清晰职责
- 让粒子运行时和渲染/材质系统解耦，只通过稳定的桥接数据交互

当前实现已经支持一批典型 Sprite 粒子效果，但它的真正价值不只是“能出火焰和烟雾”，而是已经具备了继续扩展成更完整特效框架的基本骨架。

## 总体分层

当前源码目录已经按职责整理为三块主域：

```text
source/runtime/asset/particle/
  core/
  module/
  value/
```

对应的架构分层如下：

```text
ParticleSystemComponent / World / Scene
                ↓
        ParticleSystem / ParticleEmitter
                ↓
     ModuleStack + ModuleBase 派生模块
                ↓
  ParticlePool + ParticleContext + 数据组件
                ↓
     Render / Material / Debug bridge
```

其中：

- `core` 是粒子运行时骨架
- `module` 是行为实现层
- `value` 是参数和值表达式层
- `component`、`render`、`material` 下的粒子代码属于桥接层，不是粒子核心本体

这个边界很重要。粒子核心负责“模拟和状态”，桥接层负责“挂载、提交绘制、接材质输入”。

## 1. Core 层

`core` 目录是当前粒子系统真正的内核。

### 1.1 `ParticleSystem`

`ParticleSystem` 是系统级容器，职责包括：

- 持有多个 `ParticleEmitter`
- 持有系统级 `spawn_modules` 和 `update_modules`
- 维护 `SystemContext`
- 负责 `play / replay / pause / stop / tick`
- 负责在播放前 `compile()` 全部发射器

它解决的是“整套粒子系统如何被驱动”的问题，而不是“单个粒子怎么更新”。

### 1.2 `ParticleEmitter`

`ParticleEmitter` 是粒子运行时的核心执行单元，职责包括：

- 持有发射器配置和运行时状态
- 持有 `ParticlePool`
- 持有五组模块栈
  - `Emitter Spawn`
  - `Emitter Update`
  - `Particle Spawn`
  - `Particle Update`
  - `Particle Render`
- 维护包围盒与可见性
- 管理粒子创建、销毁、更新和渲染阶段触发

可以把它理解成“一个局部可执行粒子图”。

### 1.3 Context 体系

当前上下文分四层：

- `SystemContext`
- `EmitterContext`
- `ParticleContext`
- `ModuleContext`

这套设计的作用是把不同层级的数据访问收束到显式上下文里，而不是让模块随意回溯全局状态。

设计收益：

- 模块执行接口统一为 `execute(ModuleContext&)`
- 系统级模块、发射器级模块、粒子级模块可以共享调用模型
- 上下文可扩展，后续加黑板、调试信息、事件流时不需要改模块接口

### 1.4 `ParticlePool`

`ParticlePool` 是粒子数据存储层。它不是固定 struct 数组，而是按组件布局动态组织。

这意味着：

- 是否存在某个粒子字段，不由“引擎默认定义”决定
- 而由启用模块在 `collect_particle_components()` 阶段声明决定

这条设计是整个系统最关键的抽象之一。它让粒子数据从“静态大对象”变成了“按行为需求拼装的最小集合”。

### 1.5 `ModuleBase` 与 `ModuleStack`

`ModuleBase` 解决两个问题：

- 模块属于哪个执行阶段
- 模块如何暴露统一生命周期

`ModuleStack` 解决两个问题：

- 某一阶段如何有序持有多个模块
- 某一阶段如何统一执行、重置和管理模块对象

当前 `EModuleUsage` 明确区分了：

- `SystemSpawn`
- `SystemUpdate`
- `EmitterSpawn`
- `EmitterUpdate`
- `ParticleSpawn`
- `ParticleUpdate`
- `ParticleRender`

也就是说，这个系统不是“把所有模块都塞进一条链里”，而是从一开始就按执行阶段做了结构分离。

## 2. Module 层

`module` 目录按执行阶段拆成三组：

```text
module/
  system/
  emitter/
  particle/
```

这个拆法的意义不是为了目录整齐，而是为了让“阶段职责”成为代码组织的一等概念。

### 2.1 `module/system`

这一层是系统级模块，目前数量不多，主要承担：

- 系统级 spawn 占位
- 系统级循环策略承载

当前它更像“系统范围配置层”，而不是复杂逻辑层。

### 2.2 `module/emitter`

这一层决定发射器本身如何运行，典型职责包括：

- 生命周期管理
- 每帧生成数计算
- 读取其他发射器状态

这一层回答的是：“这一帧该不该出粒子，要出多少粒子，发射器处于什么状态”。

### 2.3 `module/particle`

这是当前最重要的一层，里面同时包含：

- 粒子生成模块
- 粒子更新模块
- 粒子渲染配置模块
- 生成/更新共享模块

它们共同定义：

- 粒子出生时写入什么状态
- 粒子每帧如何演化
- 粒子最终如何被渲染系统解释

当前 `particle_common_modules.*` 放在这里，表示它并不是“公共工具库”，而是“跨 Spawn / Update 两个阶段复用的粒子模块”。

## 3. Value 层

`value` 层负责表达“模块参数来自哪里”。

当前主要类型有：

- `ParticleScalar`
- `ParticleVec2`
- `ParticleVec3`
- `CurveData`

这一层的价值在于，它把“模块逻辑”和“参数来源”分离了。

例如一个模块想要缩放尺寸，它不需要关心输入究竟来自：

- 常量
- 随机区间
- 曲线
- 运行时绑定

模块只消费统一的值接口。

这比把所有模块都写成“大量 if/else + 参数模式分支”要更健康，后续新增绑定源时也更容易收口。

## 4. 运行时数据流

从运行时角度看，当前粒子系统的数据流可以概括成：

```text
Component 挂载 ParticleSystem
    ↓
ParticleSystem::play()
    ↓
compile() 收集组件需求并重建粒子池布局
    ↓
tick(delta_time)
    ↓
System Update
    ↓
Emitter Update
    ↓
Particle Spawn
    ↓
Particle Update
    ↓
Particle Render
    ↓
Scene 收集 ParticleRenderData
    ↓
Forward / Deferred Pass 绘制
```

这里有两个设计点值得单独强调。

### 4.1 先编译布局，再执行运行时

`compile()` 不是附属步骤，而是运行前的布局编译阶段。

模块通过 `collect_particle_components()` 声明粒子组件需求，发射器再据此重建 `ParticlePool`。这保证了运行时访问的组件集合和启用模块是一致的。

换句话说：

- 模块决定数据布局
- 不是数据布局反过来限制模块

### 4.2 更新和渲染显式分离

当前 `Sprite` 模块本身不直接发 draw call，它更像一个“渲染配置模块”。真正把粒子转成渲染数据的是 `Scene::update_particle_render_data(...)`。

这意味着：

- 粒子模拟阶段只写状态
- 渲染桥接阶段读取状态并组装 `ParticleInstanceData`
- 真正绘制仍由现有渲染 pass 完成

这种拆分让粒子系统不需要自己维护一套独立渲染器。

## 5. 粒子数据模型

当前常用粒子组件有：

- `ParticleLifetimeState`
- `ParticleTransformState`
- `ParticleMotionState`
- `ParticleVisualState`
- `ParticleGroundState`
- `ParticleOrbitState`

这几类组件分别覆盖：

- 生命周期
- 真实位置
- 力和速度
- 渲染可视状态
- 特殊行为缓存

这里存在一个很有价值的设计取舍：

- `transform.position` 表达的是粒子模拟位置
- `visual.render_position` 表达的是最终渲染位置

它允许像 `Orbit` 这样的模块只修改“渲染偏移”而不污染粒子真实模拟位置。这种“模拟状态”和“显示状态”分离的思路，为后续加入拖尾、面向相机修正、屏幕空间修正等效果留出了空间。

## 6. 桥接层设计

粒子核心本身并不直接依赖完整渲染语义，而是通过三个桥接点接入引擎。

### 6.1 组件桥接

`ParticleSystemComponent` 是游戏对象层入口，职责很单纯：

- 持有 `ParticleSystem`
- 在组件生命周期里驱动系统
- 对外暴露 `play / replay / pause / stop`

这让粒子系统可以像普通组件一样挂在实体上，而不需要粒子核心知道世界系统如何组织对象。

### 6.2 渲染桥接

`Scene::update_particle_render_data(...)` 和 `ParticleRenderData` 组成渲染桥接层。

这个桥接层负责：

- 找到发射器的 `Sprite` 渲染模块
- 读取材质
- 读取 `ParticleVisualState`
- 组装 `ParticleInstanceData`
- 按混合模式分组
- 把结果交给前向/延迟 pass

这样做的好处是：

- 粒子系统不直接知道具体 render pass 细节
- 现有渲染管线不需要理解粒子模拟逻辑
- 两边只通过一份稳定的实例数据结构耦合

### 6.3 材质输入桥接

`asset/material/particle/` 下的节点，例如：

- `ParticleColorNode`
- `ParticleSubUvNode`
- `DynamicParameterNode`
- `ParticlePositionWsNode`

负责把粒子可视状态映射为材质图可消费的输入。

这层的意义是：

- 粒子材质不需要知道粒子系统如何模拟
- 粒子系统也不需要知道材质图如何组织节点
- 两边通过约定好的语义输入对接

这实际上是“粒子运行时”和“材质系统”的 ABI。

## 7. 并行与执行模型

当前并行只发生在 `Particle Update` 阶段，而且是按连续线程安全模块段切分。

设计思路是：

- 模块自己声明 `is_thread_safe()`
- 发射器预先把连续线程安全模块整理成并行段
- 并行段内部对所有粒子执行 `par_unseq`
- 非线程安全模块保持串行

这套方案的优点是很务实：

- 不需要引入复杂任务图
- 不要求所有模块都线程安全
- 可以逐步把已有模块升级为并行友好

它不是最激进的方案，但很适合当前阶段。

## 8. 当前边界与取舍

从架构上看，当前粒子系统有几条明确取舍。

### 8.1 先做 Sprite 路线

当前系统最成熟的是 Sprite 粒子路径。它优先把“粒子模块化 + 材质输入稳定 + 渲染桥接打通”做扎实，而没有急着上 GPU 粒子或通用脚本执行层。

这是合理的，因为 Sprite 路径已经覆盖大量常见效果，同时能把整个架构最关键的抽象验证出来。

### 8.2 模块是 C++ 类型，不是脚本字节码

当前模块是反射对象和 C++ 类，而不是 Niagara 式脚本节点编译结果。

这带来的优点是：

- 行为直观
- 调试成本低
- 性能边界清晰

代价是：

- 表达力还不够开放
- 模块组合能力受限于已有类型

### 8.3 渲染是桥接，不是内嵌

当前架构刻意避免让粒子系统直接拥有整套渲染提交流程。这是一个好取舍，因为它保持了主渲染管线的一致性，但也意味着某些高级粒子特性需要通过桥接层继续扩展。

## 9. 为什么这次目录重构是必要的

在重构之前，粒子代码同时混着三种维度：

- 运行时骨架
- 模块实现
- 参数值表达

而且 `function/context`、`function/emitter`、`function/math` 这些目录名更像历史产物，不足以表达真实架构。

重构为 `core / module / value` 后，几个关键边界变清晰了：

- `core` 不再混入具体模块实现
- `module` 明确按执行阶段组织
- `value` 成为独立的参数表达层
- 外部系统看到的是“粒子核心 + 桥接层”关系，而不是一堆散目录

这让后续继续扩展时，决策会简单很多。一个新类该放哪，不再需要靠猜。

## 10. 后续演进建议

基于当前结构，比较自然的下一步演进方向有四个。

### 10.1 把桥接层进一步收口成显式子域

现在 `component`、`render`、`material` 侧的粒子代码还分散在各自系统里。后续可以考虑引入更明确的粒子桥接命名，例如：

- component bridge
- render bridge
- material bridge

这样架构文档和代码边界会完全一致。

### 10.2 增加事件和跨发射器通信的正式抽象

目前“从其他发射器读取”能力已经存在，但更像定制模块能力。后续如果要走 Niagara 风格，需要把事件流和数据集交换提升为更显式的系统能力。

### 10.3 为 GPU / Compute 路线预留新的执行后端

当前架构里最容易复用的是：

- 模块分阶段模型
- 参数和值表达层
- 材质输入语义层

未来如果新增 GPU 粒子，更合理的做法不是推翻现有结构，而是保留这些上层抽象，只替换部分执行和数据存储后端。

### 10.4 补齐编辑器与序列化工具链

当前核心运行时已经有不错骨架，但要真正走向稳定资产工作流，还需要：

- 更清晰的粒子资产编辑器
- 模块面板和调试视图
- 更稳定的材质输入可视化
- 更强的导入/复刻工具链

## 总结

当前粒子系统可以概括为一句话：

它是一个以 `ParticleSystem / ParticleEmitter / ModuleStack / ParticlePool` 为核心，以 `core / module / value` 为代码组织，以组件、渲染、材质桥接层接入引擎其余部分的组合式粒子运行时。

这套设计已经具备几个正确的长期特征：

- 运行时骨架和具体模块分离
- 粒子数据布局按需生成
- 模拟状态和渲染状态分离
- 渲染和材质通过桥接语义接入
- 并行执行以模块线程安全声明为基础渐进扩展

它距离“完整 Niagara 级系统”还有距离，但作为 ZetaEngine 当前阶段的粒子架构，这个方向是健康的，后续扩展也有明确着力点。

| 标准    | 年份   | 重要特性                                                                              |
| ----- | ---- | --------------------------------------------------------------------------------- |
| C++11 | 2011 | auto、lambda、智能指针、移动语义、范围 for、nullptr、constexpr、uniform initialization、std::thread |
| C++14 | 2014 | 泛型 lambda、变量模板、std::make_unique、二进制字面量、return type deduction                      |
| C++17 | 2017 | 结构化绑定、if constexpr、std::optional/variant、并行算法、std::string_view、std::filesystem    |
| C++20 | 2020 | concepts、ranges、coroutines、modules、<=>、consteval/constinit                        |
| C++23 | 2023 | std::expected、std::print、deducing this、std::stacktrace、#embed、更多 constexpr        |
| C++26 | 2026 | 静态反射（compile-time reflection）、Contracts（契约）、std::execution（异步并发框架）、内存安全改进         |
