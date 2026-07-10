# 嘉立创专业版 源码级完整 API 提取

> 来源: `E:\lceda-pro\resources\app\assets\pro-api\0.1.197\api.js` 逆向提取
> 所有 API 通过 `_.extensionApiMessageBus2.rpcCall("extensionApi.XXX.YYY", params)` 实现

---

## 工程文档树 (DMT_)

### DMT_Project
```
rpcCall("extensionApi.DMT_Project.openProject", projectUuid)
rpcCall("extensionApi.DMT_Project.createProject", friendlyName, name, teamUuid, folderUuid, desc, collabMode)
rpcCall("extensionApi.DMT_Project.moveProjectToFolder", projectUuid, folderUuid?)
rpcCall("extensionApi.DMT_Project.getAllProjectsUuid", teamUuid?, folderUuid?)
rpcCall("extensionApi.DMT_Project.getProjectInfo", projectUuid)
rpcCall("extensionApi.DMT_Project.getCurrentProjectInfo")
rpcCall("extensionApi.DMT_Project.getCurrentProjectUuid")
```

### DMT_Board
```
rpcCall("extensionApi.DMT_Board.createBoard", schematicUuid?, pcbUuid?)
rpcCall("extensionApi.DMT_Board.modifyBoardName", originalName, newName)
rpcCall("extensionApi.DMT_Board.copyBoard", sourceName)
rpcCall("extensionApi.DMT_Board.getAllBoardsInfo")
rpcCall("extensionApi.DMT_Board.getCurrentBoardInfo")
rpcCall("extensionApi.DMT_Board.getBoardInfo", boardName)
rpcCall("extensionApi.DMT_Board.deleteBoard", boardName)
```

### DMT_Schematic
```
rpcCall("extensionApi.DMT_Schematic.createSchematic", boardName?)
rpcCall("extensionApi.DMT_Schematic.createSchematicPage", schematicUuid)
rpcCall("extensionApi.DMT_Schematic.modifySchematicName", uuid, name)
rpcCall("extensionApi.DMT_Schematic.modifySchematicPageName", pageUuid, name)
rpcCall("extensionApi.DMT_Schematic.modifySchematicPageTitleBlock", showTitle?, titleData?)
rpcCall("extensionApi.DMT_Schematic.copySchematic", uuid, boardName?)
rpcCall("extensionApi.DMT_Schematic.copySchematicPage", pageUuid, schematicUuid?)
rpcCall("extensionApi.DMT_Schematic.getSchematicInfo", uuid)
rpcCall("extensionApi.DMT_Schematic.getSchematicPageInfo", pageUuid)
rpcCall("extensionApi.DMT_Schematic.getAllSchematicsInfo")
rpcCall("extensionApi.DMT_Schematic.getAllSchematicPagesInfo")
rpcCall("extensionApi.DMT_Schematic.getCurrentSchematicAllSchematicPagesInfo")
rpcCall("extensionApi.DMT_Schematic.getCurrentSchematicInfo")
rpcCall("extensionApi.DMT_Schematic.getCurrentSchematicPageInfo")
rpcCall("extensionApi.DMT_Schematic.reorderSchematicPages", uuid, pageItemsArray)
rpcCall("extensionApi.DMT_Schematic.deleteSchematic", uuid)
rpcCall("extensionApi.DMT_Schematic.deleteSchematicPage", pageUuid)
```

### DMT_Pcb
```
rpcCall("extensionApi.DMT_Pcb.createPcb", boardName?)
rpcCall("extensionApi.DMT_Pcb.modifyPcbName", uuid, name)
rpcCall("extensionApi.DMT_Pcb.copyPcb", uuid, boardName?)
rpcCall("extensionApi.DMT_Pcb.getAllPcbsInfo")
rpcCall("extensionApi.DMT_Pcb.getCurrentPcbInfo")
rpcCall("extensionApi.DMT_Pcb.getPcbInfo", uuid)
rpcCall("extensionApi.DMT_Pcb.deletePcb", uuid)
```

### DMT_Panel
```
rpcCall("extensionApi.DMT_Panel.createPanel")
rpcCall("extensionApi.DMT_Panel.modifyPanelName", uuid, name)
rpcCall("extensionApi.DMT_Panel.copyPanel", uuid)
rpcCall("extensionApi.DMT_Panel.getPanelInfo", uuid)
rpcCall("extensionApi.DMT_Panel.getAllPanelsInfo")
rpcCall("extensionApi.DMT_Panel.getCurrentPanelInfo")
rpcCall("extensionApi.DMT_Panel.deletePanel", uuid)
```

### DMT_EditorControl
```
rpcCall("extensionApi.DMT_EditorControl.openDocument", docUuid, splitScreenId?)
rpcCall("extensionApi.DMT_EditorControl.openLibraryDocument", libUuid, libType, uuid, splitScreenId?)
rpcCall("extensionApi.DMT_EditorControl.closeDocument", tabId)
rpcCall("extensionApi.DMT_EditorControl.activateDocument", tabId)
rpcCall("extensionApi.DMT_EditorControl.activateSplitScreen", splitScreenId)
rpcCall("extensionApi.DMT_EditorControl.getSplitScreenTree")
rpcCall("extensionApi.DMT_EditorControl.getSplitScreenIdByTabId", tabId)
rpcCall("extensionApi.DMT_EditorControl.getTabsBySplitScreenId", splitScreenId)
rpcCall("extensionApi.DMT_EditorControl.createSplitScreen", direction, tabId)
rpcCall("extensionApi.DMT_EditorControl.moveDocumentToSplitScreen", tabId, splitScreenId)
rpcCall("extensionApi.DMT_EditorControl.tileAllDocumentToSplitScreen")
rpcCall("extensionApi.DMT_EditorControl.mergeAllDocumentFromSplitScreen")
rpcCall("extensionApi.DMT_EditorControl.zoomTo", x?, y?, scaleRatio?, tabId?)
rpcCall("extensionApi.DMT_EditorControl.zoomToRegion", left, right, top, bottom, tabId?)
rpcCall("extensionApi.DMT_EditorControl.zoomToAllPrimitives", tabId?)
rpcCall("extensionApi.DMT_EditorControl.zoomToSelectedPrimitives", tabId?)
rpcCall("extensionApi.DMT_EditorControl.getCurrentRenderedAreaImage", tabId?)
rpcCall("extensionApi.DMT_EditorControl.generateIndicatorMarkers", markers, color?, lineWidth?, zoom?, tabId?)
rpcCall("extensionApi.DMT_EditorControl.removeIndicatorMarkers", tabId?)
```

### DMT_Folder
```
rpcCall("extensionApi.DMT_Folder.createFolder", name, teamUuid, parentFolderUuid?, desc?)
rpcCall("extensionApi.DMT_Folder.modifyFolderName", teamUuid, folderUuid, name)
rpcCall("extensionApi.DMT_Folder.modifyFolderDescription", teamUuid, folderUuid, desc?)
rpcCall("extensionApi.DMT_Folder.moveFolderToFolder", teamUuid, folderUuid, parentFolderUuid?)
rpcCall("extensionApi.DMT_Folder.getAllFoldersUuid", teamUuid)
rpcCall("extensionApi.DMT_Folder.getFolderInfo", teamUuid, folderUuid)
rpcCall("extensionApi.DMT_Folder.deleteFolder", teamUuid, folderUuid)
```

### DMT_Team, DMT_Workspace, DMT_SelectControl
```
rpcCall("extensionApi.DMT_Team.getAllTeamsInfo")
rpcCall("extensionApi.DMT_Team.getAllInvolvedTeamInfo")
rpcCall("extensionApi.DMT_Team.getCurrentTeamInfo")
rpcCall("extensionApi.DMT_Workspace.getAllWorkspacesInfo")
rpcCall("extensionApi.DMT_Workspace.toggleToWorkspace", workspaceUuid?)
rpcCall("extensionApi.DMT_Workspace.getCurrentWorkspaceInfo")
rpcCall("extensionApi.DMT_SelectControl.getCurrentDocumentInfo")
```

