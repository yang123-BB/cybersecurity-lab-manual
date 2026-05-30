# 🔒 实验13：使用 OpenSSL 加密文件

> **难度**：⭐⭐⭐ 中级  
> **预计时间**：50分钟  
> **工具**：OpenSSL、GPG

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 使用 OpenSSL 安全加密文件（AES-256-CBC）
- ✅ 理解密码加密与密钥加密的区别
- ✅ 创建和验证数字签名
- ✅ 实现安全文件传输流程
- ✅ 加密敏感数据的最佳实践

---

## 📖 背景知识

### 为什么需要文件加密？

在数字时代，数据安全至关重要：

```
需要加密的场景：

个人隐私：
├── 个人身份证件扫描件
├── 财务记录和银行信息
├── 私人日记和照片
└── 密码和账户信息

企业数据：
├── 客户隐私数据
├── 商业机密文档
├── 财务报表
└── 源代码和专利

传输安全：
├── 邮件附件
├── 云存储上传
├── USB 传输
└── 网络共享
```

### 加密方式对比

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 密码加密 | 简单易用 | 密码可能被猜测/破解 | 个人文件保护 |
| 密钥加密 | 安全性高 | 密钥管理复杂 | 企业级应用 |
| 混合加密 | 兼顾安全和便捷 | 实现较复杂 | 大文件加密 |
| 公钥加密 | 无需共享密钥 | 速度慢 | 小文件/密钥传输 |

### AES-256-CBC 详解

```
AES-256-CBC 工作原理：

AES（Advanced Encryption Standard）
├── 256位密钥长度 = 32字节
├── 块大小：128位 = 16字节
└── 分组加密，对数据分块处理

CBC（Cipher Block Chaining）模式
├── 每个明文块先与前一个密文块异或
├── 第一个块与初始化向量（IV）异或
├── 相同明文产生不同密文
└── 提供数据完整性保护

安全要求：
├── 使用随机生成的 IV
├── 密钥必须足够复杂
├── 建议使用 PBKDF2 派生密钥
└── 添加 HMAC 进行完整性验证
```

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 加密技术受法律监管，请遵守当地法规
> - 仅对自己拥有的文件进行加密操作
> - 加密不等于绝对安全，要结合其他安全措施
> - 不要使用加密隐藏非法内容

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 OpenSSL 版本（建议 1.1.1+）
openssl version

# 创建工作目录
mkdir -p ~/encrypt_lab/{files,keys,output}
cd ~/encrypt_lab

# 准备测试文件
echo "这是一份敏感文件，包含重要信息。" > files/sensitive.txt
echo "账户：admin，密码：SecretPass123" >> files/sensitive.txt
```

---

## 👨‍💻 实验步骤

### Step 1：基础密码加密

最简单的加密方式，适合快速保护文件：

```bash
# 加密文件
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 \
    -in files/sensitive.txt \
    -out output/sensitive.enc

# 输入密码两次
# enter aes-256-cbc encryption password:
# Verifying - enter aes-256-cbc encryption password:

# 查看加密结果
file output/sensitive.enc
# 输出：data

xxd output/sensitive.enc | head -5
# 文件头显示 "Salted__"
```

**参数详解：**

| 参数 | 说明 |
|------|------|
| `-aes-256-cbc` | 使用 AES-256-CBC 算法 |
| `-salt` | 添加随机盐值 |
| `-pbkdf2` | 使用 PBKDF2 密钥派生函数 |
| `-iter 100000` | PBKDF2 迭代次数（越高越安全） |
| `-in` | 输入文件 |
| `-out` | 输出文件 |

```bash
# 解密文件
openssl enc -aes-256-cbc -d -pbkdf2 -iter 100000 \
    -in output/sensitive.enc \
    -out output/sensitive_dec.txt

# 输入加密时使用的密码

# 验证解密结果
diff files/sensitive.txt output/sensitive_dec.txt
# 无输出表示文件相同

cat output/sensitive_dec.txt
```

---

### Step 2：密钥文件加密（更安全）

使用随机密钥文件，避免密码猜测风险：

```bash
# 生成随机密钥（256位 = 32字节）
openssl rand -out keys/secret.key 32

# 生成随机初始化向量（128位 = 16字节）
openssl rand -out keys/iv.bin 16

# 查看密钥内容（十六进制）
xxd keys/secret.key

