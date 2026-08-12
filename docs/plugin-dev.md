# 插件开发指南

> 本文档描述如何为易游插件管理器开发插件。涵盖 manifest.json 规范、index.js 规范、PluginContext API、前端 API、菜单/快捷键配置，以及完整示例。

---

## 1. 插件目录结构

```
plugins/{pluginId}/
├── manifest.json        ← 必填。元信息、UI 页面、菜单、快捷键
├── index.js             ← 必填。生命周期、命令处理器
├── README.md            ← 可选。插件文档（上传到商店后会展示在详情页）
└── ui/
    └── index.html       ← 可选。内嵌 UI 页面（如果不用独立前端项目）
```

- `{pluginId}` 采用反向域名格式，如 `com.lx.packet_hijacker`
- `index.js` 是 CommonJS 模块（`module.exports = { ... }`）
- `manifest.json` 提供 meta 元信息；`index.js` 提供 lifecycle / commands。ui/menu/shortcuts 可同时在两处声明，`manifest.json` 优先
- **manifest.json 为必填**，不存在则插件加载失败（旧格式已弃用）

---

## 2. manifest.json 规范

### 2.1 完整示例

```json
{
  "id": "com.lx.my_plugin",
  "name": "我的插件",
  "version": "1.0.0",
  "type": "plugin",
  "description": "插件描述",
  "author": "lx",
  "tags": ["示例", "工具"],
  "entry": "index.js",
  "priority": 0,
  "dependencies": ["com.lx.command"],

  "ui": {
    "pages": [
      {
        "id": "home",
        "title": "我的插件窗口",
        "entry": "ui/index.html",
        "window": {
          "width": 900,
          "height": 650,
          "resizable": true,
          "alwaysOnTop": false
        },
        "menu": [
          {
            "id": "view",
            "label": "视图",
            "submenu": [
              { "id": "reload", "label": "刷新", "command": "reload" },
              { "id": "devtools", "label": "开发者工具", "command": "toggleDevTools" }
            ]
          }
        ]
      }
    ]
  },

  "menu": [
    { "id": "root", "label": "我的插件", "command": "openPage" }
  ],

  "shortcuts": [
    { "id": "f5-refresh", "key": "F5", "command": "reload", "scope": "window" },
    { "id": "f12-devtools", "key": "F12", "command": "toggleDevTools", "scope": "window" },
    { "id": "ctrl-h-focus", "key": "H", "modifiers": ["control"], "command": "focusWindow", "scope": "global" }
  ]
}
```

### 2.2 字段说明

#### 必填字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 全局唯一标识，如 `com.author.name` |
| `name` | string | 显示名称 |
| `version` | string | 语义化版本号 |
| `type` | `"plugin"` | 插件类型 |

#### 可选字段

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `description` | string | `""` | 描述文本 |
| `author` | string | `""` | 作者名 |
| `tags` | string[] | `[]` | 标签列表 |
| `entry` | string | `"index.js"` | JS 入口文件名 |
| `priority` | int | `0` | 菜单排序优先级（越大越靠前） |
| `dependencies` | string[] | `[]` | 依赖的其他插件 ID 列表 |

#### ui.pages

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 页面标识（`ctx.ui.openPage(pageId)` 的参数） |
| `title` | string | 窗口标题 |
| `entry` | string | 页面入口，支持本地路径（`"ui/index.html"`）或 URL（`"http://localhost:5173/"`） |
| `window.width` | int? | 窗口宽度 |
| `window.height` | int? | 窗口高度 |
| `window.resizable` | bool? | 是否可调整大小 |
| `window.alwaysOnTop` | bool? | 是否置顶 |
| `menu` | array? | 窗口级菜单 |

#### menu（插件级和窗口级通用）

每个菜单项：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 唯一标识 |
| `label` | string | 显示文本 |
| `enabled` | bool? | 是否可用 |
| `visible` | bool? | 是否可见 |
| `type` | `"normal"` \| `"separator"` \| `"checkbox"` \| `"radio"`? | 类型 |
| `command` | string? | 点击时触发的命令名（对应 index.js 的 commands） |
| `payload` | any? | 传给命令的额外参数 |
| `submenu` | array? | 子菜单 |

