# 嘉立创 EDA 内部逆向 — 给你的 bridge 的完整参考

> 来源: `E:\lceda-pro\` V3.2.69 全套逆向 | 2026-07-12

---

## 0. RPC 核心机制

你的 bridge 跑在 EDA 扩展沙箱里，所有 `eda.*` 调用最终走：

```javascript
_.extensionApiMessageBus2.rpcCall("extensionApi.XXX.YYY", arg1, arg2, arg3, ...)
```

参数是 **positional**（按位置传递），不是 `{key: value}` 对象。

Handler 在引擎文件（pcb.js/sch.js/pcb-main.js/sch-main.js）里通过：
```javascript
_.extensionApiMessageBus2.rpcService("extensionApi.XXX.YYY", async (arg1, arg2, arg3) => {...})
```
注册。

---

## 1. 完整 extensionApi RPC 清单 (从 api.js 源码提取)

### DMT_ 文档树 (工程/板/原理图/PCB 管理)

```
extensionApi.DMT_Project.openProject(projectUuid)
extensionApi.DMT_Project.createProject(friendlyName, projectName?, teamUuid?, folderUuid?, desc?, collabMode?)
extensionApi.DMT_Project.moveProjectToFolder(projectUuid, targetTeamUuid?, targetFolderUuid?)
extensionApi.DMT_Project.getAllProjectsUuid(teamUuid?, folderUuid?, workspaceUuid?)
extensionApi.DMT_Project.getProjectInfo(projectUuid)
extensionApi.DMT_Project.getCurrentProjectInfo()
extensionApi.DMT_Project.getCurrentProjectUuid()

extensionApi.DMT_Board.createBoard(schematicUuid?, pcbUuid?)
extensionApi.DMT_Board.modifyBoardName(originalName, newName)
extensionApi.DMT_Board.copyBoard(sourceName)
extensionApi.DMT_Board.getAllBoardsInfo()
extensionApi.DMT_Board.getCurrentBoardInfo()
extensionApi.DMT_Board.getBoardInfo(boardName)
extensionApi.DMT_Board.deleteBoard(boardName)

extensionApi.DMT_Pcb.createPcb(boardName?)
extensionApi.DMT_Pcb.modifyPcbName(uuid, name)
extensionApi.DMT_Pcb.copyPcb(uuid, boardName?)
extensionApi.DMT_Pcb.getAllPcbsInfo()
extensionApi.DMT_Pcb.getCurrentPcbInfo()
extensionApi.DMT_Pcb.getPcbInfo(uuid)
extensionApi.DMT_Pcb.deletePcb(uuid)

extensionApi.DMT_Schematic.createSchematic(boardName?)
extensionApi.DMT_Schematic.createSchematicPage(schematicUuid)
extensionApi.DMT_Schematic.modifySchematicName(uuid, name)
extensionApi.DMT_Schematic.modifySchematicPageName(pageUuid, name)
extensionApi.DMT_Schematic.modifySchematicPageTitleBlock(showTitle?, titleBlockData?)
extensionApi.DMT_Schematic.copySchematic(uuid, boardName?)
extensionApi.DMT_Schematic.copySchematicPage(pageUuid, schematicUuid?)
extensionApi.DMT_Schematic.getSchematicInfo(uuid)
extensionApi.DMT_Schematic.getSchematicPageInfo(pageUuid)
extensionApi.DMT_Schematic.getAllSchematicsInfo()
extensionApi.DMT_Schematic.getAllSchematicPagesInfo()
extensionApi.DMT_Schematic.getCurrentSchematicInfo()
extensionApi.DMT_Schematic.getCurrentSchematicPageInfo()
extensionApi.DMT_Schematic.getCurrentSchematicAllSchematicPagesInfo()
extensionApi.DMT_Schematic.reorderSchematicPages(uuid, pageItemsArray)
extensionApi.DMT_Schematic.deleteSchematic(uuid)
extensionApi.DMT_Schematic.deleteSchematicPage(pageUuid)

extensionApi.DMT_Panel.createPanel()
extensionApi.DMT_Panel.modifyPanelName(uuid, name)
extensionApi.DMT_Panel.copyPanel(uuid)
extensionApi.DMT_Panel.getPanelInfo(uuid)
extensionApi.DMT_Panel.getAllPanelsInfo()
extensionApi.DMT_Panel.getCurrentPanelInfo()
extensionApi.DMT_Panel.deletePanel(uuid)

