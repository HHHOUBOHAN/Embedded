IIC实验--读取AHT10温湿度数据

本篇主要讲解了如何使用STM32CubeMX配置星辰派实现对AHT10温湿度的读取。

一.实验目的

掌握 STM32CubeMX 配置硬件 I2C 外设（传输速度、引脚、地址格式等）、串口及系统时钟的方法，理解 I2C 通信协议的核心逻辑；

理解 AHT10 温湿度传感器的工作原理，掌握其初始化、触发测量、数据读取与校准状态判断的编程实现；

掌握温湿度原始数据到实际值的转换方法，完成基于 I2C 的 AHT10 驱动开发及串口打印温湿度数据的完整功能；

熟悉 STM32 工程编译、下载调试流程，验证温湿度读取功能的实际效果。

二.准备工作

硬件准备

星辰派STM32F407ZET6开发板

AHT10温湿度模块

开发板正面左下方板载了一个AHT10温湿度读取模块，相关原理图如下：

￼

软件准备

完成STM32CubeMx安装

参考Keil 编辑器MDK开发环境配置

三.实验过程

STM32CubeMX配置参数介绍

相关配置介绍

￼

1、NVIC Settings：中断配置，主要开启中断、配置中断优先级

2、DMA Settings：DMA配置，开启DMA通道，便于I2C数据传输

3、GPIO Settings：GPIO配置，可以修改GPIO配置，别名等等，一般配置默认

4、Parameter Settings：参数配置，I2C模式，传输速度等配置

5、User Constants：用户常量，常量进行宏定义定义

中断配置

￼

1、I2C事件中断：主要是启用I2C发送/接收中断

2、I2C错误中断：在I2C传输出现错误时会调用中断函数

DMA配置

￼

Add新增和Delete删除可以对DMA进行管理

GPIO配置

￼

该页面主要展示了I2C所使用到的GPIO引脚，可以看到相关配置，默认即可，修改配置可能会导致无法正常使用I2C

I2C参数配置

1、Master Features主机功能组中，

￼

第一个参数为I2C的传输速度，有两个模式可以选择：

Standard Mode普通模式：最大速度100KHz

Fast Mode快速模式：最大速度400KHz

￼

￼

第二个参数时钟速度决定了I2C的传输速度，普通模式速度最大100KHz，快速模式最大可达400KHz

￼

在快速模式下的占空比选择中，您可以选择以下两种配置：

Tlow/Thigh = 2：这种配置使得SCL时钟的低电平时间是高电平时间的两倍。

Tlow/Thigh=16/9：这种配置使得低电平时间与高电平时间的比例为16:9。

这两种占空比选择会影响数据采样，但在大多数情况下，选择其中一种并不会有显著差别，除非对时序要求非常严格。

￼

2、SlaveFeatures从属功能组中，从设备参数。

Clock No Stretch Mode，禁止时钟延长。设置为 Disabled 表示允许时钟延长。

PrimaryAddress Length selection,设备主地址长度。可选7-bit 或10-bit,这里选择 7-bit

Dual Address Acknowledge,双地址确认。从设备可以有两个地址,如果设置为 Enabled,还会出现一个Secondary slave address参数，用于设置从设备副地址。

Primary slave address，从设备主地址。设置从设备主地址，作为I2C从设备时才需要设置。

General Call address detection，广播呼叫检测。设置为 Disabled 表示禁止广播呼叫，不对地址 0x00应答;否则，就是允许广播呼叫，对地址0x00应答。

用户常量参数

￼

这个功能列表新增变量可直接帮我们宏定义变量

生成MDK工程

配置串口

STM32F407ZET6开发板板载USB转TTL模拟串口，使用的是串口3，USB->TTL原理图如下：

￼

右边引脚图找到PC10和PC11，分别选中USART3_TX和USART3_RX，接下来配置串口3参数，波特率、数据位、停止位、校验位我们默认即可。

￼

配置硬件I2C

根据下面原理图，AHT10温湿度传感器使用的是PF0和PF1引脚，该引脚为I2C2的SDA和SCL引脚

￼

接下来配置I2C2，相关参数默认即可

￼

编译测试

编写用户代码

AHT10温湿度模块驱动编写

确定IIC器件地址

根据AHT10的Datasheet可知，AHT10是7位IIC地址，设备地址为0x38，可知AHT10的读地址为0x71 ，写地址为0x70，Datasheet中还提供了，初始化命令0xE1，触发测量命令0xAC和软复位命令0xBA：

￼

首先在AHT10.c中编写AHT10相关宏定义