**注意**：快捷键统一使用顶层 `shortcuts[]` 声明。

#### shortcuts

快捷键绑定到命令（不是菜单）。宿主根据 `scope` 自动选择绑定方式：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | 标识 |
| `key` | string | 按键（如 `"F5"`, `"R"`） |
| `modifiers` | array? | 修饰键（`["shift","control","alt","meta"]`） |
| `command` | string | 触发的命令名（对应 index.js 的 commands） |
| `payload` | any? | 额外参数 |
| `scope` | `"window"` \| `"global"` | 默认 `"window"`。`"window"`：绑定到插件窗口菜单（不同窗口同键互不冲突）；`"global"`：系统全局快捷键（先到先得，冲突时后注册者失败） |

---

## 3. index.js 规范

### 3.1 完整示例

```js
module.exports = {
  lifecycle: {
    onLoad:    async (ctx) => { ctx.log.info("已加载"); },
    onEnable:  async (ctx) => { ctx.log.info("已启用"); },
    onDisable: async (ctx) => { ctx.log.info("已禁用"); },
    onUnload:  async (ctx) => { ctx.log.info("已卸载"); },
    onOpen:    async (ctx) => { await ctx.ui.openPage("home"); },
    onDaily:   async (ctx) => {
      // 每日任务逻辑（可选实现）
      ctx.log.info("执行每日任务");
    },
  },

  commands: {
    openPage: async (ctx) => ctx.ui.openPage("home"),
    reload: async (ctx, payload) => {
      if (payload?.windowId) ctx.ui.reload(payload.windowId);
    },
    toggleDevTools: async (ctx, payload) => {
      if (payload?.windowId) ctx.ui.toggleDevTools(payload.windowId);
    },
    myCommand: async (ctx, payload) => {
      ctx.log.info("收到命令 myCommand", payload);
      return { ok: true };
    },
  },
};
```

### 3.2 生命周期

| 钩子 | 触发时机 | 参数 | 说明 |
|------|---------|------|------|
| `onLoad(ctx)` | 插件被加载（启动时或手动加载） | ctx | 只执行一次，此时插件尚未启用 |
| `onEnable(ctx)` | 插件被启用 | ctx | 每次启用都执行 |
| `onDisable(ctx)` | 插件被禁用 | ctx | 每次禁用都执行 |
| `onUnload(ctx)` | 插件被卸载 | ctx | 只执行一次（退出或卸载时） |
| `onOpen(ctx)` | 用户点击"打开"按钮 | ctx | 对应 UI 打开逻辑；如果没有定义，openPlugin 会返回错误 |
| `onDaily(ctx)` | 日常任务执行 | ctx | 可选实现；没有定义则 runDailyPlugin 返回错误 |

**执行顺序**：`onLoad` → `onEnable` → ... → `onDisable` → `onUnload`（退出时）

### 3.3 命令处理器

每个命令函数签名：`(ctx: PluginContext, payload: any) => any`

返回值会作为命令的执行结果。命令通过以下方式触发：
- 菜单项 click → `dispatchCommand(pluginId, command, payload)`
- 快捷键 → `dispatchCommand(pluginId, command, { windowId, key, modifiers })`
- 其他插件/宿主手动调用
- index.js 的 `lifecycle.onOpen` 绑定的命令

**内置命令**（由宿主处理，无需在 index.js 中定义）：

| 命令 | 宿主行为 |
|------|---------|
| `reload` | 调用 `ctx.ui.reload(payload.windowId)` |
| `toggleDevTools` | 调用 `ctx.ui.toggleDevTools(payload.windowId)` |

---

## 4. PluginContext API

`PluginContext`（简称 `ctx`）是宿主提供给插件的受控能力集合。

```ts
type PluginContext = {
  host:   { isDev: boolean };                    // 宿主环境信息
  plugin: { id: string; dir: string };          // 当前插件信息
  log:    LogAPI;                                // 日志接口
  game:   GameAPI;                                // 游戏通信接口
  ui:     UiAPI;                                  // 窗口管理接口
  menu:   MenuAPI;                                // 菜单刷新
  storage: StorageAPI;                            // 本地持久化存储
};
```

### 4.1 host

