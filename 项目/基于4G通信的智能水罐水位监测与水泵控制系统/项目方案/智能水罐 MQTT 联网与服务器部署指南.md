# 1. 部署方案的选择
 端口分布：
 ![[image/EMQX_Windows_SSH隧道故障排查与无窗口自启动解决文档 (1).docx]]

| 方案               | 资源占用 | 上手难度 | 设备承载量   | 核心优势               | 适合你的阶段    |
| ---------------- | ---- | ---- | ------- | ------------------ | --------- |
| 原生 Mosquitto     | 最低   | 最简单  | 几十～百台以内 | 调试方便、命令运维、贴合 Linux | 现阶段主力首选   |
| Docker Mosquitto | 中等   | 中等   | 百台内     | 环境隔离、易迁移           | 后期环境杂乱再换  |
| Docker EMQX      | 最高   | 偏复杂  | 上万台     | 高并发、可视化、功能全        | 设备大规模扩容升级 |

- 上次温湿度已经用过了这个原生 Mosquitto，所以这个项目就直接Docker EMQX
- Docker 是应用运行环境，不是第二台服务器。它把 EMQX 与宝塔、Nginx、MySQL 等已有服务隔离开，EMQX 配置、数据和日志都放在独立目录，便于重启、升级、备份和回滚。
- EMQX 是 MQTT Broker。它负责转发设备状态和网页控制命令，并提供 Dashboard，可查看在线客户端、订阅、主题通信、认证和授权规则。
实现效果：
```text

MQTTX 模拟水罐设备

        │ MQTT TCP

        v

腾讯云轻量服务器

        └─ Docker -> EMQX Broker

                │ MQTT WebSocket

                v

          网页端控制台

```
# 2. 安装Docker
## 2.1 阶段目标
本阶段不是部署 EMQX，也不是开放 MQTT 端口，而是先给服务器安装一套稳定的**容器运行环境**。
完成后的关系是：

```
腾讯云服务器
├── 宝塔
├── Nginx
├── MySQL
└── Docker
    └── 后续运行 EMQX 容器
```

Docker 可以理解为给 EMQX 准备的独立运行空间。以后启动、停止、升级 EMQX，主要操作 Docker 容器，不需要把 EMQX 的程序文件直接混进宝塔、Nginx 或 MySQL 的目录。
## 2.2 检查服务器系统
- 原因：防止软件包冲突，
## 2.3 确认软件源
## 2.4 刷新软件源缓存
## 2.5 检查 Docker 软件包能不能找到
## 2.5 正式安装 Docker
## 2.6 启动 Docker，并设置开机自启
```
systemctl enable --now docker
```

- `--now` 立即启动Docker 服务
- `enable`让 Docker 在服务器重启后自动启动
## 2.7 检查 Docker 组件版本
## 2.8 运行 `hello-world` 测试容器
## 2.9 检查当前运行中的容器
# 3. 创建 EMQX 工程目录和 Compose 配置
## 3.1 阶段目标

```
/opt/smart-water-tank/emqx/
├── compose.yml    # EMQX 容器启动配置   写明 EMQX 用哪个版本、映射哪些端口、数据放在哪里  
├── data/          # 认证、授权、运行数据 保存账号、认证、授权和运行数据
└── log/           # EMQX 运行日志 保存 EMQX 运行日志
```       
 
## 3.1 检查端口和系统安全状态
## 3.2 下载固定版本 EMQX 镜像
Docker 镜像：已经封装好 EMQX 程序和运行环境的安装模板
固定版本：不使用这个`latest`,使用这个`6.2.2`
固定版本原因：不会因为 `latest` 自动变化而偷偷换版本。Docker Hub 当前提供 `emqx/emqx:6.2.2`，并支持你服务器使用的 `linux/amd64` 架构
### 下载镜像
```bash 
docker pull emqx/emqx:6.2.2
```
### 检查镜像
```bash
docker image ls emqx/emqx
```
## 3.3 确认 EMQX 使用什么用户运行
### 为啥要确认
服务器上的 `data` 和 `log` 目录刚创建时属于 `root`。
但 EMQX 容器通常使用普通用户运行。若普通用户不能写入这两个目录，启动时就可能报： `Permission denied`
### 查询
```bash
docker run --rm --entrypoint id emqx/emqx:6.2.2
```
## 3.4 创建 `compose.yml`
### Compose 文件
EMQX 的“启动说明书”
- 使用 EMQX 6.2.2；
- **容器**叫什么；
- **节点名**称是什么；
- **映射**哪些端口；
- **数据保存到服务器**哪里；
- 容器异常退出后是否自动恢复；
- 怎样判断 EMQX 是否真正健康
### 创建配置文件
```bash
cat > compose.yml <<'EOF'
services:
  emqx:
    image: emqx/emqx:6.2.2//使用那个版本
    container_name: water-tank-emqx//容器名称
    hostname: water-tank-broker//固定节点名
    restart: unless-stopped//自动恢复

    environment:
      EMQX_NODE_NAME: emqx@water-tank-broker

    ports:
      - "1883:1883"
      - "8083:8083"
      - "18083:18083"//端口映射

    volumes:
      - ./data:/opt/emqx/data
      - ./log:/opt/emqx/log//数据持久化

    healthcheck:
      test: ["CMD", "/opt/emqx/bin/emqx", "ctl", "status"]//验证这个EMQX内部服务是否正常
      interval: 10s
      timeout: 10s
      retries: 12
      start_period: 30s
EOF
```
## 3.5 查看并验证配置

