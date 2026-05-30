# 🌐 实验10：使用 Wireshark 捕获 Google 流量

> **难度**：⭐⭐⭐ 进阶级  
> **预计时间**：60分钟  
> **工具**：Wireshark、浏览器

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 Google 服务使用的网络协议（DNS over HTTPS、TLS、QUIC）
- ✅ 掌握如何过滤和识别 Google 流量
- ✅ 学会分析 TLS 握手过程
- ✅ 了解 HTTPS 加密原理和局限性
- ✅ 掌握使用 Wireshark 解密 HTTPS 流量的方法（条件受限）
- ✅ 理解现代网络协议的隐私保护机制

---

## 📖 背景知识

### Google 服务使用的协议

Google 是全球最大的互联网公司之一，其服务使用多种现代网络协议：

| 协议 | 端口 | 说明 |
|------|------|------|
| **HTTPS (HTTP over TLS)** | 443 | 加密的网页传输 |
| **DNS over HTTPS (DoH)** | 443 | 加密的 DNS 查询 |
| **DNS over TLS (DoT)** | 853 | 加密的 DNS 查询（另一种方式） |
| **QUIC** | 443 (UDP) | Google 开发的快速 UDP 互联网连接协议 |
| **HTTP/2** | 443 | 多路复用 HTTP 协议 |
| **HTTP/3** | 443 (UDP) | 基于 QUIC 的新一代 HTTP 协议 |
| **gRPC** | 443 | Google 开发的高性能 RPC 框架 |

### TLS（Transport Layer Security）握手

TLS 是 HTTPS 的基础，用于在客户端和服务器之间建立安全连接。

**TLS 1.2 握手过程：**

```
客户端                    服务器
  |   -------- Client Hello -------->  |
  |     (包含支持的加密套件、随机数)      |
  |   <------- Server Hello ---------  |
  |     (选择加密套件、证书、随机数)      |
  |   <------- Certificate ---------  |
  |     (服务器证书，包含公钥)           |
  |   <------- Server Hello Done ----  |
  |   -------- Client Key Exchange ->  |
  |     (使用服务器公钥加密的预主密钥)     |
  |   -------- Change Cipher Spec -->  |
  |   -------- Finished ------------->  |
  |   <------- Change Cipher Spec ---  |
  |   <------- Finished -------------  |
  |   ====== 加密通信开始 ==========    |
```

**TLS 1.3 握手过程（简化，只需 1 RTT）：**

```
客户端                    服务器
  |   -------- Client Hello -------->  |
  |     (包含支持的加密套件、密钥共享)    |
  |   <------- Server Hello ---------  |
  |     (证书、密钥共享、Finished)       |
  |   -------- Finished ------------->  |
  |   ====== 加密通信开始 ==========    |
```

### HTTPS 流量能否被解密？

**答案：取决于是否拥有密钥。**

| 情况 | 能否解密 | 说明 |
|------|---------|------|
| **拥有服务器私钥** | ✅ 可以 | 使用 Wireshark + 服务器 RSA 私钥 |
| **使用 Diffie-Hellman 密钥交换** | ❌ 不可以 | 即使有私钥也无法解密（前向安全性） |
| **客户端支持 SSLKEYLOGFILE** | ✅ 可以 | 浏览器导出会话密钥（仅限测试） |
| **中间人代理（MITM）** | ✅ 可以 | 但需要客户端信任代理的证书 |

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己访问的 Google 服务**进行流量分析（即你自己的流量）
> - 不得捕获、分析他人的 Google 流量
> - 不得尝试解密不属于自己的 HTTPS 流量
> - 不得利用本实验技术进行窃听、监控等恶意行为
> - 建议使用虚拟机或测试环境进行实验

---

## 🔧 环境要求

### 系统要求

