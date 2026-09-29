# 本地 MQTT Broker 搭建与测试手册

## 1. 这一步要实现什么

本手册先搭建电脑上的本地 MQTT Broker。此阶段 **不修改 STM32 代码**，也不要求 AHT20 到货。

最终先验证这一条最小链路：

```text
mosquitto_pub -> Mosquitto Broker -> mosquitto_sub
```

验证成功后，电脑才具备作为临时 MQTT 服务器的条件。后续 ESP-01S、网页和 MQTTX 都连接这台电脑。

## 2. 当前网络方案

```text
手机热点：0137
    |
    +-- 电脑：运行 Mosquitto、MQTTX、网页
    |
    +-- STM32 + ESP-01S：后续连接该热点
```

当前记录的电脑热点 IPv4 地址：

```text
10.131.143.31
```

> 该地址不是永久固定的。手机热点重启或电脑重新连接热点后，必须重新运行 `ipconfig` 确认。ESP 和网页后续应使用当次实际 IPv4 地址。

固定目录与端口：

```text
Mosquitto 安装目录：D:\Tools\mosquitto
项目配置目录：D:\mqtt-temp-fan
普通 MQTT TCP：1883
网页 MQTT WebSocket：9001
```

## 3. MQTT 的工作原理

MQTT 不让网页和设备直接相互寻找，而是让它们都连接 Broker（消息中转站）。

```text
STM32 -> ESP-01S -> 发布 status -> Broker -> 网页订阅并显示
网页 -> 发布 command -> Broker -> ESP-01S -> STM32 执行
```

本项目固定主题：

```text
设备编号：TEMP-FAN-001

状态上报：temp-fan/TEMP-FAN-001/status
控制命令：temp-fan/TEMP-FAN-001/command
执行结果：temp-fan/TEMP-FAN-001/ack
```

端口用途：

| 端口 | 谁连接 | 用途 |
|---|---|---|
| `1883` | ESP-01S、MQTTX | 普通 MQTT TCP |
| `9001` | 浏览器网页 | MQTT over WebSocket |

浏览器不能直接使用 `1883`，因此网页后续必须连：

```text
ws://10.131.143.31:9001
```

## 4. 下载并安装 Mosquitto

