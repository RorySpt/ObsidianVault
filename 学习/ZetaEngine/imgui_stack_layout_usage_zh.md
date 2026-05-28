# ImGui StackLayout 使用说明

本文档说明 ZetaEngine 当前集成的 ImGui `StackLayout` 能力如何使用，以及在现有编辑器代码中应如何落地。

## 背景

当前仓库中的 ImGui 已包含 PR `ocornut/imgui#846` 的核心接口：

- `ImGui::BeginHorizontal()`
- `ImGui::EndHorizontal()`
- `ImGui::BeginVertical()`
- `ImGui::EndVertical()`
- `ImGui::Spring()`
- `ImGui::SuspendLayout()`
- `ImGui::ResumeLayout()`

这些接口不是简单的容器封装，而是直接接管了布局期间 `ItemSize()` 的推进逻辑。也就是说，一旦进入 `BeginHorizontal()` 或 `BeginVertical()`，当前区域里的控件排布方式已经切换为 StackLayout。

## 基本规则

### 1. `BeginHorizontal/BeginVertical` 会切换布局系统

进入 StackLayout 之后，控件会按照布局方向自动推进：

- `BeginHorizontal()` 中，控件默认从左到右排布
- `BeginVertical()` 中，控件默认从上到下排布

因此，布局内部不要再把它当成传统 ImGui 的“普通作用域”。

### 2. `Spring()` 用于分配剩余空间

`Spring(weight, spacing)` 会在当前布局中插入一个弹性空白区域。

典型用法：

```cpp
ImGui::BeginHorizontal("Toolbar", ImVec2(ImGui::GetContentRegionAvail().x, 0.0f));
ImGui::Button("Left A");
ImGui::Button("Left B");
ImGui::Spring();
ImGui::Button("Right A");
ImGui::EndHorizontal();
```

效果：

- 左侧按钮保持靠左
- `Spring()` 吃掉中间剩余宽度
- 右侧按钮被推到右边

### 3. 根布局要给明确主轴尺寸

这是最容易踩坑的点。

对于根 `Horizontal` 布局，如果宽度传 `0`，该实现会把它视为“自动宽度”。此时 `Spring()` 的可分配空间可能为 `0`，表现就是：

- `Spring()` 看起来“没生效”
- 右侧控件没有被推开

推荐写法：

```cpp
ImGui::BeginHorizontal("Toolbar", ImVec2(ImGui::GetContentRegionAvail().x, 0.0f));
```

对于竖向布局，同理需要关注高度是否应该明确给出。

## 不要混用的写法

### 1. 不要在 StackLayout 内继续依赖 `SameLine()` 作为主布局手段

错误倾向：

```cpp
ImGui::BeginHorizontal("Toolbar", ImVec2(ImGui::GetContentRegionAvail().x, 0.0f));
ImGui::Button("A");
ImGui::SameLine();
ImGui::Button("B");
ImGui::EndHorizontal();
```

原因：

- `BeginHorizontal()` 已经接管横向推进
- `SameLine()` 属于旧布局体系
- 混用后 Metrics 树、光标推进和对齐行为会变得难以预测

结论：

- 新代码优先直接使用 StackLayout 自己的顺序排布
- 需要间距时优先依赖 `Style.ItemSpacing`
- 需要弹性留白时使用 `Spring()`

### 2. 不要把 `Group` 当成布局替代品

`BeginGroup()/EndGroup()` 只是把一批控件当作一个 item 处理，方便：

- 一起测量
- 一起 hover
- 一起被父布局当作单个块参与排布

它不是横向或纵向布局系统，不负责替代 `BeginHorizontal/BeginVertical`。

## 兼容旧代码的桥接方式

如果一段老 UI 已经大量依赖：

- `SameLine()`
- 手工 `SetCursorPosX()`
- 基于固定像素偏移的排布

但你又想把它放进新的 StackLayout 外层，那么应该使用：

- `SuspendLayout()`
- `ResumeLayout()`

示例：

