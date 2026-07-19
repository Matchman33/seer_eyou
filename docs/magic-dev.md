# ✨ 魔法脚本开发指南

> 魔法脚本是用 JavaScript 编写的轻量级自动化脚本，通过 fork 子进程运行，适合快速实现自定义游戏逻辑。

---

## 1. 脚本目录结构

```
magic/{pluginId}/
├── manifest.json    ← 必填。元信息与入口配置
└── {entry}.mjs       ← 脚本入口（ES Module）
```

`{pluginId}` 为唯一标识，如 `com.lx.auto_battle`。

---

## 2. manifest.json

```json
{
  "type": "magic",
  "id": "com.lx.auto_battle",
  "name": "自动战斗",
  "version": "1.0.0",
  "description": "自动识别怪物并战斗",
  "author": "lx",
  "tags": ["战斗", "自动化"],
  "entry": "index.mjs"
}
```

| 字段 | 必填 | 说明 |
|------|------|------|
| `type` | ✅ | 固定为 `"magic"` |
| `id` | ✅ | 全局唯一标识 |
| `name` | ✅ | 显示名称 |
| `version` | | 版本号 |
| `description` | | 描述 |
| `entry` | | 入口文件名，默认 `index.mjs` |

---

## 3. 入口脚本

使用 ES Module 语法（`.mjs` 后缀）：

```js
// index.mjs
import { connect, log, sleep, packPacket, unpackPacket } from "./sdk.mjs";

const game = await connect();

log.info("脚本启动");

// 发送封包
game.send(packPacket({ cmd: 1001, data: "000102" }));

// 等待
await sleep(1000);

// 监听收包
game.onPacket(2002, (data) => {
  log.info("收到响应", data);
  game.exit();
});
```

---

## 4. SDK API（sdk.mjs）

### 连接与生命周期

| API | 说明 |
|-----|------|
| `connect({ host?, port?, timeout? })` | 连接 DLL → `Promise<Game>` |
| `game.status` | `"idle"` / `"running"` / `"closed"` |
| `game.protocolVersion` | DLL 协议版本号 |
| `game.whenReady()` | 等待连接就绪 → `Promise<void>` |
| `game.onWhenReady(cb)` | 连接就绪回调 → 取消函数 |
| `game.onClosed(cb)` | 连接关闭回调 → 取消函数 |
| `game.disconnect()` | 主动断开连接 |
| `game.exit()` | 断开 + `process.exit(0)` |

### 事件通信

| API | 说明 |
|-----|------|
| `game.on(eventName, callback)` | 注册事件监听 → 取消函数 |
| `game.off(eventName, callback?)` | 取消事件监听 |
| `game.emit(eventName, params?, callback?)` | 发送命令（回调风格） |
| `game.emitAsync(eventName, params?, timeoutMs?)` | 发送命令 → `Promise` |

### 封包操作

| API | 说明 |
|-----|------|
| `game.send(hex)` | `game.packet.send` 快捷方式 |
| `game.onPacket(cmdId, callback)` | 按 cmdId 过滤收包 |
| `packPacket({ cmd, data, account?, version? })` | 构造封包 hex |
| `unpackPacket(hex)` | 解析封包 → `{ length, version, cmd, account, data }` |

### 流式监听/劫持

| API | 说明 |
|-----|------|
| `game.watch(mode, cmdIds, direction, callback, options?)` | 开启流式监听/劫持 → `{ id, close() }` |

| 参数 | 说明 |
|------|------|
| `mode` | `"hijack"`（劫持）/ `"intercept"`（监听） |
| `cmdIds` | 封包 cmdId 数组，`hijack` 模式不支持 `"all"` |
| `direction` | `"recv"`（收包）/ `"sent"`（发包） |
| `callback(data)` | 返回 `{ action: "pass" \| "modify" \| "drop", packet?: hex }` |
| `options.timeout` | 劫持超时（ms），默认 3000 |

### 工具函数

| API | 说明 |
|-----|------|
| `log.info(msg)` / `log.warn(msg)` / `log.error(msg)` | 日志输出 |
| `sleep(ms)` | 异步等待（毫秒） |
| `game.pause()` / `game.resume()` | 倍速控制（factor=0/1） |

---

## 5. 编辑器使用

在客户端「本地插件」→ 切换到「魔法」类型 → 点击脚本进入编辑器：

- **新建**：输入 pluginId 自动生成 manifest.json + 模板脚本
- **编辑**：语法高亮，Ctrl+S 保存
- **运行**：点击运行按钮，fork 子进程执行
- **停止**：点击停止按钮，kill 子进程
- **日志**：运行日志实时输出到控制台
