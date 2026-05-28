实现总结

  1. 组件层 (Component)

  文件：
  - function/component/billboard/billboard_component.h
  - function/component/billboard/billboard_component.cpp

  功能：
  - texture_ - 纹理
  - material_ - 材质
  - size_ - 尺寸 (Vec2)
  - color_ - 颜色 (Vec4)
  - calc_bounds() - 球形包围盒计算

  2. 渲染数据层 (Render Data)

  文件：
  - render/render_data/billboard_render_data.h
  - render/render_data/billboard_render_data.cpp

  功能：
  - BillboardInstanceData - 实例数据结构 (position, color, size)
  - BillboardElement - 渲染元素，包含实例缓冲区
  - draw_billboards() - GPU 绘制调用

  3. 场景层 (Scene)

  修改文件：
  - render/scene/scene.h - 添加 billboard_render_data_
  - render/scene/scene.cpp - 注册更新器
  - render/scene/scene_component_updaters.cpp - 实现 update_billboard_component

  4. 渲染通道层 (Render Pass)

  修改文件：
  - render/render_pass/forward/forward_pass.h
  - render/render_pass/forward/forward_pass.cpp
  - render/renderer/scene/scene_renderer.cpp
  - render/renderer/thumbnail/thumbnail_renderer.cpp

  5. Shader 层

  新增文件：
  - shader/common/billboard.h - Billboard 顶点着色器逻辑

  修改文件：
  - shader/common/mesh.vert - 添加 BILLBOARD 分支
  - shader/pbr/material.h - 添加 BILLBOARD 输入变量声明

  使用方式
```cpp
// 创建 Billboard 组件
auto billboard = entity->add_component<BillboardComponent>();
billboard->set_texture(my_texture);
billboard->set_material(my_material);  // 可选
billboard->set_billboard_size(Vec2(2.0f, 2.0f));
billboard->set_color(Vec4(1.0f, 1.0f, 1.0f, 1.0f));
```


  渲染流程

  1. Scene::update_billboard_component 收集所有 BillboardComponent 数据
  2. 按混合模式分组到 billboard_render_data_
  3. ForwardPass::execute 中使用 BILLBOARD shader define 绘制
  4. Shader 中计算面向相机的四边形顶点位置


# 优化

## instance少量实体优化
```cpp
if (element.instance_data.size() <= 2) // 如果实例数为两个或以下，instance信息使用pushcontent传递
{
    auto material_constant = element.material_render_data->get_material_constant();
    std::memcpy(material_constant, element.instance_data.data(), element.instance_data.size());
}
```

```cpp
if (std::exchange(billboard_element.instance_buffer_dirty, false))
{
    if (billboard_element.instance_data.size() > 2)
    {
        RenderBufferManager::get_instance().update(billboard_element.instance_buffer, billboard_element.instance_data,
                                                vk::BufferUsageFlagBits::eVertexBuffer);
    }
    else if (billboard_element.shared_data->empty_instance_buffer.empty())
    {
        std::array<BillboardElement, 2> empty_instances;
        RenderBufferManager::get_instance().update(billboard_element.shared_data->empty_instance_buffer, empty_instances.data(),
                                                vk::BufferUsageFlagBits::eVertexBuffer);
    }
}
```