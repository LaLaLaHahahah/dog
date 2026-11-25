# ESP32-S3 四足互动机器人 需求规格说明书（REQUIREMENTS.md）

| 项 | 内容 |
|---|---|
| 项目名称 | ESP32-S3 Quadruped Robot |
| 文档版本 | V1.1 |
| 开发阶段 | 第一版基础固件（P0–P6） |
| 目标平台 | ESP32-S3 N16R8 |
| 开发语言 | C |
| 开发框架 | ESP-IDF 5.x + FreeRTOS |
| 文档性质 | 需求规格 + 架构约束（不含具体实现） |

> 本文件是需求与架构约束的冻结基线。凡标注 **【待确认】** 或 **【建议值，待确认】** 的内容，均带"关闭阶段 + 关闭判据"（见 §9），未关闭前对应的验收条款不生效。§7.2 / §7.3 为**已冻结的架构约束**，变更须走需求变更。

---

## 0. 已确认决策

| # | 议题 | 结论 |
|---|---|---|
| D1 | 文档范围 | 覆盖整个 V1.0（P0–P6） |
| D2 | 动作中断 | 立即打断当前动作，从当前指令位置平滑切换到新动作；紧急停止优先级最高 |
| D3 | 平滑/速度验收 | 由分析师提出客观数值阈值，作为【建议值，待确认】 |
| D4 | 步态定位（原 M1） | "前进/后退/转向"保留为 V1.0 目标，但增加 **P0 实物可行性准入实验**作为 P5 范围的准入闸门 |
| D5 | 速度参数（原 M2） | 删除 `motion_smoothness`；`motion_speed` 定义为 0–1 全局缩放系数（默认 1.0）；限速是 Servo Controller 的强制职责 |
| D6 | 安全保护（原 M4） | 新增与配置无关的**绝对脉宽硬保护**（建议 500–2500µs）与**校准模式** |

---

## 1. 项目背景

### 1.1 项目目标

基于 ESP32-S3 开发一台小型四足互动机器人。第一版使用四个 MG90S 180° 位置舵机，通过 PCA9685 输出 PWM 信号，实现四足运动、动作交互与表情显示。

除运动能力外，第一版要求具备良好的软件架构，便于后续增加动作、步态算法、交互方式与硬件模块。项目采用模块化分层设计，将硬件驱动、舵机控制、动作执行、步态算法、行为管理与表情系统相互解耦。

### 1.2 当前工程状态（已核实）

- 代码仓库 `/Users/mac/IdeaProjects/dog` 当前基本为空：仅有 5 字节 `README.md` 与 `.idea/`，无 ESP-IDF 工程骨架（无 `sdkconfig`、`CMakeLists.txt`、`main/`）。
- 结论：**从零搭建（greenfield）**，P0 需先完成 ESP-IDF 工程初始化。

### 1.3 硬件 BOM 与到位状态

| 配件 | 数量 | 说明 | 状态 |
|---|---|---|---|
| ESP32-S3 N16R8 | 1 | 主控 | 已有 |
| PCA9685 | 1 | 16 路 PWM，第一版用 4 路 | 待确认 |
| MG90S 180° | 4 | 位置控制舵机 | 待确认 |
| 2S 电池 | 1 | 主电源 | 已有 |
| 5V 降压模块 | 1 | 舵机供电，建议持续 ≥5A | 待确认 |
| 0.42 英寸小屏幕 | 1 | 表情显示，型号/接口待确认 | 待确认 |
| 电源开关 | 1 | 电源控制 | 待确认 |
| 连接线 | 若干 | 电源及信号连接 | 待确认 |

### 1.4 硬件连接原则（约束）

1. ESP32-S3 通过 I2C 控制 PCA9685；PCA9685 输出四路 PWM 控制四个 MG90S。
2. 舵机独立 5V 降压供电，与逻辑供电分开；ESP32、PCA9685、舵机电源**必须共地**。
3. 舵机供电端预留足够滤波电容。
4. 屏幕接口与驱动方式依实际型号确定，但**型号与总线归属必须在 P0 关闭**（见 §7.4、§9）。

> 舵机供电电流须按**多舵机同时启动与堵转**工况估算，不能仅按正常运行电流估算。

---

## 2. 功能目标

### 2.1 第一版核心功能

**必须档（V1.0 无条件实现）：**
1. 四个舵机独立控制。
2. 舵机零点校准、方向调整、角度限制。
3. 舵机参数持久化保存（NVS）。
4. 基础姿态控制：站立、趴下等。
5. 基础动作：招手、点头、摇摆等。
6. 动作序列执行与平滑插值。
7. 基础四足步态算法——**仅限周期性运动**（原地踏步/重心摆动/单腿试探摆动）。
8. 0.42 英寸小屏幕显示表情。
9. 串口调试控制台。
10. 为未来增加 360° 连续旋转舵机预留扩展能力（属架构约束，见 §7.8）。

**条件实现档（受 P0 准入实验闸门约束）：**
- 简单前进、简单后退、左右转向。

> 上述三项因 1-DOF 腿结构无法保证产生净位移（见 §4.5 限制），保留为目标但须先通过 §4.5 的**实物可行性准入实验**；准入结论决定 P5 范围，未通过则该项作废并记录，V1.0 定位收缩为"四足姿态互动机器人"。

### 2.2 长期扩展方向（**不属于 V1.0 必须范围**）

360° 连续旋转舵机；更复杂步态算法；翻跟斗、跳跃等动作；WiFi 远程控制；Web 参数配置界面；ESP32 无线遥控器；麦克风/扬声器/语音交互；接入大语言模型；按状态自动组合动作与表情。

---

## 3. 功能清单