---

## PCB 文档操作 (PCB_Document)

```
rpcCall("extensionApi.PCB_Document.save", uuid)
rpcCall("extensionApi.PCB_Document.importChanges", uuid?)
rpcCall("extensionApi.PCB_Document.importAutoRouteJsonFile", file: File)
rpcCall("extensionApi.PCB_Document.importAutoLayoutJsonFile", file: File)
rpcCall("extensionApi.PCB_Document.getRatlineStatus")
rpcCall("extensionApi.PCB_Document.setRatlineStatus", status)  // "active"|"inactive"
rpcCall("extensionApi.PCB_Document.convertCanvasOriginToDataOrigin", x, y)
rpcCall("extensionApi.PCB_Document.convertDataOriginToCanvasOrigin", x, y)
rpcCall("extensionApi.PCB_Document.getCanvasOrigin")
rpcCall("extensionApi.PCB_Document.setCanvasOrigin", offsetX, offsetY)
rpcCall("extensionApi.PCB_Document.navigateToCoordinates", x, y)
rpcCall("extensionApi.PCB_Document.navigateToRegion", left, right, top, bottom)
rpcCall("extensionApi.PCB_Document.getPrimitiveAtPoint", x, y)
rpcCall("extensionApi.PCB_Document.getPrimitivesInRegion", left, right, top, bottom, leftToRight?)
rpcCall("extensionApi.PCB_Document.zoomToBoardOutline")
rpcCall("extensionApi.PCB_Document.getCurrentFilterConfiguration")
rpcCall("extensionApi.PCB_Document.getAllPCBTabData")
```

---

## PCB 图元 — 通用 (PCB_Primitive)

```
rpcCall("extensionApi.PCB_Primitive.getPrimitivesBBox", primitiveIds[])
```

---

## PCB 图元 — 走线 (PCB_PrimitiveLine)

```typescript
// create
rpcCall("extensionApi.PCB_PrimitiveLine.create",
  net: string,        // 网络名 "GND"
  layer: TPCB_LayersOfLine,  // EPCB_LayerId.TOP=1
  startX: number,     // mil
  startY: number,
  endX: number,
  endY: number,
  lineWidth?: number, // mil, default 10
  primitiveLock?: boolean
) => IPCB_PrimitiveLine

// delete
rpcCall("extensionApi.PCB_PrimitiveLine.delete", ids: string|IPCB_PrimitiveLine|string[]|IPCB_PrimitiveLine[])

// get (single)
rpcCall("extensionApi.PCB_PrimitiveLine.getByIds", id: string) => IPCB_PrimitiveLine | undefined
// get (batch)
rpcCall("extensionApi.PCB_PrimitiveLine.getByIds", ids: string[]) => IPCB_PrimitiveLine[]

// getAll (filtered)
rpcCall("extensionApi.PCB_PrimitiveLine.getAll", net?: string, layer?, primitiveLock?: boolean)

// getAllPrimitiveId
rpcCall("extensionApi.PCB_PrimitiveLine.getAll")  // returns string[]

// modify
rpcCall("extensionApi.PCB_PrimitiveLine.modify", id: string|IPCB_PrimitiveLine, property: {
  net?, layer?, startX?, startY?, endX?, endY?, lineWidth?, primitiveLock?
})

// getAdjacentPrimitives
rpcCall("extensionApi.PCB_PrimitiveLine.getAdjacentPrimitives", id) => (IPCB_PrimitiveLine|IPCB_PrimitiveVia|IPCB_PrimitiveArc)[]

// getEntireTrack (整个网络的走线)
rpcCall("extensionApi.PCB_PrimitiveLine.getEntireTrack", id, includeVias?: boolean) => IPCB_PrimitiveLine[]
```

---

## PCB 图元 — 圆弧走线 (PCB_PrimitiveArc)

```typescript
rpcCall("extensionApi.PCB_PrimitiveArc.create",
  net, layer, startX, startY, endX, endY, arcAngle: number,
  lineWidth?, interactiveMode?: EPCB_PrimitiveArcInteractiveMode, primitiveLock?)
rpcCall("extensionApi.PCB_PrimitiveArc.delete", ids)
rpcCall("extensionApi.PCB_PrimitiveArc.getByIds", ids)  // single/batch
rpcCall("extensionApi.PCB_PrimitiveArc.getAll", net?, layer?, primitiveLock?)
rpcCall("extensionApi.PCB_PrimitiveArc.modify", id, property: {
  net?, layer?, startX?, startY?, endX?, endY?, arcAngle?, lineWidth?, interactiveMode?, primitiveLock?
})
rpcCall("extensionApi.PCB_PrimitiveArc.getAdjacentPrimitives", id)
rpcCall("extensionApi.PCB_PrimitiveArc.getEntireTrack", id)
```

---

## PCB 图元 — 过孔 (PCB_PrimitiveVia)

```typescript
rpcCall("extensionApi.PCB_PrimitiveVia.create",
  net: string, x, y,
  holeDiameter: number,     // 内径 mil
  diameter: number,          // 外径 mil
  viaType?: EPCB_PrimitiveViaType,  // VIA=0, BLIND=1, SUTURE=2
  designRuleBlindViaName?: string|null,
  solderMaskExpansion?: IPCB_PrimitiveSolderMaskAndPasteMaskExpansion|null,
  primitiveLock?: boolean
) => IPCB_PrimitiveVia

rpcCall("extensionApi.PCB_PrimitiveVia.delete", ids)
rpcCall("extensionApi.PCB_PrimitiveVia.getByIds", ids)
rpcCall("extensionApi.PCB_PrimitiveVia.getAll", net?, primitiveLock?)
rpcCall("extensionApi.PCB_PrimitiveVia.modify", id, property: {
  net?, x?, y?, holeDiameter?, diameter?, viaType?, designRuleBlindViaName?,
  solderMaskExpansion?, primitiveLock?
})
rpcCall("extensionApi.PCB_PrimitiveVia.getAdjacentPrimitives", id)
```

---

## PCB 图元 — 焊盘 (PCB_PrimitivePad)

```typescript
rpcCall("extensionApi.PCB_PrimitivePad.create",
  layer: TPCB_LayersOfPad,
  x, y, 
  pad: TPCB_PrimitivePadShape,     // {shape: ELLIPSE|RECT|OVAL|POLYGON, width, height}
  padNumber: string,                // "1"
  net?: string,
  padType?: EPCB_PrimitivePadType,  // NORMAL=0, TEST=1, MARK_POINT=2
  hole?: TPCB_PrimitivePadHole,     // {shape: ROUND|RECT|SLOT, width, height}
  holeOffsetX?, holeOffsetY?, holeRotation?,
  metallization?: boolean,
  specialPad?: TPCB_PrimitiveSpecialPadShape,
  solderMaskAndPasteMaskExpansion?: IPCB_PrimitiveSolderMaskAndPasteMaskExpansion|null,
  heatWelding?: IPCB_PrimitivePadHeatWelding,
  rotation?: number,
  primitiveLock?: boolean
) => IPCB_PrimitivePad

rpcCall("extensionApi.PCB_PrimitivePad.delete", ids)
rpcCall("extensionApi.PCB_PrimitivePad.getByIds", ids)
rpcCall("extensionApi.PCB_PrimitivePad.getAll", layer?, net?, primitiveLock?, padType?)
rpcCall("extensionApi.PCB_PrimitivePad.modify", id, property: { /* all of the above as optional */ })
rpcCall("extensionApi.PCB_PrimitivePad.getConnectedPrimitives", id)
```

---

## PCB 图元 — 器件 (PCB_PrimitiveComponent)