```ts
ctx.host.isDev  // boolean — 宿主是否运行在开发模式
```

### 4.2 plugin

```ts
ctx.plugin.id   // string — 插件 ID
ctx.plugin.dir  // string — 插件目录的绝对路径
```

### 4.3 log

```ts
ctx.log.info(...args)   // info 级别日志，同时输出到 console 和 LogBus
ctx.log.warn(...args)   // warn 级别日志
ctx.log.error(...args)  // error 级别日志
```

所有日志自动附带 `[plugin:{pluginId}]` 前缀，并路由到 LogBus 环形缓冲。

### 4.4 game

上接 game API 基础说明


### 4.4.1 `game.on` / `game.emit` — 事件发布订阅

`game.on` 和 `game.emit` 是插件间通信和与 DLL 交互的核心机制，遵循**发布-订阅**模式：

| API | 方向 | 说明 |
|-----|------|------|
| `game.on(eventName, callback)` | 订阅 | 监听指定事件，返回取消函数 |
| `game.emit(eventName, params, callback?)` | 发布 | 发送事件/命令，可带回调获取返回值 |

**对应关系**：同一 `eventName` 的 `emit` 和 `on` 一一对应。`emit` 发送的消息会被所有注册了相同 `eventName` 的 `on` 回调接收到。多个插件可以同时监听同一个事件，彼此隔离。

**跨插件通信**：A 插件 `emit("myEvent", data)`，B 插件通过 `on("myEvent", handler)` 即可接收。事件名建议使用 `插件ID.事件名` 避免冲突，如 `com.lx.packet_hijacker.rulesUpdated`。

**DLL 通信**：`emit` 发往 DLL 的命令会自动通过 TCP 发送，DLL 的推送事件通过 `on` 接收。常用 DLL 事件：

| eventName | 方向 | 说明 |
|-----------|------|------|
| `game.packet.recv` | DLL→插件 | 收到游戏封包 |
| `game.packet.sent` | DLL→插件 | 发出游戏封包 |
| `game.login` | DLL→插件 | 游戏登录 |
| `game.logout` | DLL→插件 | 游戏登出 |
| `game.packet.send` | 插件→DLL | 向游戏发送封包 |
| `game.speed.set` | 插件→DLL | 设置游戏倍速 |
| `game.refresh` | 插件→DLL | 重置游戏状态 |

**示例 — 跨插件协作**：

```js
// 插件 A (com.lx.auto_battle)：发布战斗结果
module.exports = {
  lifecycle: {
    onEnable: async (ctx) => {
      const client = ctx.game.getClient();
      client.on("game.packet.recv", (data) => {
        if (data.cmd === 5001) {
          // 战斗结束，通知其他插件
          client.emit("com.lx.auto_battle.result", { win: true, exp: 1200 });
        }
      });
    },
  },
};

// 插件 B (com.lx.log_viewer)：订阅战斗结果
module.exports = {
  lifecycle: {
    onEnable: async (ctx) => {
      const client = ctx.game.getClient();
      client.on("com.lx.auto_battle.result", (data) => {
        ctx.log.info(`战斗结果: ${JSON.stringify(data)}`);
        // win: true, exp: 1200
      });
    },
  },
};
```

**示例 — 多层封装**：

```js
// 业务层封装
function watchBattleEnd(ctx, handler) {
  const client = ctx.game.getClient();
  return client.on("game.packet.recv", (data) => {
    if (data.cmd === 5001) {
      const result = parseBattleResult(data);
      handler(result);
    }
  });
}

// 使用
const unsub = watchBattleEnd(ctx, (result) => {
  if (result.win) ctx.log.info(`战斗胜利，获得 ${result.exp} 经验`);
});
```



### 4.4.2 DLL 事件参考

DLL 通过 TCP 推送的 JSON 事件，使用 client.on(eventName, callback) 订阅。

**DLL 推送事件**：

| eventName | data | 说明 |
|-----------|------|------|
| game.login | { userId: number } | 游戏登录成功 |
| game.logout | { userId: number } | 游戏登出 |
| game.packet.recv | { packet, cmdId, userId, length } | 收到游戏封包 |
| game.packet.sent | { packet, cmdId, userId, length } | 发出游戏封包 |
| _protocol_version | { version: number } | 握手协议版本 |

