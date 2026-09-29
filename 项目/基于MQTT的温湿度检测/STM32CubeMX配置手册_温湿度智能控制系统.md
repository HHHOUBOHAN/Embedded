# STM32CubeMX 配置手册
## 基于 STM32F103C8T6 的温湿度监测与智能控制系统

**工程目标：** 生成一个与本项目 PCB 完全匹配的 HAL 空工程。  
**开发环境：** STM32CubeMX + Keil MDK + HAL 库。  
**本手册范围：** 只讲 CubeMX 配置和生成工程；不直接编写 AHT20、OLED、MQTT 等完整业务代码。
## 0. 配置前必须确认的硬件事实(本项目最终引脚表)

| 功能                  | STM32 引脚              | CubeMX 选择      | 是否本次配置 |
| ------------------- | --------------------- | -------------- | ------ |
| AHT20 / OLED I2C 时钟 | PB6                   | `I2C1_SCL`     | 是      |
| AHT20 / OLED I2C 数据 | PB7                   | `I2C1_SDA`     | 是      |
| ESP-01S 接收 STM32 数据 | PA9                   | `USART1_TX`    | 是      |
| ESP-01S 发送 STM32 数据 | PA10                  | `USART1_RX`    | 是      |
| 红 LED               | PA4                   | GPIO Output    | 是      |
| 绿 LED               | PA5                   | GPIO Output    | 是      |
| 蜂鸣器控制               | PB0                   | GPIO Output    | 是      |
| 风扇 MOS Gate         | PA8                   | `TIM1_CH1` PWM | 是      |
| KEY1                | PB12                  | GPIO Input     | 是      |
| KEY2                | PB13                  | GPIO Input     | 是      |
| KEY3                | PB14                  | GPIO Input     | 是      |
| SWD 下载数据            | PA13                  | SYS_SWDIO      | 是，自动占用 |
| SWD 下载时钟            | PA14                  | SYS_SWCLK      | 是，自动占用 |
| 8 MHz 晶振            | PD0 / PD1             | RCC HSE        | 是，自动占用 |
| 备用排针                | PA0 / PA1 / PA2 / PA3 | 暂不配置           | 否      |
| BOOT0、NRST          | 专用硬件脚                 | 不在普通 GPIO 中配置  | 否      |

---

## 1. 新建工程与芯片选择

### 1.1 作用

让 CubeMX 知道封装、外设数量和可用引脚，进而生成 STM32F1 对应 HAL 代码。

### 1.2 操作

1. 打开 STM32CubeMX。
2. 选择 `New Project`。
3. 在 `MCU/MPU Selector` 搜索：`STM32F103C8Tx`。
4. 选择 LQFP48 封装对应的 `STM32F103C8Tx`，点击 `Start Project`。

### 1.3 为什么这样选

`STM32F103C8T6` 的核心型号是 `STM32F103C8Tx`；最后的 `T6` 表示封装和温度等级。CubeMX 选择 `C8Tx` 后会给出 48 引脚封装和正确外设资源。

### 1.4 不要这样做

```text
不要选 STM32F103RCT6：封装、Flash 和可用 IO 不同。
不要选 Blue Pill 开发板：那只是开发板，不是你的自制 PCB。
不要选 Generic MCU：会失去正确的引脚约束。
```

---

## 2. 先配置 SYS 与 RCC

配置顺序很重要：先确定调试接口与外部晶振，再配置通信和 GPIO，可减少 CubeMX 自动抢占引脚的情况。

## 2.1 SYS：保留 SWD 下载调试

### 作用

SWD 是 ST-Link 下载程序、单步调试、查看变量的接口。没有 SWD，第一块自制板一旦程序异常，调试会非常痛苦。

### 操作

1. 左侧 `System Core` -> `SYS`。
2. `Debug` 选择：`Serial Wire`。
3. `Timebase Source` 保持：`SysTick`。

### 为什么这样配置

| 项目 | 原因 |
|---|---|
| `Serial Wire` | 只占 PA13/PA14，保留 SWD 下载；比 Full JTAG 少占用引脚 |
| `SysTick` | HAL 默认每 1 ms 产生系统时基，`HAL_Delay()`、`HAL_GetTick()` 和软件定时任务都依赖它 |

### 配完后应看到

```text
PA13 = SYS_SWDIO
PA14 = SYS_SWCLK
```

若 PA13/PA14 仍是普通 GPIO，说明 SYS 没配对；不要继续往下配。

---

## 2.2 RCC：启用 8 MHz 外部晶振

