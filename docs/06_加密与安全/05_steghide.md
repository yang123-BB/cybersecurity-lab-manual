# 🎭 实验18：使用 Steghide 隐藏数据

> **难度**：⭐⭐⭐ 中级  
> **预计时间**：45分钟  
> **工具**：Steghide、Stegdetect、Stegseek

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解隐写术的基本原理和应用场景
- ✅ 使用 Steghide 在图片中隐藏和提取数据
- ✅ 分析隐写术的检测方法
- ✅ 使用隐写分析工具检测隐藏内容
- ✅ 了解隐写术在安全领域的双重用途

---

## 📖 背景知识

### 什么是隐写术（Steganography）？

**隐写术**是将秘密信息隐藏在普通载体中的技术，目的是让信息的存在本身不被发现。

```
隐写术 vs 加密：

加密（Encryption）：
├── 目的：保护内容机密性
├── 特点：数据不可读，但能看出被加密
└── 例子：AES、RSA

隐写术（Steganography）：
├── 目的：隐藏通信的存在
├── 特点：数据看起来正常，内部隐藏秘密
└── 例子：图片隐写、音频隐写

组合使用：
├── 先加密数据
├── 再进行隐写
└── 达到双重保护
```

### 隐写术载体类型

| 载体类型 | 原理 | 常用工具 |
|---------|------|---------|
| 图片（JPEG/PNG/BMP） | 修改像素值 LSB | Steghide、OpenStego |
| 音频（WAV/MP3） | 修改音频采样 | OpenPuff、DeepSound |
| 视频（AVI/MP4） | 修改帧数据 | OpenPuff |
| 文档（PDF/DOC） | 修改格式数据 | Stegdetect |
| 网络流量 | 隐藏在协议字段 | CovertTCP |

### LSB 隐写原理

```
LSB（Least Significant Bit）隐写：

图像像素值示例（8位灰度）：
原始值：11010011（十进制 211）
修改后：11010010（十进制 210）

只改变最低位，人眼几乎无法察觉差异。

一个像素可以隐藏 1 bit 信息
一张 1920x1080 的图片可以隐藏约 259KB 数据

RGB 图片：
每个像素有 R、G、B 三个通道
每个通道可以存储 1 bit
容量 = 1920 × 1080 × 3 = 6,220,800 bits ≈ 777KB
```

### 隐写术应用场景

```
合法应用：
├── 数字水印（版权保护）
├── 隐蔽通信（记者、活动家）
├── 安全传输敏感数据
└── CTF 竞赛挑战

非法应用（需防范）：
├── 恶意软件隐藏
├── 敏感数据外泄
├── 恐怖分子通信
└── 商业间谍活动
```

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 隐写术可能被用于非法目的，请负责任地使用
> - 不要使用隐写术隐藏或传播非法内容
> - 学习检测方法是为了保护组织安全
> - 在授权环境下进行实验练习

---

## 🔧 环境要求

### 系统要求

```bash
# 检查系统
cat /etc/os-release

# 安装 Steghide
sudo apt update
sudo apt install -y steghide

# 验证安装
steghide --version

# 安装检测工具
sudo apt install -y stegseek

# Stegdetect（可能需要从源码安装）
# git clone https://github.com/abeluck/stegdetect.git
# cd stegdetect
# ./configure && make && sudo make install

# 创建工作目录
mkdir -p ~/stego_lab/{images,secret,output}
cd ~/stego_lab
```

### 准备测试文件

```bash
# 创建秘密消息
echo "这是隐藏的秘密消息！密码是：Admin@123" > secret/message.txt

# 创建测试图片（或使用现有图片）
# 方法1：使用系统自带图片
cp /usr/share/pixmaps/*.jpg images/ 2>/dev/null || \
cp /usr/share/backgrounds/*.jpg images/ 2>/dev/null

# 方法2：创建简单图片
convert -size 200x200 xc:blue images/blue.png 2>/dev/null || \
echo "需要 ImageMagick 来创建测试图片"

# 方法3：下载示例图片
wget -q https://picsum.photos/800/600 -O images/photo.jpg 2>/dev/null || \
echo "使用本地图片即可"
```

---

## 👨‍💻 实验步骤

### Step 1：Steghide 基本用法

```bash
# 查看帮助
steghide --help

# 查看支持的格式
steghide info --list

# 输出示例：
# Supported formats:
#   JPEG, BMP, WAV, AU
```

---

### Step 2：在图片中隐藏数据

