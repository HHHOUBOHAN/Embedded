# 2.1 FreeROTS总图
## 2.1.1 FreeROTS总图
1. FreeROTS 调度器 负责切换任务
	- 拆分问题
	- 编写任务代码
	- 创建任务
	- 启动调度器
2. FreeROTS总图![[Pasted image 20261003212143.png]]
## 2.1.2 调度器和任务
### 2.1.2.1 调度器
1. 调度算法：调度器切换任务时所采用策略
2. 调度算法三种类型
	 - 带时间片的抢占式调度
	 - 不带时间片的抢占式调度
	 - 协作式调度
### 2.1.2.2 任务
1. 任务类型：
	- 调度器启动时创建
	- 用户创建（空闲任务，定时器任务）  启动调度器的时候，函数内部系统会帮创建空闲任务和定时器任务![[Pasted image 20261003213350.png]]![[Pasted image 20261003214905.png]]
2. 空闲任务
	- 单片机里的CPU 就像一颗永不停歇的大脑，当非空闲任务都陷入停滞的时候，调度器就会选择空闲任务执行
	- 空闲任务的启动场景![[Pasted image 20261003214425.png]]![[Pasted image 20261003214514.png]]
3. 定时器任务：软件定时器就必须需要创建定时器任务
## 2.1.3 三种中断：SVC,pendSV和Systick
1. SVC 中断：启动首个任务（优先级最高）
2. pendSV中断：用于后序任务切换（优先级最低）
3. Systick中断：用于时间片的轮询调度（优先级最低）产生时间片，系统节拍
4. 示例：                    ![[Pasted image 20261003215359.png]]![[Pasted image 20261003222400.png]]
## 2.1.4 内核对象
1. 队列、队列集合、事件组、软件定时器、各种信号量（二进制信号量、计数信号量、互斥锁和递归互斥锁）、事件组、消息缓冲区和流缓冲区等。除此之外，任务也属于内核对象（不管是调度器创建的任务，还是用户创建的任务）。
## 2.1.5 堆内存管理方案
1. 5种堆内存管理方案：heap_1、heap_2、heap_3、heap_4和heap_5
![[Pasted image 20261003232001.png]]
# 2.2 内核的文件结构
## 2.2.1 内核移植
1. LTS:长期支持版本，官方承诺较长时间内提供持续的维护，这个时间一般为两年
2. 文件夹层次结构![[Pasted image 20261004110835.png]]
## 2.2.2 EreeRTOS-Kernel下的众多.c文件
1. 文件作用![[Pasted image 20261004111533.png]]
2. 移植（设置调试器，创建FreeRTOS文件夹是Source Folder）![[Pasted image 20261004112232.png]]
## 2.2.3 Include 文件夹
1. 文件作用                            ![[Pasted image 20261004112653.png]]
2. 移植                                   ![[Pasted image 20261004112904.png]]
## 2.2.4 portable文件夹(移植层)
1. 移植层：FreeRTOS为了适应不同**工具链类型**和**单片机内核种类**而提供的一组代码（移植层portable相当于FreeROTS内核与单片机和开发工具链两者的中介）![[Pasted image 20261004113449.png]]![[Pasted image 20261004113753.png]]
2. 移植层文件夹结构：portable\开发工具链\内核类型![[Pasted image 20261004113915.png]]
 3. 移植                               ![[Pasted image 20261004114110.png]]
## 2.2.5 MemMang文件夹 (内存管理)
1. 文件夹中有五个文件 ，对应了五种不同的堆内存管理
2. 移植![[Pasted image 20261004114645.png]]
## 2.2.6 FreeRTOSConfig.h文件 (配置文件模板)
1. 移植：example\template configuration\FreeRTOSConfig.h
## 2.2.7 头文件引用
1. FreeRTOS.h（内核头文件）， task.h（任务和任务调度）前面这两个必须引用，后面就是看需求![[Pasted image 20261004115442.png]]
# 2.3 配置文件
## 2.3.1 配置项1:ConfigCpU-CLOCK-HZ(告诉FreeRTOS内核的运行频率)
1. 这个配置项用来告诉FreeRTOS你的单片机的内核频率是多少
2. **例1：**请先以上节课的程序为基础计算configCPU_CLOCK_HZ的大小。然后改变单片机的时钟配置，将单片机的时钟设置为最大值，重新计算configCPU_CLOCK_HZ的值
3. STM32F1系列的单片机通过**HCLK给Cortex-M3内核提供时钟**，如果没有配置过时钟树，默认情况下单片机从HSI（内部RC振荡器）获取8MHz的时钟，因此HCLK的时钟频率也是8MHz，Cortex-M3内核从HCLK上直接获取时钟，既然HCLK等于8MHz，那么Cortex-M3内核的时钟频率也应该是8MHz
## 2.3.2 配置项2:COnfigTICK-TYpE-WIDTH-IN-BITS（设置TickType-t类型的位宽）
1. 位宽: 用多少个比特位来表示一个整数  ![[Pasted image 20261004131509.png]]
2. TickType-t: 用来存储系统节拍（时间），用来记录FreeRTOS的系统节拍的数据类型，说白了就是用来记录时间的。
3. FreeRTOS内部维护了一个时间值，也要有一套函数用来获取当前时间或者延迟。![[Pasted image 20261004132300.png]]![[Pasted image 20261004133947.png]]
4. 系统节拍如何转换成时间![[Pasted image 20261004132627.png]]

	- 为了记录当前时间FreeRTOS也声明了一个xTickCount变量。它以ms为单位，每次进入SYSTICK中断的时候xTickCount的值自动递增1，因此它的值就等于FreeRTOS运行的时间。所以，调用xTaskGetTickCount()获取系统时间其实就是直接读取xTickCount的值。
