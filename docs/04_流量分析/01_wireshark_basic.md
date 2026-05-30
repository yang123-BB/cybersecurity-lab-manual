# 🔍 实验09：使用 Wireshark 进行网络分析

> **难度**：⭐⭐ 入门级  
> **预计时间**：45分钟  
> **工具**：Wireshark

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 Wireshark 的工作原理和界面布局
- ✅ 掌握 Wireshark 的基本抓包操作
- ✅ 学会使用 Wireshark 分析常见网络协议（HTTP、DNS、TCP）
- ✅ 能够识别和解读网络流量中的数据包
- ✅ 理解网络协议分析在网络安全中的重要性

---

## 📖 背景知识

### 什么是 Wireshark？

**Wireshark** 是全球最流行的**网络协议分析器**，它是一个功能强大的开源工具，用于：

- 📡 **实时捕获网络数据包**
- 🔎 **深入分析网络协议**
- 🐛 **网络故障排查**
- 🔐 **网络安全分析**
- 📊 **网络性能优化**

### Wireshark 的工作原理

```
Wireshark 工作流程：

1. 选择网络接口（网卡）
      ↓
2. 将网卡设置为混杂模式（Promiscuous Mode）
      ↓
3. 捕获经过网卡的所有数据包
      ↓
4. 解析数据包的各层协议（物理层→应用层）
      ↓
5. 显示可读的协议信息和负载数据
```

### 支持的网络协议

Wireshark 支持分析 **1000+ 种协议**，包括：

| 协议层 | 协议示例 | 端口/协议号 | 说明 |
|--------|----------|-------------|------|
| **应用层** | HTTP/HTTPS | 80/443 | 网页传输 |
|  | DNS | 53 | 域名解析 |
|  | FTP | 21 | 文件传输 |
|  | SMTP | 25 | 邮件发送 |
|  | SSH | 22 | 安全远程登录 |
| **传输层** | TCP | - | 可靠传输 |
|  | UDP | - | 不可靠传输 |
| **网络层** | IPv4/IPv6 | - | 网络互连 |
|  | ICMP | - | 网络控制消息 |
| **数据链路层** | Ethernet | - | 以太网帧 |

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或授权的网络**进行流量分析
> - 未经授权捕获他人网络流量可能违反《网络安全法》和《个人信息保护法》
> - 建议在本地虚拟机、Docker 容器或隔离实验网络中练习
> - 不得利用 Wireshark 进行窃听、窃取密码等恶意行为

---

## 🔧 环境要求

### 系统要求

```bash
# Windows 用户
# 下载并安装 Wireshark：https://www.wireshark.org/download.html
# 安装时勾选 "Install WinPcap" 或 "Npcap"

# Linux (Kali/Ubuntu) 用户
# 检查 Wireshark 是否已安装
wireshark --version

# 如未安装，执行：
sudo apt update
sudo apt install -y wireshark

# 允许普通用户捕获数据包（避免每次用 sudo）
sudo usermod -a -G wireshark $USER
# 注销并重新登录后生效

# 启动 Wireshark
wireshark &
```

### 准备实验环境

为了安全练习，建议准备以下环境：

```bash
# 方案1：使用本地回环接口（最安全）
# 在本地机器上访问网页、ping 等操作，Wireshark 捕获 lo 接口

# 方案2：使用隔离的虚拟机网络
# 在 VMware/VirtualBox 中创建仅主机（Host-Only）网络

# 方案3：使用 Docker 容器生成测试流量
docker run -d --name webserver nginx   # 启动测试 Web 服务器
docker run -d --name dnsserver nginx    # 启动测试 DNS 服务器（简化示例）

# 查看容器 IP
docker inspect webserver | grep IPAddress
```

或者在 Kali 虚拟机内直接进行本地流量捕获练习。

---

## 👨‍💻 实验步骤

### Step 1：熟悉 Wireshark 界面

启动 Wireshark 后，你会看到以下主要区域：

