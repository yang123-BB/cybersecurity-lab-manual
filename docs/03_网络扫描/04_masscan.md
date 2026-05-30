# ⚡ 实验17：使用 Masscan 扫描端口

> **难度**：⭐⭐⭐ 初级  
> **预计时间**：60分钟  
> **工具**：Masscan

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 Masscan 的工作原理和特点
- ✅ 掌握 Masscan 的基本命令语法
- ✅ 使用 Masscan 进行大规模端口扫描
- ✅ 控制扫描速率（避免被封禁）
- ✅ 对比 Masscan 和 Nmap 的优缺点

---

## 📖 背景知识

### 什么是 Masscan？

**Masscan** 是一款**高速端口扫描器**，被称为"互联网规模的端口扫描器"。它由 Robert David Graham 开发，设计目标是**极速扫描**。

**核心特点：**

| 特点 | 说明 |
|------|------|
| **极速** | 可以每秒发送 **1000万个数据包** |
| **异步** | 使用异步 I/O，类似于 `scanrand`、`unicornscan` |
| **TCP/IP 栈** | 自带简化的 TCP/IP 栈，不依赖操作系统 |
| **无状态** | 不维护连接状态表 |
| **兼容 Nmap** | 参数语法与 Nmap 类似 |

### Masscan vs Nmap

| 特性 | Masscan | Nmap |
|------|---------|------|
| **速度** | 极快（1000万pps） | 较慢（几千pps） |
| **准确性** | 较高（可能丢包） | 非常高 |
| **功能** | 专注端口扫描 | 功能全面（服务检测、脚本等） |
| **资源占用** | 较高（需要调优） | 较低 |
| **适用场景** | 大规模扫描（整个互联网） | 精确扫描（单个主机/子网） |
| **输出格式** | 支持 Nmap 格式 | 多种格式 |
| **脚本支持** | 无 | 强大的 NSE |

### Masscan 的工作原理

```
Masscan 工作流程：

1. 生成目标 IP 列表
   ↓
2. 生成目标端口列表
   ↓
3. 使用自带的 TCP/IP 栈构造数据包
   ↓
4. 高速发送 SYN 包（异步 I/O）
   ↓
5. 接收 SYN-ACK 响应（端口开放）
   ↓
6. 发送 RST 终止连接
   ↓
7. 输出结果
```

**速度对比：**

```
扫描整个互联网（所有端口）：

Nmap：   需要数年
Masscan： 需要几天（6天扫描全部IPv4地址的1-65535端口）
```

### 速率控制的重要性

Masscan 的极速特性是一把双刃剑：

| 问题 | 说明 |
|------|------|
| **网络拥塞** | 高速扫描可能导致网络拥塞 |
| **被防火墙封禁** | 触发 IDS/IPS 告警 |
| **丢包** | 速率过高导致响应包丢失 |
| **法律问题** | 异常流量可能引起 ISP 注意 |

**因此，必须控制扫描速率！**

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或明确授权的系统**进行扫描
> - **禁止扫描整个互联网**（即使技术上可行）
> - 大规模扫描可能违反 ISP 的服务条款
> - 建议在**本地虚拟机或隔离网络**中练习
> - 使用 `--rate` 参数控制扫描速率

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 Masscan 是否已安装
masscan --version

# 如未安装：

# Ubuntu/Debian
sudo apt update
sudo apt install -y masscan

# CentOS/RHEL
sudo yum install -y masscan

# macOS
brew install masscan

# 从源码编译（最新版本）
sudo apt install -y git gcc make libpcap-dev
git clone https://github.com/robertdavidgraham/masscan.git
cd masscan
make
sudo make install
```

### 准备靶机

```bash
# 启动多个容器模拟网络
docker run -d --name web1 -p 80:80 nginx
docker run -d --name web2 -p 8080:80 apache2
docker run -d --name db -p 3306:3306 mysql:5.7

# 查看容器 IP
docker network inspect bridge | grep IPAddress
```

---

## 👨‍💻 实验步骤

### Step 1：了解 Masscan 基本语法

```bash
# 查看帮助
masscan --help

