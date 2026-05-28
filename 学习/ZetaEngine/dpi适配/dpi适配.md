# Image缩放
~~修改ImGui::ImageWithBg函数，手动使用dpi缩放image的size~~
`ImGui::ImageWithBg`和`Image::Image`系列函数是为了正确适应不同dpi应该将size乘以`ImGui::GetWindowDpiScale()`

