# 嘉立创 EDA 深度逆向 #3 — 文档状态/事件/导航/UI框架

> 来源: pro-api-types.d.ts + PCB_Event/SCH_Event/SYS_IFrame/SYS_HeaderMenu 文档 + SKILL.md

---

## 1. 文档状态就绪 (openDocument/activateDocument 后何时可写)

### 1.1 核心流程

```javascript
// 1. 打开文档 → 返回 tabId
const tabId = await eda.dmt_EditorControl.openDocument(pageUuid);
// tabId: string | undefined

// 2. 激活标签页 → 让画布获得焦点
await eda.dmt_EditorControl.activateDocument(tabId);

// 3. ⚠️ 关键: 验证文档类型后才是真正可操作
const doc = await eda.dmt_SelectControl.getCurrentDocumentInfo();
// doc.documentType 必须匹配你要操作的 API 域

if (doc?.documentType !== EDMT_EditorDocumentType.SCHEMATIC_PAGE) {
  throw new Error("原理图未激活，无法操作");
}
// 此时原理图画布已就绪，可执行 sch_* 操作
```

### 1.2 就绪判断规则

```typescript
// 无专门 "canvasReady" 事件。用以下规则判断:
// ✅ openDocument() resolve 后 → tabId 有效
// ✅ activateDocument() resolve 后 → 该 tab 获得焦点
// ✅ getCurrentDocumentInfo() 返回匹配类型 → 画布可写

// ❌ 不需要额外等待 (无异步渲染延迟)
// ❌ 不需要 poll (documentType 不会延迟变化)
```

### 1.3 save() 后的状态

```javascript
await eda.pcb_Document.save(uuid);
// save() 是 async，resolve 即表示落盘完成
// 不需要额外等待
// sch_Document.save() 同理
```

### 1.4 importChanges 后的状态

```javascript
// importChanges 返回 true → PCB 已更新完成
const ok = await eda.pcb_Document.importChanges(schematicUuid);
if (ok) {
  // PCB 组件/网络已更新，无需额外等待
  const comps = await eda.pcb_PrimitiveComponent.getAll();
  // 此时可读取最新组件列表
}
// 如果需要在原理图看结果: 先 openDocument(schPageUuid) 再 activateDocument
```

---

## 2. 图元生命周期事件

### 2.1 PCB_Event — 鼠标/网络/图元/交叉选择

```typescript
// 鼠标事件 — 每次鼠标点击/移动
eda.pcb_Event.addMouseEventListener(
  id: string,                    // 唯一事件ID (防重复注册)
  eventType: 'all' | EPCB_MouseEventType,  // 或 ESCH_MouseEventType
  callFn: (
    eventType: EPCB_MouseEventType,
    props: [{
      primitiveId: string;
      primitiveType: EPCB_PrimitiveType;
      net?: string;
      designator?: string;
      parentComponentPrimitiveId?: string;
      parentComponentDesignator?: string;
    }]
  ) => void | Promise<void>,
  onlyOnce?: boolean
): void

// 图元事件 — create/modify/delete 时触发
eda.pcb_Event.addPrimitiveEventListener(
  id: string,
  eventType: 'all' | EPCB_PrimitiveEventType,  // CREATE/MODIFY/DELETE
  callFn: (
    eventType: EPCB_PrimitiveEventType,
    props: [{ primitiveId, primitiveType, net?, designator?, 
              parentComponentPrimitiveId?, parentComponentDesignator? }]
  ) => void | Promise<void>,
  onlyOnce?: boolean
): void

// 网络事件 — 选中网络时触发 (仅在过滤面板选网络 或 工程设计→网络内选中时)
eda.pcb_Event.addNetEventListener(
  id: string,
  eventType: 'all' | EPCB_NetEventType,
  callFn: (eventType, props: [{ net: string }]) => void | Promise<void>,
  onlyOnce?: boolean
): void

// 交叉选择 — 从原理图交叉选中PCB元件 / 反向
eda.pcb_Event.addCrossProbeSelectEventListener(
  id: string,
  callFn: (props: any) => void | Promise<void>
): void

// 工具方法
eda.pcb_Event.isEventListenerAlreadyExist(id): boolean
eda.pcb_Event.removeEventListener(id): boolean
```

### 2.2 SCH_Event — 原理图事件

