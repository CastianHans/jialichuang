# 嘉立创 EDA 内部实现深度逆向

> 来源：DFM checker / Diff-pair assistant / pro-api-types 源码 + E:\lceda-pro\ 逆向
> 给 bridge 开发用的实现细节

---

## 1. DRC/DFM 结果结构 (完整接口)

```typescript
interface PcbDfmResult {
  timestamp: number;
  results: CheckResult[];
  passed: boolean;
  errorCount: number;
  warningCount: number;
}

interface SmtDfmResult extends PcbDfmResult {
  standard: 'economy' | 'standard';
}

interface CheckResult {
  number: number;
  item: string;           // 检查项中文名
  actualValue: string;    // 实测值
  standardValue: string;  // 标准值
  result: 'success' | 'warning' | 'error';
  violations?: ViolationCoord[];
}

interface ViolationCoord {
  x: number; y: number;   // 坐标 (mil)
  id: string;             // primitiveId → 可点击定位
  reason: string;
  type: string;
  locateType?: string;    // "rect" 等
}
```

## 2. 图元属性访问模式 (生产级用法)

```typescript
// 遍历所有走线
const allLines = await eda.pcb_PrimitiveLine.getAll();
const allLineIds = await eda.pcb_PrimitiveLine.getAllPrimitiveId();
// getAll() 和 getAllPrimitiveId() 结果按 INDEX 对齐

for (let i = 0; i < allLines.length; i++) {
  const line = allLines[i];
  const id = allLineIds[i];
  const layer = line.getState_Layer();        // EPCB_LayerId 值
  const net = line.getState_Net();            // 网络名
  const width = line.getState_LineWidth();    // mil
  const sx = line.getState_StartX();
  const sy = line.getState_StartY();
  const ex = line.getState_EndX();
  const ey = line.getState_EndY();
}

// 遍历所有过孔
const allVias = await eda.pcb_PrimitiveVia.getAll();
for (const via of allVias) {
  const diam = via.getState_Diameter();         // 外径 mil
  const hole = via.getState_HoleDiameter();     // 内径 mil
  const x = via.getState_X();
  const y = via.getState_Y();
  const net = via.getState_Net();
  const type = via.getState_PrimitiveType();    // EPCB_PrimitiveType.VIA
  const viaType = via.getState_ViaType();       // VIA=0, BLIND=1, SUTURE=2
  const blindName = via.getState_DesignRuleBlindViaName(); // 盲埋孔规则名
}

// 遍历所有焊盘
const allPads = await eda.pcb_PrimitivePad.getAll();
for (const pad of allPads) {
  const padShape = pad.getState_Pad();  // ["ELLIPSE", width, height] 或 ["RECT", w, h] 或 ["OVAL", w, h] 或 ["NGON", ...]
  const hole = pad.getState_Hole();     // null | ["ROUND", diameter] | ["SLOT", width, length]
  const metallization = pad.getState_Metallization();  // true=金属化孔
  const layer = pad.getState_Layer();   // 1=TOP, 2=BOT, 12=MULTI
  const net = pad.getState_Net();
  const rotation = pad.getState_Rotation();  // 弧度!
  const x = pad.getState_X();
  const y = pad.getState_Y();
}

// 遍历所有器件
const allComps = await eda.pcb_PrimitiveComponent.getAll();
for (const comp of allComps) {
  const designator = comp.getState_Designator();  // "R1"
  const name = comp.getState_Name();
  const uniqueId = comp.getState_UniqueId();      // 跨SCH/PCB关联KEY
  const footprint = comp.getState_Footprint();    // {libraryUuid, uuid, name?}
  const component = comp.getState_Component();    // {libraryUuid, uuid, name?}
  const pads = comp.getState_Pads();              // [{primitiveId, net, padNumber}]
  const layer = comp.getState_Layer();            // 1=TOP, 2=BOT
  const x = comp.getState_X();
  const y = comp.getState_Y();
  const rotation = comp.getState_Rotation();
  const locked = comp.getState_PrimitiveLock();
  const addIntoBom = comp.getState_AddIntoBom();
  const mfr = comp.getState_Manufacturer();
  const mfrId = comp.getState_ManufacturerId();
  const supplier = comp.getState_Supplier();
  const supplierId = comp.getState_SupplierId();
  const model3D = comp.getState_Model3D();        // {libraryUuid, uuid, name?}
  const other = comp.getState_OtherProperty();    // {[key:string]: string|number|boolean}
}

// 遍历覆铜
const allPours = await eda.pcb_PrimitivePour.getAll();
for (const pour of allPours) {
  const net = pour.getState_Net();
  const layer = pour.getState_Layer();
  const poly = pour.getState_ComplexPolygon();  // IPCB_Polygon
  const method = pour.getState_PourFillMethod(); // "solid"|"45grid"|"90grid"
  const preserveSilos = pour.getState_PreserveSilos();
  const name = pour.getState_PourName();
  const priority = pour.getState_PourPriority();
}

// 遍历填充
const allFills = await eda.pcb_PrimitiveFill.getAll();
for (const fill of allFills) {
  const layer = fill.getState_Layer();
  const net = fill.getState_Net();
  const poly = fill.getState_ComplexPolygon();   // IPCB_Polygon
  const mode = fill.getState_FillMode();         // SOLID=0, MESH=1, INNER_ELECTRICAL_LAYER=2
  // 转其他类型: await fill.convertToPolyline() / convertToPour() / convertToRegion()
}

// 遍历文本
const allStrings = await eda.pcb_PrimitiveString.getAll();
for (const s of allStrings) {
  const text = s.getState_Text();
  const font = s.getState_FontFamily();
  const fontSize = s.getState_FontSize();
  const layer = s.getState_Layer();
  const x = s.getState_X();
  const y = s.getState_Y();
  const align = s.getState_AlignMode();
  const rotation = s.getState_Rotation();
  const mirror = s.getState_Mirror();
  const reverse = s.getState_Reverse();
}

// 图元 BBox
const bbox = await eda.pcb_Primitive.getPrimitivesBBox(allLineIds);
// → { minX, minY, maxX, maxY } | undefined
```

