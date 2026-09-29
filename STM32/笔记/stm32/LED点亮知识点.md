# 闪烁
1. HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET);
    ![[Pasted image 20260328095310.png]]
    这个LED 是因为用户标签是LED 
    LED亮的机制其实是一个简单的  “电位差”**（电压差）原理
    外部电压接电就是3.3 但是这个内部电压由
    GPIO_PIN_RESET// 复位状态：低电平 (0V)
    GPIO_PIN_SET // 置位状态：高电平 (3.3V)
    - **`RESET`(复位)**：对应逻辑**0。**在硬件上，这意味着引脚被内部的N-MOS管拉到了**GND**（地），输出电压为**0V**。
    - **`SET`(置位)**：对应逻辑**1。**在硬件上，这意味着引脚被内部的P-MOS管推向了**VCC**，输出电压通常为**3.3V**。
 
---

### 1. 核心原理：水位（电位）调节法

STM32的引脚不是一个简单的“电源”，而是一个**可调水位（电位）的排水口**。

- **外部环境（固定水位）**：你的LED一头接在$3.3\text{V}$，这个水位是永恒不变的。
    
- **内部设置（可调水位）**：引脚通过代码在$0\text{V}$和$3.3\text{V}$之间切换。
    

---

### 2.亮灭逻辑的“开关”

根据你的硬件（低电平点亮），逻辑如下：

|**动作（软件）**|**常量定义**|**引脚电位**|**与外部3.3伏的压差**|**LED状态**|
|---|---|---|---|---|
|**RESET（复位）**|`GPIO_PIN_RESET`|**$0\text{V}$**|**$3.3\text{V}$(有压差)**|**亮（流过电流）**|
|**SET (置位)**|`GPIO_PIN_SET`|**$3.3\text{V}$**|**$0\text{V}$(无压差)**|**灭（堵住电流）**|

---

### 3. CubeMX初始化的“起跑线”

您在工具里设置的**GPIO 输出电平**决定了芯片“睡醒”后的第一个姿势：

- 设置**Low** ：开机重新执行`RESET` $\rightarrow$引脚$0\text{V}$ $\rightarrow$ **启动即亮**。
    
- SetHigh**：**开机重新执行`SET` $\rightarrow$引脚$3.3\text{V}$ $\rightarrow$ **启动即灭**。
    

---

### 4. 延伸到“输入”逻辑（按键）

此时你懂了电压差，看按键（输入）就秒懂了：

- **按键未按下**：引脚通过“上拉电阻”接在内部$3.3\text{V}$。
    
- **按键按下**：引脚被强行按在外部的$GND$（$0\text{V}$）。
    
- **程序判断**：如果你发现引脚电压变低了（变成了`RESET`状态），你就知道：“噢！有人按了按键，把电位拽下去了！”
    

---

### 💡老师傅的叮嘱

在嵌入式开发中，永远记住：**逻辑（1和0）是虚的，电压差（电位高低）才是实的。**只要你能分析出电流从哪里流到哪里，你就永远不会写错代码。
# 其他
while (1)
  {
//		HAL_GPIO_WritePin(GPIOA,GPIO_PIN_5,GPIO_PIN_SET);
//		HAL_Delay(400);
//		HAL_GPIO_WritePin(GPIOA,GPIO_PIN_5,GPIO_PIN_RESET);
//		HAL_Delay(400);
		
		if(HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_3)==GPIO_PIN_RESET)
		{
			HAL_Delay(20);
			if(HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_3)==GPIO_PIN_RESET)
			{
				HAL_GPIO_WritePin(GPIOA,GPIO_PIN_5,GPIO_PIN_SET);
				HAL_Delay(400);
				HAL_GPIO_WritePin(GPIOA,GPIO_PIN_5,GPIO_PIN_RESET);
				HAL_Delay(400);
				while(HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_3)==GPIO_PIN_RESET);
			}
			HAL_Delay(20);
				
		}
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
这是这三个函数的使用例程
1. 未注释代码解析
    `if(HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_3)==GPIO_PIN_RESET)`
    `HAL_GPIO_ReadPin`去读引脚
    `(GPIOA, GPIO_PIN_3)`是读引脚的参数
     `GPIOA` 是芯片引脚是一组的，这里指**A组**。
     `GPIO_PIN_3`是这一组的第三号引脚 最后组成PA3
    最后是将读出的电压与这个GPIO_PIN_RESET 复位状态对比（0V）最后计算电压差

    `HAL_Delay(20);`消抖

    `if(HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_3)==GPIO_PIN_RESET)`同第一个一样再一次判断  双重保险 

```
    HAL_GPIO_WritePin(GPIOA,GPIO_PIN_5,GPIO_PIN_SET);
				HAL_Delay(400);
				HAL_GPIO_WritePin(GPIOA,GPIO_PIN_5,GPIO_PIN_RESET);
				HAL_Delay(400);
```
    这个是进行这个 初始的是低电平（是由自己按按钮形成的） 而且形成了电位差  
    第一步是由亮变暗 
    中间延时了400ms 让这暗持续400ms  
    然后再由暗变亮 让这亮持续  400ms
    执行完后看看自己是否松开按钮

    `while(HAL_GPIO_ReadPin(GPIOA,GPIO_PIN_3)==GPIO_PIN_RESET);
			}
			HAL_Delay(20);`
			首先通过这个防止连发：这个因为 `while(1)` 循环跑得极快，
			即使你只是飞快地按了一下（比如持续了 100ms），
			程序可能已经循环了几千次。结果就是：你本想让灯闪一次，
			结果灯像抽风一样闪了几十次。


![[Pasted image 20260328110110.png]]
这个最后两行是根据按钮按下来决定  就是第一步按钮按下后如果灯还没有亮完 执行到这个最后两行 已经是0V 直接完成这个循环了  但是还有一种就是一直按这着 最后两行还是没有松开  这个时候只能等这个按钮松开   这个循环才能成立  但是在这个期间 这个灯是一直亮着的