# 使用密钥加密
openssl enc -aes-256-cbc \
    -K $(xxd -p keys/secret.key | tr -d '\n') \
    -iv $(xxd -p keys/iv.bin | tr -d '\n') \
    -in files/sensitive.txt \
    -out output/sensitive_key.enc

# 使用密钥解密
openssl enc -aes-256-cbc -d \
    -K $(xxd -p keys/secret.key | tr -d '\n') \
    -iv $(xxd -p keys/iv.bin | tr -d '\n') \
    -in output/sensitive_key.enc \
    -out output/sensitive_key_dec.txt

# 验证
cat output/sensitive_key_dec.txt
```

**密钥文件安全存储：**

```bash
# 将密钥文件加密存储
openssl enc -aes-256-cbc -salt -pbkdf2 \
    -in keys/secret.key \
    -out keys/secret.key.enc

# 删除原始密钥文件
shred -u keys/secret.key

# 需要使用时先解密密钥
openssl enc -aes-256-cbc -d -pbkdf2 \
    -in keys/secret.key.enc \
    -out keys/secret.key
```

---

### Step 3：密码 vs 密钥对比

```bash
# 创建测试文件
dd if=/dev/urandom of=files/test_10mb.bin bs=1M count=10

# 使用密码加密（速度测试）
time openssl enc -aes-256-cbc -pbkdf2 -iter 100000 \
    -pass pass:MyPassword123 \
    -in files/test_10mb.bin \
    -out output/test_pass.enc

# 使用密钥加密（速度测试）
time openssl enc -aes-256-cbc \
    -K $(openssl rand -hex 32) \
    -iv $(openssl rand -hex 16) \
    -in files/test_10mb.bin \
    -out output/test_key.enc

# 对比结果
ls -lh files/test_10mb.bin output/test_*.enc
```

**对比总结：**

| 特性 | 密码加密 | 密钥加密 |
|------|---------|---------|
| 安全性 | 依赖密码强度 | 随机密钥，更安全 |
| 便捷性 | 只需记住密码 | 需要安全存储密钥 |
| 速度 | 需要密钥派生，较慢 | 直接加密，较快 |
| 适用场景 | 个人使用 | 企业/高安全需求 |

---

### Step 4：数字签名与验证

数字签名确保文件来源可信且未被篡改：

```bash
# 生成 RSA 密钥对
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem

# 查看私钥
cat keys/private.pem

# 对文件签名
openssl dgst -sha256 -sign keys/private.pem \
    -out output/sensitive.sig \
    files/sensitive.txt

# 查看签名（二进制格式）
xxd output/sensitive.sig | head -5

# 验证签名
openssl dgst -sha256 -verify keys/public.pem \
    -signature output/sensitive.sig \
    files/sensitive.txt
# 输出：Verified OK

# 模拟篡改
echo "篡改内容" >> files/sensitive.txt

# 再次验证（会失败）
openssl dgst -sha256 -verify keys/public.pem \
    -signature output/sensitive.sig \
    files/sensitive.txt
# 输出：Verification Failure

# 恢复原文件
git checkout files/sensitive.txt 2>/dev/null || \
    echo "这是一份敏感文件，包含重要信息。" > files/sensitive.txt && \
    echo "账户：admin，密码：SecretPass123" >> files/sensitive.txt
```

---

### Step 5：加密 + 签名完整流程

结合加密和签名，提供机密性和完整性：

```bash
# 完整的安全文件处理流程

# 1. 生成签名
openssl dgst -sha256 -sign keys/private.pem \
    -out output/document.sig \
    files/sensitive.txt

# 2. 加密文件
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 \
    -in files/sensitive.txt \
    -out output/document.enc

# 3. 加密签名（可选，防止签名泄露信息）
openssl enc -aes-256-cbc -salt -pbkdf2 \
    -in output/document.sig \
    -out output/document.sig.enc

# 传输文件：document.enc, document.sig（或 document.sig.enc）

# 接收方解密验证流程

# 1. 解密文件
openssl enc -aes-256-cbc -d -pbkdf2 -iter 100000 \
    -in output/document.enc \
    -out output/document_dec.txt

# 2. 解密签名（如果加密了）
openssl enc -aes-256-cbc -d -pbkdf2 \
    -in output/document.sig.enc \
    -out output/document.sig

# 3. 验证签名
openssl dgst -sha256 -verify keys/public.pem \
    -signature output/document.sig \
    output/document_dec.txt