```typescript
rpcCall("extensionApi.PCB_PrimitiveComponent.create",
  component: {libraryUuid, uuid} | ILIB_DeviceItem | ILIB_DeviceSearchItem
           | {libraryType: ELIB_LibraryType.FOOTPRINT, libraryUuid, uuid}
           | ILIB_FootprintItem | ILIB_FootprintSearchItem,
  layer: TPCB_LayersOfComponent,  // TOP=1, BOTTOM=2
  x, y,
  rotation?: number,
  primitiveLock?: boolean
) => IPCB_PrimitiveComponent

rpcCall("extensionApi.PCB_PrimitiveComponent.delete", ids)
rpcCall("extensionApi.PCB_PrimitiveComponent.getByIds", ids)
rpcCall("extensionApi.PCB_PrimitiveComponent.getAll", layer?, primitiveLock?)
rpcCall("extensionApi.PCB_PrimitiveComponent.getAllPinsByPrimitiveId", id) => IPCB_PrimitiveComponentPad[]
rpcCall("extensionApi.PCB_PrimitiveComponent.placeComponentWithMouse", component)
rpcCall("extensionApi.PCB_PrimitiveComponent.setAttribute", id, key, value)
rpcCall("extensionApi.PCB_PrimitiveComponent.modify", id, property: {
  // component?, layer?, x?, y?, rotation?, primitiveLock?
})
```

---

## PCB 图元 — 覆铜边框 (PCB_PrimitivePour)

```typescript
rpcCall("extensionApi.PCB_PrimitivePour.create",
  net: string,
  layer: TPCB_LayersOfCopper,   // TOP=1, BOTTOM=2, INNER_1..32
  complexPolygon: IPCB_Polygon,
  pourFillMethod?: EPCB_PrimitivePourFillMethod,  // "solid"|"45grid"|"90grid"
  preserveSilos?: boolean,       // 保留孤岛
  pourName?: string,
  pourPriority?: number,
  lineWidth?: number,
  primitiveLock?: boolean
) => IPCB_PrimitivePour

rpcCall("extensionApi.PCB_PrimitivePour.delete", ids)
rpcCall("extensionApi.PCB_PrimitivePour.getByIds", ids)
rpcCall("extensionApi.PCB_PrimitivePour.getAll", net?, layer?, primitiveLock?)
rpcCall("extensionApi.PCB_PrimitivePour.modify", id, property: {
  net?, layer?, complexPolygon?, pourFillMethod?, preserveSilos?,
  pourName?, pourPriority?, lineWidth?, primitiveLock?
})
rpcCall("extensionApi.PCB_PrimitivePour.getCopperRegion", id)
rpcCall("extensionApi.PCB_PrimitivePour.rebuildCopperRegion", id)
rpcCall("extensionApi.PCB_PrimitivePour.convertToType", id, targetType)  // → Fill/Polyline/Region
```

---

## PCB 图元 — 覆铜填充 (PCB_PrimitivePoured)

```typescript
rpcCall("extensionApi.PCB_PrimitivePoured.delete", ids)
rpcCall("extensionApi.PCB_PrimitivePoured.getAll")
rpcCall("extensionApi.PCB_PrimitivePoured.convertToFill", pourFillId)
rpcCall("extensionApi.PCB_PrimitivePoured.addSolderMaskFill", pourFillId)
rpcCall("extensionApi.PCB_PrimitivePoured.deletePourFills", pourFillIds)
```

---

## PCB 图元 — 填充 (PCB_PrimitiveFill)

```typescript
rpcCall("extensionApi.PCB_PrimitiveFill.create",
  layer: TPCB_LayersOfFill,
  complexPolygon: IPCB_Polygon,
  net?: string,
  fillMode?: EPCB_PrimitiveFillMode,  // SOLID=0, MESH=1, INNER_ELECTRICAL_LAYER=2
  lineWidth?: number,
  primitiveLock?: boolean
) => IPCB_PrimitiveFill

rpcCall("extensionApi.PCB_PrimitiveFill.delete", ids)
rpcCall("extensionApi.PCB_PrimitiveFill.getByIds", ids)
rpcCall("extensionApi.PCB_PrimitiveFill.getAll", layer?, net?, primitiveLock?)
rpcCall("extensionApi.PCB_PrimitiveFill.modify", id, property: {
  layer?, complexPolygon?, net?, fillMode?, lineWidth?, primitiveLock?
})
rpcCall("extensionApi.PCB_PrimitiveFill.convertToType", id, targetType)  // → Polyline/Pour/Region
```

---

## PCB 图元 — 折线 (PCB_PrimitivePolyline)

```typescript
rpcCall("extensionApi.PCB_PrimitivePolyline.create",
  net, layer: TPCB_LayersOfLine,
  polygon: IPCB_Polygon,
  lineWidth?, primitiveLock?
) => IPCB_PrimitivePolyline

rpcCall("extensionApi.PCB_PrimitivePolyline.delete/getByIds/getAll/modify", ...)
rpcCall("extensionApi.PCB_PrimitivePolyline.convertToType", id, targetType)  // → Fill/Pour/Region
```

---

## PCB 图元 — 区域 (PCB_PrimitiveRegion)

```typescript
rpcCall("extensionApi.PCB_PrimitiveRegion.create",
  layer: TPCB_LayersOfRegion,
  complexPolygon: IPCB_Polygon,
  ruleType?: EPCB_PrimitiveRegionRuleType[],  // NO_COMPONENTS=2,NO_VIAS=3,NO_WIRES=5,NO_FILLS=6,NO_POURS=7,NO_INNER_ELECTRICAL_LAYERS=8,FOLLOW_REGION_RULE=9
  regionName?: string,
  lineWidth?: number,
  primitiveLock?: boolean
) => IPCB_PrimitiveRegion

rpcCall("extensionApi.PCB_PrimitiveRegion.delete/getByIds/getAll/modify", ...)
rpcCall("extensionApi.PCB_PrimitiveRegion.convertToType", id, targetType)  // → Fill/Polyline/Pour
```

---

## PCB 图元 — 文本 (PCB_PrimitiveString)

```typescript
rpcCall("extensionApi.PCB_PrimitiveString.create",
  layer: TPCB_LayersOfImage,
  x, y,
  text: string,
  fontFamily?: string,
  fontSize?: number,
  lineWidth?: number,
  alignMode?: EPCB_PrimitiveStringAlignMode,  // LEFT_TOP=1..RIGHT_BOTTOM=9
  rotation?: number,
  reverse?: boolean,
  expansion?: number,
  mirror?: boolean,
  primitiveLock?: boolean
) => IPCB_PrimitiveString

rpcCall("extensionApi.PCB_PrimitiveString.delete/getByIds/getAll/modify", ...)
```

---

## PCB 图元 — 尺寸标注 (PCB_PrimitiveDimension)

```typescript
rpcCall("extensionApi.PCB_PrimitiveDimension.create",
  dimensionType: EPCB_PrimitiveDimensionType,  // LENGTH, RADIUS, ANGLE
  coordinateSet: TPCB_PrimitiveDimensionCoordinateSet,
  layer?: TPCB_LayersOfDimension,
  unit?: ESYS_Unit,  // MILLIMETER|CENTIMETER|INCH|MIL
  lineWidth?: number,
  precision?: number,  // 小数位数
  primitiveLock?: boolean
) => IPCB_PrimitiveDimension

rpcCall("extensionApi.PCB_PrimitiveDimension.delete/getByIds/getAll/modify", ...)
```

---

## PCB 图元 — 图像 (PCB_PrimitiveImage)

```typescript
rpcCall("extensionApi.PCB_PrimitiveImage.create",
  x, y,
  complexPolygon: TPCB_PolygonSourceArray | IPCB_Polygon | IPCB_ComplexPolygon,
  layer: TPCB_LayersOfImage,
  width?, height?, rotation?, horizonMirror?, primitiveLock?
) => IPCB_PrimitiveImage

rpcCall("extensionApi.PCB_PrimitiveImage.delete/getByIds/getAll/modify", ...)
```

---

## PCB 图元 — 内嵌对象 (PCB_PrimitiveObject)

```typescript
rpcCall("extensionApi.PCB_PrimitiveObject.create",
  layer: TPCB_LayersOfObject,
  topLeftX, topLeftY,
  binaryData: string,  // base64?
  width, height,
  rotation?, mirror?, fileName?, primitiveLock?
) => IPCB_PrimitiveObject

rpcCall("extensionApi.PCB_PrimitiveObject.delete/getByIds/getAll/modify", ...)
```

