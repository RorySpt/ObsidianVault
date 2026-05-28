# ImGui 分层封装架构设计

## 背景与目标

本项目的编辑器界面基于 ImGui。后续如果要提升 UI 代码的一致性、可维护性和复用率，直接对 ImGui 做全量二次封装并不是一个合适的方向。

ImGui 的核心优势是 immediate mode、低抽象成本和可随时落回原生 API。如果把整套 API 再包一层，通常会带来几个问题：

- 封装层和原生行为逐渐偏离，调试成本升高
- 维护成本持续增加，需要反复跟进 ImGui 升级
- 开发者仍然必须理解原生 ImGui 语义，封装价值有限
- 过重的抽象会削弱编辑器侧快速迭代的能力

因此，这里的目标不是替代 ImGui，而是在保留原生 `ImGui::` 直用能力的前提下，建立一层 `ui::` / `editor_ui::` 辅助库：

- 收口高频样板代码
- 统一项目内的布局与视觉约定
- 抽出可复用的交互模式
- 为资产浏览、日志视图、属性面板等复杂编辑器控件提供稳定承载层

设计原则如下：

- 封装模式，不封装全部 API
- 封装项目约定，不遮蔽 ImGui 本体
- 封装高频重复代码，不限制开发者直接使用原生 ImGui
- 依赖方向始终向下，领域层不反向污染基础层

## 总体分层

建议将 ImGui 相关封装拆成四层：

1. `Core`
2. `Layout / Style`
3. `Patterns`
4. `Domain Widgets`

层级关系如下：

```text
Domain Widgets
    ↓
Patterns
    ↓
Layout / Style
    ↓
Core
    ↓
ImGui
```

其中：

- 越往下越接近原生 ImGui，越偏基础设施
- 越往上越接近编辑器业务，越偏复合控件
- 上层只能依赖下层
- 任意层都不应禁止直接调用 `ImGui::`

## 1. Core 层

### 职责

`Core` 层只负责对成对出现的调用做安全收口，避免 `Push/Pop`、`Begin/End`、`Open/Close` 这类调用失配。它不改变 ImGui 的语义，也不引入新的控件系统。

这层应尽量薄，目标是让窗口代码在保持原有表达能力的同时更安全、更少样板代码。

### 建议接口

- `ui::ScopedID`
- `ui::ScopedStyleColor`
- `ui::ScopedStyleVar`
- `ui::ScopedFont`
- `ui::ScopedDisable`
- `ui::WindowGuard`
- `ui::ChildGuard`
- `ui::TableGuard`
- `ui::PopupGuard`

### 设计要求

- 仅封装生命周期和对称调用
- 不持有业务数据
- 不做状态机
- 不镜像封装 `Button`、`Text`、`InputText`、`Selectable` 等基础 API
- 保证调用侧可自然混写 `ImGui::XXX()`

### 适用示例

- 在窗口函数中使用 guard 保证 early return 时不会遗漏 `EndTable()`
- 在复杂渲染路径中用 `ScopedStyleColor` 减少多处 `Push/PopStyleColor`

## 2. Layout / Style 层

### 职责

`Layout / Style` 层用于统一编辑器的布局与风格约定，把零散散落在窗口代码中的尺寸、留白、颜色、列宽策略和 DPI 适配规则抽离出来。

它解决的是“整个编辑器看起来和行为上是否一致”的问题，而不是具体的业务逻辑问题。

### 建议接口

- `editor_ui::EditorStyleTokens`
- `editor_ui::ToolbarLayout`
- `editor_ui::PanelPadding`
- `editor_ui::PropertyTableStyle`
- `editor_ui::IconButtonStyle`
- `editor_ui::StatusBarStyle`

### 建议沉淀内容

- 工具栏常用控件宽度
- 缩略图默认尺寸和网格 cell padding
- 属性面板双列表格配置
- 通用 hover / selected / disabled 颜色
- DPI 感知后的标准尺寸换算函数
- 常用图标按钮的尺寸和背景策略

### 设计要求

- 这一层只表达视觉与布局约定
- 不依赖具体业务模块
- 不感知资源、日志、场景对象等领域概念
- 不承载点击、选择、过滤等业务状态

## 3. Patterns 层

### 职责

`Patterns` 层负责抽象项目内反复出现的 UI 使用模式。它是最有价值的一层，因为它能显著减少重复逻辑，又不会把封装做得过重。

这层适合承载：

- 大列表/大表格/大网格的虚拟化遍历
- 单选、多选、范围选择的通用状态模型
- 搜索栏、工具栏、属性行等重复布局模式
- 上下文菜单、行操作、空状态面板等交互模式

### 建议模块

- `ui::draw_clipped_list`
- `ui::draw_clipped_rows`
- `ui::draw_virtual_grid`
- `ui::SelectionModel`
- `ui::SearchBarState`
- `ui::ToolbarBuilder`
- `ui::draw_property_row`
- `ui::ContextMenuPattern`

### 最小推荐接口

```cpp
namespace ui
{
template <class DrawItemFn>
void draw_clipped_list(int count, DrawItemFn&& draw_item);

template <class DrawRowFn>
void draw_clipped_rows(int row_count, DrawRowFn&& draw_row);

template <class DrawCellFn>
void draw_virtual_grid(int item_count, int columns, DrawCellFn&& draw_cell);
}
```

### 设计要求

- `clipper` helper 只负责可见区遍历
- 选择模型只负责输入规则和状态，不负责业务行为
- 搜索、过滤、排序 helper 只负责组织状态和调用约定
- 不在这一层处理“打开资源”“删除对象”“跳转场景”等领域动作