```
┌─────────────────────────────────────────────────────────┐
│  菜单栏：File, Edit, View, Go, Capture, Analyze, ...   │
├─────────────────────────────────────────────────────────┤
│  工具栏：开始捕获、停止、重启、过滤器输入等              │
├─────────────────────────────────────────────────────────┤
│  接口列表：选择要捕获的网络接口（eth0, wlan0, lo...）   │
└─────────────────────────────────────────────────────────┘
```

**开始第一个捕获：**

1. 以管理员权限启动 Wireshark
2. 在主界面选择网络接口（例如 `eth0` 或 `Wi-Fi`）
3. 点击 **"Start Capturing Packets"**（蓝色鲨鱼鳍图标）
4. 你会看到数据包开始滚动显示

**Wireshark 主界面布局：**

```
┌─────────────────────────────────────────────────────────┐
│  Packet List Pane（数据包列表）                         │
│  No. | Time | Source | Destination | Protocol | Info    │
│   1   0.000  192.168.1.100 -> 8.8.8.8    DNS      ...  │
│   2   0.123  192.168.1.100 -> 1.1.1.1    TCP      ...  │
├─────────────────────────────────────────────────────────┤
│  Packet Details Pane（数据包详情）                       │
│  ▼ Frame 1: 74 bytes on wire                           │
│    ▼ Ethernet II, Src: ...                             │
│      ▼ Internet Protocol Version 4, Src: ...           │
│        ▼ Transmission Control Protocol, Src Port: ...  │
├─────────────────────────────────────────────────────────┤
│  Packet Bytes Pane（数据包字节）                         │
│  0000  00 11 22 33 44 55 66 77 88 99 aa bb cc dd ee   │
│  0010  ff 00 01 02 03 04 05 06 07 08 09 0a 0b 0c ...  │
└─────────────────────────────────────────────────────────┘
```

**界面区域说明：**

| 区域 | 名称 | 功能 |
|------|------|------|
| 顶部 | Packet List Pane | 显示捕获的所有数据包列表 |
| 中部 | Packet Details Pane | 显示选中数据包的协议层级结构 |
| 底部 | Packet Bytes Pane | 显示数据包的原始十六进制和 ASCII |

---

### Step 2：捕获第一个数据包

```bash
# 在另一个终端/命令行窗口中生成一些网络流量

# 1. Ping 一个网址（生成 ICMP 流量）
ping -c 4 www.baidu.com

# 2. 访问一个网页（生成 HTTP/HTTPS 流量）
curl http://example.com

# 3. DNS 查询（生成 DNS 流量）
nslookup www.google.com

# 4. 访问 FTP 服务器（如果需要测试 FTP）
# ftp ftp.example.com
```

**在 Wireshark 中观察：**

1. 你会看到 `ping` 生成的 **ICMP** 数据包
2. `curl` 生成的 **TCP 三次握手** 和 **HTTP** 请求/响应
3. `nslookup` 生成的 **DNS** 查询和响应

**停止捕获：**

- 点击工具栏的红色方块 **"Stop"** 按钮
- 或者按快捷键 `Ctrl + E`

**保存捕获文件：**

```
File → Save As → 选择保存位置 → 输入文件名（如 lab09_pcap.pcapng）
```

---

### Step 3：分析 HTTP 协议

HTTP（Hypertext Transfer Protocol）是网页传输的基础协议。

**捕获 HTTP 流量：**

```bash
# 确保 Wireshark 正在捕获
# 在浏览器中访问 http://example.com （注意是 http，不是 https）
# 或者使用 curl
curl http://example.com
```

**在 Wireshark 中过滤 HTTP 流量：**

```
# 在过滤器栏输入（然后按 Enter）
http

# 或者更具体的过滤
http.request.method == "GET"
http.request.method == "POST"
http.response.code == 200
```

**分析 HTTP 请求数据包：**