### 作用

RCC 是时钟控制模块。你 PCB 上已焊接 8 MHz 晶振，必须告诉 STM32 使用它，后续才能稳定运行到 72 MHz，并保证 UART 波特率准确。

### 操作

1. 左侧 `System Core` -> `RCC`。
2. 在 `High Speed Clock (HSE)` 选择：`Crystal/Ceramic Resonator`。
3. `Low Speed Clock (LSE)` 保持：`Disable`。

### 为什么这样配置

| 设置 | 原因 |
|---|---|
| HSE = Crystal/Ceramic Resonator | PD0、PD1 接的是外部 8 MHz 晶振，不是外部有源时钟模块 |
| 不选 Bypass Clock Source | `Bypass` 只用于外部模块直接输出方波时钟；你的电路是晶振加负载电容 |
| LSE Disable | 板上没有 32.768 kHz RTC 晶振，V1 不使用 RTC |

### 常见错误

| 错误 | 现象 |
|---|---|
| HSE 选 Disable | 程序可能仍运行，但时钟频率、UART 波特率与设计不一致 |
| HSE 选 Bypass | 外部晶振不起振，程序卡在时钟初始化或运行异常 |
| 晶振电路焊接有误 | CubeMX 配置正确也不能启动；先用 ST-Link 检查芯片是否能连接 |

---

## 3. 配置时钟树：8 MHz 到 72 MHz

### 3.1 作用

时钟决定 CPU 执行速度、UART 波特率、I2C 频率和 PWM 频率。这里配错，最常见现象是 ESP8266 串口乱码、延时不准、PWM 频率不对。

### 3.2 操作

点击顶部 `Clock Configuration`，按下表设置。CubeMX 会自动标红无法使用的组合；看到红色时不要直接忽略。

| 时钟项 | 目标值 | 如何得到 | 原因 |
|---|---:|---|---|
| HSE | 8 MHz | PCB 外部晶振 | 硬件实际值 |
| PLL Source | HSE | 选择 HSE | 以外部晶振为基准 |
| PLL MUL | x9 | 8 x 9 = 72 | F103C8 的最高系统频率 |
| SYSCLK | 72 MHz | PLL 输出 | CPU 主频 |
| AHB Prescaler | /1 | 72 / 1 | HCLK = 72 MHz |
| APB1 Prescaler | /2 | 72 / 2 | PCLK1 = 36 MHz，不超过 APB1 限制 |
| APB2 Prescaler | /1 | 72 / 1 | PCLK2 = 72 MHz，TIM1/USART1 位于 APB2 |
| Flash Latency | 2 WS | CubeMX 通常自动设置 | 72 MHz 下 Flash 读取需要等待周期 |

最后确认时钟树中至少显示：

```text
SYSCLK = 72 MHz
HCLK   = 72 MHz
PCLK1  = 36 MHz
PCLK2  = 72 MHz
TIM1CLK = 72 MHz
USART1CLK = 72 MHz
```

### 3.3 为什么 APB1 要除以 2

STM32F103 的 APB1 总线最高是 36 MHz。若设置 APB1 `/1`，CubeMX 会报错或配置不合法。I2C1 位于 APB1，因此它获得的是 36 MHz 外设时钟；这已经足够产生 100 kHz I2C。

### 3.4 为什么 TIM1 仍是 72 MHz

TIM1 位于 APB2，而 APB2 设置为 `/1`，所以 TIM1 时钟为 72 MHz。PWM 频率计算以 72 MHz 为基础：

```text
PWM Frequency = 72,000,000 / (PSC + 1) / (ARR + 1)
              = 72,000,000 / 1 / 3600
              = 20,000 Hz
```

---

## 4. 配置 I2C1：AHT20 与 OLED 共用总线

### 4.1 作用

I2C 是两根线的通信总线：`SCL` 提供时钟，`SDA` 传输数据。AHT20 与 OLED 可以挂在同一对 PB6/PB7 上，因为它们的 I2C 地址不同。

### 4.2 操作

1. 左侧 `Connectivity` -> 点击 `I2C1`。
2. 在 Mode 中选择 `I2C`。
3. CubeMX 应自动分配：

```text
PB6 = I2C1_SCL
PB7 = I2C1_SDA
```

4. 在 `Parameter Settings` 设置：

| 参数 | 值 |
|---|---|
| I2C Speed Mode | Standard Mode |
| I2C Clock Speed | 100000 Hz |
| Duty Cycle | 2 |
| Own Address 1 | 0 |
| Addressing Mode | 7-bit |
| Dual Address Mode | Disable |
| General Call Mode | Disable |
| No Stretch Mode | Disable |

