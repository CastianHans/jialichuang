# 嘉立创专业版 (EasyEDA Pro) MCP 接口全量分析报告

> **版本**: V3.2.69 | **安装路径**: `E:\lceda-pro\` | **分析日期**: 2026-07-11

---

## 目录

1. [总体架构](#1-总体架构)
2. [接入方式选择](#2-接入方式选择)
3. [WebSocket Bridge 协议](#3-websocket-bridge-协议)
4. [eda 全局 API 总览](#4-eda-全局-api-总览)
5. [PCB 画板核心 API](#5-pcb-画板核心-api)
6. [原理图 API](#6-原理图-api)
7. [文档树/工程管理 API](#7-文档树工程管理-api)
8. [库管理 API](#8-库管理-api)
9. [系统工具 API](#9-系统工具-api)
10. [坐标系与单位](#10-坐标系与单位)
11. [异步图元修改模式](#11-异步图元修改模式)
12. [关键枚举速查](#12-关键枚举速查)
13. [文件格式 (JSONL)](#13-文件格式-jsonl)
14. [扩展系统](#14-扩展系统)
15. [MCP Server 设计方案](#15-mcp-server-设计方案)

---

## 1. 总体架构

```
┌──────────────────┐   HTTP POST    ┌──────────────────┐   WebSocket    ┌─────────────────┐
│   MCP Server     │ ─────────────► │  Bridge Server   │ ◄───────────► │  EasyEDA Pro    │
│   (你写的)        │ ◄───────────── │  (Node.js)       │               │  (Electron)     │
│                  │   JSON result  │  端口: 49620-629 │               │  E:\lceda-pro\  │
└──────────────────┘                └──────────────────┘               └─────────────────┘
```

### 关键路径

| 组件 | 路径 | 说明 |
|------|------|------|
| 主进程入口 | `E:\lceda-pro\resources\app\app.js` | 2.6MB Electron 主进程 |
| 扩展 API 定义 | `E:\lceda-pro\resources\app\assets\pro-api\0.1.197\api.js` | 1.5MB 全局 `eda.*` API |
| WebSocket 服务 | `E:\lceda-pro\resources\app\assets\pro-mgr\3.2.66\js\ws-service.js` | 扩展间通信 |
| PCB 引擎 | `E:\lceda-pro\resources\app\assets\pro-pcb\3.2.55\js\pcb.js` | 10.5MB PCB 核心 |
| 原理图引擎 | `E:\lceda-pro\resources\app\assets\pro-sch\3.2.68\js\sch.js` | 6.3MB 原理图核心 |
| UI 层 | `E:\lceda-pro\resources\app\assets\pro-ui\3.2.69\js\ui.js` | 17MB UI |
| Pangolin (3D) | `E:\lceda-pro\resources\app\assets\pangolin\0.2.32\` | 3D 渲染引擎 |
| Chameleon | `E:\lceda-pro\resources\app\assets\chameleon\3.12.17\` | 格式转换引擎 |
| Jerboa (自动布线) | `E:\lceda-pro\resources\app\assets\jerboa\2.0.2\` | Freerouting 集成 |
| OCCAPI | `E:\lceda-pro\resources\app\assets\occapi\1.3.16\` | OpenCascade WASM (24.8MB) |

### Electron IPC

Preload 脚本 (`assets/js/preload.js`) 通过 `contextBridge` 暴露:

```javascript
window.electronAPI = {
    openWindow: (url) => ipcRenderer.send('openWindow', url),
    openWindowSelf: (url) => ipcRenderer.send('openWindowSelf', url),
    focusWindow: () => ipcRenderer.send('focusWindow'),
    getWinId: () => ipcRenderer.sendSync('getWinId'),
}
```

---

## 2. 接入方式选择

有 **三种方式** 可以驱动嘉立创专业版:

### 方式 A: WebSocket Bridge (✅ 推荐)

**原理**: 启动一个本地 Node.js Bridge Server，通过 HTTP POST 把 JavaScript 代码发送到 EasyEDA 内执行。

**优点**:
- 官方 Skill 已实现，成熟稳定
- 不需要写扩展，直接发送 JS 代码
- 完整的 `eda.*` API 全都可以调用
- 多窗口支持

**缺点**:
- 需要在 EasyEDA 中安装 `run-api-gateway.eext` 扩展
- 需要启动 Bridge Server 进程
- 有 30s 超时限制

**架构**:
```
MCP Server → HTTP POST /execute → Bridge Server → WebSocket → EasyEDA Client
           ← JSON result         ←              ←             ←
```

### 方式 B: Extension 扩展开发

**原理**: 开发 `.eext` 扩展包，安装到 EasyEDA 内运行。

**优点**: 深度集成、菜单/UI 定制、事件监听
**缺点**: 开发周期长、需要打包部署、MCP 无法直接通信

### 方式 C: 文件格式直接操作

**原理**: 直接读写 `.epro` / `.epcb` / `.esch` (ZIP + JSONL) 文件。

**优点**: 不需要 EasyEDA 运行
**缺点**: 无官方文档支持、格式可能变化、无实时预览

### 结论: MCP 应该使用 **方式 A (WebSocket Bridge)**

---

## 3. WebSocket Bridge 协议

> ⚠️ **关键发现**: 嘉立创专业版**自身不包含本地 WebSocket Server**。它的 WebSocket 连接到远程服务器 `wss://modules.lceda.cn/api/ws/notify`。
> 官方的 `easyeda-api-skill` Bridge Server 是一个**独立的外部 Node.js 进程**，通过 `run-api-gateway.eext` 扩展让 EDA 客户端连接到它。

### 3.0 内部通信系统 (逆向结果)

在 Bridge 之下，EDA 内部有两层通信系统:

**WebSocket 层** (`ws-service.js`): 连接到远程服务器，用于在线协作。消息格式:
```json
{"topic": "/some/path", "fromConnectId": "uuid", "connectId": "uuid", "message": "payload"}
```

**MessageBus2 层** (进程内 Pub/Sub + RPC): 扩展 API 的实际通信通道。RPC 格式:
```javascript
// Request (push)
{ message: "payload", replyTopic: "_MSG_BUS_RPC_-topic-uuid-ticketNum" }
// Response (publish to replyTopic)  
{ success: true, result: value }  // or { success: false, err: "message" }
```

**扩展 API 的 RPC topic 前缀**: `"extensionApi."`，例如:
- `extensionApi.runStandaloneScript` — 执行独立脚本
- `extensionApi.userScript` — 运行/保存/删除用户脚本
- `extensionApi.callFunctionInExtension` — 调用扩展函数
- `extensionApi.getExtensionsIndex` — 获取扩展列表

**代码执行沙箱** (在 `api.js` 中):
扩展代码运行在严格受限的沙箱中——`document`, `fetch`, `WebSocket`, `localStorage`, `indexedDB` 等全部被设为 `undefined`。只有 `eda.*` API 和 `console` 可用。

**代码执行入口** (最直接的方式):
```javascript
// 通过 MessageBus2 发布到 userScript topic
_.extensionApiMessageBus2.publish("extensionApi.userScript", {
    operation: "run",
    userScript: "return await eda.dmt_Project.getCurrentProjectInfo();"
})
```

### 3.1 启动 Bridge Server

```bash
cd easyeda-api-skill && npm install && node scripts/bridge-server.mjs &
```

服务器自动在 **49620-49629** 端口范围内选择第一个可用端口。

### 3.2 健康检查

```bash
curl http://localhost:49620/health
# → {"service":"easyeda-bridge","edaConnected":true,"activeWindowId":"abc-123"}
```

### 3.3 列出连接的 EDA 窗口

```bash
curl http://localhost:49620/eda-windows
# → {"windows":[{"windowId":"abc-123","connected":true,"active":true}],"count":1}
```

### 3.4 选择活动窗口

```bash
curl -X POST http://localhost:49620/eda-windows/select \
  -H "Content-Type: application/json" \
  -d '{"windowId":"abc-123"}'
```

### 3.5 执行代码 (核心接口)

```bash
curl -X POST http://localhost:49620/execute \
  -H "Content-Type: application/json" \
  -d '{"code": "return await eda.dmt_Project.getCurrentProjectInfo();"}'
```

### 3.6 消息格式

| type | 方向 | 说明 |
|------|------|------|
| `handshake` | Server→Client | 握手, `service: "easyeda-bridge"` |
| `ping/pong` | 双向 | 心跳 |
| `execute` | AI→EDA | 执行 JS 代码 |
| `result` | EDA→AI | 返回结果 |
| `error` | EDA→AI | 返回错误 |

### 3.7 代码执行上下文

所有代码在 EasyEDA 的浏览器运行时中执行:

```javascript
async function(eda) {
  // 你的代码 — eda 是全局 API 对象
  // 必须用 return 返回结果
  // console.log 不会被捕获
  // 所有 Promise 必须 await
}
```

### 3.8 连接 EasyEDA 客户端

需在 EasyEDA Pro 中安装扩展: https://ext.lceda.cn/item/oshwhub/run-api-gateway

---

## 4. eda 全局 API 总览

全局变量 `eda` 是 `EDA` 类的实例，所有 API 通过 `eda.xxx` 调用。

### 4.1 完整模块清单 (77 个实例)

#### 文档树 (DMT_, 10 个)
| 实例 | 类 | 说明 |
|------|-----|------|
| `eda.dmt_Board` | DMT_Board | 板子管理 |
| `eda.dmt_EditorControl` | DMT_EditorControl | 编辑器控制 |
| `eda.dmt_Folder` | DMT_Folder | 文件夹管理 |
| `eda.dmt_Panel` | DMT_Panel | 面板管理 |
| `eda.dmt_Pcb` | DMT_Pcb | PCB 文档管理 |
| `eda.dmt_Project` | DMT_Project | 工程管理 |
| `eda.dmt_Schematic` | DMT_Schematic | 原理图管理 |
| `eda.dmt_SelectControl` | DMT_SelectControl | 文档选择控制 |
| `eda.dmt_Team` | DMT_Team | 团队管理 |
| `eda.dmt_Workspace` | DMT_Workspace | 工作区管理 |

