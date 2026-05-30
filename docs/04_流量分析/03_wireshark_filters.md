# 🔎 实验14：在 Wireshark 中过滤流量

> **难度**：⭐⭐ 入门级  
> **预计时间**：45分钟  
> **工具**：Wireshark

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 Wireshark 显示过滤器（Display Filter）和捕获过滤器（Capture Filter）的区别
- ✅ 掌握显示过滤器的语法和常用表达式
- ✅ 掌握捕获过滤器的语法（BPF 格式）
- ✅ 学会过滤特定 IP、端口、协议的流量
- ✅ 能够使用逻辑运算符组合复杂过滤条件
- ✅ 掌握保存和加载过滤器的技巧

---

## 📖 背景知识

### 为什么需要过滤？

在真实的网络环境中，Wireshark 可能捕获到**成千上万**的数据包。如果不使用过滤器，分析特定流量就像"大海捞针"。

**过滤器的作用：**

- 🎯 **精准定位**：快速找到感兴趣的流量
- 🚀 **提高效率**：减少不必要的数据包显示
- 🔐 **保护隐私**：隐藏敏感信息（如密码、Cookie）
- 📊 **聚焦分析**：专注于特定协议或主机

### 显示过滤器 vs 捕获过滤器

Wireshark 有两种过滤器，很多人会混淆：

| 特性 | 显示过滤器（Display Filter） | 捕获过滤器（Capture Filter） |
|------|---------------------------|---------------------------|
| **作用时机** | 捕获**之后**，用于过滤显示 | 捕获**之前**，用于减少捕获的数据量 |
| **语法** | Wireshark 专用语法 | Berkeley Packet Filter (BPF) 语法 |
| **性能影响** | 捕获所有流量，但只显示匹配的 | 只捕获匹配的流量，节省资源 |
| **使用场景** | 分析已捕获的文件 | 实时捕获时减少数据量 |
| **示例** | `ip.addr == 192.168.1.1` | `host 192.168.1.1` |
| **输入框位置** | 主界面顶部（绿色背景） | 捕获选项界面（开始捕获前） |

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或授权的网络**进行流量分析
> - 不得利用过滤器隐藏恶意流量或逃避安全检测
> - 不得过滤并窃取他人的敏感信息（如密码、Cookie）
> - 建议在本地虚拟机或隔离实验网络中练习

---

## 🔧 环境要求

### 系统要求

```bash
# Windows 用户
# 下载并安装 Wireshark：https://www.wireshark.org/download.html

# Linux (Kali/Ubuntu) 用户
sudo apt update
sudo apt install -y wireshark

# 启动 Wireshark
wireshark &
```

### 准备实验环境

**生成测试流量：**

```bash
# 方案1：使用 ping 生成 ICMP 流量
ping -c 10 8.8.8.8

# 方案2：使用 curl 生成 HTTP/HTTPS 流量
curl http://example.com
curl https://www.google.com

# 方案3：使用 nslookup 生成 DNS 流量
nslookup www.baidu.com

# 方案4：使用 telnet 生成 TCP 流量
telnet example.com 80

# 方案5：使用 nc (netcat) 生成自定义流量
echo "Hello" | nc -u 8.8.8.8 53   # UDP 流量
```

**或者：** 直接开始捕获，然后在浏览器中访问几个网站，生成混合流量。

---

## 👨‍💻 实验步骤

### Step 1：理解显示过滤器语法

**显示过滤器语法规则：**

```
<协议>.<字段> <运算符> <值>
```

**示例：**

```
ip.addr == 192.168.1.1         # IP 地址等于 192.168.1.1
tcp.port == 80                  # TCP 端口等于 80
http.request.method == "GET"    # HTTP 请求方法是 GET
dns.qry.name contains "google"  # DNS 查询名包含 "google"
```

**常用运算符：**

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `==` | 等于 | `ip.addr == 192.168.1.1` |
| `!=` | 不等于 | `ip.addr != 192.168.1.1` |
| `>` `>=` | 大于/大于等于 | `tcp.window_size > 1024` |
| `<` `<=` | 小于/小于等于 | `frame.len < 100` |
| `contains` | 包含（字符串） | `http.host contains "google"` |
| `matches` | 正则匹配 | `http.host matches ".*\.com$"` |
| `&&` | 逻辑与 | `ip.src == 192.168.1.1 && tcp.port == 80` |
| `||` | 逻辑或 | `dns || icmp` |
| `!` | 逻辑非 | `!arp` （不包含 ARP 协议） |

