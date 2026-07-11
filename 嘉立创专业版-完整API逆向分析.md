# 嘉立创专业版 完整 API 逆向分析

> 版本 V3.2.69 | 安装路径 `E:\lceda-pro\` | 2026-07-11
> 面向 MCP Server 开发的全量接口参考

---

## 目录

1. [总体架构](#1-总体架构)
2. [接入方式：WebSocket Bridge](#2-接入方式websocket-bridge)
3. [原理图→PCB 关联机制 (importChanges)](#3-原理图pcb-关联机制-importchanges)
4. [PCB 图元 API 完整签名](#4-pcb-图元-api-完整签名)
5. [原理图图元 API 完整签名](#5-原理图图元-api-完整签名)
6. [图层/叠层/规则 API](#6-图层叠层规则-api)
7. [覆铜/铺铜 API](#7-覆铜铺铜-api)
8. [制造输出 API](#8-制造输出-api)
9. [文档树/工程管理 API](#9-文档树工程管理-api)
10. [库搜索 API](#10-库搜索-api)
11. [系统工具 API](#11-系统工具-api)
12. [完整枚举值](#12-完整枚举值)
13. [坐标系与单位](#13-坐标系与单位)
14. [MCP Server 设计方案](#14-mcp-server-设计方案)
15. [缺口分析](#15-缺口分析)

---

## 1. 总体架构

```
MCP Server → HTTP POST /execute → Bridge Server (Node.js :49620-49629) → WebSocket → EasyEDA Pro
```

EasyEDA Pro 是 Electron 应用，主窗口 sandboxed。所有 API 通过全局 `eda` 对象调用，底层走 `MessageBus2` RPC。

**启动 Bridge：**
```bash
cd easyeda-api-skill && npm install && node scripts/bridge-server.mjs &
```

**执行代码：**
```bash
curl -X POST http://localhost:49620/execute \
  -H "Content-Type: application/json" \
  -d '{"code": "return await eda.dmt_Project.getCurrentProjectInfo();"}'
```

**执行上下文：**
```javascript
async function(eda) {
  // eda 是全局 API 对象
  // 必须用 return 返回结果 (console.log 不会被捕获)
  // 所有 Promise 必须 await
}
```

---

## 2. 接入方式：WebSocket Bridge

### 启动流程
1. 启动 Bridge Server：端口 49620-49629
2. EasyEDA 安装 `run-api-gateway.eext` 扩展
3. 扩展自动扫描端口并完成 WebSocket 握手
4. 健康检查：`GET /health` → `{"service":"easyeda-bridge","edaConnected":true}`
5. 多窗口：`GET /eda-windows` 列出，`POST /eda-windows/select` 选择

### 扩展沙箱限制
以下在扩展代码中均为 `undefined`：
`document`, `fetch`, `WebSocket`, `XMLHttpRequest`, `localStorage`, `sessionStorage`, `indexedDB`, `window`, `location`, `navigator`

可用：`console`, `eda.*`, 标准 JS 对象

---

## 3. 原理图→PCB 关联机制 (importChanges)

### 核心：`uniqueId` 是跨 SCH/PCB 的唯一钥匙

```
SCH_PrimitiveComponent              PCB_PrimitiveComponent
├── designator: "R1"               ├── designator: "R1"
├── uniqueId: "abc-123"  ═══link═══→ ├── uniqueId: "abc-123"
├── component: {libUuid, uuid}     ├── footprint: {libUuid, uuid}
├── footprint: {libUuid, uuid}     ├── component: {libUuid, uuid}
└── pins: [{number, net}]          └── pads: [{primitiveId, net, padNumber}]
```

### importChanges API

```typescript
// PCB ← 原理图
PCB_Document.importChanges(uuid?: string): Promise<boolean>
// uuid = 原理图 UUID，默认同 Board 下的原理图；游离 PCB 返回 false

// 原理图 ← PCB
SCH_Document.importChanges(): Promise<boolean>
```

**内部流程：**
1. 读取原理图所有 component（按 uniqueId 索引）
2. PCB 已有 component：uniqueId 匹配 → 更新 footprint/designator/net/pads；匹配不上 → 删除
3. 原理图新增 component → 在 PCB 创建新 component，放默认位置
4. 根据原理图 net 连接，给 PCB pads 分配网络

### PCB Component 内部构造器

```typescript
class IPCB_PrimitiveComponent {
  constructor(
    component: { libraryUuid: string; uuid: string },
    layer: TPCB_LayersOfComponent,  // TOP=1, BOTTOM=2
    x: number, y: number,
    rotation?: number,
    primitiveLock?: boolean,
    addIntoBom?: boolean,
    primitiveId?: string,
    footprint?: { libraryUuid: string; uuid: string },
    model3D?: { libraryUuid: string; uuid: string },
    designator?: string,        // 位号 "R1"
    pads?: Array<{ primitiveId: string; net: string; padNumber: string }>,
    name?: string,
    uniqueId?: string,          // ← 跨 SCH/PCB 关联 KEY
    manufacturer?: string,
    manufacturerId?: string,
    supplier?: string,
    supplierId?: string,
    otherProperty?: { [key: string]: string | number | boolean }
  )
}
```

### autoRouting / autoLayout 参数结构

```typescript
autoRouting(props?: {
  uuids?: Array<string>;
  netlist?: {
    component: {
      [uniqueId: string]: {
        pinInfoMap: {
          [pinNumber: string]: {
            name: string; number: string; net: string;
            props: { 'Pin Number': string };
          };
        };
      };
    };
  };
}): Promise<any>

