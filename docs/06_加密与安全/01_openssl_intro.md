# 🔐 实验05：OpenSSL 加密入门

> **难度**：⭐⭐ 入门级  
> **预计时间**：45分钟  
> **工具**：OpenSSL

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 OpenSSL 的基本概念和工作原理
- ✅ 掌握对称加密（AES）的使用方法
- ✅ 掌握非对称加密（RSA）的使用方法
- ✅ 使用 OpenSSL 计算文件哈希值（MD5/SHA）
- ✅ 生成和管理密钥对

---

## 📖 背景知识

### 什么是 OpenSSL？

**OpenSSL** 是一个开源的加密工具库，提供了丰富的加密算法实现和 SSL/TLS 协议支持。它是互联网安全基础设施的核心组件之一，广泛应用于：

- Web 服务器（HTTPS）
- 邮件服务器（SMTPS、IMAPS）
- VPN 连接
- 代码签名
- 文件加密

### 加密算法分类

```
加密算法分类：

1. 对称加密（Symmetric Encryption）
   ├── AES（Advanced Encryption Standard）
   ├── DES/3DES
   ├── RC4
   └── Blowfish
   特点：加密和解密使用同一把密钥，速度快

2. 非对称加密（Asymmetric Encryption）
   ├── RSA
   ├── DSA
   ├── ECC（椭圆曲线）
   └── ElGamal
   特点：公钥加密、私钥解密，安全性高但速度慢

3. 哈希算法（Hash）
   ├── MD5（已不安全）
   ├── SHA-1（已不安全）
   ├── SHA-256/SHA-512
   └── BLAKE2
   特点：单向不可逆，用于数据完整性验证
```

### 加密模式说明

| 模式 | 说明 | 特点 |
|------|------|------|
| ECB | 电子密码本模式 | 相同明文产生相同密文，不安全 |
| CBC | 密码块链接模式 | 需要初始化向量（IV），安全 |
| CFB | 密码反馈模式 | 支持流式加密 |
| OFB | 输出反馈模式 | 类似流密码 |
| GCM | 伽罗瓦计数器模式 | 支持认证加密，高效安全 |

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 加密技术受各国法律监管，请遵守当地法规
> - 不要使用弱加密保护敏感数据
> - 仅对自己拥有的文件进行加密操作
> - 禁止使用加密技术隐藏非法内容

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 OpenSSL 是否已安装
openssl version

# 预期输出：
# OpenSSL 1.1.1f  31 Mar 2020

# 查看支持的加密算法
openssl enc -list

# 创建工作目录
mkdir -p ~/openssl_lab
cd ~/openssl_lab
```

---

## 👨‍💻 实验步骤

### Step 1：OpenSSL 基础操作

```bash
# 查看 OpenSSL 版本和编译选项
openssl version -a

# 查看帮助
openssl help
openssl enc -help

# 查看支持的密码套件
openssl ciphers -v
```

---

### Step 2：哈希计算（MD5/SHA）

哈希函数可以将任意长度的数据转换为固定长度的摘要，常用于：

- 文件完整性验证
- 密码存储
- 数字签名

```bash
# 创建测试文件
echo "Hello, OpenSSL!" > plaintext.txt

# 计算 MD5 哈希（不推荐用于安全场景）
openssl md5 plaintext.txt
# 输出：MD5(plaintext.txt)= d41d8cd98f00b204e9800998ecf8427e

# 直接计算字符串的 MD5
echo -n "Hello" | openssl md5
# 输出：(stdin)= 8b1a9953c4611296a827abf8c47804d7

# 计算 SHA-256 哈希（推荐）
openssl sha256 plaintext.txt
# 输出：SHA256(plaintext.txt)= ...

# 计算 SHA-512 哈希
openssl sha512 plaintext.txt

# 使用 dgst 命令（更通用）
openssl dgst -sha256 plaintext.txt
openssl dgst -sha512 -out hash.txt plaintext.txt
cat hash.txt
```

**验证文件完整性示例：**

```bash
# 创建文件并计算哈希
echo "重要文件内容" > important.txt
openssl sha256 important.txt > important.sha256

# 修改文件后重新计算
echo "被篡改的内容" > important.txt
openssl sha256 important.txt

# 对比哈希值
diff <(openssl sha256 important.txt) <(cat important.sha256)
# 如果哈希不同，说明文件已被修改
```

---

### Step 3：对称加密（AES）

对称加密使用相同的密钥进行加密和解密，适合大量数据的加密。

```bash
# 创建测试文件
echo "这是一段敏感信息，需要加密保护。" > secret.txt