---

## PCB 图元 — 属性 (PCB_PrimitiveAttribute)

```typescript
// 只读: 删除/查询/修改属性图元
rpcCall("extensionApi.PCB_PrimitiveAttribute.delete", ids)
rpcCall("extensionApi.PCB_PrimitiveAttribute.getByIds", ids)
rpcCall("extensionApi.PCB_PrimitiveAttribute.getAll", parentPrimitiveId?)
rpcCall("extensionApi.PCB_PrimitiveAttribute.modify", id, property: { /* key?, value?, visible?... */ })
```

---

## PCB 选择控制 (PCB_SelectControl)

```typescript
rpcCall("extensionApi.PCB_SelectControl.getAllSelectedPrimitiveIds") => string[]
rpcCall("extensionApi.PCB_SelectControl.getAllSelectedPrimitives") => IPCB_Primitive[]
rpcCall("extensionApi.PCB_SelectControl.doSelectPrimitives", ids: string|string[])
rpcCall("extensionApi.PCB_SelectControl.doCrossProbeSelect", components?, pins?, nets?, highlight?, select?)
rpcCall("extensionApi.PCB_SelectControl.doCrossProbeSelectByObject", object)
rpcCall("extensionApi.PCB_SelectControl.clearSelected")
rpcCall("extensionApi.PCB_SelectControl.getCurrentMousePosition") => {x,y}|undefined
```

---

## PCB 图层 (PCB_Layer)

```typescript
rpcCall("extensionApi.PCB_Layer.selectLayerById", layerId)  // EPCB_LayerId.TOP=1, BOTTOM=2...
rpcCall("extensionApi.PCB_Layer.setLayerStatus", layerId, visible?, active?)
// setLayerStatus variants for: visible/invisible/active/inactive/locked/unlocked
rpcCall("extensionApi.PCB_Layer.setTheNumberOfCopperLayers", n: 2|4|6|8|10|12|14|16|18|20|22|24|26|28|30|32)
rpcCall("extensionApi.PCB_Layer.getTheNumberOfCopperLayers")
rpcCall("extensionApi.PCB_Layer.setLayerColorConfiguration", config: EPCB_LayerColorConfiguration)
rpcCall("extensionApi.PCB_Layer.setInactiveLayerTransparency", transparency: number) // 0-100
rpcCall("extensionApi.PCB_Layer.setInactiveLayerDisplayMode", mode: EPCB_InactiveLayerDisplayMode)
rpcCall("extensionApi.PCB_Layer.setPcbType", type: EPCB_PcbPlateType)  // NORMAL=1, FPC=2
rpcCall("extensionApi.PCB_Layer.addCustomLayer") => TPCB_LayersOfCustom | undefined
rpcCall("extensionApi.PCB_Layer.removeLayer", layer: TPCB_LayersOfCustom)
rpcCall("extensionApi.PCB_Layer.modifyLayer", layer, property: {name?, type?, color?, transparency?})
rpcCall("extensionApi.PCB_Layer.getAllLayers") => IPCB_LayerItem[]
```

---

## PCB 网络 (PCB_Net)

```typescript
rpcCall("extensionApi.PCB_Net.getAllNets") => IPCB_NetInfo[]
rpcCall("extensionApi.PCB_Net.getNet", netName) => IPCB_NetInfo|undefined
rpcCall("extensionApi.PCB_Net.getAllNetName") => string[]
rpcCall("extensionApi.PCB_Net.getNetLength", netName) => number|undefined
rpcCall("extensionApi.PCB_Net.setNetColor", netName, color)
rpcCall("extensionApi.PCB_Net.getAllPrimitivesByNet", netName, primitiveTypes?: EPCB_PrimitiveType[])
rpcCall("extensionApi.PCB_Net.selectNet", netName)
rpcCall("extensionApi.PCB_Net.unselectNet", netName)
rpcCall("extensionApi.PCB_Net.unselectAllNets")
rpcCall("extensionApi.PCB_Net.highlightNet", netName)
rpcCall("extensionApi.PCB_Net.unhighlightNet", netName)
rpcCall("extensionApi.PCB_Net.unhighlightAllNets")
rpcCall("extensionApi.PCB_Net.getNetlist", type?: ESYS_NetlistType) => string
rpcCall("extensionApi.PCB_Net.setNetlist", type, netlist: string)
```

---

## PCB DRC (PCB_Drc)

```typescript
// DRC 检查
rpcCall("extensionApi.PCB_Drc.check", strict: boolean, userInterface: boolean, includeVerboseError: boolean)

// 规则配置
rpcCall("extensionApi.PCB_Drc.getCurrentRuleConfigurationName") => string|undefined
rpcCall("extensionApi.PCB_Drc.getCurrentRuleConfiguration") => {[key:string]:any}|undefined
rpcCall("extensionApi.PCB_Drc.getRuleConfiguration", name) => {[key:string]:any}|undefined
rpcCall("extensionApi.PCB_Drc.getAllRuleConfigurations", includeSystem?: boolean)
rpcCall("extensionApi.PCB_Drc.saveRuleConfiguration", config, name, allowOverwrite?)
rpcCall("extensionApi.PCB_Drc.renameRuleConfiguration", oldName, newName)
rpcCall("extensionApi.PCB_Drc.deleteRuleConfiguration", name)
rpcCall("extensionApi.PCB_Drc.getDefaultRuleConfigurationName")
rpcCall("extensionApi.PCB_Drc.setAsDefaultRuleConfiguration", name)

// 网络规则
rpcCall("extensionApi.PCB_Drc.getNetRules")
rpcCall("extensionApi.PCB_Drc.overwriteNetRules", rules)
rpcCall("extensionApi.PCB_Drc.getNetByNetRules")
rpcCall("extensionApi.PCB_Drc.overwriteNetByNetRules", rules)
rpcCall("extensionApi.PCB_Drc.getRegionRules")
rpcCall("extensionApi.PCB_Drc.overwriteRegionRules", rules)

// 网络类
rpcCall("extensionApi.PCB_Drc.createNetClass", name, nets: string[], color)
rpcCall("extensionApi.PCB_Drc.deleteNetClass", name)
rpcCall("extensionApi.PCB_Drc.modifyNetClassName", oldName, newName)
rpcCall("extensionApi.PCB_Drc.addNetToNetClass", className, net: string|string[])
rpcCall("extensionApi.PCB_Drc.removeNetFromNetClass", className, net: string|string[])
rpcCall("extensionApi.PCB_Drc.getAllNetClasses") => IPCB_NetClassItem[]

// 差分对
rpcCall("extensionApi.PCB_Drc.createDifferentialPair", name, positiveNet, negativeNet)
rpcCall("extensionApi.PCB_Drc.deleteDifferentialPair", name)
rpcCall("extensionApi.PCB_Drc.modifyDifferentialPairName", oldName, newName)
rpcCall("extensionApi.PCB_Drc.modifyDifferentialPairPositiveNet", name, net)
rpcCall("extensionApi.PCB_Drc.modifyDifferentialPairNegativeNet", name, net)
rpcCall("extensionApi.PCB_Drc.getAllDifferentialPairs") => IPCB_DifferentialPairItem[]

// 等长网络组
rpcCall("extensionApi.PCB_Drc.createEqualLengthNetGroup", name, nets: string[], color)
rpcCall("extensionApi.PCB_Drc.deleteEqualLengthNetGroup", name)
rpcCall("extensionApi.PCB_Drc.modifyEqualLengthNetGroupName", oldName, newName)
rpcCall("extensionApi.PCB_Drc.addNetToEqualLengthNetGroup", groupName, net: string|string[])
rpcCall("extensionApi.PCB_Drc.removeNetFromEqualLengthNetGroup", groupName, net: string|string[])
rpcCall("extensionApi.PCB_Drc.getAllEqualLengthNetGroups") => IPCB_EqualLengthNetGroupItem[]

// 焊盘对组
rpcCall("extensionApi.PCB_Drc.createPadPairGroup", name, padPairs: [string,string][])
rpcCall("extensionApi.PCB_Drc.deletePadPairGroup", name)
rpcCall("extensionApi.PCB_Drc.modifyPadPairGroupName", oldName, newName)
rpcCall("extensionApi.PCB_Drc.addPadPairToPadPairGroup", groupName, padPair: [string,string]|[string,string][])
rpcCall("extensionApi.PCB_Drc.removePadPairFromPadPairGroup", groupName, padPair)
rpcCall("extensionApi.PCB_Drc.getAllPadPairGroups") => IPCB_PadPairGroupItem[]
rpcCall("extensionApi.PCB_Drc.getPadPairGroupMinWireLength", groupName)
```

