# FinsROV 项目解读（上海交大 FINS 实验室）

> 仓库：`FPJ-GAOGE/FinsROV-An-Underwater-Camera-Based-Multi-Robot-Platform`
> 265★ ｜ 23 fork ｜ MIT ｜ 主语言 C ｜ 创建 2026-01-30 ｜ 最后推送 2026-04-05
> 本地位置：`./FinsROV/`（稀疏克隆，28 MB，已剔除 220 MB 无关文件）

---

## 一、这是什么

一台**低成本开源水下机器人平台**，由上海交通大学 FINS 实验室开源。配套论文 AUCAMP（水下多机器人低成本分布式定位，arXiv 2506.09876）。

**它的定位是「教学 + 科研平台」，不是产品。** 这一点决定了它对你的价值——它天生就是给人读、给人改的。

### 硬指标

| 项目 | 参数 |
|---|---|
| 尺寸 | 40 × 35 × 15 cm |
| 空气中重量 | 4.85 kg |
| 作业深度 | **10–20 m**（已过内压测试） |
| 功率 / 续航 | 约 13.75 W / 约 2 h |
| 静水最大速度 | 约 3 m/s |
| 最大加速度 | 4.6 m/s² |
| 最大转向角速度 | 5.7 rad/s |
| 控制频率 | **150 Hz**（README 写 150，代码里 TIM1 实际是 300 Hz，见下方"代码与文档对不上的地方"） |
| 串口波特率 | 115200 |

### 配置

- **推进器 ×8**（内圈 4 个垂直 + 外圈 4 个水平）——八推全向运动
- **舵机 ×4**（两个二自由度云台，分列两侧，每舵机 ±90°）
- **水深传感器 ×4**（MS5837-30BA，分布式布置在四个位置）
- 防水舱 + 铝合金 CNC 结构件 + 3D 打印内部支架

---

## 二、硬件架构（这是它最值得学的设计）

```
        ┌─────────────── 防水舱 ───────────────┐
        │                                      │
上位机 ──USB──> 422转USB ──422──> TTL转422 ──> C板 UART6
(手柄)                                          │
                                        大疆 RM C 板
                                      (STM32F407IGHx)
                                        │          │
                              I2C ──────┘          └────── I2C
                                │                          │
                     ┌──────────▼─────────┐      ┌─────────▼──────────┐
                     │ I2C 扩展板 TCA9548A │      │ PWM 扩展板 PCA9685  │
                     │   （8 路通道切换）   │      │                    │
                     └──┬──┬──┬──┬────┬───┘      └──┬───────┬─────────┘
                     0  1  2  3     4              0-7     8-11
                     │  │  │  │     │              │        │
                   水深计×4        └─>PWM扩展板   推进器×8   舵机×4
```

**关键点：C 板 I2C 引脚不够用，所以挂了一块 TCA9548A 做多路复用。**

- I2C 输出端口 0–3 → 4 个水深传感器（每路靠 `TCA_SetChannel(i)` 切通道分时访问）
- 端口 4 → 级联到 PCA9685 PWM 扩展板
- PCA9685 通道 0–7 → 8 个推进器；通道 8–11 → 4 个舵机

**这个"一个 I2C 挂在扩展板上、先切通道再读写"的套路，是你自己做任何多传感器水下平台都会遇到的第一个硬件瓶颈。** 值得重点看。

---

## 三、软件架构（下位机）

### 1. Device 抽象——整个固件的骨架

`Usermain.h` 把每个硬件模块抽象成一个 `Device` 类，三个纯虚函数：

| 接口 | 何时被调用 | 作用 |
|---|---|---|
| `Init()` | `main()` 里循环调用 | 初始化硬件 |
| `Receive()` | **串口空闲中断**里 | 把收到的数据交给各设备解析 |
| `Handle()` | **定时器中断置标志 → 主循环**里 | 周期性处理 |

`Usermain.cpp` 里用**指针数组**集中注册设备：

