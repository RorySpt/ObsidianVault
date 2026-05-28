# ZetaEngine DPI 改造方案

## 背景

当前引擎内的 DPI 处理可以在部分场景下工作，但整体设计存在一个根本问题：窗口逻辑尺寸、帧缓冲物理尺寸、DPI 缩放系数三者没有明确分层，导致不同模块对“size”与“坐标”的理解并不一致。

现有实现中，窗口层会在创建主窗口时直接用显示器 DPI 缩放配置中的窗口宽高；与此同时，ImGui、渲染窗口、视口输入、运行时 GUI 又分别在不同位置继续处理尺寸与缩放。这种方式短期能跑通，但长期会带来以下问题：

1. `window size` 究竟表示逻辑尺寸还是物理像素尺寸，不清晰。
2. ImGui 的 `DisplaySize` 和 `DisplayFramebufferScale` 语义没有被完整贯彻。
3. 输入坐标、视口坐标、渲染目标尺寸容易混用。
4. 多显示器、不同 DPI 显示器拖动、平台窗口分离等场景下稳定性不足。

因此，需要把 DPI 处理从“局部修补”改造成“统一的数据模型和坐标语义”。

## 目标

改造后的 DPI 方案应明确区分三类数据：

### 1. WindowSizeLogical

逻辑窗口尺寸，单位是逻辑点。

用于：

1. ImGui `DisplaySize`
2. UI 布局
3. 输入命中
4. viewport 逻辑区域计算

### 2. FramebufferSizePhysical

帧缓冲实际像素尺寸，单位是物理像素。

用于：

1. Swapchain extent
2. 渲染目标尺寸
3. viewport/scissor
4. GPU pick、物理像素采样

### 3. ContentScale

逻辑坐标到物理像素的缩放系数。

通常来自：

1. `glfwGetWindowContentScale`
2. `framebuffer_size / window_size`

## 改造原则

1. 配置文件中的窗口宽高始终表示逻辑尺寸，不再在创建窗口前手动乘 DPI。
2. `NativeWindow` 必须同时暴露逻辑尺寸、帧缓冲尺寸和内容缩放，禁止继续使用语义模糊的统一 `get_size()`。
3. 输入统一使用逻辑坐标；渲染统一使用物理像素尺寸。
4. ImGui 的 `DisplaySize` 与 `FramebufferScale` 按标准语义使用，避免依赖“窗口尺寸预先乘 DPI”的隐式行为。
5. 主窗口和平台窗口的 DPI 应该以“当前窗口 content scale”为准，而不是只取启动时主显示器 DPI。
6. DPI 变化必须支持运行时更新，而不是只在初始化阶段读取一次。

## 当前实现中的主要问题

### 1. 主窗口创建时提前把配置尺寸乘 DPI

当前 `NativeWindow` 构造函数会获取主显示器 DPI，并将配置里的 `window.width` 和 `window.height` 直接乘以 DPI 后传给 GLFW 创建窗口。

这会带来两个问题：

1. 上层无法区分配置尺寸和真实窗口尺寸的语义。
2. 逻辑尺寸被“偷换”为物理尺寸，导致后续模块继续把它当逻辑尺寸使用时容易出现双重缩放或平台耦合行为。

### 2. RenderWindow 初始化时直接使用窗口 size 作为 surface extent

渲染窗口初始化 swapchain 时直接使用 `NativeWindow::get_size()`。

如果这个接口返回的是逻辑尺寸，则 swapchain 尺寸不正确；如果返回的是物理尺寸，则 UI 层再使用同一接口时会混淆逻辑与物理语义。当前实现没有建立严格边界。

### 3. ImGui 的 DisplayFramebufferScale 设置了，但渲染路径没有完整使用

运行时 GUI 会设置 `ImGui::GetIO().DisplayFramebufferScale`，但 `GuiPass` 实际渲染时并未完整使用 `draw_data->FramebufferScale`，而是把 framebuffer scale 相关逻辑注释掉并固定为 `1.0f`。

这意味着当前高 DPI 是否表现正确，依赖的是“窗口创建阶段是否已经把尺寸放大”，而不是标准 ImGui framebuffer 语义。

### 4. 输入系统混用 ImGui 与 GLFW 坐标

编辑器视口正常路径下使用 `ImGui::GetMousePos()`，但右键锁鼠路径下又使用 `glfwGetCursorPos()` 计算 delta。与此同时，视口矩形本身使用的是 ImGui 屏幕坐标。

如果没有统一的坐标语义保证，这种混用在高 DPI、多平台、多 viewport 下都存在风险。

### 5. DPI 只在启动阶段读取一次

当前实现中 DPI 更多依赖启动时主显示器信息，没有完整引入运行中内容缩放变化回调。窗口在不同缩放显示器之间移动时，主窗口与运行时 GUI 的行为并不可靠。

## 改造方案

### 阶段一：建立统一的窗口指标模型