# 使用 AES-256-CBC 加密（需要输入密码）
openssl enc -aes-256-cbc -salt -in secret.txt -out secret.enc

# 过程会提示输入密码两次：
# enter aes-256-cbc encryption password:
# Verifying - enter aes-256-cbc encryption password:

# 查看加密后的文件（乱码）
cat secret.enc

# 解密文件
openssl enc -aes-256-cbc -d -in secret.enc -out secret_decrypted.txt

# 验证解密结果
diff secret.txt secret_decrypted.txt
cat secret_decrypted.txt
```

**使用密钥文件加密（更安全）：**

```bash
# 生成随机密钥（256位 = 32字节）
openssl rand -hex 32 > key.bin
cat key.bin

# 生成随机初始化向量（128位 = 16字节）
openssl rand -hex 16 > iv.bin
cat iv.bin

# 使用密钥文件加密
openssl enc -aes-256-cbc \
    -K $(cat key.bin) \
    -iv $(cat iv.bin) \
    -in secret.txt \
    -out secret_key.enc

# 使用密钥文件解密
openssl enc -aes-256-cbc -d \
    -K $(cat key.bin) \
    -iv $(cat iv.bin) \
    -in secret_key.enc \
    -out secret_key_dec.txt

# 验证
cat secret_key_dec.txt
```

**常用加密参数说明：**

| 参数 | 说明 |
|------|------|
| `-aes-256-cbc` | 使用 AES-256-CBC 算法 |
| `-salt` | 添加随机盐值（增强安全性） |
| `-in` | 输入文件 |
| `-out` | 输出文件 |
| `-d` | 解密模式 |
| `-k` | 直接指定密码（不推荐，会显示在历史记录中） |
| `-K` | 指定密钥（十六进制） |
| `-iv` | 指定初始化向量 |
| `-pbkdf2` | 使用 PBKDF2 密钥派生（推荐） |
| `-iter` | PBKDF2 迭代次数 |

---

### Step 4：非对称加密（RSA）

RSA 是最常用的非对称加密算法，使用公钥加密、私钥解密。

```bash
# 生成私钥（2048位）
openssl genrsa -out private.pem 2048

# 查看私钥内容
cat private.pem

# 从私钥提取公钥
openssl rsa -in private.pem -pubout -out public.pem

# 查看公钥内容
cat public.pem

# 查看私钥详细信息
openssl rsa -in private.pem -text -noout

# 查看公钥详细信息
openssl rsa -in public.pem -pubin -text -noout
```

**使用 RSA 加密和解密：**

```bash
# 创建测试文件
echo "RSA 加密测试数据" > rsa_test.txt

# 使用公钥加密
openssl rsautl -encrypt -inkey public.pem -pubin -in rsa_test.txt -out rsa_test.enc

# 或者使用 pkeyutl（推荐，新版本）
openssl pkeyutl -encrypt -pubin -inkey public.pem -in rsa_test.txt -out rsa_test.enc

# 使用私钥解密
openssl rsautl -decrypt -inkey private.pem -in rsa_test.enc -out rsa_test_dec.txt

# 或使用 pkeyutl
openssl pkeyutl -decrypt -inkey private.pem -in rsa_test.enc -out rsa_test_dec.txt

# 验证解密结果
cat rsa_test_dec.txt
```

**注意**：RSA 只能加密少量数据（密钥长度 - 填充长度）。对于大文件，通常使用混合加密：

1. 生成随机对称密钥
2. 用对称密钥加密文件
3. 用 RSA 公钥加密对称密钥

---

### Step 5：数字签名

数字签名用于验证数据的来源和完整性：

```bash
# 创建需要签名的文件
echo "这是一份需要签名的文档" > document.txt

# 使用私钥生成签名
openssl dgst -sha256 -sign private.pem -out signature.bin document.txt

# 查看签名文件（二进制）
xxd signature.bin | head -5

# 验证签名
openssl dgst -sha256 -verify public.pem -signature signature.bin document.txt
# 输出：Verified OK

# 修改文档后验证（会失败）
echo "文档已被修改" > document.txt
openssl dgst -sha256 -verify public.pem -signature signature.bin document.txt
# 输出：Verification failure
```

---

### Step 6：证书生成

```bash
# 生成自签名证书
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes

# 按提示输入证书信息：
# Country Name (2 letter code) [AU]:CN
# State or Province Name (full name) [Some-State]:Beijing
# Locality Name (eg, city) []:Beijing
# Organization Name (eg, company) [Internet Widgits Pty Ltd]:Test Company
# Common Name (e.g. server FQDN or YOUR name) []:localhost

# 查看证书内容
openssl x509 -in cert.pem -text -noout