```cpp
static Device *device[] = {
    &IMU::imu,                          // BMI088 + IST8310
    &PressureSensor::pressure_sensor,   // 4 路 MS5837
    &propeller_i2c,                     // 8 推进器
    &servo_i2c,                         // 4 舵机
    &led,
};
#define DEVICE_NUM (sizeof(device) / sizeof(device[0]))
```

**这是全仓库最值得抄的一段设计。** 想加设备？写个继承 `Device` 的类，往数组里加一行。加设备不改调度逻辑——这就是正确抽象的样子。

### 2. 调度：TIM1 定时器 + 主循环标志位

```cpp
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if(!system_init_flag) return;
    if(htim == &htim1) tick_300hz = true;   // 只置标志，不干活
}

while (1) {
    if(tick_300hz){
        tick_300hz = false;
        for(int i = 0; i < DEVICE_NUM; ++i)
            device[i]->Handle();             // 轮询所有设备
    }
}
```

**中断里只置标志、主循环里干活**——这是嵌入式必须遵守的纪律。中断里跑长任务会丢事件。

### 3. 串口：空闲中断收不定长数据

`UART6` 用 `HAL_UARTEx_ReceiveToIdle_IT()`，收到一帧（总线空闲）触发 `HAL_UARTEx_RxEventCallback()`，然后遍历 `device[]` 调 `Receive()`，处理完 `memset` 清缓冲再重新开启接收。

> 上位机→ROV 的物理链路是 **USB → 422 → TTL → UART6**。为什么要转 422 而不是直接 USB 转 TTL？因为线缆要拉几十米到水下，TTL 走不了长线，422 差分能抗干扰。这个选型理由你必须理解。

### 4. 姿态解算：两条独立的估计路径

这是 FinsROV 比较特别的地方——**姿态不是只靠 IMU**。

**路径 A（姿态基准）**：BMI088（陀螺仪 + 加速度计，SPI + DMA 分段读）+ 可选 IST8310 磁力计 → `offset_correction()` 零偏补偿 → **MahonyAHRS** 解四元数 → 欧拉角 → `attitude_update()` 更新 yaw/pitch/roll。

**路径 B（深度闭环 + 姿态外环）**：4 路 MS5837-30BA 水压计，I2C 分时读取，三段流水线：

1. `GET_TEMPERATURE` —— 依次切 4 路，下发"开始温度转换"
2. `GET_PRESSURE` —— 依次读温度结果，再下发"开始压力转换"
3. `CALCULATE` —— 依次读压力结果，温度补偿 + 校准 → 深度值

> **注意这个三段式。** 传感器转换需要时间，如果"下发→死等→读取"会把控制周期撑爆。这里是"这一轮下发，下一轮取结果"，用流水线把等待时间藏起来。这是实时系统的核心技巧。

### 5. 控制：PID 与双环结构

`Propeller.cpp` 里的关键设计：

- **`flag_float` 悬浮模式**：开启后内圈推进器做闭环（深度 + 横滚 + 俯仰），外圈推进器走开环平移/旋转（WASD/QE/Z）。关闭则全部输出 `InitPWM`（停）。
- **深度**：`DepthPID(Target_depth, 实测深度)`
- **横滚/俯仰：双环 PID**，但**外环的反馈量不是 IMU 角度，而是水压计差分**：
  - 外环：`RollOutPID(Target_roll, pressure_roll_diff)`、`PitchOutPID(Target_pitch, pressure_pitch_diff)`
  - 内环：用 IMU 角速度 `neg_roll_v`（注意符号取反）、`pitch_v`

**为什么用水压差分算姿态角？** 因为 4 个水压计分布在四个位置，机器倾斜时各点深度不同，深度差 → 几何解算倾斜角。好处是不累积漂移（不像积分类姿态估计），缺点是**动态响应慢**——所以内环必须用 IMU 陀螺仪的高频角速度补上。