```typescript
eda.sch_Event.addMouseEventListener(
  id, eventType: 'all' | ESCH_MouseEventType,
  callFn: (eventType: ESCH_MouseEventType) => void | Promise<void>,
  onlyOnce?
): void

eda.sch_Event.addPrimitiveEventListener(
  id, eventType: 'all' | ESCH_PrimitiveEventType,
  callFn: (eventType: ESCH_PrimitiveEventType, 
           props: { primitiveIds: Array<string> }) => void | Promise<void>,
  onlyOnce?
): void
// ⚠️ SCH 的 props 是 { primitiveIds: Array<string> }, 不是单对象数组

eda.sch_Event.addSimulationEnginePullEventListener(
  id, eventType: 'all',
  callFn: (eventType, props: { [key: string]: any }) => void | Promise<void>
): void

eda.sch_Event.isEventListenerAlreadyExist(id): boolean
eda.sch_Event.removeEventListener(id): boolean
```

### 2.3 事件生命周期管理

```javascript
// 注册
const eventId = "my-ext-mouse-watcher";
eda.pcb_Event.addPrimitiveEventListener(
  eventId, "all",
  async (type, props) => {
    for (const p of props) {
      console.log(`${type}: ${p.primitiveType} ${p.primitiveId}`);
    }
  }
);

// 检查是否已存在
if (eda.pcb_Event.isEventListenerAlreadyExist(eventId)) { ... }

// 移除
eda.pcb_Event.removeEventListener(eventId);

// ⚠️ 扩展卸载时不会自动清理 — 需在扩展 deactivate 时手动 removeEventListener
```

### 2.4 关键：没有文档切换事件

```typescript
// ❌ 没有 DMT_EditorControl 的 "onDocumentOpened"/"onTabChanged" 事件
// ❌ 没有全局 "onProjectOpened"/"onProjectClosed" 事件
// ❌ 没有 DRC 完成事件 (只能 await check())

// ✅ 图元级事件有 (PCB_Event / SCH_Event)
// ✅ SYS_Window 有主题切换事件
// ✅ 可以通过 extension.json 声明 onStartupFinished / onEditorPcb / onEditorSch 等激活钩子
```

---

## 3. 导航/定位 API — 完整参数与效果

### 3.1 缩放操作 (DMT_EditorControl)

```typescript
// 缩放到坐标 — 定位到指定坐标
eda.dmt_EditorControl.zoomTo(
  x?: number,        // 中心X (mil=PCB, 0.01inch=SCH)
  y?: number,        // 中心Y, 不传=保持当前
  scaleRatio?: number, // 缩放比 1/100, 200=200%, 不传=保持
  tabId?: string     // 不传=当前焦点画布
): Promise<{ left, right, top, bottom } | false>
// 返回缩放后的视口区域坐标; false=不支持或tabId不存在

// 缩放到区域 — 适配矩形框
eda.dmt_EditorControl.zoomToRegion(
  left: number, right: number, top: number, bottom: number,
  tabId?: string
): Promise<boolean>

// 缩放到所有图元 (适应全部)
eda.dmt_EditorControl.zoomToAllPrimitives(
  tabId?: string
): Promise<{ left, right, top, bottom } | false>

// 缩放到选中图元
eda.dmt_EditorControl.zoomToSelectedPrimitives(
  tabId?: string
): Promise<{ left, right, top, bottom } | false>
```

### 3.2 画布定位 (PCB_Document / SCH_Document)

```typescript
// PCB 定位到数据坐标 (单位: mil)
eda.pcb_Document.navigateToCoordinates(x: number, y: number): Promise<boolean>
// 提示: 先用 setCanvasOrigin(0,0) 保证画布坐标=数据坐标

// PCB 定位到区域 (单位: mil)
eda.pcb_Document.navigateToRegion(
  left: number, right: number, top: number, bottom: number
): Promise<boolean>

// PCB 缩放到板框
eda.pcb_Document.zoomToBoardOutline(): Promise<boolean>
```

### 3.3 选择/高亮 (PCB_SelectControl)

```typescript
// 选中图元 (传入 primitiveId 数组)
eda.pcb_SelectControl.doSelectPrimitives(
  ids: string | Array<string>
): Promise<boolean>

// 清除选中
eda.pcb_SelectControl.clearSelected(): Promise<boolean>

// 交叉选择 (跨 SCH/PCB 联动选中)
eda.pcb_SelectControl.doCrossProbeSelect(
  components?: Array<string>,  // 器件位号数组 ["R1","U1"]
  pins?: Array<string>,        // 引脚 ["R1:1","U1:VCC"]
  nets?: Array<string>,        // 网络名 ["GND","VCC"]
  highlight?: boolean,         // 是否高亮
  select?: boolean             // 是否选中
): Promise<boolean>

// 读取选中
eda.pcb_SelectControl.getAllSelectedPrimitives_PrimitiveId(): Promise<Array<string>>
eda.pcb_SelectControl.getAllSelectedPrimitives(): Promise<Array<IPCB_Primitive>>
eda.pcb_SelectControl.getCurrentMousePosition(): Promise<{x,y} | undefined>
```