#### PCB & 封装 (PCB_, 19 个)
| 实例 | 类 | 说明 |
|------|-----|------|
| `eda.pcb_Document` | PCB_Document | PCB 文档操作 |
| `eda.pcb_Drc` | PCB_Drc | 设计规则检查 |
| `eda.pcb_Event` | PCB_Event | PCB 事件 |
| `eda.pcb_Layer` | PCB_Layer | 图层操作 |
| `eda.pcb_ManufactureData` | PCB_ManufactureData | 生产资料导出 |
| `eda.pcb_MathPolygon` | PCB_MathPolygon | 多边形数学工具 |
| `eda.pcb_Net` | PCB_Net | 网络管理 |
| `eda.pcb_Primitive` | PCB_Primitive | 图元基类 |
| `eda.pcb_PrimitiveArc` | PCB_PrimitiveArc | 圆弧线 |
| `eda.pcb_PrimitiveAttribute` | PCB_PrimitiveAttribute | 属性文本 |
| `eda.pcb_PrimitiveComponent` | PCB_PrimitiveComponent | PCB 器件 |
| `eda.pcb_PrimitiveDimension` | PCB_PrimitiveDimension | 尺寸标注 |
| `eda.pcb_PrimitiveFill` | PCB_PrimitiveFill | 填充区域 |
| `eda.pcb_PrimitiveImage` | PCB_PrimitiveImage | 图像 |
| `eda.pcb_PrimitiveLine` | PCB_PrimitiveLine | 走线/直线 |
| `eda.pcb_PrimitiveObject` | PCB_PrimitiveObject | 嵌入对象 |
| `eda.pcb_PrimitivePad` | PCB_PrimitivePad | 焊盘 |
| `eda.pcb_PrimitivePolyline` | PCB_PrimitivePolyline | 折线 |
| `eda.pcb_PrimitivePour` | PCB_PrimitivePour | 覆铜边框 |
| `eda.pcb_PrimitivePoured` | PCB_PrimitivePoured | 覆铜填充 |
| `eda.pcb_PrimitiveRegion` | PCB_PrimitiveRegion | 禁止/约束区域 |
| `eda.pcb_PrimitiveString` | PCB_PrimitiveString | 文本 |
| `eda.pcb_PrimitiveVia` | PCB_PrimitiveVia | 过孔 |
| `eda.pcb_RayTracerEngine` | PCB_RayTracerEngine | 光线追踪 |
| `eda.pcb_SelectControl` | PCB_SelectControl | 选择控制 |

#### 原理图 (SCH_, 15 个)
| 实例 | 类 | 说明 |
|------|-----|------|
| `eda.sch_Document` | SCH_Document | 原理图文档操作 |
| `eda.sch_Drc` | SCH_Drc | 原理图 DRC |
| `eda.sch_Event` | SCH_Event | 原理图事件 |
| `eda.sch_ManufactureData` | SCH_ManufactureData | BOM/网表导出 |
| `eda.sch_Net` | SCH_Net | 原理图网络 |
| `eda.sch_Netlist` | SCH_Netlist | 网表 |
| `eda.sch_Primitive` | SCH_Primitive | 图元基类 |
| `eda.sch_PrimitiveArc` | SCH_PrimitiveArc | 圆弧 |
| `eda.sch_PrimitiveAttribute` | SCH_PrimitiveAttribute | 属性 |
| `eda.sch_PrimitiveBus` | SCH_PrimitiveBus | 总线 |
| `eda.sch_PrimitiveCircle` | SCH_PrimitiveCircle | 圆 |
| `eda.sch_PrimitiveComponent` | SCH_PrimitiveComponent | 器件 |
| `eda.sch_PrimitiveObject` | SCH_PrimitiveObject | 嵌入对象 |
| `eda.sch_PrimitivePin` | SCH_PrimitivePin | 引脚 |
| `eda.sch_PrimitivePolygon` | SCH_PrimitivePolygon | 多边形 |
| `eda.sch_PrimitiveRectangle` | SCH_PrimitiveRectangle | 矩形 |
| `eda.sch_PrimitiveText` | SCH_PrimitiveText | 文本 |
| `eda.sch_PrimitiveWire` | SCH_PrimitiveWire | 导线 |
| `eda.sch_SelectControl` | SCH_SelectControl | 选择控制 |
| `eda.sch_SimulationEngine` | SCH_SimulationEngine | 仿真引擎 |
| `eda.sch_Utils` | SCH_Utils | 工具类 |

#### 综合库 (LIB_, 9 个)
| 实例 | 类 | 说明 |
|------|-----|------|
| `eda.lib_3DModel` | LIB_3DModel | 3D 模型 |
| `eda.lib_Cbb` | LIB_Cbb | 复用模块 |
| `eda.lib_Classification` | LIB_Classification | 分类索引 |
| `eda.lib_Device` | LIB_Device | 器件 |
| `eda.lib_Footprint` | LIB_Footprint | 封装 |
| `eda.lib_LibrariesList` | LIB_LibrariesList | 库列表 |
| `eda.lib_PanelLibrary` | LIB_PanelLibrary | 面板库 |
| `eda.lib_SelectControl` | LIB_SelectControl | 库选择控制 |
| `eda.lib_Symbol` | LIB_Symbol | 符号 |

#### 系统 (SYS_, 22 个)
| 实例 | 类 | 说明 |
|------|-----|------|
| `eda.sys_ClientUrl` | SYS_ClientUrl | HTTP 请求 |
| `eda.sys_Dialog` | SYS_Dialog | 对话框 |
| `eda.sys_Environment` | SYS_Environment | 运行环境 |
| `eda.sys_FileManager` | SYS_FileManager | 文件管理 |
| `eda.sys_FileSystem` | SYS_FileSystem | 文件系统操作 |
| `eda.sys_FontManager` | SYS_FontManager | 字体管理 |
| `eda.sys_FormatConversion` | SYS_FormatConversion | 格式转换 |
| `eda.sys_HeaderMenu` | SYS_HeaderMenu | 顶部菜单 |
| `eda.sys_I18n` | SYS_I18n | 多语言 |
| `eda.sys_IFrame` | SYS_IFrame | 内嵌 iframe |
| `eda.sys_LoadingAndProgressBar` | SYS_LoadingAndProgressBar | 加载进度条 |
| `eda.sys_Log` | SYS_Log | 日志 |
| `eda.sys_Message` | SYS_Message | 消息通知 |
| `eda.sys_MessageBox` | SYS_MessageBox | 消息框 |
| `eda.sys_MessageBus` | SYS_MessageBus | 消息总线(扩展间通信) |
| `eda.sys_PanelControl` | SYS_PanelControl | 面板控制 |
| `eda.sys_RightClickMenu` | SYS_RightClickMenu | 右键菜单 |
| `eda.sys_Setting` | SYS_Setting | 设置 |
| `eda.sys_ShortcutKey` | SYS_ShortcutKey | 快捷键 |
| `eda.sys_Storage` | SYS_Storage | 键值存储 |
| `eda.sys_Timer` | SYS_Timer | 定时器 |
| `eda.sys_ToastMessage` | SYS_ToastMessage | Toast 通知 |
| `eda.sys_Tool` | SYS_Tool | 工具函数 |
| `eda.sys_Unit` | SYS_Unit | 单位转换 |
| `eda.sys_WebSocket` | SYS_WebSocket | WebSocket |
| `eda.sys_Window` | SYS_Window | 窗口管理 |

---

## 5. PCB 画板核心 API

> ⚠️ **重要**: PCB 坐标单位是 **1mil** (1 单位 = 1mil = 0.001 inch = 0.0254 mm)
> 1mm ≈ 39.37 单位

### 5.1 PCB_Document — 文档级操作

```typescript
// 自动布线 (BETA)
await eda.pcb_Document.autoRouting(props)
// 清除布线
await eda.pcb_Document.clearRouting(type)
// 坐标转换 (画布→数据)
eda.pcb_Document.convertCanvasOriginToDataOrigin(x, y)
// 坐标转换 (数据→画布)
eda.pcb_Document.convertDataOriginToCanvasOrigin(x, y)
// 获取飞线计算状态
await eda.pcb_Document.getCalculatingRatlineStatus()
// 获取画布原点偏移
eda.pcb_Document.getCanvasOrigin()
// 获取画布过滤器配置 (BETA)
eda.pcb_Document.getCurrentFilterConfiguration()
// 获取坐标点的图元 (BETA)
await eda.pcb_Document.getPrimitiveAtPoint(x, y)
// 获取区域内所有图元 (BETA)
await eda.pcb_Document.getPrimitivesInRegion(left, right, top, bottom, leftToRight)
// 导入自动布局 JSON (BETA)
await eda.pcb_Document.importAutoLayoutJsonFile(autoLayoutFile)
// 导入自动布线 JSON (BETA)
await eda.pcb_Document.importAutoRouteJsonFile(autoRouteFile)
// 导入自动布线 SES (BETA)
await eda.pcb_Document.importAutoRouteSesFile(autoRouteFile)
// 从原理图导入变更
await eda.pcb_Document.importChanges(uuid)
// 定位到坐标
await eda.pcb_Document.navigateToCoordinates(x, y)
// 定位到区域 (BETA)
await eda.pcb_Document.navigateToRegion(left, right, top, bottom)
// 保存文档
await eda.pcb_Document.save()
// 设置画布原点
eda.pcb_Document.setCanvasOrigin(offsetX, offsetY)
// 启动飞线计算
await eda.pcb_Document.startCalculatingRatline()
// 停止飞线计算
await eda.pcb_Document.stopCalculatingRatline()
// 缩放到板框 (BETA)
await eda.pcb_Document.zoomToBoardOutline()
```

### 5.2 PCB_PrimitiveLine — 创建走线

```typescript
// create 静态方法
await eda.pcb_PrimitiveLine.create(
    net: string,        // 网络名, 如 "GND", "VCC"
    layer: TPCB_LayersOfLine,  // 层枚举, 如 EPCB_LayerId.TOP
    startX: number,     // 起点X (mil)
    startY: number,     // 起点Y (mil)
    endX: number,       // 终点X (mil)
    endY: number,       // 终点Y (mil)
    lineWidth: number,  // 线宽 (mil)
    primitiveLock: boolean // 是否锁定
): Promise<IPCB_PrimitiveLine>

// get 获取已有走线
const line = await eda.pcb_PrimitiveLine.get([primitiveId])

// 修改走线 (异步模式)
const asyncLine = line.toAsync()
asyncLine.setState_StartX(newX)
asyncLine.setState_Layer(EPCB_LayerId.BOTTOM)
asyncLine.done()
```

### 5.3 PCB_PrimitivePad — 创建焊盘