autoLayout(props?: { /* 同上结构 */ }): Promise<any>
```

---

## 4. PCB 图元 API 完整签名

> PCB 坐标单位：**1 mil** (1mm ≈ 39.37)

### 4.1 走线 PCB_PrimitiveLine

```
create(net: string, layer: TPCB_LayersOfLine, startX: number, startY: number,
       endX: number, endY: number, lineWidth?: number, primitiveLock?: boolean)
       → Promise<IPCB_PrimitiveLine | undefined>

delete(ids: string | IPCB_PrimitiveLine | Array<string> | Array<IPCB_PrimitiveLine>) → Promise<boolean>
get(id: string) → Promise<IPCB_PrimitiveLine | undefined>
get(ids: string[]) → Promise<Array<IPCB_PrimitiveLine>>
getAll(net?: string, layer?: TPCB_LayersOfLine, primitiveLock?: boolean) → Promise<Array<IPCB_PrimitiveLine>>
getAllPrimitiveId(net?, layer?, primitiveLock?) → Promise<Array<string>>
modify(id, property: {net?,layer?,startX?,startY?,endX?,endY?,lineWidth?,primitiveLock?}) → Promise<IPCB_PrimitiveLine | undefined>
getAdjacentPrimitives(id) → Promise<Array<IPCB_PrimitiveLine | IPCB_PrimitiveVia | IPCB_PrimitiveArc>>
getEntireTrack(id, includeVias?: boolean) → Promise<Array<IPCB_PrimitiveLine>>
```

### 4.2 过孔 PCB_PrimitiveVia

```
create(net: string, x: number, y: number,
       holeDiameter: number,     // 内径 mil
       diameter: number,          // 外径 mil
       viaType?: EPCB_PrimitiveViaType,  // VIA=0, BLIND=1, SUTURE=2
       designRuleBlindViaName?: string | null,
       solderMaskExpansion?: IPCB_PrimitiveSolderMaskAndPasteMaskExpansion | null,
       primitiveLock?: boolean) → Promise<IPCB_PrimitiveVia | undefined>

delete / get / getAll / modify / getAdjacentPrimitives (同上模式)
```

### 4.3 焊盘 PCB_PrimitivePad

```
create(layer: TPCB_LayersOfPad, x: number, y: number,
       pad: TPCB_PrimitivePadShape,    // {shape: ELLIPSE|RECT|OVAL|POLYGON, width, height}
       padNumber: string,              // "1"
       net?: string,
       padType?: EPCB_PrimitivePadType,  // NORMAL=0, TEST=1, MARK_POINT=2
       hole?: TPCB_PrimitivePadHole,     // {shape: ROUND|RECT|SLOT, width, height}
       holeOffsetX?, holeOffsetY?, holeRotation?,
       metallization?: boolean,
       specialPad?: TPCB_PrimitiveSpecialPadShape,
       solderMaskAndPasteMaskExpansion?: IPCB_PrimitiveSolderMaskAndPasteMaskExpansion | null,
       heatWelding?: IPCB_PrimitivePadHeatWelding,
       rotation?: number, primitiveLock?: boolean)
       → Promise<IPCB_PrimitivePad | undefined>

delete / get / getAll / modify / getConnectedPrimitives (同上模式)
```

### 4.4 器件 PCB_PrimitiveComponent

```
create(component: {libraryUuid, uuid} | ILIB_DeviceItem,
       layer: TPCB_LayersOfComponent,  // TOP=1, BOTTOM=2
       x: number, y: number,
       rotation?: number, primitiveLock?: boolean)
       → Promise<IPCB_PrimitiveComponent | undefined>

delete / get / getAll / getAllPrimitiveId / modify / placeComponentWithMouse
getAllPinsByPrimitiveId(primitiveId) → Promise<Array<IPCB_PrimitiveComponentPad> | undefined>
getAllPropertyNames() → Promise<string[]>
setAttribute(id, key, value)
```

### 4.5 圆弧走线 PCB_PrimitiveArc

```
create(net, layer, startX, startY, endX, endY, arcAngle: number,
       lineWidth?, interactiveMode?: EPCB_PrimitiveArcInteractiveMode, primitiveLock?)
       → Promise<IPCB_PrimitiveArc | undefined>
// interactiveMode: TWO_POINT_ARC=1, CENTER_ARC=2

delete / get / getAll / modify / getAdjacentPrimitives / getEntireTrack
```

### 4.6 覆铜边框 PCB_PrimitivePour

```
create(net: string, layer: TPCB_LayersOfCopper,
       complexPolygon: IPCB_Polygon,
       pourFillMethod?: EPCB_PrimitivePourFillMethod,  // "solid"|"45grid"|"90grid"
       preserveSilos?: boolean,
       pourName?: string, pourPriority?: number,
       lineWidth?: number, primitiveLock?: boolean)
       → Promise<IPCB_PrimitivePour | undefined>