---

## PCB 制造输出 (PCB_ManufactureData)

```typescript
rpcCall("extensionApi.PCB_ManufactureData.getGerberFile", fileName?, silk?, unit?, format?, drilledHole?, layerConfigs?, objectConfigs?)
rpcCall("extensionApi.PCB_ManufactureData.get3DFile", fileName?, fileType?:'step'|'obj', elements?, modelMode?:'Outfit'|'Parts', autoGen?)
rpcCall("extensionApi.PCB_ManufactureData.get3DShellFile", fileName?, fileType?:'stl'|'step'|'obj')
rpcCall("extensionApi.PCB_ManufactureData.getPickAndPlaceFile", fileName?, fileType?:'xlsx'|'csv', unit?)
rpcCall("extensionApi.PCB_ManufactureData.getFlyingProbeTestFile", fileName?)
rpcCall("extensionApi.PCB_ManufactureData.getBomTemplates") => string[]
rpcCall("extensionApi.PCB_ManufactureData.uploadBomTemplateFile", file, template?)
rpcCall("extensionApi.PCB_ManufactureData.getBomTemplateFile", template)
rpcCall("extensionApi.PCB_ManufactureData.deleteBomTemplate", template)
rpcCall("extensionApi.PCB_ManufactureData.getBomDefaultParams")
rpcCall("extensionApi.PCB_ManufactureData.getBomFile", fileName?, fileType?:'xlsx'|'csv', template?, filterOptions?, statistics?, property?, columns?)
rpcCall("extensionApi.PCB_ManufactureData.getTestPointFile", fileName?, fileType?:'xlsx'|'csv')
rpcCall("extensionApi.PCB_ManufactureData.getNetlistFile", fileName?, netlistType?)
rpcCall("extensionApi.PCB_ManufactureData.getDxfFile", fileName?, layers?, objects?)
rpcCall("extensionApi.PCB_ManufactureData.getPDFFile", fileName?, outputMethod?, ...)
rpcCall("extensionApi.PCB_ManufactureData.getIPCFile", fileName?)  // IPC-D-356A
rpcCall("extensionApi.PCB_ManufactureData.getOpenDatabaseDoublePlusFile", fileName?, unit?, otherData?, layers?, objects?)
rpcCall("extensionApi.PCB_ManufactureData.getInteractiveBomFile")
rpcCall("extensionApi.PCB_ManufactureData.getDsnFile", fileName?)
rpcCall("extensionApi.PCB_ManufactureData.getAutoRouteJson", fileName?)
rpcCall("extensionApi.PCB_ManufactureData.getAutoLayoutJson", fileName?)
rpcCall("extensionApi.PCB_ManufactureData.getADFile", fileName?)
rpcCall("extensionApi.PCB_ManufactureData.getPADSFile", fileName?)
rpcCall("extensionApi.PCB_ManufactureData.getPcbInfoFile", fileName?)
rpcCall("extensionApi.PCB_ManufactureData.getOrderComponents")
rpcCall("extensionApi.PCB_ManufactureData.getOrderSMT")
rpcCall("extensionApi.PCB_ManufactureData.getOrderPCB")
rpcCall("extensionApi.PCB_ManufactureData.getOrder3DShellFile")
rpcCall("extensionApi.PCB_ManufactureData.getManufactureData")  // 全部制造数据打包
rpcCall("extensionApi.PCB_ManufactureData.updateAutoRouteRule", rule)
```

---

## PCB 事件 (PCB_Event)

```typescript
rpcCall("extensionApi.PCB_Event.mouseEvent", id, eventType, callback, onlyOnce?)
rpcCall("extensionApi.PCB_Event.primitiveEvent", id, eventType, callback, onlyOnce?)
rpcCall("extensionApi.PCB_Event.netEvent", id, eventType, callback, onlyOnce?)
rpcCall("extensionApi.PCB_Event.crossProbeSelectEvent", id, callback)
```

---

## PCB 多边形数学 (PCB_MathPolygon)

```typescript
rpcCall("extensionApi.PCB_MathPolygon.convertImageToComplexPolygon", blob: Blob, width, height, tolerance?, simplification?, smoothing?, despeckling?, whiteAsBg?, inversion?)
rpcCall("extensionApi.PCB_MathPolygon.getCenter", polygon)
```

---

## 原理图文档 (SCH_Document)

```typescript
rpcCall("extensionApi.SCH_Document.save")
rpcCall("extensionApi.SCH_Document.importChanges")
rpcCall("extensionApi.SCH_Document.autoRouting", data)    // 参数未文档化
rpcCall("extensionApi.SCH_Document.autoLayout", data)      // 参数未文档化
```

---

## 原理图图元 — 通用 (SCH_Primitive)

```typescript
rpcCall("extensionApi.SCH_Primitive.getPrimitiveTypeByPrimitiveId", id)
rpcCall("extensionApi.SCH_Primitive.getPrimitivesBBox", ids)
```

---

## 原理图图元 — 导线 (SCH_PrimitiveWire)

```typescript
rpcCall("extensionApi.SCH_PrimitiveWire.create",
  line: number[] | number[][],  // 路径点 [x1,y1,x2,y2,...] 或 [[x1,y1],[x2,y2]]
  net?: string,
  color?: string|null,
  lineWidth?: number|null,
  lineType?: ESCH_PrimitiveLineType|null  // SOLID=0,DASHED=1,DOTTED=2,DOT_DASHED=3
) => ISCH_PrimitiveWire

rpcCall("extensionApi.SCH_PrimitiveWire.delete/get/getAll/modify", ...)
```

---

## 原理图图元 — 器件 (SCH_PrimitiveComponent)

```typescript
rpcCall("extensionApi.SCH_PrimitiveComponent.create",
  component: {libraryUuid, uuid} | ILIB_DeviceItem | ILIB_DeviceSearchItem,
  x, y,          // 0.01inch 单位!
  subPartName?: string,
  rotation?: number,  // 度数
  mirror?: boolean,
  addIntoBom?: boolean,
  addIntoPcb?: boolean
) => ISCH_PrimitiveComponent

// 网络标识符快捷创建
rpcCall("extensionApi.SCH_PrimitiveComponent.createNetFlag", identification: 'Power'|'Ground'|'AnalogGround'|'ProtectGround', net, x, y, rotation?, mirror?)
rpcCall("extensionApi.SCH_PrimitiveComponent.createNetPort", direction: 'IN'|'OUT'|'BI', net, x, y, rotation?, mirror?)
rpcCall("extensionApi.SCH_PrimitiveComponent.createShortCircuitFlag", x, y, rotation?, mirror?)

rpcCall("extensionApi.SCH_PrimitiveComponent.delete/get/getAll/getAllPrimitiveId/modify/placeComponentWithMouse/getAllPinsByPrimitiveId/getDeviceData", ...)
rpcCall("extensionApi.SCH_PrimitiveComponent.getAllPropertyNames") => string[]

// 设置默认的 Power/Ground/Port 器件 UUID
rpcCall("extensionApi.SCH_PrimitiveComponent.setNetFlagComponentUuid_Power", component)
rpcCall("extensionApi.SCH_PrimitiveComponent.setNetFlagComponentUuid_Ground", component)
rpcCall("extensionApi.SCH_PrimitiveComponent.setNetFlagComponentUuid_AnalogGround", component)
rpcCall("extensionApi.SCH_PrimitiveComponent.setNetFlagComponentUuid_ProtectGround", component)
rpcCall("extensionApi.SCH_PrimitiveComponent.setNetPortComponentUuid_IN", component)
rpcCall("extensionApi.SCH_PrimitiveComponent.setNetPortComponentUuid_OUT", component)
rpcCall("extensionApi.SCH_PrimitiveComponent.setNetPortComponentUuid_BI", component)
```