1. 在 Packet List 中找到一个 `GET` 请求
2. 在 Packet Details 中展开 **Hypertext Transfer Protocol**
3. 你会看到：
   ```
   GET / HTTP/1.1\r\n
   Host: example.com\r\n
   User-Agent: curl/7.68.0\r\n
   Accept: */*\r\n
   ```

**分析 HTTP 响应数据包：**

1. 找到对应的响应包（通常是下一个包）
2. 展开 **Hypertext Transfer Protocol**
3. 你会看到：
   ```
   HTTP/1.1 200 OK\r\n
   Content-Type: text/html; charset=UTF-8\r\n
   Content-Length: 1256\r\n
   \r\n
   <html>...</html>
   ```

**查看 HTTP 负载（Payload）：**

- 右键点击 HTTP 数据包 → **Follow → HTTP Stream**
- 会弹出一个新窗口，显示完整的 HTTP 请求和响应内容

**预期输出示例：**

```
Follow HTTP Stream 窗口内容：

=======================================
REQUEST:
=======================================
GET / HTTP/1.1
Host: example.com
User-Agent: curl/7.68.0
Accept: */*

=======================================
RESPONSE:
=======================================
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 1256

<!doctype html>
<html>
<head>
    <title>Example Domain</title>
    ...
</html>
```

---

### Step 4：分析 DNS 协议

DNS（Domain Name System）用于将域名解析为 IP 地址。

**捕获 DNS 流量：**

```bash
# 在终端执行 DNS 查询
nslookup www.google.com
dig www.baidu.com

# 或者在浏览器中访问一个新网址
```

**在 Wireshark 中过滤 DNS 流量：**

```
# 显示过滤器
dns

# 更具体的过滤
dns.qry.name == "www.google.com"   # 查询特定域名
dns.flags.response == 1             # 只显示响应
dns.resp.addr == 8.8.8.8          # 查询返回特定 IP 的响应
```

**分析 DNS 查询数据包：**

1. 找到一个 `Standard query` 包
2. 展开 **Domain Name System (query)**
3. 你会看到：
   ```
   Transaction ID: 0x1a2b
   Flags: 0x0100 (Standard query)
   Queries:
       Name: www.google.com
       Type: A (Host Address)
       Class: IN (Internet)
   ```

**分析 DNS 响应数据包：**

1. 找到对应的 `Standard query response` 包
2. 展开 **Domain Name System (response)**
3. 你会看到：
   ```
   Transaction ID: 0x1a2b
   Flags: 0x8180 (Standard query response, No error)
   Answers:
       Name: www.google.com
       Type: A (Host Address)
       Address: 142.250.80.100
   ```

**常见 DNS 记录类型：**

| 类型 | 说明 | 示例 |
|------|------|------|
| A | IPv4 地址 | www.google.com → 142.250.80.100 |
| AAAA | IPv6 地址 | ipv6.google.com → 2404:6800:4008:c15::65 |
| CNAME | 别名 | www.google.com → www.l.google.com |
| MX | 邮件服务器 | google.com → mail.google.com |
| NS | 域名服务器 | google.com → ns1.google.com |
| TXT | 文本记录 | google.com → "v=spf1 ..." |

---

### Step 5：分析 TCP 协议

TCP（Transmission Control Protocol）是面向连接的可靠传输协议。

**捕获 TCP 流量：**

```bash
# 访问一个网页（会建立 TCP 连接）
curl http://example.com

# 或者使用 telnet 测试 TCP 连接
telnet example.com 80
```

**在 Wireshark 中过滤 TCP 流量：**

```
# 显示过滤器
tcp

# 查看特定端口的 TCP 流量
tcp.port == 80
tcp.port == 443

# 查看特定主机的 TCP 流量
ip.addr == 192.168.1.100 && tcp

# 查看 TCP 三次握手
tcp.flags.syn == 1 && tcp.flags.ack == 0   # SYN
tcp.flags.syn == 1 && tcp.flags.ack == 1   # SYN-ACK
tcp.flags.ack == 1 && tcp.flags.syn == 0   # ACK
```

**分析 TCP 三次握手（Three-Way Handshake）：**

