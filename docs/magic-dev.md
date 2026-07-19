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
import { log, game, sleep } from "./sdk.mjs";

log.info("脚本启动");

// 发送封包
const packet = game.packPacket({
  cmd: 1001,
  data: "00010203"
});
game.send(packet);

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

| API | 说明 |
|-----|------|
| `log.info(msg)` / `log.warn(msg)` / `log.error(msg)` | 日志输出 |
| `game.send(packet)` | 发送封包 hex 字符串 |
| `game.onPacket(cmdId, callback)` | 监听指定 cmdId 的收包 |
| `game.offPacket(cmdId)` | 取消监听 |
| `game.packPacket({cmd, data})` | 封装封包 hex |
| `game.unpackPacket(hex)` | 解析封包 |
| `game.pause()` / `game.resume()` | 暂停/恢复封包转发 |
| `sleep(ms)` | 异步等待（毫秒） |
| `game.exit()` | 退出脚本 |

---

## 5. 编辑器使用

在客户端「本地插件」→ 切换到「魔法」类型 → 点击脚本进入编辑器：

- **新建**：输入 pluginId 自动生成 manifest.json + 模板脚本
- **编辑**：语法高亮，Ctrl+S 保存
- **运行**：点击运行按钮，fork 子进程执行
- **停止**：点击停止按钮，kill 子进程
- **日志**：运行日志实时输出到控制台