---

## 原理图图元 — 复用模块器件 (SCH_PrimitiveComponent3)

```typescript
rpcCall("extensionApi.SCH_PrimitiveComponent3.create", ...)
rpcCall("extensionApi.SCH_PrimitiveComponent3.delete/get/getAll/getAllPrimitiveId/modify/placeComponentWithMouse/getDeviceData", ...)
rpcCall("extensionApi.SCH_PrimitiveComponent3.createCbbSymbol", ...)
rpcCall("extensionApi.SCH_PrimitiveComponent3.placeCbbSchematicPage", ...)
```

---

## 原理图图元 — 总线 (SCH_PrimitiveBus), 圆 (SCH_PrimitiveCircle), 圆弧 (SCH_PrimitiveArc)

```typescript
// 总线
rpcCall("extensionApi.SCH_PrimitiveBus.create", busName, line, color?, lineWidth?, lineType?)
// 圆
rpcCall("extensionApi.SCH_PrimitiveCircle.create", centerX, centerY, radius, color?, fillColor?, lineWidth?, lineType?, fillStyle?)
// 圆弧
rpcCall("extensionApi.SCH_PrimitiveArc.create", startX, startY, referenceX, referenceY, endX, endY, color?, fillColor?, lineWidth?, lineType?)
```

---

## 原理图图元 — 引脚 (SCH_PrimitivePin), 多边形 (SCH_PrimitivePolygon), 矩形 (SCH_PrimitiveRectangle), 文本 (SCH_PrimitiveText)

```typescript
// 引脚
rpcCall("extensionApi.SCH_PrimitivePin.create", x, y, pinNumber, pinName?, rotation?, pinLength?, pinColor?, pinShape?: ESCH_PrimitivePinShape, pinType?: ESCH_PrimitivePinType)
// 多边形
rpcCall("extensionApi.SCH_PrimitivePolygon.create", line: number[], color?, fillColor?, lineWidth?, lineType?)
// 矩形
rpcCall("extensionApi.SCH_PrimitiveRectangle.create", topLeftX, topLeftY, width, height, cornerRadius?, rotation?, color?, fillColor?, lineWidth?, lineType?, fillStyle?)
// 文本
rpcCall("extensionApi.SCH_PrimitiveText.create", x, y, content, rotation?, textColor?, fontName?, fontSize?, bold?, italic?, underLine?, alignMode?)
```

---

## 原理图 — 选择控制、DRC、网表、事件、仿真、制造输出

```typescript
// 选择控制
rpcCall("extensionApi.SCH_SelectControl.getAllSelectedPrimitives_PrimitiveId")
rpcCall("extensionApi.SCH_SelectControl.getAllSelectedPrimitives")
rpcCall("extensionApi.SCH_SelectControl.getSelectedPrimitives_PrimitiveId")
rpcCall("extensionApi.SCH_SelectControl.getSpecifiedPrimitive", ...)
rpcCall("extensionApi.SCH_SelectControl.getObjParams")
rpcCall("extensionApi.SCH_SelectControl.doSelectPrimitives", ids)
rpcCall("extensionApi.SCH_SelectControl.getCurrentMousePosition")

// DRC
rpcCall("extensionApi.SCH_Drc.check", strict, userInterface, includeVerboseError)

// 网表
rpcCall("extensionApi.SCH_Netlist.getNetlist", type?) => string
rpcCall("extensionApi.SCH_Netlist.setNetlist", type, netlist)

// 事件
rpcCall("extensionApi.SCH_Event.mouseEvent", id, eventType, callback, onlyOnce?)
rpcCall("extensionApi.SCH_Event.primitiveEvent", id, eventType, callback, onlyOnce?)

// 制造输出
rpcCall("extensionApi.SCH_ManufactureData.getAssemblyVariantsConfigs")
rpcCall("extensionApi.SCH_ManufactureData.getBomFile", fileName?, fileType?, template?, filterOptions?, statistics?, property?, columns?, variantConfig?)
rpcCall("extensionApi.SCH_ManufactureData.getNetlistFile", fileName?, netlistType?)
rpcCall("extensionApi.SCH_ManufactureData.getExportDocumentFile", ...)
rpcCall("extensionApi.SCH_ManufactureData.placeComponentsOrder", interactive?, ignoreWarning?)
rpcCall("extensionApi.SCH_ManufactureData.placeSmtComponentsOrder", interactive?, ignoreWarning?)
```

---

## 面板 (PNL_Document)

```typescript
rpcCall("extensionApi.PNL_Document.save")
```

---

## 库 — 库列表 (LIB_LibrariesList)

```typescript
rpcCall("extensionApi.LIB_LibrariesList.getSystemLibraryUuid")
rpcCall("extensionApi.LIB_LibrariesList.getPersonalLibraryUuid")
rpcCall("extensionApi.LIB_LibrariesList.getProjectLibraryUuid")
rpcCall("extensionApi.LIB_LibrariesList.getFavoriteLibraryUuid")
rpcCall("extensionApi.LIB_LibrariesList.getAllLibrariesList")
rpcCall("extensionApi.LIB_LibrariesList.registerExtendLibrary", libInfo)
```

---

## 库 — 器件 (LIB_Device)

```typescript
rpcCall("extensionApi.LIB_Device.create", libUuid, deviceName, classification?, association?, description?, property?)
rpcCall("extensionApi.LIB_Device.delete", deviceUuid, libUuid)
rpcCall("extensionApi.LIB_Device.modify", deviceUuid, libUuid, deviceName?, ...)
rpcCall("extensionApi.LIB_Device.get", deviceUuid, libUuid?)
rpcCall("extensionApi.LIB_Device.copy", deviceUuid, libUuid, targetLibUuid, targetClassification?, newName?)
rpcCall("extensionApi.LIB_Device.search", key, libUuid?, classification?, symbolType?, itemsOfPage?, page?)
rpcCall("extensionApi.LIB_Device.getByLcscIds", lcscIds: string|string[], libUuid?, allowMultiMatch?)
```

---

## 库 — 封装/符号/3D模型/复用模块/面板库

```typescript
// LIB_Footprint: create/delete/modify/get/copy/search/openInEditor/updateDocumentSource
// LIB_Symbol:    create/delete/modify/get/copy/search/openInEditor/updateDocumentSource/getRenderImage
// LIB_3DModel:   create/delete/modify/get/copy/search
// LIB_Cbb:       create/delete/modify/get/copy/search/openProjectInEditor/openSymbolInEditor
// LIB_PanelLibrary: create/delete/modify/get/copy/search/openInEditor
// LIB_Classification: createPrimary/createSecondary/getIndexByName/getNameByUuid/getAllClassificationTree/deleteByUuid/deleteByIndex
// LIB_SelectControl: getBottomComponentLibraryCurrentRowInfo
```

---

## 系统 — 对话框/消息 (SYS_MessageBox, SYS_Message, SYS_Dialog — 注意: MessageBox 是旧的, Message 是新 API)

```typescript
rpcCall("extensionApi.SYS_MessageBox.showInformationMessage", content, title?, buttonTitle?)
rpcCall("extensionApi.SYS_MessageBox.showConfirmationMessage", content, title?, mainButtonTitle?, buttonTitle?, callback?)
rpcCall("extensionApi.SYS_MessageBox.showInputDialog", ...)
rpcCall("extensionApi.SYS_MessageBox.showSelectDialog", options, beforeContent?, afterContent?, title?, defaultOption?, multiple?, callback?)
rpcCall("extensionApi.SYS_MessageBox.insertScriptToDialog", ...)

// 新 API
rpcCall("extensionApi.SYS_Message.showFollowMouseTip", tip, timeout?)
rpcCall("extensionApi.SYS_Message.removeFollowMouseTip", tip?)
```

