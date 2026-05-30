# 🕷️ 实验19：使用 Nikto 扫描 Web 服务器

> **难度**：⭐⭐⭐ 初级  
> **预计时间**：60分钟  
> **工具**：Nikto

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解 Nikto 的工作原理和应用场景
- ✅ 掌握 Nikto 的基本命令语法
- ✅ 使用 Nikto 进行 Web 漏洞扫描
- ✅ 解读 Nikto 扫描结果
- ✅ 掌握绕过 WAF 的技巧
- ✅ 结合 Nmap 使用 Nikto
- ✅ 了解防御措施（WAF/IPS 部署）

---

## 📖 背景知识

### 什么是 Nikto？

**Nikto** 是一款开源的 **Web 服务器扫描器**，用于检测 Web 服务器的危险文件和配置问题。

**主要功能：**

| 功能 | 说明 |
|------|------|
| **漏洞检测** | 检测超过 6700 个危险文件/程序 |
| **配置问题** | 检测服务器配置错误 |
| **过时组件** | 检测过时的服务器版本 |
| **CGI 扫描** | 扫描 CGI 漏洞 |
| **SSL 检测** | 检测 HTTPS 配置问题 |
| **多平台支持** | 支持 Apache、IIS、Nginx 等 |

### Nikto 的工作原理

```
Nikto 工作流程：

1. 主机发现（Host Discovery）
   ↓
   确认目标 Web 服务器可达
      ↓
2. 端口扫描（Port Scanning）
   ↓
   默认扫描 80, 443, 8080, 8180 等
      ↓
3. 服务识别（Service Detection）
   ↓
   获取服务器类型和版本
      ↓
4. 漏洞扫描（Vulnerability Scanning）
   ↓
   发送大量测试请求（基于数据库）
      ↓
5. 结果分析（Analysis）
   ↓
   输出发现的问题和漏洞
```

### Nikto 检测的常见问题

| 问题类型 | 说明 | 风险等级 |
|---------|------|---------|
| **危险文件** | 如 `/admin.php`、`/phpinfo.php` | 中/高 |
| **过时服务器** | 如 Apache 2.2.x（已停止支持） | 中 |
| **默认页面** | 如默认 IIS 页面 | 低 |
| **目录遍历** | 可访问敏感目录 | 高 |
| **XSS 漏洞** | 跨站脚本攻击 | 高 |
| **SQL 注入** | 数据库注入漏洞 | 严重 |
| **SSL 问题** | 弱加密、自签名证书 | 中 |
| **HTTP 方法** | 启用了危险的 HTTP 方法（如 PUT） | 中/高 |

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能对**自己拥有或明确授权的 Web 服务器**进行扫描
> - 未经授权扫描他人网站属于**违法行为**
> - 某些测试可能触发 **WAF/IP 封禁**
> - 建议使用**本地靶机**练习（如 DVWA、Metasploitable2）
> - 不要对生产环境进行扫描

---

## 🔧 环境要求

### 系统要求

```bash
# 检查 Nikto 是否已安装
nikto -Version

# 如未安装：

# Ubuntu/Debian
sudo apt update
sudo apt install -y nikto

# CentOS/RHEL
sudo yum install -y nikto

# macOS
brew install nikto

# 从源码安装（最新版本）
git clone https://github.com/sullo/nikto.git
cd nikto/program
perl nikto.pl -Version
```

### 准备靶机（强烈推荐）

```bash
# 方法1：使用 DVWA（Damn Vulnerable Web Application）
docker run -d -p 80:80 vulnerables/web-dvwa

# 查看容器 IP
docker inspect <容器ID> | grep IPAddress

# 方法2：使用 Metasploitable2
docker run -d -p 80:80 -p 443:443 \
  --name metasploitable tleemcjr/metasploitable2

# 方法3：使用本地 Web 服务器
# 安装 Apache
sudo apt install -y apache2
sudo systemctl start apache2
```

### 验证靶机可达

