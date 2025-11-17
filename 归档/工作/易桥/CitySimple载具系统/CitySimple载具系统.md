# 继承链
- [[ACitySampleVehicleBase]] :   
		- public AWheeledVehiclePawn,  
		   - public ICitySampleTraversalInterface,  
		   - public [[ICitySampleUIComponentInterface]],  
		   - public ICitySampleControlsOverlayInterface,  
		   - public ICitySampleInputInterface,  
		   - public IMassActorPoolableInterface,  
		   - public IMassTrafficVehicleControlInterface
- [[BP_VehicleBase]]
- [[BP_VehicleBase_Drivable]]
- BP_VehicleBase_Deformable
- BP_VehicleBase_Destruction
- BP_VehicleBase_SandBox
- BP_vehVan_vehicle01_Sandbox
## AI 提示
> [!tip]
> **核心功能：车辆模拟**
> 
> - **ACitySampleVehicleBase:** 这是整个车辆系统的基础类。从 `AWheeledVehiclePawn` 继承，表明这是一个基于轮子的车辆 Pawn（游戏中的角色）。这意味着它处理车辆的移动、物理模拟、碰撞检测等基本功能。
> 
> **接口功能 (Interfaces):**
> 
> - **ICitySampleTraversalInterface:** 车辆可以进行路径规划和移动，可能涉及寻路、避障等功能。
> - **ICitySampleUIComponentInterface:** 车辆可以与用户界面交互，例如显示车辆信息、控制面板等。
> - **ICitySampleControlsOverlayInterface:** 车辆可以显示控制叠加层，可能用于车辆控制提示或信息展示。
> - **ICitySampleInputInterface:** 车辆可以接收玩家输入，例如加速、刹车、转向等。
> - **IMassActorPoolableInterface:** 车辆可以被放入对象池中进行管理，提高性能，避免频繁创建和销毁。
> - **IMassTrafficVehicleControlInterface:** 车辆可以参与交通模拟，例如遵循交通规则、与其他车辆互动等。
> 
> **BP_VehicleBase：**
> 
> - 这是一个蓝图（Blueprint）类，继承自 `ACitySampleVehicleBase`。 蓝图是 Unreal Engine 中一种可视化脚本系统，用于快速创建游戏逻辑。 `BP_VehicleBase` 应该定义了车辆的基本属性和行为，例如速度、加速度、转向角度等。 可以理解为车辆的基础模板。
> 
> **BP_VehicleBase_Drivable, BP_VehicleBase_Deformable, BP_VehicleBase_Destruction, BP_VehicleBase_SandBox:**
> 
> - 这些都是继承自 `BP_VehicleBase` 的蓝图类，分别实现了不同的车辆特性：
>     - **BP_VehicleBase_Drivable:** 增加了可驾驶的功能，例如玩家控制车辆的逻辑。
>     - **BP_VehicleBase_Deformable:** 增加了车辆可变形的功能，例如车辆在碰撞时发生形变。
>     - **BP_VehicleBase_Destruction:** 增加了车辆可破坏的功能，例如车辆在受到足够伤害时被摧毁。
>     - **BP_VehicleBase_SandBox:** 可能与沙盒模式相关，例如允许玩家自定义车辆属性或行为。
> 
> **BP_vehVan_vehicle01_Sandbox:**
> 
> - 这是一个具体的车辆蓝图，继承自 `BP_VehicleBase_SandBox`。 很可能是一个货车类型的车辆，并且集成了沙盒模式的功能。
> 
> **总结：**
> 
> 整个继承链构建了一个灵活的车辆系统，允许开发者通过继承和组合不同的蓝图类，创建各种具有不同功能的车辆。 这个系统支持基本的车辆模拟、可驾驶、可变形、可破坏、沙盒模式等功能，并且可以方便地扩展和定制。 它很可能用于一个城市模拟或者开放世界类型的游戏。




