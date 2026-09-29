# STM32F407 ADC + LCD + 蜂鸣器实验笔记

## 1. 项目目标

本项目使用星辰派 STM32F407ZET6 开发板完成一个 ADC 综合实验：

```text
板载电位器 -> PC3 / ADC1_IN13 -> ADC采集
ADC值 -> USART3 串口打印
ADC值 -> FSMC LCD 显示
ADC值/电压 -> PC2 控制蜂鸣器
```

最终现象：

```text
拧动蓝色电位器
串口打印 ADC 值和电压
LCD 显示 ADC 值、电压值、蜂鸣器状态
电压超过阈值后蜂鸣器响
```

## 2. 硬件对应关系

### 2.1 电位器为什么对应 ADC1_IN13

开发板原理图中，板载电位器连接到：

```text
PC3
```

STM32F407 中：

```text
PC3 = ADC1_IN13
```

所以 CubeMX 中选择：

```text
ADC1 -> IN13
```

这不是随便选的，而是硬件原理图决定的。

### 2.2 板子上哪个是电位器

实物图中，三个黄色按键下面的蓝色小方块就是电位器。

```text
不是 RES
不是黄色按键
是蓝色带一字槽的小旋钮
```

用小螺丝刀旋转它，PC3 上的电压会在 0V 到 3.3V 之间变化。

### 2.3 蜂鸣器引脚

原理图中蜂鸣器由 PC2 控制：

```text
PC2 -> R51 -> Q2 三极管 -> BUZZER
```

所以代码里应写：

```c
GPIOC, GPIO_PIN_2
```

注意不要写成：

```c
GPIO_PIN_12
```

之前蜂鸣器不响的原因就是把 PC2 写成了 PC12。

## 3. CubeMX 配置

本项目建议基于已有 LCD 例程修改：

```text
10.2.FSMC-LCD屏幕+触摸屏
```

因为 FSMC LCD 配置较复杂，例程里已经配置好了。

### 3.1 不需要改的部分

```text
FSMC
LCD 引脚
USART3
时钟树
RCC
```

### 3.2 需要新增的部分

```text
ADC1 -> IN13
PC2 -> GPIO_Output
```

如果给 PC2 起 Label：

```text
BEEP
```

CubeMX 会在 `main.h` 里生成：

```c
#define BEEP_Pin GPIO_PIN_2
#define BEEP_GPIO_Port GPIOC
```

之后可以写：

```c
HAL_GPIO_WritePin(BEEP_GPIO_Port, BEEP_Pin, GPIO_PIN_SET);
```

如果不起名，也可以直接写：

```c
HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_SET);
```

## 4. RCC 配置有什么用

RCC 是 Reset and Clock Control，也就是复位与时钟控制。

STM32 中 CPU 和外设都需要时钟：

```text
ADC 需要 ADC 时钟
USART 需要串口时钟
GPIO 需要 GPIO 时钟
FSMC LCD 需要 FSMC 和 GPIO 时钟
TIM 定时器需要 TIM 时钟
```

RCC 的作用就是配置：

```text
时钟来源
系统频率
总线频率
外设时钟开关
```

本项目基于 LCD 例程修改，原例程 LCD 和串口都能正常工作，所以 RCC 和时钟树保持默认即可。

结论：

```text
本项目不需要改 RCC
不需要改时钟树
只加 ADC1_IN13 和 PC2 GPIO_Output
```

## 5. Project Name 为什么改不了

打开已有 `.ioc` 工程时，CubeMX 中部分项目设置会变灰，不能直接修改。

常见不能改的项：

```text
Project Name
Project Location
Application Structure
```

这是正常现象。

工程名叫 `TEST` 不影响编译、下载和运行。

如果一定要改项目名，推荐：

```text
1. 复制整个工程文件夹
2. 重命名文件夹
3. 重命名 .ioc 文件
4. 修改 .ioc 里的 ProjectManager.ProjectName
5. 再重新 Generate Code
```

## 6. 代码放置位置

CubeMX 工程里，自己写的代码尽量放在：

```c
/* USER CODE BEGIN xxx */
/* USER CODE END xxx */
```

中间。

原因是 CubeMX 重新生成代码时，会保留这些区域。

常用位置：

```text
USER CODE BEGIN Includes   放头文件
USER CODE BEGIN PV         放全局变量
USER CODE BEGIN 0          放自定义函数，例如 fputc
USER CODE BEGIN 2          放初始化后只执行一次的代码
while(1)                   放循环执行的代码
```

本项目中：

```text
lcd.h / stdio.h / delay_us.h  放 Includes
adc_value / voltage / buffer  放 PV
fputc                         放 USER CODE BEGIN 0
LCD_Init 和标题显示            放 USER CODE BEGIN 2
ADC采集、LCD显示、蜂鸣器控制    放 while(1)
```

## 7. 头文件

在：

```c
/* USER CODE BEGIN Includes */
```

