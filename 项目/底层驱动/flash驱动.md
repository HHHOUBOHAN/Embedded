- 作用：保证运行时保存参数时，绝对不会擦掉程序；同时保证突然断电时，尽量还能恢复上一份完整配置。
# 硬件和工作原理
## STM32F103C8T6
```
官方Flash：64KB
起始地址：0x08000000
结束地址：0x0800FFFF
页大小：1KB
总页数：64页
编程单位：16位半字
擦除单位：整页
```
## 链接配置
### 找链接文件
```powershell
Get-ChildItem -Path . -Recurse -Filter *.ld   //查找路径  
code .\STM32F103XX_FLASH.ld                   //打开文件
```
```

RAM (xrw)   : ORIGIN = 0x20000000, LENGTH = 20K
FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 64K
```
1. 地址范围：`0x08000000 ～ 0x0800FFFF`
2. Flash：大小是`64KB`,同时保存（程序代码 常量和字符串 初始化数据 用户配置），但是这个`flash`没有提供这个**配置存储区**，如果不主动预留，链接器会认为整个64KB都可以放程序，现在程序只占用这个约`10kB`,暂时不会用到后面几页，但是如果后面增加`4G驱动 MQTT JSON解析 OLED页面 报警逻辑 传感器服务 日志字符串`,程序会不断变大，链接器可能逐渐把代码放到最后两页，如果最后一页放入一部分程序，此时设备保存配置，执行了这个擦除`0x0800FC00`这一页,`配置页被擦除 程序代码也被一起擦除 设备跑飞、HardFault或者无法再次启动`,所以“当前程序还小”并不能代替预留。我们必须明确告诉链接器:最后两页不能放程序，
3. 配置页被擦除 程序代码也被一起擦除：这个是因为，这个最后两页同时有这个用户配置和这个程序，擦除是按页擦除，所以就会直接把配置页和程序代码一起擦除
4. 解决：之所以会发生这种情况，是因为这个链接器只认识链接脚本，如果这个链接脚本写`FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 64K`,链接器，会认为这个从0x08000000开始的64KB全部属于程序区域，`flash.c`如果写了`#define CONFIG_ADDRESS 0x0800F800`, 链接器也看不到这个约定，于是会出现这个互相不知道对方存在的模块，`链接器：最后2KB可以放程序 Flash驱动：最后2KB用于保存配置`,就可能发生地址冲突，导致这个擦除错误，把程序修改为`FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 62K`,链接器就只能用`0x08000000 ～ 0x0800F7FF`，程序超过`62KB`,编译器会直接报错；
5. 分区：
```
STM32内部64KB Flash
│
├─ 程序区：62KB
│  0x08000000 ～ 0x0800F7FF
│
├─ 配置A页：1KB
│  0x0800F800 ～ 0x0800FBFF
│
└─ 配置B页：1KB  
   0x0800FC00 ～ 0x0800FFFF
```
6. 分区的保证：
```
链接脚本 → 禁止程序进入配置区 flash.c → 只允许擦写配置区 storage_service.c → 决定配置格式和A/B页切换
```
7. `Flash` 擦除操做：Flash页大小为`1KB = 1024字节 = 0x400字节`,Flash不能像RAM一样随便覆盖,擦除后一个字节通常是`0xFF 二进制：11111111`,编程操作主要把位从`1~0`,但是想从`0~1`,普通写入时做不到的，需要擦除，而这个`stm32f103c8t6`最小擦除单位时一整夜，即使修改了一个字节，只要是由这个从`0~1`，也可能需要擦除整整1024字节，
8. 为啥要分`A B`两页，如果只有一页， 保存流程一般是：`擦除旧配置 → 写入新配置`，加入擦除以后突然断电，就会造成`旧配置已经没了 新配置还没写完`,下下一次上电就没有一份完整的配置，但是使用两页后，`A页：当前有效配置 B页：准备写入新配置`,保存新配置的时候
```
1. 保留A页不动 2. 擦除B页 3. 把新配置写入B页 4. 读取B页检查 5. 检查CRC 6. 确认B页完整后，将B页视为最新配置
如果写入B页过程中断电：B页可能无效 A页仍然保留旧配置
下次上电：检查A页 检查B页 → 选择CRC正确且序号更新的一页
下一次保存再反过来：当前B页有效 → 把新配置写入A页
好处：提高突然断电时的恢复能力；擦写分散到两个页面，降低单页集中磨损
```
9. 配置页保存内容：
```
保存：
数据标识Magic
配置格式版本
配置长度
保存序号
AUTO/MANUAL模式
水位启动阈值
水位停止阈值
压力报警阈值
压力恢复阈值
校准参数
CRC校验值

不保存：
继电器状态
pump_command_on 
APP_RUN_RUNNING 
断电前水泵是否运行
```
10. 进行配置
```powershell
Copy-Item ` .\STM32F103XX_FLASH.ld ` .\STM32F103XX_FLASH.ld.backup    //备份链接脚本
//修改MEMORY区域
//将原来的
MEMORY { RAM (xrw) : ORIGIN = 0x20000000, LENGTH = 20K FLASH (rx) : ORIGIN = 0x8000000, LENGTH = 64K }
//改成
MEMORY
{
    /*
     * STM32F103C8T6 internal SRAM.
     */
    RAM (xrw) :
        ORIGIN = 0x20000000,
        LENGTH = 20K

    /*
     * Firmware program region.
     *
     * Only the first 62 KB can be used by the linker.
     */
    FLASH (rx) :
        ORIGIN = 0x08000000,
        LENGTH = 62K

    /*
     * Configuration storage region.
     *
     * The last two Flash pages, 2 KB in total.
     */
    FLASH_CONFIG (r) :
        ORIGIN = 0x0800F800,
        LENGTH = 2K
}

/*
 * Flash configuration region symbols.
 *
 * These symbols can be referenced from C code later.
 */
__flash_config_start__ =
    ORIGIN(FLASH_CONFIG);

__flash_config_end__ =
    ORIGIN(FLASH_CONFIG) + LENGTH(FLASH_CONFIG);

__flash_config_page_size__ =
    0x400;

__flash_config_page_a__ =
    ORIGIN(FLASH_CONFIG);

__flash_config_page_b__ =
    ORIGIN(FLASH_CONFIG) + 0x400;


//检查相关内容
Select-String `
    -Path .\STM32F103XX_FLASH.ld `
    -Pattern "FLASH|flash_config|LENGTH"
    
//重新构建
$sizeTool = Get-ChildItem "$env:LOCALAPPDATA\stm32cube\bundles" `
    -Recurse `
    -Filter arm-none-eabi-size.exe |
    Select-Object -First 1

$elfFile = Get-ChildItem .\build `
    -Recurse `
    -Filter *.elf |
    Select-Object -First 1

& $sizeTool.FullName $elfFile.FullName


```

