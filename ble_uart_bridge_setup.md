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

三端代码必须使用**同一个协议**，确保无论谁写的代码，换一个项目也能互通。

### 协议格式

```
帧分隔符：\n (换行符，0x0A)
编码：ASCII（纯文本，禁止二进制/JSON）
最大帧长：512 字节（超出丢弃并告警）
```

### 方向1：小程序 → MCU（命令）

每条命令以 `\n` 结尾，MCU 收到 `\n` 后立即解析并执行。

#### 标准命令集（所有项目必须实现）

| 命令 | 格式 | 说明 | 示例 |
|------|------|------|------|
| **PING** | `PING\n` | 心跳检测，MCU 收到后必须回复 `OK\n` | `PING\n` |
| **STOP** | `STOP\n` | 紧急停车，电机立即制动并保持停车状态，直到收到 GO | `STOP\n` |
| **GO** | `GO\n` | 解除停车，恢复正常运行 | `GO\n` |
| **RST** | `RST\n` | 软复位，重置所有参数到默认值并重新初始化 | `RST\n` |

#### 参数设置命令

| 命令 | 格式 | 说明 | 示例 |
|------|------|------|------|
| **SET** | `SET_参数名=值\n` | 设置单个参数值 | `SET_kp=0.35\n` |
| **CFG** | `CFG_参数名=值\n` | 同 SET，别名（兼容旧代码） | `CFG_speed=200\n` |

**参数命名规范**：使用小写+下划线，数值为整数或浮点数，禁止空格。例如：
- `SET_kp=0.35`（比例增益）
- `SET_ki=0.002`（积分增益）
- `SET_kd=0.03`（微分增益）
- `SET_base_speed=300`（基础速度）
- `SET_turn_speed=150`（转弯速度）

#### 查询命令

| 命令 | 格式 | 说明 |
|------|------|------|
| **GET** | `GET_参数名\n` | 查询单个参数，MCU 回复 `VAL_参数名=值\n` |
| **LIST** | `LIST\n` | 列出所有可调参数，MCU 逐行回复 `CFG_参数名=值\n` |

### 方向2：MCU → 小程序（遥测 + 响应）

#### 命令响应

| 响应 | 格式 | 说明 |
|------|------|------|
| 命令执行成功 | `OK\n` | PING/STOP/GO/SET 执行成功的通用回复 |
| 未知命令 | `ERR_UNKNOWN_CMD\n` | 收到的命令不在已知列表中 |
| 参数值查询 | `VAL_参数名=值\n` | 响应 GET 查询 |
| 参数列表 | `CFG_参数名1=值\nCFG_参数名2=值\n` | 响应 LIST 查询 |

#### 遥测/调试数据

MCU 定时（建议 5Hz = 200ms）发送一行状态数据：

```
STAT tick=<N> state=<S> error=<E> speed=<L>,<R> [自定义字段...]\n
```

固定字段：

| 字段 | 说明 |
|------|------|
| `tick` | 主循环计数器（uint32），用于判断 MCU 是否在运行 |
| `state` | 当前状态机状态码（0=待机, 1=运行, 2=转弯, 3=停车, -1=故障） |
| `error` | 当前控制误差值 |
| `speed` | 左右轮实际速度，逗号分隔 |

自定义字段按 `key=value` 追加，用空格分隔。

### 协议实现要求

**MCU 端**：
- 环形缓冲区累积 UART 数据，扫描 `\n` 切出完整帧
- 帧按第一个空格或等号前的 token 匹配命令（如 `STOP\n` → `STOP`, `SET_kp=0.35\n` → `SET`）
- 数值用 `atof()` 解析
- 发送 `OK\n` / `STAT\n` 等响应到 TX 环形缓冲区

**ESP32 端**：
- UART→BLE：收到 `\n` 后立即 flush 缓冲到 BLE
- BLE→UART：接收 BLE Write 后直接写 UART2（不解析不缓存），追加 `\n`（如果缺少）
- **透传 = 不解析协议内容**，只保证帧完整性

**小程序端**：
- BLE 接收按 `\n` 行缓冲拼接
- `STAT` 行解析为 key=value map，更新 UI
- `OK` / `ERR_*` 行用于反馈命令执行结果
- `LIST` 响应用于动态生成参数面板

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
| **透传原则** | **不解析、不修改**协议内容，只保证帧边界（按 `\n` 切帧） |

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
[  15.5] 📱→MCU [18B]: SET_kp=0.35                        ← 手机发来的每条命令
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

### 2. MCU 端（UART 收发 + 协议解析）

#### 2.1 UART 中断配置（导致命令静默丢失的根因）

**最常见的坑**：SysConfig/CubeMX 的默认值对短命令是致命的。

| 设置 | 错误默认值 | 正确值 | 原因 |
|------|-----------|--------|------|
| RX FIFO Threshold | `>= 1/2 full` (2字节) | `>= 1 byte` | `GO\n` 3 字节：前 2 字节触发中断 → `\n` 只剩 1 字节 → 不到阈值 → 永远卡死 |
| RX Timeout Counts | `0` (禁用) | `10` (~1ms) | 奇数字节的尾字节在无超时机制下永远不被清空 |

**强制措施**：在代码的 `UART_init()` 中显式覆盖这两个值为正确值，不依赖图形化配置工具的默认值。

#### 2.2 环形缓冲区 + 协议解析