5. `NVIC Settings` 中，I2C1 Event / Error Interrupt **先不勾选**。

### 4.3 为什么这样配置

| 设置 | 原因 |
|---|---|
| 100 kHz | AHT20、SSD1306 都能稳定支持；走线短、设备少，先求稳定而不是追求速度 |
| 7-bit 地址 | AHT20 与多数 OLED 均使用常规 7-bit I2C 地址 |
| Own Address = 0 | STM32 作为主机，不作为从机响应其他设备 |
| No Stretch Disable | 允许从设备在需要时拉低 SCL，兼容性更好 |
| 不开 I2C 中断 | 温湿度每秒读一次、OLED 半秒刷一次，HAL 轮询加超时已经足够，初学调试最简单 |

### 4.4 与 PCB 的关系

PCB 上已经有两个 4.7k 上拉电阻把 SCL、SDA 拉到 3.3V。I2C 的引脚本质是开漏输出，外部上拉电阻负责把总线恢复到高电平。

因此：

```text
不要把 PB6/PB7 配成普通 GPIO Output。
不要人为再加内部 Pull-up 代替外部 4.7k。
不要把 I2C 设备供电到 5V 再把信号直连 STM32。
```

### 4.5 配完后的验证目标

生成工程后先写一个 I2C 扫描小程序，不要立即写 AHT20 驱动。预期能扫描到：

```text
AHT20：通常为 0x38
OLED：通常为 0x3C 或 0x3D
```

地址必须以实际扫描结果为准。

---

## 5. 配置 USART1：STM32 与 ESP-01S 通信

### 5.1 作用

USART1 是 STM32 与 ESP-01S 之间的文本通信通道。STM32 通过它发送 `AT` 指令和 MQTT 配置，ESP 返回 `OK`、Wi-Fi 状态、MQTT 下发命令等内容。

### 5.2 操作

1. 左侧 `Connectivity` -> 点击 `USART1`。
2. Mode 选择 `Asynchronous`。
3. 确认引脚：

```text
PA9  = USART1_TX
PA10 = USART1_RX
```

4. 在 `Parameter Settings` 设置：

| 参数 | 值 | 原因 |
|---|---|---|
| Baud Rate | 115200 Bits/s | 常见 ESP-AT 默认波特率，后续必须与 ESP 实际一致 |
| Word Length | 8 Bits | ESP AT 常用 `8N1` 格式 |
| Parity | None | 对应 `8N1` 的 N |
| Stop Bits | 1 | 对应 `8N1` 的 1 |
| Data Direction | Receive and Transmit | 既要发 AT 指令，也要接收回复 |
| Hardware Flow Control | Disable | PCB 没有连接 CTS/RTS |
| Oversampling | 16（若界面显示） | 默认稳定配置 |

5. 打开 `NVIC Settings`，勾选 `USART1 global interrupt`。
6. `Preemption Priority` 和 `Sub Priority` **第一版保持 CubeMX 默认的 `0 / 0`，不用修改**。部分 STM32F1 / CubeMX 版本会把这两个单元格锁定或不提供细分修改，这不是错误。

### 5.3 为什么 UART 要开中断

ESP 的返回数据出现时间不固定：可能是 `OK`、`ERROR`、`WIFI CONNECTED` 或 MQTT 下发消息。如果主循环用阻塞接收等待，STM32 在等待期间可能错过按键、传感器和报警任务。

第一版推荐：

```text
HAL_UART_Receive_IT() 按字节接收
-> 放入软件环形缓冲区
-> esp8266.c 在主循环解析完整行或完整 MQTT 消息
```

这比一开始上 DMA 容易理解，也足以支持本项目。

> 本项目目前只有 SysTick 和 USART1 两类中断，UART 接收中断中只做“收一个字节、放入缓冲区”，执行时间很短。因此默认的 `0 / 0` 优先级不会造成实际问题。等未来加入 DMA、多个定时器和外部中断后，再统一设计中断优先级。

### 5.4 必须确认 ESP 实际波特率

`115200` 是推荐起点，不是保证值。ESP-01S 上电后，先用 USB-TTL 串口助手单独发送：

```text
AT
AT+GMR
```

若串口助手有乱码或无回复，先确认 ESP 的固件与波特率，再回 CubeMX 修改 USART1 波特率。不要把“ESP 没回复”误认为 STM32 程序错误。

---

