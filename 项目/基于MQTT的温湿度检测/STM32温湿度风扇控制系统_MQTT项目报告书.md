# 基于 STM32F103 与 MQTT 的温湿度风扇控制系统项目报告书

**副标题：本地联调、云端部署与问题复盘**  
**项目阶段：电脑端 MQTT 链路已验证，ESP-01S 与 STM32 实机接入待完成**  

---

## 摘要

本项目设计一套基于 STM32F103C8T6、AHT20 温湿度传感器、ESP-01S Wi-Fi 模块、风扇、蜂鸣器、红绿指示灯和网页前端的温湿度监测与控制系统。STM32 负责传感器采集和执行器控制，ESP-01S 负责联网与 MQTT 通信，Mosquitto 作为消息代理服务器，网页通过 MQTT over WebSocket 获取设备状态并发送控制命令。

当前已经完成电脑端本地 MQTT 环境搭建和三条通信链路验证：MQTTX 发布状态后网页能够显示温湿度与风扇状态；网页发送控制命令后 MQTTX 能够接收；MQTTX 返回 ACK 后网页能够显示控制结果。尚未完成的部分主要是 ESP-01S 固件、STM32 与 ESP-01S 串口协议、实物设备接入，以及云服务器正式部署。

**关键词：** STM32F103；ESP-01S；Mosquitto；MQTT；WebSocket；AHT20；风扇控制；物联网

---

## 1. 项目背景与目标

### 1.1 项目背景

传统的单片机温湿度项目只能在本地 OLED 上显示数据，用户必须在设备旁边才能查看。本项目在原有温湿度采集、蜂鸣器报警和风扇控制基础上加入 Wi-Fi 与网页，使手机和电脑能够查看设备状态并发送控制命令。

### 1.2 项目目标

1. STM32 周期性读取 AHT20 温湿度数据。
2. 根据阈值自动控制风扇、蜂鸣器和红绿指示灯。
3. 支持手动模式、自动模式、风扇开关或占空比控制、蜂鸣器静音。
4. ESP-01S 将设备状态发布到 MQTT Broker。
5. 网页订阅设备状态并显示温度、湿度、模式、风扇和报警信息。
6. 网页发送控制命令，设备执行后返回 ACK。
7. 先完成电脑本地联调，后续迁移到云服务器，实现异地访问。

### 1.3 当前完成情况

| 模块               | 当前状态             | 说明                               |
| ---------------- | ---------------- | -------------------------------- |
| STM32 硬件板        | 已完成主要设计与焊接       | 已验证主控板能够运行程序                     |
| AHT20 驱动         | 已学习并编写           | 包括初始化、状态判断、数据读取、CRC 和数据转换        |
| 网页前端             | 已完成基本界面与 MQTT 接口 | 可显示状态并发送命令                       |
| Mosquitto 本地服务   | 已完成              | 1883 和 9001 端口均已验证               |
| MQTTX 联调         | 已完成              | 能模拟设备发布、订阅和返回 ACK                |
| ESP-01S 实机联网     | 待完成              | 需要编写 AT 指令或专用固件                  |
| STM32 与 ESP 串口协议 | 待完成              | 需要定义串口数据格式和状态机                   |
| 云服务器部署           | 待完成              | 计划采用宝塔、Nginx、Mosquitto、域名和 HTTPS |

---

## 2. 系统总体设计

### 2.1 硬件组成

- **STM32F103C8T6：** 系统主控制器。
- **AHT20：** 通过 I2C 采集温度和湿度。
- **OLED：** 本地显示温湿度和设备状态。
- **ESP-01S：** 通过 USART1 与 STM32 通信，并连接 Wi-Fi/MQTT。
- **风扇驱动：** PA8 输出 PWM，经 AO3400A MOS 管控制两线风扇。
- **蜂鸣器驱动：** PB0 控制 S8050 三极管，驱动有源蜂鸣器。
- **红绿 LED：** PA4 红灯表示报警，PA5 绿灯表示正常。
- **三个按键：** PB12、PB13、PB14，用于阈值设置、手自动切换和静音。