| 模块 | 职责 | 阶段 |
|---|---|---|
| 硬件驱动（I2C / PCA9685 / Servo / GPIO） | I2C 总线、PWM 输出、舵机物理信号 | P0 |
| Servo Controller | 目标角度→限幅→限速→方向/偏移→PWM，唯一控制与限速入口 | P0–P1 |
| 校准（Calibration + Calibration Mode） | 中位/方向/偏移/限幅/脉宽；校准模式下的安全探索 | P1 |
| 存储（Config Store） | 配置加载/保存/复位，CRC+版本，默认值兜底 | P1 |
| 紧急停止（ESTOP） | 全局立即停止、失能、锁定 | P0/P2（接口贯穿） |
| Action Engine | 动作序列、时间轴、插值、速度限制、状态机 | P2 |
| Pose Controller | 姿态定义与切换（经 Action Engine 平滑执行） | P3 |
| Expression | 屏幕驱动、表情渲染、眨眼、视线 | P4 |
| Gait Engine | 周期步态（正弦波起步）、相位/步频/步幅 | P5 |
| Behavior Manager | 行为决策，组合动作/姿态/表情 | P6 |
| Serial Console | 命令解析、参数校验、状态查询、校准模式入口 | P0–P6 |
| 扩展预留（ServoMode / 关节数） | POSITION / VELOCITY 枚举、`ROBOT_JOINT_COUNT` | 架构层 |

---

## 4. 具体行为描述

### 4.1 舵机控制系统（Servo Controller）

**职责**：全项目唯一将"目标角度"转换为"PWM 输出"的模块，也是**唯一的限速点**。业务层、动作层、步态层均不得绕过它直接操作 PCA9685 或 PWM 数值。

**关键行为**：
1. 每个舵机拥有独立配置 `ServoConfig`（见 §5.1）。
2. 目标角度在**逻辑域**限幅（夹在 `min_angle` 与 `max_angle` 之间）。
3. 方向（`direction` ∈ {+1,−1}）与零点偏移（`offset`）由本层统一处理；`direction` 只作用于增量，不参与限幅比较。
4. 角度→PWM 映射按 §4.1 的**冻结公式**执行。
5. **每 tick 增量限速**：单次更新角度变化量 ≤ `max_speed × motion_speed × tick`，`max_speed` 为单舵机硬上限，`motion_speed` 为全局 0–1 缩放系数（D5）。
6. 输出脉宽在进入硬件前一律夹取到**绝对硬保护区间**（§4.1、D6），该区间与配置无关、不可通过串口修改。
7. 任何进入映射函数的数值在做算术前必须先通过 `isfinite` 校验（NaN/Inf 一律拒绝，保持原状态）。
8. 支持单个舵机使能/失能（语义已冻结，见 §5.1）。

**冻结的角度→脉宽映射公式**（M3）：

```
pulse_center_us = (min_pulse_us + max_pulse_us) / 2
us_per_deg      = (max_pulse_us - min_pulse_us) / (max_angle - min_angle)
a               = clamp(angle, min_angle, max_angle)        // 逻辑域限幅
pulse_raw       = pulse_center_us + direction * (a - center_angle + offset) * us_per_deg
pulse           = clamp(pulse_raw, HARD_MIN_US, HARD_MAX_US) // 绝对硬保护（D6）
```

配置加载时必须校验（不满足则回退默认并记日志）：
- `min_angle < max_angle`
- `min_pulse_us < max_pulse_us`
- `min_angle ≤ center_angle + offset ≤ max_angle`
- `direction ∈ {+1, −1}`
- 所有浮点字段 `isfinite`

> 公式的**端点标定值与符号约定**在 P1 用实物确认，但**结构与限幅域自本版冻结**。

**绝对脉宽硬保护**（D6）：`HARD_MIN_US` / `HARD_MAX_US` 为代码级常量，建议 500µs / 2500µs【建议值，待实测】，不受任何配置影响、不可串口修改，越界即记日志并夹取。任何情况都不得输出区间外的脉宽。

### 4.2 校准功能（Calibration）

支持：修改单个舵机机械中位、方向、零点偏移、最大/最小角度、PWM 脉宽范围；保存；重启自动恢复（NVS）。

**校准模式（Calibration Mode）**（D6，解决"限幅下无法探索限幅"的矛盾）：
1. `calib_enter(id)`：进入后解除该舵机**逻辑限幅**；`max_speed` 降为标定值（建议 30°/s）；单次步进固定（建议 1°）。
2. `calib_step(id, ±deg)`：按固定步进缓慢移动，用于观察机械干涉/堵转并记录限位。
3. `calib_exit()`：退出校准模式后**立即回到安全姿态**（回中位或默认姿态，经正常限速）。
4. 校准模式期间仅操作被校准舵机，其余舵机保持安全姿态。
5. 校准模式优先级 = MANUAL（见 §4.9 所有权）。

**状态变化**：校准修改 → 可 `save`（写 NVS）或丢弃；`load` 恢复已存；`reset` 恢复默认。

### 4.3 姿态控制系统（Pose Controller）

姿态代表机器人某个稳定的目标状态。**Pose 是"最终要保持的状态"，Action 是"从一个状态运动到另一个状态的过程"。**

第一版提供：`pose_stand()`、`pose_sit()`、`pose_crouch()`、`pose_lie_down()`、`pose_neutral()`。

行为：
- 定义姿态目标角度。
- 姿态切换**不直接持有通道**，而是把目标交给 Action Engine 平滑执行（生产者=ACTION，见 §4.9）。
- 示例流程：站立 → 招手 → 恢复站立。

> 注：`sit`/`crouch`/`lie_down` 在 1-DOF 结构下可能退化为同一物理姿态，是否合并待机械结构确定（§9）。