## 3. 网络推断 (生产级)

```typescript
// 权威方法: 遍历所有网络
const allNetNames = await eda.pcb_Net.getAllNetsName();
for (const netName of allNetNames) {
  const prims = await eda.pcb_Net.getAllPrimitivesByNet(netName);
  // prims 包含该网络的所有图元
}

// 补充: 从焊盘推断 (通过连接性)
const connectedPrims = await pad.getConnectedPrimitives(false);
// onlyCentreConnection=false → 返回 Line|Arc|Via|Polyline|Fill

// 覆铜网络: 点在多边形内判断
// 用 getState_ComplexPolygon() 获取多边形, 然后判断焊盘坐标是否在内部
```

## 4. 叠层/物理层解析 (生产级)

```typescript
// 从文档源解析 LAYER_PHYS
const docSource = await eda.sys_FileManager.getDocumentSource();
// docSource 是换行分隔的紧凑JSON, 每行格式: recordJSON||payload
// LAYER_PHYS 记录: ["LAYER_PHYS", {id, thickness, material?, ...}]
// id: 1=TOP, 2=BOTTOM, 15-44=INNER_1..INNER_30
// thickness: mil

// 外层铜厚: parse id=1 和 id=2 的 LAYER_PHYS
// 内层铜厚: parse id=15..44 的 LAYER_PHYS
// 铜厚 mil → oz: thickness / 1.378, 就近取 0.5 oz 整数倍

// 设计层数
const designLayers = await eda.pcb_Layer.getTheNumberOfCopperLayers();

// 实际有铜层数: 从 getAllLines/getAllPolylines/getAllPours/getAllFills 统计
// 统计哪些 INNER 层上实际有图元
```

## 5. 盲埋孔规则解析 (生产级)

```typescript
// 从文档源解析 BLIND 规则
const docSource = await eda.sys_FileManager.getDocumentSource();
// BLIND 记录包含 "ruleContext.blinds.content"
// 每个盲埋孔规则: { staLayerId, endLayerId, name, ... }
// 如果 staLayerId==1 或 endLayerId==2 → blind (盲孔, 一端在表面)
// 否则 → buried (埋孔, 两端都在内层)
// 通孔: designRuleBlindViaName === null
```

## 6. 制造输出调用模式