```typescript
// create 静态方法
await eda.pcb_PrimitivePad.create(
    layer: TPCB_LayersOfPad, // 层
    x: number,               // X坐标 (mil)
    y: number,               // Y坐标 (mil)
    pad: TPCB_PrimitivePadShape, // 焊盘形状
    padNumber: string,       // 焊盘编号 "1"
    net?: string,            // 网络
    padType?: EPCB_PrimitivePadType, // 焊盘类型
    hole?: TPCB_PrimitivePadHole, // 钻孔
    rotation?: number,       // 旋转角度
    primitiveLock?: boolean
): Promise<IPCB_PrimitivePad>
```

### 5.4 PCB_PrimitiveVia — 创建过孔

```typescript
await eda.pcb_PrimitiveVia.create(
    net: string,
    x: number,
    y: number,
    diameter: number,       // 外径 (mil)
    holeDiameter: number,   // 内径 (mil)
    viaType?: EPCB_PrimitiveViaType, // 过孔类型
    primitiveLock?: boolean
): Promise<IPCB_PrimitiveVia>
```

### 5.5 PCB_PrimitiveArc — 创建圆弧走线

```typescript
await eda.pcb_PrimitiveArc.create(
    net: string,
    layer: TPCB_LayersOfLine,
    startX: number,
    startY: number,
    endX: number,
    endY: number,
    arcAngle: number,       // 圆弧角度
    lineWidth: number,
    interactiveMode?: EPCB_PrimitiveArcInteractiveMode,
    primitiveLock?: boolean
): Promise<IPCB_PrimitiveArc>
```

### 5.6 PCB_PrimitiveComponent — 放置器件

```typescript
await eda.pcb_PrimitiveComponent.create(
    component: { libraryUuid: string; uuid: string; name?: string },
    x: number,
    y: number,
    layer?: TPCB_LayersOfComponent,
    rotation?: number,
    mirror?: boolean,
    primitiveLock?: boolean
): Promise<IPCB_PrimitiveComponent>
```

### 5.7 PCB_PrimitivePour — 创建覆铜

```typescript
await eda.pcb_PrimitivePour.create(
    complexPolygon: IPCB_Polygon, // 覆铜多边形区域
    net: string,
    layer: TPCB_LayersOfCopper,
    lineWidth?: number,
    pourFillMethod?: EPCB_PrimitivePourFillMethod,
    pourName?: string,
    pourPriority?: number,
    preserveSilos?: boolean,
    primitiveLock?: boolean
): Promise<IPCB_PrimitivePour>
```

### 5.8 PCB_PrimitiveFill — 创建填充

```typescript
await eda.pcb_PrimitiveFill.create(
    complexPolygon: IPCB_Polygon,
    layer: TPCB_LayersOfFill,
    net?: string,
    lineWidth?: number,
    fillMode?: EPCB_PrimitiveFillMode,
    primitiveLock?: boolean
): Promise<IPCB_PrimitiveFill>
```

### 5.9 PCB_PrimitiveRegion — 创建禁止/约束区域

```typescript
await eda.pcb_PrimitiveRegion.create(
    complexPolygon: IPCB_Polygon,
    layer: TPCB_LayersOfRegion,
    ruleType?: Array<EPCB_PrimitiveRegionRuleType>,
    regionName?: string,
    lineWidth?: number,
    primitiveLock?: boolean
): Promise<IPCB_PrimitiveRegion>
```

### 5.10 PCB_PrimitiveString — 创建文本

```typescript
await eda.pcb_PrimitiveString.create(
    layer: TPCB_LayersOfImage,
    x: number,
    y: number,
    text: string,
    fontFamily?: string,
    fontSize?: number,
    lineWidth?: number,
    alignMode?: EPCB_PrimitiveStringAlignMode,
    rotation?: number,
    reverse?: boolean,
    expansion?: number,
    mirror?: boolean,
    primitiveLock?: boolean
): Promise<IPCB_PrimitiveString>
```

### 5.11 PCB_PrimitiveDimension — 创建尺寸标注

```typescript
await eda.pcb_PrimitiveDimension.create(
    layer: TPCB_LayersOfDimension,
    coordinateSet: TPCB_PrimitiveDimensionCoordinateSet,
    dimensionType: EPCB_PrimitiveDimensionType,
    lineWidth?: number,
    unit?: ESYS_Unit,
    precision?: number,
    primitiveLock?: boolean
): Promise<IPCB_PrimitiveDimension>
```

### 5.12 PCB_PrimitivePolyline — 创建折线

```typescript
await eda.pcb_PrimitivePolyline.create(
    polygon: IPCB_Polygon,
    layer: TPCB_LayersOfLine,
    net: string,
    lineWidth: number,
    primitiveLock?: boolean
): Promise<IPCB_PrimitivePolyline>
```

### 5.13 PCB_SelectControl — 选择控制

```typescript
// 获取当前选中图元的 PrimitiveId 列表
eda.pcb_SelectControl.getAllSelectedPrimitives_PrimitiveId(): Array<string>

// 选中指定 ID 的图元
eda.pcb_SelectControl.selectPrimitives(primitiveIds: Array<string>)

// 取消所有选中
eda.pcb_SelectControl.unselectAllPrimitives()
```

### 5.14 PCB_Layer — 图层操作

```typescript
// 获取所有图层
await eda.pcb_Layer.getAllLayersInfo()

// 获取图层颜色配置
await eda.pcb_Layer.getLayerColorConfiguration(layerId)

// 设置图层颜色
await eda.pcb_Layer.setLayerColorConfiguration(layerId, config)

// 获取图层显示状态
await eda.pcb_Layer.getLayerDisplayStatus(layerId)

// 设置图层显示状态
await eda.pcb_Layer.setLayerDisplayStatus(layerId, status)
```

### 5.15 PCB_Net — 网络管理

```typescript
// 获取所有网络
await eda.pcb_Net.getAllNetsInfo()

// 获取网络信息
await eda.pcb_Net.getNetInfo(netName)

// 创建差分对
await eda.pcb_Net.createDifferentialPair(diffPairItem)

// 创建等长网络组
await eda.pcb_Net.createEqualLengthNetGroup(groupItem)

// 创建网络类
await eda.pcb_Net.createNetClass(netClassItem)
```

### 5.16 PCB_Drc — 设计规则检查

```typescript
// 运行 DRC
await eda.pcb_Drc.check(
    checkTrackWidth: boolean,
    checkClearance: boolean,
    checkVia: boolean
): Promise<boolean> // true=通过, false=有错误
```

### 5.17 PCB_ManufactureData — 制造文件导出

```typescript
// 导出 Gerber
await eda.pcb_ManufactureData.exportGerber(options)

// 导出 BOM
await eda.pcb_ManufactureData.exportBom(options)

// 导出坐标文件 (Pick and Place)
await eda.pcb_ManufactureData.exportCoordinate(options)

// 导出 PDF
await eda.pcb_ManufactureData.exportPdf(options)

// 导出 3D 模型
await eda.pcb_ManufactureData.export3DModel(options)
```

### 5.18 PCB_Primitive — 图元通用操作

```typescript
// 获取图元 BBox (BETA)
await eda.pcb_Primitive.getPrimitivesBBox(primitiveIds: Array<string>)

// 获取图元边框线 (BETA)
await eda.pcb_Primitive.getPrimitiveBoardLine(primitiveId, layers)

// 删除图元
await eda.pcb_Primitive.remove(primitiveIds: Array<string>)

// 复制图元
await eda.pcb_Primitive.copy(primitiveIds: Array<string>)
```

---

## 6. 原理图 API

> ⚠️ **重要**: 原理图坐标单位是 **0.01inch (10mil)**
> 1mm ≈ 3.937 单位

### 6.1 SCH_PrimitiveComponent — 放置器件

```typescript
await eda.sch_PrimitiveComponent.create(
    component: { libraryUuid: string; uuid: string },
    x: number,           // 0.01inch 单位
    y: number,
    subPartName: string,
    rotation: number,    // 度数
    mirror: boolean,
    addIntoBom: boolean,
    addIntoPcb: boolean
): Promise<ISCH_PrimitiveComponent>
```

### 6.2 SCH_PrimitiveWire — 画导线

```typescript
await eda.sch_PrimitiveWire.create(
    net: string,
    line: Array<number> | Array<Array<number>>, // 路径点数组
    lineWidth?: number,
    color?: string,
    lineType?: ESCH_PrimitiveLineType
): Promise<ISCH_PrimitiveWire>
```

### 6.3 其他原理图图元

```typescript
eda.sch_PrimitiveBus.create()      // 总线
eda.sch_PrimitiveCircle.create()   // 圆
eda.sch_PrimitiveArc.create()      // 圆弧
eda.sch_PrimitiveRectangle.create() // 矩形
eda.sch_PrimitivePolygon.create()  // 多边形
eda.sch_PrimitiveText.create()     // 文本
eda.sch_PrimitivePin.create()      // 引脚
```

---

## 7. 文档树/工程管理 API

### 7.1 工程操作完整流程

```javascript
// 1. 获取当前工程信息
const project = await eda.dmt_Project.getCurrentProjectInfo()

// 2. 获取所有团队
const teams = await eda.dmt_Team.getAllTeamsInfo()

// 3. 获取团队下所有工程 UUID
const projectUuids = await eda.dmt_Project.getAllProjectsUuid(teamUuid, folderUuid)

// 4. 创建新工程
const newProjectUuid = await eda.dmt_Project.createProject(
    "MyProject",           // friendlyName
    "my-project",          // name (可选)
    teamUuid,              // 团队 UUID (可选)
    folderUuid,            // 文件夹 UUID (可选)
    "Description",         // 描述 (可选)
    collaborationMode      // 协作模式 (可选)
)

// 5. 打开工程 (必须先打开才能操作文档!)
await eda.dmt_Project.openProject(projectUuid)

// 6. 获取工程详情
const info = await eda.dmt_Project.getProjectInfo(projectUuid)
```

### 7.2 板子操作

```javascript
// 创建板子
await eda.dmt_Board.createBoard(schematicUuid?, pcbUuid?)

// 获取所有板子
await eda.dmt_Board.getAllBoardsInfo()

// 获取当前板子
await eda.dmt_Board.getCurrentBoardInfo()
```

### 7.3 文档打开/切换

