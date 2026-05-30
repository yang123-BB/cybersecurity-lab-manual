# 🔓 实验15：使用 John the Ripper 破解 ZIP 密码

> **难度**：⭐⭐⭐ 初级  
> **预计时间**：60分钟  
> **工具**：John the Ripper、zip2john、hashcat（可选）

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 John the Ripper（JtR）的工作原理
- ✅ 掌握离线密码破解的基本概念
- ✅ 学会创建加密 ZIP 文件用于测试
- ✅ 使用 zip2john 提取 ZIP 文件的哈希
- ✅ 使用 JtR 进行字典攻击、暴力破解和规则攻击
- ✅ 了解如何破解 Linux shadow 文件和 Windows NTLM 哈希
- ✅ 掌握强加密文件和密码的安全实践

---

## 📖 背景知识

### 什么是 John the Ripper？

**John the Ripper**（简称 JtR）是一款著名的**离线密码破解工具**，由 OpenWall 团队开发。与 Hydra 等**在线破解工具**不同，JtR 是**离线**的，意味着：

| 对比项 | 在线破解（Hydra） | 离线破解（JtR） |
|--------|------------------|----------------|
| **工作原理** | 实时向目标发送登录请求 | 先获取密码哈希，本地破解 |
| **速度** | 受网络延迟和目标限制 | 仅受本地硬件限制（很快） |
| **被检测风险** | 高（产生大量网络流量） | 低（完全本地操作） |
| **适用场景** | 破解在线服务（SSH、FTP等） | 破解哈希文件（shadow、NTLM等） |
| **示例** | 破解 SSH 登录密码 | 破解 /etc/shadow 文件 |

### John the Ripper 的工作原理

```
JtR 工作流程：

1. 获取密码哈希（如从 /etc/shadow、ZIP 文件等）
      ↓
2. 使用 zip2john、unshadow 等工具转换哈希格式
      ↓
3. JtR 读取哈希文件
      ↓
4. 根据攻击模式（字典、暴力、规则），生成候选密码
      ↓
5. 计算候选密码的哈希值，与目标哈希比对
      ↓
6. 匹配成功，输出密码
```

**关键概念：**

- **哈希（Hash）**：密码经过单向加密函数后的结果（如 MD5、SHA-1、bcrypt）
- **彩虹表（Rainbow Table）**：预计算的哈希值对应表（JtR 支持，但现代哈希加盐后无效）
- **字典攻击**：使用密码字典逐个尝试
- **暴力破解**：枚举所有可能的密码组合
- **规则攻击**：对字典中的密码进行变换（如大小写转换、添加数字等）

---

### JtR 攻击模式

John the Ripper 支持多种攻击模式：

#### 1. 字典攻击（Dictionary Attack）

```bash
# 基本字典攻击
john --wordlist=passwords.txt hashes.txt

# 使用 Kali 自带字典
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

**优点**：速度快，如果密码在字典中，可迅速破解。  
**缺点**：如果密码不在字典中，无法破解。

---

#### 2. 暴力破解（Brute-Force Attack）

```bash
# 暴力破解（枚举所有组合）
john --incremental hashes.txt

# 指定字符集
john --incremental=Lower    # 仅小写字母
john --incremental=Upper    # 仅大写字母
john --incremental=Digits   # 仅数字
john --incremental=Alpha    # 大小写字母
john --incremental=Alnum    # 大小写字母+数字
```

**优点**：理论上可以破解任何密码。  
**缺点**：耗时极长（密码长度每增加1位，时间指数级增长）。

---

#### 3. 规则攻击（Rule-Based Attack）

```bash
# 使用规则对字典进行变换
john --wordlist=passwords.txt --rules hashes.txt

# 使用特定规则集
john --wordlist=passwords.txt --rules=Jumbo hashes.txt