# 基本语法格式
masscan [目标] -p [端口] [选项]
```

**最简单扫描：**

```bash
# 扫描单个 IP 的 80 端口
masscan 192.168.1.100 -p 80

# 扫描多个端口
masscan 192.168.1.100 -p 22,80,443

# 扫描端口范围
masscan 192.168.1.100 -p 1-1000

# 扫描子网
masscan 192.168.1.0/24 -p 80
```

**输出示例：**

```
Starting masscan 1.3.2 (http://bit.ly/14GZzcT) at 2026-05-30 13:00:00 GMT
Initiating SYN Stealth Scan
Scanning 256 hosts [1 port/host]

Discovered open port 80/tcp on 192.168.1.100
Discovered open port 80/tcp on 192.168.1.105
```

---

### Step 2：控制扫描速率（重要！）

```bash
# 设置扫描速率（每秒数据包数）
masscan 192.168.1.0/24 -p 1-1000 --rate 100

# 速率建议：
# --rate 100     ：适合本地网络（慢速）
# --rate 1000    ：适合本地网络（中速）
# --rate 10000   ：适合高速网络（快速）
# --rate 100000  ：极快（可能导致丢包）

# 示例：以每秒 1000 个数据包的速率扫描
masscan 192.168.1.0/24 -p 1-1000 --rate 1000

# 示例：极速扫描（慎用！）
masscan 192.168.1.0/24 -p 1-65535 --rate 100000
```

**速率选择指南：**

| 场景 | 推荐速率 | 说明 |
|------|---------|------|
| **本地虚拟机** | 100-1000 | 避免宿主机资源耗尽 |
| **局域网** | 1000-10000 | 根据网络带宽调整 |
| **互联网扫描** | 100-1000 | 避免触发 IDS/IPS |
| **授权渗透测试** | 10000+ | 需要对方书面授权 |

---

### Step 3：扫描整个子网

```bash
# 扫描整个 /24 子网的 80 端口
masscan 192.168.1.0/24 -p 80 --rate 1000

# 扫描整个 /24 子网的所有端口（1-65535）
masscan 192.168.1.0/24 -p 1-65535 --rate 1000

# 扫描多个子网
masscan 192.168.1.0/24,192.168.2.0/24 -p 80 --rate 1000

# 从文件读取目标列表
echo "192.168.1.100" > targets.txt
echo "192.168.1.101" >> targets.txt
masscan -iL targets.txt -p 80 --rate 1000
```

**输出示例（子网扫描）：**

```
Starting masscan 1.3.2 (http://bit.ly/14GZzcT) at 2026-05-30 13:00:00 GMT
Initiating SYN Stealth Scan
Scanning 256 hosts [1000 ports/host]

Discovered open port 80/tcp on 192.168.1.100
Discovered open port 443/tcp on 192.168.1.100
Discovered open port 22/tcp on 192.168.1.105
Discovered open port 3306/tcp on 192.168.1.110
```

---

### Step 4：输出格式选项

```bash
# 默认输出（终端）
masscan 192.168.1.0/24 -p 80 --rate 1000

# 输出到文件（简单列表格式）
masscan 192.168.1.0/24 -p 80 --rate 1000 -oG result.grep

# 输出为 Nmap 兼容格式（推荐）
masscan 192.168.1.0/24 -p 80 --rate 1000 -oN result.nmap

# 输出为 JSON 格式（便于程序解析）
masscan 192.168.1.0/24 -p 80 --rate 1000 -oJ result.json

# 输出为 XML 格式
masscan 192.168.1.0/24 -p 80 --rate 1000 -oX result.xml
```

**输出格式参数：**

| 参数 | 格式 | 说明 |
|------|------|------|
| `-oN` | Nmap 格式 | 兼容 Nmap 输出 |
| `-oX` | XML 格式 | 结构化数据 |
| `-oG` | Greppable 格式 | 便于 grep 处理 |
| `-oJ` | JSON 格式 | 便于程序解析 |
| `-oB` | Binary 格式 | 二进制格式（仅供 Masscan 读取） |

**JSON 输出示例：**

```json
{
  "ip": "192.168.1.100",
  "timestamp": "2026-05-30T13:00:00.000000",
  "ports": [
    {
      "port": 80,
      "proto": "tcp",
      "status": "open",
      "reason": "syn-ack",
      "ttl": 64
    }
  ]
}
```

---

### Step 5：排除特定主机

```bash
# 排除单个 IP
masscan 192.168.1.0/24 -p 80 --exclude 192.168.1.1 --rate 1000

# 排除多个 IP
masscan 192.168.1.0/24 -p 80 --exclude 192.168.1.1,192.168.1.254 \
  --rate 1000

# 从文件读取排除列表
echo "192.168.1.1" > exclude.txt
echo "192.168.1.254" >> exclude.txt
masscan 192.168.1.0/24 -p 80 --excludefile exclude.txt --rate 1000
```

---

### Step 6：结合 Nmap 使用

Masscan 速度快但功能有限，通常与 Nmap 结合使用：

```bash
# 第1步：使用 Masscan 快速发现开放端口
masscan 192.168.1.0/24 -p 1-65535 --rate 10000 -oN masscan_results.nmap

# 第2步：使用 Nmap 对开放端口进行详细扫描
# 提取开放端口列表
grep "Discovered open port" masscan_results.nmap | \
  awk '{print $6}' | cut -d'/' -f1 | sort -n | uniq

# 第3步：使用 Nmap 进行服务版本检测和脚本扫描
nmap -sV -sC -p 22,80,443,3306 192.168.1.100
```

**自动化脚本示例：**

```bash
cat > masscan_nmap.sh << 'EOF'
#!/bin/bash

SUBNET="192.168.1.0/24"
RATE=10000

echo "[*] 第1步：使用 Masscan 快速扫描..."
masscan $SUBNET -p 1-65535 --rate $RATE -oN masscan_results.nmap

echo "[*] 第2步：提取开放端口..."
OPEN_PORTS=$(grep "Discovered open port" masscan_results.nmap | \
  awk '{print $6}' | cut -d'/' -f1 | sort -n | uniq | \
  tr '\n' ',' | sed 's/,$//')

echo "[*] 发现的开放端口：$OPEN_PORTS"

echo "[*] 第3步：使用 Nmap 详细扫描..."
grep "Discovered open port" masscan_results.nmap | \
  awk '{print $4}' | sort -u | while read IP; do
    echo "  扫描 $IP ..."
    nmap -sV -sC -p $OPEN_PORTS $IP -oN nmap_$IP.nmap
done

echo "[*] 扫描完成！"
EOF

chmod +x masscan_nmap.sh
./masscan_nmap.sh
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] Masscan 和 Nmap 的主要区别是什么？
- [ ] 为什么需要控制 Masscan 的扫描速率（`--rate`）？
- [ ] 如何将 Masscan 的扫描结果保存为 Nmap 格式？
- [ ] Masscan 适合扫描整个互联网吗？为什么？
- [ ] 如何结合使用 Masscan 和 Nmap？

---

## 💡 进阶挑战

### 挑战1：扫描大型网络

```bash
# 扫描 /16 网络（65534 台主机）
# 警告：这会非常慢，即使使用 Masscan
masscan 192.168.0.0/16 -p 80 --rate 10000 -oJ result.json

# 使用多个网卡（如果有）
masscan 192.168.0.0/16 -p 80 --rate 10000 \
  --adapter eth0 --adapter eth1 -oJ result.json
```

### 挑战2：优化扫描性能

```bash
# 增加发送缓冲区（需要 root 权限）
sudo masscan 192.168.1.0/24 -p 1-65535 --rate 100000 \
  --max-rate 1000000

# 使用特定的网络适配器
masscan 192.168.1.0/24 -p 80 --rate 10000 --adapter eth0

# 设置源 IP（欺骗源 IP，某些场景下有用）
masscan 192.168.1.0/24 -p 80 --rate 10000 --source-ip 192.168.1.200

# 设置源端口范围
masscan 192.168.1.0/24 -p 80 --rate 10000 --source-port 40000-50000
```

### 挑战3：解析 JSON 输出

```bash
# 使用 jq 解析 JSON 输出
masscan 192.168.1.0/24 -p 80 --rate 1000 -oJ result.json

# 提取开放端口的主机 IP
cat result.json | jq -r '.[].ip' | sort -u

# 提取每个主机开放的端口
cat result.json | jq -r '.[] | "\(.ip):\(.ports[].port)"'

# 生成易于阅读的报告
cat result.json | jq -r '.[] | "\(.ip) -> \(.ports[].port)/\(.ports[].proto)"' | \
  sort > report.txt
```

### 挑战4：模拟互联网扫描（仅限授权环境！）

```bash
# 扫描某个组织授权的 IP 段
# 示例：扫描 10.0.0.0/8（私有网络，仅用于演示）
masscan 10.0.0.0/8 -p 80,443,22,21,23,3389 --rate 100000 \
  --banners -oJ internet_scan.json

# --banners ：尝试获取 Banner 信息（类似 Nmap 的版本检测）
```

---

## 🛡️ 防御措施（如何防御大规模端口扫描）

### 网络层防御

| 防御措施 | 说明 |
|---------|------|
| **防火墙** | 限制入站流量，只开放必要端口 |
| **IDS/IPS** | 检测并阻断异常扫描行为（如 Snort、Suricata） |
| **速率限制** | 限制单位时间内来自同一 IP 的连接数 |
| **黑洞路由** | 将攻击流量引流到黑洞 |
| **Anycast** | 分散流量到多个节点 |

### 主机层防御

| 防御措施 | 说明 |
|---------|------|
| **关闭不必要端口** | 减少攻击面 |
| **端口敲门（Port Knocking）** | 动态开放端口 |
| **单包授权（SPA）** | 使用 fwknop 隐藏端口 |
| **Fail2Ban** | 自动封禁异常 IP |

### 配置示例

```bash
# Linux iptables - 限制连接速率
sudo iptables -A INPUT -p tcp --syn -m limit \
  --limit 1/s --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

# Linux iptables - 检测端口扫描（利用 recent 模块）
sudo iptables -A INPUT -p tcp --syn -m recent \
  --name portscan --set
sudo iptables -A INPUT -p tcp --syn -m recent \
  --name portscan --rcheck --seconds 60 --hitcount 20 -j DROP

# 使用 fwknop 隐藏 SSH 端口
sudo apt install -y fwknop-server fwknop-client
sudo fwknopd --restart

# 客户端访问（需要先发送 SPA 包）
fwknop -A tcp/22 -a <客户端IP> -D <服务器IP>
```

### 部署 Suricata 检测 Masscan

```bash
# Suricata 规则检测高速端口扫描
sudo cat > /etc/suricata/rules/masscan.rules << 'EOF'
alert tcp any any -> any any (msg:"Possible Masscan Port Scan"; \
  flags:S; threshold: type both, track by_src, count 100, seconds 1; \
  sid:1000003; rev:1;)
EOF

# 重启 Suricata
sudo systemctl restart suricata
```

---

## 📚 参考资源

- [Masscan 官方 GitHub](https://github.com/robertdavidgraham/masscan)
- [Masscan 使用指南](https://github.com/robertdavidgraham/masscan/blob/master/README.md)
- [高性能端口扫描技术](https://blog.zsec.uk/masscan-lab/)
- [Masscan vs Nmap 对比](https://www.hackingarticles.in/masscan-vs-nmap/)
- [防御端口扫描](https://www.sans.org/white-papers/33343/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
扫描的目标：
使用的速率（--rate）：
扫描结果：
与 Nmap 的对比：
遇到的问题：
解决方案：
```

---

*最后更新：2026-05-30*
