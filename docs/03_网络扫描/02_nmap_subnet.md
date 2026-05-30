# 🌐 实验04：使用 Nmap 扫描子网

> **难度**：⭐⭐⭐ 初级  
> **预计时间**：60分钟  
> **工具**：Nmap (Network Mapper)

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 CIDR 表示法和子网划分
- ✅ 使用 Nmap 批量扫描整个子网
- ✅ 排除特定主机或地址范围
- ✅ 分析和解读大规模扫描结果
- ✅ 结合 Nmap 脚本进行深度扫描

---

## 📖 背景知识

### CIDR 表示法

**CIDR**（Classless Inter-Domain Routing，无类别域间路由）是一种表示 IP 地址和子网的方法。

**格式：** `IP地址/前缀长度`

```
示例：192.168.1.0/24

192.168.1.0  → 网络地址
/24           → 前缀长度（前24位是网络部分）

二进制表示：
11000000.10101000.00000001.xxxxxxxx
↑ 前24位固定                                ↑ 后8位可变

可变位数 = 32 - 24 = 8
主机数量 = 2^8 = 256 个 IP 地址
可用主机 = 256 - 2 = 254 个（去掉网络地址和广播地址）
```

### 常用 CIDR 块

| CIDR | 子网掩码 | 主机数量 | 可用主机 | 说明 |
|------|---------|---------|---------|------|
| /32 | 255.255.255.255 | 1 | 1 | 单个主机 |
| /30 | 255.255.255.252 | 4 | 2 | 点对点链路 |
| /29 | 255.255.255.248 | 8 | 6 | 小型网络 |
| /28 | 255.255.255.240 | 16 | 14 | 小型 LAN |
| /27 | 255.255.255.224 | 32 | 30 | 中型 LAN |
| /26 | 255.255.255.192 | 64 | 62 | 中型 LAN |
| /25 | 255.255.255.128 | 128 | 126 | 大型 LAN |
| /24 | 255.255.255.0 | 256 | 254 | 典型 LAN（C类） |
| /16 | 255.255.0.0 | 65536 | 65534 | B类私有网络 |
| /8 | 255.0.0.0 | 16777216 | 16777214 | A类私有网络 |

### 子网扫描的应用场景

| 场景 | 说明 |
|------|------|
| **网络资产发现** | 发现网络中所有活跃设备 |
| **安全评估** | 识别网络中的潜在攻击面 |
| **网络管理** | 检查设备在线状态 |
| **漏洞扫描** | 为后续漏洞扫描准备目标列表 |
| **渗透测试** | 信息收集阶段的重要内容 |

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或明确授权的网络**进行扫描
> - 扫描他人的网络可能违反《网络安全法》和公司政策
> - 建议在本地虚拟机、Docker 容器或隔离的实验网络中练习
> - 企业环境中扫描前必须获得书面授权

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 Nmap 版本
nmap --version

# 建议使用 Nmap 7.80 或更高版本
# 如未安装，请参考实验03的安装方法
```

### 准备实验环境

```bash
# 方法1：在本地虚拟机网络中扫描
# 如果你使用 VirtualBox 或 VMware，可以扫描虚拟网络
# 例如：192.168.56.0/24（仅主机网络）

# 方法2：使用 Docker 创建多个容器模拟网络
# 启动多个容器
docker run -d --name web1 nginx
docker run -d --name web2 apache2
docker run -d --name db mysql:5.7

# 查看容器 IP
docker network inspect bridge

# 方法3：扫描本地回环网络（最安全）
# 127.0.0.0/8 包含了大量地址，但只有 127.0.0.1 是活跃的
nmap 127.0.0.1/32
```

---

## 👨‍💻 实验步骤

### Step 1：理解 CIDR 表示法

```bash
# 扫描单个 IP
nmap 192.168.1.100

# 扫描整个 /24 子网（254 台主机）
nmap 192.168.1.0/24

# 扫描 /16 子网（65534 台主机，会很慢！）
nmap 192.168.0.0/16

# 扫描 /8 子网（不推荐，太大了）
# nmap 10.0.0.0/8
```

**CIDR 示例：**

```bash
# 扫描 192.168.1.1 - 192.168.1.254
nmap 192.168.1.0/24