delete / get / getAll / getAllPrimitiveId / modify
getCopperRegion() → Promise<IPCB_PrimitivePoured | undefined>
rebuildCopperRegion() → Promise<IPCB_PrimitivePoured | undefined>
convertToFill() / convertToPolyline() / convertToRegion()
```

### 4.7 覆铜填充 PCB_PrimitivePoured

```
delete(ids) / getAll() → Promise<Array<IPCB_PrimitivePoured>>
convertToFill(pourFillId) → Promise<IPCB_PrimitiveFill | undefined>
addSolderMaskFill(pourFillId) → Promise<IPCB_PrimitiveFill | undefined>
deletePourFills(pourFillIds) → Promise<boolean>
```

### 4.8 填充 PCB_PrimitiveFill

```
create(layer: TPCB_LayersOfFill, complexPolygon: IPCB_Polygon,
       net?: string,
       fillMode?: EPCB_PrimitiveFillMode,  // SOLID=0, MESH=1, INNER_ELECTRICAL_LAYER=2
       lineWidth?: number, primitiveLock?: boolean)
       → Promise<IPCB_PrimitiveFill | undefined>

delete / get / getAll / modify
convertToPolyline() / convertToPour() / convertToRegion()
```

### 4.9 折线 PCB_PrimitivePolyline

```
create(net, layer: TPCB_LayersOfLine, polygon: IPCB_Polygon,
       lineWidth?, primitiveLock?)
       → Promise<IPCB_PrimitivePolyline | undefined>

delete / get / getAll / modify
convertToFill() / convertToPour() / convertToRegion()
```

### 4.10 区域 PCB_PrimitiveRegion

```
create(layer: TPCB_LayersOfRegion, complexPolygon: IPCB_Polygon,
       ruleType?: EPCB_PrimitiveRegionRuleType[],  // NO_COMPONENTS=2,NO_VIAS=3,NO_WIRES=5,NO_FILLS=6,NO_POURS=7,NO_INNER_ELECTRICAL_LAYERS=8,FOLLOW_REGION_RULE=9
       regionName?: string, lineWidth?: number, primitiveLock?: boolean)
       → Promise<IPCB_PrimitiveRegion | undefined>

delete / get / getAll / modify
convertToFill() / convertToPolyline() / convertToPour()
```

### 4.11 文本 PCB_PrimitiveString

```
create(layer: TPCB_LayersOfImage, x, y, text: string,
       fontFamily?, fontSize?, lineWidth?,
       alignMode?: EPCB_PrimitiveStringAlignMode,
       rotation?, reverse?, expansion?, mirror?, primitiveLock?)
       → Promise<IPCB_PrimitiveString | undefined>
// alignMode: LEFT_TOP=1..RIGHT_BOTTOM=9, CENTER=5
```

### 4.12 尺寸标注 PCB_PrimitiveDimension

```
create(dimensionType: EPCB_PrimitiveDimensionType,  // LENGTH, RADIUS, ANGLE
       coordinateSet: TPCB_PrimitiveDimensionCoordinateSet,
       layer?: TPCB_LayersOfDimension,
       unit?: ESYS_Unit, lineWidth?, precision?, primitiveLock?)
       → Promise<IPCB_PrimitiveDimension | undefined>
```

### 4.13 图像 PCB_PrimitiveImage

```
create(x, y, complexPolygon: TPCB_PolygonSourceArray | IPCB_Polygon | IPCB_ComplexPolygon,
       layer: TPCB_LayersOfImage, width?, height?, rotation?, horizonMirror?, primitiveLock?)
```

### 4.14 内嵌对象 PCB_PrimitiveObject

```
create(layer: TPCB_LayersOfObject, topLeftX, topLeftY,
       binaryData: string, width, height,
       rotation?, mirror?, fileName?, primitiveLock?)
```

### 4.15 属性图元 PCB_PrimitiveAttribute

```
get / getAll(parentPrimitiveId?) / modify / delete
// 属性的 key/value/visibility 通过 modify 修改
```

### 4.16 选择控制 PCB_SelectControl

```
getAllSelectedPrimitiveIds() → Promise<string[]>
getAllSelectedPrimitives() → Promise<IPCB_Primitive[]>
doSelectPrimitives(ids: string | string[]) → Promise<boolean>
doCrossProbeSelect(components?, pins?, nets?, highlight?, select?) → Promise<boolean>
clearSelected() → Promise<boolean>
getCurrentMousePosition() → Promise<{x,y} | undefined>
```

### 4.17 修改图元的标准模式

```javascript
const prim = await eda.pcb_PrimitiveLine.get([id])
const a = prim.toAsync()
a.setState_StartX(100).setState_Net("VCC")
await a.done()
// 或撤销: await a.reset()
```

---

## 5. 原理图图元 API 完整签名

> 原理图坐标单位：**0.01 inch (10 mil)** (1mm ≈ 3.937)

### 5.1 器件 SCH_PrimitiveComponent

```
create(component: {libraryUuid, uuid} | ILIB_DeviceItem,
       x: number, y: number,
       subPartName?: string, rotation?: number, mirror?: boolean,
       addIntoBom?: boolean, addIntoPcb?: boolean)
       → Promise<ISCH_PrimitiveComponent | undefined>

// 网络标识符快捷创建
createNetFlag(identification: 'Power'|'Ground'|'AnalogGround'|'ProtectGround',
              net, x, y, rotation?, mirror?)
createNetPort(direction: 'IN'|'OUT'|'BI', net, x, y, rotation?, mirror?)
createShortCircuitFlag(x, y, rotation?, mirror?)

modify(id, property: {x?,y?,rotation?,mirror?,addIntoBom?,addIntoPcb?,
        designator?,name?,uniqueId?,manufacturer?,manufacturerId?,
        supplier?,supplierId?,otherProperty?})
```

### 5.2 导线 SCH_PrimitiveWire

```
create(line: number[] | number[][],  // [x1,y1,x2,y2] 或 [[x1,y1],[x2,y2]]
       net?: string, color?: string|null, lineWidth?: number|null,
       lineType?: ESCH_PrimitiveLineType|null)  // SOLID=0,DASHED=1,DOTTED=2,DOT_DASHED=3