## 设计 Flash 驱动接口
### 功能作用
```
负责：
识别配置A页和B页
检查页编号是否合法
检查偏移和长度是否合法
擦除指定配置页
读取指定配置页
写入指定配置页
检查页面是否已经擦除
写入后读取校验
不负责（这些交给storage_service）：
选择A页还是B页作为最新配置 
保存AUTO/MANUAL 
保存水位和压力阈值 
计算CRC 
判断配置版本 
判断哪一页更新
```
### 设计思路
#### 禁止外部直接传入Flash绝对地址
1. 直接传入Flash绝对地址
`Flash_ErasePage(0x0800FC00); Flash_Write(0x0800F810, data, length);`
- 上层如果错误传入，就可能导致擦除程序开头
- 更安全的设计是只允许传入页编号
```
FLASH_CONFIG_PAGE_A
FLASH_CONFIG_PAGE_B
```
- 底层驱动自己根据链接符号找到真实地址
- 调用方式：`Flash_ErasePage(FLASH_CONFIG_PAGE_A);`外部不能要求它擦除程序区。
#### 配置页枚举
```c
typedef enum
{
    FLASH_CONFIG_PAGE_A = 0,
    FLASH_CONFIG_PAGE_B,

    FLASH_CONFIG_PAGE_COUNT
} Flash_ConfigPage_t;
```
1. 枚举对应关系

