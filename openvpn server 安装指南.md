# 在 Ubuntu 上部署与配置 OpenVPN 服务端

[source: 1] 在 Ubuntu 上部署 OpenVPN 服务端，最推荐且极少出错的方式是使用社区广泛认可的一键自动化安装脚本（`openvpn-install`）。

它会自动处理证书生成（Easy-RSA）、防火墙规则、IP 转发以及生成客户端配置文件（`.ovpn`）。

---

## 一、 快速安装流程

### 1. 下载并运行一键安装脚本

需要 `root` 权限。以 `root` 身份运行以下命令启动交互式安装程序：

```bash
wget https://raw.githubusercontent.com/Nyr/openvpn-install/master/openvpn-install.sh -O openvpn-install.sh
sudo bash openvpn-install.sh

```

### 2. 配置服务器交互参数

按需确认默认设置。脚本会询问几个关键设置，绝大多数情况下**直接按回车保持默认**即可：

* **Protocol**: 选择 `1` (UDP，传输速度更快)
* **Port**: 默认 `1194`
* **DNS**: 选择你偏好的 DNS（例如选择 Cloudflare 或 Google）
* **Client Name**: 输入第一个客户端的名称（例如 `client1`）

### 3. 自动生成客户端配置文件

安装完成后，脚本会在当前用户的家目录（或 `/root/`）下自动生成一个以客户端名称命名的配置文件：

```plaintext
/root/client1.ovpn

```

> **提示**：后续有新设备需要连接时，只需再次运行 `sudo bash openvpn-install.sh` 即可添加或管理客户端配置文件。

---

## 二、 检查服务状态与防火墙设置

安装完成后，确保 OpenVPN 服务正常运行，并允许相应的端口通过：

### 1. 检查 OpenVPN 服务状态

```bash
sudo systemctl status openvpn-server@server.service

```

> 如果状态显示为 `active (running)`，说明服务端已成功启动。

### 2. 开放系统防火墙（UFW）

如果你启用了 UFW 防火墙，需放行默认端口（UDP 1194）：

```bash
sudo ufw allow 1194/udp

```

> **重要注意**：如果你的 Ubuntu 部署在云服务器（如阿里云、腾讯云、AWS 等），请务必前往云厂商控制台的 **“安全组” (Security Group)** 规则中，添加一条入方向规则，允许 **UDP 端口 1194** 的流量。

---

## 三、 进阶：配置仅监听 IPv6（可选）

如果你需要将 OpenVPN 配置为纯 IPv6 监听，请按以下步骤操作：

### 第一步：修改 OpenVPN 配置文件

打开 OpenVPN 主配置文件（如果使用前面的一键脚本安装，默认路径通常为 `/etc/openvpn/server/server.conf`）：

```bash
sudo nano /etc/openvpn/server/server.conf

```

在文件中找到有关 `proto` 和 `bind` 的配置项，按以下规则修改或添加：

```conf
proto udp6
local ::
bind ipv6only

```

**参数说明**：

* `proto udp6`：告诉 OpenVPN 使用 IPv6 UDP 协议。
* `local ::`：表示绑定到服务器的所有 IPv6 地址。
* `bind ipv6only`：强制 Linux 内核将套接字设置为 `IPV6_V6ONLY = 1`，彻底关闭对 IPv4 流量的响应。

---

### 第二步：重启服务并验证

1. **重新启动 OpenVPN 服务**：
```bash
sudo systemctl restart openvpn-server@server.service

```


2. **检查运行状态**：
```bash
sudo systemctl status openvpn-server@server.service

```


> 状态应该会变成绿色的 `active (running)`。


3. **确认端口监听**：
```bash
sudo ss -tulpn | grep 1194

```


> 输出中如果包含 `[::]:1194` 且没有 `0.0.0.0:1194`，说明服务端已成功配置为**仅监听 IPv6**。



---

### 第三步：客户端 `.ovpn` 文件的对应修改

服务端改成纯 IPv6 监听后，你的客户端配置文件（`.ovpn`）也必须同步更新：

1. 打开生成的 `.ovpn` 文件。
2. 找到 `remote` 那一行，将其修改为你的**服务器 IPv6 地址**或**绑定的 IPv6 域名**：

```plaintext
# 使用 IPv6 地址（必须加方括号）
remote [2001:db8::1] 1194 udp6

# 或使用指向 IPv6 的域名
remote yourdomain.com 1194 udp6

```



要让通过 IPv6 连接到 OpenVPN 的客户端能够访问你服务器所在的 IPv4 内网（192.168.5.0/24），需要做两件事：在服务端配置路由推送，以及在服务器上开启系统 IP 转发和 SNAT 转换（因为客户端是 IPv6，需要通过网关转换成内网认识的 IPv4 流量）。

请按照以下步骤进行配置：

第一步：修改服务端配置，推送内网路由
打开 OpenVPN 服务端配置文件：

Bash
sudo nano /etc/openvpn/server/server.conf
在文件中找到（或手动添加）以下 push 指令，告诉客户端自动添加通往内网的路由：

