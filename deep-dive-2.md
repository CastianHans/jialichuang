# 嘉立创 EDA 深度逆向 #2 — 在线库/Board生命周期/SCH-PCB同步/覆铜/DRC

> 来源: pro-api-types.d.ts (19129行) + DFM checker 源码 + 官方参考文档

---

## 1. 在线元件库完整调用链

### 1.1 搜索器件

```typescript
// 获取库 UUID
const sysLibUuid = await eda.lib_LibrariesList.getSystemLibraryUuid();

// 搜索 (分页参数: itemsOfPage, page — 从0开始)
const results = await eda.lib_Device.search(
  "STM32F103",           // keyword
  sysLibUuid,            // libraryUuid (可选, 不传=搜所有库)
  undefined,             // classification (分类过滤)
  undefined,             // symbolType (ELIB_SymbolType 过滤)
  20,                    // itemsOfPage (每页条数, 默认?)
  0                      // page (页码, 从0开始)
);
// → Promise<Array<ILIB_DeviceSearchItem>>
```

### 1.2 ILIB_DeviceSearchItem 完整结构

```typescript
interface ILIB_DeviceSearchItem {
  uuid: string;                    // 器件 UUID
  name: string;                    // 器件名称 (如 "STM32F103C8T6")
  libraryUuid: string;             // 所属库 UUID
  ordinal: number;                 // 排序序号
  classification?: ILIB_ClassificationIndex | Array<string>;  // 分类
  description?: string;            // 描述

  // 关联 (新版 — 直接返回完整对象)
  symbol: { name: string; uuid: string; libraryUuid: string; };    // 原理图符号 ⭐
  footprint?: { name: string; uuid: string; libraryUuid: string; }; // PCB封装 ⭐
  model3D?: { name: string; uuid: string; libraryUuid: string; };  // 3D模型 ⭐
  imageUuid?: string;              // 图片 UUID

  // 供应链/库存/价格 (已废弃, 改用 otherProperty)
  manufacturer?: string;           // @deprecated → otherProperty
  manufacturerId?: string;         // @deprecated → otherProperty
  supplier?: string;               // @deprecated → otherProperty
  supplierId?: string;             // @deprecated → otherProperty
  jlcInventory?: number;           // @deprecated → otherProperty
  jlcPrice?: number;               // @deprecated → otherProperty
  jlcLibraryCategory?: ELIB_DeviceJlcLibraryCategory; // @deprecated
  lcscInventory?: number;          // @deprecated → otherProperty
  lcscPrice?: number;              // @deprecated → otherProperty

  // 扩展属性 (替代上面废弃字段)
  otherProperty?: {
    [key: string]: boolean | number | string | undefined;
    // 常见 key: "Manufacturer", "Manufacturer Part Number", "Supplier",
    //            "Supplier Part Number", "LCSC Part Number",
    //            "JLC Inventory", "JLC Price", "LCSC Inventory", "LCSC Price",
    //            "Package", "Description", "Datasheet", ...
  };

  // 旧版兼容 (已废弃)
  symbolName?: string;             // @deprecated → symbol.name
  symbolUuid: string;              // @deprecated → symbol.uuid
  footprintName?: string;          // @deprecated → footprint.name
  footprintUuid: string;           // @deprecated → footprint.uuid
  model3DName?: string;            // @deprecated → model3D.name
  model3DUuid: string;             // @deprecated → model3D.uuid
}
```

### 1.3 ILIB_DeviceItem (get() 返回的完整器件)