### 4.4 动作引擎（Action Engine）

**职责**：以"动作数据"驱动舵机，而非在代码里写延时与舵机控制语句。

**动作执行模型**：
- 非阻塞执行（禁止阻塞等待）。
- 固定周期更新（20ms，Control Task 内执行）。
- 平滑插值；支持循环、暂停、恢复、中断、切换、速度调整；为更多舵机预留扩展。

**每次更新（20ms tick）流程**：
1. 获取当前动作；2. 获取时间进度；3. 计算当前帧；4. 角度插值；5. 应用速度限制（由 Servo Controller 统一执行）；6. 提交目标位置；7. 检查是否进入下一帧；8. 检查动作是否结束。

**插值算法（第一版）**：LINEAR、EASE_IN、EASE_OUT、EASE_IN_OUT、SMOOTHSTEP。**easing 为帧级**（`ActionFrame.ease`），插值连续，避免跳变。

**动作数据语义（已冻结，M8）**：
- `duration_ms` = 从**上一帧角度**过渡到**本帧角度**所用时间；首帧从当前指令位置过渡（时长由该帧给出）；末帧到达后**保持**。
- 校验：`duration_ms ≥ 20`、`frame_count ≥ 1`，违反则 `action_play` 返回 `ESP_ERR_INVALID_ARG` 且不改变当前状态。
- 序列必须指向**静态/常量存储**（`const`），调用方不得传栈上临时序列。
- 非循环动作自然结束 → 进入 IDLE 态，舵机保持在末帧角度；是否回姿态由行为层决定。

**动作状态机（已冻结，M6）**：

| 当前态 | 事件 | 下一态 | 行为 |
|---|---|---|---|
| IDLE | `action_play` | RUNNING | 从当前指令位置过渡到首帧 |
| RUNNING | `action_play`(新) | RUNNING | 立即打断，从当前位置平滑过渡新动作（D2） |
| RUNNING | `action_pause` | PAUSED | 冻结在当前位置 |
| RUNNING | `action_stop` | IDLE | 停止（末帧保持，回姿态由行为层决定） |
| RUNNING | `robot_estop` | ESTOP | 立即终止、清队列、失能、锁定 |
| PAUSED | `action_resume` | RUNNING | 从暂停处继续 |
| PAUSED | `action_play`(新) | RUNNING | **丢弃暂停上下文**，从当前位置过渡新动作 |
| PAUSED | `action_stop` | IDLE | 丢弃暂停上下文，停止 |
| PAUSED | `robot_estop` | ESTOP | 同 RUNNING |
| ESTOP | `robot_estop_clear` | IDLE | 解锁，经限速恢复到默认姿态 |
| ESTOP | 其余动作事件 | ESTOP（拒绝） | 返回 `ESP_ERR_INVALID_STATE` |
| RUNNING(loop) | 完成一轮 | RUNNING | 回到首帧继续；`loop=false` 则自然结束→IDLE |

**动作中断/切换（D2）**：新动作立即打断，从**当前指令位置**（`last commanded angle`，非假定真实角度，见 §5.1）平滑过渡，不回零跳变。**命令去抖**：连续高频打断时，单次切换至少持续一段最小过渡时间（建议 ≥100ms【待确认】），防止振荡。

### 4.5 步态系统（Gait Engine）

**职责**：持续周期性运动，独立于 Action Engine（生产者=GAIT，见 §4.9）。

**第一版目标（必须档）**：原地踏步；重心摆动；单腿试探摆动；调整步频/步幅/四腿相位差。

**条件档（受准入闸门约束）**：简单前进、简单后退、左右转向。

**初步算法**：正弦波步态起步：`angle[i] = offset[i] + amplitude * sinf(phase[i])`；后续演进为支撑相（Stance）/摆动相（Swing）模型。`swing_ratio` 字段在当前纯正弦算法下无消费者，**随支撑/摆动相模型引入时再启用**。

**实物可行性准入实验（D4）**：P0 阶段先用实物完成最小实验——手动驱动四腿正弦摆动，观测机体在**限定地面材质**（建议地毯/硬地面各测一次）上是否产生稳定、方向可重复的净位移。结论作为 P5 范围准入条件：
- 通过 → P5 实现前进/后退/转向，判定标准为"X 秒内产生 ≥Y cm 位移且方向可重复"（X/Y 在准入实验时定义）。
- 未通过 → 该项作废并记录，V1.0 收缩为姿态互动机器人。

**重要限制**：第一版每条腿只有一个自由度。因此：
- 不能直接套用常规三自由度四足运动学模型；
- 不能假设具备完整抬腿/前后摆腿能力；
- 步态算法须按实际机械结构设计；
- 优先验证简单动作，再尝试连续行走；
- 暂不引入逆运动学。

**输出**：四个舵机目标位置，交 Servo Controller 执行。

### 4.6 表情系统（Expression）

使用 0.42 英寸小屏幕显示眼睛与表情。独立运行，屏幕刷新不得阻塞运动控制。

**第一版表情（枚举）**：NORMAL、HAPPY、SAD、ANGRY、SLEEPY、SURPRISED。

**行为**：切换表情；自动眨眼（间隔可配）；眼睛左右看；后续支持简单动画。

**原则**：屏幕驱动与表情逻辑分离；第一版不使用 LVGL；优先简单绘图函数或预定义位图；`EmotionType` 为 6 种情绪，"自动眨眼""左右看"属动画/视线行为，实现上区分。

### 4.7 行为管理系统（Behavior Manager）

**职责**：决定机器人"该做什么"。不直接控制舵机，而是调用动作、姿态、表情模块（生产者关系见 §4.9）。