```bash
docker compose config
```
# 4. 启动 EMQX
**切换制定目录**
```bash
cd /opt/smart-water-tank/emqx
```
**启动**
```bash
docker compose up -d
```
**查看**
```bash
docker compose ps
```
**再等待约 30～60 秒后查看健康状态:**
```bash
docker inspect water-tank-emqx \
  --format 'Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}'
```
**查看最近日志**
```bash 
docker logs --tail 100 water-tank-emqx
```
**检查端口**
```bash
ss -lntp | grep -E ':(1883|8083|18083)\b'
```
# 5. 初始化 EMQX Dashboard 与访问控制
## 5.1 先确认 EMQX 已经健康
**进入工程**： 
```bash
cd /opt/smart-water-tank/emqx
```
**等待几十秒**
```bash
sleep 40//等待容器内部初始化、加载配置、启动监听端口、健康探针就绪
```
**检查容器和EMQX**
```bash
docker inspect water-tank-emqx \
  --format 'Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}'
```
**检查内部节点**
```bash
docker exec water-tank-emqx \ /opt/emqx/bin/emqx ctl status
```
## 5.2 修改 Erlang Cookie
**为什么要修改**
- 多台 EMQX 节点组成集群，节点之间要互相通信（同步订阅、消息、配置、连接状态）；
- Erlang 集群通信依靠**Cookie 密钥鉴权**：两个节点 Cookie 完全一致，才允许建立 TCP 互联、互相调用内部接口；Cookie 不一样，直接拒绝互通；
- 哪怕单机单节点，EMQX 本地进程内部也会走 Erlang 分布式逻辑，本机 EMQX 各个子进程之间校验 Cookie
- 类比：Cookie = 集群局域网的门禁密码。所有 EMQX 大门必须密码相同才能串门。
**自动执行**
```bash
cd /opt/smart-water-tank/emqx

umask 077
printf 'EMQX_NODE_COOKIE=%s\n' "$(openssl rand -hex 32)" > .env
chmod 600 .env     //openssl 生成高强度随机密钥 → 杜绝弱密码；   写入.env → Docker 容器加载配置，EMQX 替换默认 Cookie；
```
## 5.3 修改 `compose.yml`
**进行备份**
```bash
cd /opt/smart-water-tank/emqx
cp compose.yml compose.yml.bak-$(date +%Y%m%d-%H%M%S)
```
**直接写入新配置**
```bash
cat > compose.yml <<'EOF'
services:
  emqx:
    image: emqx/emqx:6.2.2
    container_name: water-tank-emqx
    hostname: water-tank-broker
    restart: unless-stopped

    environment:
      EMQX_NODE_NAME: emqx@water-tank-broker
      EMQX_NODE__COOKIE: ${EMQX_NODE_COOKIE}

    ports:
      - "1883:1883"
      - "8083:8083"
      - "127.0.0.1:18083:18083"

    volumes:
      - ./data:/opt/emqx/data
      - ./log:/opt/emqx/log

    healthcheck:
      test: ["CMD", "/opt/emqx/bin/emqx", "ctl", "status"]
      interval: 10s
      timeout: 10s
      retries: 12
      start_period: 30s
**EOF**
```
**检查配置** **但不显示** **Cookie**
```bash
docker compose config >/dev/null && echo "compose 配置检查通过"
```
**生效新配置** **等待**  **检查健康情况**
```bash
docker compose up -d --force-recreate
sleep 40
docker inspect water-tank-emqx \ --format 'Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}'
```
 **检查默认** **Cookie** **警告是否消失**
 ```bash
 docker logs --tail 100 water-tank-emqx | grep -i cookie || echo "未发现 Cookie 警告"
 ```
 **检查端口绑定**
 ```bash
  ss -lntp | grep -E ':(1883|8083|18083)\b'
 ```
## 5.4安全进入 Dashboard 并初始化管理员账号
**创建SSH隧道**
- Dashboard 设置的是这个`127.0.0.1:18083`,代表只能腾讯云服务器自己能够访问 18083，互联网上其他电脑不能直接连接，这个就实现了这个访问的安全效果
- 可以使用这个SSH 隧道可以把你电脑的一个本地端口，通过 SSH 加密连接，转发到服务器本机的 `127.0.0.1:18083`。OpenSSH 的 `-L` 参数就是用于本地端口转发
- 通过上述方法，就可以达成**只有当前搭建这条 SSH 转发隧道的电脑，才能打开 EMQX 面板**，其他所有设备都无法访问
- 最终链路：
```
你的浏览器
http://127.0.0.1:18083
          │
          ▼
你的 Windows 本地端口 18083
          │
          ▼ SSH 加密隧道
腾讯云服务器 127.0.0.1:18083
          │
          ▼
EMQX Dashboard
```
**在Windows电脑建立 SSH 隧道**
- 在终端输入:`ssh -N -L 127.0.0.1:18083:127.0.0.1:18083 root@81.70.187.191`
- 会进行这个root的密码输入 输入后可能会卡在哪里， 验证可以访问你设置对应链接http://127.0.0.1:18083或者直接新打开一个终端输入以下命令
```bash
Test-NetConnection 127.0.0.1 -Port 18083
curl.exe -I http://127.0.0.1:18083
```
**配置永久SSH连接**
- Windows 登录后自动建立 EMQX Dashboard 的 SSH 隧道，并在网络断开后自动重连。
- 配置专用 SSH 密钥：`ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\emqx_tunnel_ed25519" -C "emqx-dashboard-tunnel"`
- 会直接询问这个`Enter passphrase: Enter same passphrase again:`  两次都直接按 **Enter**，暂时不设置密钥口令。原因是登录后自动任务必须能在无人输入密码的情况下启动
- 检查文件：`Get-ChildItem "$env:USERPROFILE\.ssh\emqx_tunnel_ed25519*"`  会显示两个文件
	- emqx_tunnel_ed25519 = 私钥，绝对不能给别人 
	- emqx_tunnel_ed25519.pub = 公钥，可以放到服务器
