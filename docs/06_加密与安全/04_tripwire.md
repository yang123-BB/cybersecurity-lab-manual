# 🛡️ 实验16：使用 Tripwire 监控文件

> **难度**：⭐⭐⭐ 中级  
> **预计时间**：60分钟  
> **工具**：Tripwire、inotify-tools

---

## 🎯 学习目标

通过本实验，你将学会：

- ✅ 理解文件完整性监控的原理和重要性
- ✅ 安装和配置 Tripwire
- ✅ 建立文件系统基线数据库
- ✅ 检测和分析文件篡改
- ✅ 配置监控策略和告警
- ✅ 使用其他文件监控工具

---

## 📖 背景知识

### 什么是文件完整性监控（FIM）？

**文件完整性监控（File Integrity Monitoring, FIM）** 是一种安全技术，用于：

```
监控目标：
├── 系统配置文件（/etc/）
├── 关键二进制文件（/bin/, /sbin/）
├── 用户数据文件
├── Web 目录文件
└── 日志文件

监控内容：
├── 文件内容的修改
├── 文件属性的变更（权限、所有者）
├── 文件的创建和删除
└── 文件访问时间（可选）

检测场景：
├── 恶意软件感染
├── 未授权的配置修改
├── 内部人员违规操作
├── Webshell 上传
└── 合规审计要求
```

### Tripwire 工作原理

```
Tripwire 工作流程：

1. 初始化阶段
   ├── 扫描指定目录
   ├── 计算每个文件的哈希值
   ├── 记录文件属性
   └── 生成基线数据库

2. 检查阶段
   ├── 重新扫描目录
   ├── 对比当前状态与基线
   ├── 检测差异
   └── 生成报告

3. 更新阶段
   ├── 审核变更是否合法
   ├── 更新数据库（合法变更）
   └── 保持数据库最新
```

### 哈希算法说明

| 算法 | 输出长度 | 安全性 | 用途 |
|------|---------|--------|------|
| MD5 | 128位 | 已破解 | 快速对比，不推荐安全场景 |
| SHA-1 | 160位 | 已破解 | 遗留系统 |
| SHA-256 | 256位 | 安全 | 推荐使用 |
| SHA-512 | 512位 | 安全 | 高安全需求 |
| BLAKE2 | 256/512位 | 安全 | 高性能场景 |

### 合规要求

文件完整性监控是多项安全标准的要求：

- **PCI DSS**：要求监控关键文件的变更
- **NIST 800-53**：要求检测未授权修改
- **ISO 27001**：要求保护系统完整性
- **CIS Benchmark**：建议部署 FIM 工具

### ⚠️ 法律声明

> **本实验仅供学习使用！**
> - 只能监控**自己管理的系统**
> - 在监控他人系统前需获得明确授权
> - 监控数据应妥善保管，不得泄露
> - 遵守组织的安全策略和隐私法规

---

## 🔧 环境要求

### 系统要求

```bash
# 检查系统（建议在 Linux 环境运行）
cat /etc/os-release

# 安装 Tripwire（Debian/Ubuntu）
sudo apt update
sudo apt install -y tripwire

# 安装过程会提示：
# 1. 是否使用站点密钥？选择 Yes
# 2. 输入站点密钥密码（用于签名配置文件和数据库）
# 3. 是否使用本地密钥？选择 Yes
# 4. 输入本地密钥密码（用于签名报告）

# 验证安装
tripwire --version
```

### 安装后初始化

```bash
# 生成密钥（如果安装时未生成）
sudo twadmin -m -G -s /etc/tripwire/site.key
sudo twadmin -m -G -L /etc/tripwire/$(hostname)-local.key

# 创建配置文件
sudo twadmin -m -c /etc/tripwire/twcfg.txt -S /etc/tripwire/site.key

# 查看默认配置
cat /etc/tripwire/twcfg.txt

# 关键配置项：
# POLFILE      = /etc/tripwire/tw.pol        # 策略文件
# DBFILE       = /var/lib/tripwire/$(HOSTNAME).twd  # 数据库
# REPORTFILE   = /var/lib/tripwire/report/$(HOSTNAME)-$(DATE).twr  # 报告
# SITEKEYFILE  = /etc/tripwire/site.key      # 站点密钥
# LOCALKEYFILE = /etc/tripwire/$(HOSTNAME)-local.key  # 本地密钥
```

---

## 👨‍💻 实验步骤

### Step 1：配置监控策略

Tripwire 使用策略文件定义要监控的文件和规则：

