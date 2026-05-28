
**目标**：v1.91.8 -> v1.91.9 -> v1.92.5

- [x] v1.91.8 -> v1.91.9
- [ ] v1.91.9 -> v1.92.5

#  v1.91.9 -> v1.92.5
- [x] 编译通过
- [ ] 启动成功

# 现有问题：
```bash
Assertion failed: g.Style.WindowMinSize.x >= 1.0f && g.Style.WindowMinSize.y >= 1.0f && "Invalid style setting!", file E:\workspace\ZetaEngine\external\imgui\imgui.cpp, line 11414
```

1. [x] ImGuizmo dpi单独适配，使用ImGui内部的dpi系数（可动态感知变化），去除引擎内部所有dpi相关逻辑
2. [x] 修改ImGui::ImageWithBg函数，手动使用dpi缩放image的size（添加详细注释说明）
3. [x] 完善ViewportClient上方工具栏的spacing逻辑，改为严格右对齐
4. 删除sky light postprocess volume报错问题

- [x] 属性面板分割竖线显示错误
![[Pasted image 20260213100138.png]]
资产下拉列表右端对不齐
![[Pasted image 20260213100212.png]]
日志窗口改为可选择、复制
![[Pasted image 20260213100456.png]]
拖动资产到场景中，和场景物体求交吸附
![[Pasted image 20260213100758.png]]

undo/redo


# property_window改造

- [x] Tables/Synced instances
- [x] 高亮鼠标所在行
- [ ] 将表格边框改为缝隙
- [ ] 边框占用了cell上边的一像素，导致内部上下边距不一致

## Tables/Synced instances

> @ImGui Tables/Synced instances
> Multiple tables with the same identifier will share their settings, width, visibility, order etc.
> 具有相同标识符的多个表格将共享其设置、宽度、可见性、顺序等。

多个表同步宽度
![[20260225-0928-18.2698468.mp4]]

## 缩进
```cpp
//window->DC.CursorPos.x = window->Pos.x + window->DC.Indent.x + window->DC.ColumnsOffset.x;
ImGui::Indent(indent_step_);  // 修改全局缩进
ImGui::Text("%s", name.c_str());  
ImGui::Unindent(indent_step_);// 还原全局缩进
```

## 高亮鼠标所在行