5. 位宽会影响时间戳回绕的频率：![[Pasted image 20261004132900.png]]
	- 由于有这个位宽会影响回绕的频率，回绕会对系统造成影响，为了消除这个影响，就必须在代码里对这种情况做一些特殊的处理，处理过程中就会消耗一定的系统性能，所以要让两次溢出的时间间隔越宽越好，时间间隔的增加就需要改变我们这个计数值的一个位宽，也就是TickType-t类型的位宽
## 2.3.3 配置项3:ConfigKERNEL-INTERRUpT-pRIORITY（设置pendSV和Systick中断的优先级）
1. STM32F103系列单片机的优先级范围是：0~15，0是最高优先级，15是最低优先级
2. 请结合当前单片机的情况给configKERNEL_INTERRUPT_PRIORITY赋值
	- Cortex-M3内核使用8位二进制数表示中断优先级，STM32F1只使用了高4位（优先级范围为0~15，四位就够了）![[Pasted image 20261004134808.png]]
	- 如果你直接写数字 `15`，它的二进制表示是`00001111`,如果直接把 `15` 写入 8 位寄存器，落入的是芯片未实现的低 4 位，高 4 位全是 0。硬件直接忽略低 4 位，结果内核实际识别到的优先级变成了 **0（最高优先级）**，与预期完全相反; 所以要通过左移4 位（`15 << 4`），这样有效数据正好占满 STM32 硬件支持的高 4 位
	- 设置pendSV和Systick中断的优先级：这两个的优先级都是最低优先级，也就是15
## 2.3.4 配置项4: configMAX_SYSCALL_INTERRUPT_PRIORITY（能够调用FreeRTOS API的最高优先级）
 1. 单片机的中断一共有15级优先级，数字越大优先级越低，数字越小优先级越高![[Pasted image 20261004141029.png]]
 2. configMAX_SYSCALL_INTERRUPT_PRIORITY对所有16级优先级进行划分。现在我们把configMAX_SYSCALL_INTERRUPT_PRIORITY设置成5，这样就把16级优先级一分为二了
 3. 黄色部分表示优先级比configMAX_SYSCALL_INTERRUPT_PRIORITY高的中断。我们用这些中断处理一些非常紧急的代码，比如对时间非常敏感的电机控制代码。因为这些中断所处理的代码对实时性的要求非常高，所以不希望被别的什么东西干扰，甚至包括FreeRTOS内核。因此，这些中断里边是不能调用FreeRTOS的任何API（实际原因是因为FreeRTOS的临界区，这里就不详细介绍了）。
 4. 绿色部分部分表示优先级小于或等于configMAX_SYSCALL_INTERRUPT_PRIORITY的中断。这些中断优先级较低，因此被FreeRTOS干扰一下也没啥事（实际还是因为临界区），所以你可以在这些中断里边调用FreeRTOS的API。
## 2.3.5 配置项5:configUSE_TIME_SLICING（是否使用时间片）
1. configUSE_TIM_SLICING用来设置是否使用时间片，英文单词slicing是切片的意思。