---

### Step 2：过滤特定 IP 地址

**目标：** 学会过滤特定主机的流量。

**基本 IP 过滤：**

```
# 显示所有与 192.168.1.100 相关的流量（源或目的）
ip.addr == 192.168.1.100

# 显示源 IP 是 192.168.1.100 的流量
ip.src == 192.168.1.100

# 显示目的 IP 是 192.168.1.100 的流量
ip.dst == 192.168.1.100

# 排除某个 IP 的流量
!(ip.addr == 192.168.1.100)
ip.addr != 192.168.1.100
```

**过滤 IP 范围：**

```
# 使用 CIDR 表示法（推荐）
ip.addr in 192.168.1.0/24

# 使用区间（较慢）
ip.addr >= 192.168.1.1 && ip.addr <= 192.168.1.254
```

**实战练习：**

1. 开始捕获流量
2. 在另一个终端 ping 一个外部 IP（如 `ping -c 5 8.8.8.8`）
3. 在 Wireshark 中应用过滤器：
   ```
   ip.dst == 8.8.8.8
   ```
4. 你应该只看到目的 IP 是 8.8.8.8 的包（ICMP Echo Request）
5. 修改过滤器为：
   ```
   ip.src == 8.8.8.8
   ```
6. 你应该只看到源 IP 是 8.8.8.8 的包（ICMP Echo Reply）

**预期输出示例：**

```
No.  Time     Source       Destination    Protocol  Info
1    0.000    192.168.1.100  8.8.8.8   ICMP      Echo (ping) request
2    0.123    8.8.8.8        192.168.1.100   ICMP  Echo (ping) reply
3    1.000    192.168.1.100  8.8.8.8   ICMP      Echo (ping) request
4    1.124    8.8.8.8        192.168.1.100   ICMP  Echo (ping) reply
```

---

### Step 3：过滤特定端口

**目标：** 学会过滤特定服务的流量。

**基本端口过滤：**

```
# 显示所有 TCP 端口 80 的流量（源或目的）
tcp.port == 80

# 显示源端口是 80 的流量（通常是服务器响应）
tcp.srcport == 80

# 显示目的端口是 80 的流量（通常是客户端请求）
tcp.dstport == 80

# 显示 UDP 端口 53 的流量（DNS）
udp.port == 53

# 多个端口
tcp.port in {80, 443, 22}
```

**常用端口和协议：**

| 端口 | 协议 | 说明 |
|------|------|------|
| 21 | FTP | 文件传输（明文） |
| 22 | SSH | 安全远程登录 |
| 23 | Telnet | 远程登录（明文，不安全） |
| 25 | SMTP | 邮件发送 |
| 53 | DNS | 域名解析（UDP/TCP） |
| 80 | HTTP | 网页传输（明文） |
| 110 | POP3 | 邮件接收 |
| 143 | IMAP | 邮件接收 |
| 443 | HTTPS | 加密网页传输 |
| 3306 | MySQL | MySQL 数据库 |
| 3389 | RDP | 远程桌面 |
| 8080 | HTTP | 替代 HTTP 端口 |

**实战练习：**

1. 在浏览器中访问 `http://example.com`（HTTP，端口 80）
2. 在 Wireshark 中应用过滤器：
   ```
   tcp.port == 80 && http
   ```
3. 你应该看到 HTTP 请求和响应
4. 修改过滤器为：
   ```
   tcp.port == 443 && tls
   ```
5. 在浏览器中访问 `https://www.google.com`，你应该看到 TLS 握手

**预期输出示例：**

```
No.  Time     Source          Destination     Protocol  Info
1    0.000    192.168.1.100  93.184.216.34  TCP       54321 → 80 [SYN]
2    0.123    93.184.216.34  192.168.1.100  TCP       80 → 54321 [SYN, ACK]
3    0.124    192.168.1.100  93.184.216.34  HTTP      GET / HTTP/1.1
4    0.345    93.184.216.34  192.168.1.100  HTTP      HTTP/1.1 200 OK
```