```

---

### Step 6：批量加密脚本

```bash
#!/bin/bash
# encrypt_folder.sh - 批量加密脚本

FOLDER=$1
PASSWORD=$2
OUTPUT="./encrypted_backup"

if [ -z "$FOLDER" ] || [ -z "$PASSWORD" ]; then
    echo "用法: $0 <文件夹> <密码>"
    exit 1
fi

# 创建输出目录
mkdir -p "$OUTPUT"

# 创建压缩包
tar -czf temp.tar.gz "$FOLDER"

# 加密压缩包
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 \
    -pass pass:"$PASSWORD" \
    -in temp.tar.gz \
    -out "$OUTPUT/backup_$(date +%Y%m%d_%H%M%S).enc"

# 清理临时文件
rm -f temp.tar.gz

echo "加密完成: $OUTPUT"
ls -lh "$OUTPUT"
```

**解密脚本：**

```bash
#!/bin/bash
# decrypt_backup.sh - 解密脚本

FILE=$1
PASSWORD=$2

if [ -z "$FILE" ] || [ -z "$PASSWORD" ]; then
    echo "用法: $0 <加密文件> <密码>"
    exit 1
fi

# 解密
openssl enc -aes-256-cbc -d -pbkdf2 -iter 100000 \
    -pass pass:"$PASSWORD" \
    -in "$FILE" \
    -out temp.tar.gz

# 解压
tar -xzf temp.tar.gz

# 清理
rm -f temp.tar.gz

echo "解密完成"
```

---

### Step 7：实战 - 加密敏感文件并安全传输

模拟真实场景：加密并发送敏感文档

```bash
# 场景：Alice 向 Bob 发送机密文件

# Alice 的操作
# 1. 准备文件
echo "机密项目计划书" > project_plan.txt
echo "预算：100万元" >> project_plan.txt
echo "截止日期：2024-12-31" >> project_plan.txt

# 2. 获取 Bob 的公钥（假设 Bob 已共享）
# Bob 生成密钥对
openssl genrsa -out bob_private.pem 2048
openssl rsa -in bob_private.pem -pubout -out bob_public.pem

# Alice 拿到 bob_public.pem

# 3. 生成随机 AES 密钥（用于加密大文件）
openssl rand -hex 32 > session_key.txt

# 4. 用 AES 密钥加密文件
openssl enc -aes-256-cbc \
    -K $(cat session_key.txt) \
    -iv $(openssl rand -hex 16) \
    -in project_plan.txt \
    -out project_plan.enc

# 5. 用 Bob 的公钥加密 AES 密钥
openssl pkeyutl -encrypt -pubin -inkey bob_public.pem \
    -in session_key.txt \
    -out session_key.enc

# 6. 删除明文密钥
rm -f session_key.txt project_plan.txt

# 发送给 Bob：project_plan.enc + session_key.enc

# Bob 的操作（接收方）
# 1. 解密会话密钥
openssl pkeyutl -decrypt -inkey bob_private.pem \
    -in session_key.enc \
    -out session_key_dec.txt

# 2. 用会话密钥解密文件
openssl enc -aes-256-cbc -d \
    -K $(cat session_key_dec.txt) \
    -iv $(openssl rand -hex 16) \
    -in project_plan.enc \
    -out project_plan_dec.txt

# 注意：IV 需要和加密时一致，实际应用中 IV 通常和密文一起传输

# 验证
cat project_plan_dec.txt
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] AES-256-CBC 中 256 和 CBC 分别代表什么？
- [ ] 密码加密和密钥加密各有什么优缺点？
- [ ] 数字签名的作用是什么？
- [ ] 为什么建议使用 PBKDF2 和高迭代次数？
- [ ] 混合加密的工作流程是什么？

---

## 💡 进阶挑战

### 挑战1：创建加密压缩备份工具

```bash
#!/bin/bash
# secure_backup.sh - 安全备份工具

SOURCE=$1
BACKUP_DIR="./backups"
PASS_FILE="$HOME/.backup_key"

# 首次运行创建密钥文件
if [ ! -f "$PASS_FILE" ]; then
    openssl rand -base64 32 > "$PASS_FILE"
    chmod 600 "$PASS_FILE"
    echo "已创建加密密钥: $PASS_FILE"
fi

# 创建备份
mkdir -p "$BACKUP_DIR"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
ARCHIVE="$BACKUP_DIR/backup_$TIMESTAMP.tar.gz.enc"

# 压缩并加密
tar -czf - "$SOURCE" | \
    openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 \
    -pass file:"$PASS_FILE" \
    -out "$ARCHIVE"

echo "备份创建: $ARCHIVE"
echo "大小: $(ls -lh "$ARCHIVE" | awk '{print $5}')"
```