![](https://kcn06ia36zs4.feishu.cn/space/api/box/stream/download/asynccode/?code=NjlkZjk2MWZiMDJkNDI1MGRiMGMyMzc4NzZhMjRlZjNfZjBJVUVVcHJxREVrcDJTU0R3cmtEYUtxNXVYdGVmVWZfVG9rZW46RWdjUmI3WVZCb05xRm14TnN2TWNPRWNobk1mXzE3OTEwOTU0MTI6MTc5MTA5OTAxMl9WNA&add_watermark=true&scene_type=CCM)

2. FreeRTOS的调度器有3种可选的调度算法，分别是：不带时间片的抢占式调度、带时间片的抢占式调度和协作式调度。同样使用抢占式调度的前提下，如果configUSE_TIM_SLICING等于1，那就是带时间片的抢占式调度；如果configUSE_TIM_SLICING等于0，就是不带时间片的抢占式调度。在一般情况下，我们习惯带时间片，所以需要把configUSE_TIM_SLICING的值设置成1。
# 2.4 安装中断服务程序
1. 把中断安装到我们的中断向量表中![[Pasted image 20261004144634.png]]
## 2.4.1 安装中断服务程序
**例1： 请将port.c文件提供的3个中断服务程序安装到单片机的中断向量表里，以确保中断发生时这些函数能够被调用。

  
**解： 如下图，当我们使用HAL库编写STM32F103RCT6的程序的时候，单片机的中断向量表位于启动文件当中，具体路径是“工程根目录 \ Core \ startup \ startup_stm32f103rctx.s”。精确到行的话是从131行开始，到253行结束。

![](https://kcn06ia36zs4.feishu.cn/space/api/box/stream/download/asynccode/?code=NjM3MDBlZjcyNmI1NGNiOTdjNGYxY2QyYmQ3YTE3MWNfNTE2OGNaSFZ2eVhzZmVmUkpGRU1GT3NLYk1ENXF6bFFfVG9rZW46RG5ZNGJ0aTRUb2F3NVl4SEJRTGNFMW9TbnZiXzE3OTEwOTY0MjU6MTc5MTEwMDAyNV9WNA&add_watermark=true&scene_type=CCM)

我们只要在中断向量表里找到这3个中断所在的位置，然后用port.c文件提供的函数把它们替换掉就可以了。
## 2.4.2 使用其它定时器作为HAL库的时间基准
1. 我们已经把中断向量表里原有的中断服务程序替换成了FreeRTOS提供的函数，其中就包括了Systick定时器的中断服务程序。之前我们介绍过，HAL库要使用Systick中断来递增当前时间，代码如下：
```
// @作用：这是SYSTICK定时器的中断服务程序

**void** **SysTick_Handler**(**void**)

{

HAL_IncTick(); // 每次进入中断递增uwTick的值

}

```
2. 如果我们用FreeRTOS提供的xPortSystickHandler()替换掉了原有的Systick_Handler()，那么是不是就意味着HAL库的计时功能失效了呢？事实确实如此，一旦我们为FreeRTOS安装了中断服务程序，HAL库里和时间有关的函数就都不能用了。比如，HAL_GetTick()、 HAL_Delay()，甚至一切带超时时间的函数都将失去效果。

3. 如果我既想使用FreeRTOS提供的函数，又想HAL库提供的函数，那该怎么办？当然，你可以让FreeRTOS和HAL库共享Systick，但一般基于效率考虑我们会让HAL库让出Systick定时器，因为共用Systick会略微降低FreeRTOS的运行效率。让出Systick之后，HAL库可以改用其它的定时器。![](https://kcn06ia36zs4.feishu.cn/space/api/box/stream/download/asynccode/?code=NzQ2M2YwMzM1ZmYzZDBlN2I5NDIxYjBjNDAxZDc2NDFfUm45TzU5czJlZ0lsVG1VcmUzb2tsSDRjOG15cnR0YktfVG9rZW46TktNZ2J1QW1mb05aeU54djFWRWNCZjNCbm9mXzE3OTEwOTY5MzE6MTc5MTEwMDUzMV9WNA&add_watermark=true&scene_type=CCM)
## 2.4.2 最终测试
1. 使用移植好FreeRTOS的工程编写一段代码，实现LED1每1秒闪烁一次，LED3每200ms闪烁一次
2. 头文件的引用
```c
 #include** "FreeRTOS.h" // FreeRTOS的核心头文件
 #include** "task.h" // 与任务、调度器相关的头文件
```
3. 分别为LED1和LED3编写任务代码，LED1每1秒闪烁一次，LED3每200毫秒闪烁一次
```c
void vLED1Task(void *pvParameters)

{

for(;;)

{

HAL_GPIO_WritePin(LED1_GPIO_Port,LED1_Pin,GPIO_PIN_SET);

vTaskDelay(pdMS_TO_TICKS(500));

HAL_GPIO_WritePin(LED1_GPIO_Port,LED1_Pin,GPIO_PIN_RESET);

vTaskDelay(pdMS_TO_TICKS(500));

}

}

void vLED3Task(void *pvParameters)

{

for(;;)

{

HAL_GPIO_WritePin(LED3_GPIO_Port,LED3_Pin,GPIO_PIN_SET);

vTaskDelay(pdMS_TO_TICKS(100));

HAL_GPIO_WritePin(LED3_GPIO_Port,LED3_Pin,GPIO_PIN_RESET);

vTaskDelay(pdMS_TO_TICKS(100));

}

}
```
3. 在main()方法里分别创建LED1任务和LED3任务，最后启动调度器
```c
xTaskCreate(vLED1Task,"LED1",128,NULL,1,NULL);

xTaskCreate(vLED3Task,"LED3",128,NULL,1,NULL);

  

vTaskStartScheduler();
```
## 2.4.3 练习题
![[Pasted image 20261004151859.png]]
   