```bash
# 查看默认策略模板
cat /etc/tripwire/twpol.txt | head -100

# 创建自定义策略（监控 Web 目录）
sudo cat > /etc/tripwire/twpol_web.txt << 'EOF'
# Tripwire Web 目录监控策略

# 规则定义
(
  rulename = "Web Content",
  severity = $(SIG_HI)
)
{
  # 监控整个 Web 目录
  /var/www/html          -> $(SEC_BIN) ;
  
  # 监控特定文件类型
  /var/www/html/index.html -> $(SEC_CONFIG) ;
  
  # 排除临时文件
  !/var/www/html/tmp ;
  !/var/www/html/cache ;
  
  # 监控配置文件
  /etc/apache2/apache2.conf -> $(SEC_CONFIG) ;
  /etc/nginx/nginx.conf -> $(SEC_CONFIG) ;
}

# 系统关键文件监控
(
  rulename = "System Binaries",
  severity = $(SIG_HI)
)
{
  /bin -> $(SEC_BIN) ;
  /sbin -> $(SEC_BIN) ;
  /usr/bin -> $(SEC_BIN) ;
  /usr/sbin -> $(SEC_BIN) ;
}

# 配置文件监控
(
  rulename = "System Configuration",
  severity = $(SIG_MED)
)
{
  /etc/passwd -> $(SEC_CONFIG) ;
  /etc/shadow -> $(SEC_CONFIG) ;
  /etc/sudoers -> $(SEC_CONFIG) ;
  /etc/ssh/sshd_config -> $(SEC_CONFIG) ;
}
EOF

# 编译策略文件
sudo twadmin -m -P /etc/tripwire/twpol_web.txt -S /etc/tripwire/site.key

# 验证策略文件语法
sudo twadmin -m -p | head -20
```

**策略规则类型说明：**

| 变量 | 说明 | 监控内容 |
|------|------|---------|
| `$(SEC_BIN)` | 二进制文件 | 所有属性 |
| `$(SEC_CONFIG)` | 配置文件 | 内容和属性 |
| `$(SEC_LOG)` | 日志文件 | 增长模式 |
| `$(SEC_INVARIANT)` | 不变文件 | 任何变化 |
| `$(SIG_HI)` | 高严重性 | 红色警报 |
| `$(SIG_MED)` | 中严重性 | 黄色警报 |
| `$(SIG_LOW)` | 低严重性 | 绿色警报 |

---

### Step 2：初始化数据库

```bash
# 初始化数据库（扫描文件并建立基线）
sudo tripwire --init

# 输出示例：
# Parsing policy file: /etc/tripwire/tw.pol
# Generating database...
# Wrote database file: /var/lib/tripwire/debian.twd

# 查看数据库信息
sudo ls -la /var/lib/tripwire/

# 验证数据库
sudo tripwire --check
```

**常见错误处理：**

```bash
# 如果初始化失败，可能缺少某些文件
# 方法1：从策略中排除不存在的文件
sudo twadmin -m -p > /tmp/twpol.txt
# 编辑 /tmp/twpol.txt，添加 !/nonexistent/path
sudo twadmin -m -P /tmp/twpol.txt -S /etc/tripwire/site.key

# 方法2：创建缺失的目录
sudo mkdir -p /missing/directory
```

---

### Step 3：检测文件篡改

```bash
# 首次检查（应该无变化）
sudo tripwire --check

# 输出示例：
# Open Source Tripwire(R) 2.4.3.7 built for x86_64-pc-linux-gnu
# 
# Tripwire Database located at /var/lib/tripwire/debian.twd
# Report created: Sat May 30 21:30:00 2026
# 
# No violations were found.

# 模拟文件篡改
echo "恶意内容" | sudo tee -a /var/www/html/index.html

# 修改文件权限
sudo chmod 777 /etc/passwd

# 创建新文件（模拟 Webshell）
echo "<?php system(\$_GET['cmd']); ?>" | sudo tee /var/www/html/shell.php

# 再次检查
sudo tripwire --check

# 输出示例（显示变更）：
# Rule Name: Web Content (Severity: High)
# Added:
# "/var/www/html/shell.php"
# 
# Modified:
# "/var/www/html/index.html"
# 
# Rule Name: System Configuration (Severity: Medium)
# Modified:
# "/etc/passwd"
```

---

### Step 4：分析监控报告

```bash
# 生成详细报告
sudo tripwire --check --email-report

# 查看存储的报告
sudo ls -la /var/lib/tripwire/report/

# 查看最新报告
REPORT=$(sudo ls -t /var/lib/tripwire/report/*.twr | head -1)
sudo twprint -m -r $REPORT

# 输出报告关键信息
sudo twprint -m -r $REPORT | grep -A 10 "Added:\|Modified:\|Removed:"
```

**报告解读：**

```
Tripwire 报告结构：

1. 报告头部信息
   ├── 扫描时间
   ├── 数据库位置
   └── 策略文件

2. 规则分组
   ├── 规则名称
   ├── 严重性级别
   └── 违规数量

3. 变更详情
   ├── Added: 新增文件
   ├── Removed: 删除文件
   └── Modified: 修改文件
       ├── 属性变更
       └── 哈希值变化
```