```javascript
// 打开文档
const tabId = await eda.dmt_EditorControl.openDocument(documentUuid)

// 激活标签页
await eda.dmt_EditorControl.activateDocument(tabId)

// 关闭文档
await eda.dmt_EditorControl.closeDocument(tabId)

// 获取当前文档信息
const docInfo = await eda.dmt_SelectControl.getCurrentDocumentInfo()
// docInfo.documentType: EDMT_EditorDocumentType.PCB | SCHEMATIC_PAGE | ...

// 检查文档类型
if (docInfo?.documentType !== EDMT_EditorDocumentType.PCB) {
    // 需要先打开 PCB 文档!
}
```

---

## 8. 库管理 API

### 8.1 搜索器件

```javascript
// 搜索 STM32
const devices = await eda.lib_Device.search("STM32", libraryUuid?, classification?, itemsOfPage?, page?)

// 通过 LCSC 编号查找
const device = await eda.lib_Device.getByLcscIds("C123456")

// 获取器件详情
const deviceInfo = await eda.lib_Device.get(deviceUuid, libraryUuid?)
```

### 8.2 搜索封装

```javascript
const footprints = await eda.lib_Footprint.search("SOT-23", libraryUuid?)
const footprint = await eda.lib_Footprint.get(footprintUuid, libraryUuid?)
```

### 8.3 获取库列表

```javascript
const allLibs = await eda.lib_LibrariesList.getAllLibrariesList()
const personalLib = await eda.lib_LibrariesList.getPersonalLibraryUuid()
const systemLib = await eda.lib_LibrariesList.getSystemLibraryUuid()
```

---

## 9. 系统工具 API

```javascript
// Toast 通知
eda.sys_Message.showToastMessage("操作完成!")

// 确认对话框
const confirmed = await eda.sys_Dialog.showConfirmationMessage("确认删除?")

// 输入对话框
const input = await eda.sys_Dialog.showInputDialog("请输入值")

// 选择对话框
const selection = await eda.sys_Dialog.showSelectDialog(items)

// 存储键值 (扩展间共享)
await eda.sys_Storage.setExtensionUserConfig("myKey", JSON.stringify(data))
const data = JSON.parse(await eda.sys_Storage.getExtensionUserConfig("myKey"))

// 国际化
const text = eda.sys_I18n.text("Save")

// 文件系统操作
await eda.sys_FileSystem.saveFile(new File([content], "test.txt"))
const content = await eda.sys_FileSystem.readFileFromFileSystem(path)

// 单位转换
eda.sys_Unit.convert(value, fromUnit, toUnit)

// 定时器 (扩展内使用, 不使用 setTimeout)
const timer = eda.sys_Timer.setInterval(() => { ... }, 1000)
eda.sys_Timer.clearInterval(timer)
```

---

## 10. 坐标系与单位

### 关键区别 (最容易犯的错误!)

| 域 | 单位 | 换算 |
|----|------|------|
| **PCB** | 1 mil | 1mm ≈ 39.37 单位 |
| **原理图** | 0.01 inch = 10 mil | 1mm ≈ 3.937 单位 |

### 坐标系方向
- **Y 轴向下** (屏幕坐标系, 与数学坐标系相反)
- 旋转角度: 0° = 上, 逆时针增加

### PCB 坐标换算速查表

| 实际尺寸 | PCB 单位 (mil) | 原理图单位 (0.01inch) |
|----------|---------------|----------------------|
| 1 mm | 39.37 | 3.937 |
| 10 mm | 393.7 | 39.37 |
| 1 inch | 1000 | 100 |
| 1 cm | 393.7 | 39.37 |

---

## 11. 异步图元修改模式

修改已存在的 PCB/原理图图元时, **必须使用异步模式**:

```javascript
// 1. 获取图元
const prim = await eda.pcb_PrimitiveVia.get([viaId])

// 2. 转为异步模式
const asyncPrim = prim.toAsync()

// 3. 设置新值 (链式调用)
asyncPrim.setState_X(newX)
asyncPrim.setState_Y(newY)
asyncPrim.setState_Diameter(newDiameter)

// 4. 应用更改
await asyncPrim.done()

// 5. 取消更改 (可选)
// await asyncPrim.reset()
```

所有 `IPCB_Primitive*` 和 `ISCH_Primitive*` 接口都支持:
- `toAsync()` — 转为异步模式
- `toSync()` — 转回同步模式
- `done()` — 提交更改
- `reset()` — 撤销更改
- `isAsync()` — 检查是否异步模式
- `getState_*()` — 读取属性
- `setState_*()` — 设置属性

---

## 12. 关键枚举速查

### EPCB_LayerId (图层 ID)

| 枚举值 | 说明 |
|--------|------|
| `EPCB_LayerId.TOP` | 顶层铜 |
| `EPCB_LayerId.BOTTOM` | 底层铜 |
| `EPCB_LayerId.INNER1` ~ `INNER30` | 内层 1-30 |
| `EPCB_LayerId.TOP_SOLDER_MASK` | 顶层阻焊 |
| `EPCB_LayerId.BOTTOM_SOLDER_MASK` | 底层阻焊 |
| `EPCB_LayerId.TOP_PASTE_MASK` | 顶层钢网 |
| `EPCB_LayerId.BOTTOM_PASTE_MASK` | 底层钢网 |
| `EPCB_LayerId.TOP_SILK_SCREEN` | 顶层丝印 |
| `EPCB_LayerId.BOTTOM_SILK_SCREEN` | 底层丝印 |
| `EPCB_LayerId.BOARD_OUTLINE` | 板框 |
| `EPCB_LayerId.TOP_ASSEMBLY` | 顶层装配 |
| `EPCB_LayerId.BOTTOM_ASSEMBLY` | 底层装配 |
| `EPCB_LayerId.DRC_ERROR` | DRC 错误 |
| `EPCB_LayerId.MECHANICAL` | 机械层 |

### EPCB_PrimitiveType (图元类型)

| 枚举值 | 说明 |
|--------|------|
| `EPCB_PrimitiveType.TRACK` | 走线 |
| `EPCB_PrimitiveType.PAD` | 焊盘 |
| `EPCB_PrimitiveType.VIA` | 过孔 |
| `EPCB_PrimitiveType.ARC` | 圆弧 |
| `EPCB_PrimitiveType.COMPONENT` | 器件 |
| `EPCB_PrimitiveType.COMPONENT_PAD` | 器件焊盘 |
| `EPCB_PrimitiveType.FILL` | 填充 |
| `EPCB_PrimitiveType.POUR` | 覆铜边框 |
| `EPCB_PrimitiveType.POURED` | 覆铜填充 |
| `EPCB_PrimitiveType.REGION` | 区域 |
| `EPCB_PrimitiveType.TEXT` | 文本 |
| `EPCB_PrimitiveType.DIMENSION` | 尺寸标注 |
| `EPCB_PrimitiveType.POLYLINE` | 折线 |
| `EPCB_PrimitiveType.IMAGE` | 图像 |
| `EPCB_PrimitiveType.ATTRIBUTE` | 属性 |
| `EPCB_PrimitiveType.OBJECT` | 嵌入对象 |

### EPCB_PrimitivePadShapeType (焊盘形状)

| 枚举值 | 说明 |
|--------|------|
| `ELLIPSE` | 椭圆 |
| `RECT` | 矩形 |
| `OVAL` | 长圆形 |
| `POLYGON` | 多边形 |

### EPCB_PrimitiveViaType (过孔类型)

| 枚举值 | 说明 |
|--------|------|
| `THROUGH` | 通孔 |
| `BLIND` | 盲孔 |
| `BURIED` | 埋孔 |

### EDMT_EditorDocumentType (文档类型)

| 枚举值 | 说明 |
|--------|------|
| `PCB` | PCB 文档 |
| `SCHEMATIC_PAGE` | 原理图图页 |
| `FOOTPRINT` | 封装编辑 |
| `SYMBOL` | 符号编辑 |
| `PANEL` | 面板 |

---

## 13. 文件格式 (JSONL)

### 13.1 项目结构

`.epro` 文件本质是 ZIP 压缩包:

```
project.json          ← 工程清单 (标准 JSON)
device.json           ← 器件库清单
footprint.json        ← 封装库清单
symbol.json           ← 符号库清单
*.esch                ← 原理图图纸 (JSON Lines, 每行一个 JSON 数组)
*.epcb                ← PCB 图 (JSON Lines)
*.esym                ← 符号定义
*.efoo                ← 封装定义
*.eblob               ← 二进制数据 (图片等)
*.ecop                ← 覆铜数据
```

### 13.2 JSONL 格式

每行是一个独立的 JSON 数组, 第一个元素是类型:

```jsonl
["TRACK", {"gId": "gge6", "layerid": "1", "net": "GND", "x1": 0, "y1": 0, "x2": 1000, "y2": 0, "width": 10}]
["PAD", {"gId": "gge5", "shape": "ELLIPSE", "x": 382, "y": 208, "net": "VCC", "width": 6, "height": 6, "number": "1", "holeR": 1.8}]
["VIA", {"gId": "gge12", "x": 500, "y": 300, "net": "GND", "diameter": 24, "holeDiameter": 12}]
["TEXT", {"gId": "gge20", "layerid": "13", "text": "R1", "x": 100, "y": 200, "fontSize": 40}]
```

### 13.3 对象类型标识

| Key | 对象类型 |
|-----|---------|
| `TRACK` | 走线 |
| `PAD` | 焊盘 |
| `VIA` | 过孔 |
| `TEXT` | 文本 |
| `ARC` | 圆弧 |
| `RECT` | 矩形 |
| `CIRCLE` | 圆形 |
| `HOLE` | 安装孔 |
| `DIMENSION` | 尺寸标注 |
| `FOOTPRINT` | 封装 (嵌套) |
| `COPPERAREA` | 覆铜 |
| `SOLIDREGION` | 实心区域 |
| `DRCRULE` | DRC 规则 |

---

## 14. 扩展系统

### 14.1 扩展包格式

`.eext` 文件是 ZIP 包, 内含:
- `extension.json` — 扩展清单
- JS 入口文件 (esbuild 打包)
- 资源文件 (SVG, PNG, CSS 等)
- `.edaignore` — 忽略文件配置

### 14.2 extension.json 结构

```json
{
    "name": "my-extension",
    "displayName": "我的扩展",
    "version": "1.0.0",
    "engines": { "eda": "^2.3.0" },
    "entry": "./dist/index",
    "headerMenus": {
        "sch": [{ "id": "my-menu", "title": "My Menu" }],
        "pcb": [{ "id": "my-menu", "title": "My Menu" }]
    },
    "onStartupFinished": true
}
```