### 2.2 软件与网络组成

- **STM32 固件：** 采集、判断、控制、串口通信。
- **ESP-01S 固件：** Wi-Fi 连接、MQTT 连接、消息收发、断线重连。
- **Mosquitto Broker：** 所有 MQTT 消息的中转中心。
- **MQTTX：** 调试工具，当前用来模拟 ESP-01S 和设备端消息。
- **网页前端：** 使用 MQTT.js，通过 WebSocket 与 Mosquitto 通信。
- **Nginx（云端阶段）：** 提供静态网页、HTTPS 和 WSS 反向代理。

### 2.3 本地联调结构

```text
MQTTX ──普通 MQTT 1883──┐
                         │
网页 ──WebSocket 9001──> Mosquitto Broker
                         │
                         └──后续连接 ESP-01S──UART──STM32
```

### 2.4 云端最终结构

```text
浏览器 ──HTTPS/WSS 443──> Nginx ──WebSocket──> Mosquitto 9001
ESP-01S ──MQTT 1883──────────────────────────> Mosquitto 1883
```

---

## 3. MQTT 核心原理

### 3.1 MQTT 不是“服务器转换器”

MQTT 是一种轻量级发布/订阅通信协议。它不负责生成网页，也不负责存数据库。它的主要作用是让设备、网页和其他程序通过主题交换消息。

### 3.2 四个关键角色

| 角色         | 本项目对应对象          | 作用          |     |
| ---------- | ---------------- | ----------- | --- |
| Broker     | Mosquitto        | 接收消息并转发给订阅者 |     |
| Publisher  | ESP-01S、网页、MQTTX | 向某个主题发布消息   |     |
| Subscriber | 网页、ESP-01S、MQTTX | 订阅主题并接收消息   |     |
| Topic      | `temp-fan/...`   | 消息分类和路由地址   |     |

### 3.3 MQTTX 与 Mosquitto 的区别

- **Mosquitto 是服务器。** 所有消息都经过它。
- **MQTTX 是客户端调试工具。** 它只是连接 Mosquitto，用于手动发布、订阅和查看消息。
- MQTTX 关闭不会导致 Mosquitto 停止。
- Mosquitto 停止后，网页、MQTTX、ESP-01S 都无法相互通信。

### 3.4 为什么需要两个端口

|   端口 | 协议                  | 使用对象               |
| ---: | ------------------- | ------------------ |
| 1883 | 普通 MQTT/TCP         | MQTTX、ESP-01S、后端程序 |
| 9001 | MQTT over WebSocket | 浏览器网页              |
|      |                     |                    |

浏览器不能直接使用普通 MQTT TCP，因此网页需要 WebSocket 端口。1883 不能写成 883。

---

## 4. MQTT 主题与消息协议设计

### 4.1 设备 ID

当前设备 ID：

```text
TEMP-FAN-001
```

每台设备使用不同的设备 ID，避免多台设备的消息混在一起。

### 4.2 三个核心主题

| 主题                              | 方向      | 作用                  |
| ------------------------------- | ------- | ------------------- |
| `temp-fan/TEMP-FAN-001/status`  | 设备 → 网页 | 上报温湿度、风扇、模式、报警状态    |
| `temp-fan/TEMP-FAN-001/command` | 网页 → 设备 | 下发手动/自动、风扇、蜂鸣器等控制命令 |
| `temp-fan/TEMP-FAN-001/ack`     | 设备 → 网页 | 返回命令执行成功或失败         |

### 4.3 状态消息示例

```json
{
  "temperature": 30.2,
  "humidity": 65.0,
  "mode": "auto",
  "fanDuty": 50,
  "alarm": false,
  "buzzer_muted": false
}
```

### 4.4 控制命令示例

```json
{
  "mode": "manual",
  "fan": "on",
  "fanDuty": 50
}
```

### 4.5 ACK 示例

```json
{
  "success": true,
  "message": "fan duty updated",
  "fanDuty": 50
}
```

