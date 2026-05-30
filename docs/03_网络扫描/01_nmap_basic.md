# 🔍 实验03：使用 Nmap 进行网络扫描

> **难度**：⭐⭐ 入门级  
> **预计时间**：45分钟  
> **工具**：Nmap (Network Mapper)

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 Nmap 的工作原理和应用场景
- ✅ 掌握 Nmap 的基本命令语法
- ✅ 使用不同的扫描技术（TCP Connect、SYN Stealth、UDP）
- ✅ 解读 Nmap 扫描结果
- ✅ 使用 Nmap 的输出选项保存结果

---

## 📖 背景知识

### 什么是 Nmap？

**Nmap**（Network Mapper）是最著名的**网络扫描和发现工具**，由 Gordon Lyon (Fyodor) 开发。它是网络安全工程师、系统管理员和渗透测试人员的必备工具。

**主要功能：**

| 功能 | 说明 |
|------|------|
| **主机发现** | 发现网络中的活跃主机 |
| **端口扫描** | 探测目标开放的端口和服务 |
| **版本检测** | 识别服务的具体版本 |
| **操作系统识别** | 猜测目标主机的操作系统 |
| **脚本扫描** | 使用 Nmap Scripting Engine (NSE) 进行高级检测 |

### Nmap 扫描原理

Nmap 通过发送不同类型的网络数据包到目标主机，根据响应来判断：

```
Nmap 工作流程：

1. 主机发现（Host Discovery）
   ↓
   发送 ICMP Echo Request / TCP SYN 等
   ↓
   判断主机是否在线
      ↓
2. 端口扫描（Port Scanning）
   ↓
   对目标端口发送探测包
   ↓
   根据响应判断端口状态
      ↓
3. 服务/版本检测（Service/Version Detection）
   ↓
   连接开放端口，分析 Banner 信息
   ↓
   识别服务名称和版本
      ↓
4. 操作系统识别（OS Detection）
   ↓
   分析 TCP/IP 协议栈指纹
   ↓
   猜测操作系统类型和版本
```

### 端口状态说明

| 状态 | 含义 |
|------|------|
| **open** | 端口开放，有服务在监听 |
| **closed** | 端口关闭，但没有被过滤 |
| **filtered** | 端口被防火墙过滤，无法确定状态 |
| **unfiltered** | 端口可访问，但无法确定开放/关闭（ACK扫描） |
| **open\|filtered** | 无法确定是开放还是被过滤（UDP、FIN、NULL、Xmas扫描） |
| **closed\|filtered** | 无法确定是关闭还是被过滤（IP ID idle扫描） |

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或明确授权的系统**进行扫描
> - 未经授权扫描他人网络属于**违法行为**
> - 建议在本地虚拟机或隔离的实验环境中练习
> - 扫描前确保已获得书面授权

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 Nmap 是否已安装
nmap --version

# 如未安装，执行：
# Ubuntu/Debian
sudo apt update
sudo apt install -y nmap

# CentOS/RHEL
sudo yum install -y nmap

# macOS
brew install nmap

# Windows
# 下载安装包：https://nmap.org/download.html
```

### 准备靶机（强烈推荐）

为了安全合法地练习，建议启动本地靶机：

```bash
# 方法1：使用 Metasploitable2（最经典的靶机）
docker run -d -p 22:22 -p 80:80 -p 443:443 \
  --name metasploitable tleemcjr/metasploitable2

# 查看靶机IP
docker inspect metasploitable | grep IPAddress

# 方法2：使用 Kali 自带的靶机环境
# 在 Kali 中扫描自己
sudo nmap localhost

# 方法3：扫描本地网络（确保是你自己的网络）
# 例如：扫描你的路由器
nmap 192.168.1.1
```

---

## 👨‍💻 实验步骤

### Step 1：了解 Nmap 基本语法

```bash
# 查看帮助
nmap -h

# 基本语法格式
nmap [扫描类型] [选项] <目标>
```

**最简单的扫描：**

```bash
# 扫描单个 IP
nmap 192.168.1.100

# 扫描主机名
nmap scanme.nmap.org   # Nmap 官方提供的合法测试目标

# 扫描你自己的机器
nmap localhost
nmap 127.0.0.1
```

**输出示例：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.100
Host is up (0.0023s latency).
Not shown: 995 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
3389/tcp open  ms-wbt-server
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 2.34 seconds
```

---