```bash
# 测试 HTTP 连接
curl http://<靶机IP>

# 测试 HTTPS 连接
curl -k https://<靶机IP>

# 使用浏览器访问
# http://<靶机IP>
```

---

## 👨‍💻 实验步骤

### Step 1：了解 Nikto 基本语法

```bash
# 查看帮助
nikto -Help

# 基本语法格式
nikto -h <目标URL或IP>
```

**最简单的扫描：**

```bash
# 扫描 HTTP 站点
nikto -h http://192.168.1.100

# 扫描 HTTPS 站点
nikto -h https://192.168.1.100

# 扫描特定端口
nikto -h http://192.168.1.100:8080

# 扫描主机名
nikto -h http://example.com
```

**输出示例：**

```
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          192.168.1.100
+ Target Hostname:    192.168.1.100
+ Target Port:        80
+ Start Time:         2026-05-30 21:00:00 (GMT8)
---------------------------------------------------------------------------
+ Server: Apache/2.4.25 (Debian)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-Content-Type-Options header is not set.
+ Cookie PHPSESSID created without the httponly flag
+ No CSP Content-Security-Policy header found.
+ Sticky Header Cookie without SameSite flag
+ Uncommon header 'nikto-added' found
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ /phpinfo.php: Output from the phpinfo() function is displayed.
+ /admin.php: Admin page found
+ /config.php: Configuration file may contain sensitive information.
+ /backup/: Directory indexing found.
+ OSVDB-3092: /install.php: This could be a file insertion 
  vulnerability.
---------------------------------------------------------------------------
+ 15 items checked: 0 error(s) and 12 item(s) reported on remote host
+ Scan completed in 15.23 seconds.
```

---

### Step 2：使用常用参数

```bash
# 指定扫描端口
nikto -h http://192.168.1.100 -p 80,443,8080

# 使用 SSL（HTTPS）
nikto -h https://192.168.1.100 -ssl

# 只扫描特定目录
nikto -h http://192.168.1.100 -C all

# 禁用 SSL 检查（用于自签名证书）
nikto -h https://192.168.1.100 -ssl -no404

# 设置用户代理（绕过 WAF）
nikto -h http://192.168.1.100 -useragent "Mozilla/5.0"

# 使用代理
nikto -h http://192.168.1.100 -proxy http://127.0.0.1:8080

# 使用 Cookie
nikto -h http://192.168.1.100 -Cookie "PHPSESSID=abc123"

# 使用 HTTP 认证
nikto -h http://192.168.1.100 -id admin:password
```

**常用参数说明：**

| 参数 | 说明 |
|------|------|
| `-h` | 指定目标主机（URL 或 IP） |
| `-p` | 指定端口（逗号分隔或范围） |
| `-ssl` | 强制使用 SSL（HTTPS） |
| `-C` | 指定要扫描的目录（逗号分隔） |
| `-no404` | 禁用 404 检查（减少误报） |
| `-useragent` | 设置 User-Agent 头 |
| `-proxy` | 使用代理 |
| `-Cookie` | 设置 Cookie |
| `-id` | HTTP 认证（用户名:密码） |
| `-timeout` | 设置超时时间（秒） |
| `-maxtime` | 设置最大扫描时间（秒） |
| `-Tuning` | 调整扫描策略（见下文） |

---

### Step 3：调整扫描策略（-Tuning）

Nikto 使用 **Tuning** 参数控制扫描的"侵略性"。

```bash
# 查看 Tuning 选项
nikto -Help | grep Tuning

# 常用 Tuning 参数：
# 0 - 文件上传
# 1 - 配置问题
# 2 - 信息泄露
# 3 - 注入（XSS/SQL）
# 4 - 远程文件包含
# 5 - 拒绝服务
# 6 - 代码执行
# 7 - 默认文件
# 8 - 命令注入
# 9 - 目录遍历

# 只扫描配置问题（1）和信息泄露（2）
nikto -h http://192.168.1.100 -Tuning 1,2

# 扫描所有类型（默认）
nikto -h http://192.168.1.100 -Tuning x

# 扫描除了默认文件（7）之外的所有类型
nikto -h http://192.168.1.100 -Tuning x -Tuning 7
```

