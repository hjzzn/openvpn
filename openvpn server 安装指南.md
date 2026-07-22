这份指南整体非常详尽且逻辑清晰，但部分章节存在**内容重复**、**编排略显零碎**（例如 IPv6 与内网路由交叉说明）的情况。

优化后的版本合并了重复的配置步骤，优化了排版与结构，并增加了**参数检查表**，使其更适合作为一份规范的**运维部署手册**。

---

# 🛡️ Ubuntu 下 OpenVPN 服务端部署与精细化安全配置指南

本文档介绍如何在 Ubuntu 上通过一键脚本快速部署 OpenVPN，并配置为 **纯 IPv6 监听** 及 **基于最小权限原则（仅允许访问特定内网）** 的安全防火墙策略。

---

## 📋 一、 部署前准备与环境参数表

在开始前，请先核对并确认你服务器的实际网络参数（后续脚本将依赖这些参数）：

| 参数项 | 本手册默认值 | 说明 / 检查方法 |
| --- | --- | --- |
| **物理内网网卡** | `eno1` | 连接内网的真实网卡名称（运行 `ip a` 查看） |
| **虚拟 VPN 网卡** | `tun+` | OpenVPN 创建的虚拟网卡（`+` 为通配符，匹配 `tun0`, `tun1`） |
| **VPN 客户端网段** | `10.8.253.0/24` | OpenVPN 分配给客户端的虚拟 IP 池 |
| **允许访问的目标内网** | `192.168.5.0/24` | 仅允许客户端访问的特定内网网段 |
| **OpenVPN 监听端口** | `1194` (UDP6) | 客户端建立 VPN 隧道使用的协议与端口 |

---

## 🚀 二、 基础服务快速部署

### 1. 运行一键安装脚本

使用社区广泛认可的自动化脚本进行基础安装（需 `root` 权限）：

```bash
wget https://raw.githubusercontent.com/Nyr/openvpn-install/master/openvpn-install.sh -O openvpn-install.sh
sudo bash openvpn-install.sh

```

### 2. 交互式参数选择

脚本运行中会提示选择配置，建议如下：

* **Protocol**: 选择 `1` (UDP)
* **Port**: 默认 `1194`
* **DNS**: 选择偏好的公共 DNS（如 Cloudflare / Google）
* **Client Name**: 输入首个客户端名称（如 `client1`）

安装完成后，生成的配置文件放置于 `/root/client1.ovpn`。

> **💡 提示**：后续如需添加/删除客户端或证书，再次运行 `sudo bash openvpn-install.sh` 即可。

---

## ⚙️ 三、 服务端进阶配置 (仅监听 IPv6 + 内网路由)

编辑 OpenVPN 主配置文件：

```bash
sudo nano /etc/openvpn/server/server.conf

```

### 1. 配置纯 IPv6 监听

找到 `proto` 与 `bind` 相关的配置项，修改为：

```conf
proto udp6
local ::
bind ipv6only

```

* `proto udp6`: 使用 IPv6 UDP 协议。
* `local ::`: 绑定到服务器所有的 IPv6 地址。
* `bind ipv6only`: 彻底关闭对 IPv4 端口的响应。

### 2. 推送内网路由

在文件末尾（或路由设置区）添加以下指令，向客户端推送通往内网的路由：

```conf
push "route 192.168.5.0 255.255.255.0"

```

### 3. 关联防火墙生命周期脚本

在 `server.conf` 末尾追加以下配置，以便在服务启动/停止时自动加载/清理防护规则：

```conf
script-security 2
up /etc/openvpn/server/up.sh
down /etc/openvpn/server/down.sh

```

---

## 🔒 四、 基于最小权限的防火墙自动化配置

为实现**最小权限原则（Security by Default）**：

1. 客户端连接后**仅允许访问 `192.168.5.0/24**`。
2. 拒绝客户端访问外网或其他内网网段（如 `192.168.1.0/24`）。
3. 使用 SNAT/MASQUERADE 解决内网回程路由问题。

### 1. 创建启动脚本 (`/etc/openvpn/server/up.sh`)

```bash
sudo nano /etc/openvpn/server/up.sh

```

写入以下内容：

```bash
#!/bin/bash
# ==============================================================================
# OpenVPN 启动防火墙策略脚本
# ==============================================================================

# 1. 开启系统 IPv4 转发
sysctl -w net.ipv4.ip_forward=1 > /dev/null

# 2. 核心安全：默认拦截所有转发流量
iptables -P FORWARD DROP

# 3. 清理旧规则（防止反复重启服务导致规则堆叠）
iptables -D FORWARD -i tun+ -o eno1 -d 192.168.5.0/24 -j ACCEPT 2>/dev/null
iptables -D FORWARD -i eno1 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null
iptables -t nat -D POSTROUTING -s 10.8.253.0/24 -d 192.168.5.0/24 -o eno1 -j MASQUERADE 2>/dev/null

# 4. 放行客户端到目标内网 (192.168.5.0/24) 的单向访问
iptables -A FORWARD -i tun+ -o eno1 -d 192.168.5.0/24 -j ACCEPT

# 5. 放行已建立连接（RELATED, ESTABLISHED）的回包流量
iptables -A FORWARD -i eno1 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT

# 6. 对发往内网 192.168.5.0/24 的流量实施地址伪装 (NAT)
iptables -t nat -A POSTROUTING -s 10.8.253.0/24 -d 192.168.5.0/24 -o eno1 -j MASQUERADE

```