```bash
# Windows 用户
# 1. 安装 Wireshark：https://www.wireshark.org/download.html
# 2. 安装 Chrome 或 Firefox 浏览器
# 3. 确保可以访问 Google 服务（可能需要网络环境支持）

# Linux (Kali/Ubuntu) 用户
sudo apt update
sudo apt install -y wireshark google-chrome-stable

# 或者使用 Firefox
sudo apt install -y firefox-esr

# 启动 Wireshark（需要 root 权限或加入 wireshark 组）
sudo wireshark &
```

### 准备实验环境

**方案1：使用物理机（推荐）**

```bash
# 1. 以管理员权限启动 Wireshark
# 2. 选择正在使用的网络接口（如 Wi-Fi 或 Ethernet）
# 3. 开始捕获
# 4. 在浏览器中访问 Google 服务
# 5. 停止捕获并分析
```

**方案2：使用虚拟机**

```bash
# 在虚拟机中：
# 1. 配置虚拟机网络为 "桥接模式" 或 "NAT 模式"
# 2. 启动 Kali Linux 或 Ubuntu
# 3. 安装 Wireshark 和浏览器
# 4. 按照方案1的步骤操作
```

---

## 👨‍💻 实验步骤

### Step 1：捕获 Google DNS 查询流量

**注意：** 现代浏览器和操作系统可能使用 **DNS over HTTPS (DoH)** 或 **DNS over TLS (DoT)**，传统的 DNS（端口 53）查询可能不可见。

```bash
# 在终端中使用传统 DNS 查询（生成传统 DNS 流量）
nslookup www.google.com 8.8.8.8

# 或者使用 dig
dig @8.8.8.8 www.google.com
```

**在 Wireshark 中过滤 DNS 流量：**

```
# 显示过滤器
dns

# 过滤 Google 相关的 DNS 查询
dns.qry.name contains "google"
dns.qry.name contains "google.com"

# 过滤 Google DNS 服务器的响应
ip.src == 8.8.8.8 && dns
ip.src == 8.8.4.4 && dns
```

**分析 Google DNS 查询（传统方式）：**

1. 找到一个 `Standard query A www.google.com` 的包
2. 展开 **Domain Name System (query)**
3. 你会看到：
   ```
   Transaction ID: 0x3c4d
   Flags: 0x0100 (Standard query)
   Queries:
       Name: www.google.com
       Type: A (Host Address)
       Class: IN (Internet)
   ```

4. 找到对应的 `Standard query response` 包
5. 展开 **Domain Name System (response)**
6. 你会看到：
   ```
   Answers:
       Name: www.google.com
       Type: CNAME
       Name: www.l.google.com
       ...
       Name: l.google.com
       Type: A
       Address: 142.250.80.100
   ```

**捕获 DNS over HTTPS (DoH) 流量：**

DoH 将 DNS 查询封装在 HTTPS 中，端口为 443，无法直接识别为 DNS。

```
# 在 Wireshark 中过滤 DoH 流量（实际上看起来像普通 HTTPS 流量）
tls && ip.dst == 8.8.8.8
tls && ip.dst == 1.1.1.1

# 查看 TLS 握手过程中的 SNI（Server Name Indication）
tls.handshake.extensions_server_name == "dns.google"
tls.handshake.extensions_server_name == "cloudflare-dns.com"
```

**分析 TLS 握手（识别 DoH）：**

1. 找到一个 `Client Hello` 包
2. 展开 **Transport Layer Security → Handshake Protocol → Client Hello**
3. 找到 **Server Name Indication (SNI)** 扩展
4. 你会看到：
   ```
   Extension: server_name
       Server Name Indication extension
           Server Name: dns.google
   ```

**预期输出示例（传统 DNS）：**

```
No.  Time     Source        Destination   Protocol  Info
1    0.000    192.168.1.100  8.8.8.8    DNS       Standard query A www.google.com
2    0.123    8.8.8.8        192.168.1.100   DNS   Standard query response CNAME www.l.google.com A 142.250.80.100
```

---

### Step 2：捕获 Google HTTPS 流量

**在浏览器中访问 Google 服务：**

