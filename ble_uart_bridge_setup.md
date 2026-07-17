# 小程序 BLE ↔ MCU 多设备通信链路：搭建与调试提示词

将此提示词粘贴给 AI，适用于用 ESP32 做 BLE-UART 透传桥，连接微信小程序和 MCU 的场景。

---

## 架构

```
微信小程序 ←→ BLE ←→ ESP32 (透传桥) ←→ UART ←→ MCU (主控)
   [端1]              [端2]               [端3]
```

---

## 搭建要求

请按以下规范同时搭建三端代码，确保每条链路都内置诊断日志：

### 1. ESP32 透传桥（MicroPython）

#### 1.1 强制要求

| 要求 | 说明 |
|------|------|
| **设备名固定** | 使用 `JDY-23` 或用户指定的名称 |
| **Service UUID** | `0000FFE0-0000-1000-8000-00805F9B34FB` |
| **Characteristic UUID** | `0000FFE1-0000-1000-8000-00805F9B34FB` |
| **单特征双属性** | 同一个 FFE1 特征同时支持 Write + Notify，模拟常见 BLE-UART 模块 |
| **UART 参数** | 9600bps, 8N1, TX=GPIO17, RX=GPIO16（默认，可按需调整） |

#### 1.2 CCCD 陷阱（必须内置）

**问题**：微信小程序的 `wx.notifyBLECharacteristicValueChange` API 在某些 Android 机型上返回 `success`，但底层的 GATT CCCD Write 根本没发到设备。

**强制方案**：
- ESP32 端**不依赖** IRQ 回调追踪 CCCD 状态
- 使用 NimBLE 的 `gatts_write(send_update=True)` 直接更新特征值（NimBLE 内部管理 CCCD 状态）
- 或直接无条件调用 `gatts_notify()`，用返回值判断是否成功（而非用 `_notify_enabled` 门控）
- 在日志中打印：`notify尝试N 成功M 失败F`，用于对比手机端是否真的启用了通知

#### 1.3 诊断日志格式

ESP32 USB 串口（115200bps）必须持续输出以下日志：

```
================================================
  ESP32 BLE-UART 透传桥
  模拟 JDY-23 | MicroPython
================================================
[   0.0] BLE 初始化...
[   0.1] BLE active, MAC: AA:BB:CC:DD:EE:FF
[   0.1] GATT 服务已注册, chr_handle=H
[   0.1] UART2 初始化: 9600bps, TX=17, RX=16
[   0.1] 🟢 广播中... 设备名: JDY-23
[  15.3] ✅ 已连接 (第1次) | chr_handle=H
[  15.4] 🔍 CCCD 写入! handle=H+1 value=01 00 [通知=ON]   ← 关键：手机写入 CCCD 的证据
[  15.5] 📱→MCU [18B]: PK_straight_kp=0.31                ← 手机发来的每条命令
[  20.0] 🔵 统计: UART←MCU 1240B | BLE→APP 980B/20包 | notify尝试22 成功20 失败2 | 连接1次
```

**两端数据量的对标是关键诊断指标**：
- `UART←MCU` = MCU 实际发出的字节数
- `BLE→APP` = 成功 notify 到手机的字节数
- 如果 `UART←MCU` 很大但 `BLE→APP = 0` → notify 链路断（CCCD 未启用或 `gatts_notify()` 返回 False）
- 如果 `UART←MCU = 0` → MCU 没发数据（UART 或 MCU 固件问题）

#### 1.4 防坑清单

| 坑 | 预防措施 |
|----|---------|
| CCCD 写入被当数据转发给 MCU | `_IRQ_GATTS_WRITE` 中区分 `attr_handle`：等于特征句柄→数据转发，等于特征句柄+1→CCCD 状态更新，**不**转发 |
| 软复位后 BLE 残留 | `__init__` 中 try/except 重试 5 次，全失败则提示用户硬复位 |
| 固件版本不一致 | 启动时打印哈希或关键函数名，便于确认板上代码 = 磁盘代码 |
| UART 半包丢弃 | timeout 必须 > flush 间隔（建议 timeout=500ms，flush=200ms） |
| BLE 通知队列溢出 | notify 频率不超过 MCU 数据产生率（MCU 200ms 周期 → BLE ≤ 5Hz） |

---

### 2. MCU 端（UART 接收）

#### 2.1 UART 中断配置（导致命令静默丢失的根因）

**最常见的坑**：SysConfig/CubeMX 的默认值对短命令是致命的。

| 设置 | 错误默认值 | 正确值 | 原因 |
|------|-----------|--------|------|
| RX FIFO Threshold | `>= 1/2 full` (2字节) | `>= 1 byte` | `GO\n` 3 字节：前 2 字节触发中断 → `\n` 只剩 1 字节 → 不到阈值 → 永远卡死 |
| RX Timeout Counts | `0` (禁用) | `10` (~1ms) | 奇数字节的尾字节在无超时机制下永远不被清空 |

**强制措施**：在代码的 `UART_init()` 中显式覆盖这两个值为正确值，不依赖图形化配置工具的默认值。

#### 2.2 环形缓冲区

