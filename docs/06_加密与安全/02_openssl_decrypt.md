# 🔓 实验06：解密绝密文件

> **难度**：⭐⭐⭐ 中级  
> **预计时间**：60分钟  
> **工具**：OpenSSL、Base64、Python

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解常见的编码和加密方式
- ✅ 掌握 Base64 编码的识别和解码
- ✅ 破解弱加密密钥
- ✅ 使用多种工具解密文件
- ✅ 培养密码分析和破解思维

---

## 📖 背景知识

### 加密 vs 编码

很多人容易混淆**编码**和**加密**：

```
编码（Encoding）：
- 目的：数据表示转换
- 特点：无密钥，任何人都可以解码
- 例子：Base64、Hex、URL编码、UTF-8

加密（Encryption）：
- 目的：数据保密
- 特点：需要密钥才能解密
- 例子：AES、RSA、DES
```

### 常见编码识别

| 编码类型 | 特征 | 示例 |
|---------|------|------|
| Base64 | 字母大小写+数字+/和=结尾 | `SGVsbG8gV29ybGQ=` |
| Hex | 纯数字0-9和字母a-f | `48656c6c6f` |
| URL编码 | %+两位十六进制 | `%E4%BD%A0%E5%A5%BD` |
| 摩尔斯电码 | 点和划 | `.... . .-.. .-.. ---` |
| ROT13 | 字母偏移13位 | `Uryyb Jbeyq` → `Hello World` |

### 加密分析思路

```
解密分析流程：

1. 识别编码/加密类型
   ├── 查看文件特征
   ├── 使用识别工具
   └── 尝试常见格式

2. 获取密钥信息
   ├── 字典攻击
   ├── 暴力破解
   ├── 社会工程学线索
   └── 侧信道信息

3. 尝试解密
   └── 使用正确的工具和参数

4. 验证结果
   └── 检查解密内容是否合理
```

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能解密**自己拥有或授权的文件**
> - 未经授权破解他人加密数据属于**违法行为**
> - 学习目的是为了更好地保护数据安全
> - 所有练习请在本地实验环境进行

---

## 🔧 环境要求

### 系统要求

```bash
# 检查工具
openssl version
base64 --version
python3 --version

# 创建工作目录
mkdir -p ~/decrypt_lab
cd ~/decrypt_lab

# 安装常用工具
sudo apt install -y xxd file
```

### 准备练习文件

```bash
# 创建练习用的加密文件（后面会用到）
echo "秘密信息：Agent007的身份已确认" > plain1.txt

# Base64 编码
base64 plain1.txt > encoded1.txt

# AES 加密（密码：secret123）
openssl enc -aes-256-cbc -salt -pbkdf2 -in plain1.txt -out encrypted1.enc -pass pass:secret123

# 创建弱密码加密文件
openssl enc -aes-256-cbc -salt -in plain1.txt -out weak.enc -pass pass:123

# 创建多层加密
echo "多层加密测试" | base64 | openssl enc -aes-128-cbc -pass pass:password -out multi.enc
```

---

## 👨‍💻 实验步骤

### Step 1：Base64 编码识别与解码

```bash
# 查看编码文件
cat encoded1.txt
# 输出：6Z+z5a+85Z+f77yaQWdlbnQwMDfnmoTkuIDnu7boiIzor7TliLDop4blvZM=

# Base64 特征识别：
# 1. 包含大小写字母、数字、+、/
# 2. 末尾可能有 = 填充符
# 3. 长度是4的倍数

# 解码
base64 -d encoded1.txt
# 输出：秘密信息：Agent007的身份已确认

# 解码并保存
base64 -d encoded1.txt > decoded1.txt
cat decoded1.txt

# 使用 OpenSSL 解码
openssl enc -base64 -d -in encoded1.txt
```

**识别 Base64 的技巧：**

```bash
# 使用 file 命令查看文件类型
file encoded1.txt
# 输出：ASCII text

# 使用正则表达式检测
if grep -qE '^[A-Za-z0-9+/]+={0,2}$' encoded1.txt; then
    echo "可能是 Base64 编码"
fi

# 使用 Python 检测
python3 -c "
import base64
import sys
data = open(sys.argv[1]).read().strip()
try:
    decoded = base64.b64decode(data)
    print('Base64 解码成功:', decoded.decode('utf-8'))
except:
    print('不是有效的 Base64')
" encoded1.txt
```

---

### Step 2：十六进制编码处理

```bash
# 创建十六进制编码文件
echo "Hello" | xxd -p > hex.txt
cat hex.txt
# 输出：48656c6c6f0a

# 十六进制转回文本
xxd -p -r hex.txt

# 使用 OpenSSL
echo "48656c6c6f" | xxd -r -p

# 使用 Python
python3 -c "print(bytes.fromhex('48656c6c6f').decode())"
```

---

### Step 3：识别加密类型

