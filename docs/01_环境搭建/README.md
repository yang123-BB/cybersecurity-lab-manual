# 🛠️ 环境搭建指南

> 在开始实验之前，需要先准备好实验环境。本文档将指导你搭建完整的网络安全实验环境。

---

## 📋 环境方案对比

| 方案 | 优点 | 缺点 | 推荐指数 |
|------|------|------|---------|
| **方案A：Kali Linux 虚拟机** | 工具齐全，最接近实战 | 占用资源多 | ⭐⭐⭐⭐⭐ |
| **方案B：Docker 容器** | 轻量，快速启动 | 网络配置复杂 | ⭐⭐⭐⭐ |
| **方案C：本地 Ubuntu 安装工具** | 简单 | 需要手动安装工具 | ⭐⭐⭐ |
| **方案D：在线实验平台** | 无需本地环境 | 功能受限 | ⭐⭐ |

---

## 🔧 方案A：Kali Linux 虚拟机（推荐）

### 步骤1：安装虚拟机软件

下载并安装以下其中之一：
- **VirtualBox**（免费）：https://www.virtualbox.org/
- **VMware Workstation**：https://www.vmware.com/

### 步骤2：下载 Kali Linux 镜像

- 官方下载：https://www.kali.org/get-kali/
- 推荐：**Kali Linux 64-bit VM** （预装虚拟机镜像，直接导入即可）

### 步骤3：导入虚拟机

**VirtualBox 用户：**
1. 打开 VirtualBox
2. 点击「导入虚拟电脑」
3. 选择下载的 `.ova` 文件
4. 等待导入完成

**VMware 用户：**
1. 打开 VMware Workstation
2. 选择「打开虚拟机」
3. 选择下载的 `.vmx` 文件

### 步骤4：启动并登录

```
默认用户名：kali
默认密码：kali
```

### 步骤5：更新系统并安装工具

```bash
# 更新系统
sudo apt update && sudo apt full-upgrade -y

# 安装本课程所需全部工具
sudo apt install -y \
    nmap \
    hydra \
    john \
    wireshark \
    netcat-openbsd \
    openssl \
    tripwire \
    steghide \
    nikto \
    masscan \
    curl \
    wget \
    metasploit-framework

# 验证安装
nmap --version
hydra -h | head -5
john --help | head -5
```

---

## 🐳 方案B：Docker 容器（轻量）

### 快速启动

```bash
# 拉取 Kali Linux 官方镜像
docker pull kalilinux/kali-rolling

# 启动交互式容器
docker run -it --name sec-lab kalilinux/kali-rolling /bin/bash

# 在容器内安装工具
apt update && apt install -y nmap hydra john wireshark nikto
```

### Docker Compose 完整环境

创建 `docker-compose.yml`：

```yaml
version: '3'
services:
  kali:
    image: kalilinux/kali-rolling
    container_name: sec-lab
    hostname: kali-lab
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
    networks:
      - sec-net
    volumes:
      - ./workspace:/workspace
    tty: true
    stdin_open: true
    command: tail -f /dev/null

  metasploitable:
    image: tleemcjr/metasploitable2
    container_name: target-machine
    hostname: metasploitable
    networks:
      - sec-net
    command: /bin/bash

networks:
  sec-net:
    driver: bridge
```

启动：
```bash
docker-compose up -d
docker exec -it sec-lab /bin/bash
```

---

## 🎯 方案C：本地 Ubuntu 安装工具

```bash
# 添加 Kali 仓库（可选，获取最新工具）
sudo tee /etc/apt/sources.list.d/kali.list << EOF
deb http://http.kali.org/kali kali-rolling main non-free contrib
EOF

sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ED444FF07D8D0BF6

# 安装工具
sudo apt update
sudo apt install -y nmap hydra john wireshark nikto masscan steghide

# 安装 OpenSSL（通常已预装）
openssl version
```

---

## ✅ 环境验证清单

完成环境搭建后，请运行以下命令验证：

```bash
echo "=== 环境验证 ==="
echo "Nmap: $(nmap --version | head -1)"
echo "Hydra: $(hydra -h | head -1)"
echo "John: $(john --help | head -1)"
echo "Wireshark: $(wireshark --version 2>/dev/null | head -1 || echo '需要GUI')"
echo "Netcat: $(which nc)"
echo "OpenSSL: $(openssl version)"
echo "Nikto: $(nikto -Version 2>/dev/null | head -1 || echo '请检查安装')"
echo "Masscan: $(masscan --version 2>/dev/null || echo '请检查安装')"
echo "Steghide: $(steghide --version 2>/dev/null || echo '请检查安装')"
```

预期输出示例：
```
=== 环境验证 ===
Nmap: Nmap version 7.94
Hydra: Hydra v9.4
John: John the Ripper
...
```

---

## 🏗️ 搭建靶机环境（可选但推荐）

为了安全地进行实验，建议搭建一个**隔离的靶机环境**。

### 推荐靶机

| 靶机 | 描述 | 下载地址 |
|------|------|---------|
| **Metasploitable2** | 故意存在漏洞的 Linux | [SourceForge](https://sourceforge.net/projects/metasploitable/) |
| **DVWA** | Web 漏洞练习平台 | [GitHub](https://github.com/digininja/DVWA) |
| **bWAPP** | Buggy Web Application | [SourceForge](https://sourceforge.net/projects/bwapp/) |
| **VulnHub 系列** | 各种漏洞靶机 | [VulnHub](https://www.vulnhub.com/) |

### 快速启动 DVWA（Docker）

```bash
docker run -d -p 80:80 vulnerables/web-dvwa
# 访问 http://localhost
# 默认账号：admin / password
```

---

## 📝 下一步

环境搭建完成后，请继续学习：

→ [实验01：使用 Hydra 破解密码](../02_密码破解/01_hydra_password_cracking.md)

---

## 💡 常见问题

### Q1：虚拟机网络怎么配置？
**A**：推荐使用「NAT 模式」或「桥接模式」，确保攻击机和靶机在同一网段。

### Q2：Wireshark 需要图形界面吗？
**A**：可以使用命令行版本 `tshark`，或安装 Wireshark GUI 版。

### Q3：Docker 容器能进行漏洞扫描吗？
**A**：可以，但需要添加 `--cap-add=NET_ADMIN` 等权限，并正确配置网络。

### Q4：Windows 上能直接运行这些工具吗？
**A**：部分可以（如 Nmap、Wireshark 有 Windows 版），但推荐在 Linux 环境下学习。

---

*最后更新：2026-05-30*