---

### Step 4：输出格式选项

```bash
# 默认输出（终端）
nikto -h http://192.168.1.100

# 保存为文本文件
nikto -h http://192.168.1.100 -o result.txt -Format txt

# 保存为 HTML 文件
nikto -h http://192.168.1.100 -o result.html -Format html

# 保存为 XML 文件
nikto -h http://192.168.1.100 -o result.xml -Format xml

# 保存为 CSV 文件
nikto -h http://192.168.1.100 -o result.csv -Format csv

# 同时输出到终端和文件（使用 tee）
nikto -h http://192.168.1.100 | tee result.txt
```

**输出格式参数：**

| 参数 | 格式 | 说明 |
|------|------|------|
| `-Format txt` | 文本格式 | 人类可读 |
| `-Format html` | HTML 格式 | 便于浏览 |
| `-Format xml` | XML 格式 | 便于程序解析 |
| `-Format csv` | CSV 格式 | 便于表格处理 |
| `-Format msf+json` | JSON 格式 | 便于 Metasploit 导入 |

---

### Step 5：绕过 WAF 技巧

**WAF（Web Application Firewall）** 会拦截 Nikto 的扫描请求。以下是一些绕过技巧：

#### 技巧1：使用代理和 Tor

```bash
# 使用 HTTP 代理
nikto -h http://192.168.1.100 -proxy http://proxy.example.com:8080

# 使用 SOCKS 代理（需要安装 proxychains）
proxychains nikto -h http://192.168.1.100
```

#### 技巧2：修改 User-Agent

```bash
# 使用常见浏览器的 User-Agent
nikto -h http://192.168.1.100 \
  -useragent "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"

# 随机 User-Agent（需要脚本）
cat > random_ua.sh << 'EOF'
#!/bin/bash
UAS=(
  "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
  "Mozilla/5.0 (X11; Linux x86_64)"
)
RANDOM_UA=${UAS[$RANDOM % ${#UAS[@]}]}
nikto -h http://192.168.1.100 -useragent "$RANDOM_UA"
EOF
chmod +x random_ua.sh
./random_ua.sh
```

#### 技巧3：降低扫描速度

```bash# 增加延迟（毫秒）
nikto -h http://192.168.1.100 -pause 1000

# 使用单线程（默认是多线程）
nikto -h http://192.168.1.100 -nolookup
```

#### 技巧4：分片请求

```bash
# 使用 Host 头绕过（某些 WAF 配置错误）
nikto -h http://192.168.1.100 -vhost example.com

# 使用 Encoding 绕过（需要 WAF 配置不当）
nikto -h http://192.168.1.100 -evasion 1
```

**Evasion 参数（编码绕过）：**

| 参数 | 说明 |
|------|------|
| `-evasion 1` | 使用 `/./` 绕过路径过滤 |
| `-evasion 2` | 使用 `%2e%2e/` 绕过 |
| `-evasion 3` | 使用 `/./` + 双重编码 |
| `-evasion 4` | 使用 `%0a` 绕过 |
| `-evasion 5` | 使用 `%0d` 绕过 |
| `-evasion 6` | 使用 TAB（`%09`）绕过 |
| `-evasion 7` | 使用大小写混淆绕过 |
| `-evasion 8` | 使用 Windows 路径（`\`）绕过 |

---

### Step 6：结合 Nmap 使用

Nmap 和 Nikto 可以配合使用，发挥各自优势：

```bash
# 第1步：使用 Nmap 发现 Web 服务
nmap -sS -sV -p 80,443,8080 192.168.1.0/24 -oG web_hosts.grep

# 第2步：提取开放 Web 端口的主机
grep "80/open\|443/open\|8080/open" web_hosts.grep | \
  cut -d' ' -f2 > web_targets.txt