- 把公钥加入腾讯云服务器：
```
Get-Content "$env:USERPROFILE\.ssh\emqx_tunnel_ed25519.pub" |
ssh root@81.70.187.191 "umask 077; mkdir -p /root/.ssh; cat >> /root/.ssh/authorized_keys; chmod 700 /root/.ssh; chmod 600 /root/.ssh/authorized_keys"
```
- 然后进行这个密码是输入即可
- 工作逻辑
	- 读取 Windows 上的公钥
	- ↓ 
	- 通过 SSH 发给腾讯云服务器
	- ↓
	- 追加到 /root/.ssh/authorized_keys
	- ↓ 
	- 以后服务器认可这把私钥
- 进行测试：
```bash
ssh -i "$env:USERPROFILE\.ssh\emqx_tunnel_ed25519" `
  -o IdentitiesOnly=yes `
  -o BatchMode=yes `
  root@81.70.187.191 "echo SSH_KEY_OK"
```
- 预期显示：`SSH_KEY_OK`
- 用密钥测试 EMQX 隧道
```bash
ssh -N -T `
  -i "$env:USERPROFILE\.ssh\emqx_tunnel_ed25519" `
  -o IdentitiesOnly=yes `
  -o BatchMode=yes `
  -o ExitOnForwardFailure=yes `
  -o ServerAliveInterval=60 `
  -o ServerAliveCountMax=3 `
  -L 127.0.0.1:18083:127.0.0.1:18083 `
  root@81.70.187.191
```
- 预期
	- 不再要求密码；
	- PowerShell 停在那里；
	- 浏览器可以打开 `http://127.0.0.1:18083`。
**后台静默启动**
```
Windows 登录 → 后台静默启动 SSH 隧道 → 不弹 PowerShell 窗口 → 断线后每 10 秒自动重连 → 直接访问 http://127.0.0.1:18083
```
1. 创建自动重连脚本
```bash
$scriptDir = "$env:USERPROFILE\Scripts"
$scriptPath = "$scriptDir\Start-EmqxTunnel.ps1"

New-Item -ItemType Directory -Force -Path $scriptDir | Out-Null

@'
$ErrorActionPreference = "SilentlyContinue"

$sshExe = "$env:WINDIR\System32\OpenSSH\ssh.exe"
$keyFile = "$env:USERPROFILE\.ssh\emqx_tunnel_ed25519"
$logFile = "$env:USERPROFILE\Scripts\emqx-tunnel.log"

$sshArgs = @(
    "-N"
    "-T"
    "-i", $keyFile
    "-o", "IdentitiesOnly=yes"
    "-o", "BatchMode=yes"
    "-o", "ExitOnForwardFailure=yes"
    "-o", "ConnectTimeout=15"
    "-o", "ConnectionAttempts=1"
    "-o", "ServerAliveInterval=60"
    "-o", "ServerAliveCountMax=3"
    "-L", "127.0.0.1:18083:127.0.0.1:18083"
    "root@81.70.187.191"
)

while ($true) {
    & $sshExe @sshArgs

    $time = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$time SSH隧道已断开，退出码：$LASTEXITCODE，10秒后重新连接。" |
        Add-Content -Path $logFile

    Start-Sleep -Seconds 10
}
'@ | Set-Content -Path $scriptPath -Encoding UTF8

Write-Host "脚本已经创建：$scriptPath"
```
2. 检测脚本是否存在
```bash
Get-Item "$env:USERPROFILE\Scripts\Start-EmqxTunnel.ps1"
```
3. 创建window计划任务
```bash
开始菜单
→ 搜索 PowerShell
→ 右键
→ 以管理员身份运行
```
-  创建任务
```bash
$taskName = "EMQX Dashboard SSH Tunnel"
$scriptPath = "$env:USERPROFILE\Scripts\Start-EmqxTunnel.ps1"
$userId = "$env:USERDOMAIN\$env:USERNAME"
$powerShellExe = "$env:SystemRoot\System32\WindowsPowerShell\v1.0\powershell.exe"

$action = New-ScheduledTaskAction `
    -Execute $powerShellExe `
    -Argument "-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File `"$scriptPath`""

$trigger = New-ScheduledTaskTrigger `
    -AtLogOn `
    -User $userId

$principal = New-ScheduledTaskPrincipal `
    -UserId $userId `
    -LogonType Interactive `
    -RunLevel Limited

$settings = New-ScheduledTaskSettingsSet `
    -StartWhenAvailable `
    -AllowStartIfOnBatteries `
    -DontStopIfGoingOnBatteries `
    -MultipleInstances IgnoreNew `
    -RestartCount 10 `
    -RestartInterval (New-TimeSpan -Minutes 1) `
    -ExecutionTimeLimit ([TimeSpan]::Zero)

Register-ScheduledTask `
    -TaskName $taskName `
    -Action $action `
    -Trigger $trigger `
    -Principal $principal `
    -Settings $settings `
    -Description "登录Windows后自动维护EMQX Dashboard SSH隧道。" `
    -Force
```
- 两层直连
```
SSH连接断开 → 脚本每10秒重连 整个PowerShell脚本意外退出 → 任务计划程序1分钟后重新启动
```
- 测试
```bash
Start-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"//手动启动
Start-Sleep -Seconds 15//等待十五秒
Get-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel" |
    Select-Object TaskName, State//查看任务状态
```
- 检查隧道
```bash
Test-NetConnection 127.0.0.1 -Port 18083
curl.exe -I http://127.0.0.1:18083    //测试网页响应
```
- 验证开机自启动
```bash
Get-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel" |
    Select-Object TaskName, State
    Test-NetConnection 127.0.0.1 -Port 18083
```
- 预期结果
```
State : Running TcpTestSucceeded : True
```
**常用管理命令**
1. 查看任务
```
Get-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"
```
2. 停止隧道
```
Stop-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"
```
3. 重新启动隧道
```
Stop-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"
Start-Sleep -Seconds 2
Start-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"
```
4. 禁止开机启动
```
Disable-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"
```
5. 恢复开机启动
```
Enable-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"
```
6. 查看断线记录
```
Get-Content "$env:USERPROFILE\Scripts\emqx-tunnel.log" -Tail 30
```
7. 彻底删除任务
```
Unregister-ScheduledTask `
    -TaskName "EMQX Dashboard SSH Tunnel" `
    -Confirm:$false
```
## 5.5配置 MQTT 客户端认证

