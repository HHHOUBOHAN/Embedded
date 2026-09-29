# USART模块简介

1. STM32F407ZET6 芯片本身一共有 6 个串口外设
```
USART1
USART2
USART3
UART4
UART5
USART6
```
2. `USART`：支持同步/异步串口 
3. `UART`：只支持异步串口
4. USART 找到内部结构框图
# USART模块的基本用法
1. `USART` 的内部结构框图![[Pasted image 20260705112725.png]]
2. 发送流程
```
CPU 写入数据  
↓  
发送数据寄存器 TDR / DR  
↓  
发送移位寄存器  
↓  
按照波特率一位一位移出  
↓  
Tx 引脚输出高低电平波形  
↓  
对方 Rx 引脚接收
```
3. 接收流程
```
Rx 引脚收到外部波形
↓
USART 检测起始位
↓
按照波特率采样每一位
↓
接收移位寄存器组合成一个字节
↓
数据放入接收数据寄存器
↓
RXNE 置 1
↓
CPU 读取数据
↓
RXNE 清除
```
# 移位寄存器和串并转换
1. 串行/并行
	- 并行：多位数据同时传
		- 优点：一次传多位，理论上速度快
		- 缺点：需要很多根线，布线复杂，距离远了容易出问题，不同线路之间容易干扰
	- 串行：多位数据一位一位传
		- 优点：线少，布简线单，适合远距离通信，抗干扰更容易处理，成本低
		- 缺点：一次只传一位，单纯从“位数”看没有并行直观快，需要发送和接收双方约定速率，比如波特率
2. 串并转换![[Pasted image 20260705122254.png]]
3. 数据发送流程的串并转换（并转串）![[Pasted image 20260705122426.png]]
4. 数据接收流程的串并转换（串转并）![[Pasted image 20260705122917.png]]
# 数据帧格式的设置方法
1. 数据帧的格式![[Pasted image 20260705123017.png]]
2. 数据帧：起始位+数据位+停止位
	- 起始位：1位 **低电平**
	- 数据位：8~9位（数据位的最后一位可以作为校验位）**低位先发**
	- 停止位：0.5，1，1.5，2位（停止位越长，接收方有更多时间准备下一帧，但通信效率会稍微降低）
	- 校验方式：
		- 奇校验：加上校验位以后，整组数据里面 `1` 的个数要变成奇数
		- 偶校验：加上校验位以后，整组数据里面 `1` 的个数要变成偶数
		- 无校验：最常用
		- 校验位只能检查部分错误：可以发现一些数据传输异常，但不能保证发现所有错误，也不能自动修正错误
		- 例如：一位出错校验位能发现，两位出错，可能发现不了
# 波特率的设置方法
1. 波特率：：串口每秒传输多少个二进制位，单位是 bps，也就是 bit/s
2. 常用波特率：9600（每秒传输 9600 位），115200，921600 ![[Pasted image 20260705125154.png]]
3. 波特率寄存器：前十二位为整数部分，后四位为小数部分
# 编程接口
## USART_Init（初始化串口）
1. `USART_InitTypeDef USART_InitStruct;` //配置表
2. `USART_Init`
```c
void USART_Init(USART_TypeDef* USARTx, USART_InitTypeDef* USART_InitStruct);
//串口名称 初始化参数
```
2. SART_InitTypeDef 结构体成员
```c
struct USART_InitTypeDef{

    uint32_t USART_BaudRate;   // 波特率

    uint16_t USART_WordLength; // 数据位长度 - USART_WordLength_8b
                                               - USART_WordLength_9b


    uint16_t USART_StopBits;   // 停止位长度 - USART_StopBits_0_5
                                               - USART_StopBits_1
                                               - USART_StopBits_1_5
                                               - USART_StopBits_2


    uint16_t USART_Parity;     // 校验方式   - USART_Parity_No
                                               - USART_Parity_Even//偶校验
                                               - USART_Parity_Odd//奇校验


    uint16_t USART_Mode;       // 数据收发方向 - USART_Mode_Tx
                                                - USART_Mode_Rx
                                                - USART_Mode_Tx | USART_Mode_Rx
}
```































