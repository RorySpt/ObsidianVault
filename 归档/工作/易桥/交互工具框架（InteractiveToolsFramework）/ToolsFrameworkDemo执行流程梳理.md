
# AToolsFrameworkDemoGameModeBase::StartPlay()

- AToolsFrameworkDemoGameModeBase::StartPlay()
	- AToolsFrameworkDemoGameModeBase::InitializeToolsSystem()
		- ToolsSystem->InitializeToolsContext(World);
		- [[RegisterTools]]();

- AToolsFrameworkDemoGameModeBase::InitializeToolsSystem()
```cpp
	
	ToolsContext = NewObject<UInteractiveToolsContext>();
	ContextQueriesAPI = MakeShared<FRuntimeToolsContextQueriesImpl>(ToolsContext, TargetWorld); // [[#IToolsContextQueriesAPI]]
	ContextTransactionsAPI = MakeShared<FRuntimeToolsContextTransactionImpl>();
	ToolsContext->Initialize(ContextQueriesAPI.Get(), ContextTransactionsAPI.Get());

	// 创建场景历史记录
	SceneHistory = NewObject<USceneHistoryManager>(this);

	// 注册选择交互
	SelectionInteraction = NewObject<USceneObjectSelectionInteraction>();
	ToolsContext->InputRouter->RegisterSource(SelectionInteraction);

	// 创建变换交互
	TransformInteraction = NewObject<USceneObjectTransformInteraction>();

	// 创建 PDI 渲染桥接组件
	PDIRenderComponent = NewObject<UToolsContextRenderComponent>(PDIRenderActor);
	PDIRenderActor->SetRootComponent(PDIRenderComponent);

	// 注册动态网格组件的目标工厂
	ToolsContext->TargetManager->AddTargetFactory( NewObject<URuntimeDynamicMeshComponentToolTargetFactory>(this) );
	
	// 注册变换小部件实用工具助手  
	UE::TransformGizmoUtil::RegisterTransformGizmoContextObject(ToolsContext);

	// 注册对象创建 API  
	URuntimeModelingObjectsCreationAPI* ModelCreationAPI = URuntimeModelingObjectsCreationAPI::Register(ToolsContext);
```

- URuntimeModelingObjectsCreationAPI::Register(ToolsContext);
```cpp
	auto NewCreationAPI = NewObject<URuntimeModelingObjectsCreationAPI>(ToolsContext);  
	ToolsContext->ContextObjectStore->AddContextObject(NewCreationAPI);
	return NewCreationAPI;
```


## IToolsContextQueriesAPI

**Tools Framework** 的用户需要实现 `IToolsContextQueriesAPI`，以提供对场景状态信息的访问，例如当前的 `UWorld`、`USelections`行为 等。

## IToolsContextTransactionsAPI

**Tools Framework** 的用户需要实现 `IToolsContextTransactionsAPI`，以便工具能够创建事务并发出更改。请注意，这在技术上是可选的，但如果没有它，将不支持撤销/重做。

`FRuntimeToolsContextTransactionImpl` 内部操作 [[#USceneHistoryManager]]
```cpp
virtual void BeginUndoTransaction(const FText& Description) override  
{  
    URuntimeToolsFrameworkSubsystem::Get()->SceneHistory->BeginTransaction(Description);  
    bInTransaction = true;  
}
```
### USceneHistoryManager
- **USceneHistoryManager** 提供了一个简单的撤销/重做历史记录堆栈。 在这里，“事务”不是 `UObject` 属性事务，而是 `FCommandChange` 对象列表。 使用事务术语是为了使与各种 **InteractiveToolsFramework API** 的关系清晰。
- 请注意，`FCommandChange` 与 `UObject` 配对，类似于编辑器事务系统。 这是必要的，因为`Change` 实现通常不存储目标 `UObject`（因为这就是编辑器的工作方式）。
- `FChangeHistoryTransaction` 和 `FChangeHistoryRecord` 结构体是 `UStructs`，存储在 `UProperties` 中，因此它们将保持这些 `UObjects` 存活并防止其被 GC（类似于 UE 编辑器）。

## USceneObjectSelectionInteraction
- USceneObjectSelectionInteraction 实现"==Scene Objects=="的标准鼠标点击选择交互，即当前`URuntimeMeshSceneSubsystem` 中的 `URuntimeMeshSceneObject`。

- - 左键单击对象会将活动选择更改为该对象
- - 左键单击“背景”会清除活动选择
- - Shift+单击修饰符添加到选择
- - Ctrl+单击修饰符切换选择/取消选择

- 目前不支持悬停，但添加起来相对容易。

## USceneObjectTransformInteraction
- `USceneObjectTransformInteraction` 为`URuntimeMeshSceneObject` 选中集合（存储在 URuntimeMeshSceneSubsystem 中）管理一个 3D 平移/旋转/缩放 (TRS) 小部件。

- 此处不控制小部件的局部/全局坐标系，小部件会根据 URuntimeToolsFrameworkSubsystem 中 IToolsContextQueriesAPI 实现提供的 EToolContextCoordinateSystem 查找这些信息。您可以在 UpdateGizmoTargets() 中配置小部件以忽略此设置。

- TRS 小部件的行为（例如枢轴位置等）由标准的 UTransformProxy 控制。有关如何执行动态修改枢轴等操作的示例代码，请参阅 UTransformMeshesTool。

## UToolsContextRenderComponent
- `UToolsContextRenderComponent` 是一个辅助组件，可以提供 `FPrimitiveDrawInterface` API 实现。这个 PDI 可以传递给 `UInterativeTool::Render`() 和 `UInteractiveGizmo::Render()`（通过 IToolsContextRenderAPI 实现）。`UToolsContextRenderComponent` 会累积任何 `DrawLine()` 和 `DrawPoint()` 请求，然后在下一帧将其传递给它的 `SceneProxy` 进行渲染。  

- （在 UE 编辑器中，这些函数可以传递一个 Editor PDI，它可以立即绘制，但在运行时这是不可能的，因此我们使用这种累积和绘制的变通方法）  

- 待办事项：线条和点可以通过 Line/Point 组件绘制，这将允许它们进行批处理，从而提供更好的性能。

## URuntimeModelingObjectsCreationAPI

- UModelingObjectsCreationAPI 的实现，UE 建模工具使用它来发出新的“网格对象”（例如 StaticMeshAsset+Actor、DynamicMeshActor、AVolume）。
- 在这个运行时系统中，我们只会发出 URuntimeMeshSceneObject。

- UE 建模工具通过在 ToolsContext 的 ContextStore 中搜索来找到可用的 UModelingObjectsCreationAPI。因此，使用
- URuntimeModelingObjectsCreationAPI::Register(ToolsContext) 函数创建一个新实例并将其设置在 ContextStore 中（并 Deregister 以将其删除）。

- 这类似于 UEditorModelingObjectsCreationAPI，这是 UE 编辑器建模模式向工具提供的接口。

- CreateTextureObject 目前不支持。

# AToolsFrameworkDemoGameModeBase::Tick()

- AToolsFrameworkDemoGameModeBase::Tick(float DeltaTime)
	- ToolsSystem->Tick(DeltaTime);

## ToolsSystem->Tick(DeltaTime);
 