---

### Step 5：更新数据库（合法变更）

当变更合法时，需要更新数据库：

```bash
# 查看最新报告中的变更
sudo tripwire --check --interactive

# 这会打开编辑器，让你确认每个变更：
# Select [Y]es, [N]o, or [Q]uit:
# Y = 更新数据库
# N = 保持原状态
# Q = 退出

# 或者直接更新特定报告
sudo tripwire --update --twrfile $REPORT

# 查看更新后的数据库时间戳
sudo ls -la /var/lib/tripwire/*.twd
```

---

### Step 6：配置邮件告警

```bash
# 编辑配置文件启用邮件通知
sudo cat >> /etc/tripwire/twcfg.txt << EOF

# 邮件配置
MAILMETHOD       =SMTP
SMTPHOST         =localhost
SMTPPORT         =25
MAILPROGRAM      =/usr/sbin/sendmail -oi -t
EMAILREPORTLEVEL =3
REPORTLEVEL      =3
EOF

# 更新配置文件
sudo twadmin -m -C /etc/tripwire/twcfg.txt -S /etc/tripwire/site.key

# 测试邮件发送
sudo tripwire --test --email admin@example.com

# 在策略中配置邮件收件人
sudo cat >> /etc/tripwire/twpol.txt << 'EOF'
(
  rulename = "Critical Files",
  severity = $(SIG_HI),
  emailto = admin@example.com
)
{
  /etc/shadow -> $(SEC_CONFIG) ;
}
EOF

# 重新编译策略
sudo twadmin -m -P /etc/tripwire/twpol.txt -S /etc/tripwire/site.key
```

---

### Step 7：设置定时检查

```bash
# 创建 cron 任务
sudo cat > /etc/cron.daily/tripwire << 'EOF'
#!/bin/bash
# Tripwire 每日检查

/usr/sbin/tripwire --check --email-report > /var/log/tripwire.log 2>&1

# 检查执行结果
if [ $? -eq 0 ]; then
    logger -t tripwire "检查完成：无违规"
else
    logger -t tripwire "检查完成：发现违规，请查看邮件"
fi
EOF

# 设置执行权限
sudo chmod +x /etc/cron.daily/tripwire

# 或设置每小时检查
sudo cat > /etc/cron.hourly/tripwire << 'EOF'
#!/bin/bash
/usr/sbin/tripwire --check --email-report --quiet
EOF
sudo chmod +x /etc/cron.hourly/tripwire
```

---

## ✅ 验证检查

完成实验后，请确认你能回答以下问题：

- [ ] Tripwire 的工作原理是什么？
- [ ] 为什么需要先建立基线数据库？
- [ ] 如何区分合法变更和非法篡改？
- [ ] 策略文件中的 `!` 前缀表示什么？
- [ ] 为什么需要定期更新数据库？

---

## 💡 进阶挑战

### 挑战1：使用 inotify-tools 实时监控

Tripwire 是定期检查，而 inotify 可以实时监控：

```bash
# 安装
sudo apt install -y inotify-tools

# 实时监控 /var/www/html 目录
inotifywait -m -r /var/www/html -e modify,create,delete,move

# 输出示例：
# /var/www/html/ CREATE index.html
# /var/www/html/ MODIFY index.html

# 创建监控脚本
cat > monitor.sh << 'EOF'
#!/bin/bash

MONITOR_DIR="/var/www/html"
LOG_FILE="/var/log/file_monitor.log"

inotifywait -m -r --format '%w%f %e %T' --timefmt '%Y-%m-%d %H:%M:%S' \
    $MONITOR_DIR -e modify,create,delete,move,attrib \
    >> $LOG_FILE
EOF

# 后台运行
chmod +x monitor.sh
nohup ./monitor.sh &

# 查看日志
tail -f /var/log/file_monitor.log
```

### 挑战2：创建 Webhook 告警

```bash
#!/bin/bash
# webhook_alert.sh - 发送 Webhook 告警

WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
MESSAGE=$1

curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"🚨 Tripwire Alert: $MESSAGE\"}" \
    $WEBHOOK_URL

# 集成到 Tripwire
# 在策略文件中添加：
# emailto = |/path/to/webhook_alert.sh
```

### 挑战3：使用 AIDE（替代工具）

```bash
# AIDE 是另一个流行的文件完整性检查工具

# 安装
sudo apt install -y aide

# 初始化数据库
sudo aideinit

# 查看数据库位置
sudo ls -la /var/lib/aide/

# 检查变更
sudo aide --check

# 更新数据库
sudo aide --update
sudo cp /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# 配置文件
cat /etc/aide/aide.conf | head -50
```