```typescript
interface ILIB_DeviceItem {
  uuid: string;
  name: string;
  libraryUuid: string;
  readonly libraryType: ELIB_LibraryType.DEVICE;
  classification?: ILIB_ClassificationIndex | Array<string>;
  description?: string;
  subPartNames: [];                    // 子部件名数组

  association: ILIB_DeviceAssociationItem;  // 关联 ⭐
  property: ILIB_DeviceExtendPropertyItem;  // 扩展属性
}

// association 子结构:
interface ILIB_DeviceAssociationItem {
  symbol: { type: ELIB_SymbolType; uuid: string; libraryUuid: string; };
  footprint?: { uuid: string; libraryUuid: string; };
  images?: Array<string>;
  // 旧版
  symbolType: ELIB_SymbolType;   // @deprecated
  symbolUuid: string;            // @deprecated
  footprintUuid: string;         // @deprecated
}

// property 子结构:
interface ILIB_DeviceExtendPropertyItem {
  designator?: string;        // 默认位号 "R?"
  name?: string;
  manufacturer?: string;
  manufacturerId?: string;
  supplier?: string;
  supplierId?: string;
  net?: string;               // (电源器件用)
  addIntoBom?: boolean;
  addIntoPcb?: boolean;
  otherProperty?: { [key: string]: boolean | number | string | undefined; };
}
```

### 1.4 LCSC 编号反查

```typescript
// 单个
const device = await eda.lib_Device.getByLcscIds("C123456", sysLibUuid, false);
// → ILIB_DeviceSearchItem | undefined (allowMultiMatch=false 时返回单个)

// 批量
const devices = await eda.lib_Device.getByLcscIds(["C123456", "C789012"], sysLibUuid);
// → Array<ILIB_DeviceSearchItem>
```

### 1.5 从搜索结果到放置器件的完整路径

```javascript
// 1. 搜索
const results = await eda.lib_Device.search("STM32F103C8T6", sysLibUuid);
const device = results[0];

// 2. 放置到原理图 (用搜索结果直接放 — 包含 symbol UUID)
const comp = await eda.sch_PrimitiveComponent.create(
  device,     // ILIB_DeviceSearchItem 可以直接传!
  5000, 5000, // x, y (0.01inch)
  "",         // subPartName
  0,          // rotation
  false,      // mirror
  true,       // addIntoBom
  true        // addIntoPcb
);
// → ISCH_PrimitiveComponent | undefined

// 3. 设置位号
const asyncComp = comp.toAsync();
asyncComp.setState_Designator("U1");
await asyncComp.done();

// 4. 读取引脚
const pins = await eda.sch_PrimitiveComponent.getAllPinsByPrimitiveId(
  comp.getState_PrimitiveId()
);
// → ISCH_PrimitiveComponentPin[]
for (const pin of pins) {
  console.log(pin.getState_PinNumber(), pin.getState_PinName());
}

// 5. 画导线连接引脚
await eda.sch_PrimitiveWire.create(
  [pin1X, pin1Y, pin2X, pin2Y],  // line: number[]
  "VCC",                          // net
  null, null, null                // color, lineWidth, lineType
);
```

---

## 2. Board 创建生命周期

### 2.1 数据结构

```typescript
// Board = 一个完整的板子 (原理图+PCB的容器)
interface IDMT_BoardItem {
  readonly itemType: EDMT_ItemType.BOARD;
  name: string;                          // 板子名称 ⭐
  schematic: IDMT_SchematicItem;         // 下属原理图
  pcb: IDMT_PcbItem;                     // 下属 PCB
  parentProjectUuid: string;             // 所属工程 UUID
}

// Board 包含的原理图
interface IDMT_SchematicItem {
  readonly itemType: EDMT_ItemType.SCHEMATIC;
  uuid: string;
  name: string;
  parentProjectUuid: string;
  parentBoardName: string;               // 所属板子名
}

// Board 包含的 PCB
interface IDMT_PcbItem {
  readonly itemType: EDMT_ItemType.PCB | EDMT_ItemType.CBB_PCB;
  uuid: string;
  name: string;
  parentProjectUuid: string;
  parentBoardName?: string;
}
```

### 2.2 createBoard 返回值

```typescript
// 返回 Board 名称 (string)，不是 UUID！
createBoard(schematicUuid?: string, pcbUuid?: string): Promise<string | undefined>
```

