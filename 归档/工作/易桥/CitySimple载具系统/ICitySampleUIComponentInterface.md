## 代码
```cpp
class CITYSAMPLE_API ICitySampleUIComponentInterface  
{  
    GENERATED_BODY()  
  
public:  
    /** Hook for adding an overlay to the CitySampleUI component when a pawn is set. */  
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "UI")  
    UCitySamplePanel* AddOverlay(UCitySampleUIComponent* CitySampleUI, const bool bSkipAnimation=false);  
    virtual UCitySamplePanel* AddOverlay_Implementation(UCitySampleUIComponent* CitySampleUI, const bool bSkipAnimation=false);  
  
    /** Hook for removing an overlay from the CitySampleUI component when a pawn is set. */  
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category = "UI")  
    void RemoveOverlay(UCitySampleUIComponent* CitySampleUI, const bool bSkipAnimation=false);  
    virtual void RemoveOverlay_Implementation(UCitySampleUIComponent* CitySampleUI, const bool bSkipAnimation=false);  
};
```
## AI 提示
> [!NOTE]
> ## CitySample UI 组件 梳理
> 
> 这段代码定义了一个名为 `UCitySampleUIComponent` 的 Actor 组件，用于管理 CitySample 游戏的 UI 界面。下面是对代码的详细梳理，使用中文进行解释：
> 
> **1. 核心功能：**
> 
> - **UI 管理：** 组件负责创建、显示和隐藏各种 UI 面板，包括主面板、选项菜单、控制提示覆盖层和交互提示。
> - **输入绑定：** 组件管理游戏内的输入绑定，允许玩家自定义按键设置，并处理 UI 相关的输入事件。
> - **控制提示：** 根据游戏上下文，动态显示合适的控制提示信息，帮助玩家了解当前可用的操作。
> - **事件驱动：** 使用事件（delegate）来通知其他游戏系统 UI 状态的变化。
> 
> **2. 属性（Properties）：**
> 
> - **`ControlsFlavor` (ECitySampleControlsFlavor):** 当前控制提示的类型或风格。
> - **`ControlsFlavorChangedEvent` (FControlsFlavorChangedEvent):** 当 `ControlsFlavor` 改变时触发的事件。
> - __`BasePanel` (UCitySamplePanel_):_* 根 UI 面板实例。
> - __`OptionsMenu` (UCitySampleMenu_):_* 选项菜单 UI 面板实例。
> - __`ControlsOverlay` (UCitySampleControlsOverlay_):_* 控制提示覆盖层 UI 面板实例。
> - __`PromptWidget` (UUserWidget_):_* 交互提示 Widget 实例。
> - **`bOptionsMenuEnabled` (bool):** 是否启用选项菜单。
> - **`bOptionsMenuActive` (bool):** 选项菜单是否当前激活（显示）。
> - **`bControlsOverlayActive` (bool):** 控制提示覆盖层是否当前激活（显示）。
> - **`bInteractPromptActive` (bool):** 交互提示是否当前激活（显示）。
> - __`InGameInputMappingContext` (UInputMappingContext_):_* 游戏内使用的输入映射上下文。
> - **`InputMappingPriority` (int32):** 输入映射的优先级。
> - __`UpAction`, `DownAction`, `LeftAction`, `RightAction`, `TogglePrevAction`, `ToggleNextAction`, `ConfirmAction`, `CancelAction`, `HideUIAction` (UInputAction_):_* UI 相关动作的输入动作对象。
> 
> **3. 方法（Methods）：**
> 
> - **`SetupInputBindings()`:** 设置输入绑定。
> - **`AddInGameInputContext()`:** 添加游戏内输入上下文。
> - **`RemoveInGameInputContext()`:** 移除游戏内输入上下文。
> - **`AddInputContext()`:** 向玩家控制器添加输入上下文。
> - **`RemoveInputContext()`:** 从玩家控制器移除输入上下文。
> - **`UpActionBinding()`, `DownActionBinding()`, `LeftActionBinding()`, `RightActionBinding()`, `TogglePrevActionBinding()`, `ToggleNextActionBinding()`, `ConfirmActionBinding()`, `CancelActionBinding()`, `HideUIActionBinding()`:** 处理各种输入动作的绑定事件。
> - **`OnPawnChanged()`:** 当 Pawn 改变时调用的函数。
> - **`OnCitySamplePanelsCreated()`:** 当 CitySample 面板创建完成后调用的函数。
> - **`ReceivePawnChanged()`:** 接收 Pawn 改变的事件。
> - **`OnPawnChanged(AController* Controller, APawn* OldPawn, APawn* NewPawn)`:** 处理 Pawn 改变事件。
> 
> **4. UI 面板管理：**
> 
> - **`BasePanel`:** 根 UI 面板，可能包含所有其他 UI 元素的容器。
> - **`OptionsMenu`:** 选项菜单，用于调整游戏设置。
> - **`ControlsOverlay`:** 控制提示覆盖层，用于显示当前可用的操作指令。
> - **`PromptWidget`:** 交互提示，用于显示特定对象的交互信息。
> 
> **5. 输入管理：**
> 
> - **输入上下文 (Input Mapping Context):** 用于将输入动作与游戏逻辑关联起来。
> - **输入动作 (Input Action):** 定义了玩家可以执行的操作，例如移动、跳跃、射击等。
> - **输入绑定 (Input Binding):** 将输入动作与特定的按键或手柄按钮关联起来。
> 
> **6. 关键功能总结：**
> 
> - **动态控制提示:** 根据游戏状态动态显示控制提示，提供良好的用户体验。
> - **自定义按键:** 允许玩家自定义按键设置，满足不同玩家的需求。
> - **模块化 UI:** 使用不同的 UI 面板来组织不同的 UI 元素，方便维护和扩展。
> - **事件驱动:** 使用事件来通知其他游戏系统 UI 状态的变化，实现解耦。
> 
> **总而言之，`UCitySampleUIComponent` 是一个功能强大的 UI 管理组件，它提供了动态控制提示、自定义按键、模块化 UI 和事件驱动等功能，可以帮助开发者创建出用户体验良好的游戏 UI。**