**客户端命令**：

| eventName | 参数 | 说明 |
|-----------|------|------|
| game.refresh | {} | 重置游戏状态 |
| game.status | {} | 查询游戏状态 |
| game.packet.send | { packet } | 发送封包到服务器 |
| game.speed.set | { factor } | 设置倍速 |
| watch.open | { mode, cmds, direction, timeout? } | 开启流式监听/劫持 |
| watch.close | { id } | 关闭流式监听 |

**已废弃**：intercept.start / intercept.stop / intercept.response / sentIntercept.* 已由 watch 替代。game.packet.recv.intercepted 和 game.packet.sent.intercepted 仅保留兼容。




























### 4.5 ui

```ts
// 打开 UI 页面（必须先在 manifest.json 的 ui.pages 中声明）
const { windowId } = await ctx.ui.openPage("home", { width: 800, height: 600 });

ctx.ui.closeWindow(windowId)                  // 关闭窗口
ctx.ui.reload(windowId)                       // 刷新窗口
ctx.ui.toggleDevTools(windowId)               // 打开/关闭 DevTools
ctx.ui.setAlwaysOnTop(windowId, true)         // 设置窗口置顶
ctx.ui.getWindow(windowId)                    // 获取 SeerWindow 实例（Electron API 级）
ctx.ui.setMenu(windowId, menuNodes)           // 动态设置窗口菜单
```

### 4.6 menu

```ts
ctx.menu.refresh()   // 强制重建 Electron 应用菜单
```

### 4.7 storage

持久化 key-value 存储，串行化写入保证数据安全。

```ts
const val = await ctx.storage.get("myKey");      // 读取
await ctx.storage.set("myKey", { foo: 42 });     // 写入（自动 JSON 序列化）
```

- 存储位置：`plugins/.storage/{pluginId}.json`
- 写入串行化：并发 `set()` 调用自动排队，避免 read-modify-write 竞态
- 每个插件的存储独立，互不干扰

---

## 5. 前端 API（window.$xxx）

插件 UI 页面（通过 `ctx.ui.openPage` 打开的窗口）可以通过预加载脚本暴露的 API 与宿主通信。

### 5.1 $plugin

```ts
// 获取已安装插件列表
const list: Array<{
  id: string; name: string; version: string; type: string;
  description: string; author: string; tags: string[];
  enabled: boolean; dir: string; hasUi: boolean;
}> = await window.$plugin.getInstalled();

// 从云端 URL 下载并安装
const { success, pluginId } = await window.$plugin.installFromUrl("http://...");

// 从本地文件安装（弹出文件选择对话框）
const { success, pluginId, cancelled } = await window.$plugin.installFromFile();

// 启用/禁用/打开/卸载
await window.$plugin.enable("com.lx.xxx");
await window.$plugin.disable("com.lx.xxx");
await window.$plugin.open("com.lx.xxx");
await window.$plugin.uninstall("com.lx.xxx");

// 日常任务
const tasks = await window.$plugin.getDailyTasks();
await window.$plugin.saveDailyTasks([...]);
```

### 5.2 $game

```ts
// 获取 DLL 通信客户端（同步，立即返回——通过 IPC 代理主进程单例）
const game = await window.$game.getClient(port?, ip?);
//   内部通过 IPC 桥接到主进程 GameClient 单例。
//   多次调用 getClient() 返回的是同一个主进程实例的代理。

// 显式连接 / 断开
game.connect();
game.disconnect();

// 连接状态
game.status();                     // Promise<{ status, protocolVersion }>
game.whenReady();                  // Promise — 等待连接就绪
game.onWhenReady(() => { ... });   // 连接就绪回调，返回取消注册函数
game.onClosed(() => { ... });      // 连接关闭回调，返回取消注册函数

// 监听 DLL 推送事件
const unsubscribe = game.on("game.packet.recv", (data) => { ... });
game.on("game.packet.recv.intercepted", (data) => { ... });

// 精准监听封包（推荐）
const watcher = await game.watch("listen", [1001, 2002], "recv", (data) => { ... });
await watcher.close();

// 全量监听收发包（调试用）
await game.watch("listen", "all", "all", (data) => { ... });

// 劫持封包
await game.watch("hijack", [1001], "send", (data) => {
  return { action: "drop" }; // pass | modify | drop
}, { timeout: 5000 });

// 向 DLL 发送命令
game.emit("intercept.start", { cmds: [1001], timeout: 3000 }, (res) => { ... });
game.emit("intercept.stop", {});
game.emit("intercept.response", { id, action, packet? });

unsubscribe();

// 封包工具（同步，不需要 game 实例）
window.$game.packPacket(params);
window.$game.unpackPacket(hex);

// 插件元信息
window.$game.pluginMeta;  // { id, dir }

// 插件存储
await window.$game.storage.get("key");
await window.$game.storage.set("key", value);
```