第一版预留：`BEHAVIOR_IDLE`、`BEHAVIOR_WALK`、`BEHAVIOR_WAVE`、`BEHAVIOR_DANCE`。

行为示例：
- **待机（IDLE）**：保持默认姿态；每 N 秒（可配）在 6 种表情间按**固定循环或带种子伪随机**切换（不做不可复现的"随机"）；定时眨眼。
- **招手（WAVE）**：执行招手动作，切换开心表情，动作结束恢复默认姿态。
- **走路（WALK）**：启动步态引擎，切换对应表情，停止时平滑恢复站立姿态。
- **跳舞（DANCE）**：预留，需在 P6 前定义其动作组合内容（当前动作库仅招手/点头/摇摆）。

### 4.8 串口调试控制台（Serial Console）

机械调试阶段的重要工具。支持命令解析；参数错误返回明确提示；输出当前舵机配置、动作状态、步态参数；查看与修改配置；修改后可选保存。第一版不做 Web 管理界面。

**命令表（已冻结，S1）**：

| 命令 | 参数 | 范围/示例 | 说明 |
|---|---|---|---|
| `servo <id> <angle>` | id 0–3 | `servo 1 90` | 单舵机目标（生产者=MANUAL） |
| `servo all <angle>` | | `servo all 90` | 四舵机目标 |
| `offset <id> <deg>` | | `offset 0 5` | 零点偏移 |
| `direction <id> <±1>` | ±1 | `direction 1 -1` | 方向 |
| `limit <id> <min> <max>` | | `limit 0 20 160` | 限幅 |
| `calib enter\|step <±deg>\|exit` | | `calib enter` | 校准模式（D6） |
| `save` / `load` / `reset` | | `save` | 配置存储 |
| `action <name>` / `action stop` | | `action wave` | 动作 |
| `pose <name>` | stand/sit/crouch/lie/neutral | `pose stand` | 姿态 |
| `gait start\|stop\|speed <v>\|amplitude <deg>\|phase <i> <p>` | | `gait start` | 步态 |
| `emotion <type>` | happy/sad/angry/sleepy/surprised/normal | `emotion happy` | 表情 |
| `estop` / `estop clear` | | `estop` | 紧急停止/解除 |
| `status` / `config` | | `status` | 查询状态/配置 |

**约定**：成功回显 `OK`；参数错误回显 `ERR: <原因>`（如 `ERR: id out of range`、`ERR: unknown command`、`ERR: invalid state`），且不改变系统状态。

**物理通道（【待确认】，P0 关闭）**：ESP32-S3 的 USB Serial 可能是原生 USB-CDC（USB-Serial-JTAG）或 UART0 经桥接芯片，二者初始化、重启枚举、日志通道不同。需在 P0 前确定；建议 USB-CDC 用于交互 + UART0 保留日志（或反之，须写清）。

### 4.9 舵机通道所有权（Servo Ownership，已冻结，M7）

任一时刻**至多一个运动生产者**持有舵机通道组：

| 生产者 | 来源 | 说明 |
|---|---|---|
| MANUAL | 校准模式 / 串口 `servo` 直调 | 人工控制 |
| ACTION | 动作序列；姿态切换经 Action Engine 提交 | 动作与姿态 |
| GAIT | 步态引擎 | 持续周期运动 |
| ESTOP | 紧急停止 | 全局，最高 |

**抢占优先级**：`ESTOP > MANUAL > ACTION > GAIT`。

**所有权交接**：切换生产者时，先完成"交接"——从当前指令位置平滑过渡到新生产者的首帧，不跳变。**交接失败**（过渡超时，建议 1000ms【待确认】）→ 回退到当前安全姿态并记日志。

> 行为层（Behavior）是决策层，不是生产者：它通过调用 action/pose/gait 触发对应生产者。示例：GAIT 行走中收到 `pose_stand()`（生产者=ACTION，优先级更高）→ 抢占并平滑过渡到站立。

---

## 5. 输入输出定义

### 5.1 Servo Controller

```c
esp_err_t servo_set_angle(uint8_t id, float angle);   // 写内存目标（两段式）
esp_err_t servo_set_all(float a0, float a1, float a2, float a3); // 四通道同 tick 生效
esp_err_t servo_disable(uint8_t id);
esp_err_t servo_enable(uint8_t id);
esp_err_t servo_get_current(uint8_t id, float *out);   // 读当前指令位置
```

| 接口 | 输入 | 输出 | 状态变化 / 语义 |
|---|---|---|---|
| `servo_set_angle` | `id`、逻辑角度 | `esp_err_t`=是否接受 | 写内存目标；由 Control Task 下一 tick 统一"限速→输出" |
| `servo_set_all` | 四个逻辑角度 | `esp_err_t` | 四通道目标**同一 tick 内生效**（不允许跨 tick 撕裂） |
| `servo_disable` | `id` | `esp_err_t` | 该通道输出"无脉冲（off）"，舵机**释放保持力矩**（语义已冻结） |
| `servo_enable` | `id` | `esp_err_t` | 恢复输出，经限速过渡到当前目标 |
| `servo_get_current` | `id` | `esp_err_t` + `*out` | 返回**当前指令位置（last commanded angle）** |

> "当前指令位置"是系统维护的每路最新目标角，**不假设真实机械角度**（MG90S 无反馈）。D2 的过渡起点、§4.4 首帧起点均取该值。校准后机械回差、断电被手动掰动会造成该值与现实脱节，系统不感知。

**ServoConfig 字段**：