```typescript
// Gerber
const gerberFile = await eda.pcb_ManufactureData.getGerberFile(
  "board-gerber",  // fileName
  true,            // silk (丝印)
  "mm",            // unit
  "4:4",           // format
  true,            // drilledHole
  [...],           // layerConfigs
  [...]            // objectConfigs
);
// → File 对象, 可用 SYS_FileSystem.saveFile() 保存

// BOM
const bomFile = await eda.pcb_ManufactureData.getBomFile(
  "bom.xlsx",      // fileName
  "xlsx",          // fileType
  undefined,       // template
  [],              // filterOptions
  [],              // statistics
  [],              // property
  undefined        // columns
);
// → File

// 坐标文件 (Pick and Place)
const pnpFile = await eda.pcb_ManufactureData.getPickAndPlaceFile(
  "cpl.xlsx",      // fileName
  "xlsx",          // fileType
  "mm"             // unit: ESYS_Unit.MILLIMETER | ESYS_Unit.MIL
);
// → File

// 3D 文件
const stepFile = await eda.pcb_ManufactureData.get3DFile(
  "board.step",    // fileName
  "step",          // fileType: 'step'|'obj'
  ['Component Model', 'Via', 'Silkscreen', 'Wire In Signal Layer'], // elements
  "Outfit",        // modelMode: 'Outfit'|'Parts'
  true             // autoGenerateModels
);
// → File

// ODB++
const odbFile = await eda.pcb_ManufactureData.getOpenDatabaseDoublePlusFile(
  "board",         // fileName
  "inch",          // unit: ESYS_Unit.INCH
  {                // otherData
    metallizedDrilledHoles: true,
    nonMetallizedDrilledHoles: true,
    drillTable: true,
    flyingProbeTestFile: true
  },
  [...],           // layers: [{layerId, mirror}]
  [...]            // objects
);

// 导出 DSN (给外部自动布线器)
const dsnFile = await eda.pcb_ManufactureData.getDsnFile("board.dsn");

// 全部制造数据打包
const mfgFile = await eda.pcb_ManufactureData.getManufactureData();

// 自动布线/布局数据导出
const autoRouteJson = await eda.pcb_ManufactureData.getAutoRouteJson();
const autoLayoutJson = await eda.pcb_ManufactureData.getAutoLayoutJson();

// 保存文件到本地
await eda.sys_FileSystem.saveFile(gerberFile, "board-gerber.zip");
```

## 7. 差分对操作 (生产级)

```typescript
// 获取现有差分对
const pairs = await eda.pcb_Drc.getAllDifferentialPairs();
// → Array<{ name, positiveNet, negativeNet }> | {[key:string]:any}
for (const pair of pairs) {
  console.log(pair.name, pair.positiveNet, pair.negativeNet);
}

// 创建差分对
await eda.pcb_Drc.createDifferentialPair("USB_D_P", "USB_D+", "USB_D-");

// 修改差分对
await eda.pcb_Drc.modifyDifferentialPairName("USB_D_P", "USB_DP");
await eda.pcb_Drc.modifyDifferentialPairPositiveNet("USB_DP", "DP");
await eda.pcb_Drc.modifyDifferentialPairNegativeNet("USB_DP", "DM");

// 删除
await eda.pcb_Drc.deleteDifferentialPair("USB_DP");
```

## 8. 规则配置读写 (生产级)

```typescript
// 读取当前规则
const rules = await eda.pcb_Drc.getCurrentRuleConfiguration();
// → {[key:string]:any} 对象, 包含所有规则参数

// 读取网络规则
const netRules = await eda.pcb_Drc.getNetRules();
// → Array<{[key:string]:any}>

// 覆写规则
await eda.pcb_Drc.overwriteCurrentRuleConfiguration(newRules);

// 保存规则配置
await eda.pcb_Drc.saveRuleConfiguration(rules, "MyRules", true);  // allowOverwrite=true
// 设为默认
await eda.pcb_Drc.setAsDefaultRuleConfiguration("MyRules");

// 获取所有配置
const allConfigs = await eda.pcb_Drc.getAllRuleConfigurations(false);
// includeSystem=false → 只返回用户自定义配置
```

## 9. 文档源操作 (直接读写 JSONL)

```typescript
// 读取整個PCB文档源
const source = await eda.sys_FileManager.getDocumentSource();
// 返回字符串, 每行一个 JSON 记录, 用 || 分隔 payload
// 格式: ["TYPE", {fields}] 或 ["TYPE", {fields}]||{"blobData"}

// 解析:
const lines = source.split('\n');
for (const line of lines) {
  const [jsonPart, payloadPart] = line.split('||');
  const record = JSON.parse(jsonPart);
  const type = record[0];     // "META", "LAYER", "NET", "VIA", "PAD", ...
  const fields = record[1];   // {gId, ...}
  // payload: base64 编码的二进制数据 (图片/3D模型等)
}

// 写回
await eda.sys_FileManager.setDocumentSource(modifiedSource);
```