```bash
# 准备图片（假设有一张 test.jpg）
# 如果没有，使用任意 JPEG 图片

# 嵌入秘密数据
steghide embed -cf images/photo.jpg -ef secret/message.txt

# 参数说明：
# -cf (cover file)：载体文件（图片）
# -ef (embed file)：要嵌入的文件
# -p (passphrase)：密码（可选，不指定会提示输入）

# 输入密码：
# Enter passphrase: [输入密码]
# Re-Enter passphrase: [再次输入]

# 嵌入成功后，原文件不变
# 默认会创建嵌入后的文件，覆盖原文件

# 指定输出文件名
steghide embed -cf images/photo.jpg -ef secret/message.txt -sf output/stego.jpg -p mypassword

# 参数：
# -sf (stego file)：输出文件名
# -p：直接指定密码（避免交互提示）
```

**嵌入多种类型的数据：**

```bash
# 嵌入文本文件
echo "机密文档内容" > secret/secret.txt
steghide embed -cf images/photo.jpg -ef secret/secret.txt -sf output/stego_text.jpg -p pass123

# 嵌入压缩文件
tar -czf secret/archive.tar.gz secret/
steghide embed -cf images/photo.jpg -ef secret/archive.tar.gz -sf output/stego_archive.jpg -p pass123

# 嵌入加密后的文件
echo "先加密再隐写" | openssl enc -aes-256-cbc -pbkdf2 -pass pass:crypto -out secret/encrypted.bin
steghide embed -cf images/photo.jpg -ef secret/encrypted.bin -sf output/stego_encrypted.jpg -p hidden
```

---

### Step 3：提取隐藏数据

```bash
# 从隐写图片中提取数据
steghide extract -sf output/stego.jpg -p mypassword

# 参数说明：
# -sf (stego file)：包含隐藏数据的图片
# -p：密码
# -xf (extract file)：指定输出文件名（可选）

# 提取并指定输出文件名
steghide extract -sf output/stego_text.jpg -p pass123 -xf output/extracted.txt

# 查看提取的内容
cat output/extracted.txt

# 提取压缩文件
steghide extract -sf output/stego_archive.jpg -p pass123 -xf output/extracted.tar.gz
tar -xzf output/extracted.tar.gz -C output/
cat output/secret/secret.txt
```

---

### Step 4：获取隐写图片信息

```bash
# 查看图片信息（不含密码无法看到嵌入内容）
steghide info output/stego.jpg

# 如果没有嵌入数据：
# "output/stego.jpg" 的格式无法识别或不包含隐藏数据。

# 如果有嵌入数据但无密码：
# "output/stego.jpg" 包含隐藏数据。
# 您需要密码才能提取这些数据。
# Enter passphrase: [需要密码]

# 使用密码查看信息
echo "pass123" | steghide info output/stego_text.jpg -p pass123

# 输出示例：
# "output/stego_text.jpg" 包含隐藏数据。
#   已嵌入 "secret.txt"。
#   大小: 28 字节。
#   已加密: yes
#   已压缩: yes
```

---

### Step 5：嵌入选项配置

```bash
# 禁用加密（隐藏数据不加密）
steghide embed -cf images/photo.jpg -ef secret/message.txt \
    -sf output/no_encrypt.jpg \
    -p pass123 \
    -e none

# -e (encryption)：加密算法
# none: 不加密
# aes: AES 加密
# blowfish: Blowfish 加密

# 禁用压缩
steghide embed -cf images/photo.jpg -ef secret/message.txt \
    -sf output/no_compress.jpg \
    -p pass123 \
    -z 0

# -z (compression)：压缩级别 0-9
# 0: 不压缩
# 9: 最大压缩

# 查看完整嵌入信息
steghide info output/no_encrypt.jpg -p pass123
# 输出显示：已加密: no
```

---

### Step 6：隐写分析 - 检测隐藏数据

```bash
# 使用 Stegseek 进行字典攻击破解密码
stegseek --crack output/stego_text.jpg /usr/share/wordlists/rockyou.txt -xf output/cracked.txt

# 输出示例：
# StegSeek version 0.6
# Cracking "output/stego_text.jpg" using wordlist "/usr/share/wordlists/rockyou.txt"
# -- Password found: 'pass123'
# Original file saved as 'output/cracked.txt'

# 如果没有 rockyou.txt，创建小字典测试
cat > wordlist.txt << EOF
password
123456
pass123
admin
secret
EOF

stegseek --crack output/stego_text.jpg wordlist.txt -xf output/cracked.txt

# 检测是否有嵌入数据（需要正确密码）
# Stegseek 还可以提取嵌入数据的哈希用于分析
stegseek --seed output/stego_text.jpg
```

**使用 Stegdetect 检测：**

```bash
# 如果安装了 stegdetect
stegdetect output/stego_text.jpg

# 输出可能显示：
# output/stego_text.jpg: steghide(***, "AES-128", "CBC")

# 批量检测
stegdetect *.jpg

# 使用不同敏感度
stegdetect -s 2 output/stego_text.jpg
# -s 1-5，数字越大越敏感
```