# 查看可用规则
john --list=rules
```

**规则示例**：

| 原始密码 | 规则 | 变换后 |
|---------|------|--------|
| password | `:`: 原样 | password |
| password | `c`: 首字母大写 | Password |
| password | `C`: 首字母小写 | password |
| password | `r`: 反转 | drowssap |
| password | `$1`: 末尾加1 | password1 |
| password | `^A`: 开头加A | Apassword |
| password | `l`: 全部小写 | password |
| password | `u`: 全部大写 | PASSWORD |

**优点**：结合字典攻击的速度和暴力破解的覆盖面。  
**缺点**：需要理解规则语法，配置复杂。

---

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己创建或授权的文件**进行破解测试
> - 未经授权破解他人加密文件属于**违法行为**
> - 建议在本地虚拟机或 Docker 环境中练习
> - 不要将 JtR 用于非法目的

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 John the Ripper 是否已安装
john --version

# 如未安装，执行：
sudo apt install -y john

# 或者安装增强版（Jumbo version，推荐）
sudo apt install -y john-jumbo

# 检查 zip2john 工具
which zip2john

# 如未安装
sudo apt install -y john-jumbo
```

### 安装 John the Ripper Jumbo（最新版）

```bash
# 从 GitHub 编译安装最新版
sudo apt install -y build-essential libssl-dev zlib1g-dev \
                   yasm pkg-config libgmp-dev libpcap-dev \
                   libbz2-dev

cd /opt
sudo git clone https://github.com/openwall/john.git
cd john/src
sudo ./configure && sudo make -j$(nproc)

# 安装完成后，john 位于 /opt/john/run/john
```

---

## 👨‍💻 实验步骤

### Step 1：创建加密 ZIP 文件用于测试

首先，我们需要创建一个带密码的 ZIP 文件来模拟真实场景。

```bash
# 创建测试目录
mkdir -p ~/zip_crack_lab
cd ~/zip_crack_lab

# 创建测试文件
echo "这是一份机密文档！" > secret.txt
echo "账号：admin" >> secret.txt
echo "密码：SuperSecret123" >> secret.txt

# 创建加密 ZIP 文件（使用 ZIP 2.0 加密，旧算法）
zip -e -P password123 secret.zip secret.txt

# 验证 ZIP 文件
unzip -l secret.zip

# 尝试解压（需要输入密码）
unzip secret.zip
# 输入密码：password123
```

**注意**：

- `zip -e` 使用传统的 ZIP 2.0 加密（较弱，容易被破解）
- 现代 ZIP 工具支持 AES-256 加密（更强，破解难度大）

---

### Step 2：使用 zip2john 提取哈希

John the Ripper 不能直接破解 ZIP 文件，需要先将 ZIP 文件转换为 JtR 可识别的哈希格式。

```bash
# 使用 zip2john 提取哈希
zip2john secret.zip > secret_hash.txt

# 查看提取的哈希
cat secret_hash.txt

# 输出示例：
# secret.zip/secret.txt:$zip2$*0*3*0*...（一长串哈希值）
```

**哈希格式说明**：

```
$zip2$*0*3*0*...
  ↑    ↑  ↑  ↑
  |    |  |  |-- 其他元数据
  |    |  |-- 压缩级别
  |    |-- 加密类型（0=ZIP 2.0，99=AES-256）
  |-- 哈希类型标识
```

---

### Step 3：使用 JtR 破解 ZIP 密码（字典攻击）

#### 准备密码字典

```bash
# 创建小字典用于测试
cat > passwords.txt << EOF
123456
password
password123
admin
admin123
secret
SuperSecret123
qwerty
letmein
1234
12345
123456789
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
EOF

# 或者使用 Kali 自带的 rockyou.txt
# sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```

#### 执行字典攻击

```bash
# 基本字典攻击
john --wordlist=passwords.txt secret_hash.txt

# 显示详细过程
john --wordlist=passwords.txt --verbosity=5 secret_hash.txt

# 使用 Kali 自带字典（耗时较长）
john --wordlist=/usr/share/wordlists/rockyou.txt secret_hash.txt
```

**预期输出示例**：

```
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x AES])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
password123       (secret.zip/secret.txt)
1g 0:00:00:00 DONE (2026-05-30 12:00) 2.500g/s 30400p/s 30400c/s 30400C/s 123456..password123
Use the --show --format=ZIP option to display all of the cracked passwords reliably
Session completed
```

**结果解读**：