### 14.3 扩展激活事件

```javascript
// extension.json 中声明:
{
    "onStartupFinished": true,  // 启动完成
    "onEditorPcb": true,        // PCB 编辑器打开
    "onEditorSch": true,        // 原理图编辑器打开
    "onEditorFootprint": true,  // 封装编辑器打开
    "onEditorSymbol": true,     // 符号编辑器打开
    "onEditorPanel": true,      // 面板编辑器打开
    "onLogged": true,           // 登录完成
}
```

### 14.4 重要的内置扩展

嘉立创专业版本身由多个扩展组成:

| 扩展名 | 路径 | 大小 | 说明 |
|--------|------|------|------|
| pro-ui | `assets/pro-ui/3.2.69/` | 17MB JS | 主 UI |
| pro-pcb | `assets/pro-pcb/3.2.55/` | 10.5MB JS | PCB 编辑器 |
| pro-sch | `assets/pro-sch/3.2.68/` | 6.3MB JS | 原理图编辑器 |
| pro-mgr | `assets/pro-mgr/3.2.66/` | 2.2MB JS | 工程管理器 |
| pro-panel | `assets/pro-panel/3.2.52/` | | 面板编辑器 |
| pro-api | `assets/pro-api/0.1.197/` | 1.5MB JS | 扩展 API |
| jerboa | `assets/jerboa/2.0.2/` | 413KB JS | 自动布线 |
| pro-chat | `assets/pro-chat/0.1.22/` | 463KB | AI 聊天 |
| smt-ui | `assets/smt-ui/1.0.243/` | 10MB JS | SMT 贴片 |

---

## 15. MCP Server 设计方案

### 15.1 推荐架构

```
┌──────────────┐     stdio/HTTP     ┌─────────────────┐     HTTP POST     ┌──────────────────┐
│  Claude / AI │ ◄────────────────► │  MCP Server      │ ◄──────────────► │  Bridge Server   │
│              │   MCP Protocol     │  (你写的 Node.js) │                   │  (Node.js)       │
└──────────────┘                    └─────────────────┘                   └────────┬─────────┘
                                                                                   │ WebSocket
                                                                          ┌────────▼─────────┐
                                                                          │  EasyEDA Pro     │
                                                                          │  + Gateway ext   │
                                                                          └──────────────────┘
```

### 15.2 MCP Tools 设计建议

按功能域组织 MCP tools:

#### 工程工具
- `project_info` — 获取当前工程信息
- `project_open` — 打开工程
- `project_create` — 创建工程
- `project_list` — 列出所有工程

#### PCB 文档工具
- `pcb_document_open` — 打开 PCB 文档
- `pcb_document_save` — 保存
- `pcb_document_info` — 获取文档信息
- `pcb_zoom_to_board` — 缩放到板框
- `pcb_navigate_to` — 定位到坐标
- `pcb_import_changes_from_schematic` — 从原理图导入变更

#### PCB 图元创建工具
- `pcb_create_line` — 创建走线
- `pcb_create_pad` — 创建焊盘
- `pcb_create_via` — 创建过孔
- `pcb_create_arc` — 创建圆弧走线
- `pcb_create_component` — 放置器件
- `pcb_create_text` — 创建文本
- `pcb_create_fill` — 创建填充
- `pcb_create_pour` — 创建覆铜
- `pcb_create_region` — 创建禁止区域
- `pcb_create_dimension` — 创建尺寸标注
- `pcb_create_polyline` — 创建折线
- `pcb_create_board_outline` — 创建板框

#### PCB 图元修改工具
- `pcb_modify_primitive` — 通用修改 (异步模式)
- `pcb_delete_primitives` — 删除图元
- `pcb_copy_primitives` — 复制图元
- `pcb_select_primitives` — 选中图元
- `pcb_get_selected` — 获取选中

#### PCB 查询工具
- `pcb_get_primitives_in_region` — 区域查询
- `pcb_get_primitive_at_point` — 点查询
- `pcb_get_all_nets` — 获取所有网络
- `pcb_get_all_layers` — 获取所有图层

#### 自动布线工具
- `pcb_auto_route` — 自动布线 (BETA)
- `pcb_clear_routing` — 清除布线
- `pcb_calculate_ratlines` — 飞线计算

#### 制造输出工具
- `pcb_drc_check` — DRC 检查
- `pcb_export_gerber` — 导出 Gerber
- `pcb_export_bom` — 导出 BOM
- `pcb_export_pick_and_place` — 导出坐标文件

#### 原理图工具
- `sch_create_component` — 放置器件
- `sch_create_wire` — 画导线
- `sch_create_bus` — 画总线
- `sch_create_text` — 创建文本
- `sch_export_netlist` — 导出网表

#### 库工具
- `lib_search_devices` — 搜索器件
- `lib_search_footprints` — 搜索封装
- `lib_get_device` — 获取器件详情
- `lib_get_footprint` — 获取封装详情

#### 系统工具
- `sys_toast` — 显示通知
- `sys_dialog_confirm` — 确认对话框
- `sys_dialog_input` — 输入对话框
- `sys_get_unit` — 获取当前单位设置
- `sys_convert_units` — 单位转换

### 15.3 MCP Server 核心实现要点

```typescript
// MCP Server 伪代码
class EasyEDAMcpServer {
    private bridgePort: number;
    
    // 核心通信方法
    async execute(code: string, timeout = 30000): Promise<any> {
        const resp = await fetch(`http://localhost:${this.bridgePort}/execute`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ code }),
            signal: AbortSignal.timeout(timeout),
        });
        return resp.json();
    }
    
    // 构建代码模板
    buildCode(fn: string, ...args: any[]): string {
        // 序列化参数并注入到 eda 执行上下文
        return `return await (${fn})(eda, ${args.map(a => JSON.stringify(a)).join(',')});`;
    }
    
    // 工具实现示例
    async pcbCreateLine(net: string, layer: string, x1: number, y1: number, 
                         x2: number, y2: number, width: number) {
        return this.execute(`
            return await eda.pcb_PrimitiveLine.create(
                ${JSON.stringify(net)},
                ${layer},  // EPCB_LayerId.TOP
                ${x1}, ${y1}, ${x2}, ${y2}, ${width}, false
            );
        `);
    }
}
```

### 15.4 重要注意事项

1. **必须在有活动文档时操作** — 操作前检查 `eda.dmt_SelectControl.getCurrentDocumentInfo()`
2. **操作 PCB 前打开 PCB 文档** — 不能对原理图文档执行 PCB API
3. **单位转换** — PCB 用 mil, 原理图用 0.01inch, MCP 对外用 mm 并转换
4. **每次 `POST /execute` 是独立的 JS 上下文** — 不能跨调用保持变量
5. **必须 `return`** — `console.log` 不会返回到 MCP
6. **所有异步方法要 `await`** — 绝大部分 API 返回 Promise
7. **使用枚举值而非裸数字** — 查阅 `references/enums/` 目录

### 15.5 完整 API 文档资源

- 120 个类的详细方法: `easyeda-api-skill/references/classes/*.md`
- 62 个枚举: `easyeda-api-skill/references/enums/*.md`
- 70 个接口: `easyeda-api-skill/references/interfaces/*.md`
- 快速参考: `easyeda-api-skill/references/_quick-reference.md`
- 文件格式: `easyeda-api-skill/format/**/*.md`

---

## 17. 扩展 API 逆向核心发现

> 来源: 对 `api.js` (1.5MB) 的深度逆向。所有 API 调用最终通过 `_.extensionApiMessageBus2.rpcCall("extensionApi.XXX", params)` 实现。

### 17.1 内部通信架构

```
扩展代码调用 eda.pcb_PrimitiveLine.create(...)
    ↓
工厂类 Pi 调用内部 RPC
    ↓
_.extensionApiMessageBus2.rpcCall("extensionApi.PCB_PrimitiveLine.create", ...)
    ↓
渲染进程的 pro-pcb 模块处理请求
    ↓
返回结果给扩展
```

**关键**: 这意味着 MCP 理论上可以**绕过 Bridge**，直接通过浏览器的 MessageBus2 注入调用——但这需要代码运行在 EDA 进程内。Bridge 方式是最实际的选择。

### 17.2 工厂/Builder 模式 (所有图元通用)

每个图元类型有 **两套类**:

```
工厂类 (静态方法)           Builder 实例 (链式调用)
┌──────────────────┐       ┌──────────────────────┐
│ pcb_PrimitiveVia │       │ IPCB_PrimitiveVia     │
│  .create(...)    │──生成→│  .toAsync()           │
│  .delete(ids)    │       │  .setState_X(100)     │
│  .modify(inst,p) │       │  .setState_Diameter() │
│  .getAll(filter) │       │  .done()              │
│  .getByIds(ids)  │       │  .reset()             │
└──────────────────┘       └──────────────────────┘
```

### 17.3 完整枚举值 (实际数值)

**EPCB_LayerId** (60+ 层):
```
TOP=1, BOTTOM=2, TOP_SILKSCREEN=3, BOTTOM_SILKSCREEN=4,
TOP_SOLDER_MASK=5, BOTTOM_SOLDER_MASK=6, TOP_PASTE_MASK=7, BOTTOM_PASTE_MASK=8,
TOP_ASSEMBLY=9, BOTTOM_ASSEMBLY=10, BOARD_OUTLINE=11, MULTI=12, DOCUMENT=13,
MECHANICAL=14, INNER_1=15 ... INNER_32=46, RATLINE=57,
TOP_STIFFENER=58, BOTTOM_STIFFENER=59
```

**EPCB_PrimitiveType**:
```
ARC, COMPONENT, PAD, COMPONENT_PAD, POLYLINE, POUR, FILL, REGION,
LINE, VIA, DIMENSION, IMAGE, OBJECT, POURED, STRING, ATTRIBUTE
```

**EPCB_PrimitivePadShapeType**: `ELLIPSE, OBLONG(OVAL), RECTANGLE(RECT), REGULAR_POLYGON(NGON), POLYLINE_COMPLEX_POLYGON(POLYGON)`

**EPCB_PrimitivePadHoleType**: `ROUND, RECTANGLE(RECT), SLOT`

**EPCB_PrimitivePadType**: `NORMAL=0, TEST=1, MARK_POINT=2`

**EPCB_PrimitiveViaType**: `VIA=0, BLIND=1, SUTURE=2` (注意: 枚举值叫 SUTURE 不是 BURIED)

**EPCB_PrimitiveFillMode**: `SOLID=0, MESH=1, INNER_ELECTRICAL_LAYER=2`

**EPCB_PrimitiveRegionRuleType**: `NO_COMPONENTS=2, NO_VIAS=3, NO_WIRES=5, NO_FILLS=6, NO_POURS=7, NO_INNER_ELECTRICAL_LAYERS=8, FOLLOW_REGION_RULE=9`

**EPCB_PrimitiveStringAlignMode**: `LEFT_TOP=1, LEFT_MIDDLE=2, LEFT_BOTTOM=3, CENTER_TOP=4, CENTER=5, CENTER_BOTTOM=6, RIGHT_TOP=7, RIGHT_MIDDLE=8, RIGHT_BOTTOM=9`

**EPCB_PrimitivePourFillMethod**: `GRID45="45grid", GRID="90grid", SOLID="solid"`

**ESCH_PrimitiveLineType**: `SOLID=0, DASHED=1, DOTTED=2, DOT_DASHED=3`

**ESCH_PrimitivePinShape**: `NONE, INVERTED, CLOCK, INVERTED_CLOCK`

**ESCH_PrimitivePinType**: `IN, OUT, BI, PASSIVE, OPEN_COLLECTOR, OPEN_EMITTER, POWER, GROUND, HIZ, TERMINATOR, UNDEFINED`

**ESYS_Unit**: `MILLIMETER=mm, CENTIMETER=cm, METER=m, INCH=inch, MIL=mil`

**EDMT_EditorDocumentType**: `BLANK=0, SCHEMATIC_PAGE=1, SYMBOL_COMPONENT=2, PCB=3, PCB_2D_PREVIEW=12, PCB_3D_PREVIEW=15, PANEL=26, PANEL_3D_PREVIEW=27, FOOTPRINT_EDITOR=...`

**ESYS_ToastMessageType**: `ERROR, WARNING, INFO, SUCCESS, ASK`

### 17.4 单位转换工具 (SYS_Unit)

```javascript
eda.sys_Unit.getFrontendDataUnit()  // 用户选择的显示单位 (mm/cm/inch/mil)
eda.sys_Unit.getSystemDataUnit()    // 始终返回 "mil"