extensionApi.DMT_EditorControl.openDocument(docUuid, splitScreenId?)
extensionApi.DMT_EditorControl.openLibraryDocument(libUuid, libType, uuid, splitScreenId?)
extensionApi.DMT_EditorControl.closeDocument(tabId)
extensionApi.DMT_EditorControl.activateDocument(tabId)
extensionApi.DMT_EditorControl.activateSplitScreen(splitScreenId)
extensionApi.DMT_EditorControl.getSplitScreenTree()
extensionApi.DMT_EditorControl.getSplitScreenIdByTabId(tabId)
extensionApi.DMT_EditorControl.getTabsBySplitScreenId(splitScreenId)
extensionApi.DMT_EditorControl.createSplitScreen(direction, tabId)
extensionApi.DMT_EditorControl.moveDocumentToSplitScreen(tabId, splitScreenId)
extensionApi.DMT_EditorControl.tileAllDocumentToSplitScreen()
extensionApi.DMT_EditorControl.mergeAllDocumentFromSplitScreen()
extensionApi.DMT_EditorControl.zoomTo(x?, y?, scaleRatio?, tabId?)
extensionApi.DMT_EditorControl.zoomToRegion(left, right, top, bottom, tabId?)
extensionApi.DMT_EditorControl.zoomToAllPrimitives(tabId?)
extensionApi.DMT_EditorControl.zoomToSelectedPrimitives(tabId?)
extensionApi.DMT_EditorControl.getCurrentRenderedAreaImage(tabId?)
extensionApi.DMT_EditorControl.generateIndicatorMarkers(markers, color?, lineWidth?, zoom?, tabId?)
extensionApi.DMT_EditorControl.removeIndicatorMarkers(tabId?)

extensionApi.DMT_Folder.createFolder(name, teamUuid, parentFolderUuid?, desc?)
extensionApi.DMT_Folder.modifyFolderName(teamUuid, folderUuid, name)
extensionApi.DMT_Folder.modifyFolderDescription(teamUuid, folderUuid, desc?)
extensionApi.DMT_Folder.moveFolderToFolder(teamUuid, folderUuid, parentFolderUuid?)
extensionApi.DMT_Folder.getAllFoldersUuid(teamUuid)
extensionApi.DMT_Folder.getFolderInfo(teamUuid, folderUuid)
extensionApi.DMT_Folder.deleteFolder(teamUuid, folderUuid)

extensionApi.DMT_Team.getAllTeamsInfo()
extensionApi.DMT_Team.getAllInvolvedTeamInfo()
extensionApi.DMT_Team.getCurrentTeamInfo()

extensionApi.DMT_Workspace.getAllWorkspacesInfo()
extensionApi.DMT_Workspace.toggleToWorkspace(workspaceUuid?)
extensionApi.DMT_Workspace.getCurrentWorkspaceInfo()

extensionApi.DMT_SelectControl.getCurrentDocumentInfo()
```

### PCB_Document

```
extensionApi.PCB_Document.save(uuid)
extensionApi.PCB_Document.importChanges(uuid?)
extensionApi.PCB_Document.importAutoRouteJsonFile(file: File)
extensionApi.PCB_Document.importAutoLayoutJsonFile(file: File)
extensionApi.PCB_Document.getRatlineStatus()
extensionApi.PCB_Document.setRatlineStatus(status)
extensionApi.PCB_Document.convertCanvasOriginToDataOrigin(x, y)
extensionApi.PCB_Document.convertDataOriginToCanvasOrigin(x, y)
extensionApi.PCB_Document.getCanvasOrigin()
extensionApi.PCB_Document.setCanvasOrigin(offsetX, offsetY)
extensionApi.PCB_Document.navigateToCoordinates(x, y)
extensionApi.PCB_Document.navigateToRegion(left, right, top, bottom)
extensionApi.PCB_Document.getPrimitiveAtPoint(x, y)
extensionApi.PCB_Document.getPrimitivesInRegion(left, right, top, bottom, leftToRight?)
extensionApi.PCB_Document.zoomToBoardOutline()
extensionApi.PCB_Document.getCurrentFilterConfiguration()
extensionApi.PCB_Document.getAllPCBTabData()
extensionApi.PCB_Document.clearRouting(type?)
```

### PCB 图元 (所有 create/delete/get/getAll/getAllPrimitiveId/modify)

每种图元有相同模式：

```
extensionApi.PCB_PrimitiveLine.create(net, layer, startX, startY, endX, endY, lineWidth?, primitiveLock?)
extensionApi.PCB_PrimitiveLine.delete(ids)
extensionApi.PCB_PrimitiveLine.getByIds(id)         // 单 id → 单对象
extensionApi.PCB_PrimitiveLine.getByIds([ids])       // 数组 → 对象数组
extensionApi.PCB_PrimitiveLine.getAll(net?, layer?, primitiveLock?)
extensionApi.PCB_PrimitiveLine.modify(id, {net?,layer?,startX?,startY?,endX?,endY?,lineWidth?,primitiveLock?})
extensionApi.PCB_PrimitiveLine.getAdjacentPrimitives(id)
extensionApi.PCB_PrimitiveLine.getEntireTrack(id, includeVias?)

extensionApi.PCB_PrimitiveArc.create(net, layer, startX, startY, endX, endY, arcAngle, lineWidth?, interactiveMode?, primitiveLock?)
extensionApi.PCB_PrimitiveArc.delete/getByIds/getAll/modify/getAdjacentPrimitives/getEntireTrack