```

### 5.3 其他原理图图元

```
// 总线
SCH_PrimitiveBus.create(busName, line, color?, lineWidth?, lineType?)

// 圆
SCH_PrimitiveCircle.create(centerX, centerY, radius, color?, fillColor?, lineWidth?, lineType?, fillStyle?)

// 圆弧
SCH_PrimitiveArc.create(startX, startY, referenceX, referenceY, endX, endY, color?, fillColor?, lineWidth?, lineType?)

// 引脚
SCH_PrimitivePin.create(x, y, pinNumber, pinName?, rotation?, pinLength?, pinColor?, pinShape?, pinType?)

// 多边形
SCH_PrimitivePolygon.create(line: number[], color?, fillColor?, lineWidth?, lineType?)

// 矩形
SCH_PrimitiveRectangle.create(topLeftX, topLeftY, width, height, cornerRadius?, rotation?, color?, fillColor?, lineWidth?, lineType?, fillStyle?)

// 文本
SCH_PrimitiveText.create(x, y, content, rotation?, textColor?, fontName?, fontSize?, bold?, italic?, underLine?, alignMode?)

// 复用模块 (CBB)
SCH_PrimitiveComponent3.create / createCbbSymbol / placeCbbSchematicPage
```

### 5.4 原理图 DRC / 网表 / 制造

```
SCH_Drc.check(strict, userInterface, includeVerboseError) → Promise<boolean>
SCH_Netlist.getNetlist(type?: ESYS_NetlistType) → Promise<string>
SCH_Netlist.setNetlist(type, netlist) → Promise<void>
SCH_ManufactureData.getBomFile / getNetlistFile / getExportDocumentFile / getAssemblyVariantsConfigs
SCH_Document.save / importChanges / autoRouting / autoLayout
```

---

## 6. 图层/叠层/规则 API

### 6.1 图层管理 PCB_Layer

```
selectLayer(layer) → Promise<boolean>                    // 选中图层
setLayerVisible(layer?, setOtherInvisible?)              // 设置可见
setLayerInvisible(layer?, setOtherVisible?)              // 设置不可见
lockLayer(layer?) → Promise<boolean>                     // 锁定
unlockLayer(layer?) → Promise<boolean>                   // 解锁

setTheNumberOfCopperLayers(n: 2|4|6|8|10|12|14|16|18|20|22|24|26|28|30|32)  // 设置铜箔层数
getTheNumberOfCopperLayers() → Promise<number>

modifyLayer(layer, property: {name?, type?: SIGNAL|INTERNAL_ELECTRICAL, color?, transparency?: 0-100})
// 仅内层允许改类型；INTERNAL_ELECTRICAL = 负片/电源层

addCustomLayer() → Promise<TPCB_LayersOfCustom | undefined>
removeLayer(layer: TPCB_LayersOfCustom) → Promise<boolean>
getAllLayers() → Promise<Array<IPCB_LayerItem>>

setPcbType(type: EPCB_PcbPlateType)  // NORMAL=1, FPC=2
setLayerColorConfiguration(config)   // JLCEDA=1, ALTIUM=2, PADS=3, KICAD=4
setInactiveLayerDisplayMode(mode)    // NORMAL=0, GRAY=1, HIDE=2
setInactiveLayerTransparency(pct)    // 0-100
```

### 6.2 图层 ID 完整映射

```
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
SUBSTRATE_1=53 ... SUBSTRATE_32=84
TOP_STIFFENER=58, BOTTOM_STIFFENER=59
COMPONENT_SHAPE=99, LEAD_SHAPE=100
```

### 6.3 DRC 规则管理 PCB_Drc

```
// DRC 检查
check(strict: boolean, userInterface: boolean, includeVerboseError: boolean)
  → Promise<boolean> 或 (verboseError=true 时) Promise<Array<any>>

// 规则配置
getCurrentRuleConfigurationName() → Promise<string | undefined>
getCurrentRuleConfiguration() → Promise<{[key:string]:any} | undefined>
getRuleConfiguration(name) → Promise<{[key:string]:any} | undefined>
getAllRuleConfigurations(includeSystem?) → Promise<Array<{[key:string]:any}>>
saveRuleConfiguration(config, name, allowOverwrite?) → Promise<boolean>
renameRuleConfiguration(old, new) → Promise<boolean>
deleteRuleConfiguration(name) → Promise<boolean>
getDefaultRuleConfigurationName() → Promise<string | undefined>
setAsDefaultRuleConfiguration(name) → Promise<boolean>

// 规则覆写
getNetRules() → Promise<Array<{[key:string]:any}>>
overwriteNetRules(rules: Array<{[key:string]:any}>) → Promise<boolean>
getNetByNetRules() → Promise<{[key:string]:any}>
overwriteNetByNetRules(rules: {[key:string]:any}) → Promise<boolean>
getRegionRules() → Promise<Array<{[key:string]:any}>>
overwriteRegionRules(rules: Array<{[key:string]:any}>) → Promise<boolean>
overwriteCurrentRuleConfiguration(config: {[key:string]:any}) → Promise<boolean>
```

### 6.4 网络类 / 差分对 / 等长组 / 焊盘对组

```typescript
// 网络类
createNetClass(name, nets: string[], color: {r,g,b,alpha}|null) → Promise<boolean>
deleteNetClass(name) → Promise<boolean>
modifyNetClassName(old, new) → Promise<boolean>
addNetToNetClass(className, net: string|string[]) → Promise<boolean>
removeNetFromNetClass(className, net: string|string[]) → Promise<boolean>
getAllNetClasses() → Promise<Array<IPCB_NetClassItem>>

