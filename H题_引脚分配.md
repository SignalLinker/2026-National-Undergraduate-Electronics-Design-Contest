# H题 MSPM0G3507 引脚分配草案

依据文件：

- `C:/Users/ASUS/Desktop/原理图Schematic1_转换为Markdown.md`
- 当前已确认外设：8 路数字循迹、I2C OLED、TB6612、电机、MSPM0G3507

说明：原理图 Markdown 存在中文乱码，但网络名、外设名和引脚号基本可读。TB6612 和灰度模块的截图已补齐此前缺失的信号连接，以下以截图和原理图网络名为准。

## 1. 已确认引脚

### 1.1 OLED 显示屏

OLED 使用 I2C0，总线与板载陀螺仪共用。

| OLED 引脚 | 网络 | MSPM0G3507 引脚 |
| --- | --- | --- |
| VCC | +5V | - |
| GND | GND | - |
| SDA | I2C0_SDA | PA28 |
| SCL | I2C0_SCL | PA31 |

注意：

- 常见 SSD1306 OLED 地址通常是 `0x3C` 或 `0x3D`。
- I2C0 还可能连接板载陀螺仪，软件初始化时不能只假设总线上只有 OLED。

### 1.2 TB6612 电机和方向控制

TB6612 的两路 PWM 和四个方向脚均已确认。

| TB6612 引脚 | 功能 | 网络 | MSPM0G3507 引脚 |
| --- | --- | --- | --- |
| PWMA | A 通道 PWM | TIMG0_C0_PA12 | PA12 |
| AIN1 | A 通道方向 1 | - | PB17 |
| AIN2 | A 通道方向 2 | - | PB19 |
| PWMB | B 通道 PWM | TIMG0_C1_PA13 | PA13 |
| BIN1 | B 通道方向 1 | - | PA16 |
| BIN2 | B 通道方向 2 | - | PB24 |
| STBY | 待机控制 | +5V | 常使能 |

建议配置：

- PWM 频率：20kHz
- 初始占空比：0
- 左右电机方向先低速测试后再确定正反

### 1.3 电机接口和编码器

两个电机接口均带有 `VCC/A/B/GND` 编码器信号，因此当前电机具备编码器闭环条件。

| 接口 | TB6612 输出 | 电机电源线 | 编码器 A | 编码器 B | 备注 |
| --- | --- | --- | --- | --- | --- |
| MOTOR1 | B 通道 | M1=BO1，M2=BO2 | PA25，TIMG12_C1 | PA14，TIMG12_C0 | 物理左右待低速测试确认 |
| MOTOR2 | A 通道 | M3=AO2，M4=AO1 | PA26，TIMG8_C0 | PA27，TIMG8_C1 | 物理左右待低速测试确认 |

编码器使用建议：

- 两路 A/B 信号都配置为 GPIO 双边沿中断，先完成正交解码。
- 如果计数方向与实际反向，只交换 A/B 解码逻辑或取计数负值，不要改动硬件。
- 外层循迹 PID 输出左右轮目标速度，内层编码器 PID 调节左右电机 PWM。

### 1.4 舵机接口

虽然当前还没有舵机，但板上已有 4 路舵机 PWM 接口。

| 接口 | PWM 网络 | MSPM0G3507 引脚 | 建议用途 |
| --- | --- | --- | --- |
| 舵机 1 | TIMA0_C0_PA21 | PA21 | 摆杆控制首选 |
| 舵机 2 | TIMA0_C1_PA22 | PA22 | 备用 |
| 舵机 3 | TIMA0_C2_PA15 | PA15 | 备用 |
| 舵机 4 | TIMA0_C3_PA17 | PA17 | 备用 |

建议摆杆舵机优先接舵机 1，即 `PA21`。

### 1.5 可用 UART

后续摄像头/视觉模块建议使用 UART 与主控通信。

| 串口 | TX | RX | 建议用途 |
| --- | --- | --- | --- |
| UART0 | PA0 | PA1 | Zigbee 或备用 |
| UART1 | PB6 | PB7 | 摄像头/OpenMV 首选 |
| UART2 | PA23 | PA24 | 调试或备用 |
| UART3 | PB2 | PB3 | 调试或备用 |