```bash
# 查看加密文件
cat encrypted1.enc

# 使用 file 命令识别
file encrypted1.enc
# 输出：data（二进制数据）

# 查看文件头部（识别加密格式）
xxd encrypted1.enc | head -5

# OpenSSL 加密文件特征：
# 如果使用 -salt 参数，文件开头是 "Salted__"
head -c 8 encrypted1.enc
# 输出：Salted__

# 检测是否是 OpenSSL 加密文件
head -c 8 encrypted1.enc | cat -v
# 如果显示 Salted__ 说明是 OpenSSL salted 格式
```

**加密文件类型判断脚本：**

```bash
#!/bin/bash
# identify_cipher.sh - 加密类型识别脚本

FILE=$1

echo "=== 文件信息 ==="
file "$FILE"
echo ""

echo "=== 文件头（前16字节）==="
xxd -l 16 "$FILE"
echo ""

echo "=== 特征检测 ==="
MAGIC=$(head -c 8 "$FILE" | cat -v)
if [[ "$MAGIC" == "Salted__" ]]; then
    echo "✓ OpenSSL salted 加密格式"
    echo "建议尝试：openssl enc -d -aes-256-cbc -in $FILE"
else
    echo "未识别的加密格式"
fi
```

---

### Step 4：破解弱密码加密

当已知或猜测密码时，可以直接解密：

```bash
# 尝试常见密码解密
for pass in password 123456 admin secret 123 password123; do
    echo "尝试密码: $pass"
    openssl enc -aes-256-cbc -d -in encrypted1.enc -pass pass:$pass 2>/dev/null && echo "成功！" && break
done

# 已知密码解密
openssl enc -aes-256-cbc -d -in encrypted1.enc -pass pass:secret123

# 解密弱密码文件
openssl enc -aes-256-cbc -d -in weak.enc -pass pass:123
```

**字典攻击破解：**

```bash
# 创建密码字典
cat > wordlist.txt << EOF
password
123456
admin
secret
secret123
letmein
qwerty
EOF

# 暴力尝试
while read pass; do
    echo -n "尝试: $pass ... "
    if openssl enc -aes-256-cbc -d -pbkdf2 -in encrypted1.enc -pass pass:$pass 2>/dev/null; then
        echo "✓ 成功！"
        break
    else
        echo "✗ 失败"
    fi
done < wordlist.txt
```

---

### Step 5：确定加密算法

有时加密算法未知，需要尝试不同算法：

```bash
# 列出所有支持的算法
openssl enc -list | grep -i aes

# 尝试不同算法解密
for cipher in aes-128-cbc aes-192-cbc aes-256-cbc aes-128-ecb des-ede3-cbc; do
    echo "尝试算法: $cipher"
    openssl enc -$cipher -d -in encrypted1.enc -pass pass:secret123 2>/dev/null && echo "成功！"
done
```

**自动检测脚本：**

```bash
#!/bin/bash
# crack_cipher.sh - 自动尝试解密

FILE=$1
PASS=$2

CIPHERS="aes-128-cbc aes-192-cbc aes-256-cbc aes-128-cfb aes-256-cfb des-ede3-cbc bf-cbc"

for cipher in $CIPHERS; do
    result=$(openssl enc -$cipher -d -pbkdf2 -in "$FILE" -pass pass:$PASS 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo "算法: $cipher"
        echo "结果: $result"
        break
    fi
done
```

---

### Step 6：多层加密解密

```bash
# 创建多层加密示例
echo "机密数据 Level 3" > original.txt

# 第一层：AES加密
openssl enc -aes-256-cbc -pbkdf2 -pass pass:level1 -in original.txt -out layer1.enc

# 第二层：Base64编码
base64 layer1.enc > layer2.b64

# 第三层：再次AES加密
openssl enc -aes-256-cbc -pbkdf2 -pass pass:level3 -in layer2.b64 -out layer3.enc

echo "三层加密完成：layer3.enc"

# 解密过程（逆向操作）
# 第三层 → 第二层 → 第一层 → 原文

echo "=== 解密第三层 ==="
openssl enc -aes-256-cbc -d -pbkdf2 -pass pass:level3 -in layer3.enc > temp.b64

echo "=== 解码第二层 ==="
base64 -d temp.b64 > temp.enc

echo "=== 解密第一层 ==="
openssl enc -aes-256-cbc -d -pbkdf2 -pass pass:level1 -in temp.enc

# 清理临时文件
rm -f temp.b64 temp.enc
```

---

### Step 7：实战挑战 - 解密神秘文件

假设收到一个加密文件，需要分析并解密：