| 字段 | 含义 |
|---|---|
| `channel` | PCA9685 通道 |
| `center_angle` | 机械中位 |
| `offset` | 零点偏移 |
| `direction` | 运动方向，1 或 -1 |
| `min_angle` / `max_angle` | 逻辑域最小/最大允许角度 |
| `max_speed` | 单舵机最大运动速度（°/s，硬上限） |
| `min_pulse_us` / `max_pulse_us` | 最小/最大 PWM 脉宽（微秒） |

### 5.2 Action Engine

```c
typedef struct {
    uint16_t duration_ms;   // 从上一帧过渡到本帧的时长
    float angle[ROBOT_JOINT_COUNT];
    uint8_t ease;           // 帧级插值类型（LINEAR/EASE_IN/.../SMOOTHSTEP）
} ActionFrame;

typedef struct {
    const char *name;
    const ActionFrame *frames;   // 必须指向静态/常量存储
    uint16_t frame_count;        // ≥1
    bool loop;
} ActionSequence;

esp_err_t action_play(const ActionSequence *action);
esp_err_t action_stop(void);
esp_err_t action_pause(void);
esp_err_t action_resume(void);
```

| 接口 | 输入 | 输出 | 状态变化 |
|---|---|---|---|
| `action_play` | 动作序列指针 | `esp_err_t` | 见 §4.4 状态机；`frame_count=0`/`duration<20` 返回 `ESP_ERR_INVALID_ARG` 且不变态 |
| `action_stop` | 无 | `esp_err_t` | RUNNING/PAUSED → IDLE |
| `action_pause` | 无 | `esp_err_t` | RUNNING → PAUSED |
| `action_resume` | 无 | `esp_err_t` | 仅 PAUSED 有效，否则 `ESP_ERR_INVALID_STATE` |

### 5.3 紧急停止（ESTOP，M6）

```c
esp_err_t robot_estop(void);        // 任意任务可调，最高优先级
esp_err_t robot_estop_clear(void);  // 仅 ESTOP 态有效
```

**语义（已冻结）**：立即终止所有 Action/Gait；清空命令队列；所有舵机失能（释放力矩）；锁定在 ESTOP 态直到显式 `robot_estop_clear()`，其后经限速恢复到默认姿态。

> estop 输出策略默认"失能（释放力矩）"。若产品希望 estop 时保持姿态（hold），此为可配置项，默认失能——该选择在 P0/P1 前确认（§9）。

### 5.4 姿态 / 步态 / 表情 / 行为 / 存储

```c
esp_err_t pose_stand(void); esp_err_t pose_sit(void); esp_err_t pose_crouch(void);
esp_err_t pose_lie_down(void); esp_err_t pose_neutral(void);

esp_err_t gait_start(const GaitConfig *cfg);
esp_err_t gait_stop(void);
esp_err_t gait_set_frequency(float f); esp_err_t gait_set_amplitude(float a);
esp_err_t gait_set_phase(uint8_t leg, float p);

esp_err_t expression_set(EmotionType t);
esp_err_t expression_blink(void);
esp_err_t expression_look_left(void); esp_err_t expression_look_right(void);

esp_err_t behavior_set(BehaviorType b);

esp_err_t robot_config_load(void);
esp_err_t robot_config_save(void);   // 唯一触发写 NVS 的入口
esp_err_t robot_config_reset(void);
```

```c
typedef struct {
    ServoConfig servos[ROBOT_JOINT_COUNT];
    GaitConfig   gait;
    float        motion_speed;      // 0–1 全局缩放（D5，默认 1.0）
    uint16_t     blink_interval_ms;
    uint8_t      display_brightness; // 依屏幕型号确认保留或删除（§9）
} RobotConfig;

typedef struct {
    float frequency;      // 步频
    float amplitude;      // 步幅（角度幅度）
    float phase[ROBOT_JOINT_COUNT];
    float offset[ROBOT_JOINT_COUNT];
    float swing_ratio;    // 预留：当前正弦算法不消费，随支撑/摆动相模型启用
} GaitConfig;
```

### 5.5 调用权限矩阵（M5，已冻结）

| API | 允许调用任务 | 执行模型 | 返回值含义 | 异常行为 |
|---|---|---|---|---|
| `servo_set_angle` / `servo_set_all` | 任意（加锁） | 同步写内存目标，立即返回 | 请求是否被接受 | 参数非法→拒绝并保持原状态 |
| `servo_disable` / `servo_enable` | 任意 | 同步（内存标记） | 同上 | — |
| `servo_get_current` | 任意 | 同步读 | 状态 + 出参 | — |
| `action_*` | Behavior / Input | 同步（更新动作引擎状态） | 同上 | 按 §4.4 状态机 |
| `pose_*` | Behavior / Input | 同步（提交姿态目标经 Action） | 同上 | 受所有权抢占 |
| `gait_*` | Behavior / Input | 同步 | 同上 | 受所有权抢占 |
| `expression_*` | Behavior / Input | 同步（更新表情状态，渲染异步） | 同上 | — |
| `robot_estop` / `robot_estop_clear` | 任意（最高优先级） | 同步，立即生效 | 同上 | 唯一可跨所有生产者抢占 |
| `robot_config_*` | 任意（低频） | 同步 | 同上 | NVS 错误 → 回退默认 |

**两段式输出模型（已冻结）**：`servo_set_angle`/`servo_set_all` 只写内存目标；**只有 Control Task** 在每个 tick 内统一执行"限速→输出"。硬件错误（I2C NACK/超时）通过事件通知上报，不通过 `servo_set_*` 的返回值同步传递。

---

## 6. 异常与边界条件