TCP 建立连接需要三次握手：

```
客户端                    服务器
  |   -------- SYN -------->  |
  |   <----- SYN-ACK -------  |
  |   -------- ACK -------->  |
```

在 Wireshark 中找到这三个包：

1. **第一次握手（SYN）**：
   - Source Port: 随机端口（如 54321）
   - Destination Port: 80
   - Flags: `[SYN]`
   - Sequence Number: 0 (相对值)

2. **第二次握手（SYN-ACK）**：
   - Source Port: 80
   - Destination Port: 54321
   - Flags: `[SYN, ACK]`
   - Sequence Number: 0 (相对值)
   - Acknowledgment Number: 1

3. **第三次握手（ACK）**：
   - Source Port: 54321
   - Destination Port: 80
   - Flags: `[ACK]`
   - Sequence Number: 1
   - Acknowledgment Number: 1

**分析 TCP 四次挥手（Four-Way Wavehand）：**

TCP 断开连接需要四次挥手：

```
客户端                    服务器
  |   -------- FIN -------->  |
  |   <-------- ACK --------  |
  |   <-------- FIN --------  |
  |   -------- ACK -------->  |
```

在 Wireshark 中找到这四个包：

1. **第一次挥手（FIN）**：Flags: `[FIN, ACK]`
2. **第二次挥手（ACK）**：Flags: `[ACK]`
3. **第三次挥手（FIN）**：Flags: `[FIN, ACK]`
4. **第四次挥手（ACK）**：Flags: `[ACK]`

**查看 TCP 会话：**

- 右键点击任意 TCP 数据包 → **Follow → TCP Stream**
- 会显示完整的 TCP 会话内容（包括请求和响应）

---

### Step 6：综合练习 - 追踪一个完整的 HTTP 会话

**目标：** 使用 Wireshark 追踪从 DNS 查询到 HTTP 页面获取的完整过程。

```bash
# 在终端执行
curl -v http://example.com
```

**在 Wireshark 中操作：**

1. 应用显示过滤器：`http || dns`
2. 你会看到以下顺序的数据包：
   ```
   1. DNS Query: example.com → IP地址
   2. DNS Response: IP地址
   3. TCP 三次握手（SYN, SYN-ACK, ACK）
   4. HTTP GET 请求
   5. HTTP 200 OK 响应
   6. TCP 四次挥手（FIN, ACK, FIN, ACK）
   ```

3. 展开每个数据包，观察协议字段
4. 使用 **Follow TCP Stream** 查看完整会话

**预期输出示例：**

```
No.  Time     Source          Destination     Protocol  Info
1    0.000    192.168.1.100  8.8.8.8        DNS       Standard query A example.com
2    0.123    8.8.8.8        192.168.1.100   DNS       Standard query response A 93.184.216.34
3    0.124    192.168.1.100  93.184.216.34   TCP       54321 → 80 [SYN]
4    0.234    93.184.216.34  192.168.1.100   TCP       80 → 54321 [SYN, ACK]
5    0.234    192.168.1.100  93.184.216.34   TCP       54321 → 80 [ACK]
6    0.235    192.168.1.100  93.184.216.34   HTTP      GET / HTTP/1.1
7    0.456    93.184.216.34  192.168.1.100   HTTP      HTTP/1.1 200 OK
8    0.567    192.168.1.100  93.184.216.34   TCP       54321 → 80 [FIN, ACK]
9    0.678    93.184.216.34  192.168.1.100   TCP       80 → 54321 [ACK]
10   0.789    93.184.216.34  192.168.1.100   TCP       80 → 54321 [FIN, ACK]
11   0.789    192.168.1.100  93.184.216.34   TCP       54321 → 80 [ACK]
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] Wireshark 的三个主要界面区域是什么？
- [ ] 如何过滤出所有的 HTTP 流量？
- [ ] DNS 查询和响应的区别是什么？
- [ ] TCP 三次握手的过程是怎样的？
- [ ] 如何使用 "Follow TCP Stream" 功能？
- [ ] 你能从捕获的 HTTP 流量中看到密码吗？（如果能，为什么？如果不能，为什么？）

---

## 💡 进阶挑战

### 挑战1：捕获和分析 HTTPS 流量

```bash
# HTTPS 流量是加密的，无法直接看到内容
# 但你可以分析 TLS 握手过程