**进行客户端认证**
认证方式：Password-Based 
数据源：内置数据库 Built-in Database
配置：
- 账号类型：username
- 密码哈希：bcrypt
- 其他参数默认
**创建两个MQTT账号**
- 设备账号：用户名：water_tank_001 密码：water_tank_001 超级用户：关闭
- 使用情景：MQTTX模拟设备 STM32 + Air780E真实设备
- 建议的连接参数：
```
Client ID：water_tank_001 Username：water_tank_001 Password：你刚创建的设备密码
```
- 网页账号:用户名：water_web 密码：另一个不同的强密码 超级用户：关闭
- 使用情景：MQTTX模拟网页 网页控制台
 - 建议的连接参数：
```
Client ID：water_web Username：water_web Password：water_web
```
# 6 MQTT Topic 授权
## 6.1 创建授权数据库
- 保存一张 **MQTT 权限表** 告诉EMQX  某个账号登录以后，允许发布哪些 Topic，允许订阅哪些 Topic，其他操作是否拒绝
```
客户端授权
新建
Built-in Database
Max Rules：100
```
## 6.2 检查授权总设置
```bash
No Match Action：Deny 
Deny Action：Ignore 
Authorization Cache：开启
没有明确允许 → 默认拒绝 
发现违规发布或订阅 → 拒绝这次操作，但测试阶段先不把客户端踢下线
```
## 6.3 配置设备账号权限
```
在 Built-in Database 这一行点击：权限 / Permissions
进入：Username / 用户名
```
1. 第一个用户名填写：water_tank_001
2. 添加五个权限：上传状态 上传报警 上传执行结果 接收控制命令 接收参数配置

|Permission|Action|Topic|QoS|Retain|
|---|---|---|---|---|
|Allow|Publish|`water_tank/water_tank_001/status`|0、1|true|
|Allow|Publish|`water_tank/water_tank_001/alarm`|0、1|false|
|Allow|Publish|`water_tank/water_tank_001/ack`|0、1|false|
|Allow|Subscribe|`water_tank/water_tank_001/command`|0、1|All|
|Allow|Subscribe|`water_tank/water_tank_001/config`|0、1|All|

2. 第二个用户名填写：water_web
3. 添加五个权限：接收设备状态 接收报警 接收执行结果 发送控制命令 发送参数配置

| Permission | Action    | Topic                               | QoS | Retain |
| ---------- | --------- | ----------------------------------- | --- | ------ |
| Allow      | Subscribe | `water_tank/water_tank_001/status`  | 0、1 | All    |
| Allow      | Subscribe | `water_tank/water_tank_001/alarm`   | 0、1 | All    |
| Allow      | Subscribe | `water_tank/water_tank_001/ack`     | 0、1 | All    |
| Allow      | Publish   | `water_tank/water_tank_001/command` | 0、1 | false  |
| Allow      | Publish   | `water_tank/water_tank_001/config`  | 0、1 | false  |
4. 不用一个通配符直接放行：设备若获得全目录发布权限，就能伪造控制命令；网页若获得全目录发布权限，就能伪造传感器状态
## 6.4 MQTTX 联调
**给本机增加1883 SSH 隧道**
1. 原因：现在自动任务只转发18083：EMQX Dashboard，但是这个MQTTX需要1883：MQTT通信
2. 停止自动任务：
```bash
Stop-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"
Start-Sleep -Seconds 3
```
3. 打开脚本
```bash
notepad "$env:USERPROFILE\Scripts\Start-EmqxTunnel.ps1"
```
4. 进行替换
```bash
$ErrorActionPreference = "SilentlyContinue"

$sshExe = "$env:WINDIR\System32\OpenSSH\ssh.exe"
$keyFile = "$env:USERPROFILE\.ssh\emqx_tunnel_ed25519"
$logFile = "$env:USERPROFILE\Scripts\emqx-tunnel.log"

$sshArgs = @(
    "-N"
    "-T"

    "-i", $keyFile

    "-o", "IdentitiesOnly=yes"
    "-o", "BatchMode=yes"
    "-o", "ExitOnForwardFailure=yes"
    "-o", "ConnectTimeout=15"
    "-o", "ConnectionAttempts=1"
    "-o", "ServerAliveInterval=60"
    "-o", "ServerAliveCountMax=3"

    # EMQX Dashboard 管理后台
    "-L", "127.0.0.1:18083:127.0.0.1:18083"

    # EMQX MQTT 服务，供 MQTTX 测试
    "-L", "127.0.0.1:1883:127.0.0.1:1883"

    "root@81.70.187.191"
)

while ($true) {
    & $sshExe @sshArgs

    $time = Get-Date -Format "yyyy-MM-dd HH:mm:ss"

    "$time SSH隧道已断开，退出码：$LASTEXITCODE，10秒后重新连接。" |
        Add-Content -Path $logFile

    Start-Sleep -Seconds 10
}
```
5. 重新启动任务
```bash
Start-ScheduledTask -TaskName "EMQX Dashboard SSH Tunnel"
```
6. 等待十五秒
```bash
Start-Sleep -Seconds 15
```
7. 检查两个隧道和Dashboard
```bash
Test-NetConnection 127.0.0.1 -Port 18083
Test-NetConnection 127.0.0.1 -Port 1883
curl.exe -I http://127.0.0.1:18083
```
**MQTTX创建两个连接**
1. 设备模拟器
```
名称：Water Tank Device Simulator
Client ID：water_tank_001_sim
Username：water_tank_001
Host：127.0.0.1
Port：1883
```
 2. 订阅 `water_tank/water_tank_001/command water_tank/water_tank_001/config`
 3. 网页模拟器
 ```
 名称：Water Tank Web Simulator 
 Client ID：water_web_sim U
 sername：water_web 
 Host：127.0.0.1 
 Port：1883
 ```
 4. 订阅：`water_tank/water_tank_001/status water_tank/water_tank_001/alarm water_tank/water_tank_001/ack`
 5. 所有订阅先使用这个：`QoS：1`