extensionApi.PCB_PrimitiveVia.create(net, x, y, holeDiameter, diameter, viaType?, designRuleBlindViaName?, solderMaskExpansion?, primitiveLock?)
extensionApi.PCB_PrimitiveVia.delete/getByIds/getAll/modify/getAdjacentPrimitives

extensionApi.PCB_PrimitivePad.create(layer, x, y, padShape, padNumber, net?, padType?, hole?, holeOffsetX?, holeOffsetY?, holeRotation?, metallization?, specialPad?, solderMaskExpansion?, heatWelding?, rotation?, primitiveLock?)
extensionApi.PCB_PrimitivePad.delete/getByIds/getAll/modify/getConnectedPrimitives

extensionApi.PCB_PrimitiveComponent.create(component, layer, x, y, rotation?, primitiveLock?)
extensionApi.PCB_PrimitiveComponent.delete/getByIds/getAll/modify/placeComponentWithMouse/getAllPinsByPrimitiveId/setAttribute/getAllPropertyNames

extensionApi.PCB_PrimitivePour.create(net, layer, complexPolygon, pourFillMethod?, preserveSilos?, pourName?, pourPriority?, lineWidth?, primitiveLock?)
extensionApi.PCB_PrimitivePour.delete/getByIds/getAll/modify/getCopperRegion/rebuildCopperRegion/convertToType

extensionApi.PCB_PrimitivePoured.delete/getAll/convertToFill/addSolderMaskFill/deletePourFills

extensionApi.PCB_PrimitiveFill.create(layer, complexPolygon, net?, fillMode?, lineWidth?, primitiveLock?)
extensionApi.PCB_PrimitiveFill.delete/getByIds/getAll/modify/convertToType

extensionApi.PCB_PrimitivePolyline.create(net, layer, polygon, lineWidth?, primitiveLock?)
extensionApi.PCB_PrimitivePolyline.delete/getByIds/getAll/modify/convertToType

extensionApi.PCB_PrimitiveRegion.create(layer, complexPolygon, ruleType?, regionName?, lineWidth?, primitiveLock?)
extensionApi.PCB_PrimitiveRegion.delete/getByIds/getAll/modify/convertToType

extensionApi.PCB_PrimitiveString.create(layer, x, y, text, fontFamily?, fontSize?, lineWidth?, alignMode?, rotation?, reverse?, expansion?, mirror?, primitiveLock?)
extensionApi.PCB_PrimitiveString.delete/getByIds/getAll/modify

extensionApi.PCB_PrimitiveDimension.create(dimensionType, coordinateSet, layer?, unit?, lineWidth?, precision?, primitiveLock?)
extensionApi.PCB_PrimitiveDimension.delete/getByIds/getAll/modify

extensionApi.PCB_PrimitiveImage.create(x, y, complexPolygon, layer, width?, height?, rotation?, horizonMirror?, primitiveLock?)
extensionApi.PCB_PrimitiveImage.delete/getByIds/getAll/modify

extensionApi.PCB_PrimitiveObject.create(layer, topLeftX, topLeftY, binaryData, width, height, rotation?, mirror?, fileName?, primitiveLock?)
extensionApi.PCB_PrimitiveObject.delete/getByIds/getAll/modify

extensionApi.PCB_PrimitiveAttribute.delete/getByIds/getAll/modify