### 4.6 为什么需要 ACK

网页发布命令只说明 Mosquitto 收到了消息，不代表 STM32 已经执行。设备执行完成后返回 ACK，网页才能准确显示“控制成功”或“控制失败”。

---

## 5. 本地 MQTT 环境实现

### 5.1 Mosquitto 安装位置

```text
D:\STM32\Tools\mosquitto\mosquitto.exe
```

配置文件：

```text
D:\mqtt-temp-fan\mosquitto-temp-fan.conf
```

### 5.2 Mosquitto 配置

```conf
allow_anonymous true

listener 1883 0.0.0.0
protocol mqtt

listener 9001 0.0.0.0
protocol websockets

log_type all
```

含义：

- `allow_anonymous true`：测试阶段允许无用户名密码连接。
- `listener 1883`：提供普通 MQTT 服务。
- `listener 9001`：提供浏览器 WebSocket 服务。
- `0.0.0.0`：监听电脑所有网卡，而不是只允许本机访问。

### 5.3 网站服务器

项目目录：

```text
D:\code\webstorm code\STM32_TEMP_FAN_CTRL
```

启动命令：

```powershell
cd "D:\code\webstorm code\STM32_TEMP_FAN_CTRL"
npx http-server . -p 8080 -a 0.0.0.0 -c-1
```

本机访问：

```text
http://127.0.0.1:8080
```

局域网设备访问：

```text
http://电脑局域网IP:8080
```

### 5.4 MQTTX 配置

```text
Host：127.0.0.1
Port：1883
MQTT Version：3.1.1
Username：空
Password：空
SSL/TLS：关闭
订阅：temp-fan/TEMP-FAN-001/#
```

### 5.5 网页配置

```text
Broker：ws://127.0.0.1:9001
设备 ID：TEMP-FAN-001
用户名：空
密码：空
```

### 5.6 已完成的三条测试链路

1. MQTTX 向 `status` 发布 JSON，网页成功显示温湿度和风扇状态。
2. 网页向 `command` 发布控制命令，MQTTX 成功收到。
3. MQTTX 向 `ack` 发布确认消息，网页成功显示控制结果。

这证明网页、Mosquitto 和 MQTTX 之间的主题设计、消息格式和端口配置均已打通。

---

## 6. STM32、ESP-01S 与 MQTT 的职责划分

### 6.1 STM32 的职责

- 初始化 GPIO、I2C、UART、PWM。
- 读取 AHT20。
- 计算温湿度实际值。
- 根据阈值控制风扇、蜂鸣器和 LED。
- 接收 ESP-01S 下发的控制命令。
- 将当前状态通过 UART 交给 ESP-01S。

### 6.2 ESP-01S 的职责

- 连接 Wi-Fi。
- 连接 Mosquitto Broker。
- 订阅 `command`。
- 将命令通过 UART 转给 STM32。
- 接收 STM32 状态并发布到 `status`。
- 执行结果发布到 `ack`。
- 负责断线重连和心跳。

### 6.3 为什么不让 STM32 直接处理网页

STM32F103 的存储和网络能力有限，让它直接运行网页服务器会增加复杂度。采用 ESP-01S + MQTT 后，STM32 专注实时控制，ESP 专注联网，职责更清晰，也更容易调试。

### 6.4 推荐的串口数据格式

初期可以使用一行一个 JSON：

```text
STM32 -> ESP：{"type":"status","temperature":30.2,"humidity":65.0,"fanDuty":50}\r\n
ESP -> STM32：{"type":"command","mode":"manual","fan":"on","fanDuty":50}\r\n
```

后续数据量增大时，再考虑帧头、长度、命令字和 CRC。

---

## 7. 是否需要后端程序和数据库

### 7.1 当前阶段不需要

当前需求是：

```text
设备实时上报状态
网页实时显示
网页下发控制命令
设备返回执行结果
```

这些功能由 MQTT 已经可以完成，因此当前不必增加后端程序和 MySQL。

### 7.2 后端程序的作用