// 差分对
createDifferentialPair(name, positiveNet, negativeNet) → Promise<boolean>
deleteDifferentialPair(name) → Promise<boolean>
modifyDifferentialPairName(old, new) → Promise<boolean>
modifyDifferentialPairPositiveNet(name, net) → Promise<boolean>
modifyDifferentialPairNegativeNet(name, net) → Promise<boolean>
getAllDifferentialPairs() → Promise<Array<IPCB_DifferentialPairItem> | {[key:string]:any}>

// 等长网络组
createEqualLengthNetGroup(name, nets: string[], color) → Promise<boolean>
deleteEqualLengthNetGroup(name) → Promise<boolean>
modifyEqualLengthNetGroupName(old, new) → Promise<boolean>
addNetToEqualLengthNetGroup(groupName, net) → Promise<boolean>
removeNetFromEqualLengthNetGroup(groupName, net) → Promise<boolean>
getAllEqualLengthNetGroups() → Promise<Array<IPCB_EqualLengthNetGroupItem>>

// 焊盘对组
createPadPairGroup(name, padPairs: [string,string][]) → Promise<boolean>
// padPair 格式: "e0" (游离焊盘), "R1:1" (器件焊盘)
deletePadPairGroup / modifyPadPairGroupName / addPadPairToPadPairGroup
removePadPairFromPadPairGroup / getAllPadPairGroups
getPadPairGroupMinWireLength(groupName)
```

---

## 7. 覆铜/铺铜 API

### 7.1 IPCB_PrimitivePour (覆铜边框)

```typescript
// 属性
getState_PrimitiveId() → string
getState_PrimitiveType() → EPCB_PrimitiveType           // POUR
getState_Net() → string
getState_Layer() → TPCB_LayersOfCopper
getState_ComplexPolygon() → IPCB_Polygon
getState_PourFillMethod() → EPCB_PrimitivePourFillMethod  // "solid"|"45grid"|"90grid"
getState_PreserveSilos() → boolean                       // 保留孤岛
getState_PourName() → string
getState_PourPriority() → number
getState_LineWidth() → number
getState_PrimitiveLock() → boolean

// 操作
toAsync() → IPCB_PrimitivePour
setState_Net(net) / setState_Layer(layer) / setState_ComplexPolygon(poly) / ...
done() → Promise<IPCB_PrimitivePour>
reset() → Promise<IPCB_PrimitivePour>

getCopperRegion() → Promise<IPCB_PrimitivePoured | undefined>
rebuildCopperRegion() → Promise<IPCB_PrimitivePoured | undefined>
convertToFill() / convertToPolyline() / convertToRegion()
```

### 7.2 IPCB_PrimitivePoured (覆铜填充)

```typescript
getState_PourPrimitiveId() → string         // 关联的覆铜边框 ID
getState_PourFills() → Array<IPCB_PrimitivePouredPourFill>

// 操作
convertToFill(pourFillId) → Promise<IPCB_PrimitiveFill | undefined>
addSolderMaskFill(pourFillId) → Promise<IPCB_PrimitiveFill | undefined>
deletePourFills(pourFillIds) → Promise<boolean>
```

### 7.3 多边形创建

```typescript
// 创建简单多边形
PCB_MathPolygon.createPolygon(polygon: number[]) → IPCB_Polygon | undefined
// polygon 格式: [x1, y1, x2, y2, x3, y3, ...]

// 创建复杂多边形 (可用于挖空)
PCB_MathPolygon.createComplexPolygon(source) → IPCB_ComplexPolygon | undefined
// 先创建多个 IPCB_Polygon，用 IPCB_ComplexPolygon.addSource() 合并
IPCB_ComplexPolygon.addSource(polygon | polygon[] | IPCB_Polygon[])

// 图像转多边形
PCB_MathPolygon.convertImageToComplexPolygon(blob, width, height, tolerance?, ...)
```

---

## 8. 制造输出 API

### PCB_ManufactureData

```
getGerberFile(fileName?, silk?, unit?, format?, drilledHole?, layerConfigs?, objectConfigs?) → File
get3DFile(fileName?, fileType?:'step'|'obj', elements?, modelMode?:'Outfit'|'Parts', autoGen?) → File
get3DShellFile(fileName?, fileType?:'stl'|'step'|'obj') → File
getPickAndPlaceFile(fileName?, fileType?:'xlsx'|'csv', unit?) → File
getBomFile(fileName?, fileType?:'xlsx'|'csv', template?, filterOptions?, statistics?, property?, columns?) → File
getDxfFile(fileName?, layers?, objects?) → File
getPDFFile(fileName?, outputMethod?, ...) → File
getNetlistFile(fileName?, netlistType?) → File
getTestPointFile(fileName?, fileType?:'xlsx'|'csv') → File
getFlyingProbeTestFile(fileName?) → File
getDsnFile(fileName?) → File                             // Specctra DSN
getIPCFile(fileName?) → File                             // IPC-D-356A
getADFile / getPADSFile → File                           // Altium/PADS 导出
getOpenDatabaseDoublePlusFile(...) → File                 // ODB++
getInteractiveBomFile() → File
getPcbInfoFile(fileName?) → File

// 自动布线/布局数据导出
getAutoRouteJson(fileName?) → File
getAutoLayoutJson(fileName?) → File