# 扫描 192.168.1.1 - 192.168.1.126
nmap 192.168.1.0/25

# 扫描 192.168.1.129 - 192.168.1.254
nmap 192.168.1.128/25

# 扫描 10.0.0.0 - 10.0.0.255
nmap 10.0.0.0/24
```

---

### Step 2：批量扫描整个子网

```bash
# 基本子网扫描（Ping 扫描 + 端口扫描）
nmap 192.168.1.0/24

# 只进行主机发现（Ping 扫描），不扫描端口
nmap -sn 192.168.1.0/24

# 扫描特定端口范围
nmap -p 1-1000 192.168.1.0/24

# 快速扫描（只扫描常用100个端口）
nmap -F 192.168.1.0/24

# 极速模式（谨慎使用）
sudo nmap -sS -T4 -p 1-1000 192.168.1.0/24
```

**输出示例（主机发现）：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.1
Host is up (0.0023s latency).
Nmap scan report for 192.168.1.100
Host is up (0.0056s latency).
Nmap scan report for 192.168.1.105
Host is up (0.0089s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 5.23 seconds
```

**输出示例（完整扫描）：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.1
Host is up (0.0012s latency).
PORT   STATE SERVICE
53/tcp open  domain
80/tcp open  http

Nmap scan report for 192.168.1.100
Host is up (0.0034s latency).
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https

Nmap scan report for 192.168.1.105
Host is up (0.0067s latency).
PORT     STATE SERVICE
22/tcp   closed ssh
80/tcp   open  http
3306/tcp open  mysql

Nmap done: 256 IP addresses (3 hosts up) scanned in 45.67 seconds
```

---

### Step 3：排除特定主机

```bash
# 排除单个 IP
nmap 192.168.1.0/24 --exclude 192.168.1.1

# 排除多个 IP
nmap 192.168.1.0/24 --exclude 192.168.1.1,192.168.1.254

# 排除 IP 范围
nmap 192.168.1.0/24 --exclude 192.168.1.1-50

# 从文件中读取排除列表
echo "192.168.1.1" > exclude.txt
echo "192.168.1.100" >> exclude.txt
nmap 192.168.1.0/24 --excludefile exclude.txt
```

**实战场景：**

```bash
# 排除网关（通常是 .1）
nmap 192.168.1.0/24 --exclude 192.168.1.1

# 排除打印机、IP 摄像头等 IoT 设备
nmap 192.168.1.0/24 --exclude 192.168.1.200-254

# 排除已知的安全设备（防火墙、IDS等）
nmap 192.168.1.0/24 --exclude 192.168.1.10,192.168.1.11
```

---

### Step 4：输出结果分析

```bash
# 保存扫描结果（多种格式）
nmap -sS -sV -oA subnet_scan 192.168.1.0/24

# 这会生成三个文件：
# subnet_scan.nmap   （普通文本格式）
# subnet_scan.xml    （XML 格式，便于解析）
# subnet_scan.gnmap  （Greppable 格式，便于 grep）

# 查看普通文本结果
cat subnet_scan.nmap

# 使用 grep 提取开放端口的主机
grep "open" subnet_scan.nmap

# 提取所有开放了 SSH 端口的主机
grep "22/tcp" subnet_scan.nmap | grep "open"
```

**解析 XML 输出：**

```bash
# 使用 xmllint 格式化 XML
xmllint --format subnet_scan.xml | less

# 使用 Python 解析 XML
python3 << 'EOF'
import xml.etree.ElementTree as ET

tree = ET.parse('subnet_scan.xml')
root = tree.getroot()

for host in root.findall('host'):
    addr = host.find('address').get('addr')
    ports = host.find('ports')
    if ports is not None:
        for port in ports.findall('port'):
            state = port.find('state').get('state')
            if state == 'open':
                portid = port.get('portid')
                service = port.find('service')
                if service is not None:
                    svcname = service.get('name')
                    print(f"{addr}:{portid} - {svcname}")