---

## 系统 — 文件 (SYS_FileSystem, SYS_FileManager)

```typescript
rpcCall("extensionApi.SYS_FileSystem.openReadFileDialog", extensions?, multiFiles?) => File|File[]|undefined
rpcCall("extensionApi.SYS_FileSystem.saveFile", data: File|Blob, fileName?)
rpcCall("extensionApi.SYS_FileSystem.readFileFromFileSystem", uri) => File|undefined
rpcCall("extensionApi.SYS_FileSystem.saveFileToFileSystem", uri, data: File|Blob, fileName?, force?)
rpcCall("extensionApi.SYS_FileSystem.listFilesOfFileSystem", folderPath, recursive?) => ISYS_FileSystemFileList[]
rpcCall("extensionApi.SYS_FileSystem.deleteFileInFileSystem", uri, force?)
rpcCall("extensionApi.SYS_FileSystem.getEdaPath") => string
rpcCall("extensionApi.SYS_FileSystem.getDocumentsPath") => string
rpcCall("extensionApi.SYS_FileSystem.getLibrariesPaths") => string[]
rpcCall("extensionApi.SYS_FileSystem.getProjectsPaths") => string[]

// 文件管理器
rpcCall("extensionApi.SYS_FileManager.getProjectFile", fileName?, password?, fileType?:'epro'|'epro2')
rpcCall("extensionApi.SYS_FileManager.getDocumentFile", fileName?, password?, fileType?:'epro'|'epro2')
rpcCall("extensionApi.SYS_FileManager.getDocumentSource") => string|undefined
rpcCall("extensionApi.SYS_FileManager.setDocumentSource", source) => boolean
rpcCall("extensionApi.SYS_FileManager.getDeviceFileByDeviceUuid", uuid|uuid[], libUuid?, fileType?)
rpcCall("extensionApi.SYS_FileManager.getSymbolFileBySymbolUuid", uuid, libUuid?)
rpcCall("extensionApi.SYS_FileManager.getFootprintFileByFootprintUuid", uuid|uuid[], libUuid?, fileType?)
rpcCall("extensionApi.SYS_FileManager.getCbbFileByCbbUuid", uuid, libUuid?, props?)
rpcCall("extensionApi.SYS_FileManager.getPanelLibraryFileByPanelLibraryUuid", uuid|uuid[], libUuid?, fileType?)
rpcCall("extensionApi.SYS_FileManager.importProjectByProjectFile", ...)
rpcCall("extensionApi.SYS_FileManager.extractProjectInfo", file)
rpcCall("extensionApi.SYS_FileManager.extractLibInfo", file|file[])
```

---

## 系统 — 字体/格式转换/环境/快捷键

```typescript
rpcCall("extensionApi.SYS_FontManager.getFontsList") => string[]
rpcCall("extensionApi.SYS_FontManager.addFont", name)
rpcCall("extensionApi.SYS_FontManager.deleteFont", name)

rpcCall("extensionApi.SYS_FormatConversion.convertAltiumDesignerLibrariesToEasyEDASingleFile", file|file[])
rpcCall("extensionApi.SYS_FormatConversion.convertAltiumDesignerLibrariesToEasyEDAMultiFiles", file|file[])
rpcCall("extensionApi.SYS_FormatConversion.convertDisaLibrariesToEasyEDASingleFile", file|file[])
rpcCall("extensionApi.SYS_FormatConversion.convertDisaLibrariesToEasyEDAMultiFiles", file|file[])

rpcCall("extensionApi.SYS_Environment.setKeepProjectHasOnlyOneBoard", status)
rpcCall("extensionApi.SYS_Environment.getEditorCurrentVersion")
rpcCall("extensionApi.SYS_Environment.getEditorCompliedDate")
rpcCall("extensionApi.SYS_Environment.getUserInfo")

rpcCall("extensionApi.SYS_ShortcutKey.registerShortcutKey", key, title, callback, documentTypes?, scenes?)
rpcCall("extensionApi.SYS_ShortcutKey.unregisterShortcutKey", key)
rpcCall("extensionApi.SYS_ShortcutKey.getShortcutKeys", includeSystem?)
```

---

## 系统 — 面板/菜单/iframe/存储/日志/工具

```typescript
// 面板
rpcCall("extensionApi.SYS_PanelControl.openLeftPanel/closeLeftPanel/toggleLeftPanelLockState/isLeftPanelLocked", tab?)
rpcCall("extensionApi.SYS_PanelControl.openRightPanel/closeRightPanel/toggleRightPanelLockState/isRightPanelLocked", tab?)
rpcCall("extensionApi.SYS_PanelControl.openBottomPanel/closeBottomPanel/toggleBottomPanelLockState/isBottomPanelLocked", tab?)

// 菜单
rpcCall("extensionApi.SYS_HeaderMenu.insertSystemHeaderMenuItem", env, id, props)
rpcCall("extensionApi.SYS_HeaderMenu.removeSystemHeaderMenuItem", id, props?)
rpcCall("extensionApi.SYS_HeaderMenu.registerHeaderMenus", headerMenus)

// iframe
rpcCall("extensionApi.SYS_IFrame.openIFrame", htmlFileName, width?, height?, id?, props?)
rpcCall("extensionApi.SYS_IFrame.closeIFrame", id?)
rpcCall("extensionApi.SYS_IFrame.hideIFrame", id?)
rpcCall("extensionApi.SYS_IFrame.showIFrame", id?)

// 存储
rpcCall("extensionApi.SYS_Setting.restoreDefault")

// 日志
rpcCall("extensionApi.SYS_Log.add", msg, type?)
rpcCall("extensionApi.SYS_Log.clear")
rpcCall("extensionApi.SYS_Log.export", types?)
rpcCall("extensionApi.SYS_Log.sort", types?)
rpcCall("extensionApi.SYS_Log.find", msg, types?)

// 工具
rpcCall("extensionApi.SYS_Tool.netlistComparison", netlist1, netlist2)

// 窗口
rpcCall("extensionApi.SYS_Window.openUI", uiName, args?)
rpcCall("extensionApi.SYS_Window.getCurrentTheme")

// 加载/进度条
rpcCall("extensionApi.SYS_LoadingAndProgressBar.showProgressBar", progress?, title?)
rpcCall("extensionApi.SYS_LoadingAndProgressBar.destroyProgressBar")
rpcCall("extensionApi.SYS_LoadingAndProgressBar.showLoading")
rpcCall("extensionApi.SYS_LoadingAndProgressBar.destroyLoading")

// 右键菜单
rpcCall("extensionApi.SYS_RightClickMenu.changeMenu", menuId, menuItems)
```

---

## 系统 — 单位/WebSocket/客户端URL

```typescript
rpcCall("extensionApi.SYS_Unit.getFrontendDataUnit") => ESYS_Unit
// 静态方法 (不需要 RPC): milToMm, mmToMil, inchToMil, milToInch, inchToMm, mmToInch

// WebSocket (扩展可用)
rpcCall("extensionApi.SYS_WebSocket.register", id, url, onMessage, onConnect, protocols?)
rpcCall("extensionApi.SYS_WebSocket.send", id, data)
rpcCall("extensionApi.SYS_WebSocket.close", id, code?, reason?)

// HTTP 请求 (扩展可用 — 需 allowExternalInteractions 权限)
rpcCall("extensionApi.SYS_ClientUrl.request", url, method?, data?, options?, successCallback?)
```

---

## 原理图内部 RPC (sch-main.js — 仅供内部使用, 非扩展 API)