**内部行为：**
- 不传参数 → 自动创建游离原理图 + 游离 PCB
- 传 schematicUuid → 关联已有原理图
- 传 pcbUuid → 关联已有 PCB
- 返回的是 **Board name**，`undefined` = 创建失败

### 2.3 创建 Board 的完整流程

```javascript
// 1. 确保工程已打开
const project = await eda.dmt_Project.getCurrentProjectInfo();
if (!project) throw new Error("No project open");

// 2. 创建 Board (自动创建 Schematic + PCB)
const boardName = await eda.dmt_Board.createBoard();
// → "Board_1" (自动命名) 或 undefined

// 3. 读取 Board 信息 (获取 schematic/pcb UUID)
const boards = await eda.dmt_Board.getAllBoardsInfo();
const board = boards.find(b => b.name === boardName);
// board = {
//   itemType: "Board",
//   name: "Board_1",
//   schematic: { uuid: "sch-uuid-xxx", name: "Schematic_1", ... },
//   pcb: { uuid: "pcb-uuid-yyy", name: "PCB_1", ... },
//   parentProjectUuid: "proj-uuid"
// }

// 4. 打开原理图 (操作前必须打开文档)
const schUuid = board.schematic.uuid;
const pages = await eda.dmt_Schematic.getAllSchematicPagesInfo();
const pageUuid = pages.find(p => p.parentSchematicUuid === schUuid)?.uuid;
const schTabId = await eda.dmt_EditorControl.openDocument(pageUuid);

// 5. 确认文档已激活
await eda.dmt_EditorControl.activateDocument(schTabId);
const doc = await eda.dmt_SelectControl.getCurrentDocumentInfo();
// doc.documentType === EDMT_EditorDocumentType.SCHEMATIC_PAGE (=1) ✓

// 6. 在原理图上操作...
// (放置器件、画线等)

// 7. 保存原理图
await eda.sch_Document.save();

// 8. 导出网表
const netlist = await eda.sch_Netlist.getNetlist("JLCEDA");

// 9. 打开 PCB 并导入变更
const pcbTabId = await eda.dmt_EditorControl.openDocument(board.pcb.uuid);
await eda.dmt_EditorControl.activateDocument(pcbTabId);
await eda.pcb_Document.importChanges(schUuid);
```

---

## 3. SCH → PCB 同步机制

### 3.1 importChanges 行为

```typescript
// PCB_Document.importChanges(uuid?: string): Promise<boolean>
// uuid = 原理图 UUID (同 Board 时可不传)
```

**内部匹配逻辑：**
1. 读取 SCH 所有 `ISCH_PrimitiveComponent`，按 **uniqueId** 建索引
2. 读取 PCB 所有 `IPCB_PrimitiveComponent`，按 **uniqueId** 建索引
3. **匹配**: SCH 组件在 PCB 中存在相同 uniqueId → 更新 footprint/designator/net/pads
4. **新增**: SCH 组件在 PCB 中不存在 → 创建新 PCB component (放默认原点附近)
5. **删除**: PCB 组件在 SCH 中不存在 → 从 PCB 移除
6. 根据 SCH 的网络连接关系，分配 PCB pad 的网络

**uniqueId 生成：**
- `SCH_PrimitiveComponent.create()` 时自动生成
- 格式类似 UUID，在 SCH 组件生命期内保持不变
- 修改 designator 不影响 uniqueId

### 3.2 同步后验证

```javascript
// 同步后检查
const pcbComps = await eda.pcb_PrimitiveComponent.getAll();
for (const comp of pcbComps) {
  console.log(
    comp.getState_Designator(),   // "U1"
    comp.getState_UniqueId(),     // "abc-123-def"
    comp.getState_Footprint()     // {libraryUuid, uuid, name}
  );
}

// 检查焊盘网络分配
const pads = comp.getState_Pads();
// → [{primitiveId: "gge123", net: "VCC", padNumber: "1"}, ...]
```

