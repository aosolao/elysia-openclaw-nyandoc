# OpenClaw Sandbox Image

[![Docker Pulls](https://img.shields.io/docker/pulls/aosolao/openclaw-sandbox.svg)](https://hub.docker.com/r/aosolao/openclaw-sandbox)
[![Image Size](https://img.shields.io/docker/image-size/aosolao/openclaw-sandbox/py313-node26-trixie)](https://hub.docker.com/r/aosolao/openclaw-sandbox/tags)
[![License](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-blue.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

## 📦 镜像简介

为 [OpenClaw](https://github.com/openclaw/openclaw) 设计的增强型沙箱镜像，基于 Debian Trixie，预装 Python 3.13、Node 26 和丰富的工具链，适用于 AI Agent 的安全隔离执行环境。

## ✨ 主要特性

- **双架构支持**：`linux/amd64` 和 `linux/arm64`（包括 Apple Silicon）
- **现代化运行时**：Python 3.13 + Node 26
- **浏览器自动化**：Playwright（Chromium + Firefox）+ Xvfb 反检测
- **金融数据分析**：pandas、akshare、yfinance、polars、duckdb
- **DevOps 工具**：gh、glab、kubectl、Grafana CLI、Jenkins CLI
- **安全隔离**：非 root 用户运行，UID 501 匹配宿主机 macOS 用户

## 🚀 快速开始

### 拉取镜像

```bash
docker pull aosolao/openclaw-sandbox:py313-node26-trixie
```

### 在 OpenClaw 中配置

```bash
# 设置沙箱镜像
openclaw config set agents.defaults.sandbox.docker.image "aosolao/openclaw-sandbox:py313-node26-trixie"

# 启用沙箱
openclaw config set agents.defaults.sandbox.mode "all"
openclaw config set agents.defaults.sandbox.scope "agent"

# 重启 Gateway
openclaw gateway restart
```

## 📊 镜像信息

- **镜像大小**：约 3.5 GB（压缩后约 1.2 GB）
- **基础镜像**：`nikolaik/python-nodejs:python3.13-nodejs26`
- **操作系统**：Debian 13 (Trixie)
- **最后更新**：2026-08-12
- **标签**：`py313-node26-trixie`
- **架构支持**：`linux/amd64`, `linux/arm64`

## ⚠️ 已知限制

- **无 GPU 支持**：镜像未包含 CUDA/cuDNN，不支持 GPU 加速
- **浏览器自动化**：已配置 Xvfb 虚拟显示器，支持 headful 模式反检测
- **非 root 运行**：使用 `sandbox` 用户（UID 501），某些系统级操作可能受限
- **中国镜像源**：默认使用清华/npmmirror 镜像，海外构建需通过 `--build-arg` 切换
- **Playwright 浏览器**：仅预装 Chromium 和 Firefox，未包含 WebKit

## 📝 更新日志

完整更新日志请访问 [GitHub Releases](https://github.com/aosolao/elysia-openclaw-nyandoc/releases)

## 🛠️ 预装工具

### Python 包

| 类别 | 工具 |
|------|------|
| **HTTP 客户端** | requests, httpx, aiohttp, urllib3 |
| **数据处理** | pandas, numpy, scipy, polars, pyarrow, duckdb |
| **金融数据** | yfinance, akshare, pandas-datareader |
| **网页解析** | beautifulsoup4, lxml, html5lib |
| **文件处理** | openpyxl, xlsxwriter, odfpy, python-docx, python-pptx, Pillow, pdfplumber, pypdf |
| **配置格式** | pyyaml, toml, python-dotenv |
| **终端/可视化** | rich, tqdm, click, typer, matplotlib, plotly |
| **浏览器自动化** | playwright, playwright-stealth, selenium, scrapy, fake-useragent |
| **加密/SSH** | cryptography, paramiko |
| **时间处理** | python-dateutil, pytz, tzdata |
| **其他** | pydantic, tenacity, cachetools, psutil, uv |

### 系统工具

- **CLI 工具**：gh (GitHub), glab (GitLab), kubectl, grafana, jenkins-cli
- **现代 CLI**：ripgrep, fd, bat, fzf, jq, yq, tree
- **文档处理**：pandoc, poppler-utils (pdftotext)
- **多媒体**：ffmpeg, imagemagick
- **数据库客户端**：sqlite3, postgresql-client, mysql-client, redis-tools
- **网络诊断**：curl, wget, openssl, net-tools, dnsutils

### 浏览器环境

- **Playwright 浏览器**：Chromium + Firefox（安装在 `/opt/ms-playwright`）
- **Xvfb 虚拟显示器**：自动启动，支持 headful 模式反检测
- **CJK 字体**：fonts-noto-cjk（中日韩字体支持）

## ⚙️ 配置说明

### 环境变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `TZ` | 时区 | `UTC` |
| `LANG` / `LC_ALL` | 字符编码 | `C.UTF-8` |
| `DISPLAY` | X11 显示 | `:99`（Xvfb 自动设置） |
| `PLAYWRIGHT_BROWSERS_PATH` | 浏览器路径 | `/opt/ms-playwright` |

### 用户权限

- **运行用户**：`sandbox` (UID 501)
- **工作目录**：`/home/sandbox`
- **启动命令**：`sleep infinity`（由 OpenClaw 通过 docker exec 注入命令）

### 挂载建议

```bash
docker run -d \
  --name openclaw-sandbox \
  -v ~/.openclaw/workspace:/workspace:rw \
  -v ~/.openclaw/workspace/openclaw-work:/workspace-project:rw \
  -e TZ=Asia/Shanghai \
  aosolao/openclaw-sandbox:py313-node26-trixie
```

## 🔨 自行构建

如需自定义镜像，可从源码构建：

```bash
# 克隆仓库
git clone https://github.com/aosolao/elysia-openclaw-nyandoc.git
cd elysia-openclaw-nyandoc/docker-sandbox

# 单架构构建
docker build -t my-openclaw-sandbox:latest .

# 多架构构建（需要 buildx）
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t my-openclaw-sandbox:latest \
  --push .
```

### 构建参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `APT_MIRROR` | Debian 软件源 | `https://mirrors.tuna.tsinghua.edu.cn` |
| `PYPI_MIRROR` | PyPI 镜像 | `https://pypi.tuna.tsinghua.edu.cn/simple` |
| `NPM_MIRROR` | npm 镜像 | `https://registry.npmmirror.com` |
| `PLAYWRIGHT_MIRROR` | Playwright CDN | `https://cdn.playwright.dev` |
| `TZ` | 时区 | `UTC` |

**海外构建示例**（使用官方源）：

```bash
docker build \
  --build-arg APT_MIRROR="http://deb.debian.org" \
  --build-arg PYPI_MIRROR="https://pypi.org/simple" \
  --build-arg NPM_MIRROR="https://registry.npmjs.org" \
  -t my-openclaw-sandbox:latest .
```

## 📚 文档

完整配置指南请访问项目仓库：

- **GitHub**：https://github.com/aosolao/elysia-openclaw-nyandoc
- **第 9 章 - 沙箱配置**：详细的沙箱配置说明
- **OpenClaw 官方文档**：https://docs.openclaw.ai

## 📄 许可证

本镜像的 Dockerfile 和文档基于 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可证发布。

预装的软件包遵循各自的开源许可证。

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

---

**Made with ❤️ for the OpenClaw community**
