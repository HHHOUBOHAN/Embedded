# 1.1 逐次逼近型ADC

## 1.1.1 ADC是什么
1. 一种将模拟信号转换为数字信号的电子器件或电路模块（电压信号转换数字量）                                                                                   ![image-1793765442329.png](attachments/image-1793765442329.png)
2. 模拟信号&&数字信号                                     ![image-1793765542870.png](attachments/image-1793765542870.png)
## 1.1.2 单片机里的ADC模块

![[image-1793765542870.png]]
1. 内部结构框图（ADC1,ADC2）: 十二位逐次逼近型ADC![image-1793765801843.png](attachments/image-1793765801843.png)
## 1.1.3 采样深度的概念
1. 采样深度：用多少二进制数表示转换结果，不同的采样深度导致其量化误差不同![image-1793766002220.png](attachments/image-1793766002220.png)
## 1.1.4 逐次性逼近型ADC简介
1. ADC 的转换原理（通过不同方案实现模拟量到数字量的转换）：![image-1793766217181.png](attachments/image-1793766217181.png)
2. 逐次逼近型ADC（天平称重）：通过不断尝试、比较和调整，逐步逼近输入电压，最终确定对应的数字量![image-1793766315405.png](attachments/image-1793766315405.png)
3. 内部结构框图：模拟信号输入（重物）→采样保持电路（托盘）→比较器（天平比较机构：两个输入：正输入，负输入，比较器的作用就是对这两个输入进行比较）→结果寄存器（四位二进制组成，即这个采样深度等于四）→电压发生器（通过结果寄存器的值来产生电压）![image-1793766451181.png](attachments/image-1793766451181.png)
## 1.1.5 采样保持电路
1. 采样保持电路：
	- 采样：将模拟信号每一个时间点的值采集
	- 保持：由于转换的时候结果寄存器匹配需要时间，所以在这个时间段，希望采集的值不变，所以加入一个电容
2. 工作原理：待测的模拟信号输入，采样（开关闭合，模拟信号就会给这个电容充电，充电完成后，电容两端电压就等于模拟信号两端的电压），保持（保持电容两端电压不变（断开开关，同时比较器正负输入端有虚短虚短的特性，这样就保证电容无法放电，电容两端电压保持不变）） 采样（充电）保持（断电）![image-1793772050052.png](attachments/image-1793772050052.png)
## 1.1.6 例子
1. 工作原理：模拟信号→数字信号，  闭合开关，电压输入，给电容充电，电容两端电压等于模拟信号两端电压，结果寄存器四位（采样深度等于四，把0~3.3V分成了16），向结构寄存器写0，然后给B3写1，转换后为1.76V，然后后面以此类推进行对比，最终极端得到最终的数字值![image-1793772398761.png](attachments/image-1793772398761.png)

# 1.2 ADC模块的结构框图
## 1.2.1 框图
![image-1793791954288.png](attachments/image-1793791954288.png)
![image-1793791965942.png](attachments/image-1793791965942.png)
## 1.2.2 ADC的多路复用
1. 多路复用:一个ADC同时转换多路模拟信号 ![image-1793791975966.png](attachments/image-1793791975966.png)
2. 多路复用工作原理: 给每一路模拟信号都有一个开关，闭合一个开关，然后进行转换，转换后断开开关，继续下一路转换，依次转换，就可以实现多路复用的原理 ![image-1793791984689.png](attachments/image-1793791984689.png)![image-1793791996269.png](attachments/image-1793791996269.png)![image-1793792023333.png](attachments/image-1793792023333.png)
	 - ADC总共有十个普通通道和两个特殊通道温度和参考电压
	 - stm32f103c8t6单片机没有PC的四个引脚