Plaintext
push "route 192.168.5.0 255.255.255.0"
保存并退出编辑器。

第二步：开启服务器系统的 IPv4 转发
由于 OpenVPN 服务器本身充当路由网关，必须允许 Linux 内核转发不同网段之间的流量。

临时开启 IP 转发：

Bash
sudo sysctl -w net.ipv4.ip_forward=1
为了让重启后依然生效，编辑 sysctl 配置文件：

Bash
sudo nano /etc/sysctl.conf
找到或取消注释下面这一行：

Plaintext
net.ipv4.ip_forward=1
保存并退出，然后应用配置：

Bash
sudo sysctl -p




这份指导专门针对你当前这套系统的**防火墙（`iptables`）配置**进行深度拆解和说明。

它涵盖了从**底层原理解析**、**规则设计逻辑**，到**脚本建立**、**生效验证**及**故障排查**的全过程，便于你后续维护、修改或在其他服务器上复刻。

---

## 一、 防火墙设计目标与策略逻辑

本次防火墙配置的核心需求是**最小权限原则（Security by Default）**：

1. **入口安全**：仅开放 OpenVPN 的 IPv6 服务端口（1194）。
2. **精准放行**：VPN 客户端（网段 `10.8.253.0/24`）通过虚拟网卡（`tun+`）连入后，**只能访问特定的内网网段 `192.168.5.0/24**`。
3. **默认拒绝**：将 `FORWARD`（转发）链的默认策略设置为 `DROP`，彻底切断客户端通过 VPN 服务器跳板访问互联网或其他未经授权网段（如 `192.168.1.0/24`）的路径。
4. **源地址伪装（SNAT/MASQUERADE）**：内网设备（`192.168.5.x`）通常没有通往 `10.8.253.0/24` 的回程路由。数据包从服务器的真实物理网卡（`eno1`）发往内网时，必须将其源 IP 伪装成服务器自身的内网 IP。

---

## 二、 关键组件与变量说明

在配置前，请确保确认以下变量与你服务器的实际环境一致：

| 变量/概念 | 当前配置值 | 含义说明 |
| --- | --- | --- |
| **物理网卡** | `eno1` | 服务器连接内网 `192.168.5.0/24` 的真实网络接口 |
| **虚拟网卡** | `tun+` | OpenVPN 创建的虚拟隧道网卡（`+` 为通配符，代表 `tun0`、`tun1` 等） |
| **VPN 客户端网段** | `10.8.253.0/24` | OpenVPN 分配给连接用户的虚拟 IP 地址池 |
| **目标内网网段** | `192.168.5.0/24` | 允许客户端访问的目标内网 |
| **OpenVPN 端口** | `1194` (UDP6) | 客户端建立隧道使用的通信端口 |

---

## 三、 脚本编写与自动化集成

为了避免手动输入命令在重启后失效，或者由于命令重复执行导致 `iptables` 规则堆叠，我们将策略写入 OpenVPN 的生命周期脚本（`up.sh` / `down.sh`）中。

### 1. 编写启动脚本 (`up.sh`)

当 OpenVPN 服务启动并创建网络接口时，会自动调用该脚本：

```bash
sudo nano /etc/openvpn/server/up.sh

```

将以下完整代码粘贴进去：

```bash
#!/bin/bash
# ==============================================================================
# OpenVPN 启动防火墙策略脚本
# ==============================================================================

# 1. 开启 Linux 内核 IPv4 数据包转发功能
sysctl -w net.ipv4.ip_forward=1 > /dev/null

# 2. 安全核心：将 FORWARD 链的默认策略设为 DROP (拦截所有未显式允许的转发流量)
iptables -P FORWARD DROP

# 3. 防重复保护：清理可能残存的同类规则，避免反复重启服务导致规则堆叠
iptables -D FORWARD -i tun+ -o eno1 -d 192.168.5.0/24 -j ACCEPT 2>/dev/null
iptables -D FORWARD -i eno1 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null
iptables -t nat -D POSTROUTING -s 10.8.253.0/24 -d 192.168.5.0/24 -o eno1 -j MASQUERADE 2>/dev/null

# 4. Filter 表 (FORWARD 链)：放行客户端到内网 192.168.5.0/24 的单向请求
iptables -A FORWARD -i tun+ -o eno1 -d 192.168.5.0/24 -j ACCEPT

# 5. Filter 表 (FORWARD 链)：放行内网对已建立连接（RELATED, ESTABLISHED）的回包
iptables -A FORWARD -i eno1 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT

# 6. NAT 表 (POSTROUTING 链)：仅对目的地为 192.168.5.0/24 的流量实施地址伪装
iptables -t nat -A POSTROUTING -s 10.8.253.0/24 -d 192.168.5.0/24 -o eno1 -j MASQUERADE

```

### 2. 编写停止脚本 (`down.sh`)

当 OpenVPN 服务停止时，清空相关规则并恢复默认策略：

