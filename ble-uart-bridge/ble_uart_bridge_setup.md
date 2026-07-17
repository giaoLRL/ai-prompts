# 小程序 BLE ↔ MCU 多设备通信链路：搭建与调试提示词

将此提示词粘贴给 AI，适用于用 ESP32 做 BLE-UART 透传桥，连接微信小程序和 MCU 的场景。

---

## 架构

```
微信小程序 ←→ BLE ←→ ESP32 (透传桥) ←→ UART ←→ MCU (主控)
   [端1]              [端2]               [端3]
```

---

## 通信协议（强制统一）

三端代码必须使用同一个协议，确保无论谁写的代码，换一个项目也能互通。

### 协议格式

```
帧分隔符：\n (换行符，0x0A)
编码：ASCII（纯文本，禁止二进制/JSON）
最大帧长：512 字节（超出丢弃并告警）
```

### 方向1：小程序 → MCU（命令）

| 命令 | 格式 | 说明 |
|------|------|------|
| PING | `PING\n` | 心跳，MCU 回复 `OK\n` |
| STOP | `STOP\n` | 紧急停车 |
| GO | `GO\n` | 解除停车 |
| RST | `RST\n` | 软复位 |
| SET | `SET_参数名=值\n` | 设置参数 |
| CFG | `CFG_参数名=值\n` | 同 SET |
| GET | `GET_参数名\n` | 查询参数 |
| LIST | `LIST\n` | 列出所有参数 |

### 方向2：MCU → 小程序（遥测）

| 响应 | 格式 | 说明 |
|------|------|------|
| OK | `OK\n` | 命令执行成功 |
| ERR | `ERR_UNKNOWN_CMD\n` | 未知命令 |
| VAL | `VAL_参数名=值\n` | GET 响应 |
| STAT | `STAT tick=N state=S error=E speed=L,R\n` | 遥测数据（5Hz） |

---

## CCP-CD 陷阱

微信 `notifyBLECharacteristicValueChange` 的 success 回调在某些 Android 机型上不可靠。ESP32 端必须使用 `gatts_write(send_update=True)` 或无条件调用 `gatts_notify()` 并用返回值判断，不能依赖 IRQ 回调追踪 CCCD。

## UART FIFO 陷阱

SysConfig/CubeMX 默认 RX FIFO Threshold = `>= 1/2 full` (2字节) 会导致奇数字节的短命令尾字节卡死。必须在代码中覆盖为 `>= 1 byte` + RX Timeout ≠ 0。

## 三端防坑

- **ESP32**: CCCD 写入不转发给 MCU、软复位后 BLE 残留重试、固件版本校验
- **MCU**: 环形缓冲区 + 非阻塞 TX、命令解析 + 错误回复
- **小程序**: 变量声明防崩溃、注册监听先于扫描、notify 重试 3 次、行缓冲防分片

## 三端数据流对照

| 步骤 | 小程序端 | ESP32 日志 | MCU 端 | 通过标志 |
|------|---------|-----------|--------|---------|
| 1 | — | 🟢 广播中 | — | 看到广播日志 |
| 2 | ✅ 已连接 | ✅ 已连接 | — | 两端都显示 |
| 3 | ✅ Notify 启用 | 🔍 CCCD 写入! | — | ESP32 收到 CCCD |
| 4 | 📡 收到: OK | 📱→MCU: PING | cmd_count++ | 小程序收 OK |
| 5 | — | 🔵 BLE→APP > 0 | — | 遥测数据流通 |

## 快速使用

```
我要搭建一个小程序控制 MCU 的系统，架构为：
微信小程序 ←→ BLE ←→ ESP32 ←→ UART ←→ [MCU 型号]
MCU UART 引脚：[TX, RX]
控制对象：[电机/舵机/电磁铁]
需要在小程序上调节的参数：[kp, ki, kd, 速度]
请按照 ble_uart_bridge_setup.md 规范生成三端代码。
```