EOF
```

---

### Step 5：结合脚本扫描

```bash
# 使用默认脚本扫描（-sC 等价于 --script=default）
nmap -sC -sV 192.168.1.0/24

# 使用特定脚本
nmap --script http-title,http-headers 192.168.1.0/24

# 使用脚本类别
nmap --script "discovery,intrusive" 192.168.1.0/24

# 常见脚本类别：
# - auth          ：认证相关的脚本
# - broadcast    ：广播相关的脚本
# - brute        ：暴力破解脚本
# - default      ：默认脚本（-sC）
# - discovery    ：信息收集脚本
# - dos          ：拒绝服务测试（慎用！）
# - exploit      ：漏洞利用脚本（危险！）
# - external     ：外部资源查询
# - fuzzer       ：模糊测试
# - intrusive    ：侵入性脚本（可能影响目标）
# - malware      ：恶意软件检测
# - safe         ：安全脚本
# - version      ：版本检测增强
# - vuln         ：漏洞检测

# 扫描 HTTP 服务
nmap --script http-enum,http-headers,http-title \
  -p 80,443,8080 192.168.1.0/24

# 扫描 SMB 服务
nmap --script smb-os-discovery,smb-enum-shares \
  -p 445 192.168.1.0/24
```

**输出示例（脚本扫描）：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.100
Host is up (0.0034s latency).
PORT   STATE SERVICE
80/tcp open  http
| http-title: Apache2 Default Page
| http-headers: 
|   Server: Apache/2.4.25 (Debian)
|   Last-Modified: Sat, 20 Nov 2021 12:34:56 GMT
|_  Content-Type: text/html

Nmap scan report for 192.168.1.105
Host is up (0.0056s latency).
PORT     STATE SERVICE
445/tcp  open  microsoft-ds
| smb-os-discovery: 
|   OS: Windows 10 Pro 19042 (Windows 10 Pro 19042)
|   Computer name: DESKTOP-ABC123
|   NetBIOS computer name: DESKTOP-ABC123
|   Workgroup: WORKGROUP
|_  System time: 2026-05-30T21:00:00+08:00
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] CIDR 表示法 `/24` 表示什么？包含多少个 IP 地址？
- [ ] 如何扫描整个子网但不扫描网关？
- [ ] `-sn` 参数的作用是什么？
- [ ] 如何将扫描结果保存为 XML 格式？
- [ ] 如何使用 Nmap 脚本扫描 HTTP 服务的标题？

---

## 💡 进阶挑战

### 挑战1：生成存活主机列表

```bash
# 只进行主机发现，输出到文件
nmap -sn 192.168.1.0/24 -oG ping_results.grep

# 提取存活主机 IP
grep "Up" ping_results.grep | cut -d' ' -f2 > live_hosts.txt

# 查看结果
cat live_hosts.txt
```

### 挑战2：针对特定服务扫描

```bash
# 找出所有开放 SSH 的主机
nmap -p 22 --open 192.168.1.0/24 -oG ssh_hosts.grep
grep "22/open" ssh_hosts.grep | cut -d' ' -f2

# 找出所有开放 HTTP/HTTPS 的主机
nmap -p 80,443 --open 192.168.1.0/24 -oG web_hosts.grep
grep "80/open\|443/open" web_hosts.grep | cut -d' ' -f2
```

### 挑战3：自动化扫描和分析

```bash
# 创建自动化扫描脚本
cat > subnet_scan.sh << 'EOF'
#!/bin/bash

SUBNET="192.168.1.0/24"
OUTPUT_DIR="scan_results_$(date +%Y%m%d_%H%M%S)"

mkdir -p "$OUTPUT_DIR"

echo "[*] 开始主机发现..."
nmap -sn "$SUBNET" -oG "$OUTPUT_DIR/ping.grep"
grep "Up" "$OUTPUT_DIR/ping.grep" | cut -d' ' -f2 > "$OUTPUT_DIR/live_hosts.txt"

echo "[*] 发现 $(wc -l < "$OUTPUT_DIR/live_hosts.txt") 台主机"