```cpp
ImGui::BeginHorizontal("Toolbar", ImVec2(ImGui::GetContentRegionAvail().x, 0.0f));

ImGui::BeginGroup();
ImGui::SuspendLayout();
paint_legacy_left_toolbar();
ImGui::ResumeLayout();
ImGui::EndGroup();

ImGui::Spring();

ImGui::BeginGroup();
ImGui::SuspendLayout();
paint_legacy_right_toolbar();
ImGui::ResumeLayout();
ImGui::EndGroup();

ImGui::EndHorizontal();
```

含义：

- 外层仍由 StackLayout 决定“左块 / 弹性间距 / 右块”
- 内层老代码暂时退出 StackLayout，继续走原有 `SameLine()` 体系

这是迁移旧 UI 到 StackLayout 时最稳妥的过渡方案。

## 推荐模式

### 模式一：纯 StackLayout

适合新写界面。

```cpp
ImGui::BeginHorizontal("Row", ImVec2(ImGui::GetContentRegionAvail().x, 0.0f));
ImGui::Button("保存");
ImGui::Button("加载");
ImGui::Spring();
ImGui::Button("设置");
ImGui::EndHorizontal();
```

优点：

- 结构清晰
- Metrics 更容易读
- 不依赖手工偏移

### 模式二：外层 StackLayout，内层旧布局

适合改造已有工具栏。

```cpp
ImGui::BeginHorizontal("Toolbar", ImVec2(ImGui::GetContentRegionAvail().x, 0.0f));

ImGui::BeginGroup();
ImGui::SuspendLayout();
paint_left_legacy_widgets();
ImGui::ResumeLayout();
ImGui::EndGroup();

ImGui::Spring();

ImGui::BeginGroup();
ImGui::SuspendLayout();
paint_right_legacy_widgets();
ImGui::ResumeLayout();
ImGui::EndGroup();

ImGui::EndHorizontal();
```

优点：

- 改动小
- 风险低
- 适合渐进式迁移

### 模式三：分层嵌套 StackLayout

适合完全重写内部排版。

```cpp
ImGui::BeginHorizontal("Toolbar", ImVec2(ImGui::GetContentRegionAvail().x, 0.0f));

ImGui::BeginHorizontal("LeftCluster");
ImGui::Button("透视");
ImGui::Button("着色");
ImGui::Button("显示");
ImGui::EndHorizontal();

ImGui::Spring();

ImGui::BeginHorizontal("RightCluster");
ImGui::Button("移动");
ImGui::Button("旋转");
ImGui::Button("缩放");
ImGui::EndHorizontal();

ImGui::EndHorizontal();
```

优点：

- 完全统一到一套布局语义
- 不需要 `SuspendLayout()`
- 后续维护成本最低

## 项目中的建议

对于本项目编辑器 UI，建议按下面的优先级使用：

1. 新写 UI：直接使用纯 StackLayout
2. 老 UI 改造第一步：外层 StackLayout，内层 `SuspendLayout()/ResumeLayout()`
3. 老 UI 稳定后：再逐步把内部 `SameLine()` 改成嵌套 StackLayout

## 常见问题

### `Spring()` 没有效果

优先检查：

- 当前是否真的在 `BeginHorizontal/BeginVertical` 内部
- 根布局是否给了明确主轴尺寸
- 是否被旧的手工偏移逻辑抵消

### Metrics 看起来不对

优先检查：

- 是否在 StackLayout 内继续混用了大量 `SameLine()`
- 是否应该在旧代码段外包一层 `SuspendLayout()/ResumeLayout()`
- 是否把多个控件块正确收敛成了 `Group`

### 什么时候该用 `Group`

当你希望一整块旧 UI 在父布局里被当成“一个 item”时，用 `BeginGroup()/EndGroup()` 包起来最合适。

常见场景：

- 左侧一整组工具按钮
- 右侧一整组吸附与速度控件
- 某个复合控件块需要整体 hover / tooltip / 测量

## 结论

`StackLayout` 的正确理解是：

- 它是一套新的布局推进机制
- 不是旧式 `SameLine()` 的补充
- 迁移老代码时，应通过 `SuspendLayout()/ResumeLayout()` 明确切换边界

在工具栏这种“左边一组、右边一组、中间弹性空白”的场景里，推荐写法就是：

- 外层 `BeginHorizontal(full_width)`
- 左组
- `Spring()`
- 右组
- 必要时在组内使用 `SuspendLayout()/ResumeLayout()` 兼容旧代码