## 10. 工程文件操作

```typescript
// 导出工程文件
const projectFile = await eda.sys_FileManager.getProjectFile(
  "my-project.epro",  // fileName
  undefined,           // password
  "epro"               // fileType: 'epro'|'epro2'
);
await eda.sys_FileSystem.saveFile(projectFile);

// 读取工程文件
const file = await eda.sys_FileSystem.openReadFileDialog(['epro', 'epro2']);
if (file) {
  // 导入
  await eda.sys_FileManager.importProjectByProjectFile(file);
}
```

## 11. 完整几何工具

```typescript
// 多边形
const poly = eda.pcb_MathPolygon.createPolygon([x1,y1, x2,y2, x3,y3, x4,y4]);

// 复杂多边形 (带挖空)
const complex = eda.pcb_MathPolygon.createComplexPolygon(outerPoly);
complex.addSource(innerHolePoly);  // 添加挖空

// 图像→多边形
const complexFromImage = await eda.pcb_MathPolygon.convertImageToComplexPolygon(
  imageBlob, imgWidth, imgHeight,
  tolerance?, simplification?, smoothing?, despeckling?,
  whiteAsBackground?, inversion?
);

// BBox 计算
const width = eda.pcb_MathPolygon.calculateBBoxWidth(polygonSource);
const height = eda.pcb_MathPolygon.calculateBBoxHeight(polygonSource);

// 多边形中心
const center = await eda.pcb_MathPolygon.getCenter(polygon);
// → {x, y}
```

## 12. 文档/标签页切换模式

```typescript
// 获取当前文档类型
const doc = await eda.dmt_SelectControl.getCurrentDocumentInfo();
// → { documentType: EDMT_EditorDocumentType, uuid, tabId, parentProjectUuid?, parentLibraryUuid? }

// 判断: doc.documentType === EDMT_EditorDocumentType.PCB (=3)

// 打开文档
const tabId = await eda.dmt_EditorControl.openDocument(pcbUuid);
// 切换焦点
await eda.dmt_EditorControl.activateDocument(tabId);
// 保存
await eda.pcb_Document.save(pcbUuid);
// 关闭
await eda.dmt_EditorControl.closeDocument(tabId);
```

## 13. 给你的 bridge 的完整 op 建议