- 使用环形缓冲区接收 UART 数据，ISR 只写 `rx_buf`，主循环 `poll()` 中解析
- `parse_line()` 以 `\n` 为分隔符提取完整命令
- 发送用 `send_str()` 非阻塞化：TX FIFO 满时保存断点返回，`poll()` 中续传

#### 2.3 诊断建议

- 加一个 `g_debug_cmd_count` 累积已解析的命令数
- 加一个 `g_debug_uart_rx_bytes` 累积 UART RX 总字节数
- 方便在调试器中直接对比：
  - ESP32 日志显示 `📱→MCU 100B`，MCU 端 `g_debug_uart_rx_bytes = 100` → 物理链路 OK
  - ESP32 日志显示 `📱→MCU 100B`，但 MCU 端 `g_debug_cmd_count = 0` → UART 中断/解析有问题

---

### 3. 小程序端

#### 3.1 强制要求

| 要求 | 说明 |
|------|------|
| **deviceName 变量** | 声明为页面数据或 `this` 属性，确保在 `findChars()` 等回调中可访问 |
| **charNotify 初始化** | 明确初始化为 `false`，并在成功启用 notify 后置 `true` |
| **`onBLEConnectionStateChange`** | 必须注册，断线时更新 UI 状态并清理连接 |
| **`sendCmd` fail 回调** | 写入失败时 toasting 用户 + 清理连接状态 |
| **notify 订阅重试** | `wx.notifyBLECharacteristicValueChange` success 回调**不可靠**（见上），加 3 次重试间隔 500ms/1000ms/2000ms |
| **数据缓冲** | BLE 接收用行缓冲（按 `\n` 分割），512 字节安全上限，防分片导致半包 |
| **UUID 过滤** | 用 `device.name` 或 `localName` 过滤扫描结果，避免列出不相干的蓝牙设备 |
| **`charWrite` 动态发现** | 不硬编码 `serviceId`，遍历服务找到 FFE1 的 Write 属性特征 |

#### 3.2 必须的输出日志

在微信开发者工具 vConsole 中应能看到：

```
[BLE] 开始扫描...
[BLE] 发现设备: JDY-23 (deviceId=xxx)
[BLE] 正在连接...
[BLE] ✅ 已连接
[BLE] 发现 1 个服务, 1 个特征
[BLE] 🔔 开始启用通知 (尝试1/3)...
[BLE] ✅ Notify 启用成功                      ← 可能不可靠
[BLE] 📡 收到数据: PK_straight_kp=0.31         ← 实际数据到达的证据
```

### 3.3 防坑清单

| 坑 | 预防 |
|----|------|
| `name` 未定义 → 连接崩溃 | 所有回调 closure 中确保所有使用的变量已声明 |
| `connectBLE` 先扫描后注册监听 → 竞态丢包 | 先注册 `onBLEConnectionStateChange`，再 `startBluetoothDevicesDiscovery` |
| `charNotify` 未初始化 → 监听器重复注册 | 初始化为 `false`，注册后置 `true` |
| `sendCmd` 静默失败 | 必须有 `fail` 回调 → toast + 清理连接 |
| 数据分片导致解析错误 | 行缓冲按 `\n` 拼接，超 512 字节丢弃并告警 |
| `c.properties.notify` 不可靠 | 不依赖此字段，只凭 UUID 匹配就尝试启用 notify |

---

## 排查决策树（当通信不通时按此顺序排查）

```
小程序连接不上 ESP32？
  │
  ├→ ESP32 USB 日志看到 🟢 广播中 了吗？
  │   └→ 没看到 → ESP32 固件没运行或 BLE 初始化失败 → 硬复位 ESP32
  │
  ├→ 看到 ✅ 已连接 了吗？
  │   └→ 没看到 → 手机蓝牙权限 / 设备名不匹配 / BLE 广播间隔过大
  │
  ├→ 看到 🔍 CCCD 写入! handle=H+1 了吗？
  │   └→ 没看到 → 微信 API 假成功，尝试断开重连 / 换手机测试
  │
  ├→ BLE→APP > 0 了吗？
  │   └→ = 0，但 CCCD 已启用 → `gatts_notify()` 失败，检查数据长度是否超 MTU
  │   └→ > 0 → ✅ BLE 链路 OK，看 MCU 端
  │
  ├→ MCU 收到命令了吗？（g_debug_cmd_count > 0）
  │   └→ = 0，但 ESP32 显示 📱→MCU 有数据 → UART 中断配置问题
  │        → 检查 FIFO Threshold = 1 byte, RX Timeout ≠ 0
  │        → 检查 TX/RX 引脚是否接反
  │
  └→ MCU 解析了命令但没响应？
       → UART TX 发送是否使用了非阻塞续传？
       → 命令解析逻辑是否正确？
```

---

## 使用方式

```
我要搭建一个小程序控制 MCU 的系统，架构为：
微信小程序 ←→ BLE ←→ ESP32 ←→ UART ←→ [我的 MCU 型号]

蓝牙模块：[如果已有，写上型号]
MCU UART 引脚：[TX=?, RX=?]
通信协议：[命令格式，如 \n 分隔的文本协议]
电机/大功率外设：[是否有时需要烧录时断开]

请按照「小程序 BLE ↔ MCU 通信链路搭建规范」生成三端代码，
要求每条链路都内置诊断日志，方便后续排查。
```