extensionApi.PCB_Primitive.getPrimitivesBBox(primitiveIds)
```

### PCB_SelectControl

```
extensionApi.PCB_SelectControl.getAllSelectedPrimitiveIds()
extensionApi.PCB_SelectControl.getAllSelectedPrimitives()
extensionApi.PCB_SelectControl.doSelectPrimitives(ids)
extensionApi.PCB_SelectControl.doCrossProbeSelect(components?, pins?, nets?, highlight?, select?)
extensionApi.PCB_SelectControl.doCrossProbeSelectByObject(object)
extensionApi.PCB_SelectControl.clearSelected()
extensionApi.PCB_SelectControl.getCurrentMousePosition()
```

### PCB_Layer

```
extensionApi.PCB_Layer.selectLayerById(layerId)
extensionApi.PCB_Layer.setLayerStatus(layerId, visible?, active?)
extensionApi.PCB_Layer.setTheNumberOfCopperLayers(n)
extensionApi.PCB_Layer.getTheNumberOfCopperLayers()
extensionApi.PCB_Layer.setLayerColorConfiguration(config)
extensionApi.PCB_Layer.setInactiveLayerTransparency(transparency)
extensionApi.PCB_Layer.setInactiveLayerDisplayMode(mode)
extensionApi.PCB_Layer.setPcbType(type)
extensionApi.PCB_Layer.addCustomLayer()
extensionApi.PCB_Layer.removeLayer(layer)
extensionApi.PCB_Layer.modifyLayer(layer, {name?, type?, color?, transparency?})
extensionApi.PCB_Layer.getAllLayers()
```

### PCB_Net

```
extensionApi.PCB_Net.getAllNets()
extensionApi.PCB_Net.getNet(netName)
extensionApi.PCB_Net.getAllNetName()
extensionApi.PCB_Net.getAllNetsName()
extensionApi.PCB_Net.getNetLength(netName)
extensionApi.PCB_Net.getNetColor(netName)
extensionApi.PCB_Net.setNetColor(netName, color)
extensionApi.PCB_Net.getAllPrimitivesByNet(netName, primitiveTypes?)
extensionApi.PCB_Net.selectNet(netName)
extensionApi.PCB_Net.unselectNet(netName)
extensionApi.PCB_Net.unselectAllNets()
extensionApi.PCB_Net.highlightNet(netName)
extensionApi.PCB_Net.unhighlightNet(netName)
extensionApi.PCB_Net.unhighlightAllNets()
extensionApi.PCB_Net.getNetlist(type?)
extensionApi.PCB_Net.setNetlist(type, netlist)
```

### PCB_Drc

```
extensionApi.PCB_Drc.check(strict, userInterface, includeVerboseError)
extensionApi.PCB_Drc.getCurrentRuleConfigurationName()
extensionApi.PCB_Drc.getCurrentRuleConfiguration()
extensionApi.PCB_Drc.getRuleConfiguration(name)
extensionApi.PCB_Drc.getAllRuleConfigurations(includeSystem?)
extensionApi.PCB_Drc.saveRuleConfiguration(config, name, allowOverwrite?)
extensionApi.PCB_Drc.renameRuleConfiguration(old, new)
extensionApi.PCB_Drc.deleteRuleConfiguration(name)
extensionApi.PCB_Drc.getDefaultRuleConfigurationName()
extensionApi.PCB_Drc.setAsDefaultRuleConfiguration(name)
extensionApi.PCB_Drc.getNetRules()
extensionApi.PCB_Drc.overwriteNetRules(rules)
extensionApi.PCB_Drc.getNetByNetRules()
extensionApi.PCB_Drc.overwriteNetByNetRules(rules)
extensionApi.PCB_Drc.getRegionRules()
extensionApi.PCB_Drc.overwriteRegionRules(rules)
extensionApi.PCB_Drc.overwriteCurrentRuleConfiguration(config)
extensionApi.PCB_Drc.createNetClass(name, nets, color)
extensionApi.PCB_Drc.deleteNetClass(name)
extensionApi.PCB_Drc.modifyNetClassName(old, new)
extensionApi.PCB_Drc.addNetToNetClass(className, net)
extensionApi.PCB_Drc.removeNetFromNetClass(className, net)
extensionApi.PCB_Drc.getAllNetClasses()
extensionApi.PCB_Drc.createDifferentialPair(name, posNet, negNet)
extensionApi.PCB_Drc.deleteDifferentialPair(name)
extensionApi.PCB_Drc.modifyDifferentialPairName(old, new)
extensionApi.PCB_Drc.modifyDifferentialPairPositiveNet(name, net)
extensionApi.PCB_Drc.modifyDifferentialPairNegativeNet(name, net)
extensionApi.PCB_Drc.getAllDifferentialPairs()
extensionApi.PCB_Drc.createEqualLengthNetGroup(name, nets, color)
extensionApi.PCB_Drc.deleteEqualLengthNetGroup(name)
extensionApi.PCB_Drc.modifyEqualLengthNetGroupName(old, new)
extensionApi.PCB_Drc.addNetToEqualLengthNetGroup(groupName, net)
extensionApi.PCB_Drc.removeNetFromEqualLengthNetGroup(groupName, net)
extensionApi.PCB_Drc.getAllEqualLengthNetGroups()
extensionApi.PCB_Drc.createPadPairGroup(name, padPairs)
extensionApi.PCB_Drc.deletePadPairGroup(name)
extensionApi.PCB_Drc.modifyPadPairGroupName(old, new)
extensionApi.PCB_Drc.addPadPairToPadPairGroup(groupName, padPair)
extensionApi.PCB_Drc.removePadPairFromPadPairGroup(groupName, padPair)
extensionApi.PCB_Drc.getAllPadPairGroups()
extensionApi.PCB_Drc.getPadPairGroupMinWireLength(groupName)
```

### PCB_ManufactureData

```
extensionApi.PCB_ManufactureData.getGerberFile(fileName?, silk?, unit?, format?, drilledHole?, layerConfigs?, objectConfigs?)
extensionApi.PCB_ManufactureData.get3DFile(fileName?, fileType?, elements?, modelMode?, autoGen?)
extensionApi.PCB_ManufactureData.get3DShellFile(fileName?, fileType?)
extensionApi.PCB_ManufactureData.getPickAndPlaceFile(fileName?, fileType?, unit?)
extensionApi.PCB_ManufactureData.getFlyingProbeTestFile(fileName?)
extensionApi.PCB_ManufactureData.getBomTemplates()
extensionApi.PCB_ManufactureData.uploadBomTemplateFile(file, template?)
extensionApi.PCB_ManufactureData.getBomTemplateFile(template)
extensionApi.PCB_ManufactureData.deleteBomTemplate(template)
extensionApi.PCB_ManufactureData.getBomDefaultParams()
extensionApi.PCB_ManufactureData.getBomFile(fileName?, fileType?, template?, filterOptions?, statistics?, property?, columns?)
extensionApi.PCB_ManufactureData.getTestPointFile(fileName?, fileType?)
extensionApi.PCB_ManufactureData.getNetlistFile(fileName?, netlistType?)
extensionApi.PCB_ManufactureData.getDxfFile(fileName?, layers?, objects?)
extensionApi.PCB_ManufactureData.getPDFFile(fileName?, outputMethod?, ...)
extensionApi.PCB_ManufactureData.getIPCFile(fileName?)
extensionApi.PCB_ManufactureData.getOpenDatabaseDoublePlusFile(...)
extensionApi.PCB_ManufactureData.getInteractiveBomFile()
extensionApi.PCB_ManufactureData.getDsnFile(fileName?)
extensionApi.PCB_ManufactureData.getAutoRouteJson(fileName?)
extensionApi.PCB_ManufactureData.getAutoLayoutJson(fileName?)
extensionApi.PCB_ManufactureData.getADFile(fileName?)
extensionApi.PCB_ManufactureData.getPADSFile(fileName?)
extensionApi.PCB_ManufactureData.getPcbInfoFile(fileName?)
extensionApi.PCB_ManufactureData.getOrderComponents()
extensionApi.PCB_ManufactureData.getOrderSMT()
extensionApi.PCB_ManufactureData.getOrderPCB()
extensionApi.PCB_ManufactureData.getOrder3DShellFile()
extensionApi.PCB_ManufactureData.getManufactureData()
extensionApi.PCB_ManufactureData.updateAutoRouteRule(rule)
extensionApi.PCB_ManufactureData.placeComponentsOrder(interactive?, ignoreWarning?)
extensionApi.PCB_ManufactureData.placeSmtComponentsOrder(interactive?, ignoreWarning?)
extensionApi.PCB_ManufactureData.placePcbOrder(interactive?, ignoreWarning?)
extensionApi.PCB_ManufactureData.place3DShellOrder(interactive?, ignoreWarning?)
```

### PCB_Event, PCB_MathPolygon

```
extensionApi.PCB_Event.mouseEvent(id, eventType, callback, onlyOnce?)
extensionApi.PCB_Event.primitiveEvent(id, eventType, callback, onlyOnce?)
extensionApi.PCB_Event.netEvent(id, eventType, callback, onlyOnce?)
extensionApi.PCB_Event.crossProbeSelectEvent(id, callback)