- `password123`：破解得到的密码
- `secret.zip/secret.txt`：对应的文件
- `2.500g/s`：每秒尝试 2.500 个密码（g/s = guesses per second）
- `30400p/s`：每秒处理 30400 个候选密码（p/s = passwords per second）

---

### Step 4：查看破解结果

```bash
# 查看已破解的密码
john --show secret_hash.txt

# 输出示例：
# secret.zip/secret.txt:password123:...:secret.txt:secret.zip::secret.zip

# 只显示密码
john --show --format=ZIP secret_hash.txt | cut -d: -f2

# 保存结果到文件
john --show secret_hash.txt > cracked_passwords.txt
```

---

### Step 5：使用 JtR 进行暴力破解

如果字典攻击失败，可以尝试暴力破解。

```bash
# 暴力破解（枚举所有组合）
john --incremental secret_hash.txt

# 指定字符集和长度
# 编辑 John 配置文件
sudo nano /etc/john/john.conf

# 添加自定义增量模式
# [Incremental:Custom]
# File = $JOHN/custom.chr
# MinLen = 1
# MaxLen = 8
# CharCount = 95

# 使用自定义增量模式
john --incremental=Custom secret_hash.txt
```

**注意**：暴力破解非常耗时。对于 ZIP 2.0 加密（较弱），可以尝试；对于 AES-256 加密，基本不可行。

---

### Step 6：使用规则攻击

规则攻击是对字典攻击的增强，通过对字典中的密码进行变换来覆盖更多可能性。

```bash
# 使用默认规则
john --wordlist=passwords.txt --rules secret_hash.txt

# 使用 Jumbo 规则集（更激进）
john --wordlist=passwords.txt --rules=Jumbo secret_hash.txt

# 查看所有可用规则
john --list=rules

# 使用特定规则
john --wordlist=passwords.txt --rules=Single secret_hash.txt
john --wordlist=passwords.txt --rules=Wordlist secret_hash.txt
```

**常用规则解释**：

| 规则 | 说明 | 示例 |
|------|------|------|
| `:` | 原样输出 | password → password |
| `c` | 首字母大写 | password → Password |
| `C` | 首字母小写 | PASSWORD → pASSWORD |
| `l` | 全部小写 | PASSWORD → password |
| `u` | 全部大写 | password → PASSWORD |
| `r` | 反转 | password → drowssap |
| `$1` | 末尾加1 | password → password1 |
| `$!` | 末尾加! | password → password! |
| `^A` | 开头加A | password → Apassword |
| `d` | 重复 | password → passwordpassword |
| `p` | 反转并重复 | password → passworddrowssap |

---

### Step 7：使用 GPU 加速（可选）

如果你有 NVIDIA GPU，可以使用 Hashcat（另一个强大的离线破解工具）进行 GPU 加速。

```bash
# 安装 Hashcat
sudo apt install -y hashcat

# 查看 GPU 设备
hashcat -I

# 使用 Hashcat 破解 ZIP 哈希
# 首先，需要识别哈希类型（ZIP 2.0 = 13600，ZIP AES = 18200）
hashcat -m 13600 -a 0 secret_hash.txt passwords.txt

# 参数说明：
# -m 13600   → 哈希类型（13600 = WinZip）
# -a 0       → 攻击模式（0 = 字典攻击）
# secret_hash.txt → 哈希文件
# passwords.txt   → 字典文件

# 使用暴力破解
hashcat -m 13600 -a 3 secret_hash.txt ?a?a?a?a?a?a?a?a

# ?a = 所有可打印字符
# ?l = 小写字母
# ?u = 大写字母
# ?d = 数字
# ?s = 特殊字符
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] John the Ripper 和 Hydra 有什么区别？
- [ ] 如何使用 zip2john 提取 ZIP 文件的哈希？
- [ ] JtR 支持哪几种攻击模式？各有什么优缺点？
- [ ] 如何查看 JtR 的破解结果？
- [ ] 规则攻击中的 `c`、`$1`、`r` 分别代表什么？

---

## 💡 进阶挑战

### 挑战1：破解强加密的 ZIP 文件（AES-256）

```bash
# 创建 AES-256 加密的 ZIP 文件（使用 7z）
sudo apt install -y p7zip-full