目标：先把基础数据结构和接口语义固定下来。

建议新增统一结构：

```cpp
struct WindowMetrics
{
    glm::uvec2 window_size;       // logical size
    glm::uvec2 framebuffer_size;  // physical size
    glm::vec2 content_scale;      // framebuffer / window
};
```

`NativeWindow` 提供：

1. `glm::uvec2 get_window_size() const;`
2. `glm::uvec2 get_framebuffer_size() const;`
3. `glm::vec2 get_content_scale() const;`
4. `float get_content_scale_max() const;`
5. `WindowMetrics get_metrics() const;`

同时逐步废弃旧的 `get_size()`，避免继续出现语义模糊的调用。

#### 需要修改的模块

1. `source/runtime/platform/window/native_window.h`
2. `source/runtime/platform/window/native_window.cpp`
3. `source/runtime/platform/window/window_system.h`
4. `source/runtime/platform/window/window_system.cpp`
5. `source/editor/window/native/glfw_window_proxy.cpp`

#### GLFW 层需要增加的接口绑定

1. `glfwGetFramebufferSize`
2. `glfwGetWindowContentScale`
3. 可选：`glfwSetWindowContentScaleCallback`

#### 这一阶段的关键要求

1. 配置文件中的窗口宽高继续保留为逻辑尺寸。
2. 创建主窗口时不再手动乘显示器 DPI。
3. monitor DPI 只保留为参考信息，不再直接参与窗口初始尺寸计算。

### 阶段二：渲染链路切换到物理像素尺寸

目标：让所有 GPU 相关对象只依赖 framebuffer size 或由其推导出的物理尺寸。

#### RenderWindow

`RenderWindow` 初始化 swapchain extent 时改为使用：

1. `native_window_->get_framebuffer_size()`

而不是继续使用语义不明确的窗口 size。

#### Viewport

`Viewport::rect_` 继续定义为逻辑坐标矩形。

同时新增物理渲染尺寸接口，例如：

1. `glm::uvec2 get_render_size() const;`

默认逻辑如下：

1. `render_size = rect_.size * current_window_content_scale`
2. 对结果做 round 或 ceil
3. 如果设置了 fixed size，则 fixed size 的语义也必须明确说明是逻辑尺寸还是物理尺寸，建议统一为逻辑尺寸

#### 渲染目标分配

所有 viewport 相关 render target、picking buffer、scene render target 的尺寸分配都应改为使用 `get_render_size()`，而不是直接使用逻辑矩形尺寸。

#### 需要修改的模块

1. `source/runtime/render/render_window.cpp`
2. `source/runtime/render/render_window.h`
3. `source/runtime/render/viewport/viewport.h`
4. `source/runtime/render/viewport/viewport.cpp`
5. 各类具体 viewport 实现（如 scene viewport、texture viewport）

### 阶段三：修正 ImGui 数据流

目标：按 ImGui 标准语义处理逻辑尺寸与 framebuffer 缩放。

#### 编辑器主循环

每帧设置：

1. `io.DisplaySize = main_window->get_window_size()`
2. `io.DisplayFramebufferScale = framebuffer_size / window_size`

注意：

1. `DisplaySize` 是逻辑尺寸
2. `DisplayFramebufferScale` 是逻辑到物理像素的缩放

#### 运行时 GUI

不要只在 `GuiContext` 构造时设置一次 `DisplayFramebufferScale`。应在每帧或每窗口上下文更新时同步当前窗口的 metrics。

#### GuiPass

恢复标准 ImGui 渲染逻辑：

1. `fb_width = draw_data->DisplaySize.x * draw_data->FramebufferScale.x`
2. `fb_height = draw_data->DisplaySize.y * draw_data->FramebufferScale.y`
3. `clip_scale = draw_data->FramebufferScale`

projection 仍基于 `DisplaySize`，scissor 则在 clip rect 上乘 framebuffer scale 后再下发给 Vulkan。

#### 需要修改的模块

1. `source/editor/editor.cpp`
2. `source/runtime/render/gui/gui_context.cpp`
3. `source/runtime/render/render_pass/gui/gui_pass.cpp`

### 阶段四：统一输入坐标语义

目标：让所有输入事件在进入 viewport/client/widget 时都落在同一套逻辑坐标体系内。

#### 统一规则

1. `ViewportClient::handle_mouse_pos()` 的输入参数始终是 viewport-local logical coordinates。
2. `Widget` 命中测试和布局使用逻辑坐标。
3. 只有在 GPU pick、纹理采样或物理像素命中时，才从逻辑坐标显式转换到物理像素坐标。

#### 建议新增辅助函数

```cpp
glm::vec2 logical_to_physical(glm::vec2 pos, glm::vec2 scale);
glm::vec2 physical_to_logical(glm::vec2 pos, glm::vec2 scale);
```

#### 锁鼠路径