// 转换函数 (全静态方法)
eda.sys_Unit.milToMm(value, precision=4)    // mil × 0.0254
eda.sys_Unit.mmToMil(value, precision=4)    // mm / 0.0254
eda.sys_Unit.inchToMil(value, precision=4)  // inch × 1000
eda.sys_Unit.milToInch(value, precision=4)  // mil / 1000
```

### 17.5 消息总线系统 (MessageBus)

所有扩展 API 通信基于 MessageBus2 的 RPC:

```javascript
// 发布/订阅
_.extensionApiMessageBus2.publish("topic", message)
_.extensionApiMessageBus2.subscribe("topic", callback)

// RPC 调用
_.extensionApiMessageBus2.rpcCall("extensionApi.XXX", params, timeout?)

// 广播 (跨扩展)
_.extensionApiBroadcastChannelMessageBus (BroadcastChannel "extensionApi")

// 扩展间通信
_.extensionSYS_MessageBus.public.messageBus (MessageBus2)
_.extensionSYS_MessageBus.public.broadcastChannelMessageBus (BroadcastChannel "extensionApiExtensionsPublic")
```

### 17.6 扩展沙箱限制

扩展代码运行在严格受限的环境中，以下全部为 `undefined`:
```
document, fetch, WebSocket, XMLHttpRequest
localStorage, sessionStorage, indexedDB
window, self, location, navigator
postMessage, MessageChannel
FileSystem*, IDB*, Storage*
showOpenFilePicker, showSaveFilePicker, showDirectoryPicker
Headers, Request, Response
```

**可用的**: `console`, `eda.*` 全部 API, `Promise`, `Array`, `Object` 等标准 JS 对象

---

## 18. PCB/原理图引擎内部通信 (逆向结果)

> 来源: 对 `pcb-main.js` (2.1MB) 和 `sch-main.js` (6.1MB) 的内部 MessageBus 分析

### 18.1 内部命令分发机制

PCB 编辑器通过 MessageBus topic 分发命令:

```javascript
// PCB 主命令分发
We.messageBus.publish("Pcb/doCommand", commandName)

// 原理图菜单/操作分发
We.messageBus.publish("sch/" + command, null)
```

### 18.2 内部 RPC 服务 (渲染进程暴露)

```javascript
// PCB 相关
"copyData"                    // 复制到剪贴板
"pasteData"                   // 粘贴
"clearImageCopy"              // 清除图片复制
"pcb/getIframeUrl"            // 获取编辑器 iframe URL
"getPcbAnnotationInfo"        // 批注连接信息
"handle3DModelFile"           // 3D 模型导入
"newOrReplace3DModel"         // 3D 模型替换
"pcb/decodePcbWorldInWorker"  // PCB 数据解码

// 图层查询
"ui/getAllEnableLayer"        // 获取启用的图层列表
"hasUnableLayerObj"           // 检查禁用层图元
```

### 18.3 原理图命令列表

```javascript
// 编辑操作
"sch/copy", "sch/paste", "sch/cut"

// 网络管理
"openNetClassManager", "openDifferentialPairManager", "openEqualLengthGroupManager"
"selectNet", "selectConnection", "selectSegment"

// 工具
"showRuler", "showDrawingAttrDlg"
"setOriginByCenter", "setOriginByFirstPin"
"DeleteSelectedTrack", "DeleteTrackNetAll"
```

### 18.4 文档序列化格式

内部文档数据使用 **管道分隔的 JSON 记录**:

```
recordJSON||optionalPayload|
```

每行一条记录，记录类型:
```
META, META_CREATE, META_MODIFY, META_SORT, META_Z_INDEX,
FONT, BLOB, INSTANCE_ATTR, DELETE_DOC, VARIANT_GROUPED,
GROUP_INDEX, GROUP_DATA, RULE_TEMPLATE, RULE, RULE_SELECTOR,
ELE_PLACEHOLDER, META_PLACEHOLDER
```

### 18.5 图元类型 (内部枚举，与扩展 API 枚举对照)

**PCB 内部图元类型** (命名空间 `kL`):
```
Board, SlotRegion, ProhibitedRegion, ConstraintRegion, Component,
Track, TestPoint, Pad, Via, SutureHole, Text, ColorfulImage,
Image, FPCStiffener, Dimension, CopperRegion, FillRegion,
Line, Teardrop, D3Shell, DesignRule
```

**PCB 元素类型** (命名空间 `XC`, 用于数据序列化):
```
META, CANVAS, LAYER, LAYER_PHYS, ACTIVE_LAYER, PARTITION, NET,
PRIMITIVE, GROUP, SILK_OPTS, PREFERENCE, VIA, PAD, LINE, ARC, OBJ,
EQLEN_GRP, POLY, FILL, LAYER_FILL, REGION, POUR, POURED, IMAGE,
TEARDROP, FPC_FILL, SHELL, CREASE, SHELLCUT, SHELL_ENTITY, BOSS,
STRING, ATTR, DIMENSION, COMPONENT, PAD_NET, RULE_TEMPLATE, RULE,
RULE_SELECTOR, PANELIZE
```

**原理图图元类型** (内部):
```
META, CANVAS, GROUP, ATTR, LINE, WIRE, BUS, TEXT, POLY, CIRCLE,
ARC, BUSENTRY, RECT, MASK_REGION, PIN, OBJ, COMPONENT, PART,
ELLIPSE, BEZIER, TABLE, ELE_PLACEHOLDER, NG_SETTING
```

### 18.6 图层 ID 映射 (完整 100 层)

```
 1: TopLayer            2: BottomLayer          3: TopSilkLayer
 4: BottomSilkLayer     5: TopPasteMaskLayer    6: BottomPasteMaskLayer
 7: TopSolderMaskLayer  8: BottomSolderMaskLayer 9: Ratlines