> **React 组件中推荐 `useRef` 存 game 实例**，避免闭包捕获旧值：
> ```tsx
> const gameRef = useRef(null);
> useEffect(() => {
>   window.$game.getClient().then(g => { gameRef.current = g; });
> }, []);
> // 回调中用 gameRef.current.emit(...)
> ```

**DLL 劫持命令参考**：

### 5.2.0 watch 精准监听/劫持（推荐）

`watch` 是封包监听和劫持的统一入口：

```ts
const watcher = await game.watch("listen", [1001], "recv", (data) => {
  console.log(data.cmdId, data.packet);
});

await watcher.close();
```

参数：

| 参数 | 类型 | 说明 |
|------|------|------|
| `mode` | `"listen" | "hijack"` | 监听或劫持 |
| `cmds` | `number[] | "all"` | 要匹配的 cmdId；`"all"` 仅允许 listen |
| `direction` | `"send" | "recv" | "all"` | 封包方向 |
| `callback` | `(data) => void | { action, packet? }` | 监听模式忽略返回值，劫持模式同步返回处理动作 |
| `options.timeout` | `number?` | 劫持等待响应超时，默认 3000ms |

规则：

- `watch("listen", "all", "all", callback)` 允许全量监听，用于调试。
- 多个客户端可以监听同一个 `cmdId + direction`。
- 同一个 `cmdId + direction` 只能被一个客户端劫持。
- `watch("hijack", "all", ...)` 明确禁止。


### 5.2.1 封包监听/劫持（watch）

新版统一使用 client.watch() 替代旧的 intercept/sentIntercept 命令。详见 4.4.2 节和 6.3 节。


### 5.3 $log

```ts
const entries = await window.$log.getRecent();
const unsubscribe = window.$log.subscribe((entry) => {
  // entry: { timestamp, pluginId, level, message }
});
window.$log.info("message");
window.$log.warn("message");
window.$log.error("message");
window.$log.clear();
```

### 5.4 $settings

```ts
const { path, exists } = window.$settings.getDefaultPath();
const selectedPath = await window.$settings.selectPath();
const { exists } = window.$settings.checkPath("D:\\...");
await window.$settings.launchGame();
await window.$settings.injectMod();
await window.$settings.restoreMod();
const { injected } = window.$settings.checkModStatus();
```

---

## 6. 完整示例

### 6.1 最小插件（无 UI，只有命令）

```
plugins/com.lx.hello/
├── manifest.json
└── index.js
```

**manifest.json**：
```json
{
  "id": "com.lx.hello",
  "name": "Hello",
  "version": "1.0.0",
  "type": "plugin",
  "entry": "index.js"
}
```

**index.js**：
```js
module.exports = {
  lifecycle: {
    onLoad: async (ctx) => { ctx.log.info("Hello 已加载"); },
  },
  commands: {
    sayHello: async (ctx) => {
      ctx.log.info("Hello World!");
      return { message: "Hello World!" };
    },
  },
};
```

### 6.2 带 UI 的插件（内嵌 HTML）

```
plugins/com.lx.hello/
├── manifest.json
├── index.js
└── ui/
    └── index.html
```