---

### Step 7：统计检测方法

```bash
# 比较原图和隐写图的差异

# 使用 ImageMagick 比较
compare images/photo.jpg output/stego_text.jpg diff.png

# 查看差异图
# 如果嵌入数据量小，差异可能不明显

# 文件大小对比
ls -la images/photo.jpg output/stego_text.jpg

# 哈希对比
md5sum images/photo.jpg output/stego_text.jpg

# 使用 Python 进行统计检测
python3 << 'EOF'
from PIL import Image
import numpy as np

def analyze_lsb(image_path):
    """分析图片 LSB 分布"""
    img = Image.open(image_path)
    pixels = np.array(img)
    
    # 提取最低位
    lsb_values = pixels & 1
    ones = np.sum(lsb_values)
    total = lsb_values.size
    
    # 理论上随机分布应该是 50%
    ratio = ones / total
    
    print(f"图片: {image_path}")
    print(f"LSB 为 1 的比例: {ratio:.4f}")
    print(f"与 0.5 的偏差: {abs(ratio - 0.5):.4f}")
    
    if abs(ratio - 0.5) < 0.01:
        print("结论: 可能是自然图片")
    else:
        print("结论: 可能存在隐写数据")

analyze_lsb("images/photo.jpg")
analyze_lsb("output/stego_text.jpg")
EOF
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] 隐写术和加密有什么区别？
- [ ] LSB 隐写的工作原理是什么？
- [ ] Steghide 支持哪些载体格式？
- [ ] 如何检测图片中是否包含隐藏数据？
- [ ] 为什么嵌入数据后图片看起来没有变化？

---

## 💡 进阶挑战

### 挑战1：批量隐写工具

```bash
#!/bin/bash
# batch_stego.sh - 批量隐写工具

MODE=$1
FOLDER=$2
PASSWORD=$3