### 挑战4：可视化监控面板

```python
#!/usr/bin/env python3
# file_monitor_dashboard.py

import os
import hashlib
import json
from datetime import datetime
from http.server import HTTPServer, BaseHTTPRequestHandler

MONITOR_DIR = "/var/www/html"
STATE_FILE = "file_state.json"

def calculate_hash(filepath):
    """计算文件哈希"""
    sha256 = hashlib.sha256()
    try:
        with open(filepath, 'rb') as f:
            for chunk in iter(lambda: f.read(8192), b''):
                sha256.update(chunk)
        return sha256.hexdigest()
    except:
        return None

def scan_directory():
    """扫描目录"""
    state = {}
    for root, dirs, files in os.walk(MONITOR_DIR):
        for file in files:
            filepath = os.path.join(root, file)
            relpath = os.path.relpath(filepath, MONITOR_DIR)
            state[relpath] = {
                'hash': calculate_hash(filepath),
                'size': os.path.getsize(filepath),
                'mtime': datetime.fromtimestamp(os.path.getmtime(filepath)).isoformat()
            }
    return state

def detect_changes():
    """检测变更"""
    current_state = scan_directory()
    
    try:
        with open(STATE_FILE, 'r') as f:
            old_state = json.load(f)
    except:
        old_state = {}
    
    changes = {
        'added': [],
        'removed': [],
        'modified': []
    }
    
    # 检测新增和修改
    for file, info in current_state.items():
        if file not in old_state:
            changes['added'].append(file)
        elif info['hash'] != old_state[file]['hash']:
            changes['modified'].append(file)
    
    # 检测删除
    for file in old_state:
        if file not in current_state:
            changes['removed'].append(file)
    
    # 保存当前状态
    with open(STATE_FILE, 'w') as f:
        json.dump(current_state, f, indent=2)
    
    return changes

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        changes = detect_changes()
        
        html = f"""
        <html>
        <head>
            <title>File Monitor Dashboard</title>
            <meta http-equiv="refresh" content="5">
        </head>
        <body>
            <h1>File Integrity Monitor</h1>
            <p>Monitoring: {MONITOR_DIR}</p>
            <p>Updated: {datetime.now().isoformat()}</p>
            
            <h2>Changes Detected</h2>
            <h3 style="color:green">Added ({len(changes['added'])})</h3>
            <ul>
                {"".join(f"<li>{f}</li>" for f in changes['added'])}
            </ul>
            
            <h3 style="color:red">Removed ({len(changes['removed'])})</h3>
            <ul>
                {"".join(f"<li>{f}</li>" for f in changes['removed'])}
            </ul>
            
            <h3 style="color:orange">Modified ({len(changes['modified'])})</h3>
            <ul>
                {"".join(f"<li>{f}</li>" for f in changes['modified'])}
            </ul>
        </body>
        </html>
        """
        
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(html.encode())

if __name__ == '__main__':
    server = HTTPServer(('localhost', 8080), Handler)
    print("Dashboard running at http://localhost:8080")
    server.serve_forever()
```

---

## 🛡️ 防御措施

### 文件完整性监控最佳实践

| 实践 | 说明 |
|------|------|
| **保护数据库** | 将数据库存储在只读介质或远程服务器 |
| **定期检查** | 至少每日检查一次关键文件 |
| **及时更新** | 合法变更后及时更新基线 |
| **告警机制** | 配置邮件或 Webhook 告警 |
| **日志审计** | 保留历史报告用于审计 |
| **多工具配合** | 结合实时监控和定期扫描 |

### 数据库安全存储

```bash
# 将数据库备份到只读存储
sudo cp /var/lib/tripwire/*.twd /mnt/read-only-backup/

# 使用远程存储
rsync -avz /var/lib/tripwire/ backup-server:/backup/tripwire/

# 只读挂载
sudo mount -o remount,ro /var/lib/tripwire/

# 检查前临时挂载为读写
sudo mount -o remount,rw /var/lib/tripwire/
sudo tripwire --check
sudo mount -o remount,ro /var/lib/tripwire/
```

---

## 📚 参考资源

- [Tripwire 官方文档](https://github.com/Tripwire/tripwire-open-source)
- [AIDE 文档](https://aide.github.io/)
- [inotify-tools](https://github.com/inotify-tools/inotify-tools/wiki)
- [NIST 文件完整性指南](https://csrc.nist.gov/publications/detail/sp/800-92/final)

---

## 📝 实验笔记

> 在这里记录你的实验过程、遇到的问题和解决方案：

```
实验日期：
监控的目标目录：
发现的文件变更：
配置的策略规则：
遇到的问题：
解决方案：
```

---

*最后更新：2026-05-30*