后端主要用于：

- 保存历史温湿度数据。
- 统计日、周、月曲线。
- 用户登录和权限控制。
- 多设备管理。
- 告警记录和消息推送。
- 远程修改设备配置。
- 对外提供 HTTP API。

### 7.3 数据库什么时候需要

只有当数据需要“以后还能查到”时才需要数据库。若网页只显示当前值，断电后无需保留历史记录，就不需要 MySQL。

### 7.4 后端能否使用 C++

可以。后续可使用 C++ 的 Crow、Drogon 等框架连接 Mosquitto 和数据库。但对当前项目而言，C++ 后端会引入编译、部署、依赖管理和内存安全等额外工作，因此应先完成 MQTT 设备链路，再决定是否增加后端。

---

## 8. 云服务器最终实现方案

### 8.1 服务器软件

```text
宝塔 Linux 面板
Nginx
Mosquitto
静态网页
域名与 HTTPS 证书
```

### 8.2 测试阶段连接

```text
MQTTX / ESP-01S：mqtt://服务器公网IP:1883
网页：ws://服务器公网IP:9001
```

直接连接 Mosquitto 9001 时，不需要在后面添加 `/mqtt`。

### 8.3 正式阶段连接

```text
网页：https://你的域名
网页 MQTT：wss://你的域名/mqtt
ESP-01S：mqtt://服务器公网IP:1883
```

此时 `/mqtt` 是 Nginx 中配置的反向代理路径，不是 Mosquitto 自己要求的路径。

### 8.4 Nginx WebSocket 反向代理示例

```nginx
location /mqtt {
    proxy_pass http://127.0.0.1:9001;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
}
```

### 8.5 公网 IP 与私网 IP

- `127.0.0.1`：只代表当前这台机器。
- `192.168.x.x`、`10.x.x.x`：通常是局域网或云服务器私网地址。
- 云服务器公网 IP：互联网中的手机和 ESP-01S 才能访问。

网页在用户手机上运行时，不能连接 `127.0.0.1`，因为那会指向手机自己。

---

## 9. 安全性与可靠性

### 9.1 匿名访问只用于测试

当前配置：

```conf
allow_anonymous true
```

放到公网正式运行后，应改成：

```conf
allow_anonymous false
password_file /etc/mosquitto/passwd
```

网页、MQTTX 和 ESP-01S 使用统一的 MQTT 用户名和密码。

### 9.2 端口开放原则

- 1883：只在设备确实需要公网直连时开放。
- 9001：正式阶段可只允许 Nginx 本机访问，不直接向公网暴露。
- 80：用于 HTTP 和证书验证。
- 443：用于 HTTPS/WSS。
- 宝塔面板端口：应限制访问来源并使用强密码。

### 9.3 QoS 建议

| 消息 | 建议 QoS | 原因 |
|---|---:|---|
| 周期性状态 `status` | 0 | 下一条状态很快会覆盖，允许少量丢失 |
| 控制命令 `command` | 1 | 至少送达一次，更适合控制 |
| 执行确认 `ack` | 1 | 避免网页漏掉执行结果 |

设备端需要对重复命令做好处理，因为 QoS 1 可能重复送达。

### 9.4 Retain 建议

- `status` 可以考虑 Retain，让新打开的网页立即得到最后一次状态。
- `command` 不建议 Retain，避免设备重连后误执行旧命令。
- `ack` 通常不需要 Retain。

### 9.5 断线重连

ESP-01S 需要分别处理：

1. Wi-Fi 断开后的重连。
2. MQTT 断开后的重连。
3. 重连成功后重新订阅 `command`。
4. 网络恢复后重新发布当前设备状态。

---

## 10. 我提出过的问题与最终理解

### 问题 1：MQTTX 是不是服务器

**原来的疑问：** MQTTX 是否等于 Mosquitto，是否可以直接用 MQTTX 代替服务器。

**最终理解：** MQTTX 只是一个操作和调试 MQTT 的客户端；Mosquitto 才是 Broker。MQTTX 相当于“测试遥控器和消息查看器”。