## 6. 配置 TIM1_CH1：PA8 风扇 PWM

### 6.1 作用

PWM（脉冲宽度调制）是以固定频率快速开关 PA8，通过改变高电平所占比例来改变风扇平均功率。PA8 已接到 AO3400A MOS Gate，不需要新增引脚。

### 6.2 操作

1. 左侧 `Timers` -> 点击 `TIM1`。
2. `Clock Source` 选择 `Internal Clock`。
3. 在 `Channel1` 选择 `PWM Generation CH1`。
4. CubeMX 应分配：

```text
PA8 = TIM1_CH1
```

5. 在 `Parameter Settings` 填写：

| 参数 | 值 | 原因 |
|---|---:|---|
| Prescaler (PSC) | 0 | 不分频，保持 72 MHz 计数基准 |
| Counter Mode | Up | 最常用、最容易理解的 PWM 模式 |
| Counter Period (ARR) | 3599 | 与 PSC=0 共同得到 20 kHz |
| Clock Division | DIV1 | 不额外分频 |
| Repetition Counter | 0 | 本项目不使用高级重复计数 |
| Auto-reload preload | Enable | 更新占空比更平滑，避免周期中途突变 |
| Channel 1 Mode | PWM mode 1 | 计数值小于比较值时输出高电平 |
| Pulse (CCR1) | 0 | 上电先关闭风扇，避免突然启动 |
| CH Polarity | High | PA8 高电平使 MOS 导通，风扇获得电流 |

6. `NVIC Settings` 中，TIM1 更新中断和捕获比较中断都**不勾选**。

### 6.3 为什么不使用 GPIO 软件 PWM

软件 PWM 需要 CPU 频繁定时翻转 GPIO；一旦 I2C、UART 或 `HAL_Delay()` 占用时间，频率会抖动。TIM1 是硬件定时器，CPU 不干预也能持续输出稳定 PWM，因此适合风扇控制。

### 6.4 占空比如何理解

```text
Duty = CCR1 / (ARR + 1) x 100%

0%   -> CCR1 = 0
40%  -> CCR1 = 1440
60%  -> CCR1 = 2160
80%  -> CCR1 = 2880
100% -> CCR1 = 3600（或 ARR+1，HAL 实际使用时注意边界）
```

编写 `fan.c` 时会用类似逻辑设置比较寄存器：

```c
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, compare_value);
```

实际启动 PWM 的 HAL 调用需要在模块初始化后执行：

```c
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
```

这两行不是完整业务代码，只是说明 CubeMX 配置后会生成 `htim1` 这个定时器句柄。

### 6.5 风扇 PWM 的验证顺序

1. 先只测试 0% 和 100%。
2. 100% 能稳定转后，测试 80%、60%、40%。
3. 每次由停转启动时，程序先给 100% 约 500 ms，再切目标档位。
4. 若 40% 不转或异响，记录该风扇的最小可靠占空比，不要认为 PWM 配置一定有错。

---

## 7. 配置普通 GPIO：LED、蜂鸣器、按键

### 7.1 作用

GPIO 是普通输入输出引脚：输出用于控制 LED 和蜂鸣器，输入用于读取按键状态。

### 7.2 LED 与蜂鸣器输出

在 Pinout View 中分别点击 PA4、PA5、PB0，选择 `GPIO_Output`。进入 `System Core` -> `GPIO`，在 `GPIO Settings` 中设置。

| 引脚 | User Label | Mode | Output Type | Pull | Speed | 上电初值 | 原因 |
|---|---|---|---|---|---|---|---|
| PA4 | `LED_RED` | Output | Push-Pull | No pull | Low | Low | 高电平点亮红 LED；上电先关闭 |
| PA5 | `LED_GREEN` | Output | Push-Pull | No pull | Low | Low | 上电先关闭，系统完成自检后由软件决定是否点亮 |
| PB0 | `BUZZER_CTRL` | Output | Push-Pull | No pull | Low | Low | 高电平使 S8050 导通；防止上电误鸣 |

**为什么选 Low Speed：** LED 和有源蜂鸣器不是高速信号，低速 GPIO 足够且边沿较缓，有助于减小干扰。

> 如果上电实测发现 LED 或蜂鸣器逻辑反了，不要第一时间改 CubeMX 引脚。先检查 LED 方向、三极管电路和软件 `On/Off` 宏定义。

### 7.3 三个按键输入

在 Pinout View 点击 PB12、PB13、PB14，选择 `GPIO_Input`。在 GPIO 参数中设置：