**MQTTX联调**
 - MQTTX两个连接同时在线  
	 - 设备上传状态(设备模拟器)
	 - 网页下发启动命令（网页模拟器）
	 - 设备返回命令ACK（设备模拟器）
	 - 设备上传报警（设备模拟器）
	 - 网页下发参数配置（网页模拟器）
- 断开网页模拟器
	- Status保留消息：只断开网页连接，设备模拟器保持在线，设备发布新状态，恢复网页连接
- 网页越权发布Status
- 设备越权发布Command
# 7. 开发与部署网页端
## 7.1 检查并固定服务器端口架构

```
EMQX Dashboard：
你的浏览器
→ 127.0.0.1:18083
→ SSH
→ EMQX
```
```
MQTTX模拟设备：
MQTTX
→ 127.0.0.1:1883
→ SSH
→ EMQX
```
**网页用户使用**
```
你的电脑、其他电脑、手机
→ http://81.70.187.191
→ 公网80
→ 宝塔Nginx
   ├─ /      返回网页文件
   └─ /mqtt  转发到服务器内部127.0.0.1:8083
                                  ↓
                                 EMQX
```
**端口作用**

| 端口      | 最终用途                    |
| ------- | ----------------------- |
| `18083` | 你通过 SSH 管理 EMQX         |
| `1883`  | 你电脑上的 MQTTX 通过 SSH 模拟设备 |
| `8083`  | 服务器内部给 Nginx 连接 EMQX    |
| `80`    | 所有电脑和手机访问网页             |
| `443`   | 后期 HTTPS＋WSS            |
**检查服务器端口**
```
docker compose ps  //查看docker容器状态
docker compose config //查看 compose 完整端口映射配置
ss -lntp | grep -E ':(18083|1883|8083)\b' //宿主机实际端口监听核查
docker inspect water-tank-emqx --format 'Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}'//EMQX 容器运行 + 健康检测
curl -I http://127.0.0.1:18083/     //检测 18083 管理面板


//检测8083 MQTT over Websocket
curl --http1.1 -i --max-time 5 \ 
-H "Connection: Upgrade" \ 
-H "Upgrade: websocket" \ 
-H "Sec-WebSocket-Version: 13" \
-H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \ 
-H "Sec-WebSocket-Protocol: mqtt" \ 
http://127.0.0.1:8083/mqtt
```
**端口收口**
这个移步到cloud server仓库  文章为**docker容器端口收口**
## 7.2 创建宝塔网站入口
>先不要急着**上传正式网页**，也先不要 **改Nginx配置**。第一步是检查服务器的 `80` 端口和宝塔现有网站，避免和已有网站冲突。

**服务器只读检查**
```bash
ss -lntp | grep -E ':80\b'       //80端口有没有程序监听
ps -ef | grep '[n]ginx'          //宝塔Nginx有没有运行   查看Nginx 工作进程
curl -I http://127.0.0.1/        //服务器本机访问80返回什么网页
```
```
0.0.0.0:80 
[::]:80      
-这个显示了80端口监听所有端口  80端口就是故意面向公网开放的网页入口


服务器本机访问80返回什么网页
- 在服务器本机访问 Nginx 根路径，成功返回 200；
- 证明：Nginx 本地访问完全没问题，Nginx 配置无语法错误、本地网络通畅；
- 网页静态服务、Nginx 本身功能完好。
```
## 7.3 宝塔创建智能水罐静态网站
**正式域名方案**
```
电脑或手机
→ http://water.example.com
→ 默认公网80
→ Nginx
   ├─ /      → 网页
   └─ /mqtt  → 127.0.0.1:8083
                         ↓
                        EMQX
```
网页地址：
```
http://water.example.com
```
MQTT WebSocket地址：
```
ws://water.example.com/mqtt
```
特点：

- 使用域名
- HTTP默认走80
- 后期容易升级到443
- 可以申请HTTPS证书
- 适合正式部署
**IP＋18080测试方案**
```
电脑或手机
→ http://81.70.187.191:18080
→ 公网8080
→ Nginx
   ├─ /      → 网页
   └─ /mqtt  → 127.0.0.1:8083
                         ↓
                        EMQX
```
网页地址：
```
http://81.70.187.191:18080
```
MQTT WebSocket地址：
```
ws://81.70.187.191:18080/mqtt
```
特点：