### 问题 2：MQTT 是不是把设备数据转换成服务器网页

**原来的疑问：** MQTT 是否负责将 STM32 数据直接转换成网页。

**最终理解：** MQTT 只负责传递消息。网页显示效果由 HTML、CSS、JavaScript 实现；Mosquitto 只根据主题转发 JSON 数据。

### 问题 3：1883、9001 和 883 有什么区别

**最终理解：** 正确端口是 1883，不是 883。1883 给 ESP、MQTTX 等普通 MQTT 客户端；9001 给浏览器 WebSocket 客户端。

### 问题 4：网页为什么不能使用 1883

**最终理解：** 浏览器安全模型不能直接建立普通 TCP MQTT 连接，所以 MQTT.js 需要通过 `ws://` 或 `wss://` 连接 WebSocket 端口。

### 问题 5：`ws://IP:9001/mqtt` 是否必须

**最终理解：** 网页直接连接 Mosquitto 时使用 `ws://IP:9001`，通常不加 `/mqtt`。使用 Nginx 反向代理后，才使用 `wss://域名/mqtt`。

### 问题 6：`127.0.0.1` 为什么手机不能用

**最终理解：** `127.0.0.1` 永远表示“当前设备自己”。电脑上的 `127.0.0.1` 是电脑；手机上的 `127.0.0.1` 是手机。手机访问电脑必须使用电脑局域网 IP 或云服务器公网 IP。

### 问题 7：云服务器的 `10.x.x.x` 地址能不能给手机使用

**最终理解：** 这类地址通常是云服务器私网 IP，外网设备不能直接访问。手机和 ESP-01S 应连接服务器公网 IP 或域名。

### 问题 8：Mosquitto 启动时报端口被占用

**最终理解：** 这通常不是安装失败，而是已有一个 Mosquitto 正在监听 1883/9001。先检查端口，不应重复启动第二个实例。

### 问题 9：网页 `/favicon.ico` 返回 404 是不是出错

**最终理解：** 这只表示网站没有图标，不影响 HTML、CSS、JavaScript 和 MQTT 功能。

### 问题 10：WebStorm 的 63342 和 Node 的 8080 有什么区别

**最终理解：** 63342 是 WebStorm 自带开发服务器端口；8080 是手动启动的 `http-server`。为了让手机和其他设备稳定访问，当前统一使用 8080 更清晰。

### 问题 11：电脑开热点后只要不关机就行吗

**最终理解：** 还必须保证电脑不睡眠、热点不关闭、Mosquitto 正在运行、网页 HTTP 服务正在运行。屏幕关闭没有关系。

### 问题 12：Windows 为什么无法开启移动热点

**最终理解：** Windows 移动热点通常要求先存在可共享的 Wi-Fi、网线或蜂窝数据连接。没有上游连接时开关可能变灰。长期项目更适合路由器或云服务器。

### 问题 13：是否必须使用数据库和后端

**最终理解：** 实时监控和控制只使用 MQTT 就够了。需要历史数据、用户系统、多设备管理或复杂业务时再增加后端和数据库。

### 问题 14：服务器选择系统镜像还是宝塔应用镜像

**最终理解：** 系统镜像更纯净，但所有软件都要手动安装；宝塔镜像更适合第一次部署网站。即使选择宝塔，Mosquitto 仍然需要单独安装和配置。

---

## 11. 我的最终理解（第一人称总结）

我现在理解了，这个项目不是“STM32 直接连接网页”，而是分成几个各自负责的部分。

STM32 负责真实硬件，包括读取 AHT20、判断温湿度、控制风扇、蜂鸣器和指示灯。ESP-01S 负责把 STM32 的数据送到网络，也负责把网络上的控制命令送回 STM32。Mosquitto 是整个 MQTT 通信的中转站，所有设备和网页都连接它，但设备和网页不需要彼此知道对方的 IP。

