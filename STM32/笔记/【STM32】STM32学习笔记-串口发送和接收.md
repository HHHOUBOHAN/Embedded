---
title: "【STM32】STM32学习笔记-串口发送和接收"
author: "Print World"
date: "已于 2025-01-10 16:16:02 修改"
source: "https://blog.csdn.net/u010249597/article/details/135489406"
---

### 00. 目录
 





 

#### 文章目录

 

- 
  - [00. 目录](#00__0)
  - [01. 串口简介](#01__3)
  - [02. 串口相关API](#02_API_13)
  - 
    - [2.1 USART_Init](#21_USART_Init_15)
    - [2.2 USART_InitTypeDef](#22_USART_InitTypeDef_44)
    - [2.3 USART_Cmd](#23_USART_Cmd_156)
    - [2.4 USART_SendData](#24_USART_SendData_182)
    - [2.5 USART_ReceiveData](#25_USART_ReceiveData_209)
  - [03. 串口发送接线图](#03__241)
  - [04. USB转串口模块](#04_USB_251)
  - [05. 串口发送程序示例](#05__265)
  - [06. 串口发送支持printf](#06_printf_371)
  - [07. 串口发送支持printf_v2](#07_printf_v2_572)
  - [08. 串口发送和接收接线图](#08__783)
  - [09. 串口接收示例(轮询模式)](#09__791)
  - [10. 串口接收示例(中断模式)](#10__978)
  - [11. 程序下载](#11__1219)
  - [12. 附录](#12__1229)





 


### 01. 串口简介
 


串口通讯(Serial Communication)是一种设备间非常常用的串行通讯方式，因为它简单便捷，因此大部分电子设备都支持该通讯方式， 电子工程师在调试设备时也经常使用该通讯方式输出调试信息。
 


在计算机科学里，大部分复杂的问题都可以通过分层来简化。如芯片被分为内核层和片上外设；STM32标准库则是在寄存器与用户代码之间的软件层。 对于通讯协议，我们也以分层的方式来理解，最基本的是把它分为和协议层。物理层规定通讯系统中具有机械、电子功能部分的特性， 确保原始数据在物理媒体的传输。协议层主要规定通讯逻辑，统一收发双方的数据打包、解包标准。 简单来说物理层规定我们用嘴巴还是用肢体来交流，协议层则规定我们用中文还是英文来交流。
 


### 02. 串口相关API
 


#### 2.1 USART_Init
 


```c
/**
  * @brief  Initializes the USARTx peripheral according to the specified
  *         parameters in the USART_InitStruct .
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @param  USART_InitStruct: pointer to a USART_InitTypeDef structure
  *         that contains the configuration information for the specified USART 
  *         peripheral.
  * @retval None
  */
void USART_Init(USART_TypeDef* USARTx, USART_InitTypeDef* USART_InitStruct)
功能:
	根据 USART_InitStruct 中指定的参数初始化外设 USARTx 寄存器
参数:
   USARTx：x 可以是 1，2 或者 3，来选择 USART 外设
   USART_InitStruct：指向结构 USART_InitTypeDef 的指针，包含了外设 USART 的配置信息。
返回值:
	无      
   
```
 


#### 2.2 USART_InitTypeDef
 


```c
/** 
  * @brief  USART Init Structure definition  
  */ 
typedef struct
{
  uint32_t USART_BaudRate;            /*!< This member configures the USART communication baud rate.
                                           The baud rate is computed using the following formula:
                                            - IntegerDivider = ((PCLKx) / (16 * (USART_InitStruct->USART_BaudRate)))
                                            - FractionalDivider = ((IntegerDivider - ((u32) IntegerDivider)) * 16) + 0.5 */

  uint16_t USART_WordLength;          /*!< Specifies the number of data bits transmitted or received in a frame.
                                           This parameter can be a value of @ref USART_Word_Length */

  uint16_t USART_StopBits;            /*!< Specifies the number of stop bits transmitted.
                                           This parameter can be a value of @ref USART_Stop_Bits */

  uint16_t USART_Parity;              /*!< Specifies the parity mode.
                                           This parameter can be a value of @ref USART_Parity
                                           @note When parity is enabled, the computed parity is inserted
                                                 at the MSB position of the transmitted data (9th bit when
                                                 the word length is set to 9 data bits; 8th bit when the
                                                 word length is set to 8 data bits). */
 
  uint16_t USART_Mode;                /*!< Specifies wether the Receive or Transmit mode is enabled or disabled.
                                           This parameter can be a value of @ref USART_Mode */

  uint16_t USART_HardwareFlowControl; /*!< Specifies wether the hardware flow control mode is enabled
                                           or disabled.
                                           This parameter can be a value of @ref USART_Hardware_Flow_Control */
} USART_InitTypeDef;
```
 


**USART_WordLength**
 


```c
/** @defgroup USART_Word_Length 
  * @{
  */ 
  
#define USART_WordLength_8b                  ((uint16_t)0x0000)
#define USART_WordLength_9b                  ((uint16_t)0x1000)
```
 


**USART_StopBits**
 


```c
/** @defgroup USART_Stop_Bits 
  * @{
  */ 
  
#define USART_StopBits_1                     ((uint16_t)0x0000)
#define USART_StopBits_0_5                   ((uint16_t)0x1000)
#define USART_StopBits_2                     ((uint16_t)0x2000)
#define USART_StopBits_1_5                   ((uint16_t)0x3000)
```
 


**USART_Parity**
 


```c
/** @defgroup USART_Parity 
  * @{
  */ 
  
#define USART_Parity_No                      ((uint16_t)0x0000)
#define USART_Parity_Even                    ((uint16_t)0x0400)
#define USART_Parity_Odd                     ((uint16_t)0x0600) 
```
 


**USART_Mode**
 


```c
/** @defgroup USART_Mode 
  * @{
  */ 
  
#define USART_Mode_Rx                        ((uint16_t)0x0004)
#define USART_Mode_Tx                        ((uint16_t)0x0008)
```
 


**USART_HardwareFlowControl**
 


```c
/** @defgroup USART_Hardware_Flow_Control 
  * @{
  */ 
#define USART_HardwareFlowControl_None       ((uint16_t)0x0000)
#define USART_HardwareFlowControl_RTS        ((uint16_t)0x0100)
#define USART_HardwareFlowControl_CTS        ((uint16_t)0x0200)
#define USART_HardwareFlowControl_RTS_CTS    ((uint16_t)0x0300)
```
 


#### 2.3 USART_Cmd
 


```c
/**
  * @brief  Enables or disables the specified USART peripheral.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *         This parameter can be one of the following values:
  *           USART1, USART2, USART3, UART4 or UART5.
  * @param  NewState: new state of the USARTx peripheral.
  *         This parameter can be: ENABLE or DISABLE.
  * @retval None
  */
void USART_Cmd(USART_TypeDef* USARTx, FunctionalState NewState)
功能:
	使能或者失能 USART 外设
参数:
   USARTx：x 可以是 1，2 或者 3，来选择 USART 外设
   NewState: 外设 USARTx 的新状态这个参数可以取：ENABLE 或者 DISABLE
返回值:
	无           
```
 


#### 2.4 USART_SendData
 


```c
/**
  * @brief  Transmits single data through the USARTx peripheral.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @param  Data: the data to transmit.
  * @retval None
  */
void USART_SendData(USART_TypeDef* USARTx, uint16_t Data)
功能:
   通过外设 USARTx 发送单个数据
参数:
   USARTx：x 可以是 1，2 或者 3，来选择 USART 外设
   Data: 待发送的数据
返回值:
	无      
    
    
```
 


#### 2.5 USART_ReceiveData
 


```c
/**
  * @brief  Returns the most recent received data by the USARTx peripheral.
  * @param  USARTx: Select the USART or the UART peripheral. 
  *   This parameter can be one of the following values:
  *   USART1, USART2, USART3, UART4 or UART5.
  * @retval The received data.
  */
uint16_t USART_ReceiveData(USART_TypeDef* USARTx)
功能:
   返回 USARTx 最近接收到的数据
参数:
   USARTx：x 可以是 1，2 或者 3，来选择 USART 外设
返回值:
	接收到的字      
    
```
 


### 03. 串口发送接线图
 


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/bd9722a74e9023c42d8e88fc3a4f8aad.png#pic_center)
 


### 04. 模块
 


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8ca134b8698a172194b9904e853cf4fc.png#pic_center)
 


### 05. 串口发送程序示例
 


.h
 


```c
#ifndef __UART_H__
#define __UART_H__

#include "stm32f10x.h"           

void uart_init(void);

void uart_send_byte(uint8_t byte);


#endif /**/
```
 


uart.c
 


```c
#include "uart.h"

void uart_init(void)
{
	GPIO_InitTypeDef GPIO_InitStruct;
	USART_InitTypeDef USART_InitStruct;
	
	
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

	//GPIO初始化  PA9 TX
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);

	USART_InitStruct.USART_BaudRate = 9600;
	USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
	USART_InitStruct.USART_Mode = USART_Mode_Tx;
	USART_InitStruct.USART_Parity = USART_Parity_No;
	USART_InitStruct.USART_StopBits = USART_StopBits_1;
	USART_InitStruct.USART_WordLength = USART_WordLength_8b;
	
	USART_Init(USART1, &USART_InitStruct);
	
	USART_Cmd(USART1, ENABLE);
}

void uart_send_byte(uint8_t byte)
{
	USART_SendData(USART1, byte);
	
	while(USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
}
```
 


main.c
 


```c
#include "stm32f10x.h"

#include "delay.h"
#include "oled.h"
#include "uart.h"

 int main(void)
 {	 
	 
	 //初始化
	 OLED_Init();
	 
	 uart_init();

	 //显示一个字符
	 OLED_ShowChar(1, 1, 'A');

	 uart_send_byte(0x41);
	 
	 while(1)
	 {
		 
	 }
	 
	 return 0;
 }

```
 


运行结果
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/737a674d2c91856ba1971bd96c1479fc.png#pic_center)
 


### 06. 串口发送支持
 


配置：
 


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/be558f3cbffddf6b8e488cc5ff094097.png#pic_center)
 


uart.h
 


```c
#ifndef __UART_H__
#define __UART_H__

#include "stm32f10x.h"           

void uart_init(void);

void uart_send_byte(uint8_t byte);

void uart_send_array(uint8_t *arr, uint16_t len);


void uart_send_string(char *str);

void uart_send_number(uint32_t num, uint8_t len);

#endif /**/


```
 


uart.c
 


```c
#include "uart.h"

#include <stdio.h>

void uart_init(void)
{
	GPIO_InitTypeDef GPIO_InitStruct;
	USART_InitTypeDef USART_InitStruct;
	
	
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

	//GPIO初始化  PA9 TX
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);

	USART_InitStruct.USART_BaudRate = 9600;
	USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
	USART_InitStruct.USART_Mode = USART_Mode_Tx;
	USART_InitStruct.USART_Parity = USART_Parity_No;
	USART_InitStruct.USART_StopBits = USART_StopBits_1;
	USART_InitStruct.USART_WordLength = USART_WordLength_8b;
	
	USART_Init(USART1, &USART_InitStruct);
	
	USART_Cmd(USART1, ENABLE);
}

void uart_send_byte(uint8_t byte)
{
	USART_SendData(USART1, byte);
	
	while(USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
}


void uart_send_array(uint8_t *arr, uint16_t len)
{
	uint16_t i;
	for (i = 0; i < len; i++)
	{
		uart_send_byte(arr[i]);
	}
}


void uart_send_string(char *str)
{
	uint16_t i = 0;
	
	while(*(str + i) != '\0')
	{
		uart_send_byte(str[i]);
		i++;
	}
}

//x的y次方
uint32_t uart_pow(uint32_t x, uint32_t y)
{
	uint32_t result = 1;
	
	while(y)
	{
		result *= x;
		y--;
	}
	
	return result;
}

void uart_send_number(uint32_t num, uint8_t len)
{
	uint8_t i;
	for (i = 0; i < len; i++)
	{
		uart_send_byte(num / uart_pow(10, len - i - 1) % 10 + '0');
	}
	
}

int fputc(int ch, FILE *fp)
{
	uart_send_byte(ch);
	
	return ch;
}

```
 


main.c
 


```c
#include "stm32f10x.h"
#include <stdio.h>
#include "delay.h"
#include "oled.h"
#include "uart.h"

 int main(void)
 {	
	 
	 uint8_t arr[] = {0x42, 0x43, 0x44, 0x45, 0x46};

	 //初始化
	 OLED_Init();
	 
	 uart_init();

	 //显示一个字符
	 OLED_ShowChar(1, 1, 'A');

#if 0
	 uart_send_byte('B');
	 
	 //发送数组
	 uart_send_array(arr, 5);
	 
	 //发送字符串
	 uart_send_string("hello world\r\n");
	 uart_send_string("1234567890\r\n");

	 uart_send_number(1234, 4);
#endif

	printf("num = %d\r\n", 6666);
	
	 while(1)
	 {
		 
	 }
	 
	 return 0;
 }


```
 


运行结果
 ![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/7ba8ff67061b0a481f2283a245a9edab.png#pic_center)
 


### 07. 串口发送支持printf_v2
 


uart.h
 


```c
#ifndef __UART_H__
#define __UART_H__

#include "stm32f10x.h"           

void uart_init(void);

void uart_send_byte(uint8_t byte);

void uart_send_array(uint8_t *arr, uint16_t len);


void uart_send_string(char *str);

void uart_send_number(uint32_t num, uint8_t len);

void uart_printf(char *format, ...);

#endif /**/
```
 


uart.c
 


```c
#include "uart.h"

#include <stdio.h>
#include <stdarg.h>

void uart_init(void)
{
	GPIO_InitTypeDef GPIO_InitStruct;
	USART_InitTypeDef USART_InitStruct;
	
	
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

	//GPIO初始化  PA9 TX
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);

	USART_InitStruct.USART_BaudRate = 9600;
	USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
	USART_InitStruct.USART_Mode = USART_Mode_Tx;
	USART_InitStruct.USART_Parity = USART_Parity_No;
	USART_InitStruct.USART_StopBits = USART_StopBits_1;
	USART_InitStruct.USART_WordLength = USART_WordLength_8b;
	
	USART_Init(USART1, &USART_InitStruct);
	
	USART_Cmd(USART1, ENABLE);
}

void uart_send_byte(uint8_t byte)
{
	USART_SendData(USART1, byte);
	
	while(USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
}


void uart_send_array(uint8_t *arr, uint16_t len)
{
	uint16_t i;
	for (i = 0; i < len; i++)
	{
		uart_send_byte(arr[i]);
	}
}


void uart_send_string(char *str)
{
	uint16_t i = 0;
	
	while(*(str + i) != '\0')
	{
		uart_send_byte(str[i]);
		i++;
	}
}

//x的y次方
uint32_t uart_pow(uint32_t x, uint32_t y)
{
	uint32_t result = 1;
	
	while(y)
	{
		result *= x;
		y--;
	}
	
	return result;
}

void uart_send_number(uint32_t num, uint8_t len)
{
	uint8_t i;
	for (i = 0; i < len; i++)
	{
		uart_send_byte(num / uart_pow(10, len - i - 1) % 10 + '0');
	}
	
}

int fputc(int ch, FILE *fp)
{
	uart_send_byte(ch);
	
	return ch;
}


void uart_printf(char *format, ...)
{
	char str[128];
	
	va_list arg;
	va_start(arg, format);
	vsprintf(str, format, arg);
	va_end(arg);
	
	uart_send_string(str);
}

```
 


main.c
 


```c
#include "stm32f10x.h"
#include <stdio.h>
#include "delay.h"
#include "oled.h"
#include "uart.h"

 int main(void)
 {	
	char string[100];
	 uint8_t arr[] = {0x42, 0x43, 0x44, 0x45, 0x46};

	 //初始化
	 OLED_Init();
	 
	 uart_init();

	 //显示一个字符
	 OLED_ShowChar(1, 1, 'A');

#if 0
	 uart_send_byte('B');
	 
	 //发送数组
	 uart_send_array(arr, 5);
	 
	 //发送字符串
	 uart_send_string("hello world\r\n");
	 uart_send_string("1234567890\r\n");

	 uart_send_number(1234, 4);


	printf("num = %d\r\n", 6666);
#endif


	sprintf(string, "\r\nnum=%d", 3333);
	uart_send_string(string);


	uart_printf("\r\nnum = %d\r\n", 4444);
	uart_printf("\r\n");

	 while(1)
	 {
		 
	 }
	 
	 return 0;
 }
```
 


测试结果
 


```shell

num=3333
num = 4444


```
 


### 08. 串口发送和接收接线图
 


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/fad801838aa5cdca750b0bcd07984b60.png#pic_center)
 


### 09. 串口接收示例(轮询模式)
 


uart.h
 


```c
#ifndef __UART_H__
#define __UART_H__

#include "stm32f10x.h"           

void uart_init(void);

void uart_send_byte(uint8_t byte);

void uart_send_array(uint8_t *arr, uint16_t len);


void uart_send_string(char *str);

void uart_send_number(uint32_t num, uint8_t len);

void uart_printf(char *format, ...);

#endif /**/
```
 


uart.c
 


```c
#include "uart.h"

#include <stdio.h>
#include <stdarg.h>

void uart_init(void)
{
	GPIO_InitTypeDef GPIO_InitStruct;
	USART_InitTypeDef USART_InitStruct;
	
	
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

	//GPIO初始化  PA9 TX
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);
	
	//GPIO初始化  PA10 RX
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_10;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);

	USART_InitStruct.USART_BaudRate = 9600;
	USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
	USART_InitStruct.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
	USART_InitStruct.USART_Parity = USART_Parity_No;
	USART_InitStruct.USART_StopBits = USART_StopBits_1;
	USART_InitStruct.USART_WordLength = USART_WordLength_8b;
	
	USART_Init(USART1, &USART_InitStruct);
	
	USART_Cmd(USART1, ENABLE);
}

void uart_send_byte(uint8_t byte)
{
	USART_SendData(USART1, byte);
	
	while(USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
}


void uart_send_array(uint8_t *arr, uint16_t len)
{
	uint16_t i;
	for (i = 0; i < len; i++)
	{
		uart_send_byte(arr[i]);
	}
}


void uart_send_string(char *str)
{
	uint16_t i = 0;
	
	while(*(str + i) != '\0')
	{
		uart_send_byte(str[i]);
		i++;
	}
}

//x的y次方
uint32_t uart_pow(uint32_t x, uint32_t y)
{
	uint32_t result = 1;
	
	while(y)
	{
		result *= x;
		y--;
	}
	
	return result;
}

void uart_send_number(uint32_t num, uint8_t len)
{
	uint8_t i;
	for (i = 0; i < len; i++)
	{
		uart_send_byte(num / uart_pow(10, len - i - 1) % 10 + '0');
	}
	
}

int fputc(int ch, FILE *fp)
{
	uart_send_byte(ch);
	
	return ch;
}


void uart_printf(char *format, ...)
{
	char str[128];
	
	va_list arg;
	va_start(arg, format);
	vsprintf(str, format, arg);
	va_end(arg);
	
	uart_send_string(str);
}

```
 


main.c
 


```c
#include "stm32f10x.h"
#include <stdio.h>
#include "delay.h"
#include "oled.h"
#include "uart.h"

 int main(void)
 {	
	 uint16_t data = 0;
	 
	 OLED_Init();
	 
	 uart_init();

	 OLED_ShowChar(1, 1, 'A');


	 while(1)
	 {
		 if (USART_GetFlagStatus(USART1, USART_FLAG_RXNE) == SET)
		 {
			 data = USART_ReceiveData(USART1);
			 
			 OLED_ShowHexNum(1, 1, data, 2);
		 }
		 
	 }
	 
	 return 0;
 }
```
 


### 10. 串口接收示例(中断模式)
 


uart.h
 


```c
#ifndef __UART_H__
#define __UART_H__

#include "stm32f10x.h"           

void uart_init(void);

void uart_send_byte(uint8_t byte);

void uart_send_array(uint8_t *arr, uint16_t len);


void uart_send_string(char *str);

void uart_send_number(uint32_t num, uint8_t len);

void uart_printf(char *format, ...);

uint8_t uart_getRxFlag(void);


uint8_t uart_getRxData(void);


#endif /**/


```
 


uart.c
 


```c
#include "uart.h"

#include <stdio.h>
#include <stdarg.h>

uint8_t recvData;
uint8_t recvFlag;

void uart_init(void)
{
	GPIO_InitTypeDef GPIO_InitStruct;
	USART_InitTypeDef USART_InitStruct;
	NVIC_InitTypeDef NVIC_InitStruct;
	
	
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

	//GPIO初始化  PA9 TX
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);
	
	//GPIO初始化  PA10 RX
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_10;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);

	USART_InitStruct.USART_BaudRate = 9600;
	USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
	USART_InitStruct.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
	USART_InitStruct.USART_Parity = USART_Parity_No;
	USART_InitStruct.USART_StopBits = USART_StopBits_1;
	USART_InitStruct.USART_WordLength = USART_WordLength_8b;
	
	USART_Init(USART1, &USART_InitStruct);
	
	//设置串口中断
	USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);
	
	//设置中断分组
	NVIC_InitStruct.NVIC_IRQChannel = USART1_IRQn;
	NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;
	NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1;
	NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;
	NVIC_Init(&NVIC_InitStruct);
	
	USART_Cmd(USART1, ENABLE);
}

void uart_send_byte(uint8_t byte)
{
	USART_SendData(USART1, byte);
	
	while(USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
}


void uart_send_array(uint8_t *arr, uint16_t len)
{
	uint16_t i;
	for (i = 0; i < len; i++)
	{
		uart_send_byte(arr[i]);
	}
}


void uart_send_string(char *str)
{
	uint16_t i = 0;
	
	while(*(str + i) != '\0')
	{
		uart_send_byte(str[i]);
		i++;
	}
}

//x的y次方
uint32_t uart_pow(uint32_t x, uint32_t y)
{
	uint32_t result = 1;
	
	while(y)
	{
		result *= x;
		y--;
	}
	
	return result;
}

void uart_send_number(uint32_t num, uint8_t len)
{
	uint8_t i;
	for (i = 0; i < len; i++)
	{
		uart_send_byte(num / uart_pow(10, len - i - 1) % 10 + '0');
	}
	
}

int fputc(int ch, FILE *fp)
{
	uart_send_byte(ch);
	
	return ch;
}


void uart_printf(char *format, ...)
{
	char str[128];
	
	va_list arg;
	va_start(arg, format);
	vsprintf(str, format, arg);
	va_end(arg);
	
	uart_send_string(str);
}

void USART1_IRQHandler(void)
{
	if (USART_GetITStatus(USART1, USART_IT_RXNE) == SET)
	{
		recvData = USART_ReceiveData(USART1);
		recvFlag = 1;
	
		USART_ClearITPendingBit(USART1, USART_IT_RXNE);
	}

}

uint8_t uart_getRxFlag(void)
{
	if (1 == recvFlag)
	{
		recvFlag = 0;
		return 1;
	}
	return 0;
}


uint8_t uart_getRxData(void)
{
	return recvData;
}
```
 


main.c
 


```c
#include "stm32f10x.h"
#include <stdio.h>
#include "delay.h"
#include "oled.h"
#include "uart.h"


 int main(void)
 {	
	 uint16_t data = 0;
	 
	 OLED_Init();
	 
	 uart_init();
	 
	 //中断分组
	 NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

	 OLED_ShowChar(1, 1, 'A');


	 while(1)
	 {
		if (1 == uart_getRxFlag())
		{
			data = uart_getRxData();
			uart_send_byte(data);
			OLED_ShowHexNum(1, 1, data, 2);
		}
		 
	 }
	 
	 return 0;
 }


 
```
 


### 11. 程序下载
 


[22-UART发送.rar](https://download.csdn.net/download/u010249597/88721355)
 


[23-UART发送和接收.rar](https://download.csdn.net/download/u010249597/88721356)
 


### 12. 附录
 


参考: [【STM32】江科大STM32学习笔记汇总](https://blog.csdn.net/u010249597/article/details/134762513)

---
> 原文链接: [【STM32】STM32学习笔记-串口发送和接收](https://blog.csdn.net/u010249597/article/details/135489406)
> 作者: Print World