中添加：

```c
#include "lcd.h"
#include "stdio.h"
#include "delay_us.h"
```

作用：

```text
lcd.h       使用 LCD_ShowString、LCD_Fill 等函数
stdio.h     使用 printf、sprintf
delay_us.h  使用 delay_us 微秒延时
```

如果没有 `delay_us.h`，可以在函数声明区写：

```c
void delay_us(uint32_t nus);
```

并确认 `delay_us.c` 已加入 Keil 工程。

## 8. 全局变量

在：

```c
/* USER CODE BEGIN PV */
```

中添加：

```c
uint16_t adc_value;
float voltage;
char buffer[50];
```

含义：

```text
adc_value  保存 ADC 原始值，范围 0~4095
voltage    保存换算后的电压值
buffer     临时字符串，用于串口打印和 LCD 显示
```

`buffer` 可以重复使用，因为每次 `sprintf()` 都会覆盖旧内容。

## 9. printf 重定向

STM32 上 `printf` 默认不会自动从串口输出，需要重定向。

在：

```c
/* USER CODE BEGIN 0 */
```

中添加：

```c
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart3, (uint8_t *)&ch, 1, 50);
    return ch;
}
```

含义：

```text
printf 输出的每一个字符
都会通过 USART3 发出去
```

`HAL_UART_Transmit` 参数解释：

```text
&huart3         使用 USART3
(uint8_t *)&ch 发送字符地址
1               发送 1 个字节
50              最多等待 50ms
```

## 10. 如何查看 HAL 函数

HAL 函数存在工程的 HAL 库中：

```text
Drivers/STM32F4xx_HAL_Driver/Inc/
Drivers/STM32F4xx_HAL_Driver/Src/
```

例如：

```text
stm32f4xx_hal_uart.h / .c   串口函数
stm32f4xx_hal_adc.h / .c    ADC 函数
stm32f4xx_hal_gpio.h / .c   GPIO 函数
```

在 Keil 中查看函数：

```text
Ctrl + 鼠标点击函数名
右键 -> Go To Definition
全局搜索函数名
```

例如查看：

```c
HAL_UART_Transmit
```

通常会跳转到：

```text
stm32f4xx_hal_uart.c
```

## 11. LCD 初始化

在：

```c
/* USER CODE BEGIN 2 */
```

中添加：

```c
LCD_Init();
LCD_Fill(0, 0, lcddev.width, lcddev.height, WHITE);

LCD_ShowString(20, 20, (uint8_t *)"ADC LCD BEEP", RED, WHITE, 24, 0);
LCD_ShowString(20, 70, (uint8_t *)"ADC:", BLACK, WHITE, 16, 0);
LCD_ShowString(20, 100, (uint8_t *)"Voltage:", BLACK, WHITE, 16, 0);
LCD_ShowString(20, 130, (uint8_t *)"BEEP:", BLACK, WHITE, 16, 0);

printf("ADC LCD BEEP Start\r\n");
```

这些内容只需要执行一次，所以放在进入 `while(1)` 之前。

## 12. ADC 常用流程

ADC 单次采集常用代码：

```c
HAL_ADC_Start(&hadc1);

HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY);

adc_value = HAL_ADC_GetValue(&hadc1);

voltage = adc_value * 3.3f / 4095.0f;

sprintf(buffer, "ADC:%d Voltage:%.2fV\r\n", adc_value, voltage);

printf("%s", buffer);

HAL_ADC_Stop(&hadc1);
```

理解流程：

```text
启动 ADC
  -> 等待转换完成
  -> 读取 ADC 数值
  -> 换算成电压
  -> 拼接成字符串
  -> 串口打印
  -> 停止 ADC
```

## 13. LCD 显示 ADC 和电压

在 ADC 读取成功后添加：

```c
LCD_Fill(100, 70, 240, 150, WHITE);

sprintf(buffer, "%d", adc_value);
LCD_ShowString(100, 70, (uint8_t *)buffer, BLUE, WHITE, 16, 0);

sprintf(buffer, "%.2f V", voltage);
LCD_ShowString(100, 100, (uint8_t *)buffer, BLUE, WHITE, 16, 0);
```

`LCD_Fill()` 是局部清屏，防止旧数字残留。

例如：

```text
旧值 4095
新值 980
不清屏可能显示成 9805
```

## 14. `2.5f` 后面的 f 是什么意思

C 语言中，小数默认是 `double` 类型。

```c
2.5
```

是 `double`。

```c
2.5f
```

是 `float`。

因为本项目中：

```c
float voltage;
```

所以推荐写：

```c
if (voltage > 2.5f)
```

同理：

```c
voltage = adc_value * 3.3f / 4095.0f;
```

这里的 `3.3f` 和 `4095.0f` 也是 float。

## 15. HAL_GPIO_WritePin 的 SET 和 RESET

函数：

```c
HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_SET);
```

含义：