# 第3步：使用 Nikto 扫描这些主机
for target in $(cat web_targets.txt); do
  echo "=== 扫描 $target ==="
  nikto -h http://$target -o nikto_$target.html -Format html
done
```

**自动化脚本示例：**

```bash
cat > nmap_nikto.sh << 'EOF'
#!/bin/bash

SUBNET="192.168.1.0/24"
OUTPUT_DIR="nikto_scan_$(date +%Y%m%d_%H%M%S)"

mkdir -p "$OUTPUT_DIR"

echo "[*] 第1步：使用 Nmap 发现 Web 服务..."
nmap -sS -sV -p 80,443,8080 "$SUBNET" -oG "$OUTPUT_DIR/nmap.grep"

echo "[*] 第2步：提取 Web 服务器..."
grep "80/open\|443/open\|8080/open" "$OUTPUT_DIR/nmap.grep" | \
  cut -d' ' -f2 > "$OUTPUT_DIR/web_targets.txt"

echo "[*] 发现 $(wc -l < "$OUTPUT_DIR/web_targets.txt") 个 Web 服务器"

echo "[*] 第3步：使用 Nikto 扫描..."
while read target; do
  echo "  扫描 $target ..."
  nikto -h "http://$target" -o "$OUTPUT_DIR/nikto_$target.html" \
    -Format html -nointeractive
done < "$OUTPUT_DIR/web_targets.txt"

echo "[*] 扫描完成！结果保存在 $OUTPUT_DIR/"
EOF

chmod +x nmap_nikto.sh
./nmap_nikto.sh
```

---

### Step 7：实战 - 扫描 DVWA

```bash
# 1. 启动 DVWA 靶机
docker run -d -p 80:80 vulnerables/web-dvwa

# 2. 获取容器 IP
DVWA_IP=$(docker inspect <容器ID> | grep IPAddress | \
  tail -1 | cut -d'"' -f4)

# 3. 使用 Nikto 扫描
nikto -h http://$DVWA_IP -output dvwa_scan.html -Format html

# 4. 查看结果
cat dvwa_scan.html | grep -i "vulnerability\|dangerous\|outdated"
```

**DVWA 常见发现：**

```
+ /dvwa/: Directory indexing found.
+ /dvwa/phpinfo.php: PHP information file found.
+ /dvwa/config/config.inc.php: Configuration file found.
+ /dvwa/install.php: Installation file found.
+ Cookie DVWASESSID created without the httponly flag
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ /dvwa/login.php: Admin login page found.
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] Nikto 的主要功能是什么？
- [ ] 如何使用 Nikto 扫描 HTTPS 站点？
- [ ] `-Tuning` 参数的作用是什么？
- [ ] 如何绕过 WAF 的拦截？
- [ ] 如何结合 Nmap 和 Nikto 进行扫描？

---

## 💡 进阶挑战

### 挑战1：批量扫描

```bash
# 创建目标列表
cat > targets.txt << EOF
http://192.168.1.100
http://192.168.1.101
https://192.168.1.102
EOF

# 批量扫描
while read target; do
  echo "=== 扫描 $target ==="
  nikto -h "$target" -o "nikto_$(echo $target | cut -d'/' -f3).html" \
    -Format html
done < targets.txt
```

### 挑战2：使用 Nmap NSE 脚本

```bash
# Nmap 也有 HTTP 漏洞扫描脚本
nmap --script http-vuln* -p 80,443 <靶机IP>

# 结合 Nikto 使用
nikto -h http://<靶机IP> -o nikto_result.html -Format html
nmap --script http-vuln* -p 80,443 <靶机IP> -oX nmap_result.xml
```

### 挑战3：解析 Nikto XML 输出

```bash
# 使用 Python 解析 XML
python3 << 'EOF'
import xml.etree.ElementTree as ET

tree = ET.parse('nikto_result.xml')
root = tree.getroot()

for item in root.findall('.//scandetail/item'):
    print(f"URL: {item.find('url').text}")
    print(f"描述: {item.find('description').text}")
    print("-" * 50)
EOF
```