推荐：

- OpenMV TX -> MSPM0G3507 UART1_RX `PB7`
- OpenMV RX -> MSPM0G3507 UART1_TX `PB6`
- GND 必须共地

## 2. 8 路数字循迹复用模块

该模块不是 8 路独立输出，而是使用 3 位地址线选通 8 路灰度传感器，再从单个 `OUT` 引脚读回当前通道电平。

| 灰度模块引脚 | MSPM0G3507 引脚 | 主控方向 | 用途 |
| --- | --- | --- | --- |
| AD0 / P0 | PB0 | 输出 | 通道地址位 0 |
| AD1 / P1 | PB1 | 输出 | 通道地址位 1 |
| AD2 / P2 | PB15 | 输出 | 通道地址位 2 |
| OUT | PB16 | 输入 | 当前选中的灰度传感器数字输出 |
| GND | GND | - | 共地 |
| 5V | +5V | - | 模块供电 |

扫描规则：

```text
传感器 0: AD2 AD1 AD0 = 000
传感器 1: AD2 AD1 AD0 = 001
传感器 2: AD2 AD1 AD0 = 010
...
传感器 7: AD2 AD1 AD0 = 111
```

每次扫描流程：

```text
设置 AD2/AD1/AD0 -> 等待 5~20us -> 读取 OUT -> 保存该通道状态
```

建议每 5ms 完成一次全部 8 路扫描。软件中保留 `GRAY_BLACK_LEVEL` 配置，因为不同模块在黑线处可能输出低电平或高电平。

电平注意事项：

- 模块由 +5V 供电时，必须确认 `OUT` 输出对 MSPM0G3507 的 PB16 是安全的。
- 若 OUT 是 5V 推挽输出且板上没有电平转换，应增加分压或电平转换。
- 若现有 `06_gray8_mux_verify` 工程已经正常读取该模块，可按该工程的供电和接线方案继续使用。

## 3. 第一版程序使用的逻辑宏

建议程序中先用逻辑宏隔离硬件，方便后续改引脚。

```c
// Motor PWM
#define MOTOR_A_PWM_PORT_PIN   PA12
#define MOTOR_A_IN1_PIN        PB17
#define MOTOR_A_IN2_PIN        PB19
#define MOTOR_B_PWM_PORT_PIN   PA13
#define MOTOR_B_IN1_PIN        PA16
#define MOTOR_B_IN2_PIN        PB24

// OLED I2C
#define OLED_I2C_SDA_PIN       PA28
#define OLED_I2C_SCL_PIN       PA31

// 8-channel grayscale multiplexer
#define GRAY_AD0_PIN           PB0
#define GRAY_AD1_PIN           PB1
#define GRAY_AD2_PIN           PB15
#define GRAY_OUT_PIN           PB16

// Wheel encoders
#define MOTOR1_ENCODER_A_PIN   PA25
#define MOTOR1_ENCODER_B_PIN   PA14
#define MOTOR2_ENCODER_A_PIN   PA26
#define MOTOR2_ENCODER_B_PIN   PA27

// Servo
#define BALL_SERVO_PWM_PIN     PA21

// Vision UART
#define VISION_UART_TX_PIN     PB6
#define VISION_UART_RX_PIN     PB7
```

这些宏是“逻辑说明”，实际 MSPM0 SDK/DriverLib 代码需要按 SysConfig 生成的端口、引脚和外设名填写。

## 4. 现在的下一步

1. 建立 MSPM0G3507 SysConfig：2 路电机 PWM、4 路方向 GPIO、4 路编码器中断、3 路灰度地址输出、1 路灰度输入、I2C0 和按键。
2. 编写灰度复用扫描函数，验证 000 到 111 对应的物理传感器顺序。
3. 编写电机和编码器测试程序，确认 MOTOR1/MOTOR2 对应车体左右侧和正方向。
4. 完成速度 PID 后，再做外层循迹 PID。