```text
让 PC2 输出高电平
```

函数：

```c
HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_RESET);
```

含义：

```text
让 PC2 输出低电平
```

注意：

```text
SET 不等于一定打开设备
RESET 不等于一定关闭设备
```

SET/RESET 只代表高低电平。

设备响不响，要看硬件是高电平有效还是低电平有效。

本开发板蜂鸣器大体是：

```text
PC2 高低变化形成方波 -> 蜂鸣器响
PC2 保持低电平 -> 蜂鸣器停
```

## 16. `"ON "` 为什么有空格

如果之前 LCD 显示：

```text
OFF
```

然后新显示：

```text
ON
```

因为 `ON` 比 `OFF` 短，可能残留：

```text
ONF
```

所以写：

```c
LCD_ShowString(100, 130, (uint8_t *)"ON ", RED, WHITE, 16, 0);
```

`ON` 后面加一个空格，用来覆盖旧字符。

## 17. 蜂鸣器为什么固定高电平不持续响

调试时发现，固定写：

```c
HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_SET);
```

蜂鸣器只响一下或声音很小。

原因是这个蜂鸣器需要方波驱动。

固定高电平：

```text
蜂鸣片只被推到一个位置，不持续振动
```

快速高低变化：

```text
蜂鸣片持续振动，产生声音
```

原理图中蜂鸣器标注：

```text
2.7kHz
```

说明它在 2.7kHz 左右声音更明显。

## 18. 方波驱动蜂鸣器

代码：

```c
if (voltage > 2.5f)
{
    LCD_ShowString(100, 130, (uint8_t *)"ON ", RED, WHITE, 16, 0);

    for (int i = 0; i < 800; i++)
    {
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_SET);
        delay_us(170);
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_RESET);
        delay_us(170);
    }
}
else
{
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_RESET);
    LCD_ShowString(100, 130, (uint8_t *)"OFF ", GREEN, WHITE, 16, 0);
}
```

这段代码制造了方波：

```text
高 170us
低 170us
高 170us
低 170us
...
```

一个周期：

```text
170us + 170us = 340us
```

频率约：

```text
1 / 0.00034 ≈ 2941Hz
```

接近 2.7kHz，所以声音更明显。

`for` 循环的作用是让这个高低变化持续执行很多次，所以蜂鸣器能持续响。

## 19. 声音为什么还是偏小

原理图中蜂鸣器供电是：

```text
VCC_3V3
```

不是 5V。

很多蜂鸣器在 3.3V 下声音会偏小，这是硬件限制。

软件只能尽量调整方波频率，例如尝试：

```c
delay_us(160);
delay_us(170);
delay_us(180);
delay_us(185);
```

哪个最响就用哪个。

如果要明显更大声，需要外接 5V 有源蜂鸣器模块或更大的报警器。

## 20. 完整核心逻辑参考

```c
while (1)
{
    HAL_ADC_Start(&hadc1);

    if (HAL_ADC_PollForConversion(&hadc1, HAL_MAX_DELAY) == HAL_OK)
    {
        adc_value = HAL_ADC_GetValue(&hadc1);
        voltage = adc_value * 3.3f / 4095.0f;

        sprintf(buffer, "ADC:%d Voltage:%.2fV\r\n", adc_value, voltage);
        printf("%s", buffer);

        LCD_Fill(100, 70, 240, 150, WHITE);

        sprintf(buffer, "%d", adc_value);
        LCD_ShowString(100, 70, (uint8_t *)buffer, BLUE, WHITE, 16, 0);

        sprintf(buffer, "%.2f V", voltage);
        LCD_ShowString(100, 100, (uint8_t *)buffer, BLUE, WHITE, 16, 0);

        if (voltage > 2.5f)
        {
            LCD_ShowString(100, 130, (uint8_t *)"ON ", RED, WHITE, 16, 0);

            for (int i = 0; i < 800; i++)
            {
                HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_SET);
                delay_us(170);
                HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_RESET);
                delay_us(170);
            }
        }
        else
        {
            HAL_GPIO_WritePin(GPIOC, GPIO_PIN_2, GPIO_PIN_RESET);
            LCD_ShowString(100, 130, (uint8_t *)"OFF ", GREEN, WHITE, 16, 0);
        }
    }

    HAL_ADC_Stop(&hadc1);
}
```

## 21. 最终总结

本实验完整体现了 STM32 的常见应用流程：

```text
模拟输入
  -> ADC 采集
  -> 数据处理
  -> 串口调试
  -> LCD 人机显示
  -> GPIO 控制外设
```

要记住的核心点：

```text
PC3 是电位器输入，所以选 ADC1_IN13
PC2 是蜂鸣器控制，所以用 GPIOC GPIO_PIN_2
FSMC 和时钟树基于 LCD 例程，不需要乱改
SET/RESET 只是高低电平，不是绝对开关
蜂鸣器需要方波，接近 2.7kHz 声音更明显
```