```typescript
// 工程
'project.current'        → eda.dmt_Project.getCurrentProjectInfo()
'project.open'           → eda.dmt_Project.openProject(uuid)
'project.create'         → eda.dmt_Project.createProject(name, ...)
'project.export'         → eda.sys_FileManager.getProjectFile()
'project.import'         → eda.sys_FileManager.importProjectByProjectFile(file)

// 板/文档
'board.list'             → eda.dmt_Board.getAllBoardsInfo()
'board.create'           → eda.dmt_Board.createBoard()
'pcb.list'               → eda.dmt_Pcb.getAllPcbsInfo()
'pcb.create'             → eda.dmt_Pcb.createPcb()
'pcb.open'               → eda.dmt_EditorControl.openDocument(pcbUuid)
'pcb.save'               → eda.pcb_Document.save(uuid)

// SCH↔PCB 同步
'schematic.importChanges'  → eda.sch_Document.importChanges()
'pcb.importChanges'        → eda.pcb_Document.importChanges(schUuid?)

// 网表
'netlist.export'         → eda.pcb_Net.getNetlist(type?)
'netlist.apply'          → eda.pcb_Net.setNetlist(type, netlist)
'netlist.compare'        → eda.sys_Tool.netlistComparison(a, b)

// 图元批量创建
'pcb.createTracks'       → eda.pcb_PrimitiveLine.create(...)
'pcb.createVias'         → eda.pcb_PrimitiveVia.create(...)
'pcb.createPads'         → eda.pcb_PrimitivePad.create(...)
'pcb.placeComponents'    → eda.pcb_PrimitiveComponent.create(...)
'pcb.createTexts'        → eda.pcb_PrimitiveString.create(...)

// 图元查询
'pcb.getAllLines'        → eda.pcb_PrimitiveLine.getAll()
'pcb.getAllVias'         → eda.pcb_PrimitiveVia.getAll()
'pcb.getAllComponents'   → eda.pcb_PrimitiveComponent.getAll()
'pcb.getAllPours'        → eda.pcb_PrimitivePour.getAll()
'pcb.getAllNets'         → eda.pcb_Net.getAllNetsName()
'pcb.getPrimitivesByNet' → eda.pcb_Net.getAllPrimitivesByNet(name)
'pcb.getBBox'            → eda.pcb_Primitive.getPrimitivesBBox(ids)
'pcb.getSelected'        → eda.pcb_SelectControl.getAllSelectedPrimitives()

// 覆铜
'pcb.createCopperPour'   → eda.pcb_PrimitivePour.create(...)
'pcb.rebuildCopperPour'  → pour.rebuildCopperRegion()
'pcb.inspectCopperPour'  → pour.getCopperRegion() + getState_*()

// 图层/叠层
'pcb.configureLayers'    → eda.pcb_Layer.setTheNumberOfCopperLayers(n)
'pcb.getLayerInfo'       → eda.pcb_Layer.getAllLayers()
'pcb.createBoardOutline' → eda.pcb_PrimitiveLine.create("", BOARD_OUTLINE, ...)

// 规则
'rules.getCurrent'       → eda.pcb_Drc.getCurrentRuleConfiguration()
'rules.update'           → eda.pcb_Drc.overwriteCurrentRuleConfiguration(cfg)
'rules.detectDiffPairs'  → iterate nets + compare naming conventions
'rules.applyDiffPairs'   → eda.pcb_Drc.createDifferentialPair(...)
'pcb.applyConstraintGroups' → eda.pcb_Drc.createNetClass/createEqualLengthNetGroup

// DRC/DFM
'drc.run'                → eda.pcb_Drc.check(true, false, false)
'drc.runVerbose'         → eda.pcb_Drc.check(true, false, true) → Array<any>

// 制造输出
'pcb.exportGerber'       → eda.pcb_ManufactureData.getGerberFile(...)
'pcb.exportBom'          → eda.pcb_ManufactureData.getBomFile(...)
'pcb.exportCPL'          → eda.pcb_ManufactureData.getPickAndPlaceFile(...)
'pcb.export3D'           → eda.pcb_ManufactureData.get3DFile(...)
'pcb.exportODB'          → eda.pcb_ManufactureData.getOpenDatabaseDoublePlusFile(...)
'pcb.exportDsn'          → eda.pcb_ManufactureData.getDsnFile()
'pcb.exportMfg'          → eda.pcb_ManufactureData.getManufactureData()
'pcb.exportAutoRouteJson'→ eda.pcb_ManufactureData.getAutoRouteJson()

// 文档源 (直接操作 JSONL)
'pcb.getDocumentSource'  → eda.sys_FileManager.getDocumentSource()
'pcb.setDocumentSource'  → eda.sys_FileManager.setDocumentSource(src)

// 库
'library.search'         → eda.lib_Device.search(keyword)
'library.getByLcsc'      → eda.lib_Device.getByLcscIds(ids)

// 定位/导航
'locate.coordinates'     → eda.pcb_Document.navigateToCoordinates(x, y)
'locate.primitive'       → eda.pcb_SelectControl.doSelectPrimitives([id])
'locate.violation'       → doSelectPrimitives + navigateToRegion

// 文档源分析
'pcb.summary'            → 综合 getAll* 统计层数/网络数/器件数/...
'pcb.inspectGeometry'    → getPrimitivesInRegion + getPrimitiveAtPoint
'pcb.netLengths'         → eda.pcb_Net.getNetLength(netName)
```

## 14. 单位换算

```typescript
// SYS_Unit 本地静态方法 (不走RPC, 直接在扩展进程调用)
eda.sys_Unit.milToMm(1000)     // 25.4
eda.sys_Unit.mmToMil(25.4)     // 1000
eda.sys_Unit.inchToMil(1)      // 1000
eda.sys_Unit.milToInch(1000)   // 1
eda.sys_Unit.inchToMm(1)       // 25.4
eda.sys_Unit.mmToInch(25.4)    // 1

// 查询当前显示单位
const unit = await eda.sys_Unit.getFrontendDataUnit();
// → ESYS_Unit.MILLIMETER | CENTIMETER | INCH | MIL
```

## 15. 存储 (扩展配置持久化)

```typescript
// 读
const value = eda.sys_Storage.getExtensionUserConfig('myKey');
// → any | undefined

// 写
await eda.sys_Storage.setExtensionUserConfig('myKey', {foo: 'bar'});

// 删
await eda.sys_Storage.deleteExtensionUserConfig('myKey');

// 全清
await eda.sys_Storage.clearExtensionAllUserConfigs();
```