```
sch/selectWithShowCrossCursorByIds
sch/openProject, sch/closeProject, sch/closeDoc, sch/handleOpenDoc
sch/renderPropertyPanel, sch/renderSymbolPropertyPanel, sch/renderSchPropertyPanel
sch/setTabData, sch/setSymbolDetail, sch/setSubTabActive
sch/copy, sch/paste, sch/cut
sch/selectByIds, sch/unSelectAll
sch/highlightItems, sch/unhighlightItems
sch/highlightModelsByIds
sch/generateSymbol
sch/autoLayout, sch/autoWiring
sch/print, sch/exportFile, sch/getExportFileData
sch/getDeviceDetail
sch/getPartTreeData
sch/getModelSheetUuidSync, sch/getModelTabId, sch/getPropertyItemData
sch/validateIsConformBusName, sch/validateIsConformWireName
sch/cancelFanOutNetLabelNoConnect
sch/setProperty, sch/addProperty, sch/addSymbolDocProperty, sch/setSymbolDocProperty
sch/getNetGroupInfoByNetName
sch/tableCellSelectAreaChange
sch/adaptSelectionView
sch/hasOpened, sch/clearTarget
sch/setIsVersionReadOnly
sch/SubTabbarMapRemove
sch/showToSpecificPage, sch/showDlgFind, sch/togglePlaceToolBarShow
```

---

## PCB 内部 RPC (pcb-main.js — 仅供内部使用)

```
copyData, pasteData, clearImageCopy
pcb/getIframeUrl
getPcbAnnotationInfo
dataStr2RenderingdataStr
getLastModifyInPcb
handle3DModelFile, newOrReplace3DModel
components2newFormatByIndexDB
pcb/decodePcbWorldInWorker
```

---

## 关键枚举速查表

### EPCB_LayerId
```
TOP=1, BOTTOM=2, TOP_SILK_SCREEN=3, BOTTOM_SILK_SCREEN=4,
TOP_PASTE_MASK=5, BOTTOM_PASTE_MASK=6, TOP_SOLDER_MASK=7, BOTTOM_SOLDER_MASK=8,
RATLINES=9, BOARD_OUTLINE=10, MULTI=11, DOCUMENT=12,
TOP_ASSEMBLY=13, BOTTOM_ASSEMBLY=14, MECHANICAL=15,
TOP_COMPONENTS=16, BOTTOM_COMPONENTS=17, SUBSTRATE=18, 3D_MODEL=19,
INNER_1=21 ... INNER_32=52,
SUBSTRATE_1=53 ... SUBSTRATE_32=84,
TOP_STIFFENER=58, BOTTOM_STIFFENER=59,
COMPONENT_SHAPE=99, LEAD_SHAPE=100
```

### EPCB_PrimitiveType
```
ARC, COMPONENT, PAD, COMPONENT_PAD, POLYLINE, POUR, FILL,
REGION, LINE, VIA, DIMENSION, IMAGE, OBJECT, POURED, STRING, ATTRIBUTE
```

### EPCB_PrimitivePadShapeType
```
ELLIPSE, OVAL, RECT, POLYGON
```

### EPCB_PrimitivePadHoleType
```
ROUND, RECT, SLOT
```

### EPCB_PrimitivePadType
```
NORMAL=0, TEST=1, MARK_POINT=2
```

### EPCB_PrimitiveViaType
```
VIA=0, BLIND=1, SUTURE=2
```

### EPCB_PrimitiveFillMode
```
SOLID=0, MESH=1, INNER_ELECTRICAL_LAYER=2
```

### EPCB_PrimitiveRegionRuleType
```
NO_COMPONENTS=2, NO_VIAS=3, NO_WIRES=5, NO_FILLS=6, NO_POURS=7,
NO_INNER_ELECTRICAL_LAYERS=8, FOLLOW_REGION_RULE=9
```

### EPCB_PrimitivePourFillMethod
```
"solid", "45grid", "90grid"
```

### EPCB_PrimitiveStringAlignMode
```
LEFT_TOP=1, LEFT_MIDDLE=2, LEFT_BOTTOM=3,
CENTER_TOP=4, CENTER=5, CENTER_BOTTOM=6,
RIGHT_TOP=7, RIGHT_MIDDLE=8, RIGHT_BOTTOM=9
```

### EPCB_PrimitiveDimensionType
```
LENGTH, RADIUS, ANGLE
```

### EPCB_PrimitiveArcInteractiveMode
```
TWO_POINT_ARC=1, CENTER_ARC=2
```

### EPCB_PcbPlateType
```
NORMAL=1, FPC=2
```

### EPCB_InactiveLayerDisplayMode
```
NORMAL_BRIGHTNESS=0, TURN_GRAY=1, HIDE=2
```

### EPCB_LayerColorConfiguration
```
JLCEDA=1, EASYEDA=1, ALTIUM_DESIGNER=2, PADS=3, KICAD=4
```

### ESCH_PrimitiveType
```
ARC, BUS, CIRCLE, COMPONENT, COMPONENT_PIN, PIN, POLYGON, RECTANGLE, TEXT, WIRE, OBJECT,
BEZIER, ELLIPSE, BUSENTRY, MASK_REGION, PART, TABLE, ELE_PLACEHOLDER, NG_SETTING, ATTR
```

### ESCH_PrimitiveLineType
```
SOLID=0, DASHED=1, DOTTED=2, DOT_DASHED=3
```

### ESCH_PrimitiveFillStyle
```
NONE, SOLID, GRID, HORIZONTAL_LINE, VERTICAL_LINE,
RHOMBIC, LEFT_SLASH_LINE, RIGHT_SLASH_LINE
```

### ESCH_PrimitivePinShape
```
NONE, INVERTED, CLOCK, INVERTED_CLOCK
```

### ESCH_PrimitivePinType
```
IN, OUT, BI, PASSIVE, OPEN_COLLECTOR, OPEN_EMITTER,
POWER, GROUND, HIZ, TERMINATOR, UNDEFINED
```

### ESCH_PrimitiveTextAlignMode
```
LEFT_BOTTOM, CENTER_BOTTOM, RIGHT_BOTTOM,
LEFT_MIDDLE, CENTER_MIDDLE, RIGHT_MIDDLE,
LEFT_TOP, CENTER_TOP, RIGHT_TOP
```

### ESYS_Unit
```
MILLIMETER("mm"), CENTIMETER("cm"), METER("m"), INCH("inch"), MIL("mil")
```

### ESYS_NetlistType
```
JLCEDA, EasyEDA, ALLEGRO, PADS, PROTEL2, ALTIUM_DESIGNER, DISA
```

### ESYS_ToastMessageType
```
ERROR, WARNING, INFO, SUCCESS, ASK
```

### EDMT_EditorDocumentType
```
BLANK=0, SCHEMATIC_PAGE=1, PCB=3,
SYMBOL_COMPONENT=2, FOOTPRINT_EDITOR=4,
PCB_2D_PREVIEW=12, PCB_3D_PREVIEW=15,
PANEL=26, PANEL_3D_PREVIEW=27
```

### ELIB_LibraryType
```
CBB=1, SYMBOL=2, DEVICE=3, FOOTPRINT=4, MODEL=5, PANEL_LIBRARY=29
```

### ELIB_SymbolType
```
COMPONENT=2, NET_FLAG=18, NET_PORT=19, DRAWING=20,
NON_ELECTRICAL=21, SHORT_CIRCUIT_FLAG=22,
OFF_PAGE_CONNECTOR=25, DIFFERENTIAL_PAIRS_FLAG=31, CBB_SYMBOL=17
```

---

## 坐标系速查

| 域 | 单位 | 1mm = |
|----|------|-------|
| **PCB** | 1 mil | 39.37 |
| **原理图** | 0.01 inch (10 mil) | 3.937 |

---

## 修改图元的标准模式

```javascript
// 1. 获取
const prim = await eda.pcb_PrimitiveLine.get([id])
// 2. → Async
const a = prim.toAsync()
// 3. 修改属性
a.setState_StartX(100).setState_Net("VCC")
// 4. 提交
await a.done()
// 或撤销: await a.reset()
```