echo "[*] 开始端口扫描..."
nmap -sS -sV -p 1-1000 -iL "$OUTPUT_DIR/live_hosts.txt" \
  -oA "$OUTPUT_DIR/port_scan"

echo "[*] 开始脚本扫描..."
nmap -sC -sV -p 22,80,443,445 -iL "$OUTPUT_DIR/live_hosts.txt" \
  -oA "$OUTPUT_DIR/script_scan"

echo "[*] 扫描完成！结果保存在 $OUTPUT_DIR/"
EOF

chmod +x subnet_scan.sh
./subnet_scan.sh
```

### 挑战4：扫描限速和性能优化

```bash
# 限制扫描速率（避免被 IDS 检测）
sudo nmap -sS --max-rate 100 192.168.1.0/24

# 增加并行度（加快扫描速度）
sudo nmap -sS -T4 --min-parallelism 32 192.168.1.0/24

# 分片发送（绕过某些防火墙）
sudo nmap -f 192.168.1.0/24

# 使用时间模板
sudo nmap -sS -T2 192.168.1.0/24   # Polite 模式（慢但隐蔽）
sudo nmap -sS -T4 192.168.1.0/24   # Aggressive 模式（快但明显）
```

---

## 🛡️ 防御措施（如何防止子网扫描）

### 网络层防御

| 防御措施 | 说明 |
|---------|------|
| **网络分段** | 使用 VLAN 和子网划分，隔离不同安全域 |
| **防火墙策略** | 配置防火墙限制 ICMP 和 TCP 探测 |
| **入侵检测系统（IDS）** | 部署 Snort、Suricata 检测扫描行为 |
| **入侵防御系统（IPS）** | 自动阻断可疑扫描流量 |
| **网络访问控制（NAC）** | 控制设备接入网络的权限 |

### 主机层防御

| 防御措施 | 说明 |
|---------|------|
| **关闭不必要的服务** | 减少攻击面 |
| **修改默认端口** | 增加攻击者发现服务的难度 |
| **启用防火墙** | 使用 iptables、ufw、Windows Firewall |
| **配置 IDS/IPS** | 主机级入侵检测（如 OSSEC） |
| **禁用 ICMP 响应** | 使 Ping 扫描失效 |

### 防火墙配置示例

```bash
# Linux iptables - 防止 Ping 扫描
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Linux iptables - 限制 TCP 连接速率
sudo iptables -A INPUT -p tcp --syn -m limit \
  --limit 1/s --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --syn -j DROP

# Linux ufw - 只允许必要端口
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw enable

# Windows PowerShell - 启用防火墙
Set-NetFirewallProfile -Profile Domain,Public,Private \
  -Enabled True
```

### 部署 Suricata IDS

```bash
# 安装 Suricata
sudo apt install -y suricata

# 配置规则（检测 Nmap 扫描）
sudo cat >> /etc/suricata/suricata.yaml << 'EOF'
rule-files:
  - /etc/suricata/rules/nmap.rules
EOF

# 创建 Nmap 检测规则
sudo cat > /etc/suricata/rules/nmap.rules << 'EOF'
alert tcp any any -> any any (msg:"Nmap TCP Connect Scan"; \
  flags:S; threshold: type both, track by_src, count 20, seconds 10; \
  sid:1000001; rev:1;)

alert tcp any any -> any any (msg:"Nmap SYN Stealth Scan"; \
  flags:S,CE; threshold: type both, track by_src, count 20, seconds 10; \
  sid:1000002; rev:1;)
EOF

# 启动 Suricata
sudo systemctl start suricata
```

---

## 📚 参考资源

- [Nmap 官方文档 - 主机发现](https://nmap.org/book/man-host-discovery.html)
- [Nmap 官方文档 - 脚本使用](https://nmap.org/book/man-nse.html)
- [CIDR 计算器](https://www.cidr.net/)
- [Nmap 脚本库（NSE）](https://nmap.org/nsedoc/)
- [Suricata 官方文档](https://suricata.readthedocs.io/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
扫描的子网：
发现的活跃主机数量：
开放的常见服务：
使用的 Nmap 参数：
遇到的问题：
解决方案：
```

---

*最后更新：2026-05-30*