**"低频不漂移的量做外环 + 高频会漂移的量做内环"**，这是多传感器融合的通用思路，值得到处用。

### 6. PID 实现（`PID.cpp`，54 行，可以直接抄）

```cpp
PIDInfo.err[3] = ref - fdb;
PIDInfo.componentKp = err[3] * kp;
PIDInfo.errSum += err[3];
INRANGE(errSum, -kiMax/ki, kiMax/ki);          // 积分限幅（抗饱和）
PIDInfo.componentKi = errSum * ki;
PIDInfo.componentKd = (err[3] - err[2]) * kd;
INRANGE(componentKp, ±kpMax);                   // 各项单独限幅
INRANGE(componentKi, ±kiMax);
INRANGE(componentKd, ±kdMax);
output = componentKp + componentKi + componentKd;
INRANGE(output, ±outputMax);                    // 总输出限幅
```

四层限幅，干净利落。**改进空间**：没有 D 项低通滤波（噪声大的时候会抖）、没有微分先行、没有积分分离。你以后调参调不动的时候，从这三个方向改。

---

## 四、串口协议（上位机 ↔ ROV）

| 模块 | 命令 | 含义 |
|---|---|---|
| 推进器 | `ON` / `OFF` | 开启 / 关闭悬浮闭环控制 |
| 推进器 | `UP` / `DN` | 目标深度 ∓1 |
| 推进器 | `H:<value>` | 目标保持深度 = value/10 cm |
| 推进器 | `W` `S` `A` `D` | 前 / 后 / 左 / 右 |
| 推进器 | `Q` / `E` | 逆 / 顺时针旋转 |
| 推进器 | `Z` | 外圈停止（悬停） |
| 推进器 | `TES:pwm0..pwm7` | 直接写 8 路 PWM（测试用） |
| 舵机 | `MOT:p0,p1,p2,p3` | 4 路舵机 PWM |
| 水压计 | `CA` | 参数校准 |
| 水压计 | `VA` | 回读水压计数值 |

**注意 `TES:` 这条**——绕过所有闭环直接写 PWM。调试时靠它验证"是控制算法的问题还是硬件接线的问题"。自己做的时候一定要留这个后门。

---

## 五、成本：约 8000 元

BOM 里价格列是**行总价**（已核对中英文两版 + 列位）：

| 分类 | 明细 | 小计 |
|---|---|---|
| 机械零件 | 后舱盖支撑 240、前舱支撑 180、推进器支撑 680、水平推进器架 480、穿线螺丝 1140、水下航插 272、舵机连接件 18.4、金属舵盘 10、防水舱 515 | **≈ 3535** |
| 电气设备 | 推进器 2037（4 对正反桨含电调）、舵机 432（4 个全防水）、水深传感器 1319.6（4 路含转接板）、I2C 扩展板 10.56、PWM 扩展板 15、TTL转422 9.1、422转USB 79、无线调试器 89（2 个）、大疆 RM C 板 379 | **≈ 4370** |
| 其他 | 接线端子 97.8 | ≈ 98 |
| | **合计** | **≈ 8003 元** |

**不含**：相机、定制零浮力电缆、3D 打印件、螺丝小件、防水 O 型圈（表里无数值）。

**最贵的单件**：推进器 2037（占 1/4）、水深传感器 1319.6、穿线螺丝 1140、推进器支撑 680、防水舱 515。

> 对照一下：BlueROV2 整机成品两三万起。**FinsROV 八千块能做完，这是它最大的现实价值。**

---

## 六、已知坑（作者自己在文档里记录的，照抄即可避雷）

