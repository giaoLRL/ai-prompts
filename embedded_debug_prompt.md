# 嵌入式调试基础设施 AI 提示词

将此提示词粘贴给 AI，告诉它"帮我搭建调试基础设施"。

---

## 核心原则

1. **全局变量直接看** — 嵌入式通常没有 printf，全靠 JLink/调试器看内存，debug 结构体必须放在全局作用域 `volatile` 变量中
2. **分层捕获** — 应用层一个结构体看业务逻辑，驱动层一个结构体看硬件寄存器
3. **心跳信号** — 必须有一个 GPIO 翻转，用示波器/逻辑分析仪确认代码在跑
4. **零侵入** — debug 结构体独立文件，API 通过函数调用插入，不影响正常控制流
5. **一次改到位** — 加 debug 代码意味着重新编译+烧录，务必一次性添加足够的观测点

---

## 输出要求

请帮我创建以下文件（如果文件已存在则修改），并在主循环和关键驱动中插入调用点：

### 1. `debug.h` — 调试数据定义

#### 1.1 应用层调试结构体（`AppDebug`）

包含以下信息，用一个大结构体承载，所有字段 `volatile` 声明：

| 分类 | 必含字段 | 说明 |
|------|---------|------|
| 时间戳 | `uint32_t tick` | 主循环计数器，每轮自增 |
| 状态标志 | `uint8_t initialized / enabled / data_valid / fault_flag` | 各模块是否正常工作的布尔标志 |
| 传感器原始数据 | `float raw_accel_x/y/z`、`float raw_gyro_x/y/z` 等 | 驱动读回的原始值（未滤波未校准），用于判断传感器是否真的有数据 |
| 计算结果 | `float angle_pitch/roll/yaw` 等 | 滤波/融合后的最终值 |
| PID 内部量 | `float error / p_term / i_term / d_term / output` | 控制器每一步输出，用于判断 PID 是否发散/饱和/静默 |
| 执行器指令 | `int32_t motor_freq / motor_dir` 等 | 最终发给硬件的指令 |
| 校准参数 | `float bias_x/y/z` 等 | 零偏/标定值，用于确认校准是否通过 |

#### 1.2 环形缓冲区（可选但强烈推荐）

```c
#define DEBUG_BUF_SIZE  50  // 50 条快照，200Hz ≈ 250ms 历史
AppDebug g_debug_history[DEBUG_BUF_SIZE];
volatile uint32_t g_debug_index;
```

每条捕获时写入 `g_debug_history[g_debug_index % DEBUG_BUF_SIZE]`。这样即使断点暂停时当前值已被覆盖，历史里还有前几十条记录。

#### 1.3 心跳 GPIO

- 使用一个空闲 GPIO（如 PB0）
- 每 N 个主循环翻转一次（N 使频率为 1~2Hz）
- 提供 `debug_heartbeat()` 函数，在主循环末尾调用

#### 1.4 累积计数器

```c
volatile uint32_t g_debug_loop_count;       // 总循环次数
volatile uint32_t g_debug_read_fail_count;  // 读失败次数
volatile uint32_t g_debug_read_ok_count;    // 读成功次数
volatile uint32_t g_debug_timeout_count;    // 超时次数
```

这些累积量不会像单次快照那样被覆盖，适合长时间运行后查看。

---

### 2. 驱动层调试结构体（如 `I2CDebug`）

针对问题驱动单独定义。**不需要提前写死所有字段，但必须包含以下核心信息**：

| 必含字段 | 类型 | 说明 |
|---------|------|------|
| `fail_step` | `uint32_t` | 失败发生在流程的第几步（1/2/3...） |
| `fail_reason` | `uint32_t` | 失败原因码（1=超时, 2=状态错误, 3=数据校验失败...） |
| `status_value` | `uint32_t` | 失败时的硬件状态寄存器值（如 `DL_I2C_getControllerStatus()` 返回值） |
| `stat_raw` | `uint32_t` | 失败时的外设原始寄存器值（如 `I2C0->STAT` 的内存直接读） |
| `error_bit / idle_bit / ack_bit` | `uint32_t` | 关键状态位的独立解包值 |
| `total_fails` | `uint32_t` | 累计失败次数 |
| `last_reg` | `uint32_t` | 最后一次操作的外设寄存器地址 |
| `last_len` | `uint32_t` | 最后一次操作的数据长度 |

**关键原则：**
- 结构体放在**同一个源文件**（如 `mpu6050.c`）顶部，`volatile` 声明，避免跨文件依赖
- 每个失败路径都记录 `fail_step` 和 `fail_reason`，互不覆盖
- `stat_raw` 必须直接读外设寄存器地址（如 `(*(volatile uint32_t *)0x40080004)`），不要走 API 封装

---

### 3. `debug.c` — 实现

- `debug_init()`：清零所有结构体和计数器，初始化心跳 GPIO
- `debug_capture(...)`：接收当前帧所有关键数据，填充 `g_debug` 结构体，写入环形缓冲区
- `debug_heartbeat()`：心跳 GPIO 翻转逻辑

---

### 4. 插入点

在以下位置插入调试调用：

| 位置 | 调用 | 用途 |
|------|------|------|
| `main()` 初始化尾部 | `debug_init()` | 初始化调试基础设施 |
| 主循环末尾 | `debug_capture(...)` + `debug_heartbeat()` | 每帧捕获 |
| 驱动读失败路径 | `g_i2c_dbg.fail_step = N; g_i2c_dbg.fail_reason = R; g_i2c_dbg.status_value = ...; g_i2c_dbg.total_fails++;` | 精确记录失败点 |
| 驱动读成功路径 | `g_debug_read_ok_count++` | 比例分析 |

---

### 5. 设计约束

- **内存预算**：明确告知 AI 可用 SRAM 大小（如 MSPM0G3507 = 32KB），让 AI 计算结构体大小
- **`volatile` 关键字**：所有被中断/DMA/调试器异步访问的全局调试变量必须加 `volatile`
- **结构体对齐**：使用 `uint8_t padding[N]` 手动对齐到 4 字节边界，避免 JLink 显示错位
- **地址固定**：AI 应告知关键结构体的内存起始地址（从 `.map` 文件或调试器获取），方便 JLink 手动添加监控
- **O2 优化兼容**：用 `memset` 而非逐字段清零，用 `memcpy` 而非逐字段拷贝，避免编译器优化掉"无副作用的调试写"

---

## 使用示例

```
我这是一个 MSPM0G3507 的双轴云台项目，MPU6050 陀螺仪通过 I2C0 读取，
步进电机用 PWM 驱动。现在电机不响应陀螺仪数据，帮我搭建完整的调试基础设施，
包括应用层 DebugSnapshot、I2C 驱动层 I2CDebug、心跳 GPIO（PB0），
并指导我如何在 JLink 中查看这些变量。
```