1. 确保 Wireshark 正在捕获
2. 打开 Chrome 或 Firefox
3. 访问 `https://www.google.com`
4. 访问 `https://mail.google.com`（如果需要）
5. 访问 `https://drive.google.com`（如果需要）

**在 Wireshark 中过滤 Google HTTPS 流量：**

```
# 过滤所有 TLS 流量
tls

# 过滤 Google IP 地址的 TLS 流量
ip.dst == 142.250.0.0/16 && tls
ip.dst == 172.217.0.0/16 && tls
ip.dst == 216.58.0.0/16 && tls

# 根据 SNI 过滤（推荐）
tls.handshake.extensions_server_name contains "google"
tls.handshake.extensions_server_name == "www.google.com"

# 过滤 Google 的特定服务
tls.handshake.extensions_server_name contains "mail.google"
tls.handshake.extensions_server_name contains "drive.google"
```

**识别 Google IP 地址范围：**

Google 拥有大量 IP 地址，常见范围包括：

```bash
# 使用 whois 查询 Google IP 范围
whois 142.250.80.100

# 使用 nslookup 查询 Google 域名
nslookup www.google.com

# Google 常用 IP 范围（可能会变化）：
# 142.250.0.0/16
# 172.217.0.0/16
# 216.58.0.0/16
# 74.125.0.0/16
```

---

### Step 3：分析 TLS 握手过程

**目标：** 深入理解 TLS 握手，理解 HTTPS 的安全性。

**在 Wireshark 中找到 TLS 握手：**

1. 应用显示过滤器：`tls.handshake.type == 1`（Client Hello）
2. 你会看到一个或多个 `Client Hello` 包

**分析 Client Hello（客户端问候）：**

1. 点击一个 `Client Hello` 包
2. 在 Packet Details 中展开 **Transport Layer Security**
3. 展开 **Handshake Protocol: Client Hello**
4. 你会看到：
   ```
   Handshake Type: Client Hello (1)
   Random: [32 字节随机数]
   Session ID: [为空或上次会话 ID]
   Cipher Suites: [客户端支持的加密套件列表]
       Cipher Suite: TLS_AES_128_GCM_SHA256 (0x1301)   [TLS 1.3]
       Cipher Suite: TLS_AES_256_GCM_SHA384 (0x1302)   [TLS 1.3]
       Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f) [TLS 1.2]
       ...
   Extensions:
       Extension: server_name (SNI)
           Server Name: www.google.com
       Extension: supported_groups (椭圆曲线)
       Extension: signature_algorithms
       ...
   ```

**分析 Server Hello（服务器问候）：**

1. 找到对应的 `Server Hello` 包（通常是下一个包）
2. 展开 **Handshake Protocol: Server Hello**
3. 你会看到：
   ```
   Handshake Type: Server Hello (2)
   Random: [32 字节随机数]
   Session ID: [新的会话 ID]
   Cipher Suite: TLS_AES_128_GCM_SHA256 (0x1301)   [服务器选择的加密套件]
   Extensions:
       Extension: server_name (SNI)
       ...
   ```

**分析 Certificate（证书）：**

1. 找到 `Certificate` 包
2. 展开 **Handshake Protocol: Certificate**
3. 你会看到：
   ```
   Certificates:
       Certificate: [Google 的证书]
           Subject: CN=www.google.com
           Issuer: CN=Google Trust Services RSA Certificate Authority
           Validity: 2026-01-01 to 2026-12-31
           Public Key: RSA (2048 bits)
           ...
   ```

**查看证书详情：**

- 右键点击 `Certificate` 包 → **Protocol Preferences → Open TLS in New Window**
- 或者展开证书后，查看 **Subject**、**Issuer**、**Validity** 等字段

**预期输出示例（Client Hello）：**