---

### Step 4：过滤特定协议

**目标：** 学会过滤特定协议的流量。

**基本协议过滤：**

```
# 显示所有 HTTP 流量
http

# 显示所有 DNS 流量
dns

# 显示所有 TLS (HTTPS) 流量
tls

# 显示所有 ICMP (ping) 流量
icmp

# 显示所有 TCP 流量
tcp

# 显示所有 UDP 流量
udp

# 显示所有 ARP 流量
arp
```

**组合协议过滤：**

```
# 显示 HTTP 或 DNS 流量
http || dns

# 显示 TCP 但不显示 HTTPS
tcp && !tls

# 显示 ICMP 或 ARP（网络和链路层协议）
icmp || arp
```

**实战练习：**

1. 开始捕获流量
2. 在终端执行：
   ```bash
   ping -c 3 8.8.8.8
   nslookup www.baidu.com
   curl http://example.com
   ```
3. 在 Wireshark 中分别应用以下过滤器，观察结果：
   ```
   icmp
   dns
   http
   tcp && !tls
   ```

**预期输出示例（过滤器：`http || dns`）：**

```
No.  Time     Source          Destination     Protocol  Info
1    0.000    192.168.1.100  8.8.8.8        DNS       Standard query A example.com
2    0.123    8.8.8.8        192.168.1.100   DNS       Standard query response A 93.184.216.34
3    0.124    192.168.1.100  93.184.216.34   TCP       54321 → 80 [SYN]
4    0.345    192.168.1.100  93.184.216.34   HTTP      GET / HTTP/1.1
5    0.567    93.184.216.34  192.168.1.100   HTTP      HTTP/1.1 200 OK
```

---

### Step 5：使用逻辑运算符组合复杂过滤条件

**目标：** 学会构建复杂的过滤表达式。

**逻辑运算符：**

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `&&` 或 `and` | 逻辑与 | `ip.src == 192.168.1.100 && tcp.port == 80` |
| `||` 或 `or` | 逻辑或 | `dns || icmp` |
| `!` 或 `not` | 逻辑非 | `!tls` （不包含 TLS） |
| `()` | 括号（改变优先级） | `(http || dns) && ip.src == 192.168.1.100` |

**复杂过滤示例：**

```
# 显示来自 192.168.1.100 的 HTTP 或 DNS 流量
(ip.src == 192.168.1.100) && (http || dns)

# 显示目标端口是 80 或 443 的 TCP 流量
tcp.port in {80, 443}

# 显示不是来自本地网络的流量（外部流量）
!(ip.src in 192.168.1.0/24)

# 显示包含 "google" 的 DNS 查询
dns.qry.name contains "google"

# 显示失败的 HTTP 响应（4xx 或 5xx）
http.response.code >= 400

# 显示 SYN 包（TCP 三次握手第一次）
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

**实战练习：**

1. 捕获一段时间内的流量（访问多个网站）
2. 应用以下复杂过滤器，观察结果：
   ```
   # 只显示 Google 的 HTTPS 流量
   tls && (ip.dst == 142.250.0.0/16 || ip.src == 142.250.0.0/16)
   
   # 显示所有 DNS 查询（不包括响应）
   dns.flags.response == 0
   
   # 显示所有 TCP 重置包（连接异常）
   tcp.flags.reset == 1
   ```

---

### Step 6：理解捕获过滤器语法（BPF）

**捕获过滤器语法（BPF - Berkeley Packet Filter）：**

```
<协议> <方向> <主机/网络> <端口>
```

**示例：**

```
# 只捕获来自或去往 192.168.1.100 的流量
host 192.168.1.100

# 只捕获去往 192.168.1.100 的流量
dst host 192.168.1.100

# 只捕获来自 192.168.1.100 的流量
src host 192.168.1.100

# 只捕获端口 80 的流量
port 80

# 只捕获 TCP 端口 80 的流量
tcp port 80

# 只捕获 UDP 端口 53 的流量（DNS）
udp port 53

# 只捕获 ICMP 流量
icmp

# 组合条件（与）
host 192.168.1.100 && port 80

# 组合条件（或）
port 80 || port 443

