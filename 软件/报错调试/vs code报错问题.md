# GCC 能报错，但编辑器不能即时爆红
# vs code 关于GitHub提交问题

# CMake
## 工作流程
```
CMakeLists.txt
    ↓
Configure 配置
    ↓
Generate 生成 Ninja 构建文件
    ↓
Ninja 调用 arm-none-eabi-gcc 编译
    ↓
生成 .elf / .bin / .hex
```
## 新建 `.c` 不参与编译
1. 原因：`CMakeLists.txt`没有包括这个文件、
2. 手动加入即可
## 文件移动后，重新生成产生报错
1. 原因：文件移动后但 `CMakeLists.txt` 还保留旧路径  同时文件移动造成的配置不一致，直到今天重新 Configure 时才被正式发现。
2. 解决方法：所有自己写的固件源码，都放在 `User/` 下面
### 统一采用相同工程架构
```
smart_water_tank_f103/
├── Core/                       ← CubeMX生成，不放自定义驱动
│   ├── Inc/
│   └── Src/
│
├── Drivers/                    ← STM32 HAL、CMSIS
│
├── User/                       ← 全部手写代码
│   ├── BSP/
│   │   ├── board.c
│   │   ├── board.h
│   │   ├── buzzer.c
│   │   ├── buzzer.h
│   │   ├── key.c
│   │   ├── key.h
│   │   ├── led.c
│   │   ├── led.h
│   │   ├── relay.c
│   │   └── relay.h
│   │
│   ├── Drivers/
│   │   └── sensor/
│   │       ├── water_level.c
│   │       └── water_level.h
│   │
│   ├── Services/
│   │   ├── alarm_service.c
│   │   └── alarm_service.h
│   │
│   ├── App/
│   │   ├── app.c
│   │   └── app.h
│   │
│   └── CMakeLists.txt          ← 自动收集User目录中的.c
│
├── cmake/
│   └── stm32cubemx/            ← CubeMX管理，不手动修改
│
├── CMakeLists.txt              ← 顶层用户可维护文件
├── CMakePresets.json
└── smart_water_tank_f103.ioc
```
### 配置 `User/CMakeLists.txt`
```
CMakePresets.json
    ↓ 决定如何配置

根 CMakeLists.txt
    ↓ 创建主固件目标

cmake/stm32cubemx/CMakeLists.txt
    ↓ 添加CubeMX代码

User/CMakeLists.txt
    ↓ 添加你的用户代码
```
1. 修改当前工程
### 在 STM32 Profile 中永久关闭自动修改
 >这个非常重要 尤其是你删除文件的时候 如果不关闭 他会自动改变你这个配置  然后出现报错
 
### 创建脚本
```
新 CubeMX 工程
 ↓
运行“STM32: 初始化当前工程”
 ↓
创建 User/Inc、User/Src
 ↓ 
创建 User/CMakeLists.txt
 ↓ 
备份根 CMakeLists.txt
 ↓ 
插入 add_subdirectory(User)
 ↓ 
以后 User/Src 下新增或删除 .c 自动生效
```