// 下单数据
getOrderComponents() → File
getOrderSMT() → File
getOrderPCB() → File
getOrder3DShellFile() → File
getManufactureData() → File                              // 全部制造数据打包

// BOM 模板
getBomTemplates() → Promise<string[]>
uploadBomTemplateFile(file, template?) → Promise<string | undefined>
getBomTemplateFile(template) → Promise<File | undefined>
deleteBomTemplate(template) → Promise<boolean>

// 下单快捷操作
placeComponentsOrder(interactive?, ignoreWarning?) → Promise<boolean>
placeSmtComponentsOrder(interactive?, ignoreWarning?) → Promise<boolean>
placePcbOrder(interactive?, ignoreWarning?) → Promise<boolean>
place3DShellOrder(interactive?, ignoreWarning?) → Promise<boolean>

// 自动布线规则
updateAutoRouteRule(rule) → Promise<boolean>
```

---

## 9. 文档树/工程管理 API

### 工程 DMT_Project
```
openProject(projectUuid) → Promise<boolean>
createProject(friendlyName, name?, teamUuid?, folderUuid?, desc?, collabMode?) → Promise<string | undefined>
moveProjectToFolder(projectUuid, folderUuid?) → Promise<boolean>
getAllProjectsUuid(teamUuid?, folderUuid?) → Promise<string[]>
getProjectInfo(projectUuid) → Promise<IDMT_BriefProjectItem | undefined>
getCurrentProjectInfo() → Promise<IDMT_ProjectItem | undefined>
getCurrentProjectUuid() → Promise<string>
```

### 板子 DMT_Board
```
createBoard(schematicUuid?, pcbUuid?) → Promise<string | undefined>
modifyBoardName / copyBoard / getAllBoardsInfo / getCurrentBoardInfo / getBoardInfo / deleteBoard
```

### 原理图文档 DMT_Schematic
```
createSchematic / createSchematicPage / modifySchematicName / modifySchematicPageName
modifySchematicPageTitleBlock / copySchematic / copySchematicPage
getSchematicInfo / getSchematicPageInfo / getAllSchematicsInfo / getAllSchematicPagesInfo
getCurrentSchematicInfo / getCurrentSchematicPageInfo / getCurrentSchematicAllSchematicPagesInfo
reorderSchematicPages / deleteSchematic / deleteSchematicPage
```

### PCB 文档 DMT_Pcb
```
createPcb / modifyPcbName / copyPcb / getAllPcbsInfo / getCurrentPcbInfo / getPcbInfo / deletePcb
```

### 面板 DMT_Panel
```
createPanel / modifyPanelName / copyPanel / getPanelInfo / getAllPanelsInfo / getCurrentPanelInfo / deletePanel
```

### 编辑器控制 DMT_EditorControl
```
openDocument(docUuid, splitScreenId?) → Promise<string | undefined>    // 返回 tabId
openLibraryDocument(libUuid, libType, uuid, splitScreenId?)
closeDocument(tabId) / activateDocument(tabId) / activateSplitScreen(splitScreenId)
getSplitScreenTree / getSplitScreenIdByTabId / getTabsBySplitScreenId
createSplitScreen(direction, tabId) / moveDocumentToSplitScreen
tileAllDocumentToSplitScreen / mergeAllDocumentFromSplitScreen
zoomTo(x?, y?, scale?, tabId?) / zoomToRegion / zoomToAllPrimitives / zoomToSelectedPrimitives
getCurrentRenderedAreaImage / generateIndicatorMarkers / removeIndicatorMarkers
```

### 文件夹/团队/工作区
```
DMT_Folder: createFolder / modifyFolderName / modifyFolderDescription / moveFolderToFolder / getAllFoldersUuid / getFolderInfo / deleteFolder
DMT_Team: getAllTeamsInfo / getAllInvolvedTeamInfo / getCurrentTeamInfo
DMT_Workspace: getAllWorkspacesInfo / toggleToWorkspace / getCurrentWorkspaceInfo
```

### 文档状态检查
```
DMT_SelectControl.getCurrentDocumentInfo() → Promise<IDMT_EditorDocumentItem | undefined>
// 返回: { documentType, uuid, tabId, parentProjectUuid?, parentLibraryUuid? }
```

---

## 10. 库搜索 API

```typescript
// 库列表
LIB_LibrariesList.getAllLibrariesList() → Promise<Array<ILIB_LibraryInfo>>
LIB_LibrariesList.getSystemLibraryUuid / getPersonalLibraryUuid / getProjectLibraryUuid / getFavoriteLibraryUuid

// 器件搜索
LIB_Device.search(key, libUuid?, classification?, symbolType?, page?, itemsOfPage?)
  → Promise<Array<ILIB_DeviceSearchItem>>
LIB_Device.getByLcscIds(ids: string|string[], libUuid?, allowMultiMatch?)
  → Promise<ILIB_DeviceSearchItem[]>
LIB_Device.get(deviceUuid, libUuid?) → Promise<ILIB_DeviceItem | undefined>

// 封装搜索
LIB_Footprint.search(key, libUuid?, classification?, page?, itemsOfPage?)
LIB_Footprint.get(uuid, libUuid?) → Promise<ILIB_FootprintItem | undefined>
LIB_Footprint.openInEditor(uuid, libUuid, splitScreenId?)

// 符号搜索
LIB_Symbol.search / get / openInEditor / updateDocumentSource / getRenderImage

