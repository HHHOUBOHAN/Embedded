
## 1. LCD 显示的基本流程

在 STM32 中使用 LCD 屏幕，一般流程是：

```c
LCD_Init();                                           // 初始化 LCD
LCD_Fill(0, 0, lcddev.width, lcddev.height, WHITE);   // 清屏
LCD_ShowString(...);                                  // 显示文字
```

可以理解成：

先初始化屏幕，再清空屏幕背景，最后在指定坐标显示内容。

---

## 2. LCD_Init() 函数

```c
LCD_Init();
```

作用是初始化 LCD 屏幕。

它一般会完成这些事情：

1. 初始化 LCD 控制芯片
    
2. 设置屏幕方向
    
3. 设置屏幕宽度和高度
    
4. 让 LCD 进入可以正常显示的状态
    

使用 LCD 之前，必须先调用：

```c
LCD_Init();
```

否则后面的 `LCD_Fill()`、`LCD_ShowString()` 可能无法正常显示。

---

## 3. LCD_Fill() 函数

```c
LCD_Fill(x1, y1, x2, y2, color);
```

作用是填充一块矩形区域的颜色。

参数含义：

|参数|含义|
|---|---|
|`x1`|起始横坐标|
|`y1`|起始纵坐标|
|`x2`|结束横坐标|
|`y2`|结束纵坐标|
|`color`|填充颜色|

例如：

```c
LCD_Fill(0, 0, lcddev.width, lcddev.height, WHITE);
```

意思是：

从屏幕左上角 `(0, 0)` 开始，一直填充到整个屏幕的宽度和高度，颜色为白色。

这句代码通常用来清屏。

---

## 4. lcddev.width 和 lcddev.height

`lcddev` 一般是 LCD 驱动文件中定义的结构体变量。

它里面保存 LCD 屏幕的信息。

常用成员：

```c
lcddev.width
lcddev.height
```

含义：

|成员|含义|
|---|---|
|`lcddev.width`|LCD 屏幕宽度|
|`lcddev.height`|LCD 屏幕高度|

例如一块屏幕是 `320 × 240`，那么可能是：

```text
lcddev.width  = 320
lcddev.height = 240
```

使用 `lcddev.width` 和 `lcddev.height` 的好处是：不用自己把屏幕尺寸写死，代码适配性更好。

---

## 5. LCD 坐标系统

LCD 屏幕坐标一般从左上角开始。

```text
(0,0) ------------------> x 方向增大
  |
  |
  |
  v
y 方向增大
```

也就是说：

- `x` 越大，显示位置越靠右
    
- `y` 越大，显示位置越靠下
    

例如：

```c
LCD_ShowString(20, 20, ...);
```

意思是从屏幕左上角开始，向右 20 像素，向下 20 像素，然后开始显示文字。

---

## 6. LCD_ShowString() 函数

```c
LCD_ShowString(x, y, string, fontColor, backColor, size, mode);
```

作用是在 LCD 屏幕指定位置显示字符串。

参数含义：

|参数|含义|
|---|---|
|`x`|字符串起始横坐标|
|`y`|字符串起始纵坐标|
|`string`|要显示的字符串|
|`fontColor`|字体颜色|
|`backColor`|背景颜色|
|`size`|字体大小|
|`mode`|显示模式|

例如：

```c
LCD_ShowString(20, 20, (uint8_t *)"ADC LCD BEEP", RED, WHITE, 24, 0);
```

意思是：

在 `x = 20`，`y = 20` 的位置显示字符串 `ADC LCD BEEP`，文字颜色是红色，背景颜色是白色，字体大小是 24。

---

## 7. 为什么字符串前面要写 `(uint8_t *)`

例如：

```c
(uint8_t *)"ADC LCD BEEP"
```

因为 `LCD_ShowString()` 函数里面的字符串参数可能定义成：

```c
uint8_t *p
```

但是普通字符串：

```c
"ADC LCD BEEP"
```

默认是 `char *` 类型。

所以要写：

```c
(uint8_t *)"ADC LCD BEEP"
```

意思是：把这个字符串地址强制转换成 `uint8_t *` 类型。

本质上显示内容没有变，还是显示：

```text
ADC LCD BEEP
```

---

## 8. 常见颜色宏