### 2. 创建停止脚本 (`/etc/openvpn/server/down.sh`)

```bash
sudo nano /etc/openvpn/server/down.sh

```

写入以下内容：

```bash
#!/bin/bash
# ==============================================================================
# OpenVPN 停止防火墙清理脚本
# ==============================================================================

# 1. 删除具体的转发放行与 NAT 规则
iptables -D FORWARD -i tun+ -o eno1 -d 192.168.5.0/24 -j ACCEPT 2>/dev/null
iptables -D FORWARD -i eno1 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null
iptables -t nat -D POSTROUTING -s 10.8.253.0/24 -d 192.168.5.0/24 -o eno1 -j MASQUERADE 2>/dev/null

# 2. 恢复默认 FORWARD 策略为 ACCEPT（避免影响非 VPN 业务）
iptables -P FORWARD ACCEPT

```

### 3. 赋予执行权限并持久化系统内核设置

```bash
# 赋予脚本执行权限
sudo chmod +x /etc/openvpn/server/up.sh
sudo chmod +x /etc/openvpn/server/down.sh

# 配置系统级 IP 转发持久化（避免系统重启失效）
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

```

---

## 🌐 五、 云服务器与系统防火墙放行

### 1. 系统防火墙 (UFW)

如开启了 UFW，放行 UDP 1194 端口：

```bash
sudo ufw allow 1194/udp

```

### 2. 云厂商安全组 (Security Group)

若部署在阿里云、腾讯云、AWS 或 GCP 等环境，请在控制台安全组**入方向规则**中添加：

* **协议**：UDP
* **端口**：1194
* **源地址**：`::/0` (或指定客户端 IPv6 前缀)

---

## 📱 六、 客户端 `.ovpn` 配置与服务启动

### 1. 修改客户端 `.ovpn` 文件

下载生成的 `.ovpn` 文件，将其中的 `remote` 行修改为服务器的 **IPv6 地址** 或 **IPv6 域名**：

```conf
# 使用 IPv6 地址（必须包含在方括号内）
remote [2001:db8::1] 1194 udp6

# 或使用配置了 AAAA 记录的域名
remote vpn.yourdomain.com 1194 udp6

```

### 2. 重启 OpenVPN 服务

```bash
sudo systemctl restart openvpn-server@server.service

```

确认服务是否处于活动状态：

```bash
sudo systemctl status openvpn-server@server.service

```

---

## 🔍 七、 验证与排错指南

### 步骤 1：验证 IPv6 端口监听

```bash
sudo ss -tulpn | grep 1194

```

* **期望输出**：包含 `[::]:1194`，且**不包含** `0.0.0.0:1194`。

---

### 步骤 2：检查 iptables 策略生效状态

* **检查 FORWARD 链**：
```bash
sudo iptables -t filter -L FORWARD -n -v

```


* 确认首行显示 `Chain FORWARD (policy DROP...)`
* 确认存在前往 `192.168.5.0/24` 的 `ACCEPT` 规则。


* **检查 NAT 表**：
```bash
sudo iptables -t nat -L POSTROUTING -n -v

```


* 确认源 `10.8.253.0/24` 到目的 `192.168.5.0/24` 的出站网卡为 `eno1` 且目标为 `MASQUERADE`。



---

### 步骤 3：连通性实测（在已连接 VPN 的客户端测试）

1. **内网放行网段测试（应成功）**：
```bash
ping 192.168.5.1

```


2. **阻断网络/外网测试（应超时/失败）**：
```bash
ping 8.8.8.8          # 测试外网
ping 192.168.1.1      # 测试未授权内网

```



---

### ❓ 常见问题排查 (Troubleshooting)

* **Q: 客户端连接成功，但 Ping 不通 `192.168.5.x`？**
1. 检查物理网卡名：运行 `ip a`，确认服务器连接 `192.168.5.x` 的网卡名称是否真的为 `eno1`（若为 `eth0`，需替换 `up.sh` 和 `down.sh` 中的网卡名）。
2. 检查内网目标设备：确认目标机器（如 `192.168.5.x`）自身的防火墙未拦截跨网段流量。


* **Q: 服务器重启后防火墙规则丢失？**
* 由于规则绑定在 OpenVPN 的 `up.sh` 脚本中，只要服务设置了开机自启（`sudo systemctl enable openvpn-server@server.service`），服务启动时就会自动注入规则，无需额外安装 `iptables-persistent`。