### 3.3 反向同步 (PCB → SCH)

```typescript
// SCH_Document.importChanges(): Promise<boolean>
// 从 PCB 反向导入变更到原理图
```

---

## 4. 动态覆铜 worker 协议

### 4.1 创建覆铜边框

```javascript
// 1. 创建多边形
const poly = eda.pcb_MathPolygon.createPolygon([
  0, 0,  10000, 0,  10000, 8000,  0, 8000  // 矩形 10000x8000 mil
]);

// 2. 创建覆铜边框
const pour = await eda.pcb_PrimitivePour.create(
  "GND",                    // net
  EPCB_LayerId.TOP,        // layer = 1
  poly,                     // complexPolygon
  "solid",                  // pourFillMethod
  false,                    // preserveSilos
  "GND_POUR",              // pourName
  0,                        // pourPriority
  10,                       // lineWidth (mil)
  false                     // primitiveLock
);

// 3. 触发铺铜 (重建 = 真正生成覆铜填充)
const poured = await pour.rebuildCopperRegion();
// → IPCB_PrimitivePoured | undefined
```

### 4.2 rebuildCopperRegion 内部行为

- **同步/异步**: `rebuildCopperRegion()` 是 async 方法，跑在内部引擎中
- **结果**: 返回 `IPCB_PrimitivePoured` (覆铜填充图元)
- **如果失败**: 返回 `undefined` (如多边形自交、网络不存在等)
- **进度**: 无公开进度回调 (大板子可能耗时)
- **批量**: 可以并行 `Promise.all(pours.map(p => p.rebuildCopperRegion()))`

### 4.3 检查覆铜结果

```javascript
const poured = await pour.rebuildCopperRegion();
if (!poured) throw new Error("Rebuild failed");

// 获取填充区域列表
const fills = poured.getState_PourFills();
// → Array<IPCB_PrimitivePouredPourFill>
// 每个 fill: { id, ... }
// 如果 fills 为空 = 铺铜失败 (可能孤岛被移除)

// 获取关联的覆铜边框
const pourId = poured.getState_PourPrimitiveId();

// 将某个填充转成独立填充图元 (用于检查/编辑)
const fill = await poured.convertToFill(fills[0].id);
// → IPCB_PrimitiveFill | undefined

// 删除某个填充区域
await poured.deletePourFills(fills[0].id);

// 添加阻焊开窗
const maskFill = await poured.addSolderMaskFill(fills[0].id);
```

### 4.4 覆铜完整闭环

```javascript
// 创建 → 重建 → 检查 → (修改 → 重建)* → 确认
async function pourGND(boardOutline) {
  const pour = await eda.pcb_PrimitivePour.create("GND", 1, boardOutline, "solid", false, "GND", 0, 10, false);
  let poured = await pour.rebuildCopperRegion();
  if (!poured) return false;

  // 验证填充
  const fills = poured.getState_PourFills();
  console.log(`GND pour: ${fills.length} fill regions`);

  // 如需修改:
  // const asyncP = pour.toAsync();
  // asyncP.setState_PourFillMethod("45grid");
  // await asyncP.done();
  // poured = await pour.rebuildCopperRegion();  // 重建!

  return fills.length > 0;
}
```

---

## 5. DRC 检查完整结构

### 5.1 调用方式

```javascript
// 模式 1: 布尔返回
const passed = await eda.pcb_Drc.check(true, false, false);
// → boolean: true=通过, false=有错误

// 模式 2: 详细返回 (includeVerboseError=true)
const errors = await eda.pcb_Drc.check(true, false, true);
// → Array<any>  ← 每个元素是一条 DRC 错误
```

### 5.2 DFM Checker 验证的结构 (从官方源码逆向)

