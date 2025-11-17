```cpp
UInteractiveToolManager* ToolManager = ToolsSystem->ToolsContext->ToolManager;  
  
auto AddPrimitiveToolBuilder = NewObject<UAddPrimitiveToolBuilder>();  
AddPrimitiveToolBuilder->ShapeType = UAddPrimitiveToolBuilder::EMakeMeshShapeType::Box;  
ToolManager->RegisterToolType("AddPrimitiveBox", AddPrimitiveToolBuilder);  
  
auto DrawPolygonToolBuilder = NewObject<URuntimeDrawPolygonToolBuilder>();  
ToolManager->RegisterToolType("DrawPolygon", DrawPolygonToolBuilder);  
  
auto PolyRevolveToolBuilder = NewObject<UDrawAndRevolveToolBuilder>();  
ToolManager->RegisterToolType("PolyRevolve", PolyRevolveToolBuilder);  
  
auto PolyEditToolBuilder = NewObject<URuntimePolyEditToolBuilder>();  
ToolManager->RegisterToolType("EditPolygons", PolyEditToolBuilder);  
  
auto MeshPlaneCutToolBuilder = NewObject<UPlaneCutToolBuilder>();  
ToolManager->RegisterToolType("PlaneCut", MeshPlaneCutToolBuilder);  
  
auto RemeshMeshToolBuilder = NewObject<URuntimeRemeshMeshToolBuilder>();  
ToolManager->RegisterToolType("RemeshMesh", RemeshMeshToolBuilder);  
  
auto VertexSculptToolBuilder = NewObject<UMeshVertexSculptToolBuilder>();  
ToolManager->RegisterToolType("VertexSculpt", VertexSculptToolBuilder);  
  
auto DynaSculptToolBuilder = NewObject<URuntimeDynamicMeshSculptToolBuilder>();  
DynaSculptToolBuilder->bEnableRemeshing = true;  
ToolManager->RegisterToolType("DynaSculpt", DynaSculptToolBuilder);  
  
auto MeshBooleanToolBuilder = NewObject<URuntimeMeshBooleanToolBuilder>();  
ToolManager->RegisterToolType("MeshBoolean", MeshBooleanToolBuilder);
```