**manifest.json**：
```json
{
  "id": "com.lx.hello",
  "name": "Hello",
  "version": "1.0.0",
  "type": "plugin",
  "entry": "index.js",
  "ui": {
    "pages": [{
      "id": "home",
      "title": "Hello",
      "entry": "ui/index.html",
      "window": { "width": 600, "height": 400 },
      "menu": [{
        "id": "view",
        "label": "视图",
        "submenu": [
          { "id": "reload", "label": "刷新", "command": "reload" }
        ]
      }]
    }]
  },
  "menu": [{ "id": "root", "label": "Hello", "command": "openPage" }],
  "shortcuts": [
    { "id": "f5-refresh", "key": "F5", "command": "reload", "scope": "window" }
  ]
}
```

**index.js**：
```js
module.exports = {
  lifecycle: {
    onLoad: async (ctx) => { ctx.log.info("Hello 已加载"); },
    onOpen: async (ctx) => { await ctx.ui.openPage("home"); },
  },
  commands: {
    openPage: async (ctx) => ctx.ui.openPage("home"),
    reload: async (ctx, payload) => { if (payload?.windowId) ctx.ui.reload(payload.windowId); },
  },
};
```

**ui/index.html**：
```html
<!DOCTYPE html>
<html>
<head><meta charset="UTF-8"><title>Hello</title></head>
<body>
  <h1>Hello Plugin</h1>
  <p>pluginId: <span id="id"></span></p>
  <button onclick="sayHello()">Say Hello</button>
  <script>
    document.getElementById("id").textContent = window.$game?.pluginMeta?.id || "N/A";
    function sayHello() {
      window.$log.info("Hello from UI!");
    }
  </script>
</body>
</html>
```

### 6.3 封包劫持器（watch 方案）

新版使用 `client.watch()` 替代旧的 `intercept.start/stop` 命令。`watch` 直接向 DLL 注册流式监听，无需逐个手动回复。

```ts
const client = ctx.game.getClient();
await client.whenReady();

// 收包劫持：拦截 cmdId 1001 和 2002，30 秒内未响应自动放行
const watcher = await client.watch(
  "hijack",
  [1001, 2002],
  "recv",
  (data) => {
    // 返回 { action: "pass" | "modify" | "drop", packet?: hex }
    return { action: "pass" };
  },
  { timeout: 30000 }
);

// 停止劫持
await watcher.close();
```

| mode | 说明 |
|------|------|
| `"hijack"` | 劫持模式：拦截封包，等待客户端响应后决定放行/修改/丢弃 |
| `"intercept"` | 监听模式：只观察封包，不影响游戏流程 |

`direction` 字段：`"recv"`（收包）/ `"sent"`（发包）。


---

## 7. 最佳实践

1. **manifest.json 优先**：meta/ui/menu/shortcuts 在 manifest.json 中声明，index.js 只写运行时逻辑
2. **`ctx.ui.openPage("home")` 统一入口**：插件打开窗口的唯一方式
3. **`ctx.log` 记日志**：不要用 `console.log`，否则不会路由到 LogBus
4. **`ctx.storage` 持久化**：串行化安全写入，自动 JSON 序列化
5. **快捷键用顶层 `shortcuts[]`**：统一在 manifest.json 顶层声明快捷键绑定
6. **`window.$game` 通过 IPC 代理 DLL 连接**：插件 UI 通过 `$game.getClient()` 获取 IPC 代理对象，底层桥接到主进程 GameClient 单例，避免多窗口创建重复连接
7. **禁用状态持久化**：宿主自动处理，插件无需关心
8. **onOpen 定义**：在 index.js 的 `lifecycle.onOpen` 中定义，`openPlugin` 会触发它
9. **GameClient 连接时序**：`getClient()` 同步返回，`on()` 安全注册监听器不触发连接，`emit()` 自动连接+排队发送。建议先 `on` 注册监听再 `connect()`，或通过 `onWhenReady` / `whenReady()` 确保连接就绪
10. **类型声明**：在 `index.js` 顶部添加 `/// <reference path="./plugin.d.ts" />`，结合 JSDoc `@param {PluginContext}` 即可获得完整代码提示
11. **脚手架**：运行 `npx create-plugin` 交互式生成插件项目骨架（含 `plugin.d.ts`），可一键创建带/不带 UI 的插件项目
12. **热重载**：修改插件代码后，在「本地插件」面板点击「重载」按钮即可热加载，无需重启客户端