## 1.2.3 常规序列
1. 多路复用的两边各有两个序列，上面的是常规序列，下面的是常规序列 ![image-1793792123677.png](attachments/image-1793792123677.png)
2. 外部触发信息，常规序列本身，DR寄存器，保存转换的结果 ![image-1793792131940.png](attachments/image-1793792131940.png)
3. 常规序列:ADC模块的计划![image-1793792142360.png](attachments/image-1793792142360.png)
4. 外部触发信号:每当出现一个上升沿，就会吧常规序列执行一遍
5. 复用器左面有好多信号，有八种不同信号，前六种和定时器有关，第七种来自EXTI的11，第八种是软件启动
6. 工作流程:向常规序列中写入计划，左面选择定时器的TRGO，
7. 定时器TRGO的工作原理: 通过预分频器和自动重装寄存器分频，从而得到一次update事件要多长时间，然后设置从模式控制器，设置成update模式，右面update事件每发生一次，TRGO也发出一次信号 ![image-1793792170153.png](attachments/image-1793792170153.png)
8. 每出现一个脉冲就会触发一次常规序列![image-1793792179472.png](attachments/image-1793792179472.png)
## 1.2.4 注入序列
1. 结构框图 ![image-1793792195477.png](attachments/image-1793792195477.png)
    - 通道更少只有四个计划，常规序列序列通道是16
    - 每一个通道都有单独的结果寄存器，四个寄存器，每一个通道都有一个单独的寄存器，读取更方便
    - 优先级更高
## 1.2.5 例子
1. 注入序列优先级高:常规序列执行时，遇到了注入序列就会先执行注入序列，然后执行常规序列![image-1793792204000.png](attachments/image-1793792204000.png)
# 1.3 采样时间和转换时间
> 采样时间:开关闭合的时间长度
> 转换时间:对采样点进行转换所需要的时间
![image-1793787482738.png](attachments/image-1793787482738.png)
## 1.3.1 ADC时钟频率
1. 输入到ADC的频率不能超过14MHz  六分频和八分频符合要求 挂载在APB2上![image-1793787490901.png](attachments/image-1793787490901.png)
## 1.3.2 转换时间的计算方法
1. 结果寄存器有十二个比特位，每一个比特位都需要消耗一个周期，再加上0.5倍的额外周期![image-1793787689065.png](attachments/image-1793787689065.png)
## 1.3.3 采样时间和信号源内阻的关系
1. 采样时间：采样开关闭合的时间
2. 采样时间和信号源内阻关系：如下图，对于电池来说是含有内阻的，那对于输入的模拟信号也是含有内阻的![image-1793788074463.png](attachments/image-1793788074463.png)![image-1793788194453.png](attachments/image-1793788194453.png)
3. 闭合开关，模拟信号输入，两个电阻串联，当信号源内阻的阻值变大，流过两个电阻的电流变小，电容充电时间就越长（结论：信号源内阻越大，采样时间越长）![image-1793788251541.png](attachments/image-1793788251541.png)
## 1.3.4 信号源内阻的计算方法
1. 光敏传感器：光敏电阻（光照强度越强，阻值越小），通过AO输出模拟信号
2. 光敏传感器模块的电路图：两个电阻，阻值一个固定一个可调，两个电阻串联分压，光照强度变强，阻值变小，同时根据基尔霍夫电压定律（KVL）与欧姆定律，分压节点的电压（VoutVout​）由分压公式决定，中间点的电压，是由“夹在中间点和地（GND）之间的那个电阻”决定的，所以这个光敏电阻决定了这个中间点的电压![image-1793788554617.png](attachments/image-1793788554617.png)
3. 这个第二个到第三个的转换要用戴维男等效定理，第三张的电阻是两端所有电源归零后的电阻并联，所以最终等效的电阻肯定是小于R1的![image-1793790821515.png](attachments/image-1793790821515.png)
## 1.3.5 采样时间的计算方法
1. 公式含义：（信号源电阻+ADC采样电阻）* ADC采样电容 * 时钟频率 * 采样深度![image-1793791338528.png](attachments/image-1793791338528.png)![image-1793791403331.png](attachments/image-1793791403331.png)![image-1793791576201.png](attachments/image-1793791576201.png)
# 1.4 常规序列的单通道转换
1. 实验：通过光敏传感器亮度来调节指示灯亮灭
![image-1793799448850.png](attachments/image-1793799448850.png)
## 1.4.1 初始化IO引脚
1. 引脚八种模式：输入（输入上拉，输入下拉，输入浮空，模拟模式（本次实验输入的是模拟信号输入的，故选择模拟模式）），输出![image-1793801136230.png](attachments/image-1793801136230.png)
2. 代码
```c
//使能时钟
RCC_APB2PeriphClockCmd(RCC_APB2PeriphClock_GPIOA,ENABLE);
GPIO_InitTypeDef GPIO_InitStruct={0};
GPIO_InitStruct.Pin=GPIO_Pin_0;
GPIO_InitStruct.Mode=GPIO_Mode_AIN;

GPIO_Init(GPIOA,&GPIO_InitStruct);//这个要取地址
```
## 1.4.2 配置ADC的时钟
1. 时钟树：ADC的频率最高不能超过14MHz,当主频为72MHz的时候，最低也要六分频![image-1793802345540.png](attachments/image-1793802345540.png)
2. 代码
```c
RCC_APB2CLKConfig(RCC_PCL2_Div6);

RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1,ENABLE);
```
## 1.4.3 ADC的编程接口
1. 编程接口![image-1793803323099.png](attachments/image-1793803323099.png)
	- 获取标志位的值EOC 代表转换完成 JEOC代表注入序列转换完成