- 不使用域名；
- 地址必须明确写 `:8080`；
- 不占用现在的80端口；
- 不动 `panel_ssl_site`；
- 适合临时测试和演示。
**宝塔面板创建网站**
端口范围不合法
- 宝塔版本的“HTML项目”添加窗口
`网站->HTML项目`
宝塔版本的“HTML项目”添加窗口，不接受 `IP:端口` 这种绑定格式
- 网站 → PHP项目 → 添加站点 → 传统项目（通用）
这个虽然叫“PHP项目”，但它实际上也是宝塔的**通用网站创建入口**。只要 PHP 版本选择“纯静态”，就不会运行 PHP，实际仍然是：
`8080` 无法建站
宝塔版本把 `8080` 列入了禁止用于建站的保留端口。8080在宝塔保留端口名单中
所以改动`18080`
**在宝塔添加 `/mqtt` 代理**
```
宝塔面板
→ 网站
→ PHP项目
→ 找到 81.70.187.191:18080
→ 设置
→ 反向代理
→ 添加反向代理
```
内容：

|配置项|填写内容|
|---|---|
|开启代理|开启|
|开启缓存|关闭|
|高级功能|开启|
|代理名称|`emqx-mqtt`|
|代理目录|`/mqtt`|
|目标 URL|`http://127.0.0.1:8083`|
|发送域名|`127.0.0.1`|
|内容替换|全部留空|
**上传编写好的的网站**
**进行测试**
 

# 9. TLS/WSS 安全升级





# 遇到的问题
## EMQX 服务本身正常，但健康检查命令找不到对应的 Erlang 节点，所以 Docker 标记为 `unhealthy`


容器能运行、Dashboard 能打开，但节点名没有采用可解析的 FQDN 和 Docker 网络别名，导致 `emqx ctl status` 无法连接节点。官方建议让 Docker 的 hostname、EMQX 节点名和网络 alias 保持一致，并采用固定的 FQDN 格式
举例类比问题
```
EMQX店铺已经营业
门也已经打开
网页也能够进去

但是健康检查人员拿着：
“emqx@water-tank-broker”

去找一个没有登记完整地址的节点
所以找不到，并报告不健康
```

### 配置问题
1. 配置
```bash
hostname: water-tank-broker

environment:
  EMQX_NODE_NAME: emqx@water-tank-broker
```
2. **EMQX实际上已经启动成功**
3. 1883、8083、18083也都正常监听
4. Docker的健康检查执行：这条命令不是简单检查网页能不能打开，而是要通过EMQX底层的Erlang节点通信机制，去“呼叫”这个节点，但是返回退出码是1，健康检查命令连续返回非0，Docker就会把容器标记为
5. 最终确定的就是 EMQX底层的Erlang节点不对
### 故障现象

- EMQX容器状态为running；
- Dashboard访问返回HTTP 200；
- MQTT相关端口正常监听；
- Docker健康状态为unhealthy；
- `emqx ctl status`提示节点无法响应ping。

### 故障原因

EMQX节点使用短主机名`water-tank-broker`，且Docker网络中没有与节点身份完全一致的FQDN网络别名，导致健康检查命令无法正确找到EMQX节点。

### 解决措施

- 将容器主机名改为`water-tank-broker.emqx.local`；
- 将EMQX节点名改为`emqx@water-tank-broker.emqx.local`；
- 创建独立Docker bridge网络；
- 添加`water-tank-broker.emqx.local`网络别名；
- 重新创建容器和网络；
- 保留原有data、log和.env持久化文件。

### 处理结果

- 容器状态：running；
- 健康状态：healthy；
- EMQX节点状态检查通过；
- Docker内部DNS解析正常；
- Dashboard返回HTTP 200；
- 1883、8083端口正常监听；
- 18083仅绑定127.0.0.1；
- 故障处理完成。
## SSH隧道睡眠恢复故障
### 故障现象
- Windows睡眠唤醒后Dashboard无法访问；
- 本机18083、1883均未监听；
- 计划任务状态为Ready；
- SSH日志曾连续返回退出码255。
-  本机隧道没了，表示Windows 本机目前没有可用的端口转发，所以浏览器访问失败，推测原因层：计划任务，PowerShell脚本，ssh.exe，SSH连接
- 计划任务当前没有运行：显示的是`State：Ready` 表示这个任务已注册，可以被启动，但当前并没有运行
- 日志里的 `255` ：说明这个SSH 自身发生错误时，通常返回 255，但是不能得出这个具体的原因，能推测的原因：**服务器22端口不可达 网络还没恢复 密钥认证失败 本机转发端口被占用 服务器拒绝连接 SSH主机密钥校验异常**
 
### 排查顺序
```
第一层：网页是否能打开
        ↓
第二层：本机18083/1883是否监听
        ↓
第三层：计划任务是Running还是Ready
        ↓
第四层：PowerShell和ssh.exe是否存在
        ↓
第五层：服务器22端口是否可达
        ↓
第六层：SSH密钥认证是否成功
        ↓
第七层：服务器上的EMQX是否健康
```
1. 访问EMQX Dashboard的完整路径
```
浏览器
  ↓
Windows 本机 127.0.0.1:18083
  ↓
后台 ssh.exe 隧道
  ↓
互联网
  ↓
腾讯云 81.70.187.191:22
  ↓
服务器 SSH 服务
  ↓
服务器本机 127.0.0.1:18083
  ↓
Docker 端口映射
  ↓
EMQX Dashboard
```
2. 排查思路遵循原则
- 从近到远：先检查你的电脑，再检查互联网，再检查服务器
- 一次只验证一层：每条命令只回答一个问题，不把多个问题混在一起。
- 先观察，再修改：
```
通过观察这个状态
任务状态 进程状态 端口状态 日志 网络状态 认证状态
然后推断出具体要改动对象，最后对症下药
```
### 排查思路
#### 先检查网络和 SSH 密钥
1. 原因：为了先排除这个服务器的嫌疑
2. 第一层：**服务器22端口**  测试连接  `Test-NetConnection 81.70.187.191 -Port 22` 结果 `TcpTestSucceeded : True`  
3. 第二层：**SSH密钥认证**   结果为`SSH_KEY_OK`    通过这个可以判断    私钥文件正常 服务器 authorized_keys 正常 root 公钥认证正常 SSH 握手正常
```
ssh `
  -i "$env:USERPROFILE\.ssh\emqx_tunnel_ed25519" `
  -o IdentitiesOnly=yes `
  -o BatchMode=yes `
  root@81.70.187.191 `
  "echo SSH_KEY_OK"
  