### Step 2：TCP Connect 扫描（-sT）

**TCP Connect 扫描**是最基本的扫描方式，使用操作系统的完整 TCP 连接（三次握手）。

```bash
# TCP Connect 扫描
nmap -sT 192.168.1.100

# 指定端口范围
nmap -sT -p 1-1000 192.168.1.100

# 扫描特定端口
nmap -sT -p 22,80,443 192.168.1.100
```

**工作原理：**

```
完整 TCP 三次握手：

1. 发送 SYN →
2. ← 收到 SYN-ACK（端口开放）
   或 RST（端口关闭）
3. 发送 ACK →

优点：
- 不需要 root 权限
- 可靠性高

缺点：
- 容易被日志记录
- 速度较慢
```

**输出示例：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.100
Host is up (0.0034s latency).
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 0.45 seconds
```

---

### Step 3：SYN Stealth 扫描（-sS）

**SYN Stealth 扫描**（半开扫描）是 Nmap 的默认扫描方式，只完成 TCP 三次握手的前两步。

```bash
# SYN Stealth 扫描（需要 root 权限）
sudo nmap -sS 192.168.1.100

# 指定端口
sudo nmap -sS -p 1-65535 192.168.1.100

# 加快速度（-T4 表示快速模式）
sudo nmap -sS -T4 192.168.1.100
```

**工作原理：**

```
半开扫描（Stealth）：

1. 发送 SYN →
2. ← 收到 SYN-ACK（端口开放）
   或 RST（端口关闭）
3. 发送 RST → （不完成三次握手）

优点：
- 难以被日志记录
- 速度更快
- 不需要完成连接

缺点：
- 需要 root 权限（发送原始包）
```

**输出示例：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.100
Host is up (0.00031s latency).
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
3306/tcp open  mysql

Nmap done: 1 IP address (1 host up) scanned in 1.23 seconds
```

---

### Step 4：UDP 扫描（-sU）

**UDP 扫描**用于发现开放的 UDP 端口（如 DNS、SNMP、DHCP 等）。

```bash
# UDP 扫描（通常很慢）
sudo nmap -sU 192.168.1.100

# 同时扫描 TCP 和 UDP
sudo nmap -sS -sU 192.168.1.100

# 只扫描常用 UDP 端口（加快速度）
sudo nmap -sU --top-ports 100 192.168.1.100
```

**注意事项：**

- UDP 扫描**非常慢**（因为 UDP 是无连接协议，需要等待超时）
- 很多 UDP 端口会被标记为 `open|filtered`（无法确定状态）
- 使用 `--open` 选项只显示开放端口

**输出示例：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.100
Host is up (0.00021s latency).
PORT    STATE         SERVICE
53/udp  open          domain
67/udp  open|filtered dhcps
123/udp open          ntp
161/udp open          snmp

Nmap done: 1 IP address (1 host up) scanned in 45.23 seconds
```

---

### Step 5：服务和版本检测（-sV）

**版本检测**可以识别开放端口上运行的服务和版本信息。

```bash
# 启用版本检测
nmap -sV 192.168.1.100

# 设置版本检测强度（0-9，默认7）
nmap -sV --version-intensity 5 192.168.1.100

# 同时启用操作系统检测
sudo nmap -sV -O 192.168.1.100
```

**输出示例：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.100
Host is up (0.0021s latency).
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.4p1 Debian 10+deb9u7
80/tcp   open  http       Apache httpd 2.4.25 ((Debian))
443/tcp  open  ssl/http   Apache httpd 2.4.25 ((Debian))
3306/tcp open  mysql      MySQL 5.5.60-0+deb9u1

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.45 seconds
```

---

### Step 6：输出格式选项

Nmap 支持多种输出格式，方便保存和后续分析。

```bash
# 正常输出（默认）
nmap 192.168.1.100 -oN result.txt

# XML 格式输出（便于程序解析）
nmap 192.168.1.100 -oX result.xml

# Greppable 格式（便于 grep 处理）
nmap 192.168.1.100 -oG result.grep

# 同时输出多种格式
nmap 192.168.1.100 -oA scan_results   # 会生成 .nmap, .xml, .grep 三个文件

# 安静模式（只显示最终结果）
nmap 192.168.1.100 -oN - -q

# 详细模式（-v 或 -vv）
nmap -sS -sV -v 192.168.1.100
```

**输出文件示例（result.txt）：**

