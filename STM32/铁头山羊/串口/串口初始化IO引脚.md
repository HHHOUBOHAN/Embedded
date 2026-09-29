# USART模块有那些引脚
1. 引脚介绍
```c
USARTx_TX    发送数据
USARTx_RX    接收数据
USARTx_CK    同步时钟
USARTx_CTS   清除发送，硬件流控
USARTx_RTS   请求发送，硬件流控
```
2. `STM32F407ZET6`串口常用引脚

| 串口     | TX   | RX   | 常见用途    |
| ------ | ---- | ---- | ------- |
| USART1 | PA9  | PA10 | 常用调试串口  |
| USART2 | PA2  | PA3  | 常用串口    |
| USART3 | PB10 | PB11 | 你板子例程常用 |
| UART4  | PA0  | PA1  | 普通串口    |
| UART5  | PC12 | PD2  | 普通串口    |
| USART6 | PC6  | PC7  | 高速/扩展串口 |
# 引脚分布表

| 外设     | TX                | RX                | CK                 | CTS         | RTS         |
| ------ | ----------------- | ----------------- | ------------------ | ----------- | ----------- |
| USART1 | PA9 / PB6         | PA10 / PB7        | PA8                | PA11        | PA12        |
| USART2 | PA2 / PD5         | PA3 / PD6         | PA4 / PD7          | PA0 / PD3   | PA1 / PD4   |
| USART3 | PB10 / PC10 / PD8 | PB11 / PC11 / PD9 | PB12 / PC12 / PD10 | PB13 / PD11 | PB14 / PD12 |
| UART4  | PA0 / PC10        | PA1 / PC11        | 无                  | 无           | 无           |
| UART5  | PC12              | PD2               | 无                  | 无           | 无           |
| USART6 | PC6 / PG14        | PC7 / PG9         | PC8                | PG15        | PG8         |
1. AF复用功能
	- USART1_TX：PA9 / PB6
	- USART1_RX：PA10 / PB7
2. 
```
USART1_TX
   |
   |----> PA9
   |
   |----> PB6
USART1_RX
   |
   |----> PA10
   |
   |----> PB7
```
- USART1_TX 输出到 PA9          USART1_TX 输出到 PB6
- 
  ```
	  方案 1：
	PA9  = USART1_TX
	PA10 = USART1_RX
	
	方案 2：
	PB6  = USART1_TX
	PB7  = USART1_RX
  ```
# 重映射表
1. USART 常用 AF 映射表

|外设|TX 可选引脚|RX 可选引脚|AF编号|
|---|---|---|---|
|USART1|PA9 / PB6|PA10 / PB7|AF7|
|USART2|PA2 / PD5|PA3 / PD6|AF7|
|USART3|PB10 / PC10 / PD8|PB11 / PC11 / PD9|AF7|
|UART4|PA0 / PC10|PA1 / PC11|AF8|
|UART5|PC12|PD2|AF8|
|USART6|PC6 / PG14|PC7 / PG9|AF8|
# IO配置表
1. 配置表：![[Pasted image 20260705141640.png]]
2. 全双工：数据可以**同时进行**双向传输，具有两条通信线路![[Pasted image 20260705141007.png]]
3. 半双工：数据传输可以沿两个方向，但需要**分时进行**，也只有一条通信线路![[Pasted image 20260705141040.png]]
4. 单工：数据传输仅能沿一个方向，不能实现反向传输，只有一条通信路线 ![[Pasted image 20260705141126.png]]
5. 同步：在标准的串口基础上，增加CK线（负责在两个设备之间传输**时钟信号**（用于同步））![[Pasted image 20260705141253.png]]

6. 硬件流控：在标准的串口基础上，增加CTS,RTS引脚                       ![[Pasted image 20260705141441.png]]


