### 3.4 定位违规 (完整模式)

```javascript
// 从 DRC/DFM 结果定位到违规位置
async function locateViolation(violation) {
  // 1. 选中违规图元
  await eda.pcb_SelectControl.doSelectPrimitives([violation.id]);
  // 2. 缩放到选中 (自动适配)
  const area = await eda.dmt_EditorControl.zoomToSelectedPrimitives();
  // 3. 高亮 (可选)
  if (violation.reason) {
    eda.sys_Message.showFollowMouseTip(violation.reason, 3000);
  }
}

// 从器件位号定位
async function locateComponent(designator) {
  const comps = await eda.pcb_PrimitiveComponent.getAll();
  const target = comps.find(c => c.getState_Designator() === designator);
  if (target) {
    await eda.pcb_SelectControl.doSelectPrimitives([target.getState_PrimitiveId()]);
    await eda.dmt_EditorControl.zoomToSelectedPrimitives();
  }
}
```

---

## 4. 扩展 UI 框架

### 4.1 IFrame 面板 (最常用)

```typescript
// 打开面板
eda.sys_IFrame.openIFrame(
  htmlFileName: string,    // 扩展包内路径, 如 "/iframe/panel.html"
  width?: number,          // 像素
  height?: number,         // 像素
  id?: string,             // 面板ID (用于后续操作)
  props?: {
    maximizeButton?: boolean;
    minimizeButton?: boolean;
    minimizeStyle?: 'collapsed' | 'constricted';
    buttonCallbackFn?: (button: 'close' | 'minimize' | 'maximize') => void | Promise<void>;
    onBeforeCloseCallFn?: () => boolean | undefined | Promise<boolean | undefined>;
    grayscaleMask?: boolean;   // 背景灰度遮罩
    title?: string;
  }
): Promise<boolean>

// 控制面板
eda.sys_IFrame.showIFrame(id?: string): Promise<boolean>
eda.sys_IFrame.hideIFrame(id?: string): Promise<boolean>
eda.sys_IFrame.closeIFrame(id?: string): Promise<boolean>
// id=undefined → 关闭本扩展打开的所有面板
```

### 4.2 IFrame ↔ 扩展主进程通信

**方法 A (推荐): Storage 桥接**
```javascript
// 扩展主进程 → iframe
await eda.sys_Storage.setExtensionUserConfig('myKey', JSON.stringify(data));

// iframe 内 → 读
const data = JSON.parse(await eda.sys_Storage.getExtensionUserConfig('myKey'));
```

**方法 B: IFrame 内直接访问 eda**
```javascript
// ⚠️ iframe 内 eda 是直接注入的, 不走 window.parent.eda
// iframe 内:
const project = await eda.dmt_Project.getCurrentProjectInfo();
// ✅ 直接可用, 不需要任何跨窗口通信!
```

### 4.3 顶部菜单

```typescript
// 在系统菜单下插入子菜单
eda.sys_HeaderMenu.insertSystemHeaderMenuItem(
  env: ESYS_HeaderMenuEnvironment,  // "SCH" | "PCB" | "SYMBOL" | "FOOTPRINT" | ...
  id: Array<string>,               // ["顶级菜单ID", "我的子菜单ID"]
  props: {
    title: string;                 // 菜单显示文本
    registerFn?: string;           // 对应 export 函数名 (同 extension.json headerMenus)
    menuItems?: Array<ISYS_HeaderMenuSub1MenuItem | ISYS_HeaderMenuSub2MenuItem | null>;
    insertDividerBefore?: boolean;
    insertDividerAfter?: boolean;
    insertBefore?: string;         // 插在指定ID之前
    crossDividerWhenInsert?: boolean;
  }
): Promise<string | undefined>
// 返回重写后的ID (含扩展UUID前缀)
// ⚠️ 系统菜单一旦新增, 重启EDA才能恢复

// 移除
eda.sys_HeaderMenu.removeSystemHeaderMenuItem(
  id: Array<string>,
  props?: { removeTheBeforeDivider?, removeTheAfterDivider? }
): Promise<boolean>
```

### 4.4 顶部菜单环境枚举

```typescript
enum ESYS_HeaderMenuEnvironment {
  HOME = "HOME",
  BLANK = "BLANK", 
  SCH = "SCH",           // 原理图
  SYMBOL = "SYMBOL",     // 符号
  PCB = "PCB",           // PCB
  FOOTPRINT = "FOOTPRINT", // 封装
  PCB_VIEW = "PCB_VIEW", // PCB预览
  PANEL = "PANEL",       // 面板
  PANEL_VIEW = "PANEL_VIEW" // 面板预览
}
```

### 4.5 extension.json headerMenus