// 3D 模型 / 复用模块 / 面板库 / 分类管理 (同模式 CRUD)
```

---

## 11. 系统工具 API

```typescript
// 消息/对话框
SYS_ToastMessage.showMessage(msg, type?, timer?, bottomPanel?, buttonTitle?, buttonCallback?)
// type: ERROR | WARNING | INFO | SUCCESS | ASK
SYS_Dialog.showConfirmationMessage(content, title?, mainBtn?, btn?, callback?)
SYS_Dialog.showInformationMessage(content, title?, btn?)
SYS_Dialog.showInputDialog(...)
SYS_Dialog.showSelectDialog(options, ...)

// 文件系统
SYS_FileSystem.getDocumentsPath() → string
SYS_FileSystem.getEdaPath() → string
SYS_FileSystem.getLibrariesPaths() → string[]
SYS_FileSystem.getProjectsPaths() → string[]
SYS_FileSystem.readFileFromFileSystem(uri) → File | undefined
SYS_FileSystem.saveFileToFileSystem(uri, data, fileName?, force?) → boolean
SYS_FileSystem.saveFile(data: File|Blob, fileName?) → void
SYS_FileSystem.listFilesOfFileSystem(path, recursive?) → ISYS_FileSystemFileList[]
SYS_FileSystem.deleteFileInFileSystem(uri, force?) → boolean
SYS_FileSystem.openReadFileDialog(extensions?, multi?) → File | File[] | undefined

// 文件管理器
SYS_FileManager.getProjectFile / getDocumentFile / getDocumentSource / setDocumentSource
SYS_FileManager.importProjectByProjectFile / extractProjectInfo / extractLibInfo

// 存储 (扩展间共享)
SYS_Storage.getExtensionUserConfig(key) → any
SYS_Storage.setExtensionUserConfig(key, value) → Promise<boolean>
SYS_Storage.deleteExtensionUserConfig(key) → Promise<boolean>

// 国际化
SYS_I18n.text(tag, namespace?, language?, ...args) → string
SYS_I18n.getCurrentLanguage() → Promise<string>

// 面板
SYS_PanelControl.openLeftPanel / closeLeftPanel / toggleLeftPanelLockState
SYS_PanelControl.openRightPanel / closeRightPanel
SYS_PanelControl.openBottomPanel / closeBottomPanel

// 快捷键
SYS_ShortcutKey.registerShortcutKey / unregisterShortcutKey / getShortcutKeys

// 单位转换
SYS_Unit.mmToMil / milToMm / inchToMil / milToInch / inchToMm / mmToInch

// 日志
SYS_Log.add / clear / export / find / sort

// 定时器
SYS_Timer.setTimeoutTimer / setIntervalTimer / clearTimeoutTimer / clearIntervalTimer

// 环境
SYS_Environment.getUserInfo / getEditorCurrentVersion / isClient / isOnlineMode / isOfflineMode

// WebSocket (扩展可用)
SYS_WebSocket.register / send / close

// HTTP 请求 (需 allowExternalInteractions 权限)
SYS_ClientUrl.request(url, method?, data?, options?, callback?)

// 窗口
SYS_Window.open / openUI / getCurrentTheme / addEventListener
```

---

## 12. 完整枚举值

### EPCB_LayerId
```
TOP=1, BOTTOM=2, TOP_SILK_SCREEN=3, BOTTOM_SILK_SCREEN=4,
TOP_PASTE_MASK=5, BOTTOM_PASTE_MASK=6, TOP_SOLDER_MASK=7, BOTTOM_SOLDER_MASK=8,
RATLINES=9, BOARD_OUTLINE=10, MULTI=11, DOCUMENT=12,
TOP_ASSEMBLY=13, BOTTOM_ASSEMBLY=14, MECHANICAL=15,
TOP_COMPONENTS=16, BOTTOM_COMPONENTS=17, SUBSTRATE=18, 3D_MODEL=19,
INNER_1=21..INNER_32=52, SUBSTRATE_1=53..SUBSTRATE_32=84,
TOP_STIFFENER=58, BOTTOM_STIFFENER=59, COMPONENT_SHAPE=99, LEAD_SHAPE=100
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
NO_COMPONENTS=2, NO_VIAS=3, NO_WIRES=5, NO_FILLS=6,
NO_POURS=7, NO_INNER_ELECTRICAL_LAYERS=8, FOLLOW_REGION_RULE=9
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
JLCEDA=1, ALTIUM_DESIGNER=2, PADS=3, KICAD=4
```

### EPCB_LayerType
```
SIGNAL, INTERNAL_ELECTRICAL, SILKSCREEN, SOLDER_MASK, PASTE_MASK, ASSEMBLY, OTHER, CUSTOM
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
BLANK=0, SCHEMATIC_PAGE=1, PCB=3, SYMBOL_COMPONENT=2,
FOOTPRINT=4, PCB_2D_PREVIEW=12, PCB_3D_PREVIEW=15,
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

## 13. 坐标系与单位

| 域 | 单位 | 1mm = | Y 轴方向 |
|----|------|-------|---------|
| **PCB** | 1 mil | 39.37 | 向下 |
| **原理图** | 0.01 inch (10 mil) | 3.937 | 向下 |

```
// 转换
eda.sys_Unit.mmToMil(10, 4) → 393.7008
eda.sys_Unit.milToMm(1000, 4) → 25.4
```

---

## 14. MCP Server 设计方案

### 推荐的 MCP Tools