case $MODE in
    embed)
        # 遍历图片，嵌入随机数据
        for img in images/*.jpg; do
            name=$(basename "$img" .jpg)
            echo "隐藏数据于: $img"
            echo "Hidden in $name" > "secret/msg_$name.txt"
            steghide embed -cf "$img" -ef "secret/msg_$name.txt" \
                -sf "output/stego_$name.jpg" -p "$PASSWORD" -f
        done
        ;;
    extract)
        # 批量提取
        for img in output/stego_*.jpg; do
            name=$(basename "$img" .jpg)
            echo "提取数据从: $img"
            steghide extract -sf "$img" -p "$PASSWORD" -xf "output/extract_$name.txt" 2>/dev/null
        done
        ;;
    *)
        echo "用法: $0 <embed|extract> <folder> <password>"
        ;;
esac
```

### 挑战2：创建隐蔽通信系统

```python
#!/usr/bin/env python3
# covert_channel.py - 隐蔽通信系统

import os
import sys
import subprocess
import base64

class StegoChannel:
    def __init__(self, image_dir="images", password="secret"):
        self.image_dir = image_dir
        self.password = password
        self.current_image = 0
    
    def hide_message(self, message, output_dir="output"):
        """隐藏消息"""
        # 选择图片
        images = [f for f in os.listdir(self.image_dir) if f.endswith('.jpg')]
        if not images:
            print("没有可用的载体图片")
            return False
        
        cover = os.path.join(self.image_dir, images[self.current_image % len(images)])
        stego = os.path.join(output_dir, f"msg_{self.current_image}.jpg")
        
        # 保存消息到临时文件
        temp_file = f"/tmp/msg_{self.current_image}.txt"
        with open(temp_file, 'w') as f:
            f.write(message)
        
        # 使用 steghide 嵌入
        cmd = f'steghide embed -cf "{cover}" -ef "{temp_file}" -sf "{stego}" -p "{self.password}" -f'
        result = subprocess.run(cmd, shell=True, capture_output=True)
        
        os.remove(temp_file)
        
        if result.returncode == 0:
            print(f"消息已隐藏在: {stego}")
            self.current_image += 1
            return True
        return False
    
    def extract_message(self, stego_image):
        """提取消息"""
        temp_file = "/tmp/extracted.txt"
        
        cmd = f'steghide extract -sf "{stego_image}" -p "{self.password}" -xf "{temp_file}" -f'
        result = subprocess.run(cmd, shell=True, capture_output=True)
        
        if result.returncode == 0:
            with open(temp_file, 'r') as f:
                message = f.read()
            os.remove(temp_file)
            return message
        return None

# 使用示例
if __name__ == '__main__':
    channel = StegoChannel(password="mysecret")
    
    # 发送方：隐藏消息
    channel.hide_message("Hello, this is a secret message!")
    
    # 接收方：提取消息
    msg = channel.extract_message("output/msg_0.jpg")
    if msg:
        print(f"提取的消息: {msg}")
```

### 挑战3：数字水印实现

```python
#!/usr/bin/env python3
# watermark.py - 简单数字水印

from PIL import Image
import numpy as np

def embed_watermark(image_path, watermark_text, output_path):
    """嵌入数字水印"""
    img = Image.open(image_path)
    pixels = np.array(img, dtype=np.uint8)
    
    # 将水印转换为二进制
    watermark_bin = ''.join(format(ord(c), '08b') for c in watermark_text)
    watermark_bin += '00000000'  # 终止符
    
    # 嵌入到蓝色通道的 LSB
    idx = 0
    for i in range(pixels.shape[0]):
        for j in range(pixels.shape[1]):
            if idx < len(watermark_bin):
                # 修改蓝色通道的最低位
                pixels[i, j, 2] = (pixels[i, j, 2] & 0xFE) | int(watermark_bin[idx])
                idx += 1
            else:
                break
        if idx >= len(watermark_bin):
            break
    
    # 保存
    result = Image.fromarray(pixels)
    result.save(output_path)
    print(f"水印已嵌入: {output_path}")

def extract_watermark(image_path):
    """提取数字水印"""
    img = Image.open(image_path)
    pixels = np.array(img)
    
    # 提取蓝色通道的 LSB
    bits = []
    for i in range(pixels.shape[0]):
        for j in range(pixels.shape[1]):
            bits.append(pixels[i, j, 2] & 1)
    
    # 转换为文本
    chars = []
    for i in range(0, len(bits), 8):
        byte = bits[i:i+8]
        if len(byte) == 8:
            char = chr(int(''.join(map(str, byte)), 2))
            if char == '\x00':
                break
            chars.append(char)
    
    return ''.join(chars)

# 使用
if __name__ == '__main__':
    # 嵌入水印
    embed_watermark("images/photo.jpg", "Copyright 2024", "output/watermarked.jpg")
    
    # 提取水印
    watermark = extract_watermark("output/watermarked.jpg")
    print(f"水印内容: {watermark}")
```

### 挑战4：使用 OpenStego（GUI 工具）

```bash
# OpenStego 是一个开源的隐写工具，支持多种格式

# 下载（需要 Java）
wget https://github.com/syedjahangiralam/openstego/releases/download/v0.8.6/openstego-0.8.6.zip
unzip openstego-0.8.6.zip
cd openstego-0.8.6

# 命令行使用
java -jar openstego.jar embed -a randomlsb \
    -mf secret/message.txt \
    -cf images/photo.jpg \
    -sf output/openstego.png

# 提取
java -jar openstego.jar extract -a randomlsb \
    -sf output/openstego.png \
    -xf output/extracted.txt
```

---

## 🛡️ 防御措施

### 隐写检测策略

| 方法 | 说明 | 工具 |
|------|------|------|
| 视觉检测 | 查看图片异常区域 | 图像查看器 |
| 统计分析 | 分析 LSB 分布 | Stegdetect、自写脚本 |
| 字典攻击 | 尝试常见密码 | Stegseek |
| 特征检测 | 检测隐写工具特征 | Stegdetect |
| 文件大小 | 对比正常图片大小 | 手动分析 |

### 企业防护建议

```bash
# 1. 监控图片上传
# 使用脚本检测上传的图片

#!/bin/bash
# scan_uploads.sh
for file in uploads/*.jpg; do
    # 文件大小异常检测
    size=$(stat -f%z "$file" 2>/dev/null || stat -c%s "$file")
    if [ $size -gt 5000000 ]; then
        echo "警告: 大文件 $file"
    fi
    
    # 使用 stegseek 快速检测
    stegseek --seed "$file" 2>/dev/null && echo "可疑: $file"
done

# 2. 限制文件类型和大小
# Web 服务器配置

# 3. 部署 DLP（数据泄露防护）系统
# 检测敏感数据外泄

# 4. 网络流量分析
# 检测隐蔽通道
```

---

## 📚 参考资源

- [Steghide 官方文档](https://steghide.sourceforge.net/)
- [Stegseek 项目](https://github.com/RickdeJager/stegseek)
- [OpenStego](https://www.openstego.com/)
- [Steganography 教程](https://www.garykessler.net/library/steganography.html)
- [CTF 隐写题库](https://ctfchallenge.co.uk/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
使用的载体图片：
隐藏的数据类型：
提取结果：
检测方法尝试：
学到的知识点：
```

---

*最后更新：2026-05-30*
