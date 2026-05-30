# 🔐 网络安全实验课件

> 基于 LabEx 技能树体系，使用 Markdown 编写的网络安全实验教程

## 📚 课程大纲

本课程共包含 **19个实验项目**，分为6个模块，帮助你系统学习网络安全核心工具与技术。

---

## 🗂️ 模块目录

| 模块 | 内容 | 实验数 |
|------|------|--------|
| [01_环境搭建](./01_环境搭建/README.md) | 实验环境准备与工具安装 | 1 |
| [02_密码破解](./02_密码破解/) | Hydra & John the Ripper | 3 |
| [03_网络扫描](./03_网络扫描/) | Nmap & Masscan & Nikto | 5 |
| [04_流量分析](./04_流量分析/) | Wireshark 抓包与分析 | 3 |
| [05_网络工具](./05_网络工具/) | Netcat 网络通信 | 2 |
| [06_加密与安全](./06_加密与安全/) | OpenSSL & Tripwire & Steghide | 5 |
| [07_综合实战](./07_综合实战/) | 多工具组合实战演练 | 1 |

---

## 📋 实验列表

### 🔐 密码破解
- [ ] [实验01：使用 Hydra 破解密码](./02_密码破解/01_hydra_password_cracking.md)
- [ ] [实验02：破解特定用户账户](./02_密码破解/02_hydra_targeted_user.md)
- [ ] [实验15：使用 John the Ripper 破解 ZIP 密码](./02_密码破解/03_john_zip_cracking.md)

### 🌐 网络扫描
- [ ] [实验03：使用 Nmap 进行网络扫描](./03_网络扫描/01_nmap_basic.md)
- [ ] [实验04：使用 Nmap 扫描子网](./03_网络扫描/02_nmap_subnet.md)
- [ ] [实验12：使用 Nmap 扫描漏洞](./03_网络扫描/03_nmap_vuln.md)
- [ ] [实验17：使用 Masscan 扫描端口](./03_网络扫描/04_masscan.md)
- [ ] [实验19：使用 Nikto 扫描 Web 服务器](./03_网络扫描/05_nikto.md)

### 📡 流量分析
- [ ] [实验09：使用 Wireshark 进行网络分析](./04_流量分析/01_wireshark_basic.md)
- [ ] [实验10：使用 Wireshark 捕获 Google 流量](./04_流量分析/02_wireshark_google.md)
- [ ] [实验14：在 Wireshark 中过滤流量](./04_流量分析/03_wireshark_filters.md)

### 💬 网络工具
- [ ] [实验07：使用 Netcat 进行简单的网络通信](./05_网络工具/01_netcat_basic.md)
- [ ] [实验08：使用 Netcat 接收消息](./05_网络工具/02_netcat_listener.md)

### 🛡️ 加密与安全
- [ ] [实验05：OpenSSL 加密入门](./06_加密与安全/01_openssl_intro.md)
- [ ] [实验06：解密绝密文件](./06_加密与安全/02_openssl_decrypt.md)
- [ ] [实验13：使用 OpenSSL 加密文件](./06_加密与安全/03_openssl_encrypt.md)
- [ ] [实验16：使用 Tripwire 监控文件](./06_加密与安全/04_tripwire.md)
- [ ] [实验18：使用 Steghide 隐藏数据](./06_加密与安全/05_steghide.md)

### 🚀 综合实战
- [ ] [综合实战：完整渗透测试流程](./07_综合实战/01_comprehensive.md)

---

## 🎯 学习目标

通过本课程，你将掌握：

- ✅ 密码破解工具的使用（Hydra、John the Ripper）
- ✅ 网络扫描与漏洞发现（Nmap、Masscan、Nikto）
- ✅ 网络流量分析与抓包（Wireshark）
- ✅ 网络通信工具（Netcat）
- ✅ 加密、隐写与文件完整性监控（OpenSSL、Steghide、Tripwire）

---

## 🖥️ 环境要求

- **操作系统**：Kali Linux（推荐）或 Ubuntu 20.04+
- **硬件**：2核CPU / 4GB内存 / 20GB硬盘（最低配置）
- **网络**：可访问互联网

### 快速安装（Kali Linux）

```bash
sudo apt update
sudo apt install -y nmap hydra john wireshark netcat-openbsd \
                   openssl tripwire steghide nikto masscan
```

---

## 📖 使用说明

1. **按顺序学习**：建议从模块01开始，循序渐进
2. **动手实践**：每个实验都有具体的命令和步骤，请亲自执行
3. **记录笔记**：建议在每个实验目录下创建 `notes.md` 记录心得
4. **完成挑战**：每个实验末尾都有进阶挑战题目

---

## 🚀 发布为在线文档

本课件支持发布为在线文档，推荐使用以下平台：

### 方案A：MkDocs + GitHub Pages（推荐）

```bash
# 安装 MkDocs + Material 主题
pip install mkdocs mkdocs-material

# 本地预览
mkdocs serve
# 访问 http://localhost:8000

# 构建静态网站
mkdocs build
# 生成的文件在 site/ 目录

# 部署到 GitHub Pages
mkdocs gh-deploy
```

### 方案B：打包分享（最简单）

```bash
# 构建静态网站
mkdocs build

# 打包 site/ 目录
cd site
zip -r ../网络安全实验课件_网站版.zip *
# 分享 zip 文件给学生
# 学生解压后，双击 index.html 即可在浏览器中查看
```

---

## ⚠️ 法律声明

> **本课件仅供学习与研究使用！**
>
> - 所有实验请在**授权环境**或**自己的设备**上进行
> - 未经授权对他人系统进行扫描、破解属于**违法行为**
> - 请遵守《网络安全法》及相关法律法规

---

## 📚 参考资源

- [Kali Linux 官方文档](https://www.kali.org/docs/)
- [Nmap 官方指南](https://nmap.org/book/)
- [Wireshark 用户指南](https://www.wireshark.org/docs/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [TryHackMe 免费实验](https://tryhackme.com/)
- [Hack The Box](https://www.hackthebox.com/)

---

## 📝 贡献与反馈

欢迎提交 Issue 和 Pull Request 来改进本课件！

---

## 📄 许可证

CC BY-NC-SA 4.0（署名-非商业性使用-相同方式共享）

---

*最后更新：2026-05-30*