| 引脚 | User Label | Mode | Pull | 外部中断 | 原因 |
|---|---|---|---|---|---|
| PB12 | `KEY1` | Input | Pull-up | 不开 | 按键按下接地，读取低电平 |
| PB13 | `KEY2` | Input | Pull-up | 不开 | 同上 |
| PB14 | `KEY3` | Input | Pull-up | 不开 | 同上 |

### 7.4 为什么不用 EXTI 按键中断

机械按键有抖动。若一按就触发多次中断，初学阶段很容易发生“按一次跳两页”。本工程用 `HAL_GetTick()` 每 10 ms 扫描一次按键，并连续稳定 30 ms 才确认按下；因此 GPIO 输入加上 Pull-up 即可。

将来熟悉中断后，可以改为 EXTI 唤醒 + 软件消抖，但不是 V1 的必要条件。

---

## 8. NVIC、DMA、Middleware 配置

### 8.1 NVIC 中断

| 中断 | 是否开启 | 原因 |
|---|---|---|
| SysTick | HAL 默认 | 提供 1 ms 系统时基 |
| USART1 global interrupt | 开启 | 非阻塞接收 ESP 数据 |
| I2C1 Event / Error | 不开 | V1 用轮询 + 超时，便于调试 |
| TIM1 Update / CC | 不开 | PWM 由硬件自动输出，不需要每个周期进中断 |
| EXTI 12/13/14 | 不开 | 按键用 10 ms 扫描消抖 |

### 8.2 DMA

本版全部不配置 DMA。

**原因：** I2C 数据量非常小；ESP 的 AT 数据先用 UART 接收中断和环形缓冲区即可。DMA 能提升效率，但首次项目容易引入“空闲中断、缓冲区长度、半传输回调”等额外问题。

### 8.3 Middleware

全部不选。

| 不选择的项目 | 原因 |
|---|---|
| FreeRTOS | 本项目任务少，用软定时任务即可 |
| FATFS | 没有 SD 卡 |
| USB Device | Type-C 仅供电 |
| LwIP | 网络由 ESP-01S 处理，不是 STM32 以太网 |

---

## 9. Project Manager 与代码生成

### 9.1 作用

确定 CubeMX 输出什么工程、在哪里保存、重新生成时是否保留你手写的代码。

### 9.2 操作

点击顶部 `Project Manager`，建议如下：

| 选项 | 推荐值 | 原因 |
|---|---|---|
| Project Name | `STM32_TEMP_FAN_CTRL` | 与 PCB 名称一致，方便管理 |
| Project Location | 自己固定的工程目录 | 不要放在临时目录或中文过深路径 |
| Toolchain / IDE | `MDK-ARM` | 对应 Keil |
| Firmware Package Name | STM32Cube FW_F1 | 选择已安装的官方 F1 HAL 包 |
| Generate peripheral initialization as a pair of `.c/.h` files | 勾选 | 生成 `gpio.c`、`i2c.c`、`usart.c`、`tim.c`，结构清楚 |
| Keep User Code when re-generating | 勾选 | 避免重新生成 CubeMX 代码时删除 `USER CODE` 区域内容 |
| Set all free pins as analog | 第一版不勾选 | PA0~PA3 是备用排针；先保持直观，后续确定不用再优化低功耗 |
| Generate under root | 保持默认 | 初学阶段路径简单 |

然后点击 `GENERATE CODE`。若询问是否打开工程，可选择 `Open Project`。

### 9.3 生成后应出现的初始化函数

在 `Core/Src/main.c` 内应能看到：

```c
MX_GPIO_Init();
MX_I2C1_Init();
MX_USART1_UART_Init();
MX_TIM1_Init();
```

若缺少其中一个，说明对应外设没有启用，回 CubeMX 修正后再生成。

---

## 10. CubeMX 配置完成后的总检查

在生成代码前，回到 `Pinout & Configuration` 全屏检查下表。每一项必须与 PCB 原理图一致。

| 引脚 | 必须显示 |
|---|---|
| PB6 | `I2C1_SCL` |
| PB7 | `I2C1_SDA` |
| PA9 | `USART1_TX` |
| PA10 | `USART1_RX` |
| PA8 | `TIM1_CH1` |
| PA4 | `GPIO_Output` / `LED_RED` |
| PA5 | `GPIO_Output` / `LED_GREEN` |
| PB0 | `GPIO_Output` / `BUZZER_CTRL` |
| PB12 | `GPIO_Input` / `KEY1` |
| PB13 | `GPIO_Input` / `KEY2` |
| PB14 | `GPIO_Input` / `KEY3` |
| PA13 | `SYS_SWDIO` |
| PA14 | `SYS_SWCLK` |
| PD0 / PD1 | 外部 HSE 功能占用 |