```bash
sudo nano /etc/openvpn/server/down.sh

```

将以下完整代码粘贴进去：

```bash
#!/bin/bash
# ==============================================================================
# OpenVPN 停止防火墙清理脚本
# ==============================================================================

# 1. 删除具体的转发放行规则和 NAT 伪装规则
iptables -D FORWARD -i tun+ -o eno1 -d 192.168.5.0/24 -j ACCEPT 2>/dev/null
iptables -D FORWARD -i eno1 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null
iptables -t nat -D POSTROUTING -s 10.8.253.0/24 -d 192.168.5.0/24 -o eno1 -j MASQUERADE 2>/dev/null

# 2. 恢复 FORWARD 链的默认策略为 ACCEPT (避免影响服务器其他非 VPN 的网络需求)
iptables -P FORWARD ACCEPT

```

### 3. 授予可执行权限

脚本创建完成后，必须赋予其执行权限，否则 OpenVPN 无法调用：

```bash
sudo chmod +x /etc/openvpn/server/up.sh
sudo chmod +x /etc/openvpn/server/down.sh

```

---

## 四、 OpenVPN 服务配置文件关联

编辑 OpenVPN 的主配置文件，确保其能够在启动/停止时自动触发上述脚本：

```bash
sudo nano /etc/openvpn/server/server.conf

```

在文件末尾加入以下三行：

```text
# 允许 OpenVPN 执行外部自定义 Shell 脚本（级别 2 代表允许调用可执行文件）
script-security 2

# 指定网卡 UP 和 DOWN 时的脚本路径
up /etc/openvpn/server/up.sh
down /etc/openvpn/server/down.sh

```

修改完成后，重启 OpenVPN 服务使配置生效：

```bash
sudo systemctl restart openvpn-server@server.service

```

---

## 五、 防火墙规则验证与诊断指南

每次修改防火墙配置后，建议使用以下命令进行**三步检查**。

### 步骤 1：检查 Filter 表的 FORWARD 链

```bash
sudo iptables -t filter -L FORWARD -n -v

```

**标准输出样式：**

```text
Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source       destination
    0     0 ACCEPT     all  --  tun+   eno1    0.0.0.0/0    192.168.5.0/24
    0     0 ACCEPT     all  --  eno1   tun+    0.0.0.0/0    0.0.0.0/0        state RELATED,ESTABLISHED

```

* **核对重点**：
* 第一行首部的 `policy DROP`：代表非明确放行的流量一律拦截。
* 第一条规则的 `destination`：必须精确显示为 `192.168.5.0/24`。



---

### 步骤 2：检查 NAT 表的 POSTROUTING 链

```bash
sudo iptables -t nat -L POSTROUTING -n -v

```

**标准输出样式：**

```text
Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source       destination
    0     0 ACCEPT     all  --  *      eno1    10.8.253.0/24 192.168.5.0/24

```

* **核对重点**：
* `out`: 必须为你连接内网的网卡 `eno1`。
* `source`: 为 VPN 客户端网段 `10.8.253.0/24`。
* `destination`: 目标必须为 `192.168.5.0/24`。



---

### 步骤 3：网络连通性实测

在连上 OpenVPN 的客户端（如手机或电脑）上打开终端进行测试：

1. **放行网段测试（预期成功）**：
```bash
ping 192.168.5.1

```


*说明：应当正常收到回包，网页服务（如 `[http://192.168.5.](http://192.168.5.)X`）可正常打开。*
2. **拦截网段测试（预期失败/超时）**：
```bash
# 测试访问外网 IP
ping 8.8.8.8

# 测试访问内网其他网段（假设存在 192.168.1.0/24）
ping 192.168.1.1

```


*说明：由于 `policy DROP` 和目的地限制，这些数据包会被服务器静默丢弃，结果均为超时，这代表防火墙成功挡住了跨网段访问。*

---

## 六、 常见问题与排错 (Troubleshooting)

### Q1: 客户端连接成功，但 Ping 不通 `192.168.5.x` 内网设备？

* **检查 1**：确认内核 IP 转发是否开启：`cat /proc/sys/net/ipv4/ip_forward`，输出应为 `1`。
* **检查 2**：确认网卡名称是否匹配。使用 `ip a` 命令查看连接 `192.168.5.x` 的网卡名称，如果不是 `eno1`（例如是 `eth0` 或 `enp3s0`），需要同步修改 `up.sh` 和 `down.sh` 里的网卡名。
* **检查 3**：内网目标设备自身是否开启了本地防火墙（如 Windows Defender 防火墙或 Linux `ufw`），可能会默认丢弃来自跨网段的 Ping 流量。

### Q2: 重启服务器后规则丢失了怎么办？

由于我们将规则写在了 OpenVPN 的 `up.sh` 脚本中，**只要 OpenVPN 服务随系统开机自启**（`sudo systemctl enable openvpn-server@server.service`），系统在启动 OpenVPN 时就会自动把防火墙规则注入进去，无需依赖 `iptables-persistent` 工具。