| # | 场景 | 期望行为 |
|---|---|---|
| E1 | 目标角度超出 `min_angle`/`max_angle` | 逻辑域限幅（先于 offset 应用，见 §4.1 公式），不得越界 |
| E2 | 脉宽超出绝对硬保护区间 | 夹取到 `[HARD_MIN_US, HARD_MAX_US]` 并记日志（区间与配置无关） |
| E3 | 多舵机同时启动/堵转 | 硬件按堵转工况设计（≥5A + 滤波）；堵转实测电流与母线压降作为硬件验收（§8.0） |
| E4 | NVS 首次启动 | 自动加载默认配置，正常运行 |
| E5 | NVS 数据损坏 / CRC 失败 / 版本不匹配 | 回退默认配置并记日志，不崩溃 |
| E6 | 串口命令参数缺失/越界/未知命令 | 回显 `ERR: <原因>`，不改变系统状态 |
| E7 | 动作执行中收到新动作/命令 | 立即打断并平滑切换（D2）；受所有权与状态机约束 |
| E8 | 循环动作 | 无限循环直至 stop/estop/被抢占；非循环自然结束→IDLE |
| E9 | 插值导致单 tick 跳变 | Servo Controller 限速：增量 ≤ `max_speed×motion_speed×tick` |
| E10 | 屏幕刷新阻塞控制 | 禁止；见 §7.4 总线分配与分片约束 |
| E11 | 舵机失能/使能 | 已冻结：失能=无脉冲/释放力矩；使能=恢复并经限速过渡 |
| E12 | 配置升级（新增字段） | 结构带版本号 + CRC；加载不匹配/CRC 失败→回退默认（§7.7） |
| E13 | `direction` 非法 / 配置区间自相矛盾 | 参数校验拒绝，回退默认并记日志 |
| E14 | 参数校验失败 | 返回 `esp_err_t` 错误码，不改变状态 |
| E15 | PCA9685 上电探测失败（I2C NACK/地址不符） | 进入**无舵机安全模式**：不输出任何 PWM，串口持续报错，可重试初始化 |
| E16 | I2C 传输超时 / 总线锁死（SDA 拉低） | 触发总线恢复时序，重试计数递增；连续失败进入 E15 |
| E17 | 输入 `angle = NaN/Inf`、`id ≥ ROBOT_JOINT_COUNT` | `isfinite`/范围校验，拒绝并保持原状态 |
| E18 | 配置 `min_angle>max_angle`、`min_pulse>max_pulse`、`center+offset` 越界 | 加载校验失败→回退默认 |

---

## 7. 技术约束

### 7.1 技术选型（必须遵守）

| 项 | 约束 |
|---|---|
| 主控 | ESP32-S3 N16R8 |
| 框架 | ESP-IDF 5.x，**不使用 Arduino 框架** |
| 语言 | C |
| RTOS | ESP-IDF 自带 FreeRTOS，不引入额外 RTOS |
| PWM 驱动 | PCA9685（I2C） |
| 存储 | ESP-IDF NVS（第一版不用 SD 卡/文件系统） |
| 屏幕驱动 | 依实际型号确定 |
| 调试 | USB Serial（通道待定，见 §4.8） |
| 数学 | C 标准数学库 |

选型原则：硬件驱动尽量用官方 API/成熟驱动；核心动作系统自行实现；第一版**不引入** ROS、LVGL 等框架；尽量少第三方依赖。**V1.0 动作数据以 `const` 表存 Flash，新增动作需重新编译烧写**（运行期可编辑动作需 P7+ 引入文件系统/网络下发）。

### 7.2 架构分层约束（已冻结）

- 行为层不直接操作舵机；动作层不直接操作 PCA9685；步态算法不直接操作硬件。
- 所有舵机目标位置统一经过 Servo Controller。
- **只有一个控制任务**负责更新舵机输出（两段式，见 §5.5）。
- 表情系统独立运行，不阻塞运动控制。
- 硬件驱动与业务逻辑相互独立。

### 7.3 FreeRTOS 任务划分约束（已冻结）

| 任务 | 周期 | 职责 | 约束 |
|---|---|---|---|
| Control Task | 20ms | 动作调度、限速、舵机输出 | **唯一**可更新舵机输出 |
| Display Task | 事件/定时 | 屏幕刷新、表情动画 | 不得阻塞运动控制 |
| Behavior Task | 事件驱动 | 行为管理 | — |
| Input Task | 事件驱动 | 串口命令接收 | — |

其他任务通过 Queue/EventGroup/Mutex 提交控制请求（权限矩阵见 §5.5）；控制任务内避免耗时操作；第一版不强制双核绑定。

### 7.4 硬件/电气约束

- 共地；逻辑供电与舵机供电分离；舵机供电端预留滤波电容；5V 降压持续输出 ≥5A。
- 舵机安全角度范围须实测标定，不得假设 0°–180° 全部可达。
- **总线分配（M9）**：屏幕与 PCA9685 优先分属不同 I2C 总线/引脚；若必须共享同一总线，则 ① I2C 传输按块分片（单次持锁 ≤2ms）② Control Task 总线访问优先级高于 Display Task ③ 明确可接受的最坏持锁时长并作为验收指标。屏幕型号与总线归属为 **P0 闭项**。
- **PSRAM（R5）**：N16R8 的 8MB octal PSRAM 需在 sdkconfig 显式启用，且**占用 GPIO33–37**；P0 引脚规划须先决定 PSRAM 是否启用，避免与屏幕/舵机/I2C 撞车。

### 7.5 目录结构

原始需求给出建议目录（`main/` 下分 `app`、`hardware`、`control`、`action`、`behavior`、`gait`、`expression`、`storage`、`input`、`network`、`utils`）。**约束**：属规划结构，按阶段逐步创建，避免空文件与过度设计。`network/` 属 P7+，第一版不实现。

### 7.6 资源预算（S9）

