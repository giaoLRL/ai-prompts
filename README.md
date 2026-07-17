# AI 项目提示词仓库

从嵌入式调试实战中提炼的通用 AI 提示词模版。每个提示词放在独立文件夹中，版本更新时在同一文件夹内追加新文件。

## 提示词列表

| 文件夹 | 场景 | 一句话 |
|--------|------|--------|
| [embedded-debug-infra](./embedded-debug-infra/) | 代码跑了但不对 | 让 AI 搭建 debug.h/debug.c 调试基础设施（全局结构体 + 心跳 GPIO + 环形缓冲） |
| [hardware-root-cause](./hardware-root-cause/) | 软件排查穷尽 | 强制 AI 穷举硬件故障并做现象对照表，不再只改代码 |
| [flash-debug-connect](./flash-debug-connect/) | 烧录/调试器连不上 | 四层排查：工具链 → 硬件连接 → 供电噪声 → 芯片状态 |
| [ble-uart-bridge](./ble-uart-bridge/) | 小程序 ↔ ESP32 ↔ MCU 通信 | 统一协议 + 三端代码规范 + CCCD/FIFO 防坑 + 诊断日志 |

## 使用方式

1. 找到对应场景的文件夹
2. 复制 `.md` 文件内容
3. 粘贴给 AI，按模板填入你的项目信息

## 版本管理

每个提示词独立一个文件夹。更新时在文件夹内创建新版文件（如 `xxx_v2.md`），旧版本保留不删。