官方下载页：[Eclipse Mosquitto Download](https://mosquitto.org/download/)

Windows 64 位安装包应选择类似：

```text
mosquitto-2.1.2-install-windows-x64.exe
```

安装时：

1. 选择 `Next`。
2. 接受许可协议。
3. 安装路径改为：

```text
D:\Tools\mosquitto
```

4. 完成安装。

不要使用带中文、空格或特殊符号的安装路径，后续运行命令更稳定。

## 5. 确认安装成功

打开 PowerShell，运行：

```powershell
Test-Path "D:\Tools\mosquitto\mosquitto.exe"
```

应输出：

```text
True
```

再运行：

```powershell
& "D:\Tools\mosquitto\mosquitto.exe" --version
```

应显示 Mosquitto 的版本号。

安装目录中通常能看到：

```text
mosquitto.exe          Broker 主程序
mosquitto_pub.exe      发布测试消息
mosquitto_sub.exe      订阅测试消息
mosquitto_passwd.exe   创建账户密码文件（本阶段不用）
mosquitto.conf         默认配置
```

## 6. 停止默认 Mosquitto 服务（若存在）

安装程序可能把 Mosquitto 注册成 Windows 服务，并自动占用 `1883`。先检查：

```powershell
Get-Service mosquitto -ErrorAction SilentlyContinue
```

如果显示 `Running`，以管理员身份打开 PowerShell 后执行：

```powershell
Stop-Service mosquitto
Set-Service -Name mosquitto -StartupType Manual
```

若没有任何输出，说明没有默认服务，直接进入下一步。

## 7. 创建本项目的 Broker 配置

创建目录：

```powershell
New-Item -ItemType Directory -Path "D:\mqtt-temp-fan" -Force
```

新建配置文件：

```powershell
notepad "D:\mqtt-temp-fan\mosquitto-temp-fan.conf"
```

在记事本中粘贴以下内容并保存。保存类型选“所有文件”，文件编码选 UTF-8：

```conf
# STM32_TEMP_FAN_CTRL 本地 MQTT Broker。
# 只供手机热点局域网开发使用，不能直接用于公网。

# 开发阶段免用户名密码。
allow_anonymous true

# ESP-01S、MQTTX 使用的普通 MQTT TCP 端口。
listener 1883 0.0.0.0
protocol mqtt

# 网页浏览器使用的 MQTT WebSocket 端口。
listener 9001 0.0.0.0
protocol websockets

# 在终端显示连接、订阅、发布等日志。
log_type all
```

检查文件名：

```powershell
Get-ChildItem "D:\mqtt-temp-fan" -Force
```

必须是：

```text
mosquitto-temp-fan.conf
```

如果看到 `mosquitto-temp-fan.conf.txt`，执行：

```powershell
Rename-Item "D:\mqtt-temp-fan\mosquitto-temp-fan.conf.txt" "mosquitto-temp-fan.conf"
```

## 8. 检查端口占用

启动前检查：

```powershell
netstat -ano | findstr ":1883"
netstat -ano | findstr ":9001"
```

没有输出表示端口空闲。

如果出现 `LISTENING`，最后一列是进程 PID。例如 PID 为 `1234`，查询：

```powershell
Get-Process -Id 1234
```

如果它是 `mosquitto`，通常说明默认服务还在运行，应回到第 6 节停止服务。

## 9. 启动本地 Broker

打开一个普通 PowerShell 窗口，运行：

```powershell
& "D:\Tools\mosquitto\mosquitto.exe" -c "D:\mqtt-temp-fan\mosquitto-temp-fan.conf" -v
```

正常应出现类似：

```text
Opening ipv4 listen socket on port 1883
Opening websockets listen socket on port 9001
mosquitto version ... running
```

最关键的是 `1883` 和 `9001` 都成功监听。

> 这个 PowerShell 窗口不能关闭。关闭窗口就等于关闭 MQTT Broker。停止 Broker 时按 `Ctrl + C`。

## 10. 本机端口测试

不要关闭 Broker 窗口，另开一个 PowerShell，执行：

```powershell
Test-NetConnection 127.0.0.1 -Port 1883
Test-NetConnection 127.0.0.1 -Port 9001
```

两条命令都应看到：

```text
TcpTestSucceeded : True
```

## 11. 使用命令行完成发布/订阅测试

保持 Broker 窗口运行，再打开第二个 PowerShell 作为订阅端：

```powershell
& "D:\Tools\mosquitto\mosquitto_sub.exe" -h 127.0.0.1 -p 1883 -t "temp-fan/TEMP-FAN-001/#" -v
```

该窗口会等待消息，这是正常现象。

打开第三个 PowerShell 作为发布端：

```powershell
& "D:\Tools\mosquitto\mosquitto_pub.exe" -h 127.0.0.1 -p 1883 -t "temp-fan/TEMP-FAN-001/status" -m "hello mqtt"
```

回到订阅窗口，应看到：

```text
temp-fan/TEMP-FAN-001/status hello mqtt
```

这证明：

```text
发布端 -> Broker -> 订阅端
```

整条 MQTT 消息中转链路已工作。

## 12. 使用热点 IP 再测一次

先确认电脑当前热点 IPv4：

```powershell
ipconfig
```

当前记录为：

```text
10.131.143.31
```

订阅：

```powershell
& "D:\Tools\mosquitto\mosquitto_sub.exe" -h 10.131.143.31 -p 1883 -t "temp-fan/TEMP-FAN-001/#" -v
```

发布：

```powershell
& "D:\Tools\mosquitto\mosquitto_pub.exe" -h 10.131.143.31 -p 1883 -t "temp-fan/TEMP-FAN-001/status" -m "hotspot ip test"
```

订阅端应收到：

```text
temp-fan/TEMP-FAN-001/status hotspot ip test
```

这一步成功后，ESP-01S 后续应连接：

```text
Broker IP：10.131.143.31
Broker Port：1883
```

## 13. 设置热点网络为专用网络

查看网络配置：

```powershell
Get-NetConnectionProfile
```

找到热点对应的 `InterfaceAlias`，通常是 `WLAN`。若 `NetworkCategory` 为 `Public`，以管理员身份打开 PowerShell：

```powershell
Set-NetConnectionProfile -InterfaceAlias "WLAN" -NetworkCategory Private
```

这会让 Windows 防火墙允许局域网设备更容易访问电脑服务。

## 14. 放行 Windows 防火墙

仍以管理员身份打开 PowerShell，执行：

```powershell
New-NetFirewallRule -DisplayName "TEMP FAN MQTT 1883" -Direction Inbound -Protocol TCP -LocalPort 1883 -Action Allow -Profile Private

New-NetFirewallRule -DisplayName "TEMP FAN WebSocket 9001" -Direction Inbound -Protocol TCP -LocalPort 9001 -Action Allow -Profile Private
```

检查规则：

```powershell
Get-NetFirewallRule | Where-Object DisplayName -Like "TEMP FAN*"
```

## 15. MQTTX 测试

下载：[MQTTX 官方下载页](https://mqttx.app/zh/downloads)

创建连接：

```text
Name：TEMP_FAN_LOCAL
Host：10.131.143.31
Port：1883
Protocol：mqtt
Username：留空
Password：留空
```

连接后订阅：

```text
temp-fan/TEMP-FAN-001/#
```

发布状态测试：

主题：

```text
temp-fan/TEMP-FAN-001/status
```

内容：

```json
{
  "temperature": 26.5,
  "humidity": 58.0,
  "mode": "auto",
  "fanDuty": 0,
  "alarm": false
}
```

发布控制测试：

主题：

```text
temp-fan/TEMP-FAN-001/command
```

内容：

```json
{
  "mode": "manual",
  "fan": "on",
  "fanDuty": 80
}
```

## 16. 常见错误

| 现象 | 常见原因 | 处理方式 |
|---|---|---|
| `Address already in use` | 1883/9001 被占用 | 执行第 8 节的 `netstat` 查 PID |
| 只看到 1883，没有 9001 | 配置文件未生效或写错 | 确认 `protocol websockets` 和 `.conf` 扩展名 |
| MQTTX 无法连接电脑 IP | 防火墙或热点网络为 Public | 完成第 13、14 节 |
| `127.0.0.1` 可连，热点 IP 不可连 | 防火墙未放行或手机热点隔离客户端 | 放行端口；先确认热点允许设备互访 |
| ESP 连不上 | ESP 和电脑不在同一热点，或电脑 IPv4 变了 | 两者都连同一个热点，重新查 `ipconfig` |
| 网页连接失败 | 网页误连 1883 | 浏览器必须使用 `ws://电脑IP:9001` |

## 17. 本阶段完成标准

完成以下全部项目后，才进入 STM32 MQTT 代码阶段：

```text
[ ] mosquitto.exe 可以显示版本号
[ ] Broker 能同时监听 1883 与 9001
[ ] Test-NetConnection 的两个端口均为 True
[ ] mosquitto_pub 发布后，mosquitto_sub 能收到
[ ] MQTTX 能连接电脑热点 IPv4:1883
[ ] MQTTX 能订阅和发布 TEMP-FAN-001 的主题
```

下一阶段才是：STM32 通过 ESP-01S 连接 `电脑IP:1883`，发布 `status`，订阅 `command`。