// 定义AHT10命令 #define AHT10_INIT 0xE1// 1110 0001 #define AHT10_REST 0xBA// 1011 1010 #define AHT10_TRIG 0xAC// 1010 1100 // 定义AHT10地址 #define AHT10_Read 0x71	// 0x38<<1 | 1 #define AHT10_Write 0x70// 0x38<<1 | 0

基于HAL库提供的硬件IIC操作函数，编写AHT10的一些底层函数，如下。

触发温湿度测量

根据AHT10的数据手册可知，AHT10触发测量的流程如下：

￼

编写函数如下：

/** * @brief AHT10触发测量 * @param void * @retval void */ void Aht10_Trig(void) { 	uint8_t date[3]; 	date[0]=AHT10_TRIG; 	date[1]=0x33; 	date[2]=0x00; 	HAL_I2C_Master_Transmit(&hi2c2,AHT10_Write,date,3,0xffff); }

读取温湿度数据

根据AHT10的数据手册可知，AHT10读取温湿度数据的流程如下：

￼

数据手册中标注了 ：传感器在采集时需要时间,主机发出测量指令（0xAC） 后,延时75毫秒以上再读取转换后的数据并判断返回的状态位是否正常。若状态比特位[Bit7]为0代表数据可正常 读取,为1时传感器为忙状态,主机需要等待数据处理完成。

状态位码表：

￼

由IIC总线上获取到的相对湿度信号和相对温度信号需要通过公式进行转换，转换完毕后获取正确的相对温度和相对湿度：

￼

在AHT10.h中构建结构体：

//AHT10温湿度结构体 typedef struct{ 	float humidity;		//相对温度 	float temperature;	//相对湿度 }AHT_data;

编写函数如下：

/** * @brief AHT10 设备读取 相对湿度和温度 * @param void * @retval AHT_data 结构体 有数据代表读取数据正常; NULL-读取设备失败，设备一直处于忙状态，不能获取数据 */ AHT_data Aht10_getdata(void) { 	AHT_data sensor_data; 	uint8_t data[6]; 	uint8_t busy,cal; 	 	// 初始化返回值为空 	sensor_data.humidity = NULL; sensor_data.temperature = NULL; 	 	// 1 发送测量触发命令 	Aht10_Trig(); 	 	// 2 等待测量完成 	HAL_Delay(100);	 	 	// 3 读取AHT10传输的6字节数据 	HAL_I2C_Master_Receive(&hi2c2,AHT10_Read,data,6,0xffff); 	// 4 取出状态位 	busy=(data[0]>>7)&0x01;// 状态标志位 	cal=(data[0]>>3)&0x01;// 校准标志位 	 	// 5 检查校准状态 	if(cal==0){ 		 // 传感器未校准，复位传感器 		Aht10_Rst(); 		return sensor_data; 	} 	 	// 6 检查数据是否有效 	if(busy==0){// 数据有效 		 		// 进行数据转换 		uint32_t temp,humi; 		 		// 取出数据 		humi=(data[1]<<12)|(data[2]<<4)|((data[3]>>4)&0x0f); 		temp=((data[3]& 0x0F)<<16)|(data[4]<<8)|(data[5]); 		 		// 转换为实际温度和湿度值 		sensor_data.humidity=(humi*100.0)/1048576; 		sensor_data.temperature=((float)temp/1048576)*200-50; 		 		// 返回包含有效数据的结构体 		return sensor_data; 	}else 		// 数据无效 		return sensor_data; }

主函数编写

在main.c中编写以下代码：

int main(void) { HAL_Init(); SystemClock_Config(); MX_GPIO_Init(); MX_USART3_UART_Init(); MX_I2C2_Init(); //AHT10初始化 	Aht10_init(); 	printf("START\n"); 	AHT_data temp; while (1) { 	 //获取温湿度数据 	 temp=Aht10_getdata(); 	 //判断是否读取成功 	 if(temp.humidity!=NULL){ 		 //成功打印读取到的温湿度数据 		printf("H:%.1f,T:%.1f\n",temp.humidity,temp.temperature); 	 } 	 HAL_Delay(2000); } }

四.运行结果展示

下载运行后，打开串口助手，看到如下打印：

￼

至此，我们已经学会如何使用硬件IIC接口读取AHT10温湿度。






目录

IIC实验--读取AHT10温湿度数据

一.实验目的

二.准备工作

三.实验过程

四.运行结果展示

© 2025 河南敏纳数字科技有限公司 版权所有 |￼豫公

---