# 使用 7z 创建 AES-256 加密的 ZIP
7z a -tzip -mem=AES256 -pSecretPass123 secure.zip secret.txt

# 尝试用 zip2john 提取哈希
zip2john secure.zip > secure_hash.txt

# 使用 JtR 破解（难度很大，需要强字典或规则）
john --wordlist=/usr/share/wordlists/rockyou.txt secure_hash.txt
```

**注意**：AES-256 加密的 ZIP 文件非常难以破解，建议使用长密码（≥12位）+ 密码管理器。

---

### 挑战2：破解 Linux /etc/shadow 文件

```bash
# 模拟获取 Linux 密码哈希
# 实际场景中，需要 root 权限读取 /etc/shadow

# 创建测试哈希（模拟 /etc/shadow 中的条目）
cat > shadow_hash.txt << EOF
root:$6$saltvalue$hashedpassword:18000:0:99999:7:::
EOF

# 使用 unshadow 合并 /etc/passwd 和 /etc/shadow
# unshadow /etc/passwd /etc/shadow > combined.txt

# 使用 JtR 破解
john --wordlist=passwords.txt shadow_hash.txt

# 指定哈希格式（可选）
john --wordlist=passwords.txt --format=crypt shadow_hash.txt
```

**常见 Linux 哈希类型**：

| 哈希标识 | 算法 | 说明 |
|---------|------|------|
| `$1$` | MD5 | 较弱，容易被破解 |
| `$5$` | SHA-256 | 中等强度 |
| `$6$` | SHA-512 | 较强（现代 Linux 默认） |
| `yescrypt` | yescrypt | 非常强（最新发行版） |

---

### 挑战3：破解 Windows NTLM 哈希

```bash
# 模拟获取 Windows 密码哈希
# 实际场景中，可以使用 Mimikatz 等工具从内存提取

# 创建测试 NTLM 哈希
cat > ntlm_hash.txt << EOF
Administrator:500:AAD3B435B51404EEAAD3B435B51404EE:32ED87BDB5FDC5E9CBA88547376818D4:::
EOF

# 使用 JtR 破解 NTLM 哈希
john --wordlist=passwords.txt --format=nt ntlm_hash.txt

# 查看结果
john --show --format=nt ntlm_hash.txt
```

**Windows 哈希类型**：

| 类型 | 说明 |
|------|------|
| **LM** | LAN Manager（非常弱，已废弃） |
| **NTLM** | NT LAN Manager（Windows XP/2003 及更早版本） |
| **NTLMv2** | 增强版 NTLM（Windows Vista/2008 及以后） |

---

### 挑战4：使用自定义规则

```bash
# 创建自定义规则文件
cat > custom_rules.conf << EOF
[List.Rules:CustomRule]
# 原样
:
# 首字母大写
c
# 末尾加数字 0-9
$[0-9]
# 末尾加年份
$2 $0 $2 $4
# 重复一次
d
# 反转
r
EOF

# 使用自定义规则
john --wordlist=passwords.txt --rules=CustomRule --config=custom_rules.conf secret_hash.txt
```

---

### 挑战5：编写自动化破解脚本

```bash
# 创建自动化破解脚本
cat > auto_jtr.sh << 'EOF'
#!/bin/bash

HASH_FILE=$1
WORDLIST=$2

echo "[*] 开始破解: $HASH_FILE"
echo "[*] 使用字典: $WORDLIST"

# 1. 字典攻击
echo "[*] 阶段1：字典攻击..."
john --wordlist=$WORDLIST $HASH_FILE
if john --show $HASH_FILE | grep -q ":"; then
  echo "[+] 字典攻击成功！"
  john --show $HASH_FILE
  exit 0
fi

# 2. 规则攻击
echo "[*] 阶段2：规则攻击..."
john --wordlist=$WORDLIST --rules $HASH_FILE
if john --show $HASH_FILE | grep -q ":"; then
  echo "[+] 规则攻击成功！"
  john --show $HASH_FILE
  exit 0
fi