| 项 | 约束 | 状态 |
|---|---|---|
| PSRAM | 8MB octal，需显式启用；用途：表情帧缓冲/双缓冲 | P0 决定 |
| Flash 分区 | NVS 分区（建议 ≥16KB）、app 分区；第一版无 OTA | P0 决定 |
| Control Task 栈 | ≥4096B（建议） | 【建议值】 |
| Display Task 栈 | ≥4096B（含帧缓冲时另评估） | 【建议值】 |
| 其余任务栈 | ≥2048B（建议） | 【建议值】 |
| 表情动画帧率 | ≥15fps（测量法：记录两次整屏刷新时间戳间隔） | 目标值 |

### 7.7 存储与上电/复位安全时序（S2、S4）

**配置存储**：结构带版本号 + CRC；加载时 CRC/版本校验失败 → 回退默认；**仅 `robot_config_save` 触发写 NVS**（运行期参数修改默认只改内存，减少磨损与掉电窗口）。

**上电/复位序列**：
1. 初始化 I2C → 探测 PCA9685（失败→E15）。
2. 全部通道写"无脉冲（off）"（抑制输出，防止上电瞬间舵机抽动）。
3. 载入配置（失败→默认）。
4. 在抑制输出状态下计算安全姿态。
5. 统一时刻使能并过渡到默认姿态（经限速）。

该序列总时长设上限（建议 ≤500ms【待确认】），期间禁止接受外部命令。看门狗/掉电重启按同一序列恢复。

### 7.8 电源与低电（S3，建议补充）

- **硬件验收**：堵转实测电流 + 堵转瞬时母线压降 ≤【建议 0.5V】。
- **Brownout**：定义检测等级策略（调低阈值或保留默认并复用 §7.7 复位恢复）。
- **电池低电**：若加 ADC 分压电路（需 BOM 增项），低电时执行"强制趴下/失能"的最小安全动作；无 ADC 则标注为后续。此项为【待确认】的选做项。

### 7.9 关节数与扩展预留

- `ROBOT_JOINT_COUNT = 4` 宏统一管理关节数，业务/动作/步态层**不得硬编码 4**（`servo_set_all` 的 4 参数接口为 V1.0 便捷接口，内部按宏展开）。
- `ServoMode`（POSITION/VELOCITY）枚举预留，第一版只实现 POSITION。连续旋转舵机无反馈时无法精确控角，未来需编码器/反馈。
- 360° 预留的**可检查判据**：角度语义不为 0–180 硬编码、`ServoMode` 枚举存在、位置/速度两态不共用同一 API 语义。

---

## 8. 验收标准

> 每项力求客观可验证。**数值冻结闸门**：§8.0 所有【建议值】须在 **P1 出口前冻结**，否则 P2 及以后阶段不具备验收条件。

### 8.0 通用量化建议（供各阶段引用）

| 项 | 建议值 | 状态 |
|---|---|---|
| 控制更新周期 | 20ms（50Hz） | 既定 |
| 默认 `max_speed` | 180°/s（即每 tick 3.6°） | 【建议值】 |
| `motion_speed` | 0–1，默认 1.0 | 既定（D5） |
| 单 tick 增量上限 | `max_speed × motion_speed × 0.02s`（默认 3.6°） | 既定公式，值随 max_speed |
| 相邻 tick 一阶差分上限 | 理论增量上限 × 1.2 | 既定判据 |
| 绝对脉宽硬保护 | 500–2500µs | 【建议值，待实测】 |
| 校准模式标定速度/步进 | 30°/s、1° | 【建议值】 |
| 最小过渡时间（防振荡） | 100ms | 【建议值】 |
| 所有权交接超时 | 1000ms | 【建议值】 |
| 控制任务周期抖动 | 连续 1000 次更新无一次 >50ms；平均误差 ≤2ms | 【建议值】 |
| 默认眨眼间隔 | 3000ms | 【建议值】 |
| 堵转母线压降 | ≤0.5V | 【建议值】 |

### 8.1 P0 — 硬件驱动

- [ ] 四个舵机独立控制（`servo_set_angle` 逐路生效）。
- [ ] `servo_set_all` 四通道**同一 tick 内生效**。
- [ ] PWM 输出稳定（示波器验证 4 路脉宽符合 §4.1 公式与硬保护区间）。
- [ ] 无阻塞：周期抖动指标达标。
- [ ] 上电/复位序列符合 §7.7（上电瞬间无舵机抽动）。
- [ ] 步态**实物可行性准入实验**完成，前进/后退/转向的 P5 范围已据此裁决（D4）。

### 8.2 P1 — 舵机校准

- [ ] 不改源码即可通过串口完成中位/方向/偏移/限幅/脉宽调整。
- [ ] **校准模式**可用：进模式解除限幅、降速、固定步进、退出回安全位（D6）。
- [ ] 绝对脉宽硬保护生效：任何配置下输出脉宽不越 `[500,2500]µs`。
- [ ] 重启后校准参数自动恢复；配置 CRC/版本校验失败回退默认。
- [ ] 越界角度正确限幅，无越界 PWM。

### 8.3 P2 — 动作引擎

- [ ] 仅凭动作数据（`ActionSequence`）即可实现招手、摇摆等动作。
- [ ] 动作代码中不含阻塞式延时（静态检查 + 周期抖动指标）。
- [ ] 平滑：每 tick 增量 ≤ `max_speed×motion_speed×0.02s`，且一阶差分 ≤ 理论值 ×1.2（M10 修正）。
- [ ] 状态机（IDLE/RUNNING/PAUSED/ESTOP）按 §4.4 表正确迁移（可脚本复现）。
- [ ] 支持循环、暂停、恢复、停止；`duration<20`/`frame_count=0` 被拒绝且不变态。
- [ ] 新动作立即打断并平滑切换，无回零跳变。