## 1.4.4 初始化ADC的基本参数
1. 连续模式:![image-1793853248205.png](attachments/image-1793853248205.png)

2. 左对齐，右对齐 （寄存器比特位宽度是十六位，而这个是一个12位逐次逼近型ADC，要保存在十六位的寄存器中，就需要左对齐和右对齐）![image-1793853653351.png](attachments/image-1793853653351.png)
## 1.4.5 配置ADC的基本参数
```C
	ADC_InitTypeDef ADC_InitStruct={0};
	ADC_InitStruct.ADC_ContinuousConvMode=DISABLE;//关闭连续模式
	ADC_InitStruct.ADC_DataAlign=ADC_DataAlign_Right;//右对齐
	ADC_InitStruct.ADC_ExternalTrigConv=ADC_ExternalTrigConv_None;//常规序列的外部触发信号  软件启动
	ADC_InitStruct.ADC_Mode = ADC_Mode_Independent;//独立模式
	ADC_InitStruct.ADC_NbrOfChannel=1;   //写入序列的通道数  常规序列一个通道
	ADC_InitStruct.ADC_ScanConvMode=DISABLE;//扫描模式
	
	ADC_Init(ADC1,&ADC_InitStruct);
	
```
## 1.4.6 配置常规序列
```c
	//配置ADC常规序列
	ADC_RegularChannelConfig(ADC1,ADC_Channel_0,1,ADC_SampleTime_13Cycles5);//常规序列的配置的ADC1  通道数，序列号，采样时间
	ADC_ExternalTrigConvCmd(ADC1,ENABLE);//常规序列的外部触发信号的开关闭合
```
## 1.4.7 闭合ADC总开关
```c
	ADC_Cmd(ADC1,ENABLE);//闭合ADC总开关
```
## 1.4.8 启动并读取结果
1. 如何判断转换是否完成：使用EOC标志位（清零）
2. 读取出的dr值要进行转换成电压：右对齐范围是0~4095![image-1793874997535.png](attachments/image-1793874997535.png)
# 1.5 定时器触发
1. 实验流程：光敏传感器模拟输出，然后将通道0写入注入序列中，然后通过定时器TIM_TRGC外部触发然后启动注入序列，光敏传感器产生模拟信号，模拟信号接到PA0，然后通过串口发送给串口，然后电脑绘制波形![image-1793877086876.png](attachments/image-1793877086876.png)
2. TRGO：从模式控制器（通过TRGO向外输出控制信号，控制别的模块）定时器时基单元，从模式控制器通过接收TRGI的信号，然后通过TRGO来控制对应的片上外设![image-1793877240572.png](attachments/image-1793877240572.png)
## 1.5.1 连接电路
1. 电路图：![image-1793877867825.png](attachments/image-1793877867825.png)
2. 接线图：![image-1793877927630.png](attachments/image-1793877927630.png)
## 1.5.2 初始化串口
1. 串口初始化的流程：
	- 初始化IO引脚（TX RX）
	- 开始串口时钟
	- 配置串口参数
	- 闭合总开关
	- ![image-1793878104709.png](attachments/image-1793878104709.png)
## 1.5.3 配置定时器1的TRGO
1. TRGO信号产生的思路：从模式控制器设置为update模式，每当时基单元产生一个update的时候，TRGO就会产生一个脉冲信号
## 1.5.4 注入序列的编程接口
1. 编程接口图![image-1793881184922.png](attachments/image-1793881184922.png)
## 1.5.5 初始化ADC（注入序列）
1. 初始化IO引脚
2. 配置时钟（ADC时钟要小于14MHz）![image-1793930363067.png](attachments/image-1793930363067.png)
3. 配置ADC的基本参数![image-1793931173252.png](attachments/image-1793931173252.png)
4. 配置注入序列
5. 闭合总开关
6. 读取并发动测量结果
# 1.6 扫描模式





