# 🔮 易游插件管理器

> 赛尔号游戏辅助工具 — 插件化架构，支持封包拦截、内置插件、一键日常。

[![下载](https://img.shields.io/badge/下载-最新版本-blue)](https://github.com/Matchman33/seer_eyou/releases/latest)
[![文档](https://img.shields.io/badge/文档-开发者指南-green)](https://matchman33.github.io/seer_eyou/)

---

## ✨ 功能

- **插件系统** — 支持安装/卸载/启用/禁用，独立窗口运行
- **封包拦截** — 实时拦截游戏收发包，支持修改/丢弃/劫持
- **内置插件** — 内置插件清单，首次启动自动安装、设置页一键下载
- **倍速控制** — 调整游戏速度
- **一键日常** — 定时执行每日任务
- **自动更新** — 启动时自动检测新版本并静默更新

## 📥 下载

前往 [Releases](https://github.com/Matchman33/seer_eyou/releases/latest) 下载最新安装包。

## 📖 文档

- [插件开发指南](docs/plugin-dev.md) — 如何开发一个插件

## 🛠 技术栈

| 层 | 技术 |
|----|------|
| 客户端 | Electron + TypeScript |
| 前端 UI | React + Ant Design + Vite |
| DLL Hook | C++ + Microsoft Detours |
| 通信协议 | TCP JSON 帧（`\n` 分隔） |
| 打包分发 | electron-builder + NSIS |
| 自动更新 | electron-updater + GitHub Releases |

## 📄 License

MIT