extensionApi.PCB_MathPolygon.convertImageToComplexPolygon(blob, width, height, tolerance?, simplification?, smoothing?, despeckling?, whiteAsBg?, inversion?)
extensionApi.PCB_MathPolygon.getCenter(polygon)
```

### SCH 原理图

```
extensionApi.SCH_Document.save()
extensionApi.SCH_Document.importChanges()
extensionApi.SCH_Document.autoRouting(data)
extensionApi.SCH_Document.autoLayout(data)

extensionApi.SCH_Drc.check(strict, userInterface, includeVerboseError)

extensionApi.SCH_Primitive.getPrimitiveTypeByPrimitiveId(id)
extensionApi.SCH_Primitive.getPrimitivesBBox(ids)

extensionApi.SCH_PrimitiveWire.create(line, net?, color?, lineWidth?, lineType?)
extensionApi.SCH_PrimitiveWire.delete/get/getAll/getAllPrimitiveId/modify

extensionApi.SCH_PrimitiveComponent.create(component, x, y, subPartName?, rotation?, mirror?, addIntoBom?, addIntoPcb?)
extensionApi.SCH_PrimitiveComponent.createNetFlag(identification, net, x, y, rotation?, mirror?)
extensionApi.SCH_PrimitiveComponent.createNetPort(direction, net, x, y, rotation?, mirror?)
extensionApi.SCH_PrimitiveComponent.createShortCircuitFlag(x, y, rotation?, mirror?)
extensionApi.SCH_PrimitiveComponent.delete/modify/get/getAll/getAllPrimitiveId/placeComponentWithMouse/getDeviceData/getAllPropertyNames

extensionApi.SCH_PrimitiveComponent.setNetFlagComponentUuid_Power/Ground/AnalogGround/ProtectGround(component)
extensionApi.SCH_PrimitiveComponent.setNetPortComponentUuid_IN/OUT/BI(component)

extensionApi.SCH_PrimitiveComponent3.create/delete/modify/get/getAll/getAllPrimitiveId/placeComponentWithMouse/getDeviceData/createCbbSymbol/placeCbbSchematicPage