1. **V33-0 与 V33-2 版本的电调初始化 PWM 值不同。** 上电后如果没听到电调初始化提示音，去 `propeller.cpp` 改初始化 PWM 值。
2. **V33-0 与 V33-2 内部 RS485/422 转 TTL 的线序不同**（2026-04-05 调试发现）。若用带相机的那个线盘，接收正常但发送无响应 → 交换发送端 RS485/422 转 USB 的 **T/R+ 与 T/R−** 线序。
3. **上机前必读的操纵注意事项**（来自手柄控制说明）：
   - 推进器啸叫 / 异常噪音 → 立刻输 `OFF`，再输 `ON` 重启，一般不用断电，重复到声音正常
   - 按键无反应 → 直接断电重启
   - 用手柄控制时**必须关闭串口调试助手**，串口被占用会导致手柄失效

---

## 七、代码与文档对不上的地方（我核对发现的）

| 项 | 文档说 | 代码里实际 | 说明 |
|---|---|---|---|
| 控制频率 | 150 Hz | `HAL_TIM_PeriodElapsedCallback` 里 `htim1` 注释写 **300 Hz**，标志位也叫 `tick_300hz` | 以代码为准是 300 Hz；README 的 150 Hz 可能是旧版本或"实际有效处理频率" |
| 串口分发 | "统一分发到各设备模块解析" | `RxEventCallback` 里**逐设备无差别调用** `Receive()`，按前缀分发的代码被注释掉了 | 每个设备的 `Receive()` 自己判断是不是自己的命令。粗看是浪费，但实测更简单可靠 |
| 烧写工具 | README 说用 Keil | 仓库里同时有 `CMakeLists.txt` + `build/`（CMake 3.28/3.30） | **两条构建路径**，CMake 那条看起来是有人在用 VSCode 开发 |

---

## 八、本地已接入的内容

**位置**：`./FinsROV/`（28 MB）

```
FinsROV/
├── README.md                          ← 主文档（含参数表、BOM 链接、B站教程链接）
├── Making_Instructions/
│   ├── README.md                      ← ★ 接线指南：每个螺丝孔对应什么、端子怎么接
│   └── Images/                        ← 接线图 3 张
├── Structure/
│   ├── Configuration.STEP             ← 整机装配体（18 MB，SolidWorks 可打开）
│   ├── Images/                        ← 整机渲染图
│   └── _BOM/                          ← 我额外抓下来的物料表
│       ├── bom_zh.xlsx / BOM_en.xlsx  ← 完整采购清单（含淘宝链接）
│       └── CNC加工.xlsx                ← CNC 件图纸说明
└── Code/
    ├── Joystick_Control/
    │   ├── readme.md                  ← ★ 上位机环境与操作说明
    │   ├── usbCameraFullControl.py    ← 手柄控制主程序（pygame + opencv）
    │   └── serialPort.py              ← 串口封装
    └── Lower_Level_Controller/
        ├── README.md                  ← ★★ 全仓库最有价值的一份文档，逐模块讲架构
        ├── Inc/ Src/                  ← CubeMX 生成的初始化代码（main.c 等）
        └── userCode/                  ← ★★★ 真正的固件，150 KB
            ├── devices/               ← IMU / Sensor / Propeller / Servo / LED / Buzzer / Watchdog / Usermain
            ├── algorithms/            ← PID / MahonyAHRS / Kalman / Matrix / Map
            ├── drivers/               ← BMI088 / IST8310 寄存器级驱动
            ├── MiddleWares/            ← 传感器中间层
            └── bsp/                   ← SPI / UART 底层封装
```

**没下载的**：`Structure/En`、`Structure/zh-CN` 里各约 50 MB 的分散 STEP 零件图，以及 `Code/.../Drivers/STM32F4xx_HAL_Driver/` 下约 120 MB 的 STM32 官方手册 `.chm`。**这些是噪音**——需要某个具体零件的 STEP 时单独取即可，告诉我零件名就行。

---

## 九、学习顺序（建议）