```typescript
interface PcbDfmResult {
  timestamp: number;
  results: CheckResult[];
  passed: boolean;
  errorCount: number;
  warningCount: number;
}

interface CheckResult {
  number: number;           // 检查项编号
  item: string;             // 检查项名称 (中文)
  actualValue: string;      // 实测值
  standardValue: string;    // 标准值
  result: 'success' | 'warning' | 'error';
  violations?: ViolationCoord[];
}

interface ViolationCoord {
  x: number; y: number;     // 坐标 (mil)
  id: string;               // primitiveId → 可点击定位
  reason: string;           // 违规原因
  type: string;             // 类型
  locateType?: string;      // "rect" 等
}
```

### 5.3 DRC 规则读写

```javascript
// 读当前规则
const rules = await eda.pcb_Drc.getCurrentRuleConfiguration();
// → { [key: string]: any }
// 结构取决于当前选择的规则配置

// 读所有网络规则
const netRules = await eda.pcb_Drc.getNetRules();
// → Array<{ [key: string]: any }>
// 每个元素是一个网络的规则:
// { net: "GND", trackWidth: 10, clearance: 8, viaHoleSize: 12, ... }

// 覆写规则
await eda.pcb_Drc.overwriteCurrentRuleConfiguration(newRules);
await eda.pcb_Drc.overwriteNetRules(newNetRules);
```

---

## 6. 你的 bridge 完整 op 参数设计

### 6.1 库操作

```json
// library.search
{ "op": "library.search", "keyword": "STM32", "page": 0, "pageSize": 20 }
// → { devices: ILIB_DeviceSearchItem[], totalCount?: number }

// library.getByLcsc
{ "op": "library.getByLcsc", "lcscIds": ["C123456"] }
// → { devices: ILIB_DeviceSearchItem[] }

// library.getDevice
{ "op": "library.getDevice", "deviceUuid": "...", "libraryUuid": "..." }
// → { device: ILIB_DeviceItem }
```

### 6.2 Board/文档

```json
// board.create
{ "op": "board.create" }
// → { boardName: "Board_1", schematic: {uuid, name}, pcb: {uuid, name} }

// document.open
{ "op": "document.open", "uuid": "...", "type": "schematic_page" | "pcb" }
// → { tabId: "..." }
```

### 6.3 SCH 操作

```json
// sch.placeComponent
{ "op": "sch.placeComponent", "device": {libraryUuid, uuid}, "x": 5000, "y": 5000, "designator": "U1" }
// → { primitiveId, pins: [{number, name}] }

// sch.createWire
{ "op": "sch.createWire", "points": [x1,y1,x2,y2,...], "net": "VCC" }
// → { primitiveId }

// sch.getNetlist
{ "op": "sch.getNetlist", "type": "JLCEDA" }
// → { netlist: "..." }
```

### 6.4 PCB 同步

```json
// pcb.importChanges
{ "op": "pcb.importChanges", "schematicUuid": "..." }
// → { success: true, added: 5, updated: 3, deleted: 1 }

// pcb.getComponents (同步后验证)
{ "op": "pcb.getComponents" }
// → { components: [{designator, uniqueId, footprint, pads}] }
```

### 6.5 覆铜

```json
// pcb.createCopperPour
{ "op": "pcb.createCopperPour", "net": "GND", "layer": 1, "polygon": [x1,y1,...], "method": "solid" }
// → { pourId }

// pcb.rebuildCopperPour
{ "op": "pcb.rebuildCopperPour", "pourId": "..." }
// → { pouredId, fillCount: 3 }

// pcb.inspectCopperPour
{ "op": "pcb.inspectCopperPour", "pourId": "..." }
// → { pourId, net, layer, fillCount, fills: [...] }
```

### 6.6 DRC

```json
// drc.run
{ "op": "drc.run" }
// → { passed: true, errorCount: 0, warningCount: 0, results?: [...] }

// drc.runVerbose
{ "op": "drc.runVerbose" }
// → { passed: false, results: [{item, actualValue, standardValue, result, violations}] }
```