---

## 🛡️ 防御措施（如何防御 Nikto 扫描）

### 部署 WAF（Web 应用防火墙）

| WAF 产品 | 类型 | 说明 |
|---------|------|------|
| **ModSecurity** | 开源 | Apache/Nginx 模块 |
| **Cloudflare** | 商业 | 云 WAF |
| **AWS WAF** | 商业 | AWS 云 WAF |
| **F5 BIG-IP** | 商业 | 硬件 WAF |
| **Imperva** | 商业 | 云 WAF |

### 配置 ModSecurity 示例

```bash
# 安装 ModSecurity
sudo apt install -y libapache2-mod-security2

# 启用 ModSecurity
sudo a2enmod security2
sudo systemctl restart apache2

# 配置规则（检测 Nikto）
sudo cat > /etc/modsecurity/nikto_rules.conf << 'EOF'
# 检测 Nikto User-Agent
SecRule REQUEST_HEADERS:User-Agent "nikto" \
  "id:1001,deny,status:403,msg:'Nikto Scanner Detected'"

# 检测 Nikto 特征路径
SecRule REQUEST_URI "@contains /nikto/" \
  "id:1002,deny,status:403,msg:'Nikto Path Detected'"

# 检测常见扫描路径
SecRule REQUEST_URI "@rx (phpinfo|admin|config|backup)" \
  "id:1003,deny,status:403,msg:'Suspicious Path Detected'"
EOF

# 应用规则
sudo cat >> /etc/apache2/mods-available/security2.conf << 'EOF'
Include /etc/modsecurity/nikto_rules.conf
EOF

sudo systemctl restart apache2
```

### 部署 IPS（入侵防御系统）

```bash
# 使用 Fail2Ban 防御 Nikto
sudo apt install -y fail2ban

# 创建 Nikto 检测过滤器
sudo cat > /etc/fail2ban/filter.d/nikto.conf << 'EOF'
[Definition]
failregex = ^<HOST>.*"GET /.*phpinfo\.php.*$
            ^<HOST>.*"GET /.*admin\.php.*$
            ^<HOST>.*"GET /.*config\.php.*$
ignoreregex =
EOF

# 启用防护
sudo cat > /etc/fail2ban/jail.local << 'EOF'
[nikto]
enabled = true
filter = nikto
logpath = /var/log/apache2/access.log
maxretry = 5
bantime = 3600
EOF

sudo systemctl restart fail2ban
```

### 其他防御措施

| 防御措施 | 说明 |
|---------|------|
| **隐藏服务器信息** | 修改 Server 响应头 |
| **禁用目录浏览** | 配置 Web 服务器 |
| **删除默认文件** | 删除 `phpinfo.php`、`readme.html` 等 |
| **定期更新** | 及时更新 Web 服务器和应用程序 |
| **最小权限原则** | 只开放必要的端口和服务 |
| **监控日志** | 定期检查访问日志 |

### 配置示例（Apache）

```bash
# 隐藏服务器信息
sudo nano /etc/apache2/conf-available/security.conf
# 添加：
ServerTokens Prod
ServerSignature Off

# 禁用目录浏览
<Directory /var/www/html>
    Options -Indexes
</Directory>

# 限制 HTTP 方法
<LimitExcept GET POST HEAD>
    Require all denied
</LimitExcept>

# 重启 Apache
sudo systemctl restart apache2
```

---

## 📚 参考资源

- [Nikto 官方 GitHub](https://github.com/sullo/nikto)
- [Nikto 官方文档](https://cirt.net/nikto2-docs/)
- [Nikto 使用指南](https://www.hackingarticles.in/nikto-tutorial/)
- [ModSecurity 官方文档](https://modsecurity.org/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
目标URL：
使用的参数：
发现的漏洞：
绕过 WAF 的方法：
遇到的问题：
解决方案：
```

---

*最后更新：2026-05-30*