extensionApi.SCH_PrimitiveBus.create/create/delete/modify/get/getAll/getAllPrimitiveId
extensionApi.SCH_PrimitiveCircle.create/delete/modify/get/getAll
extensionApi.SCH_PrimitiveArc.create/delete/modify/get/getAll
extensionApi.SCH_PrimitivePin.create/delete/modify/getPinJson/getAllPrimitiveId
extensionApi.SCH_PrimitivePolygon.create/delete/modify/getAllPrimitiveId
extensionApi.SCH_PrimitiveRectangle.create/delete/modify/getAllPrimitivesJsonCache
extensionApi.SCH_PrimitiveText.create/delete/modify/get

extensionApi.SCH_PrimitiveWireBusUtil3.create/modify/get

extensionApi.SCH_SelectControl.getAllSelectedPrimitives_PrimitiveId()
extensionApi.SCH_SelectControl.getAllSelectedPrimitives()
extensionApi.SCH_SelectControl.getSelectedPrimitives_PrimitiveId()
extensionApi.SCH_SelectControl.getSpecifiedPrimitive(...)
extensionApi.SCH_SelectControl.getObjParams()
extensionApi.SCH_SelectControl.doSelectPrimitives(ids)
extensionApi.SCH_SelectControl.getCurrentMousePosition()

extensionApi.SCH_Netlist.getNetlist(type?)
extensionApi.SCH_Netlist.setNetlist(type, netlist)

extensionApi.SCH_Event.mouseEvent(id, eventType, callback, onlyOnce?)
extensionApi.SCH_Event.primitiveEvent(id, eventType, callback, onlyOnce?)

extensionApi.SCH_ManufactureData.getAssemblyVariantsConfigs()
extensionApi.SCH_ManufactureData.getBomFile(...)
extensionApi.SCH_ManufactureData.getNetlistFile(...)
extensionApi.SCH_ManufactureData.getExportDocumentFile(...)
extensionApi.SCH_ManufactureData.getSimulationNetlistFile(...)
extensionApi.SCH_ManufactureData.placeComponentsOrder(...)
extensionApi.SCH_ManufactureData.placeSmtComponentsOrder(...)

extensionApi.PNL_Document.save()
```

### LIB 库

```
extensionApi.LIB_LibrariesList.getSystemLibraryUuid/PersonalLibraryUuid/ProjectLibraryUuid/FavoriteLibraryUuid
extensionApi.LIB_LibrariesList.getAllLibrariesList()
extensionApi.LIB_LibrariesList.registerExtendLibrary(info)

extensionApi.LIB_Device.create/delete/modify/get/copy/search
extensionApi.LIB_Device.getByLcscIds(lcscIds, libUuid?, allowMultiMatch?)

extensionApi.LIB_Footprint.create/delete/modify/get/copy/search/openInEditor/updateDocumentSource/getRenderImage
extensionApi.LIB_Symbol.create/delete/modify/get/copy/search/openInEditor/updateDocumentSource/getRenderImage
extensionApi.LIB_3DModel.create/delete/modify/get/copy/search
extensionApi.LIB_Cbb.create/delete/modify/get/copy/search/openProjectInEditor/openSymbolInEditor
extensionApi.LIB_PanelLibrary.create/delete/modify/get/copy/search/openInEditor

extensionApi.LIB_Classification.createPrimary/createSecondary/getIndexByName/getNameByUuid
extensionApi.LIB_Classification.getAllClassificationTree/deleteByUuid/deleteByIndex

extensionApi.LIB_SelectControl.getBottomComponentLibraryCurrentRowInfo()
```

### SYS 系统

```
extensionApi.SYS_MessageBox.showInformationMessage(content, title?, buttonTitle?)
extensionApi.SYS_MessageBox.showConfirmationMessage(content, title?, mainButtonTitle?, buttonTitle?, callback?)
extensionApi.SYS_MessageBox.showInputDialog(...)
extensionApi.SYS_MessageBox.showSelectDialog(options, beforeContent?, afterContent?, title?, defaultOption?, multiple?, callback?)
extensionApi.SYS_MessageBox.insertScriptToDialog(...)

extensionApi.SYS_Message.showFollowMouseTip(tip, timeout?)
extensionApi.SYS_Message.removeFollowMouseTip(tip?)

extensionApi.SYS_FileSystem.openReadFileDialog(extensions?, multiFiles?)
extensionApi.SYS_FileSystem.saveFile(data, fileName?)
extensionApi.SYS_FileSystem.readFileFromFileSystem(uri)
extensionApi.SYS_FileSystem.saveFileToFileSystem(uri, data, fileName?, force?)
extensionApi.SYS_FileSystem.listFilesOfFileSystem(path, recursive?)
extensionApi.SYS_FileSystem.deleteFileInFileSystem(uri, force?)
extensionApi.SYS_FileSystem.getEdaPath()
extensionApi.SYS_FileSystem.getDocumentsPath()
extensionApi.SYS_FileSystem.getLibrariesPaths()
extensionApi.SYS_FileSystem.getProjectsPaths()