右键锁鼠模式如果仍使用 GLFW 原始坐标计算 delta，需要先验证 GLFW 返回值在当前平台上是否与 ImGui 使用的窗口逻辑坐标一致。

如果平台行为不稳定，则建议：

1. 统一改为使用 ImGui `MouseDelta`
2. 或维护一套明确的逻辑坐标 delta 缓存

#### 需要修改的模块

1. `source/editor/window/common/scene_viewport_window.cpp`
2. `source/runtime/function/viewport/viewport_client.cpp`
3. `source/runtime/function/viewport/application_viewport_client.cpp`
4. 各类 editor viewport client 与 interaction pick 入口

### 阶段五：支持运行时 DPI 变化

目标：支持窗口拖到不同 DPI 显示器时正确更新。

#### GLFW 回调

为 `NativeWindow` 增加事件：

1. `Delegate<float, float> on_content_scale;`

并在 GLFW 层注册：

1. `glfwSetWindowContentScaleCallback`

#### 回调触发后的行为

1. 更新窗口缓存的 content scale
2. 通知 ImGui metrics 更新
3. 通知 viewport 或 render target 在必要时重建
4. 如果编辑器样式需要响应 DPI，则在此处统一处理

#### 编辑器样式

当前项目中很多局部尺寸已经使用 `ImGui::GetWindowDpiScale()`，这是合理方向。

但需要注意：

1. 字体缩放和 style spacing/padding 缩放不是同一件事
2. 如果希望跨显示器切换时视觉保持一致，需要定义统一的 style scale 策略，而不是只启用 `ConfigDpiScaleFonts`

#### 需要修改的模块

1. `source/editor/window/native/glfw_window_proxy.cpp`
2. `source/runtime/platform/window/native_window.h`
3. `source/runtime/platform/window/native_window.cpp`
4. `source/editor/editor.cpp`

## 推荐实施顺序

为了降低风险，建议按以下顺序推进：

### 第一步：窗口指标接口改造

先把 `NativeWindow` 的逻辑尺寸、帧缓冲尺寸、content scale 分离出来，并移除窗口创建阶段的手动 DPI 乘法。

### 第二步：渲染窗口与 viewport 尺寸改造

让 swapchain、render target、pick buffer 改用物理尺寸；让 viewport rect 明确保留逻辑语义。

### 第三步：ImGui 数据流修正

把 `DisplaySize` 和 `FramebufferScale` 改成标准用法，并修正 `GuiPass`。

### 第四步：输入链路统一

整理 viewport、widget、interaction 的坐标流，避免逻辑坐标与物理像素混用。

### 第五步：DPI 运行时回调与多显示器适配

最后补齐内容缩放变化回调、窗口跨显示器拖动和样式刷新策略。

## 验收标准

改造完成后至少应满足以下验证项：

1. 100% 缩放下行为与当前版本保持一致。
2. 150% 与 200% 缩放下主窗口 UI 清晰、不发虚。
3. Swapchain 尺寸与 framebuffer size 一致。
4. viewport 内 pick、gizmo、drag drop 在高 DPI 下无偏移。
5. 运行时 GUI 的 scissor、裁剪、点击区域在高 DPI 下正确。
6. 主窗口在不同缩放显示器之间拖动时，字体、命中测试、渲染尺寸都能正确更新。
7. 多 viewport 平台窗口在不同显示器 DPI 下没有错位或裁剪异常。

## 风险与注意事项

### 1. 兼容旧接口的迁移风险

如果直接替换 `get_size()`，改动面会比较大。建议先引入显式新接口，再分阶段清理旧调用点。

### 2. Fixed size 语义需要提前定下来

例如 viewport fixed size、离屏渲染 fixed size、视频流尺寸等，必须明确到底是逻辑尺寸还是物理尺寸。建议绝大多数编辑器/窗口相关配置都用逻辑尺寸，只有编码输出这类场景用物理尺寸。

### 3. 平台差异

GLFW 在不同平台下 `window size`、`framebuffer size`、`content scale` 的关系可能并不完全一致。实现时不能假设所有平台都等价于 Windows 当前行为。

### 4. ImGui 多 viewport

开启 `ConfigDpiScaleViewports` 后，ImGui 会帮助处理平台窗口 DPI，但这不能替代引擎自身对主窗口、渲染窗口、swapchain、输入坐标的分层设计。

## 结论

本次 DPI 改造的核心，不是“修一两个缩放 bug”，而是建立一套明确的尺寸与坐标语义：

1. 逻辑尺寸负责 UI 和输入
2. 物理尺寸负责渲染
3. 缩放系数负责两者之间的转换

只要这三层关系明确，后续主窗口、多 viewport、运行时 GUI、输入拾取、跨显示器 DPI 切换都可以在同一模型下稳定工作。反之，如果继续依赖“创建窗口时先乘一次 DPI”的方式，后面只会不断在更多模块里出现隐式耦合和补丁式修复。