|枚举|实际地址|
|---|--:|
|`FLASH_CONFIG_PAGE_A`|`0x0800F800`|
|`FLASH_CONFIG_PAGE_B`|`0x0800FC00`|
  2. 真实地址由链接脚本提供，不能在上层业务代码里重新写死。
#### 状态码设计
- Flash操作可能出现多种结果，不能只返回简单的 `true/false`，这样也方便调试
```c
typedef enum
{
    /*
     * 操作成功。
     */
    FLASH_STATUS_OK = 0,

    /*
     * 传入了非法页编号、空指针或者长度为0。
     */
    FLASH_STATUS_INVALID_ARGUMENT,

    /*
     * offset和length超出了单页1KB范围。
     */
    FLASH_STATUS_OUT_OF_RANGE,

    /*
     * 写入地址或长度没有按照16位对齐。
     */
    FLASH_STATUS_ALIGNMENT_ERROR,

    /*
     * Flash当前忙。
     */
    FLASH_STATUS_BUSY,

    /*
     * 目标区域没有处于擦除状态。
     */
    FLASH_STATUS_NOT_ERASED,

    /*
     * HAL解锁Flash失败。
     */
    FLASH_STATUS_UNLOCK_ERROR,

    /*
     * 擦除操作失败。
     */
    FLASH_STATUS_ERASE_ERROR,

    /*
     * 编程写入失败。
     */
    FLASH_STATUS_PROGRAM_ERROR,

    /*
     * 写入后读取的数据不一致。
     */
    FLASH_STATUS_VERIFY_ERROR

} Flash_Status_t;
```
#### 最终驱动接口
```c
/*
 * 获取单个配置页大小。
 *
 * 当前应返回1024字节。
 */
uint32_t Flash_GetPageSize(void);

/*
 * 擦除指定配置页。
 *
 * 只能选择A页或者B页。
 */
Flash_Status_t Flash_ErasePage(
    Flash_ConfigPage_t page
);

/*
 * 读取配置页中的数据。
 *
 * offset是相对于当前页开头的偏移量。
 */
Flash_Status_t Flash_Read(
    Flash_ConfigPage_t page,
    uint32_t offset,
    void *destination,
    uint32_t length
);

/*
 * 向配置页写入数据。
 *
 * Write不会自动擦除页面。
 * 调用前必须先单独执行Flash_ErasePage()。
 */
Flash_Status_t Flash_Write(
    Flash_ConfigPage_t page,
    uint32_t offset,
    const void *source,
    uint32_t length
);

/*
 * 检查指定配置页是否全部为擦除状态0xFF。
 */
Flash_Status_t Flash_IsPageErased(
    Flash_ConfigPage_t page,
    bool *is_erased
);
```
#### 擦除接口
1. 不能把接口设计为：
```
调用Flash_Write()
 ↓ 
内部自动擦除
 ↓ 
内部自动写入
```
- 上层可能就是一点点修改，就意外把整个页的其他数据全部擦除
- 同时，A/B保存需要明确顺序
```
选择非活动页
    ↓
主动擦除非活动页
    ↓
确认擦除成功
    ↓
写入新配置
    ↓
确认写入成功
    ↓
校验CRC
    ↓
新页面成为活动页
```
2. 解决：擦除和写入分别设计为独立接口`Flash_ErasePage(); Flash_Write();`  然后由`storage_service` 控制先后顺序
3. 分离“硬件能力”和“保存策略”：`flash.c`只提供硬件能力，不判断`哪一页是活动页 哪一页序号更大 哪一页CRC正确 什么时候需要保存 保存什么配置`，这些都属于`storage_service`
4. 关系：
```
flash.c 提供“怎么擦、怎么读、怎么写”
 ↓ 
storage_service.c 决定“擦哪页、什么时候擦、写什么、哪页有效”
```
#### offset是什么意思
 


 