extensionApi.SYS_FileManager.getProjectFile/getDocumentFile/getDocumentSource/setDocumentSource
extensionApi.SYS_FileManager.getDeviceFileByDeviceUuid/getSymbolFileBySymbolUuid/getFootprintFileByFootprintUuid
extensionApi.SYS_FileManager.getCbbFileByCbbUuid/getPanelLibraryFileByPanelLibraryUuid
extensionApi.SYS_FileManager.importProjectByProjectFile/extractProjectInfo/extractLibInfo

extensionApi.SYS_FontManager.getFontsList/addFont/deleteFont

extensionApi.SYS_FormatConversion.convertAltiumDesignerLibrariesToEasyEDASingleFile/MultiFiles
extensionApi.SYS_FormatConversion.convertDisaLibrariesToEasyEDASingleFile/MultiFiles

extensionApi.SYS_Environment.setKeepProjectHasOnlyOneBoard/getEditorCurrentVersion/getEditorCompliedDate/getUserInfo
extensionApi.SYS_Environment.isClient/isOnlineMode/isOfflineMode/isHalfOfflineMode/isWeb
extensionApi.SYS_Environment.isEasyEDAProEdition/isJLCEDAProEdition/isProPrivateEdition

extensionApi.SYS_ShortcutKey.registerShortcutKey/unregisterShortcutKey/getShortcutKeys

extensionApi.SYS_PanelControl.openLeftPanel/closeLeftPanel/toggleLeftPanelLockState/isLeftPanelLocked
extensionApi.SYS_PanelControl.openRightPanel/closeRightPanel/toggleRightPanelLockState/isRightPanelLocked
extensionApi.SYS_PanelControl.openBottomPanel/closeBottomPanel/toggleBottomPanelLockState/isBottomPanelLocked

extensionApi.SYS_HeaderMenu.insertSystemHeaderMenuItem/removeSystemHeaderMenuItem/registerHeaderMenus

extensionApi.SYS_IFrame.openIFrame/closeIFrame/hideIFrame/showIFrame

extensionApi.SYS_Setting.restoreDefault

extensionApi.SYS_Log.add/clear/export/sort/find

extensionApi.SYS_Tool.netlistComparison(netlist1, netlist2)

extensionApi.SYS_Window.openUI/getCurrentTheme

extensionApi.SYS_LoadingAndProgressBar.showProgressBar/destroyProgressBar/showLoading/destroyLoading

extensionApi.SYS_RightClickMenu.changeMenu

extensionApi.SYS_Unit.getFrontendDataUnit()
// SYS_Unit 静态方法(本地调用，不走RPC): milToMm, mmToMil, inchToMil, milToInch, inchToMm, mmToInch

extensionApi.SYS_WebSocket.register/send/close

extensionApi.SYS_ClientUrl.request(url, method?, data?, options?, callback?)  // 需 allowExternalInteractions 权限

extensionApi.SYS_MessageBus.createPrivateMessageBus/removePrivateMessageBus
extensionApi.SYS_MessageBus.publish/subscribe/push/pull/rpcCall  (及 public 变体)
extensionApi.SYS_MessageBus.publishPublic/subscribePublic/pushPublic/pullPublic/rpcCallPublic
```

---

## 2. 关键枚举值 (源码直出)

```
EPCB_LayerId:
  TOP=1, BOTTOM=2
  TOP_SILK_SCREEN=3, BOTTOM_SILK_SCREEN=4
  TOP_PASTE_MASK=5, BOTTOM_PASTE_MASK=6
  TOP_SOLDER_MASK=7, BOTTOM_SOLDER_MASK=8
  RATLINES=9, BOARD_OUTLINE=10
  MULTI=11, DOCUMENT=12
  TOP_ASSEMBLY=13, BOTTOM_ASSEMBLY=14
  MECHANICAL=15
  TOP_COMPONENTS=16, BOTTOM_COMPONENTS=17
  SUBSTRATE=18, 3D_MODEL=19
  INNER_1=21 ... INNER_32=52
  TOP_STIFFENER=58, BOTTOM_STIFFENER=59
  COMPONENT_SHAPE=99, LEAD_SHAPE=100

EPCB_PrimitiveType:
  ARC, COMPONENT, PAD, COMPONENT_PAD, POLYLINE, POUR, FILL,
  REGION, LINE, VIA, DIMENSION, IMAGE, OBJECT, POURED, STRING, ATTRIBUTE

EPCB_PrimitivePadShapeType: ELLIPSE, OVAL, RECT, POLYGON
EPCB_PrimitivePadHoleType: ROUND, RECT, SLOT
EPCB_PrimitivePadType: NORMAL=0, TEST=1, MARK_POINT=2
EPCB_PrimitiveViaType: VIA=0, BLIND=1, SUTURE=2
EPCB_PrimitiveFillMode: SOLID=0, MESH=1, INNER_ELECTRICAL_LAYER=2
EPCB_PrimitivePourFillMethod: "solid", "45grid", "90grid"
EPCB_PrimitiveRegionRuleType:
  NO_COMPONENTS=2, NO_VIAS=3, NO_WIRES=5, NO_FILLS=6,
  NO_POURS=7, NO_INNER_ELECTRICAL_LAYERS=8, FOLLOW_REGION_RULE=9
