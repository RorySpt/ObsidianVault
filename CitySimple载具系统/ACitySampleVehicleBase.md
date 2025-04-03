# 构造

继承自AWheeledVehiclePawn：
```cpp
Mesh = CreateDefaultSubobject<USkeletalMeshComponent>(VehicleMeshComponentName);  
Mesh->SetCollisionProfileName(UCollisionProfile::Vehicle_ProfileName);  
Mesh->BodyInstance.bSimulatePhysics = false;  
Mesh->BodyInstance.bNotifyRigidBodyCollision = true;  
Mesh->BodyInstance.bUseCCD = true;  
Mesh->SetGenerateOverlapEvents(true);  
Mesh->SetCanEverAffectNavigation(false);  
RootComponent = Mesh;  

VehicleMovementComponent = CreateDefaultSubobject<UChaosVehicleMovementComponent, UChaosWheeledVehicleMovementComponent>(VehicleMovementComponentName);  
VehicleMovementComponent->SetIsReplicated(true); // Enable replication by default  
VehicleMovementComponent->UpdatedComponent = Mesh;
```
创建了`USkeletalMeshComponent`并设置为根，创建`UChaosWheeledVehicleMovementComponent` 设置`UpdatedComponent`为`Mesh`

```cpp
bHideDrivingUI = false;  
TeleportOffsetStep = 100.0f;  
FlipVehicleMaxDistance = 3000.0f;  
bBurnoutActivated = false;  
DefaultAngularDamping = 0.0f;  
  
CCDSpeedThreshold = 60.0f;  
  
const int32 DriveableVehicleWheelCount = 4;  
WheelSuspensionStates.Init(FCitySampleWheelSuspensionState(), DriveableVehicleWheelCount);  
  
TransformsOverTime.AddDefaulted(4);  
  
CreateDefaultSubobject<UMassAgentComponent>(TEXT("MassAgent"));
```
创建`UMassAgentComponent`

# 绑定按键
```cpp
void ACitySampleVehicleBase::SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent)  
{  
    check(PlayerInputComponent);  
  
    if (UEnhancedInputComponent* EnhancedInputComponent = Cast<UEnhancedInputComponent>(PlayerInputComponent))  
    {       EnhancedInputComponent->BindAction(LookAction, ETriggerEvent::Triggered, this, &ThisClass::LookActionBinding);  
       EnhancedInputComponent->BindAction(LookAction, ETriggerEvent::Completed, this, &ThisClass::LookActionBinding);  
  
       EnhancedInputComponent->BindAction(LookDeltaAction, ETriggerEvent::Triggered, this, &ThisClass::LookDeltaActionBinding);  
  
       EnhancedInputComponent->BindAction(ThrottleAction, ETriggerEvent::Triggered, this, &ThisClass::ThrottleActionBinding);  
       EnhancedInputComponent->BindAction(ThrottleAction, ETriggerEvent::Completed, this, &ThisClass::ThrottleActionBinding);  
  
       EnhancedInputComponent->BindAction(BrakeAction, ETriggerEvent::Triggered, this, &ThisClass::BrakeActionBinding);  
       EnhancedInputComponent->BindAction(BrakeAction, ETriggerEvent::Completed, this, &ThisClass::BrakeActionBinding);  
  
       EnhancedInputComponent->BindAction(SteeringAction, ETriggerEvent::Triggered, this, &ThisClass::SteeringActionBinding);  
       EnhancedInputComponent->BindAction(SteeringAction, ETriggerEvent::Completed, this, &ThisClass::SteeringActionBinding);  
        EnhancedInputComponent->BindAction(HandbrakeAction, ETriggerEvent::Started, this, &ThisClass::HandbrakeActionPressedBinding);  
       EnhancedInputComponent->BindAction(HandbrakeAction, ETriggerEvent::Completed, this, &ThisClass::HandbrakeActionReleasedBinding);  
    }    else  
    {  
       UE_LOG(LogCitySample, Warning, TEXT("Failed to setup player input for %s, InputComponent type is not UEnhancedInputComponent."), *GetName());  
    }}
```