# 3. 暴力破解（仅尝试短密码）
echo "[*] 阶段3：暴力破解（短密码）..."
john --incremental=Lower --max-run-time=300 $HASH_FILE  # 最多运行5分钟
if john --show $HASH_FILE | grep -q ":"; then
  echo "[+] 暴力破解成功！"
  john --show $HASH_FILE
  exit 0
fi

echo "[-] 未能破解密码"
EOF

# 赋予执行权限
chmod +x auto_jtr.sh

# 使用脚本
./auto_jtr.sh secret_hash.txt passwords.txt
```

---

## 🛡️ 防御措施（如何防范离线密码破解）

了解攻击方法后，用户和系统管理员应采取以下防御措施：

### 1. 使用强加密算法

#### ZIP 文件加密

```bash
# 使用 7-Zip 创建 AES-256 加密的 ZIP（推荐）
7z a -tzip -mem=AES256 -pStrongPassword123! secure.zip secret.txt

# 或者使用 7z 格式（默认使用 AES-256）
7z a -pStrongPassword123! -mhe=on secure.7z secret.txt
# -mhe=on  → 加密文件头（隐藏文件名）
```

**加密算法对比**：

| 算法 | 强度 | 破解难度 | 说明 |
|------|------|---------|------|
| ZIP 2.0（传统） | 弱 | 容易 | 使用 PKZIP 算法，易被破解 |
| AES-128 | 中等 | 困难 | 较好，但不如 AES-256 |
| **AES-256** | **强** | **非常困难** | **推荐使用** |

---

### 2. 使用长密码（≥12位）

**密码长度 vs 破解时间**（假设使用 AES-256，攻击速度 10^9 次/秒）：

| 密码长度 | 字符集 | 组合数 | 预估破解时间 |
|---------|--------|--------|-------------|
| 6 位 | 大小写+数字+符号（80字符） | 80^6 ≈ 2.62×10^11 | 262 秒（4.4分钟） |
| 8 位 | 同上 | 80^8 ≈ 1.68×10^15 | 19.4 天 |
| 10 位 | 同上 | 80^10 ≈ 1.07×10^19 | 340 千年 |
| **12 位** | 同上 | **80^12 ≈ 6.87×10^22** | **2.18 百万年** |

**结论**：密码长度 ≥12 位，即使使用弱字典，也很难被暴力破解。

---

### 3. 使用密码管理器

**推荐工具**：

- **KeePassXC**（开源，本地存储）
  ```bash
  sudo apt install -y keepassxc
  ```

- **Bitwarden**（开源，支持自托管）
  ```bash
  sudo snap install bitwarden
  ```

- **命令行密码管理器**：
  ```bash
  # 使用 pass（标准 Unix 密码管理器）
  sudo apt install -y pass
  
  # 初始化（使用 GPG 密钥）
  gpg --gen-key
  pass init <GPG_KEY_ID>
  
  # 插入密码
  pass insert secret_file
  
  # 读取密码
  pass secret_file
  ```

---

### 4. 使用盐值（Salt）和慢哈希

**盐值（Salt）**：在密码哈希前添加随机字符串，防止彩虹表攻击。

**慢哈希（Slow Hash）**：故意降低哈希计算速度，增加暴力破解难度。

#### 推荐的慢哈希算法

| 算法 | 说明 | 适用场景 |
|------|------|---------|
| **bcrypt** | 可调节成本因子 | Web 应用密码存储 |
| **scrypt** | 内存密集型 | 加密货币、密码保护 |
| **Argon2** | 2015年密码哈希竞赛冠军 | 新系统推荐 |
| **PBKDF2** | 多次迭代 | 旧系统兼容 |

#### 示例：使用 bcrypt

```bash
# 使用 htpasswd 生成 bcrypt 哈希
sudo apt install -y apache2-utils

# 生成 bcrypt 哈希（成本因子 10）
htpasswd -nbB -C 10 admin StrongPassword123!