10: BoardOutLine        11: Multi-Layer         12: Document
13: TopAssembly         14: BottomAssembly      15: Mechanical
16: TopComponents       17: BottomComponents    18: Substrate
19: 3DModel             99: ComponentShapeLayer 100: LeadShapeLayer
21-52: Inner1..Inner32
53-84: Substrate1..Substrate32
```

### 18.7 线型和填充枚举 (内部补充值)

**线端点**: `BUTT(0), ROUND(1), SQUARE(2)`
**圆弧方向**: `CW(0), CCW(1)`
**文本类型**: `PRINT, COVER, PRINT_AND_COVER`
**铺铜类型**: `COMPONENT, VIA, TRACK, FILL, COPPER, PLANE`
**布线角度**: `FORTY_FIVE("L45"), NINETY("L90"), NINETY_ARC("A90")`
**热焊连接**: `NONE, GRID, OUTLETS`

---

## 19. 完整信息源索引

本报告整合了以下信息源:

| 来源 | 类型 | 内容 |
|------|------|------|
| `E:\lceda-pro\` 本地安装 | 逆向分析 | 目录结构、Electron 架构、内部通信 |
| `easyeda-api-skill/references/` | 结构化文档 | 120 类完整 API 签名 |
| `easyeda-api-skill/SKILL.md` | 官方指南 | Bridge 协议、代码模式 |
| `prodocs.easyeda.com` | 在线文档 | 类索引、概述 |
| `resources/app/app.js` | 逆向 | Electron IPC、app:// 协议 |
| `pro-mgr/ws-service.js` | 逆向 | WebSocket、MessageBus 实现 |
| `pro-api/api.js` | 逆向 | 扩展 API、沙箱、枚举值 |
| `pro-pcb/pcb-main.js` | 逆向 | PCB 内部命令、图元枚举 |
| `pro-sch/sch-main.js` | 逆向 | 原理图命令、元素类型 |
| `jerboa/` | 逆向 | 自动布线扩展模式 |

---

## 20. 全自动缺口分析 (Blocker Audit)

> 对比"人类 PCB 工程师能做的一切"vs"当前 API 能做的一切"，找出阻挠全自动的缺口。

### 20.1 ✅ 已覆盖（无阻断）

| 人类操作 | API 覆盖 | 状态 |
|----------|---------|------|
| 放器件 (PCB) | `pcb_PrimitiveComponent.create()` | ✅ 完整 |
| 画走线 | `pcb_PrimitiveLine.create()` | ✅ 完整 |
| 画圆弧走线 | `pcb_PrimitiveArc.create()` | ✅ 完整 |
| 放过孔 | `pcb_PrimitiveVia.create()` | ✅ 完整 |
| 放焊盘 | `pcb_PrimitivePad.create()` | ✅ 完整 |
| 铺铜 | `pcb_PrimitivePour.create()` | ✅ 完整 |
| 填充区域 | `pcb_PrimitiveFill.create()` | ✅ 完整 |
| 禁止/约束区域 | `pcb_PrimitiveRegion.create()` | ✅ 完整 |
| 放置文本 | `pcb_PrimitiveString.create()` | ✅ 完整 |
| 尺寸标注 | `pcb_PrimitiveDimension.create()` | ✅ 完整 |
| 折线 | `pcb_PrimitivePolyline.create()` | ✅ 完整 |
| 图像 | `pcb_PrimitiveImage.create()` | ✅ 完整 |
| 嵌入对象 | `pcb_PrimitiveObject.create()` | ✅ 完整 |
| 删除/Copy/修改图元 | `delete()`/`modify()`/`get()`/`getAll()` | ✅ 完整 |
| 选中/取消选中 | `pcb_SelectControl` | ✅ 完整 |
| 图层显示/隐藏/锁 | `pcb_Layer.setLayerVisible()` 等 | ✅ 完整 |
| 设置铜箔层数 (2-32层) | `pcb_Layer.setTheNumberOfCopperLayers()` | ✅ 完整 |
| 设置 PCB 类型 (普通/FPC) | `pcb_Layer.setPcbType()` | ✅ 完整 |
| 修改图层属性 | `pcb_Layer.modifyLayer()` | ✅ 完整 |
| 获取所有网络 | `pcb_Net.getAllNets()` / `getAllNetsName()` | ✅ 完整 |
| 按网络查询图元 | `pcb_Net.getAllPrimitivesByNet()` | ✅ 完整 |
| 高亮/选中网络 | `highlightNet()` / `selectNet()` | ✅ 完整 |
| 设置网络颜色 | `pcb_Net.setNetColor()` | ✅ 完整 |
| 获取网表 | `pcb_Net.getNetlist()` / `sch_Netlist.getNetlist()` | ✅ 完整 |
| 更新网表 | `pcb_Net.setNetlist()` / `sch_Netlist.setNetlist()` | ✅ 完整 |
| DRC 检查 | `pcb_Drc.check()` | ✅ 完整 |
| 创建差分对 | `pcb_Drc.createDifferentialPair()` | ✅ 完整 |
| 创建等长网络组 | `pcb_Drc.createEqualLengthNetGroup()` | ✅ 完整 |
| 创建网络类 | `pcb_Drc.createNetClass()` | ✅ 完整 |
| 创建焊盘对组 | `pcb_Drc.createPadPairGroup()` | ✅ 完整 |
| 读写 DRC 规则 | `getNetRules()`/`overwriteNetRules()`/`getRegionRules()`... | ⚠️ 接口存在但规则结构不透明 |
| 导出 Gerber | `pcb_ManufactureData.getGerberFile()` | ✅ 完整 |
| 导出 BOM | `pcb_ManufactureData.getBomFile()` | ✅ 完整 |
| 导出坐标文件 | `pcb_ManufactureData.getPickAndPlaceFile()` | ✅ 完整 |
| 导出 PDF/3D/DXF | `getPDFFile()`/`get3DFile()`/`getDxfFile()` | ✅ 完整 |
| 原理图放器件 | `sch_PrimitiveComponent.create()` | ✅ 完整 |
| 原理图画导线 | `sch_PrimitiveWire.create()` | ✅ 完整 |
| 原理图总线/文本/圆/矩形 | `sch_PrimitiveBus`/`Text`/`Circle`/`Rectangle` | ✅ 完整 |
| 原理图 DRC | `sch_Drc.check()` | ✅ 完整 |
| 原理图导出 BOM/网表 | `sch_ManufactureData.*` | ✅ 完整 |
| 库搜索器件/封装 | `lib_Device.search()` / `lib_Footprint.search()` | ✅ 完整 |
| LCSC 编号查器件 | `lib_Device.getByLcscIds()` | ✅ 完整 |
| 工程 CRUD | `dmt_Project.create/open/getInfo` | ✅ 完整 |
| 导航/缩放 | `navigateToCoordinates()`/`zoomToBoardOutline()` | ✅ 完整 |

### 20.2 ⚠️ 部分覆盖（需绕路或受限）

| 人类操作 | 缺口 | 严重性 | 绕路方案 |
|----------|------|--------|---------|
| **板框绘制** | 无专门的 `createBoardOutline()` API | 🔴 高 | 用 `pcb_PrimitiveLine.create()` 在 `EPCB_LayerId.BOARD_OUTLINE`(11) 层画闭合线框 |
| **多边形/复杂多边形创建** | `IPCB_Polygon` / `IPCB_ComplexPolygon` 构造函数不明确 | 🔴 高 | 需要先创建简单 polygon 对象再传入 pour/fill/region 的 create() |
| **覆铜重建** | 修改图元后覆铜不会自动重铺 | 🟡 中 | 需要手动触发 `pcb_PrimitivePoured` 的操作，或用 UI 命令 |
| **自动布线参数** | `pcb_Document.autoRouting(props)` 的 `props` 结构未文档化 | 🔴 高 | 只能用默认参数或逆向内部传参格式 |
| **自动布线 SES 导入** | `importAutoRouteSesFile()` 的 SES 格式无文档 | 🟡 中 | 可用 Jerboa JSON 格式替代 |
| **原理图→PCB 同步** | `pcb_Document.importChanges(uuid)` 流程复杂 | 🟡 中 | 需确保工程中有原理图且已关联 |
| **DRC 规则结构** | 所有规则 API 返回 `{[key: string]: any}` 不透明 | 🟡 中 | 先用 `getCurrentRuleConfiguration()` 获取再修改特定字段 |
| **原理图自动布线** | `sch_Document.autoRouting(data)` 参数未文档化 | 🟡 中 | 同上 |
| **原理图自动布局** | `sch_Document.autoLayout(params)` 参数未文档化 | 🟡 中 | 同上 |

### 20.3 ❌ 缺失 API（阻断全自动）

| 人类操作 | 缺口 | 严重性 | 影响 |
|----------|------|--------|------|
| **泪滴 (Teardrop)** | 内部枚举 `Teardrop` 存在但无公开 API | 🔴 高 | 无法程序化添加泪滴，高频 PCB 必备 |
| **过孔缝合/围栏 (Via Stitching/Fence)** | Via fence UI 存在但无扩展 API | 🔴 高 | 无法程序化缝合过孔，EMI 控制必备 |
| **蛇形线/等长绕线 (Length Tuning)** | 无 `createTuningPattern()` 或类似 API | 🔴 高 | 无法程序化做等长绕线，DDR/USB 等高速总线必备 |
| **叠层管理 (Stackup)** | `setTheNumberOfCopperLayers()` 只能改层数，不能改叠层结构 | 🔴 高 | 无法设置 PP/Core 厚度、介电常数等 |
| **阻抗控制** | 无 `setImpedanceProfile()` 或阻抗计算 API | 🔴 高 | 无法做阻抗控制设计（90Ω USB/100Ω 差分等） |
| **交互式布线** | 无 `startRouting()` / `addRoutingSegment()` / `finishRouting()` | 🔴 高 | 只能创建独立线段，不能做连续交互式布线 |
| **从焊盘/过孔开始布线** | 无法获取焊盘坐标并从其中心自动开始布线 | 🟡 中 | 需手动查询坐标再 createLine |
| **Undo/Redo** | 无扩展 API | 🟡 中 | 操作出错后无法程序化撤销 |
| **测量工具** | 无 `measureDistance()` 或 `getCoordinatesOf()` API | 🟡 中 | 需要自己用 BBox 计算 |
| **对齐/分布** | 无 `alignLeft()`/`distributeHorizontally()` 等 API | 🟡 中 | 需手动计算目标坐标 |
| **拼板 (Panelization)** | `PNL_Document` 类存在但文档极简 | 🔴 高 | 无法程序化做 V-cut/邮票孔拼板 |
| **元件重编号** | 无 `renumberDesignators()` API | 🟡 中 | 需手动逐个修改 |
| **丝印调整** | 无 `autoAdjustSilkscreen()` API | 🟡 低 | 丝印避让无法自动化 |
| **3D 模型对齐** | 无 `align3DModel()` API | 🟡 低 | 手动对齐 3D 模型后可用 |
| **批量操作** | 每次 `POST /execute` 是一个独立 JS 上下文 | 🟡 中 | 可在一次调用内写循环，但有 30s 超时 |
| **进度回调** | `POST /execute` 只返回最终结果，中间无进度 | 🟡 低 | 长时间操作（自动布线）无进度反馈 |

### 20.4 🔴 关键阻断项分析

#### A. 板框创建（可绕路）
```javascript
// 绕路方案: 在 BoardOutline 层画闭合线框
const OUTLINE = EPCB_LayerId.BOARD_OUTLINE; // = 11
// 四条边
await eda.pcb_PrimitiveLine.create("", OUTLINE, 0, 0, width, 0, 10, false);
await eda.pcb_PrimitiveLine.create("", OUTLINE, width, 0, width, height, 10, false);
await eda.pcb_PrimitiveLine.create("", OUTLINE, width, height, 0, height, 10, false);
await eda.pcb_PrimitiveLine.create("", OUTLINE, 0, height, 0, 0, 10, false);
// 或使用 pcb_PrimitiveArc 做圆角
```

#### B. 多边形创建（可绕路）
`IPCB_Polygon` 构造函数接受坐标数组:
```javascript
// 推测用法 (需验证)
const polygon = new IPCB_Polygon([x1,y1, x2,y2, x3,y3, x4,y4]);
// 或
const complexPolygon = new IPCB_ComplexPolygon();
complexPolygon.addSource([x1,y1, x2,y2, ...]);
```

#### C. 泪滴/过孔缝合/等长绕线（无法绕路）
这三个是**真正的阻断项**——扩展 API 完全没有暴露这些功能。可行的替代方案:
1. **直接操作 JSONL 文件** — 手动构造 Teardrop/SutureHole 图元数据写入 .epcb
2. **UI 自动化** — 用快捷键或菜单命令触发
3. **等官方更新** — 这些 API 可能在未来版本暴露

#### D. 自动布线参数（可逆向）
内部 `autoRouting()` 的 props 结构可以通过以下方式获取:
```javascript
// 在 EDA 内执行, 从 UI 抓取默认参数
return await eda.pcb_Document.autoRouting({});
// 观察返回值或报错信息来推断参数结构
```

### 20.5 总结: 当前自动化覆盖率

| 设计阶段 | 覆盖率 | 阻断项 |
|----------|--------|--------|
| 工程创建与管理 | **95%** | 无 |
| 原理图绘制 | **90%** | 自动布线参数不透明 |
| 库管理 (搜索/放置) | **95%** | 无 |
| PCB 布局 (放置/移动) | **90%** | 对齐/分布缺失 |
| PCB 手工布线 (走线/过孔) | **85%** | 无交互式布线 API |
| **多层板设计** | **75%** | 叠层结构不可编程 |
| **高速设计 (差分/等长)** | **60%** | 等长绕线 API 缺失 |
| **电源完整性 (铺铜/缝合)** | **55%** | 过孔缝合/泪滴 API 缺失 |
| DRC 与规则管理 | **70%** | 规则结构不透明 |
| 制造文件导出 | **95%** | 无 |
| 拼板 | **10%** | PNL_Document API 几乎无文档 |
| **全流程自动化 (SCH→PCB→Mfg)** | **~70%** | 高速/电源/拼板 是主要缺口 |

### 20.6 Token 效率优化策略

针对 MCP 调用费 token 的问题:

1. **单次批量执行**: 一次 `POST /execute` 发送包含循环的大段代码
   ```javascript
   // 一次调用创建 100 个焊盘
   const pads = [];
   for (let r = 0; r < 10; r++) {
     for (let c = 0; c < 10; c++) {
       pads.push(await eda.pcb_PrimitivePad.create(...));
     }
   }
   return pads.length;
   ```

2. **MCP 端预计算坐标**: AI 说"放一个 10x10 的 QFN 在中心"，MCP Server 计算所有坐标再发往 EDA

3. **复用模板函数**: 在 MCP 端维护常用操作模板
   ```javascript
   const TEMPLATES = {
     placeResistor0805: `await eda.pcb_PrimitiveComponent.create({libraryUuid: libUuid, uuid: footprintUuid}, x, y, EPCB_LayerId.TOP, rot, false, true)`,
     pourGND: `await eda.pcb_PrimitivePour.create("GND", EPCB_LayerId.TOP, polygon, "solid", false, "GND_POUR", 0, 10, false)`,
   };
   ```

4. **缓存库搜索**: MCP 端缓存 LCSC 编号→器件 UUID 映射，避免重复搜索

---

## 16. Electron 主进程分析 (app.js 逆向结果)

> 来源: 对 `E:\lceda-pro\resources\app\app.js` (2.6MB webpack bundle) 的逆向分析

### 16.1 关键发现

| 发现 | 影响 |
|------|------|
| **扩展系统不在主进程** | 扩展的加载、管理、执行全部在渲染进程的 pro-mgr 模块中, 不在 app.js |
| **没有内置 WebSocket Server** | WebSocket 通信在渲染进程的 ws-service.js 中实现 |
| **主窗口必须 sandboxed** | `contextIsolation: true`, `sandbox: true`, 只暴露 4 个 preload API |

### 16.2 Electron IPC 通道

```javascript
// 窗口控制
ipcMain.on("control", (event, cmd) => ...)     // "_min", "_max", "_restore", "_close", "_settings"
ipcMain.on("openWindow", (event, url) => ...)   // 外部浏览器打开 URL
ipcMain.on("openWindowSelf", (event, url) => ...) // 新建编辑器窗口
ipcMain.on("focusWindow", () => ...)           // 聚焦窗口
ipcMain.on("getWinId", (event) => ...)          // 同步返回窗口 ID