```
# 工程
project_info → 获取当前工程信息
project_open(projectUuid) → 打开工程
project_create(name, teamUuid?, folderUuid?)

# PCB 文档
pcb_open(pcbUuid) → 打开 PCB 文档
pcb_save → 保存
pcb_import_changes(schematicUuid?) → 从原理图导入变更
pcb_zoom_to_board → 缩放到板框

# PCB 图层
pcb_set_layers(n: 2|4|6|8...) → 设置铜箔层数
pcb_get_all_layers → 获取所有图层
pcb_modify_layer(id, name?, type?, color?)

# PCB 图元创建 (坐标用 mm，MCP 端自动转换)
pcb_create_line(net, layer, x1mm, y1mm, x2mm, y2mm, width_mm?)
pcb_create_via(net, xmm, ymm, holeDiam_mm, diam_mm, type?)
pcb_create_pad(layer, xmm, ymm, padNumber, shape, width_mm, height_mm, hole?, net?)
pcb_create_component(lcsc, xmm, ymm, layer?, rotation?)
pcb_create_text(layer, xmm, ymm, text, fontSize_mm?)
pcb_create_pour(net, layer, polygon_points_mm, method?)
pcb_create_fill(layer, polygon_points_mm, net?)
pcb_create_region(layer, polygon_points_mm, ruleTypes[])
pcb_create_dimension(type, coordSet, layer?)
pcb_create_arc(net, layer, x1,y1, x2,y2, angle, width?)

# PCB 图元操作
pcb_get_selected → 获取选中图元
pcb_select(ids) → 选中图元
pcb_delete(ids) → 删除图元
pcb_modify(id, property) → 修改图元
pcb_get_all(net?, layer?, type?) → 获取所有图元

# DRC/规则
pcb_drc_check → 运行 DRC
pcb_create_netclass(name, nets, color?)
pcb_create_diffpair(name, posNet, negNet)
pcb_create_equallength_group(name, nets)

# 网络
pcb_get_all_nets → 获取所有网络
pcb_get_net_primitives(netName) → 获取网络关联图元
pcb_highlight_net(netName)

# 覆铜
pcb_rebuild_copper(pourId) → 重建指定覆铜
pcb_rebuild_all_copper → 重建所有覆铜 (getAll + forEach rebuildCopperRegion)

# 制造输出
pcb_export_gerber(fileName?) → 导出 Gerber
pcb_export_bom(fileName?, type?) → 导出 BOM
pcb_export_pick_and_place(fileName?) → 导出坐标
pcb_export_3d(fileName?) → 导出 3D

# 原理图
sch_create_component(lcsc, xmm, ymm, rotation?)
sch_create_wire(points_mm, net?)
sch_create_text(xmm, ymm, content, size?)
sch_export_netlist

# 库
lib_search_device(keyword, page?) → 搜索器件
lib_search_footprint(keyword) → 搜索封装
lib_get_by_lcsc(lcscId) → LCSC 编号反查

# 系统
sys_toast(msg, type?)
sys_dialog_confirm(msg)
```

### MCP Server 核心代码

```typescript
class EasyEDAMcpServer {
  private bridgePort: number;

  async execute(code: string, timeout = 30000): Promise<any> {
    const resp = await fetch(`http://localhost:${this.bridgePort}/execute`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code }),
      signal: AbortSignal.timeout(timeout),
    });
    return resp.json();
  }

  // 批量创建 (一次 HTTP 调用 = 一个 for 循环)
  async pcbCreateLines(lines: Array<{net,layer,x1,y1,x2,y2,width}>) {
    return this.execute(`
      const results = [];
      for (const l of ${JSON.stringify(lines)}) {
        results.push(await eda.pcb_PrimitiveLine.create(
          l.net, l.layer, l.x1, l.y1, l.x2, l.y2, l.width, false));
      }
      return results;
    `);
  }

  // 坐标转换 (MCP 端做)
  mmToMil(mm: number): number { return Math.round(mm * 39.3701); }
}
```

### 关键规则
1. 操作前必须验证文档状态：`getCurrentDocumentInfo()` → 确认 documentType
2. PCB 用 mil，MCP 对外用 mm，内部转换
3. 一次 `POST /execute` 内可包含循环/批量逻辑，节省 roundtrip
4. 跨调用无状态，不能保存变量
5. 必须 `return` 返回值

---

## 15. 缺口分析

### 已覆盖 ✅
基础图元 CRUD、图层管理、DRC 检查、制造输出、库搜索、工程管理、覆铜重建

### 可绕路 🟡
- **板框创建**：在 BOARD_OUTLINE 层画闭合走线
- **多边形创建**：用 `PCB_MathPolygon.createPolygon([x1,y1,...])`
- **交互式布线**：查询焊盘坐标后逐段 `createLine`

### 真正缺失 🔴
- **等长绕线 / 蛇形线**：无 `createTuningPattern()` API
- **泪滴**：无 Teardrop API（内部枚举存在但不公开）
- **过孔缝合/围栏**：无 ViaFence API
- **叠层详情**：只能改层数，不能按层设置 PP/Core 厚度
- **阻抗控制**：无 impedance profile API
- **拼板**：PNL_Document 只有 save()，其它操作无文档

### 全流程覆盖率：~70%

---

## 附录：文件索引

| 文件 | 说明 |
|------|------|
| `pro-api-types.d.ts` (19129 行) | 完整 TypeScript 类型定义 |
| `API完全提取-源码级.md` | 420 个 RPC 调用 |
| `easyeda-api-skill/` | 官方参考 (120 类文档/62 枚举/70 接口/Bridge Server) |