# 输出示例：
# admin:$2y$10$...（bcrypt 哈希）
```

---

### 5. 双因素认证（2FA）和硬件密钥

即使密码被破解，攻击者仍需第二个验证因素。

**推荐方案**：

- **TOTP**（Google Authenticator、Authy）
- **U2F/FIDO2 硬件密钥**（YubiKey）
- **生物识别**（指纹、面部识别）

---

### 6. 文件和磁盘加密最佳实践

#### 使用 VeraCrypt（开源磁盘加密）

```bash
# 安装 VeraCrypt
sudo apt install -y veracrypt

# 创建加密容器（使用 AES-256 + SHA-512）
veracrypt --create encrypted_volume.hc \
  --volume-type=Normal \
  --encryption=AES \
  --hash=SHA-512 \
  --filesystem=FAT \
  --password=StrongPassword123! \
  --size=100M \
  --random-source=/dev/urandom
```

#### 使用 GPG 加密文件

```bash
# 使用 GPG 对称加密（AES-256）
gpg --symmetric --cipher-algo AES256 secret.txt
# 输入密码：StrongPassword123!

# 解密
gpg secret.txt.gpg
```

---

## 📚 参考资源

- [John the Ripper 官方文档](https://www.openwall.com/john/doc/)
- [John the Ripper GitHub](https://github.com/openwall/john)
- [zip2john 使用指南](https://www.hackingarticles.in/comprehensive-guide-on-john-the-ripper/)
- [Hashcat 官方文档](https://hashcat.net/wiki/)
- [密码存储 cheat sheet（OWASP）](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [ZIP 文件格式规范](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)
- [7-Zip 官方文档](https://7-zip.org/)
- [KeePassXC 官方文档](https://keepassxc.org/docs/)
- [Argon2 算法论文](https://www.password-hashing.net/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：_______________
ZIP 文件名：_____________
加密算法：_______________
（ZIP 2.0 / AES-128 / AES-256）

提取的哈希：_____________
（粘贴 zip2john 输出的前50字符）

使用攻击模式：___________
  - 字典攻击：□
  - 暴力破解：□
  - 规则攻击：□

使用字典：_______________
字典大小（密码数量）：_____
优化参数：_______________
  - 规则集：_____________
  - 线程数：_____________
  - 其他：_______________

破解结果：_______________
  - 密码：_______________
  - 耗时：_______________
  - 速度（p/s）：________

遇到的问题：_____________
解决方案：_______________
防御措施测试：___________
  - ZIP 2.0 和 AES-256 的区别？
  - 推荐的最小密码长度？_____
  - 使用的密码管理器？_______
```

---

## 🔍 知识拓展

### 密码哈希识别

在实战中，经常需要识别哈希类型：

```bash
# 使用 hashid 工具
sudo apt install -y hashid

# 识别哈希类型
hashid '$6$salt$hash'
hashid -m '$6$salt$hash'  # 显示 Hashcat 模式编号

# 使用 hash-identifier（Python 脚本）
wget https://raw.githubusercontent.com/blackploit/hash-identifier/master/hash-id.py
python3 hash-id.py
```

**常见哈希特征**：

| 哈希示例 | 类型 | 长度 |
|---------|------|------|
| `5f4dcc3b5aa765d61d8327deb882cf99` | MD5 | 32 字符 |
| `5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8` | SHA-1 | 40 字符 |
| `$6$salt$hash` | SHA-512 | 变长 |
| `32ED87BDB5FDC5E9CBA88547376818D4` | NTLM | 32 字符 |

---

### 密码策略评估

企业应如何制定密码策略？

**NIST 密码指南（SP 800-63B）**：

1. **最小长度 8 位**（推荐 12 位）
2. **允许所有可打印字符**（包括空格和表情符号）
3. **检查密码是否常见**（与字典比对）
4. **不要强制定期更换密码**（除非有泄露证据）
5. **使用多因素认证**
6. **不要使用密码提示问题**（容易被社工）

---

### 量子计算对密码学的影响

**现状**：

- 量子计算机可以运行 **Shor 算法**，快速分解大整数（威胁 RSA、ECC）
- 但对称加密（AES）受影响较小，只需**加倍密钥长度**（AES-128 → AES-256）

**建议**：

- 长期使用：选择 **AES-256** 或 **ChaCha20**
- 后量子密码学：关注 **NIST 后量子标准化项目**

---

*最后更新：2026-05-30*