```
Frame 10: 571 bytes on wire
Internet Protocol Version 4, Src: 192.168.1.100, Dst: 142.250.80.100
Transmission Control Protocol, Src Port: 54321, Dst Port: 443
Transport Layer Security
    TLSv1.3 Record Layer: Handshake Protocol: Client Hello
        Handshake Type: Client Hello (1)
        Random: 3a4b5c6d7e8f...
        Session ID: (empty)
        Cipher Suites (17 suites)
            Cipher Suite: TLS_AES_128_GCM_SHA256
            Cipher Suite: TLS_AES_256_GCM_SHA384
            Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
            ...
        Extensions:
            server_name: www.google.com
            supported_groups: P-256, P-384, ...
            ...
```

---

### Step 4：尝试解密 HTTPS 流量（条件受限）

**重要说明：** 解密 HTTPS 流量需要特定条件，大多数情况下**无法解密**。

#### 方法1：使用服务器私钥（仅适用于 RSA 密钥交换，已过时）

**原理：** 如果 TLS 连接使用 RSA 密钥交换（不是 Diffie-Hellman），并且有服务器的私钥，可以解密。

**步骤：**

1. 获取服务器私钥（需要服务器管理员权限）
2. 在 Wireshark 中配置：
   ```
   Edit → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename
   ```
3. 或者配置 RSA 私钥：
   ```
   Edit → Preferences → Protocols → TLS → RSA keys list → Edit
   输入：IP、Port、Protocol、Key File、Password
   ```

**问题：** 现代 Google 服务使用 **ECDHE**（椭圆曲线 Diffie-Hellman），即使有私钥也无法解密（前向安全性）。

---

#### 方法2：使用 SSLKEYLOGFILE（推荐，适用于测试环境）

**原理：** 浏览器可以将 TLS 会话密钥导出到文件，Wireshark 读取该文件后可以解密。

**步骤（Chrome/Chromium）：**

```bash
# Windows (PowerShell)
$env:SSLKEYLOGFILE = "C:\temp\sslkeylog.txt"
Start-Process chrome

# Linux/macOS (bash)
export SSLKEYLOGFILE=/tmp/sslkeylog.txt
google-chrome &

# Firefox 也支持（在 about:config 中设置）
# security.tls.keylog_append = true
```

**在 Wireshark 中配置：**

1. 打开 Wireshark
2. 进入 `Edit → Preferences → Protocols → TLS`
3. 设置 **(Pre)-Master-Secret log filename** 为 `C:\temp\sslkeylog.txt`（或你的路径）
4. 点击 **OK**
5. 重新捕获流量（确保浏览器是在设置环境变量**之后**启动的）

**验证解密是否成功：**

```
# 过滤 HTTP/2 流量（解密后应该能看到）
http2

# 或者查看 TLS 包的 Info 列，应该显示 "Application Data" 而不是 "[Decrypted TLS]"
```

**预期输出示例（解密后）：**

```
No.  Time     Source          Destination     Protocol  Info
1    0.000    192.168.1.100  142.250.80.100  TLSv1.3   Client Hello
2    0.123    142.250.80.100 192.168.1.100   TLSv1.3   Server Hello, Certificate
3    0.234    192.168.1.100  142.250.80.100  HTTP2     GET / HTTP/2.0
4    0.345    142.250.80.100 192.168.1.100   HTTP2     HTTP/2 200 OK
```

**限制：**

- 只能解密**你自己浏览器**的流量（因为你需要设置环境变量）
- 不能解密其他人的流量
- 不能解密不支持 SSLKEYLOGFILE 的应用程序

---

#### 方法3：使用中间人代理（MITM）

**原理：** 使用代理工具（如 Burp Suite、mitmproxy）作为中间人，解密并重新加密流量。

**步骤（使用 mitmproxy）：**

```bash
# 安装 mitmproxy
pip install mitmproxy

# 启动 mitmproxy
mitmproxy --mode reverse:https://www.google.com --listen-port 8080

# 在浏览器中配置代理：127.0.0.1:8080
# 访问 http://mitm.it 下载并安装 mitmproxy 的 CA 证书
# 浏览器现在会信任 mitmproxy 的证书
```

**问题：**