### 挑战2：使用 GPG 加密

```bash
# GPG 是另一个强大的加密工具

# 安装
sudo apt install -y gnupg

# 生成密钥对
gpg --full-generate-key

# 加密文件
gpg --encrypt --recipient user@example.com file.txt

# 解密文件
gpg --decrypt file.txt.gpg > file.txt

# 对称加密（密码）
gpg --symmetric --cipher-algo AES256 file.txt
gpg --decrypt file.txt.gpg
```

### 挑战3：实现安全的密码管理器

```bash
#!/bin/bash
# simple_password_manager.sh

VAULT="./password_vault.enc"
MASTER_PASS=""

init_vault() {
    echo "[]" | openssl enc -aes-256-cbc -salt -pbkdf2 \
        -pass pass:"$MASTER_PASS" -out "$VAULT"
    echo "密码库已创建"
}

add_password() {
    local site=$1
    local user=$2
    local pass=$3
    
    # 解密
    local data=$(openssl enc -aes-256-cbc -d -pbkdf2 \
        -pass pass:"$MASTER_PASS" -in "$VAULT")
    
    # 添加新条目
    data=$(echo "$data" | jq ". + [{\"site\": \"$site\", \"user\": \"$user\", \"pass\": \"$pass\"}]")
    
    # 加密保存
    echo "$data" | openssl enc -aes-256-cbc -salt -pbkdf2 \
        -pass pass:"$MASTER_PASS" -out "$VAULT"
    
    echo "密码已保存"
}

get_password() {
    local site=$1
    openssl enc -aes-256-cbc -d -pbkdf2 \
        -pass pass:"$MASTER_PASS" -in "$VAULT" | \
        jq -r ".[] | select(.site==\"$site\") | .pass"
}

# 使用示例
# MASTER_PASS="my_secret_master_key" init_vault
# MASTER_PASS="my_secret_master_key" add_password "google.com" "user@gmail.com" "G00dP@ss!"
# MASTER_PASS="my_secret_master_key" get_password "google.com"
```

---

## 🛡️ 防御措施

### 加密最佳实践

| 实践 | 说明 |
|------|------|
| **强密码** | 至少16位，包含大小写、数字、符号 |
| **密钥管理** | 使用专业密钥管理系统或硬件安全模块 |
| **安全传输** | 加密文件和密钥分别传输 |
| **完整性验证** | 使用 HMAC 或数字签名 |
| **密钥轮换** | 定期更换加密密钥 |
| **安全删除** | 使用 shred 安全删除明文文件 |

### 常见错误

```bash
# ❌ 错误示例

# 1. 弱密码
openssl enc -aes-256-cbc -pass pass:123 -in file.txt -out file.enc

# 2. 命令行直接输入密码（会保存在历史记录）
history | grep openssl

# 3. 不使用 PBKDF2
openssl enc -aes-256-cbc -pass pass:password -in file.txt -out file.enc

# ✅ 正确做法

# 1. 使用强密码 + PBKDF2
openssl enc -aes-256-cbc -salt -pbkdf2 -iter 100000 \
    -in file.txt -out file.enc

# 2. 从文件读取密码
openssl enc -aes-256-cbc -salt -pbkdf2 \
    -pass file:password.txt \
    -in file.txt -out file.enc

# 3. 使用密钥文件
openssl enc -aes-256-cbc \
    -K $(cat key.hex) -iv $(cat iv.hex) \
    -in file.txt -out file.enc
```

---

## 📚 参考资源

- [OpenSSL 官方文档](https://www.openssl.org/docs/)
- [NIST 加密指南](https://csrc.nist.gov/publications/detail/sp/800-175b/rev-1/final)
- [OWASP 加密最佳实践](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [GPG 手册](https://www.gnupg.org/documentation/manuals/gnupg/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
加密的文件类型：
使用的加密方式：
遇到的挑战：
解决方案：
安全注意事项：
```

---

*最后更新：2026-05-30*