```json
{
  "headerMenus": {
    "sch": [
      {
        "id": "my-tools",
        "title": "我的工具",
        "menuItems": [
          { "id": "do-thing", "title": "执行操作", "registerFn": "doThing" },
          { "id": "sep", "title": "-" },
          { "id": "open-panel", "title": "打开面板", "registerFn": "openPanel" }
        ]
      }
    ],
    "pcb": [
      {
        "id": "my-tools",
        "title": "我的工具",
        "menuItems": [
          { "id": "do-pcb-thing", "title": "PCB操作", "registerFn": "doPcbThing" }
        ]
      }
    ]
  }
}
```

### 4.6 任务进度面板 (最小可运行示例)

```javascript
// === 扩展主进程 (src/index.ts) ===

export function openProgressPanel() {
  eda.sys_IFrame.openIFrame('/iframe/progress.html', 400, 300, 'progress-panel', {
    title: 'AI 任务进度',
    minimizeButton: true,
    onBeforeCloseCallFn: () => { /* 清理 */ }
  });
}

// 发布进度到 iframe
async function updateProgress(step, total, message) {
  await eda.sys_Storage.setExtensionUserConfig('taskProgress', JSON.stringify({
    step, total, message, timestamp: Date.now()
  }));
}
```

```html
<!-- === iframe/progress.html === -->
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"><title>进度</title></head>
<body>
  <div id="status">等待任务...</div>
  <progress id="bar" value="0" max="100"></progress>
  <div id="log"></div>
  <script>
    // ⚠️ eda 直接可用!
    let lastTimestamp = 0;
    async function poll() {
      const raw = await eda.sys_Storage.getExtensionUserConfig('taskProgress');
      if (raw) {
        const data = JSON.parse(raw);
        if (data.timestamp > lastTimestamp) {
          lastTimestamp = data.timestamp;
          document.getElementById('status').textContent = data.message;
          document.getElementById('bar').value = (data.step / data.total) * 100;
          document.getElementById('log').innerHTML += 
            `<div>${data.message}</div>`;
        }
      }
    }
    setInterval(poll, 500);
  </script>
</body>
</html>
```

### 4.7 扩展激活钩子

```json
// extension.json 中声明:
{
  "onStartupFinished": true,    // EDA启动完成
  "onEditorPcb": true,          // PCB编辑器打开
  "onEditorSch": true,          // 原理图编辑器打开
  "onEditorFootprint": true,    // 封装编辑器
  "onEditorSymbol": true,       // 符号编辑器
  "onEditorPanel": true,        // 面板编辑器
  "onLogged": true,             // 用户登录
  "onChangeAllowExternalInteractions": "on|off"  // 外部交互权限变更
}
```

### 4.8 MessageBus (扩展间/iframe通信)

```typescript
// 公共频道 (跨扩展)
eda.sys_MessageBus.publishPublic(topic: string, message: any): void
eda.sys_MessageBus.subscribePublic(topic, callback): ISYS_MessageBusTask
eda.sys_MessageBus.pushPublic(topic, message): void
eda.sys_MessageBus.pullPublic(topic, callback): ISYS_MessageBusTask
eda.sys_MessageBus.rpcCallPublic(topic, message?, timeout?): Promise<any>
eda.sys_MessageBus.rpcServicePublic(topic, callback): void

// 私有频道 (仅本扩展内)
eda.sys_MessageBus.createPrivateMessageBus(): void
eda.sys_MessageBus.publish / subscribe / push / pull / rpcCall / rpcService
// (同 public 版本, 但仅本扩展可见)

// 一次性订阅
eda.sys_MessageBus.subscribeOnce(topic, callback): ISYS_MessageBusTask
```

---

## 5. 你的 bridge 新增 op 建议

```json
// 文档状态
{ "op": "document.open", "uuid": "...", "type": "schematic_page"|"pcb" }
// → { tabId, documentType, ready: true }

// 事件订阅 (bridge 端轮询或在扩展内注册)
{ "op": "events.subscribe", "eventTypes": ["primitive","mouse","net"] }
// bridge 在扩展内注册 PCB_Event/SCH_Event 回调, 通过 WebSocket 推给 AI

// 定位
{ "op": "locate.primitive", "id": "..." }              // 选中+缩放
{ "op": "locate.component", "designator": "U1" }        // 按位号定位
{ "op": "locate.violation", "id": "...", "reason": "..." } // DRC违规定位
{ "op": "locate.region", "left": 0, "right": 10000, "top": 0, "bottom": 8000 }

// UI
{ "op": "ui.openPanel", "name": "progress" }            // 打开进度面板
{ "op": "ui.updateProgress", "step": 3, "total": 10, "msg": "正在布线..." }
{ "op": "ui.toast", "msg": "操作完成", "type": "success" }
```
