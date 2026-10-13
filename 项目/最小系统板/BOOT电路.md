# STM32程序区域
## Flash 
用户程序 main.c编译进去的（烧录进去的程序）
## System Memory
ST厂家程序  Bootloader （不能改）
## SRAM(运行内存) 
运行数据，掉电丢失
# 两套下载方式
## 调试器下载
1. 烧录流程：电脑→USB→ ST-LINK→SWD接口→STM32→Flash
2. ST-LINK直接控制STM32,吧程序写进Flash
## 芯片内部Bootloader下载
1. 厂家提前写的一段程序位于`System Memory`
2. 特点：不能擦，不呢改，出场存在
3. Bootloader启动以后，等待外部设备（串口 USB CAN）发送新程序
# 三种启动模式
1. BOOT电路的作用是用于确定单片机的启动模式,是由BOOT0，BOOT1来决定

| BOOT1 | BOOT0 | 启动            |     |
| ----- | ----- | ------------- | --- |
| 0     | 0     | Flash         |     |
| 0     | 1     | System Memory |     |
| 1     | 0     | SRAM          |     |
| 1     | 1     | SRAM          |     |
## 模式一 Main Flash启动
1. BOOT0=0，BOOT1=0
2. 流程：上电→读取BOOT引脚
## 模式二 System Memory启动
## 模式三 SRAM启动