| 步 | 做什么 | 为什么 |
|---|---|---|
| 1 | 读 `Code/Lower_Level_Controller/README.md` | 11.7 KB，双语，把整个固件架构讲完了。**先读这份，不要先读代码** |
| 2 | 读 `Making_Instructions/README.md` + 3 张接线图 | 建立"哪根线接哪"的物理直觉，代码里那些 `TCA_SetChannel`、`PCA_Setpwm` 才对得上号 |
| 3 | 读 `Usermain.h` + `Usermain.cpp` | 看 `Device` 抽象和调度骨架——**这是全项目最值得抄的 100 行** |
| 4 | 读 `userCode/devices/Src/Propeller.cpp` + `Propeller.h` | 控制核心：双环 PID、PWM 分配、`InID`/`OutID`/`Sign` 映射 |
| 5 | 读 `userCode/algorithms/Src/PID.cpp`（54 行） | 最短的一份，但四层限幅值得记住 |
| 6 | 读 `userCode/devices/Src/Sensor.cpp` | 三段式水压采集流水线，理解"为什么要藏等待时间" |
| 7 | 读 `userCode/devices/Src/IMU.cpp` + `algorithms/Src/MahonyAHRS.cpp` | 姿态解算。MahonyAHRS 15.6 KB，是全项目最"理论"的一块，**可以最后读** |
| 8 | 打开 `Structure/Configuration.STEP` | 用 CAD 对照实物，把软件里的编号和空间位置对上 |

**配套视频（作者提供，中文）**：

- 从 0-1 制作水下机器人：B站 `BV13vcvzgEv3`
- 水下机器人动力学与运动学建模：B站 `BV1VXmQBRE2a`

---

## 十、和之前给你的 ArduSub 路线怎么选

我之前说过"别自研控制器，走 ArduSub + BlueROV2"。你带来 FinsROV 之后，**这个建议要修正**——两条路根本不是一回事：

| | **FinsROV** | **ArduSub + BlueROV2** |
|---|---|---|
| 性质 | 交大实验室的**教学 / 科研平台** | 工业级**开源飞控**，事实标准 |
| 代码量 | 固件核心 **约 150 KB**，读得完 | ardupilot 15k★，相关代码你读不完 |
| 深度 | 10–20 m | 100 m+ |
| 定位能力 | 相机多机定位（AUCAMP） | 可扩展 DVL / SLAM / 声呐 |
| 文档 | **中文，含 B 站视频教程** | 英文为主，`ardusub-gitbook` |
| 零件采购 | **淘宝，全部可买** | 官方进口为主 |
| 成本 | **约 8000 元** | 整机两三万起 |
| 适合 | **搞懂原理、动手做出来、做研究** | 要可靠、要下深水、要上生产 |

**我的修正建议：把 FinsROV 定为学习主线，ArduSub 降为参照系。**

理由很直接：你的目标是"从零学会做一个水下机器人"。FinsROV 的 150 KB 固件你能**全读懂**，而 ArduSub 你只能读文档、读不懂源码——**看得完的东西才能学会**。等你想知道"工业级实现怎么处理同样的问题"时，再翻 ArduSub 对照，那时候你已经有判断力了。

原来的判断错在把"做一个能用的 ROV"和"学懂 ROV 怎么工作"当成了同一件事。它们是两件事。

---

## 十一、下一步，你说一句

1. **逐文件精讲 `userCode/`**（建议从这里开始）——我按调用链把 `Usermain` → `Device` → 各设备的实现讲一遍
2. **出采购清单**——把 BOM 整理成带淘宝链接、可勾选的清单，标出哪些能替代、哪些必须定制
3. **画一张控制数据流图**——从水压计/IMU 采样，到 PID 输出，到 PCA9685 写寄存器的完整链路
4. **对照讲 ArduSub**——同一个控制问题，工业级怎么解，讲清楚差距在哪
5. **补全没下的 STEP 文件**——告诉我要哪个零件，我单独取

我的建议是**先做第 3 条**。你现在的瓶颈不是读不懂代码细节，是脑子里还没有"数据从传感器流到推进器"的完整图景。这张图画出来，代码自己就看懂了。