# 排除某个主机
not host 192.168.1.1
```

**如何使用捕获过滤器：**

1. 打开 Wireshark
2. 点击 **"Capture" → "Options"**（或快捷键 `Ctrl + K`）
3. 在 **"Capture Filter"** 输入框中输入过滤器表达式
4. 点击 **"Start"** 开始捕获

**显示过滤器 vs 捕获过滤器对比：**

| 需求 | 显示过滤器 | 捕获过滤器 |
|------|-----------|-----------|
| 过滤 IP 192.168.1.100 | `ip.addr == 192.168.1.100` | `host 192.168.1.100` |
| 过滤端口 80 | `tcp.port == 80` | `port 80` |
| 过滤 TCP 协议 | `tcp` | `tcp` |
| 过滤 DNS | `dns` | `udp port 53` |
| 组合条件（与） | `&&` | `&&` 或 `and` |
| 组合条件（或） | `||` | `||` 或 `or` |
| 否定 | `!` 或 `not` | `not` 或 `!` |

**实战练习：**

1. 打开 Wireshark 的捕获选项
2. 在捕获过滤器中输入：
   ```
   port 80 || port 443
   ```
3. 开始捕获
4. 在浏览器中访问 HTTP 和 HTTPS 网站
5. 停止捕获，你会看到只捕获了端口 80 和 443 的流量

---

### Step 7：保存和加载过滤器

**保存过滤器：**

1. 在显示过滤器输入框中输入过滤器表达式
2. 点击输入框右侧的 **"+"** 按钮
3. 输入过滤器名称（如 "My HTTP Filter"）
4. 点击 **"OK"**
5. 过滤器会保存在 Wireshark 配置文件中

**加载过滤器：**

1. 点击显示过滤器输入框左侧的 **书签图标**
2. 选择已保存的过滤器
3. 过滤器会自动应用到输入框

**导出/导入过滤器：**

```
# 过滤器配置文件位置
# Windows: C:\Users\<用户名>\AppData\Roaming\Wireshark\dfilters
# Linux: ~/.config/wireshark/dfilters
```

**实战练习：**

1. 保存以下过滤器：
   - Name: "HTTP Traffic", Filter: `http`
   - Name: "DNS Traffic", Filter: `dns`
   - Name: "Google HTTPS", Filter: `tls && ip.dst contains "google"`
2. 加载并测试这些过滤器

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] 显示过滤器和捕获过滤器的区别是什么？
- [ ] 如何过滤特定 IP 地址的流量？
- [ ] 如何过滤特定端口的流量？
- [ ] 如何过滤 HTTP 流量？
- [ ] 如何组合多个过滤条件（逻辑与、逻辑或）？
- [ ] 捕获过滤器的语法是什么（BPF）？
- [ ] 如何保存和加载常用的过滤器？

---

## 💡 进阶挑战

### 挑战1：过滤特定 HTTP 请求方法

```
# 过滤 GET 请求
http.request.method == "GET"

# 过滤 POST 请求
http.request.method == "POST"

# 过滤所有 HTTP 请求（不包括响应）
http.request

# 过滤所有 HTTP 响应（不包括请求）
http.response
```

**任务：** 访问一个网页，然后使用 Wireshark 过滤出所有的 GET 和 POST 请求。

---

### 挑战2：过滤特定 DNS 查询

```
# 过滤查询 "www.google.com" 的 DNS 请求
dns.qry.name == "www.google.com"

# 过滤包含 "google" 的 DNS 查询
dns.qry.name contains "google"

# 过滤 DNS 响应（A 记录）
dns.flags.response == 1 && dns.resp.type == 1
```

**任务：** 捕获 DNS 流量，然后过滤出所有对 `*.google.com` 的查询。

---

### 挑战3：过滤 TCP 三次握手和四次挥手

```
# 过滤 SYN 包（第一次握手）
tcp.flags.syn == 1 && tcp.flags.ack == 0

# 过滤 SYN-ACK 包（第二次握手）
tcp.flags.syn == 1 && tcp.flags.ack == 1

# 过滤 ACK 包（第三次握手）
tcp.flags.ack == 1 && tcp.flags.syn == 0

# 过滤 FIN 包（挥手）
tcp.flags.fin == 1

