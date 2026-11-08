# IIC模块简介

# IO引脚的初始化
1. 进行重映射
2. 初始化 PB8 PB9 模式为复用输出开漏模式
--- 

## RCC使能
1. 原因：STM32 所有外设默认是**断电**的（省电设计）。你往一个没通电的外设寄存器里写数据，**写入会被硬件直接忽略**—— 读回全是 0，代码 "看起来没执行"。
2. 时钟是由时钟树一路分下来的
3. `RCC_xxxPeriphClockCmd` 就是打开这条通路上的**时钟门控开关**。**所有外设初始化第一句必然是开时钟**。
```
外部晶振 HSE(12M?) / 内部 HSI(16M)
   → PLL 倍频
   → SYSCLK 系统时钟
   → AHB 分频 → HCLK
       ├── AHB1 总线 → GPIOA/B/C/D...（GPIOB 在这）
       └── APB1 总线 → I2C1/2/3, USART2, TIM 等（I2C1 在这，最大 42MHz）
```
1. 代码：
```c
	//进行结构体定义
	GPIO_InitTypeDef GPIO_InitStruct;
	I2C_InitTypeDef I2C_InitStruct;

	
	
	
	//使能
	RCC_AHB1PeriphClockCmd(RCC_AHB1Periph_GPIOB,ENABLE);// PB8/PB9 所在端口
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1,ENABLE);// I2C1 挂在 APB1
	
	//引脚复用
	GPIO_PinAFConfig(GPIOB,GPIO_PinSource8,GPIO_AF_I2C1);
	GPIO_PinAFConfig(GPIOB,GPIO_PinSource9,GPIO_AF_I2C1);
	
	
	//GPIO模式配置
	GPIO_InitStruct.GPIO_Pin=GPIO_Pin_8 | GPIO_Pin_9;
	GPIO_InitStruct.GPIO_Mode=GPIO_Mode_AF;
	GPIO_InitStruct.GPIO_Speed=GPIO_Speed_50MHz;
	GPIO_InitStruct.GPIO_OType=GPIO_OType_OD;//开漏输出
	GPIO_InitStruct.GPIO_PuPd  = GPIO_PuPd_UP;//内部上拉
	GPIO_Init(GPIOB,&GPIO_InitStruct);
	
	//I2C模式配置
	I2C_InitStruct.I2C_Mode=I2C_Mode_I2C;//标准I2C模式
	I2C_InitStruct.I2C_DutyCycle=I2C_DutyCycle_2;
	I2C_InitStruct.I2C_OwnAddress1=0x00;
	I2C_InitStruct.I2C_Ack=I2C_Ack_Enable;
	I2C_InitStruct.I2C_ClockSpeed=100000;
	I2C_InitStruct.I2C_AcknowledgedAddress=I2C_AcknowledgedAddress_7bit;
	I2C_Init(I2C1,&I2C_InitStruct);
	
	//使能I2C1 
	I2C_Cmd(I2C1,ENABLE);

```

### ② 配复用：引脚怎么 "变成"I2C？
![[Pasted image 20260903205907.png]]
# 连接电路
# IIC的速度模式
1. 波特率=每秒钟传输的位数
2. 速度模式（常用单片机一般只支持标准模式和快速模式）

| 模式     | 缩写  | 英文全称            | 最高速率     |
| ------ | --- | --------------- | -------- |
| 标准模式   | Sm  | Standard Mode   | ≤100kbps |
| 快速模式   | Fm  | Fast Mode       | ≤400kbps |
| 快速增强模式 | Fm+ | Fast Mode Plus  | ≤1Mbps   |
| 高速模式   | HSm | High Speed Mode | ≤3.4Mbps |
| 超快模式   | UFm | Ultra Fast Mode | ≤5Mbps   |

# 时钟信号的占空比
1. 只有在快速模式下可以设置时钟信号的占空比（一个周期中高电压的占整个周期的比例）
2. 有两种占空比 (低电压：高电压):`2:1`和 `16：9`
3. 没有特殊要求的话，就是`2:1`的占空比                              ![[Pasted image 20260903215951.png]]

# IIC模块的初始化
1.IIC初始化接口                                                            ![[Pasted image 20260903220557.png]]
2. 