```bash
# 创建挑战文件
cat > challenge.txt << 'EOF'
U2FsdGVkX1+G3Y9M2dLJQwC3qF/1ZDfKZJnZzbnZhGKXJZLt6NGn5g==
提示：密码是常见的英文单词，6位字母
EOF

# 分析
# 1. "U2FsdGVkX1" Base64解码后是 "Salted__"，说明是OpenSSL加密
# 2. 需要找到正确的密码

# 尝试6位字母常见密码
for pass in secret simple master system danger warning police; do
    result=$(echo "U2FsdGVkX1+G3Y9M2dLJQwC3qF/1ZDfKZJnZzbnZhGKXJZLt6NGn5g==" | base64 -d | openssl enc -aes-256-cbc -d -pbkdf2 -pass pass:$pass 2>/dev/null)
    if [ -n "$result" ]; then
        echo "密码: $pass"
        echo "内容: $result"
        break
    fi
done
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] 如何区分 Base64 编码和 AES 加密？
- [ ] OpenSSL 加密文件的文件头特征是什么？
- [ ] 如何确定加密使用的算法？
- [ ] 为什么需要使用 -pbkdf2 参数？
- [ ] 多层加密的解密顺序是什么？

---

## 💡 进阶挑战

### 挑战1：自动化解密工具

编写一个自动解密脚本：

```python
#!/usr/bin/env python3
# auto_decrypt.py - 自动解密工具

import base64
import subprocess
import sys

def try_openssl_decrypt(file_path, password, cipher='aes-256-cbc'):
    """尝试使用 OpenSSL 解密"""
    cmd = f'openssl enc -{cipher} -d -pbkdf2 -in {file_path} -pass pass:{password}'
    result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    if result.returncode == 0:
        return result.stdout
    return None

def detect_encoding(data):
    """检测编码类型"""
    if data.startswith(b'Salted__'):
        return 'openssl_encrypted'
    try:
        base64.b64decode(data)
        return 'base64'
    except:
        pass
    if all(c in '0123456789abcdefABCDEF\n' for c in data.decode('utf-8', errors='ignore')):
        return 'hex'
    return 'unknown'

def main():
    if len(sys.argv) < 2:
        print("用法: python3 auto_decrypt.py <文件名>")
        return
    
    file_path = sys.argv[1]
    with open(file_path, 'rb') as f:
        data = f.read()
    
    enc_type = detect_encoding(data)
    print(f"检测到类型: {enc_type}")
    
    if enc_type == 'base64':
        decoded = base64.b64decode(data)
        print(f"Base64 解码: {decoded[:100]}")
    elif enc_type == 'openssl_encrypted':
        print("尝试常见密码...")
        common_passwords = ['password', '123456', 'secret', 'admin']
        for pwd in common_passwords:
            result = try_openssl_decrypt(file_path, pwd)
            if result:
                print(f"成功! 密码: {pwd}")
                print(f"内容: {result}")
                break

if __name__ == '__main__':
    main()
```

### 挑战2：隐写术检测

有时加密数据隐藏在图片或其他文件中：

```bash
# 检查文件真实类型
file suspicious.jpg
# 如果显示 data 而不是 JPEG，可能是伪装

# 提取文件末尾的数据
tail -c 1000 suspicious.jpg | xxd

# 使用 binwalk 检测嵌入文件
binwalk suspicious.jpg

# 使用 strings 提取可读字符串
strings suspicious.jpg | grep -i "password\|secret\|key"
```

### 挑战3：CTF 风格挑战

```bash
# 创建 CTF 挑战
flag="FLAG{openssl_master_2024}"
echo "$flag" | openssl enc -aes-128-cbc -pbkdf2 -pass pass:hacker | base64 > ctf_challenge.txt

echo "挑战：解密以下内容，获取 Flag"
echo "密码提示：使用电脑的人的身份"
cat ctf_challenge.txt

# 解决步骤：
# 1. Base64 解码
# 2. 尝试密码（hacker）
# 3. 获取 Flag
```

---

## 🛡️ 防御措施

### 加密安全建议

| 措施 | 说明 |
|------|------|
| **强密码** | 使用复杂密码，避免常见词汇 |
| **PBKDF2** | 使用高迭代次数增强密码强度 |
| **密钥管理** | 不要将密钥与加密文件存储在一起 |
| **算法选择** | 使用 AES-256-GCM 等现代安全算法 |
| **密钥分离** | 加密密钥和解密密钥分开存储 |

### 密码强度检测

```bash
# 使用 Python 检测密码强度
python3 -c "
import sys
pwd = sys.argv[1]
score = 0
if len(pwd) >= 8: score += 1
if len(pwd) >= 12: score += 1
if any(c.isupper() for c in pwd): score += 1
if any(c.islower() for c in pwd): score += 1
if any(c.isdigit() for c in pwd): score += 1
if any(c in '!@#\$%^&*()' for c in pwd): score += 1
print(f'密码强度: {score}/6')
" "MyP@ssw0rd123"
```

---

## 📚 参考资源

- [CyberChef](https://gchq.github.io/CyberChef/) - 在线编解码工具
- [Hash Identifier](https://hashes.com/en/tools/hash_identifier) - 哈希类型识别
- [OpenSSL 文档](https://www.openssl.org/docs/)
- [CTF 加密挑战合集](https://cryptohack.org/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
成功解密的文件：
使用的技巧：
遇到的问题：
解决方案：
学到的知识点：
```

---

*最后更新：2026-05-30*