EPCB_PrimitiveStringAlignMode:
  LEFT_TOP=1..LEFT_BOTTOM=3, CENTER_TOP=4..CENTER_BOTTOM=6, RIGHT_TOP=7..RIGHT_BOTTOM=9
  CENTER=5
EPCB_PrimitiveDimensionType: LENGTH, RADIUS, ANGLE
EPCB_PrimitiveArcInteractiveMode: TWO_POINT_ARC=1, CENTER_ARC=2
EPCB_PcbPlateType: NORMAL=1, FPC=2
EPCB_InactiveLayerDisplayMode: NORMAL_BRIGHTNESS=0, TURN_GRAY=1, HIDE=2
EPCB_LayerColorConfiguration: JLCEDA=1, ALTIUM_DESIGNER=2, PADS=3, KICAD=4
EPCB_LayerType: SIGNAL, INTERNAL_ELECTRICAL, SILKSCREEN, SOLDER_MASK, PASTE_MASK, ASSEMBLY, OTHER, CUSTOM

ESCH_PrimitiveType: ARC, BUS, CIRCLE, COMPONENT, COMPONENT_PIN, PIN, POLYGON, RECTANGLE, TEXT, WIRE, OBJECT
ESCH_PrimitiveLineType: SOLID=0, DASHED=1, DOTTED=2, DOT_DASHED=3
ESCH_PrimitiveFillStyle: NONE, SOLID, GRID, HORIZONTAL_LINE, VERTICAL_LINE
ESCH_PrimitivePinShape: NONE, INVERTED, CLOCK, INVERTED_CLOCK
ESCH_PrimitivePinType: IN, OUT, BI, PASSIVE, OPEN_COLLECTOR, OPEN_EMITTER, POWER, GROUND, HIZ, TERMINATOR, UNDEFINED

ESYS_Unit: MILLIMETER("mm"), CENTIMETER("cm"), METER("m"), INCH("inch"), MIL("mil")
ESYS_NetlistType: JLCEDA, EasyEDA, ALLEGRO, PADS, PROTEL2, ALTIUM_DESIGNER, DISA
ESYS_ToastMessageType: ERROR, WARNING, INFO, SUCCESS, ASK
EDMT_EditorDocumentType: BLANK=0, SCHEMATIC_PAGE=1, PCB=3, SYMBOL_COMPONENT=2, FOOTPRINT=4
ELIB_LibraryType: CBB=1, SYMBOL=2, DEVICE=3, FOOTPRINT=4, MODEL=5, PANEL_LIBRARY=29
```

---

## 3. 坐标系

| | PCB | SCH |
|---|---|---|
| 单位 | 1 mil | 0.01 inch (10 mil) |
| 1mm = | 39.37 | 3.937 |
| Y轴 | ↓ 向下 | ↓ 向下 |

---

## 4. 图元修改模式 (async)

```javascript
const p = await eda.pcb_PrimitiveLine.get([id])
const a = p.toAsync()
a.setState_X(100).setState_Net("VCC")
await a.done()   // 或 await a.reset() 撤销
```

---

## 5. 关键：importChanges 跨 SCH/PCB 关联

两边的 component 通过 `uniqueId` 匹配。`PCB_Document.importChanges(schematicUuid)`：
1. 读取 SCH 所有 component (按 uniqueId)
2. 匹配 PCB 已有 component：更新 footprint/designator/net/pads
3. 不匹配的：新建 PCB component 放默认位置
4. SCH 已删除的：从 PCB 移除

你的 bridge 可以暴露 `pcb.importChanges` 和 `schematic.importChanges`。

---

## 6. 你的 bridge 建议的 op 清单

```
# 工程
project.create, project.open, project.current, project.list

# 文档
board.create, board.list
schematic.createPage, schematic.list
pcb.create, pcb.list

# SCH 操作
schematic.placeComponents, schematic.createWires
schematic.exportNetlist, schematic.importChanges
schematic.drc

# PCB 操作
pcb.placeComponents, pcb.createTracks, pcb.createVias
pcb.createCopperPour, pcb.rebuildCopperPour, pcb.inspectCopperPour
pcb.createCopperFill, pcb.deleteCopperFill
pcb.createBoardOutline, pcb.configureLayers
pcb.importChanges, pcb.assignNets, pcb.applyNetlist
pcb.exportAutoLayoutJson, pcb.exportManufacturingFiles

# 规则
rules.detectDiffPairs, rules.applyDiffPairs, rules.auditHighSpeed
pcb.applyConstraintGroups

# DRC/DFM
drc.run, dfm.runJlc

# 布线
route.preflight, route.exportDsn, route.runExternalRouter, route.importSes, route.verify

# 库
library.search

# 网表
netlist.compare, netlist.compareLinkedDocuments
```