```
# Nmap 7.94 scan initiated Sun May 30 21:00:00 2026
Nmap scan report for 192.168.1.100
Host is up (0.0023s latency).
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] Nmap 的默认扫描类型是什么？它有什么优缺点？
- [ ] TCP Connect 扫描和 SYN Stealth 扫描有什么区别？
- [ ] 如何扫描 UDP 端口？为什么 UDP 扫描很慢？
- [ ] `-sV` 参数的作用是什么？
- [ ] 如何将 Nmap 扫描结果保存为 XML 格式？

---

## 💡 进阶挑战

### 挑战1：综合扫描

```bash
# 组合多种扫描技术
sudo nmap -sS -sV -O --traceroute 192.168.1.100 -oA comprehensive_scan

# 参数说明：
# -sS           SYN 扫描
# -sV           版本检测
# -O            操作系统检测
# --traceroute  追踪路由
# -oA           输出所有格式
```

### 挑战2：快速扫描模式

```bash
# 快速扫描常用端口（前100个）
nmap -F 192.168.1.100

# 极速模式（可能不准确）
nmap -T5 --top-ports 100 192.168.1.100

#  timing 模板：
# -T0  Paranoid（偏执模式，非常慢，用于IDS规避）
# -T1  Sneaky（隐蔽模式）
# -T2  Polite（礼貌模式）
# -T3  Normal（正常模式，默认）
# -T4  Aggressive（激进模式，快速）
# -T5  Insane（疯狂模式，极快但可能丢包）
```

### 挑战3：扫描防火墙后的主机

```bash
# 使用碎片包（绕过某些防火墙）
sudo nmap -f 192.168.1.100

# 使用诱饵（Decoy）
sudo nmap -D RND:10 192.168.1.100

# 指定源端口（某些防火墙允许来自特定端口的流量）
sudo nmap --source-port 53 192.168.1.100

# 使用 TCP ACK 扫描（绕过无状态防火墙）
sudo nmap -sA 192.168.1.100
```

---

## 🛡️ 防御措施（如何防止被 Nmap 扫描）

了解扫描技术后，也要知道如何防御：

| 防御措施 | 说明 |
|---------|------|
| **防火墙配置** | 使用防火墙限制入站流量（如 iptables、pf、Windows Firewall） |
| **端口过滤** | 只开放必要的端口，关闭未使用的服务 |
| **IDS/IPS 部署** | 使用入侵检测/防御系统（如 Snort、Suricata）检测扫描行为 |
| **端口敲门（Port Knocking）** | 通过特定顺序连接端口来"敲门"，动态开放端口 |
| **模糊响应** | 配置系统对扫描返回误导性信息 |
| **速率限制** | 限制单位时间内来自同一 IP 的连接数 |

### 使用 Fail2Ban 防御扫描

```bash
# 安装 Fail2Ban
sudo apt install -y fail2ban

# 配置 Nmap 扫描检测（需要自定义规则）
sudo cat > /etc/fail2ban/filter.d/nmap-scan.conf << 'EOF'
[Definition]
failregex = ^.* SRC=<HOST> .* SPT=.* DPT=.* $
ignoreregex =
EOF

# 启用防护
sudo cat > /etc/fail2ban/jail.local << 'EOF'
[nmap-scan]
enabled = true
filter = nmap-scan
logpath = /var/log/kern.log
maxretry = 10
bantime = 3600
EOF

sudo systemctl restart fail2ban
```

### 防火墙规则示例（iptables）

```bash
# 限制单个 IP 的新连接速率（防止扫描）
sudo iptables -A INPUT -p tcp --syn -m limit \
  --limit 1/s --limit-burst 3 -j ACCEPT

# 记录并丢弃异常包
sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j LOG \
  --log-prefix "NULL scan: "
sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP

sudo iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j LOG \
  --log-prefix "Xmas scan: "
sudo iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j DROP
```

---

## 📚 参考资源

- [Nmap 官方文档](https://nmap.org/book/)
- [Nmap 参考指南（中文）](https://nmap.org/man/zh/)
- [Nmap 官方教程](https://nmap.org/book/)
- [SecTools - Nmap 使用技巧](https://sectools.org/tool/nmap/)
- [Nmap 脚本库（NSE）](https://nmap.org/nsedoc/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
目标IP：
使用的扫描类型：
发现的开放端口和服务：
版本检测结果：
遇到的问题：
解决方案：
```

---

*最后更新：2026-05-30*
