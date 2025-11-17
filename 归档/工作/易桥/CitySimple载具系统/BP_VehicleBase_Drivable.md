添加组件：
- ContextualAnim（UContextualAnimSceneActorComponent）
- BP_AudioController（ChildActor）
- BP_CarExitInteraction（UCitySampleInteractionComponent）
- BP_CarEntryInteraction_Right（UCitySampleInteractionComponent）
- BP_CarEntryInteraction_Left（UCitySampleInteractionComponent）
- CitySampleAsyncActorDetection（UCitySampleAsyncActorDetectionComponent）
- DrivableVehicle（UDrivableVehicleComponent）
## AI 提示
> [!info]
> **1. ContextualAnim（UContextualAnimSceneActorComponent）**
> 
> - **作用:** 这个组件负责处理情境动画。它允许你根据游戏中的情境（例如，角色所处的位置、状态、与其他对象的交互等）来触发和控制动画。这使得角色动画更加自然、流畅和有反应性。
> - **简单来说:** 让角色根据周围环境和行为，播放不同的动画，例如走路、跑步、跳跃、互动等等。
> 
> **2. BP_AudioController（ChildActor）**
> 
> - **作用:** 一个音效控制器。 ChildActor 意味着它是一个独立的Actor，可以被添加到其他Actor中，负责管理和播放游戏中的音效。
> - **简单来说:** 负责播放各种声音，例如引擎声、环境音效、角色语音等等。
> 
> **3. BP_CarExitInteraction（UCitySampleInteractionComponent）**
> 
> - **作用:** 车辆下车交互组件。 `UCitySampleInteractionComponent` 表明这是一个用于处理交互的组件，专门用于玩家下车时的交互逻辑。这可能包括动画、音效、UI提示等。
> - **简单来说:** 控制玩家下车时的行为和反馈，例如播放下车动画、播放下车音效、显示提示信息等。
> 
> **4. BP_CarEntryInteraction_Right（UCitySampleInteractionComponent）**
> 
> - **作用:** 车辆右侧上车交互组件。 与BP_CarExitInteraction类似，但专注于玩家从车辆右侧上车的交互逻辑。
> - **简单来说:** 控制玩家从车辆右侧上车的行为和反馈。
> 
> **5. BP_CarEntryInteraction_Left（UCitySampleInteractionComponent）**
> 
> - **作用:** 车辆左侧上车交互组件。 与BP_CarExitInteraction类似，但专注于玩家从车辆左侧上车的交互逻辑。
> - **简单来说:** 控制玩家从车辆左侧上车的行为和反馈。
> 
> **6. CitySampleAsyncActorDetection（UCitySampleAsyncActorDetectionComponent）**
> 
> - **作用:** 异步Actor检测组件。 `Async` 表明这是一个异步组件，意味着它可以在后台执行Actor检测，而不会阻塞游戏主线程。 用于检测场景中存在的Actor，可能用于触发事件、激活交互等。
> - **简单来说:** 在后台检测场景中的Actor，例如检测玩家是否靠近车辆，从而触发上车/下车交互。
> 
> **7. DrivableVehicle（UDrivableVehicleComponent）**
> 
> - **作用:** 可驾驶车辆组件。 这个组件负责处理角色进入车辆和离开车辆的流程。进入车辆将控制玩家的控制器转移到车上，并将角色和座位关联起来，然后产生玩家进入车辆的通知。
> - **简单来说:** 让车辆可以被玩家控制。
> 
> **总结：**
> 
> 这些组件组合在一起，很可能构成了一个可驾驶车辆系统。
> 
> - `DrivableVehicle` 负责车辆的驾驶逻辑。
> - `ContextualAnim` 负责播放车辆和角色的动画，例如驾驶、上车、下车等。
> - `BP_AudioController` 负责播放车辆的音效，例如引擎声、碰撞声等。
> - `BP_CarExitInteraction`、`BP_CarEntryInteraction_Right` 和 `BP_CarEntryInteraction_Left` 负责处理玩家与车辆的交互，例如上车和下车。
> - `CitySampleAsyncActorDetection` 用于检测玩家是否靠近车辆，从而触发交互。