// 客户端设置
ipcMain.on("client-setting", (event, {cmd, data}) => ...)
// cmd: "Confirm", "Cancel", "libAddOne", "projectPathAddOne",
//      "changeLibPath", "changeProjectPath", "libDot", "projectPathDot"

// 渲染进程接收
ipcRenderer.on("loadingMessage", (event, {message}) => ...)  // 启动加载进度
ipcRenderer.on("client-setting", (event, data) => ...)       // 路径更新
```

### 16.3 `app://` 协议 API (主进程内部接口)

这些是 Electron 主进程暴露的内部 API, 通过 `app://` 协议调用:

```javascript
// 文件操作
app://api/client/readFileSync        // fs.readFileSync
app://api/client/writeFileSync       // fs.writeFileSync (base64)
app://api/client/listFilesSync       // fs.readdirSync (递归)
app://api/client/deleteFileSync      // fs.rmSync / fs.rmdirSync

// 临时备份
app://api/client/tmpBackupList       // 列出备份
app://api/client/tmpBackup/create    // 创建备份 (.epro zip)
app://api/client/tmpBackup/delete    // 删除备份
app://api/client/tmpBackup           // 读取备份文件

// 系统操作
app://api/system/getCacheSize        // 缓存大小
app://api/system/clearCache          // 清除缓存 (后重启)
app://api/system/openURL             // 系统浏览器打开
app://api/system/openNewEditorWindow // 新编辑窗口
app://api/system/toggleLanguage      // 切换语言
app://api/system/syncSysCtxmenuArea  // 同步右键菜单区域

// 更新管理
app://api/client/changeLaunchType    // 切换 ONLINE/HALF_OFFLINE/OFFLINE
app://api/client/changeCheckUpdate   // 切换自动更新
app://api/client/update/download     // 下载更新包
app://api/client/update/progress     // 查询下载进度
app://api/client/update/install      // 安装更新
app://api/client/update/clear        // 清除更新缓存

// 广播消息 (跨窗口通信, SQLite 轮询)
app://api/broadcast/message/maxId    // 最大消息 ID
app://api/broadcast/message/fetch    // 拉取消息
app://api/broadcast/message/add      // 发送消息

// 原生对话框
app://api/client/openDir             // 文件夹选择
app://api/client/openFolder          // 系统资源管理器打开
app://api/fileDialog/saveFile        // 保存文件对话框

// 用户数据
app://api/client/uploadFile          // 上传文件
app://api/client/online/backup       // 在线备份
app://api/client/activationInfo      // 许可证信息
```

### 16.4 窗口架构

```
┌─────────────────────────────────────────────────────┐
│  主编辑器窗口 (sandboxed)                              │
│  - contextIsolation: true                            │
│  - sandbox: true                                    │
│  - preload: preload.js (仅暴露 4 个 API)              │
│  - 加载自 CDN: https://modules.lceda.cn/pro-mgr/...  │
│  - 内嵌 BrowserView 用于设置对话框                     │
├─────────────────────────────────────────────────────┤
│  启动窗口 (unsandboxed)                                │
│  - nodeIntegration: true                             │
│  - 加载本地 HTML: assets/view/index.html              │
│  - 显示激活状态和加载进度                              │
└─────────────────────────────────────────────────────┘
```

### 16.5 启动流程

```
1. app.requestSingleInstanceLock()
2. 解析命令行参数 (--edaDebug, .eprj/.eprj2 文件)
3. 注册 app://, ftp:// 协议
4. 启动本地 HTTP 服务器 (sC)
5. 拦截 HTTP/HTTPS 请求 (protocol.interceptBufferProtocol)
   - "client" 域名 → 本地服务器
   - "modules.lceda.cn" → 条件本地
   - 其他 → 远程在线服务
6. 创建启动窗口 (k5)
7. 启动完成后创建主编编辑器窗口 (x3)
8. 从启动窗口过渡到主窗口 (eh)
```

### 16.6 对 MCP 开发的意义

- `app://` 协议 API 可用于 MCP 做**文件级操作** (直接读写 .epro/.epcb 文件)
- 扩展管理在渲染进程, MCP 通过 WebSocket Bridge 直接注入代码即可
- 广播消息系统 (SQLite 轮询) 可用于跨窗口状态同步
- 主窗口的 sandbox 限制不影响 Bridge 方式 (Bridge 代码运行在渲染进程上下文中)

---

## 附录: 本地安装文件清单

```
E:\lceda-pro\
├── lceda-pro.exe              (202MB)  Electron 主程序
├── resources\
│   ├── app.js                 (2.6MB)  Electron 主进程
│   ├── package.json                    name: "client.pro.lceda.cn"
│   └── assets\
│       ├── pro-api\0.1.197\api.js  (1.5MB)  扩展 API 定义 ⭐
│       ├── pro-pcb\3.2.55\pcb.js   (10.5MB) PCB 核心引擎
│       ├── pro-pcb\pcb-main.js      (2.1MB)  PCB 主逻辑
│       ├── pro-sch\3.2.68\sch.js    (6.3MB)  原理图引擎
│       ├── pro-ui\3.2.69\ui.js      (17MB)   UI 层
│       ├── pro-mgr\3.2.66\          (2.2MB)  工程管理器
│       ├── pro-mgr\ws-service.js    (850KB)  WebSocket 服务
│       ├── jerboa\2.0.2\            (413KB)  自动布线
│       ├── pangolin\0.2.32\         (2.6MB)  3D 渲染
│       ├── chameleon\3.12.17\       (4.3MB)  格式转换
│       ├── occapi\1.3.16\occapi.wasm (24.8MB) OpenCascade
│       ├── smt-gl-engine\3.12.12\   (2.4MB)  SMT 引擎
│       ├── smt-ui\1.0.243\          (10MB)   SMT UI
│       ├── ocr-wizard\0.1.67\       (26MB)   OCR 助手
│       ├── pro-panel\3.2.52\                面板编辑器
│       ├── pro-chat\0.1.22\        (463KB)  AI 聊天
│       ├── pro-sw\0.1.9\                    Service Worker
│       ├── view\                            HTML 入口
│       │   ├── index.html                   启动加载页
│       │   ├── control.html                 控制框架
│       │   └── dialog.html                  弹窗框架
│       ├── js\
│       │   ├── preload.js          (716B)   contextBridge
│       │   ├── action.js           (4KB)    启动页逻辑
│       │   └── translate.js        (3.5KB)  翻译
│       ├── locale\                          多语言文件
│       │   ├── zh-cn.json, en.json
│       │   └── app-menu-cn.json, app-menu-en.json
│       └── db\
│           ├── lceda-std.elib     (462MB)  标准库
│           └── example-projects.zip (49MB) 示例工程
├── jialichuangzhuanyeban\          (旧版嘉立创专业版, 类似结构)
└── locales\                        Electron 语言文件
```
