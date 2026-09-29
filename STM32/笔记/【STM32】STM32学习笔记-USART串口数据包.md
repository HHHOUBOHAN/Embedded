---
title: "【STM32】STM32学习笔记-USART串口数据包 "
author: "CC 4.0 BY-SA"
date: "于 2024-01-09 21:10:31 发布"
source: "https://blog.csdn.net/u010249597/article/details/135489947"
---

### 00. 目录
 





 

#### 文章目录

 

- 
  - [00. 目录](#00__0)
  - [01. 串口简介](#01__3)
  - [02. HEX数据包](#02_HEX_14)
  - [03. 文本数据包](#03__22)
  - [04. HEX数据包接收](#04_HEX_34)
  - [05. 文本数据包接收](#05__46)
  - [06. 预留](#06__56)
  - [07. 附录](#07__64)





 


### 01. 简介
 


串口通讯(Serial Communication)是一种设备间非常常用的串行通讯方式，因为它简单便捷，因此大部分电子设备都支持该通讯方式， 电子工程师在调试设备时也经常使用该通讯方式输出调试信息。
 


在里，大部分复杂的问题都可以通过分层来简化。如芯片被分为内核层和片上外设；STM32标准库则是在寄存器与用户代码之间的软件层。 对于通讯协议，我们也以分层的方式来理解，最基本的是把它分为物理层和协议层。物理层规定通讯系统中具有机械、电子功能部分的特性， 确保原始数据在物理媒体的传输。协议层主要规定通讯逻辑，统一收发双方的数据打包、解包标准。 简单来说物理层规定我们用嘴巴还是用肢体来交流，协议层则规定我们用中文还是英文来交流。
 


### 02. HEX
 


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/86a75d60e59b8c8431ef660ee5bf40ec.png#pic_center)
 


### 03. 文本数据包
 


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/f2a54b7f519bfa35ec12eef77ad42e60.png#pic_center)
 


### 04. HEX数据包接收
 


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/d8471333081bbbcb0862910a4952e608.png#pic_center)
 


### 05. 文本数据包接收
 


![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/b8b335cf28d02827ad817f4def586410.png#pic_center)
 


### 06. 预留
 


### 07. 附录
 


参考: [【STM32】江科大STM32学习笔记汇总](https://blog.csdn.net/u010249597/article/details/134762513)

---
> 原文链接: [【STM32】STM32学习笔记-USART串口数据包 ](https://blog.csdn.net/u010249597/article/details/135489947)
> 作者: CC 4.0 BY-SA