# 过滤 RST 包（连接重置）
tcp.flags.reset == 1
```

**任务：** 访问一个 HTTP 网站，然后使用 Wireshark 过滤出 TCP 三次握手的过程。

---

### 挑战4：使用正则表达式过滤

```
# 过滤主机名以 ".com" 结尾的 HTTP 请求
http.host matches "\\.com$"

# 过滤 URI 包含数字的 HTTP 请求
http.request.uri matches "\\d+"
```

**任务：** 使用正则表达式过滤出所有访问 `.edu` 或 `.gov` 域名的 HTTP 请求。

---

### 挑战5：分析网络攻击流量（高级）

**场景：** 假设你捕获了一段流量，怀疑其中有网络攻击（如端口扫描、DDoS）。

```
# 过滤大量 SYN 包（可能的 SYN Flood 攻击）
tcp.flags.syn == 1 && tcp.flags.ack == 0

# 过滤 ICMP 流量（可能的 ICMP Flood 攻击）
icmp

# 过滤到多个端口的 TCP 连接（可能的端口扫描）
tcp.flags.syn == 1

# 过滤异常大的数据包（可能的碎片攻击）
frame.len > 1500
```

**任务：** 使用 Wireshark 的 **Statistics → Conversations** 功能，找出产生最多流量的 IP 地址。

---

## 🛡️ 防御措施（如何使用过滤器保护隐私和安全）

了解过滤器后，也要知道如何防止自己的流量被分析：

| 防御措施 | 说明 |
|---------|------|
| **使用 VPN** | 加密所有流量，过滤器无法看到真实内容 |
| **使用 Tor** | 多层加密和匿名，难以过滤和追踪 |
| **使用 HTTPS** | 过滤器无法看到 HTTP 请求和响应的内容 |
| **使用 DNS over HTTPS (DoH)** | 过滤器无法看到 DNS 查询的域名 |
| **定期更换 IP** | 使基于 IP 的过滤器失效 |
| **使用非标准端口** | 使基于端口的过滤器失效（但可以被协议识别） |
| **使用网络入侵检测系统（NIDS）** | 如 Snort、Suricata，自动检测异常流量 |

### 示例：使用 Snort 检测端口扫描

```bash
# Snort 规则示例：检测端口扫描
alert tcp any any -> $HOME_NET any (msg:"Possible Port Scan"; flags:S; threshold:type both, track by_src, count 20, seconds 10; sid:1000001;)
```

---

## 📚 参考资源

- [Wireshark 显示过滤器参考](https://www.wireshark.org/docs/dfref/)
- [Wireshark 捕获过滤器参考](https://wiki.wireshark.org/CaptureFilters)
- [Wireshark 过滤器语法教程](https://wiki.wireshark.org/DisplayFilters)
- [BPF (Berkeley Packet Filter) 文档](https://www.tcpdump.org/manpages/pcap-filter.7.html)
- [Wireshark 官方教程](https://www.wireshark.org/docs/wsug_html_chunked/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
捕获文件名：
使用的过滤器：
遇到的问题：
解决方案：
重要发现：
```

### 实验检查清单

- [ ] 理解显示过滤器和捕获过滤器的区别
- [ ] 能够使用显示过滤器过滤特定 IP、端口、协议
- [ ] 能够使用逻辑运算符组合复杂过滤条件
- [ ] 理解捕获过滤器语法（BPF）
- [ ] 能够保存和加载常用过滤器
- [ ] 完成进阶挑战中的至少一项

### 常用过滤器速查表

```
# IP 过滤
ip.addr == 192.168.1.1
ip.src == 192.168.1.1
ip.dst == 192.168.1.1

# 端口过滤
tcp.port == 80
udp.port == 53

# 协议过滤
http
dns
tls
icmp
tcp
udp

# 逻辑组合
&& (与)
|| (或)
!  (非)

# HTTP 过滤
http.request.method == "GET"
http.response.code == 200

# DNS 过滤
dns.qry.name contains "google"
dns.flags.response == 0

# TCP 标志位过滤
tcp.flags.syn == 1
tcp.flags.ack == 1
tcp.flags.reset == 1
```

---

*最后更新：2026-05-30*