LCD 驱动中通常会提前定义一些颜色宏。

例如：

```c
#define WHITE  0xFFFF
#define BLACK  0x0000
#define RED    0xF800
#define GREEN  0x07E0
#define BLUE   0x001F
```

常用颜色：

|宏|颜色|
|---|---|
|`WHITE`|白色|
|`BLACK`|黑色|
|`RED`|红色|
|`GREEN`|绿色|
|`BLUE`|蓝色|

LCD 常用颜色格式一般是 `RGB565`，初学阶段先记住颜色宏怎么用即可。

---

## 9. 示例代码

```c
LCD_Init();
LCD_Fill(0, 0, lcddev.width, lcddev.height, WHITE);

LCD_ShowString(20, 20, (uint8_t *)"ADC LCD BEEP", RED, WHITE, 24, 0);
LCD_ShowString(20, 70, (uint8_t *)"ADC:", BLACK, WHITE, 16, 0);
LCD_ShowString(20, 100, (uint8_t *)"Voltage:", BLACK, WHITE, 16, 0);
LCD_ShowString(20, 130, (uint8_t *)"BEEP:", BLACK, WHITE, 16, 0);

printf("ADC LCD BEEP Start\r\n");
```

---

## 10. 示例代码解释

### 初始化 LCD

```c
LCD_Init();
```

初始化 LCD，让屏幕进入可以显示的状态。

---

### 清屏

```c
LCD_Fill(0, 0, lcddev.width, lcddev.height, WHITE);
```

把整个屏幕填充成白色，相当于清屏。

---

### 显示标题

```c
LCD_ShowString(20, 20, (uint8_t *)"ADC LCD BEEP", RED, WHITE, 24, 0);
```

在屏幕 `(20, 20)` 的位置显示红色标题：

```text
ADC LCD BEEP
```

---

### 显示 ADC 标签

```c
LCD_ShowString(20, 70, (uint8_t *)"ADC:", BLACK, WHITE, 16, 0);
```

在屏幕 `(20, 70)` 的位置显示：

```text
ADC:
```

后面可以在右侧显示 ADC 原始采样值。

例如：

```text
ADC: 2048
```

---

### 显示电压标签

```c
LCD_ShowString(20, 100, (uint8_t *)"Voltage:", BLACK, WHITE, 16, 0);
```

在屏幕 `(20, 100)` 的位置显示：

```text
Voltage:
```

后面可以显示换算后的电压值。

例如：

```text
Voltage: 1.65V
```

---

### 显示蜂鸣器标签

```c
LCD_ShowString(20, 130, (uint8_t *)"BEEP:", BLACK, WHITE, 16, 0);
```

在屏幕 `(20, 130)` 的位置显示：

```text
BEEP:
```

后面可以显示蜂鸣器状态。

例如：

```text
BEEP: ON
BEEP: OFF
```

---

### 串口打印

```c
printf("ADC LCD BEEP Start\r\n");
```

这句不是显示在 LCD 屏幕上，而是通过串口打印到串口助手。

前提是已经写好了 `printf` 重定向函数：

```c
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart3, (uint8_t *)&ch, 1, 50);
    return ch;
}
```

---

## 11. 最终屏幕显示效果

大概效果是：

```text
ADC LCD BEEP

ADC:

Voltage:

BEEP:
```

对应坐标：

|内容|坐标|
|---|---|
|`ADC LCD BEEP`|`(20, 20)`|
|`ADC:`|`(20, 70)`|
|`Voltage:`|`(20, 100)`|
|`BEEP:`|`(20, 130)`|

---

## 12. 代码应该放在哪里

这段 LCD 初始化和固定文字显示代码，建议放在：

```c
/* USER CODE BEGIN 2 */

LCD_Init();
LCD_Fill(0, 0, lcddev.width, lcddev.height, WHITE);

LCD_ShowString(20, 20, (uint8_t *)"ADC LCD BEEP", RED, WHITE, 24, 0);
LCD_ShowString(20, 70, (uint8_t *)"ADC:", BLACK, WHITE, 16, 0);
LCD_ShowString(20, 100, (uint8_t *)"Voltage:", BLACK, WHITE, 16, 0);
LCD_ShowString(20, 130, (uint8_t *)"BEEP:", BLACK, WHITE, 16, 0);

printf("ADC LCD BEEP Start\r\n");

/* USER CODE END 2 */
```