同时确认**没有**误启用：

```text
USB Device
ADC1
SPI1/SPI2
FreeRTOS
I2C2
USART2/USART3
PA8 普通 GPIO
PB6/PB7 普通 GPIO
```

---

## 11. 生成工程后的最小验证顺序

不要一生成工程就写全部功能。依次验证，哪一步失败就只查那一步。

### 11.1 第一步：时钟和下载

1. 用 ST-Link 连接 SWD：`3.3V、GND、SWDIO、SWCLK`。
2. BOOT0 保持正常启动位置（低电平）。
3. Keil 编译并下载空工程。
4. 程序不应反复复位。

### 11.2 第二步：GPIO 输出

先写最小 LED 测试，确认：

```text
PA4 高 -> 红灯亮
PA5 高 -> 绿灯亮
PB0 高 -> 蜂鸣器响
```

只在确认电路正确后再给蜂鸣器测试，避免在桌上持续鸣叫。

### 11.3 第三步：按键

读取 PB12、PB13、PB14 电平。预期：

```text
松开：GPIO_PIN_SET（高）
按下：GPIO_PIN_RESET（低）
```

若相反，先检查实物按键网络是否确实接地；软件可以调整“按下有效电平”宏。

### 11.4 第四步：I2C 扫描

扫描 PB6/PB7 上的设备，确认 AHT20 和 OLED 地址。没有扫描到时，检查供电、GND、排针顺序、I2C 上拉和模块电压。

### 11.5 第五步：PWM

启动 TIM1 PWM，依次输出 0% 和 100%。先确认风扇不会让 5V/3.3V 掉压导致 STM32 复位，再测试中间档位。

### 11.6 第六步：ESP 串口

先用 USB-TTL 单独确认 ESP 波特率和 AT 固件；再接回 PCB，由 STM32 USART1 发送 `AT\r\n`，期待收到 `OK`。

---

## 12. 常见错误与处理

| 现象 | 最可能的配置问题 | 检查方法 |
|---|---|---|
| Keil 下载不了 | SYS 没选 Serial Wire，或 SWD 接线错误 | 查看 PA13/PA14 是否被占用 |
| 程序一启动就卡住 | HSE 配置与晶振硬件不一致 | RCC 是否为 Crystal；检查晶振焊接 |
| ESP 串口乱码 | 时钟或波特率错误 | SYSCLK 是否 72MHz；USART 是否与 ESP 一致 |
| I2C 扫描不到设备 | PB6/PB7 不是 I2C1、供电/排针错误 | 检查 Pinout、3.3V、GND、模块引脚顺序 |
| 风扇上电就一直转 | TIM1 初始 Pulse 不为 0，或 MOS 电路问题 | TIM1 CCR1 初始值、PA8 是否选对 |
| 风扇不转 | PA8 未选 TIM1_CH1，未调用 PWM Start | 检查 CubeMX 和 `HAL_TIM_PWM_Start` |
| 按键一按多次 | 直接读取 GPIO 没有消抖 | 使用 10ms 扫描、30ms 稳定确认 |
| 蜂鸣器上电就响 | PB0 初始输出高或三极管接线问题 | GPIO 初始 Low；检查 PB0 实测 |
| CubeMX 重新生成后代码丢失 | 手写代码不在 USER CODE 区域或未勾 Keep User Code | 业务代码放入自建 `.c/.h` 文件 |

---

## 13. 你现在实际要做的顺序

1. 新建 `STM32F103C8Tx` 工程。
2. 配 `SYS -> Serial Wire`。
3. 配 `RCC -> HSE Crystal/Ceramic`。
4. 配 I2C1、USART1、TIM1 CH1。
5. 配 PA4、PA5、PB0、PB12、PB13、PB14 GPIO。
6. 在 Clock Configuration 配成 72 MHz。
7. 配 USART1 中断，其他 I2C/TIM/按键中断先不开。
8. 在 Project Manager 选 `MDK-ARM` 和保留用户代码。
9. 对照第 10 节总检查表。
10. 生成工程，先截图 Pinout 和 Clock Configuration 给我核对。

完成这十步，才进入 `LED -> 按键 -> I2C 扫描 -> AHT20 -> OLED -> 风扇 -> ESP8266` 的程序开发阶段。