```c
// 接收侧
#define RX_BUF_SIZE 512
uint8_t rx_buf[RX_BUF_SIZE];
volatile uint16_t rx_head = 0;  // ISR 写
uint16_t rx_tail = 0;           // 主循环读

// ISR：只写缓冲区
void UART_IRQHandler() {
    while (UART_有数据) {
        rx_buf[rx_head] = UART_读字节();
        rx_head = (rx_head + 1) % RX_BUF_SIZE;
    }
}

// 主循环 poll()：扫描 \n 切帧 → 解析命令 → 执行 → 回复 OK/ERR/STAT
void poll() {
    while (rx_tail != rx_head) {
        char c = rx_buf[rx_tail];
        rx_tail = (rx_tail + 1) % RX_BUF_SIZE;
        line_buf[len++] = c;
        if (c == '\n' || len >= 128) {
            parse_and_execute(line_buf, len);
            len = 0;
        }
    }
}
```

- 发送用 `send_str()` 非阻塞化：TX FIFO 满时保存断点返回，`poll()` 中续传

#### 2.3 必须实现的协议处理

| 命令 | 处理逻辑 |
|------|---------|
| `PING\n` | 回复 `OK\n` |
| `STOP\n` | 电机制动 + 设置持久停车标志 + 回复 `OK\n` |
| `GO\n` | 清除停车标志 + 回复 `OK\n` |
| `RST\n` | 恢复默认参数 + 重新初始化 + 回复 `OK\n` |
| `SET_<key>=<val>\n` | 更新参数 + 回复 `OK\n` |
| `CFG_<key>=<val>\n` | 同 SET |
| `GET_<key>\n` | 回复 `VAL_<key>=<val>\n` |
| `LIST\n` | 逐行回复参数键值 |

#### 2.4 诊断建议

- `g_debug_cmd_count`：累积解析命令数
- `g_debug_uart_rx_bytes`：累积 UART RX 字节数
- ESP32 日志 `📱→MCU 100B` 对比 MCU 端 `g_debug_uart_rx_bytes = 100` → 物理链路 OK
- ESP32 日志有数据但 `g_debug_cmd_count = 0` → UART 中断/解析有问题

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
| **协议解析** | 收到数据按 `\n` 切帧，`STAT` 行更新波形/仪表盘，`OK` 行确认命令，`ERR_*` 行 toasting |
| **参数面板** | 连接成功后发 `LIST\n` → 动态生成滑块/开关/输入框（不需要硬编码参数名） |

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
[BLE] 📡 收到: STAT tick=1024 state=1 error=12 speed=280,270
[BLE] 📡 收到: OK                              ← 命令执行确认
```

#### 3.3 防坑清单

| 坑 | 预防 |
|----|------|
| `name` 未定义 → 连接崩溃 | 所有回调 closure 中确保所有使用的变量已声明 |
| `connectBLE` 先扫描后注册监听 → 竞态丢包 | 先注册 `onBLEConnectionStateChange`，再 `startBluetoothDevicesDiscovery` |
| `charNotify` 未初始化 → 监听器重复注册 | 初始化为 `false`，注册后置 `true` |
| `sendCmd` 静默失败 | 必须有 `fail` 回调 → toast + 清理连接 |
| 数据分片导致解析错误 | 行缓冲按 `\n` 拼接，超 512 字节丢弃并告警 |
| `c.properties.notify` 不可靠 | 不依赖此字段，只凭 UUID 匹配就尝试启用 notify |

---

## 三端数据流对照表（调试验证用）

搭建完成后，按以下顺序逐段验证数据流：

| 步骤 | 操作 | 小程序端 | ESP32 日志 | MCU 端 | 通过标志 |
|------|------|---------|-----------|--------|---------|
| 1 | ESP32 上电 | — | `🟢 广播中` | — | 看到广播日志 |
| 2 | 小程序连接 | `✅ 已连接` | `✅ 已连接` | — | 两端都显示已连接 |
| 3 | 小程序启通知 | `✅ Notify 启用成功` | `🔍 CCCD 写入!` | — | ESP32 收到 CCCD 写入 |
| 4 | 小程序发 PING | `📡 收到: OK` | `📱→MCU [5B]: PING` | `g_debug_cmd_count++` | 小程序收到 OK |
| 5 | 小程序发 LIST | `📡 收到多行 CFG_*` | `📱→MCU [5B]: LIST` | 逐行回复参数 | 动态生成参数面板 |
| 6 | 滑块改参数 | `📡 收到: OK` | `📱→MCU [13B]: SET_kp=0.35` | `g_debug_cmd_count++` | 小程序收 OK + ESP32 显示命令 |
| 7 | MCU 发送遥测 | `STAT` 行更新 UI | `🔵 BLE→APP > 0` | `g_debug_uart_tx_bytes > 0` | ESP32 BLE→APP 字节数 = MCU TX 字节数 |

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
       → 命令解析逻辑是否正确？（用调试器看 line_buf 内容）
```

---

## 使用方式

```
我要搭建一个小程序控制 MCU 的系统，架构为：
微信小程序 ←→ BLE ←→ ESP32 ←→ UART ←→ [我的 MCU 型号]

MCU UART 引脚：[TX=?, RX=?]
控制对象：[电机/舵机/电磁铁等]
需要在小程序上调节的参数：[kp, ki, kd, 速度, ...]
需要在小程序上显示的数据：[速度曲线, 传感器值, 状态, ...]

请按照「小程序 BLE ↔ MCU 通信链路搭建规范」生成三端代码，
包括标准协议实现和诊断日志。
```