## AI 笔记
> [!NOTE]
> ## 车辆驱动状态与输入控制 - 详细笔记
> 
> 这份笔记基于提供的C++代码片段，详细记录了车辆驱动状态管理、输入控制以及相关参数的定义和功能。它涵盖了车辆状态跟踪、驾驶特性调整、输入绑定和一些特殊事件处理。
> 
> **一、核心状态与数据结构**
> 
> - **`FCitySampleDrivingState`:** 这是一个自定义的数据结构，用于存储车辆的各种驾驶相关信息。 具体内容未在代码片段中显示，但可以推断它包含了速度、加速度、转向角度、当前档位、燃油消耗等信息。
> - **`bIsDrifting` (bool):** 标记车辆是否正在漂移。 用于调整车辆的物理行为和视觉效果。
> - **`bBurnoutActivated` (bool):** 标记是否激活了车辆的急加速/甩尾（Burnout）功能。
> - **`bIsOkayToFireAirborneEvents` (bool):** 一个门控标志，用于防止车辆刚生成时就触发空中事件。 当车辆第一次落地时设置为true。
> - **`DefaultCenterOfMassOffset` (FVector):** 车辆的默认重心偏移量。 用于在车辆状态变化时（例如，空中状态）恢复到初始状态。
> - **`DefaultAngularDamping` (float):** 车辆的默认角阻尼。 用于在车辆状态变化时（例如，空中状态）恢复到初始状态。
> - **`bIsReversing` (bool):** 标记车辆是否正在倒车。
> - **`bIsLiterallyBreaking` (bool):** 标记车辆是否正在真正刹车。 区分了倒车时的“刹车”和正常刹车。
> - **`bThrottleDisabled` (bool):** 标记车辆油门是否被禁用（例如，因车辆损坏）。
> - **`SteeringModifier` (float):** 车辆转向的修正值，可能由车辆损坏或其他因素引起。
> - **`TimeSpentSharpSteering` (float):** 车辆保持急转弯的时间，可能用于触发特殊事件或调整驾驶体验。
> - **`PreviouslyAwake` (bool):** 记录车辆上次是否处于唤醒状态（例如，是否在移动）。 用于优化LOD（Level of Detail）切换。
> - **`BlendTimer` (float):** 用于LOD切换的计时器。
> 
> **二、驾驶特性调整**
> 
> - **重心偏移 (`DefaultCenterOfMassOffset`)**: 通过调整重心，可以改变车辆的稳定性、操控性和物理特性。
> - **角阻尼 (`DefaultAngularDamping`)**: 调整角阻尼可以控制车辆旋转的速度和幅度，影响车辆的操控性和稳定性。
> - **漂移 (`bIsDrifting`)**: 漂移状态下，车辆的物理行为会发生改变，例如轮胎抓地力降低、侧滑增加等。
> - **急加速/甩尾 (`bBurnoutActivated`)**: 激活急加速/甩尾功能后，车辆的轮胎可能会打滑，产生视觉效果和物理反馈。
> 
> **三、输入控制**
> 
> - **输入映射上下文 (`InputMappingContext`)**: 用于定义和管理车辆的输入映射，例如将键盘按键或手柄摇杆映射到特定的动作。
> - **输入动作 (`InputAction`)**: 定义了可以执行的动作，例如加速、刹车、转向、手刹等。
> - **输入绑定**: 将输入动作与特定的输入事件（例如按键按下、摇杆移动）关联起来。
> 
> **四、输入绑定函数**
> 
> 以下函数是与输入动作关联的绑定函数，用于处理玩家的输入：
> 
> - **`ThrottleActionBinding(const struct FInputActionValue& ActionValue)`**: 处理油门输入。
> - **`BrakeActionBinding(const struct FInputActionValue& ActionValue)`**: 处理刹车输入。
> - **`SteeringActionBinding(const struct FInputActionValue& ActionValue)`**: 处理转向输入。
> - **`HandbrakeActionPressedBinding(const struct FInputActionValue& ActionValue)`**: 处理手刹按下事件。
> - **`HandbrakeActionReleasedBinding(const struct FInputActionValue& ActionValue)`**: 处理手刹释放事件。
> - **`LookActionBinding(const struct FInputActionValue& ActionValue)`**: 处理视角控制的动作。
> - **`LookDeltaActionBinding(const struct FInputActionValue& ActionValue)`**: 处理视角变化的动作。
> 
> **五、事件处理**
> 
> - **`OnLiteralBrakeStart()`**: 在车辆开始刹车时调用。
> - **`OnLiteralBrakeStop()`**: 在车辆停止刹车时调用。
> - **`OnLiteralReverseStart()`**: 在车辆开始倒车时调用。
> - **`OnLiteralReverseStop()`**: 在车辆停止倒车时调用。
> - **`OnEnterAirborne()`**: 当车辆进入空中时调用。
> - **`OnLand()`**: 当车辆落地时调用。
> - **`OnEnterVehicle()`**: 当玩家进入车辆时调用。
> 
> **六、特殊事件处理**
> 
> - **空中事件控制 (`bIsOkayToFireAirborneEvents`)**: 防止车辆刚生成时就触发空中事件，确保事件的合理性。
> - **LOD切换优化 (`PreviouslyAwake`, `BlendTimer`)**: 通过记录车辆上次是否处于唤醒状态，并使用计时器，优化LOD切换过程，减少性能开销。
> 
> **七、重要参数**
> 
> - **`InputPriority`**: 用于控制输入事件的优先级，确保重要的输入事件能够及时处理。
> - **`InputMappingContext`**: 定义输入映射上下文，将输入事件与特定的动作关联起来。
> 
> **总结**
> 
> 这段代码片段展示了一个复杂的车辆驱动状态管理和输入控制系统。 它涵盖了车辆状态跟踪、驾驶特性调整、输入绑定和一些特殊事件处理。 通过合理地配置参数和事件处理函数，可以实现各种不同的驾驶体验和车辆行为。 理解这些概念对于开发高质量的车辆模拟或驾驶游戏至关重要。