### 关键建议

#### 列表

列表场景优先封装到“索引遍历”层，不要做成重量级列表控件。

#### 表格

表格场景优先封装到“按行 clip”层。因为 `ImGuiListClipper` 更适合等高行，不适合对复杂不等高 item 直接抽象成统一 item clip。

#### 网格

网格场景也应按行 clip，行内自行换算列索引。这样可以同时兼容列数变化、窗口缩放和最后一行不满的情况。

### 与当前仓库的对应关系

- 日志窗口适合使用 `draw_clipped_list` 或 `draw_clipped_rows`
- 资源浏览网格适合使用 `draw_virtual_grid`，内部按行驱动，单元格绘制仍在调用侧完成

## 4. Domain Widgets 层

### 职责

`Domain Widgets` 层承载编辑器专用的复合控件。这一层可以理解项目领域，可以直接围绕资产、世界对象、材质、日志等概念组织 UI。

它的目标不是“通用”，而是“对当前编辑器高价值且稳定”。

### 适合抽象的控件

- `editor_ui::AssetGridView`
- `editor_ui::LogView`
- `editor_ui::PropertyInspector`
- `editor_ui::OutlinerView`
- `editor_ui::ContentToolbar`

### 设计要求

- 使用下层的 `Core`、`Layout / Style`、`Patterns`
- 接口尽量面向明确的 state / view-model / callback
- 尽量避免直接读取全局单例后在内部完成所有副作用
- 复合控件之间不要互相耦合

### 当前仓库的建议落点

#### 日志视图

日志窗口的过滤、颜色规则和行渲染逻辑已经比较稳定，后续可逐步下沉为 `editor_ui::LogView`：

- 文本过滤和级别过滤保留在视图状态中
- 可见区遍历走 `ui::draw_clipped_list`
- 点击复制、颜色渲染、自动滚动作为日志视图自身行为

#### 资源浏览视图

资源浏览窗口适合拆成两层：

- `ui::draw_virtual_grid` 负责按行虚拟化遍历
- `editor_ui::AssetGridView` 负责单元格绘制、选中态、高亮、双击打开、缩略图布局

这样可以把虚拟化和领域逻辑分开，避免网格 helper 变成“万能资源控件”。

## 命名空间与边界约定

推荐对外只暴露两套命名空间：

### `ui::`

定位是通用 UI 辅助层，面向所有 ImGui 页面复用。可包含：

- RAII guard
- clipper helper
- selection model
- search bar state
- property row helper
- toolbar builder

### `editor_ui::`

定位是编辑器专用层，承载编辑器风格和领域控件。可包含：

- 视觉 token
- 资产视图
- 日志视图
- 属性检查器
- 内容浏览工具栏

### `ImGui::`

原生 `ImGui::` 必须继续保留为一等公民：

- 不禁用
- 不隐藏
- 不要求开发者必须经过 `ui::`
- 不做全量转发封装

这条边界是整个架构能长期维持健康的关键。

## 推荐落地顺序

建议按以下顺序推进，而不是一次性大重构：

### 第一阶段：Core

先补齐 guard 和样板收口类，解决最容易出错的生命周期问题。

优先项：

- `ScopedID`
- `ScopedStyleColor`
- `ScopedStyleVar`
- `ScopedDisable`
- `TableGuard`

### 第二阶段：Patterns

在不改变现有窗口行为的前提下，抽出高频模式。

优先项：

- `draw_clipped_list`
- `draw_clipped_rows`
- `draw_virtual_grid`
- `draw_property_row`
- 搜索栏和工具栏布局 helper

### 第三阶段：Layout / Style

把稳定的视觉与布局约定沉淀为 token 和 style helper，减少魔法数字扩散。

### 第四阶段：Domain Widgets

选择高重复、已稳定的编辑器视图做领域级封装。推荐先从以下两个窗口验证：

- 日志窗口
- 资源浏览窗口

## 禁止事项

以下方式不建议采用：

- 对 ImGui 做全量 API 镜像封装，例如 `ui::Button`、`ui::Text`、`ui::Checkbox` 全量重复定义
- 在基础 helper 中混入业务动作，例如在 clipper helper 中直接处理资源打开或对象删除
- 用 retained-mode widget 树替换当前 immediate mode 组织方式
- 强制所有窗口只能通过二次封装层编写，禁止原生 `ImGui::`
- 在通用层直接依赖资产系统、场景系统、日志系统等领域模块

这些做法会显著增加复杂度，并削弱 ImGui 的使用优势。

## 验收标准

后续实现该架构时，建议以以下标准验证：

- guard 类在 early return 路径下不产生 `Push/Pop`、`Begin/End` 失配
- 列表、表格、网格虚拟化 helper 不改变现有交互语义
- `ui::` 与 `editor_ui::` 引入后，窗口代码仍可与原生 `ImGui::` 自然混写
- 资产浏览与日志视图可在不引入额外复杂度的前提下完成首批迁移
- 视觉 token 和布局 helper 能减少窗口代码中的硬编码尺寸与颜色

## 结论

本项目对 ImGui 的封装方向应当是“分层增强”，而不是“整体替换”。合理的做法是保留原生 ImGui 作为基础表达层，在其上增加通用 helper、视觉约定和领域复合控件。

最终目标不是让所有代码都通过封装层间接访问 ImGui，而是让高频模式有稳定承载、让编辑器风格逐渐统一、让复杂窗口的维护成本下降，同时继续保留原生 ImGui 的灵活性。