- Google 服务使用 **HSTS（HTTP Strict Transport Security）**，可能会拒绝连接
- 如果证书验证失败，浏览器会报错
- 仅适用于测试环境，不适用于生产环境

---

### Step 5：实战 - 捕获并分析 Google 搜索请求

**目标：** 理解即使使用 HTTPS，某些信息仍然可能被泄露。

**步骤：**

1. 确保 Wireshark 正在捕获
2. 打开浏览器
3. 访问 `https://www.google.com`
4. 在搜索框中输入 `cybersecurity lab`
5. 点击搜索

**在 Wireshark 中分析：**

```
# 过滤 Google 搜索流量
tls.handshake.extensions_server_name == "www.google.com"

# 查看 TLS 握手后的 HTTP/2 请求（如果解密了）
http2 && ip.dst == 142.250.80.100

# 如果没有解密，仍然可以看到：
# - 目标 IP 地址
# - 目标域名（通过 SNI）
# - 流量大小
# - 时间戳
```

**泄露的信息（即使使用 HTTPS）：**

| 信息 | 是否泄露 | 说明 |
|------|---------|------|
| **目标服务器 IP** | ✅ 是 | 网络层必须知道 |
| **目标域名（SNI）** | ✅ 是 | TLS 握手中明文传输（TLS 1.3 仍包含 SNI） |
| **查询参数（URL路径）** | ❌ 否 | 在 HTTPS 请求中加密 |
| **搜索关键词** | ❌ 否 | 在 HTTPS 请求中加密 |
| **响应内容** | ❌ 否 | 在 HTTPS 响应中加密 |
| **流量大小** | ✅ 是 | 攻击者可以推断内容类型 |
| **流量模式** | ✅ 是 | 攻击者可以推断用户活动 |

**结论：** HTTPS 保护了内容，但没有完全隐藏元数据。

---

### Step 6：分析 QUIC 协议（Google 创新）

**QUIC（Quick UDP Internet Connections）** 是 Google 开发的基于 UDP 的协议，用于加速网页加载。

**识别 QUIC 流量：**

```
# 过滤 QUIC 流量
udp && port 443

# 过滤 Google QUIC 流量
udp && port 443 && ip.dst == 142.250.0.0/16

# QUIC 的早期版本（已过时）
quic

# 现代 QUIC (HTTP/3)
http3
```

**QUIC 的优势：**

- 减少连接建立延迟（0-RTT 或 1-RTT）
- 避免队头阻塞（Head-of-Line Blocking）
- 内置加密（基于 TLS 1.3）
- 连接迁移（即使切换网络也保持连接）

**在 Wireshark 中查看 QUIC 包：**

1. 访问支持 QUIC 的 Google 服务（如 `https://www.google.com`）
2. 在 Wireshark 中过滤 `udp && port 443`
3. 你会看到 **QUIC** 或 **HTTP3** 协议的数据包

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] Google 服务使用哪些协议？
- [ ] TLS 握手的过程是怎样的？
- [ ] 什么是 SNI？它在 TLS 握手中起什么作用？
- [ ] 为什么现代 HTTPS 流量难以解密？
- [ ] 如何使用 SSLKEYLOGFILE 解密 HTTPS 流量？
- [ ] 即使使用 HTTPS，仍然可能泄露哪些信息？
- [ ] 什么是 QUIC 协议？它有什么优势？

---

## 💡 进阶挑战

### 挑战1：识别 Google 服务

**任务：** 捕获流量并识别以下 Google 服务：

- Google 搜索（`www.google.com`）
- Google 邮箱（`mail.google.com`）
- Google 云端硬盘（`drive.google.com`）
- Google 地图（`maps.google.com`）
- YouTube（`www.youtube.com`）

**提示：** 使用 `tls.handshake.extensions_server_name` 过滤器。

---

### 挑战2：分析 HTTP/2 协议

**任务：** 解密 HTTPS 流量后，分析 HTTP/2 协议的特性：

- 多路复用（Multiplexing）
- 服务器推送（Server Push）
- 头部压缩（HPACK）

**过滤器：**