```
#### 判断计划任务是如何启动的脚本
1. 手动启动脚本，然后检查端口 端口显示的是`true`  所以最终推断出原因是 Windows 计划任务的创建或启动配置
```bash
& "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
  -NoProfile `
  -ExecutionPolicy Bypass `
  -File "C:\Users\34205\Scripts\Start-EmqxTunnel.ps1"
  Test-NetConnection 127.0.0.1 -Port 18083
Test-NetConnection 127.0.0.1 -Port 1883
```
#### 计划任务最初注册失败
1. 原因：输入下面的命令，powershell 最终返回的有效值只有这个Parallel Queue IgnoreNew ，说明你这台电脑上的 `ScheduledTasks` 模块不支持 `StopExisting` 这个枚举值
```bash
-MultipleInstances StopExisting
```
2. 最后导致这个连锁反应
```
**StopExisting 参数非法
        ↓
$settings 创建失败
        ↓
$settings 实际为 null
        ↓
Register-ScheduledTask 收到空的 Settings
        ↓
任务注册失败**
```
#### 解决措施
1. 改成：
```
-MultipleInstances IgnoreNew
如果这个任务已经有一个实例在运行，新的启动请求不再重复启动第二份。
```
2. 这对 SSH 隧道很合适: 脚本只应该运行一份。如果两份同时运行， 第一份占用 18083 和 1883 第二份也想占用 18083 和 1883，又可能会出现这个Address already in use 端口已被占用 的情况
3. 改完后检查 `$settings.MultipleInstances`  返回`IgnoreNew` ，说明`$settings` 真正创建成功了
#### 总结
欧克 可以了 这个问题是这个计划任务没有成功 没有成功的原因是这个 `ScheduledTasks` 模块不支持 `StopExisting` 这个枚举值 换成这个**`IgnoreNew`** 即可

## 电脑SSH的连接问题
```
① SSH密钥 C:\Users\34205\.ssh\emqx_tunnel_ed25519 
② 自动重连脚本 C:\Users\34205\Scripts\Start-EmqxTunnel.ps1 
③ Windows计划任务 EMQX Dashboard SSH Tunnel 
④ 后台ssh.exe进程 负责真正建立本机端口
```
### 理清当前情况
设计的端口方案

| 端口      | 用途                  | 你自己怎么访问           | 别人怎么访问            |
| ------- | ------------------- | ----------------- | ----------------- |
| `18083` | EMQX Dashboard      | 通过 Windows SSH 隧道 | 不允许访问             |
| `1883`  | MQTT TCP、MQTTX 模拟设备 | 通过 Windows SSH 隧道 | 不允许访问             |
| `8083`  | MQTT WebSocket      | 本地调试时可走 SSH       | 公开演示时由 Nginx 内部转发 |
| `80`    | 演示网页 HTTP           | 可访问               | 电脑、手机都访问这里        |
| `443`   | 后期 HTTPS＋WSS        | 后期开启              | 正式使用这里            |
端口最终设置 
80：
宝塔Nginx对外提供网页，电脑和手机访问
8083：
EMQX WebSocket，只在服务器内部给Nginx使用
18083：
你通过SSH管理EMQX
1883：
你电脑上的MQTTX通过SSH模拟设备
#### 四层排查
```
第1层：Windows SSH
18083、1883 是否正常

第2层：服务器 EMQX
18083、1883、8083 是否正常

第3层：宝塔 Nginx
80 是否正常，/mqtt 是否转到 8083

第4层：外部访问
其他电脑和手机能否通过 80 打开网页并连接 MQTT
```
#### 第一层：排查 Windows SSH
**PowerShell进行输入**
```bash
$taskName = "EMQX Dashboard SSH Tunnel"

Write-Host "`n===== 计划任务状态 ====="
Get-ScheduledTask -TaskName $taskName |
    Select-Object TaskName, State

Write-Host "`n===== 最近运行信息 ====="
Get-ScheduledTaskInfo -TaskName $taskName |
    Format-List LastRunTime,LastTaskResult,NextRunTime,NumberOfMissedRuns

Write-Host "`n===== PowerShell和SSH进程 ====="
Get-CimInstance Win32_Process |
    Where-Object {
        $_.Name -in @("powershell.exe","ssh.exe") -and
        $_.CommandLine -match "Start-EmqxTunnel|emqx_tunnel_ed25519"
    } |
    Format-List ProcessId,ParentProcessId,Name,CreationDate,CommandLine

Write-Host "`n===== 本机18083 ====="
Test-NetConnection 127.0.0.1 -Port 18083

Write-Host "`n===== 本机1883 ====="
Test-NetConnection 127.0.0.1 -Port 1883

Write-Host "`n===== 腾讯云SSH端口 ====="
Test-NetConnection 81.70.187.191 -Port 22

