```cpp
  
template <typename T, typename... Ts>concept any_of_v = // true if and only if T is in Ts    
(std::is_same_v<T, Ts> || ...);  
  
template <typename T> requires any_of_v<T, ImU8, ImS8, ImU16, ImS16, ImU32, ImS32, ImU32, ImS64>  
struct ScalarPropertyPainter  
{  
    using value_type = std::decay_t<T>;  
    using speed_type = std::conditional_t<std::is_floating_point_v<T>, float, int>;  
  
    static consteval ImGuiDataType get_gui_data_type()  
    {        if constexpr (std::same_as<T, ImS8>) { return ImGuiDataType_S8; }  
        else if constexpr (std::same_as<T, ImU8>) { return ImGuiDataType_U8; }  
        else if constexpr (std::same_as<T, ImS16>) { return ImGuiDataType_S16; }  
        else if constexpr (std::same_as<T, ImU16>) { return ImGuiDataType_U16; }  
        else if constexpr (std::same_as<T, ImS32>) { return ImGuiDataType_S32; }  
        else if constexpr (std::same_as<T, ImU32>) { return ImGuiDataType_U32; }  
        else if constexpr (std::same_as<T, ImS64>) { return ImGuiDataType_S64; }  
        else if constexpr (std::same_as<T, ImU64>) { return ImGuiDataType_U64; }  
        return {};  
    }  
    template <glm::length_t L>  
    static bool drag_scalar(std::string_view label, std::span<value_type, L> v, speed_type v_speed,  
                            std::span<value_type> v_min = {}, std::span<value_type> v_max = {}, std::string_view format,  
                            ImGuiSliderFlags flags)  
    {        if constexpr (L == 1)  
        {            return ImGui::DragScalar(label.data(), get_gui_data_type(), v.data(), v_speed,  
                                     v_min.empty() ? nullptr : v_min.data(), v_max.empty() ? nullptr : v_max.data(),  
                                     format.data(), flags);  
        }        else  
        {  
            return ImGui::DragScalarN(label.data(), get_gui_data_type(), v.data(), L, v_speed,  
                                      v_min.empty() ? nullptr : v_min.data(), v_max.empty() ? nullptr : v_max.data(),  
                                      format.data(), flags);  
        }    }  
    template <glm::length_t L>  
    static bool paint_scalar(Object* object, const PropertyBase* property)  
    {        bool changed = false;  
        glm::vec<L, T> value = object->get_property<glm::vec<L, T>>(property->get_name());  
        ImGui::PushID(object);  
        if (drag_scalar<L>(property->get_display_name().c_str(), std::span{glm::value_ptr(value), L}, 1))  
        {            object->set_property(property->get_name(), value);  
            changed = true;  
        }        ImGui::PopID();  
        return changed;  
    }};
```