```
http2
http2.headers
http2.data
```

---

### 挑战3：比较 TLS 1.2 和 TLS 1.3

**任务：** 捕获 TLS 1.2 和 TLS 1.3 的握手过程，比较两者的差异。

**如何强制使用 TLS 1.2 或 TLS 1.3：**

```bash
# Chrome: 在 chrome://flags 中搜索 "TLS"
# Firefox: 在 about:config 中搜索 "tls"
```

**比较项目：**

| 项目 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| 握手往返次数（RTT） | 2 RTT | 1 RTT (0-RTT 可选) |
| 加密套件数量 | 很多 | 大幅减少 |
| 握手消息加密 | 部分 | 全部（除 Client Hello） |
| 支持 Forward Secrecy | 可选 | 强制 |

---

### 挑战4：分析 Google 的 DNS over HTTPS (DoH)

**任务：** 配置浏览器使用 DoH，并捕获流量。

**配置 Chrome 使用 DoH：**

1. 进入 `chrome://settings/security`
2. 启用 **"使用安全 DNS"**
3. 选择 **"Google (Public DNS)"** 或 **"Cloudflare"**

**捕获并分析：**

```
# 过滤 DoH 流量（看起来像普通 HTTPS 流量）
tls && ip.dst == 8.8.8.8 && tls.handshake.extensions_server_name == "dns.google"

# 查看 HTTP/2 请求（DoH 使用 HTTP/2）
http2 && ip.dst == 8.8.8.8
```

---

## 🛡️ 防御措施（如何保护自己的隐私）

了解流量分析方法后，也要知道如何保护自己的网络隐私：

| 防御措施 | 说明 |
|---------|------|
| **使用 VPN** | 隐藏真实 IP 地址和流量模式 |
| **使用 Tor 浏览器** | 匿名上网，多层加密 |
| **使用 DNS over HTTPS (DoH)** | 防止 DNS 窃听和劫持 |
| **使用 HTTPS Everywhere** | 强制使用 HTTPS 连接 |
| **禁用 SNI 泄露（ECH）** | 使用 TLS Encrypted Client Hello（技术尚未普及） |
| **定期清除 Cookie** | 减少追踪 |
| **使用隐私友好浏览器** | 如 Firefox (严格模式)、Brave |
| **避免公共 Wi-Fi** | 或使用 VPN |

### 示例：在 Firefox 中启用严格隐私模式

```
1. 打开 Firefox
2. 进入 about:preferences#privacy
3. 选择 "严格（Strict）" 隐私保护模式
4. 启用 "DNS over HTTPS"
5. 选择 "最大化安全性" 提供商
```

---

## 📚 参考资源

- [Wireshark 官方文档 - TLS](https://wiki.wireshark.org/TLS)
- [Cloudflare - What is DNS over HTTPS?](https://www.cloudflare.com/learning/dns/dns-over-https/)
- [Google - QUIC 协议](https://www.chromium.org/quic/)
- [Mozilla - TLS 1.3](https://wiki.mozilla.org/Security/Server_Side_TLS#TLS_1.3)
- [SSLKEYLOGFILE 文档](https://wiki.wireshark.org/TLS#using-the-pre-master-secret)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
捕获文件名：
分析的 Google 服务：
TLS 版本：
是否成功解密：
遇到的问题：
解决方案：
重要发现：
```

### 实验检查清单

- [ ] 成功捕获 Google DNS 查询流量（传统 DNS 或 DoH）
- [ ] 成功捕获 Google HTTPS 流量
- [ ] 能够分析 TLS 握手过程（Client Hello、Server Hello、Certificate）
- [ ] 能够识别 SNI 扩展
- [ ] 尝试使用 SSLKEYLOGFILE 解密 HTTPS 流量（如果可能）
- [ ] 理解即使使用 HTTPS 仍然可能泄露的信息
- [ ] 识别 QUIC 协议流量
- [ ] 完成进阶挑战中的至少一项

---

*最后更新：2026-05-30*