# 在 Wireshark 中过滤
tls
tls.handshake.type == 1   # Client Hello
```

**问题：** 为什么 HTTPS 流量无法看到明文内容？

**答案：** 因为 HTTPS = HTTP + TLS/SSL 加密，数据在传输前被加密，只有客户端和服务器拥有解密密钥。

---

### 挑战2：统计网络流量

```
Statistics → Capture File Properties  # 查看捕获文件属性
Statistics → Protocol Hierarchy       # 查看协议层级分布
Statistics → Conversations            # 查看会话统计
Statistics → Endpoints                # 查看端点统计
```

**任务：** 使用 Statistics 菜单分析你的捕获文件，找出：
- 使用最多的协议是什么？
- 哪个 IP 地址产生的流量最多？
- 有多少个 TCP 会话？

---

### 挑战3：导出对象

```
File → Export Objects → HTTP   # 导出 HTTP 对象（图片、文件等）
File → Export Objects → SMB   # 导出 SMB 对象
```

**任务：** 访问一个包含图片的网页，然后使用 Wireshark 导出捕获到的图片。

---

### 挑战4：分析 TCP 重传和丢包

```bash
# 模拟网络拥塞（高级用户）
# 使用 tc (traffic control) 限制带宽或增加延迟
sudo tc qdisc add dev eth0 root netem delay 100ms 10ms 30%
sudo tc qdisc add dev eth0 root netem loss 5%

# 然后进行文件下载，观察 Wireshark 中的 TCP 重传
# 过滤器：tcp.analysis.retransmission
```

---

## 🛡️ 防御措施（如何保护网络流量不被窃听）

了解流量分析方法后，也要知道如何保护自己的网络通信：

| 防御措施 | 说明 |
|---------|------|
| **使用 HTTPS** | 加密 Web 流量，防止窃听和中间人攻击 |
| **使用 VPN** | 在公共 Wi-Fi 上加密所有流量 |
| **使用 DNS over HTTPS (DoH)** | 加密 DNS 查询，防止 DNS 劫持 |
| **使用 SSH 代替 Telnet** | Telnet 是明文传输，SSH 是加密的 |
| **使用 SFTP/FTPs 代替 FTP** | FTP 是明文传输，SFTP/FTPs 是加密的 |
| **网络分段** | 将敏感系统隔离在独立的 VLAN 中 |
| **启用端口安全** | 限制交换机端口的 MAC 地址 |
| **使用网络入侵检测系统（NIDS）** | 如 Snort、Suricata，检测异常流量 |

### 示例：在 Nginx 中启用 HTTPS

```nginx
# /etc/nginx/sites-available/default
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

---

## 📚 参考资源

- [Wireshark 官方网站](https://www.wireshark.org/)
- [Wireshark 用户指南](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Wireshark 显示过滤器参考](https://www.wireshark.org/docs/dfref/)
- [Kali Linux Wireshark 教程](https://www.kali.org/tools/wireshark/)
- [Wireshark 实战教程](https://wiki.wireshark.org/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
捕获文件名：
分析的协议：
遇到的问题：
解决方案：
重要发现：
```

### 实验检查清单

- [ ] 成功启动 Wireshark 并捕获数据包
- [ ] 能够使用显示过滤器过滤 HTTP、DNS、TCP 流量
- [ ] 能够分析 HTTP 请求和响应
- [ ] 能够分析 DNS 查询和响应
- [ ] 能够识别 TCP 三次握手
- [ ] 使用 "Follow TCP Stream" 查看完整会话
- [ ] 完成进阶挑战中的至少一项

---

*最后更新：2026-05-30*