MQTTX 不是服务器，而是调试工具。当前还没有把 ESP-01S 接入时，可以用 MQTTX 模拟设备发布状态、接收命令和返回 ACK。这样先把网页和 MQTT 服务器调通，再接硬件，出现问题时更容易判断是哪一层出错。

网页不能直接连接普通 MQTT 的 1883 端口，因此网页使用 `ws://地址:9001`。MQTTX 和 ESP-01S 使用普通 MQTT 的 1883 端口。部署到云服务器后，ESP 使用公网 IP 的 1883，网页最终通过 Nginx 使用 `wss://域名/mqtt`，这样网页可以在 HTTPS 环境下安全连接。

当前项目只需要实时显示和控制，所以不需要数据库和后端。以后想保存历史曲线、做账号登录、多设备管理和告警记录时，再增加后端程序和 MySQL。整个项目应该按照“硬件控制层、网络通信层、MQTT 消息层、网页显示层”逐层完成，而不是一次把所有功能混在一起。

---

## 12. 验收测试清单

### 12.1 本地电脑端

- [x] 1883 端口监听成功。
- [x] 9001 WebSocket 端口监听成功。
- [x] MQTTX 能连接 1883。
- [x] 网页能连接 9001。
- [x] MQTTX 发布 `status`，网页显示成功。
- [x] 网页发布 `command`，MQTTX 接收成功。
- [x] MQTTX 发布 `ack`，网页显示成功。

### 12.2 ESP-01S 接入阶段

- [ ] ESP-01S 能连接指定 Wi-Fi。
- [ ] ESP-01S 能连接 Broker 1883。
- [ ] ESP-01S 能订阅 `command`。
- [ ] STM32 状态能通过 UART 发送给 ESP。
- [ ] ESP 能发布 `status`。
- [ ] 网页命令能通过 ESP 到达 STM32。
- [ ] STM32 执行后能返回 ACK。
- [ ] Wi-Fi 和 MQTT 断开后能自动重连。

### 12.3 云服务器阶段

- [ ] Mosquitto 安装并开机自启。
- [ ] 1883 和 9001 在服务器内部正常监听。
- [ ] 阿里云防火墙和系统防火墙配置正确。
- [ ] 静态网站部署到 Nginx。
- [ ] 域名解析完成。
- [ ] HTTPS 证书申请成功。
- [ ] `wss://域名/mqtt` 连接成功。
- [ ] 关闭匿名访问并启用账号密码。

---

## 13. 结论

本项目已经完成 MQTT 电脑端通信闭环，证明主题结构、JSON 消息格式、普通 MQTT 端口、WebSocket 端口和网页逻辑能够协同工作。下一阶段不应继续增加新的云端功能，而应优先完成 ESP-01S 与 STM32 的实机通信，将 MQTTX 模拟设备逐步替换为真实设备。硬件链路稳定后，再迁移到云服务器并完成域名、HTTPS、WSS 和账号认证。

---

## 附录 A：本地启动命令

### A.1 启动网站

```powershell
cd "D:\code\webstorm code\STM32_TEMP_FAN_CTRL"
npx http-server . -p 8080 -a 0.0.0.0 -c-1
```

### A.2 检查端口

```powershell
Test-NetConnection 127.0.0.1 -Port 1883
Test-NetConnection 127.0.0.1 -Port 9001
Test-NetConnection 127.0.0.1 -Port 8080
```

### A.3 本地访问地址

```text
网页：http://127.0.0.1:8080
网页 MQTT：ws://127.0.0.1:9001
MQTTX：mqtt://127.0.0.1:1883
```

## 附录 B：主题汇总

```text
temp-fan/TEMP-FAN-001/status
temp-fan/TEMP-FAN-001/command
temp-fan/TEMP-FAN-001/ack
```

## 附录 C：参考资料

1. Eclipse Mosquitto 配置手册。
2. MQTT 3.1.1 / MQTT 5.0 协议规范。
3. MQTT.js 官方文档。
4. Nginx WebSocket 代理文档。
5. 阿里云轻量应用服务器与宝塔面板文档。