Write-Host "`n===== 最近SSH日志 ====="
Get-Content "$env:USERPROFILE\Scripts\emqx-tunnel.log" -Tail 20
```
**判断方法**
定时任务:`Ready就绪`/`Running正在运行`/`Disabled禁用`
定时任务详细运行记录，判断是否有`ssh.exe`（真正的 SSH 端口转发进程）

|结果|说明|
|---|---|
|任务 `Running`、有 `ssh.exe`、两端口均为 `True`|Windows SSH 完全正常|
|任务 `Ready`、没有 `ssh.exe`|计划任务没有持续运行|
|任务 `Running`、没有 `ssh.exe`|PowerShell 脚本在运行，但 SSH 启动失败或反复重试|
|`18083` 和 `1883` 都为 `False`|SSH 隧道没有建立|
|腾讯云 `22` 为 `False`|当前网络、腾讯云安全组或服务器 SSH 服务异常|
|日志持续出现 `255`|SSH 发生错误，但旧日志没有记录具体错误内容|



# 工作流
> STM32通过串口控制Air780E，Air780E内部完成SIM联网、TCP和MQTT协议

1. 用户：`water_web` 和 `water_tank_001` 
	- Client ID：water_web_随机编号 
	- Username：water_web 
	- Password：网页MQTT密码
2. 服务器端口方案

|         端口 | 是否对外开放 | 用途                  | 访问方式          |
| ---------: | ------ | ------------------- | ------------- |
|       `80` | 开放     | HTTP入口              | 自动跳转到443      |
|      `443` | 开放     | HTTPS网页和MQTT WSS    | 浏览器正式入口       |
|     `8083` | 不开放    | EMQX MQTT WebSocket | 仅Nginx内部转发    |
| `88/18080` | 临时开放   | 当前IP测试网页            | 正式上线后关闭       |
|    `18083` | 不开放    | EMQX管理后台            | Windows SSH隧道 |
|     `1883` | 不开放    | MQTTX明文测试           | Windows SSH隧道 |
|     `8883` | 开放     | Air780E MQTT TLS    | 真实设备入口        |
 - 网页：
	  - 通过访问：`http://81.70.187.191:88`
	  - Nginx收到`/mqtt`后转发到：`http://127.0.0.1:8083`
	  - ![[Pasted image 20260810112914.png]]
- EMQX Dashboard
```
浏览器 → 127.0.0.1:18083 → Windows SSH隧道 → 腾讯云127.0.0.1:18083 → EMQX Dashboard
```
- 网页：
```
测试：
浏览器 → http://81.70.187.191:88 → Nginx返回网页 网页JavaScript → ws://81.70.187.191:88/mqtt → Nginx代理 → 127.0.0.1:8083 → EMQX


正式：
用户浏览器
→ 公网80/443
→ 宝塔Nginx
├─ /       返回网页文件
└─ /mqtt   转发到127.0.0.1:8083
                         ↓
                    EMQX WebSocket
```
- 测试设备：
```
MQTTX → 127.0.0.1:1883 → Windows SSH隧道 → 腾讯云1883 → EMQX
```



---
# 联网
1. 4G基础联网
2. 配置MQTT参数
	- 只是写入Air780E本地内存，模块不会访问服务器，也不会判断密码是否正确


```
真实设备 ──────┐
              │
MQTTX设备模拟器 ├──► EMQX Broker ───► MQTTX网页模拟器
              │
真实网页 ──────┘
```
```
Mosquitto：轻量、基础、配置文件为主 
EMQX：功能完整、管理能力强、带Dashboard
```
两种方案
```
真实设备 ──► Mosquitto ◄── 真实网页
                 ▲
                 │
               MQTTX
```
```
真实设备 ──► EMQX ◄── 真实网页
               ▲
               │
             MQTTX
```


```
MQTTX连接EMQX
        ↓
MQTTX告诉EMQX：
“我要订阅water_tank/water_tank_001/status”
        ↓
EMQX记录这个订阅关系
        ↓
真实设备发布相同Topic
        ↓
EMQX查找所有订阅该Topic的客户端
        ↓
EMQX分别发送一份消息
```

```
EMQX收到消息以后，只发送给满足以下三个条件的客户端：

1. 客户端已经连接。
2. 客户端主动订阅了匹配的Topic。
3. 客户端账号拥有该Topic的订阅权限
```



--- 
# 网页连接流程
## 实际建立连接的详细过程

### 第一步：获取网页

```
浏览器向Nginx发送：
GET /
```

Nginx返回：

```
index.html
app.js
style.css
```

浏览器显示水罐页面。

### 第二步：JavaScript建立WebSocket

JavaScript向相同服务器发送：

```
GET /mqtt
Connection: Upgrade
Upgrade: websocket
```

它的含义是：

> 我不想获取普通网页，我想把这条HTTP连接升级为WebSocket长连接。

### 第三步：Nginx识别路径

Nginx根据路径判断：

```
/       → 返回网页文件
/mqtt   → 转发到EMQX
```

内部转发：

```
81.70.187.191:18080/mqtt
→ Nginx
→ 127.0.0.1:8083/mqtt
→ EMQX
```

### 第四步：发送MQTT账号密码

WebSocket建立以后，JavaScript才通过这条连接发送MQTT `CONNECT` 数据包：

```
Client ID：water_web_随机值
Username：water_web
Password：网页账号密码
```

EMQX验证成功后返回MQTT连接成功。

随后网页继续发送：

```
SUBSCRIBE status
SUBSCRIBE alarm
SUBSCRIBE ack
```

EMQX才能把设备消息实时推送给网页。

## 为什么都使用18080

这是为了只开放一个网页入口：

```
18080/
→ 网页

18080/mqtt
→ MQTT WebSocket
```

对外只开放18080，服务器内部由Nginx进行分流：

```
                   ┌─ /      → 网页文件
公网18080 → Nginx ─┤
                   └─ /mqtt  → EMQX 8083
```

所以不是先进入18080再让浏览器跳转到8083。浏览器始终只访问18080，8083只发生在服务器内部。

另外，目前使用的是：

```
HTTP + WS
```

账号密码会经过未加密网络。正式部署应升级为：

```
HTTPS + WSS
```

届时地址会变成：

```
https://你的域名
wss://你的域名/mqtt
```