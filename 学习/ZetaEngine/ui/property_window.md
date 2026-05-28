# Property Window UI 规则

## 概述
Property Window（属性窗口）用于显示和编辑场景中选中对象的属性。它采用表格布局，支持同步实例、行高亮、可调节列宽等特性。

## UI 规则

### 1. 布局
- 采用 **ImGui Tables** 实现，支持多列同步。
- 默认两列布局：属性名（左）和属性值（右）。

### 2. 表格样式
- **边框样式**：使用缝隙（gaps）代替实线边框，确保视觉上的连续性和一致性。
- **行高亮**：鼠标悬停的行应高亮显示，背景色使用 `ImGuiCol_TableRowBgAlt` 或自定义颜色。
- **单元格内边距**：上下内边距保持一致，避免边框占用额外像素。

### 3. 同步实例（Tables/Synced instances）
- 多个属性表共享相同的标识符（identifier），以同步列宽、可见性、顺序等设置。
- 实现方式：使用 `ImGui::BeginTable` 时传入相同的 `str_id`。
- 优点：保持多个属性窗口外观一致，调整一个窗口的列宽会自动同步到其他窗口。

### 4. 缩进与层级
- 支持属性组的缩进显示，使用 `ImGui::Indent()` 和 `ImGui::Unindent()` 控制缩进。
- 缩进步长：`indent_step_`（可配置，通常为 16.0f）。
- 代码示例：
  ```cpp
  ImGui::Indent(indent_step_);
  ImGui::Text("%s", name.c_str());
  ImGui::Unindent(indent_step_);
  ```

### 5. 属性绘制
- 基于 **PropertyPainter** 系统，为不同数据类型提供定制化的 UI 控件。
- 标量类型（整数、浮点数）使用 `ImGui::DragScalar` 或 `ImGui::DragScalarN`。
- 向量类型（glm::vec）使用多列拖拽控件。
- 颜色类型使用 `ImGui::ColorEdit`。
- 枚举类型使用 `ImGui::Combo`。

### 6. 交互行为
- **拖拽调节**：数值属性支持拖拽调节，速度可配置。
- **右键菜单**：支持右键点击属性名或值弹出上下文菜单（例如：重置为默认值、复制、粘贴）。
- **键盘导航**：支持方向键切换焦点，Enter 键激活编辑，Esc 键取消。
- **撤销/重做**：属性修改应支持撤销/重做，每次修改生成一个撤销记录。

### 7. 视觉反馈
- **修改标识**：已修改的属性应通过颜色（如橙色）或图标进行标记。
- **错误提示**：无效输入应显示红色边框或提示文本。
- **只读状态**：只读属性显示为灰色，不可交互。

### 8. 性能优化
- **懒加载**：仅当属性组展开时才绘制子属性。
- **脏标记**：仅当属性值发生变化时才更新 UI。
- **批处理**：将多个属性绘制调用合并，减少 ImGui 调用开销。

### 9. DPI 适配
- 所有尺寸（缩进、行高、图标大小）应乘以 `ImGui::GetWindowDpiScale()` 以适配高 DPI 显示器。
- 图像控件（如 `ImGui::ImageWithBg`）需手动缩放尺寸。

## 实现示例
```cpp
// 创建同步表格
if (ImGui::BeginTable("PropertyTable", 2, ImGuiTableFlags_SizingFixedFit | ImGuiTableFlags_Resizable))
{
    ImGui::TableSetupColumn("Name", ImGuiTableColumnFlags_WidthFixed, 100.0f);
    ImGui::TableSetupColumn("Value", ImGuiTableColumnFlags_WidthStretch);
    
    for (auto& property : properties)
    {
        ImGui::TableNextRow();
        ImGui::TableSetColumnIndex(0);
        ImGui::Text("%s", property.name.c_str());
        
        ImGui::TableSetColumnIndex(1);
        PropertyPainter::Paint(property);
    }
    ImGui::EndTable();
}
```

## 检查清单
- [x] 表格边框改为缝隙样式
- [x] 鼠标悬停行高亮
- [x] 同步实例支持
- [x] 缩进层级正确
- [x] 属性绘制器集成
- [x] 撤销/重做支持
- [x] DPI 适配
- [ ] 右键菜单实现
- [ ] 键盘导航完善
- [ ] 性能优化（懒加载）