# 验证证书和密钥是否匹配
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa -noout -modulus -in key.pem | openssl md5
# 两个 MD5 值应该相同
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] 对称加密和非对称加密有什么区别？
- [ ] 为什么 MD5 不再推荐用于安全场景？
- [ ] AES-256-CBC 中的 CBC 是什么意思？
- [ ] RSA 加密有什么数据长度限制？
- [ ] 数字签名的作用是什么？

---

## 💡 进阶挑战

### 挑战1：加密脚本编写

编写一个 Shell 脚本，实现文件的加密和解密：

```bash
#!/bin/bash
# encrypt.sh - 文件加密脚本

if [ $# -lt 2 ]; then
    echo "用法: $0 <encrypt|decrypt> <文件名>"
    exit 1
fi

ACTION=$1
FILE=$2

if [ "$ACTION" = "encrypt" ]; then
    openssl enc -aes-256-cbc -salt -pbkdf2 -in "$FILE" -out "${FILE}.enc"
    echo "已加密: ${FILE}.enc"
elif [ "$ACTION" = "decrypt" ]; then
    openssl enc -aes-256-cbc -d -pbkdf2 -in "$FILE" -out "${FILE%.enc}"
    echo "已解密: ${FILE%.enc}"
else
    echo "未知操作: $ACTION"
    exit 1
fi
```

### 挑战2：混合加密实现

实现 RSA + AES 混合加密：

```bash
# 生成随机 AES 密钥
KEY=$(openssl rand -hex 32)
IV=$(openssl rand -hex 16)

# 加密大文件
openssl enc -aes-256-cbc -K "$KEY" -iv "$IV" -in large_file.zip -out large_file.enc

# 用 RSA 加密 AES 密钥
echo -n "$KEY" | openssl pkeyutl -encrypt -pubin -inkey public.pem -out key.enc
echo -n "$IV" | openssl pkeyutl -encrypt -pubin -inkey public.pem -out iv.enc

# 解密流程（接收方）
KEY=$(openssl pkeyutl -decrypt -inkey private.pem -in key.enc)
IV=$(openssl pkeyutl -decrypt -inkey private.pem -in iv.enc)
openssl enc -aes-256-cbc -d -K "$KEY" -iv "$IV" -in large_file.enc -out large_file.zip
```

### 挑战3：创建自己的 CA

```bash
# 创建 CA 目录结构
mkdir -p demoCA/{private,newcerts}
touch demoCA/index.txt
echo 1000 > demoCA/serial

# 生成 CA 私钥和证书
openssl genrsa -out demoCA/private/cakey.pem 2048
openssl req -new -x509 -key demoCA/private/cakey.pem -out demoCA/cacert.pem -days 3650

# 生成用户证书请求
openssl genrsa -out userkey.pem 2048
openssl req -new -key userkey.pem -out userreq.csr

# 使用 CA 签发用户证书
openssl ca -in userreq.csr -out usercert.pem -days 365
```

---

## 🛡️ 防御措施

### 加密最佳实践

| 措施 | 说明 |
|------|------|
| **使用强算法** | AES-256、RSA-2048+、SHA-256+ |
| **密钥管理** | 使用密钥管理系统（KMS）或硬件安全模块（HSM） |
| **密钥轮换** | 定期更换密钥，降低泄露风险 |
| **避免硬编码** | 不要在代码中硬编码密钥和密码 |
| **使用 PBKDF2** | 密码派生使用高迭代次数（≥10000） |

### 常见错误

```bash
# ❌ 错误：在命令行直接输入密码（会保存在历史记录中）
openssl enc -aes-256-cbc -k "mypassword" -in file.txt -out file.enc

# ✅ 正确：让 OpenSSL 提示输入密码
openssl enc -aes-256-cbc -in file.txt -out file.enc

# ❌ 错误：使用 ECB 模式
openssl enc -aes-256-ecb -in file.txt -out file.enc

# ✅ 正确：使用 CBC 或 GCM 模式
openssl enc -aes-256-cbc -pbkdf2 -in file.txt -out file.enc
```

---

## 📚 参考资源

- [OpenSSL 官方文档](https://www.openssl.org/docs/)
- [OpenSSL 命令速查表](https://www.sslshopper.com/article-most-common-openssl-commands.html)
- [密码学入门教程](https://cryptobook.nakov.com/)
- [NIST 加密标准](https://csrc.nist.gov/publications/detail/sp/800-175b/rev-1/final)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
使用的 OpenSSL 版本：
加密算法尝试：
遇到的问题：
解决方案：
学到的知识点：
```

---

*最后更新：2026-05-30*