因为 `USER CODE BEGIN 2` 在外设初始化完成之后，`while(1)` 死循环开始之前。

所以它适合放只需要开机执行一次的代码，比如：

- LCD 初始化
    
- LCD 清屏
    
- 固定标题显示
    
- 串口启动提示
    

---

## 13. 动态数据应该放在哪里

ADC 数值、电压值、蜂鸣器状态这些会不断变化，所以应该放在 `while(1)` 里面循环刷新。

例如：

```c
while (1)
{
    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 100);
    adc_value = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);

    voltage = adc_value * 3.3f / 4095.0f;

    sprintf(buffer, "%d    ", adc_value);
    LCD_ShowString(80, 70, (uint8_t *)buffer, BLUE, WHITE, 16, 0);

    sprintf(buffer, "%.2fV    ", voltage);
    LCD_ShowString(100, 100, (uint8_t *)buffer, BLUE, WHITE, 16, 0);

    HAL_Delay(500);
}
```

固定文字只显示一次，动态数据循环刷新。

---

## 14. 动态刷新时的注意点

如果数字从长变短，可能会残留旧字符。

例如第一次显示：

```text
ADC: 4095
```

第二次显示：

```text
ADC: 888
```

如果不清除旧字符，屏幕可能变成：

```text
ADC: 8885
```

因为旧的 `5` 没有被覆盖。

解决方法一：字符串后面加空格。

```c
sprintf(buffer, "%d    ", adc_value);
LCD_ShowString(80, 70, (uint8_t *)buffer, BLUE, WHITE, 16, 0);
```

解决方法二：先清除局部区域。

```c
LCD_Fill(80, 70, 160, 90, WHITE);
LCD_ShowString(80, 70, (uint8_t *)buffer, BLUE, WHITE, 16, 0);
```

初学阶段推荐第一种，简单稳定。

---

## 15. 固定显示和动态显示的区别

|类型|放置位置|例子|
|---|---|---|
|固定显示|`USER CODE BEGIN 2`|标题、标签、界面框架|
|动态显示|`while(1)`|ADC 值、电压值、状态值|

固定显示：

```c
LCD_ShowString(20, 70, (uint8_t *)"ADC:", BLACK, WHITE, 16, 0);
```

动态显示：

```c
sprintf(buffer, "%d    ", adc_value);
LCD_ShowString(80, 70, (uint8_t *)buffer, BLUE, WHITE, 16, 0);
```

---

## 16. 常见错误

### 没有先初始化 LCD

错误写法：

```c
LCD_ShowString(20, 20, (uint8_t *)"Hello", RED, WHITE, 16, 0);
LCD_Init();
```

正确写法：

```c
LCD_Init();
LCD_ShowString(20, 20, (uint8_t *)"Hello", RED, WHITE, 16, 0);
```

---

### 坐标超出屏幕范围

如果屏幕宽度是 320，高度是 240，那么坐标范围大概是：

```text
x: 0 ~ 319
y: 0 ~ 239
```

坐标超出范围，文字可能显示不出来。

---

### 动态刷新不清除旧数据

问题：

```text
旧字符残留，显示错乱
```

解决：

```c
sprintf(buffer, "%d    ", adc_value);
```

或者：

```c
LCD_Fill(80, 70, 160, 90, WHITE);
```

---

### 把 printf 当成 LCD 显示

```c
printf("hello\r\n");
```

这个是串口输出，不是 LCD 显示。

LCD 显示要用：

```c
LCD_ShowString(...);
```

---

## 17. 总结

LCD 显示常用三步：

```c
LCD_Init();
LCD_Fill(0, 0, lcddev.width, lcddev.height, WHITE);
LCD_ShowString(...);
```

记忆方式：

|函数|作用|
|---|---|
|`LCD_Init()`|初始化屏幕|
|`LCD_Fill()`|填充颜色 / 清屏|
|`LCD_ShowString()`|显示字符串|

一句话总结：

先初始化 LCD，再清屏，然后用 `LCD_ShowString()` 在指定坐标显示文字；固定界面只显示一次，变化数据放到 `while(1)` 循环里不断刷新。