### 8.4 P3 — 姿态系统

- [ ] 可通过统一接口切换姿态，且切换经 Action Engine 平滑执行。
- [ ] 姿态变化平滑（复用 P2 平滑指标）。

### 8.5 P4 — 表情系统

- [ ] 表情可独立运行（切换期间舵机运动不中断）。
- [ ] 屏幕刷新不影响舵机运动（周期抖动指标达标；若共享 I2C，满足 §7.4 分片/持锁约束）。
- [ ] 支持 6 种情绪 + 自动眨眼 + 左右看；表情帧率 ≥15fps（若采用帧缓冲方案）。

### 8.6 P5 — 步态系统

- [ ] 步频/步幅/相位可独立调整且互不干扰。
- [ ] 步态计算与硬件驱动解耦（步态层只产出目标角度）。
- [ ] 可停止步态并恢复到指定姿态。
- [ ] 必须档：原地踏步/重心摆动/单腿试探摆动可达。
- [ ] 条件档（准入通过后）：X 秒内产生 ≥Y cm 位移且方向可重复（X/Y 于准入实验定义；地面材质限定）；未通过则本条作废并记录。

### 8.7 P6 — 行为管理

- [ ] 通过 `behavior_set` 即可触发完整行为（动作+表情+姿态组合）。
- [ ] 行为层不直接操作舵机（**可执行判据**：`behavior/` 目录源文件不得 `#include "pca9685.h"` / `"servo_controller.h"`，用 `grep`/构建期脚本自动检查）。
- [ ] 耦合可控（**可执行判据**：include 图方向单向，`behavior → action/gait/pose/expression`，反向零引用，用脚本检查）。

### 8.8 连续运行与热特性（S10）

- [ ] 连续运行 30 分钟循环动作：舵机外壳温升 ≤【建议 X℃】、校准角漂移 ≤【建议 Y°】、无 brownout 复位。作为 P2/P5 出口项之一。

### 8.9 代码质量（全程适用）

- [ ] 模块职责清晰；头文件只暴露必要接口。
- [ ] 无随意共享的全局变量。
- [ ] **统一使用 `esp_err_t` 返回类型，`void` 一律禁止**（M11）。
- [ ] 重要函数检查参数合法性；舵机角度必经限幅；关键参数可配置。
- [ ] 业务逻辑不直接依赖硬件驱动；动作不依赖阻塞延时。
- [ ] 每个阶段均可独立编译运行。

---

## 9. 尚待确认的问题（均带关闭阶段 + 关闭判据）

| # | 问题 | 关闭阶段 | 关闭判据 |
|---|---|---|---|
| H1 | 0.42" 屏幕具体型号 | **P0** | BOM 锁定 + 驱动 IC 型号确定 |
| H2 | 屏幕接口（I2C/SPI）及**总线归属**（是否与 PCA9685 共总线） | **P0** | §7.4 总线分配约束可评估（M9） |
| H3 | ESP32-S3 I2C 引脚分配 | **P0** | 引脚表定稿；与 PSRAM GPIO33–37 无冲突 |
| H4 | PCA9685 I2C 地址 | **P0** | 地址引脚接法确定（默认 0x40 为备选） |
| H5 | 控制台物理通道（USB-CDC vs UART0） | **P0** | 通道选定并写入 §4.8 |
| H6 | 四足机械结构运动方式 / 舵机安装位置 | **P0（准入实验）** | 准入实验完成并裁决 P5 范围 |
| H7 | 四个舵机机械运动方向 / 安全角度范围 | **P1 前** | 校准标定完成，写入 ServoConfig |
| H8 | 数值冻结：`max_speed` 默认、绝对脉宽区间、过渡/超时/抖动阈值等 §8.0 项 | **P1 出口** | 数值冻结（D3），否则 P2+ 无验收条件 |
| H9 | estop 输出策略（失能 vs 保持姿态 hold） | **P1 前** | 二选一定稿（默认失能） |
| H10 | `sit`/`crouch`/`lie_down` 是否合并 | 机械结构确定后 | 依 1-DOF 结构裁决，合并则更新姿态表 |
| H11 | `display_brightness` 保留或删除 | 屏幕型号确认后 | 依屏幕能力裁决 |
| H12 | `BEHAVIOR_DANCE` 的动作组合内容 | **P6 前** | 定义其动作来源（组合或新增） |
| H13 | 步态判定阈值 X 秒/Y cm | **P0（准入实验）** | 准入实验时定义 |
| H14 | 电池电压 ADC 监测与低电行为（选做，S3） | P1 前（若做） | BOM 增 ADC 分压项，否则标注后续 |
| H15 | PSRAM 是否启用（占用 GPIO33–37） | **P0** | sdkconfig 决定，引脚规划随之定稿 |

> 已由本轮决策关闭的问题（不再待确认）：Q1 插值粒度→帧级 `ease`；Q2 速度参数→D5；Q3 失能语义→§5.1 冻结；Q4 关节数→`ROBOT_JOINT_COUNT`；Q5 优先级→§4.9 所有权；Q6 中断过渡→§4.4/§5.2；Q7 步态判定→H13；Q8 版本升级→§7.7 CRC+版本；Q9 阈值→H8 冻结闸门。

---

## 10. 最终设计目标

第一版最终应成为结构清晰、易调试、易扩展的小型机器人控制系统。分层职责一句话约束：

> 硬件负责执行，控制层负责运动与限速，动作层负责时间序列，步态层负责周期运动，行为层负责决策，交互层负责接收指令。

第一版优先保证**架构正确、基础功能稳定**，不追求一次性实现所有功能。凡标注【待确认】者，均须在其关闭阶段完成裁决后方可作为验收依据。
