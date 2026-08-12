# OpenClaw Sandbox Image

> 版本：v1.0.0
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](../LICENSE)

> 📖 沙箱镜像详细文档

<!-- TOC START -->
## 目录

- [📦 镜像信息](#-镜像信息)
- [🛠️ 预装软件与工具](#️-预装软件与工具)
  - [系统工具（apt 安装）](#系统工具apt-安装)
  - [额外下载的工具](#额外下载的工具)
  - [Python 运行时](#python-运行时)
  - [Node.js 运行时](#nodejs-运行时)
  - [Python 包（通过 uv 安装）](#python-包通过-uv-安装)
  - [Playwright 浏览器](#playwright-浏览器)
- [🔧 使用样例](#-使用样例)
  - [1. 数据获取与分析](#1-数据获取与分析)
  - [2. 网页抓取](#2-网页抓取)
  - [3. 文件处理](#3-文件处理)
  - [4. 命令行工具](#4-命令行工具)
  - [5. Jenkins CLI](#5-jenkins-cli)
- [⚠️ 注意事项](#️-注意事项)
  - [1. 非 root 用户](#1-非-root-用户)
  - [2. 浏览器自动化](#2-浏览器自动化)
  - [3. 镜像源](#3-镜像源)
  - [4. 时区](#4-时区)
  - [5. Playwright 浏览器路径](#5-playwright-浏览器路径)
  - [6. 无 GPU 支持](#6-无-gpu-支持)
  - [7. Jenkins CLI](#7-jenkins-cli)
- [🔨 自行构建](#-自行构建)
  - [单架构构建](#单架构构建)
  - [多架构构建](#多架构构建)
  - [构建参数](#构建参数)
- [📚 相关文档](#-相关文档)

<!-- TOC END -->

---

## 📦 镜像信息

| 项目 | 说明 |
|------|------|
| **镜像名称** | `aosolao/openclaw-sandbox:py313-node26-trixie` |
| **基础镜像** | `nikolaik/python-nodejs:python3.13-nodejs26` |
| **操作系统** | Debian 13 (Trixie) |
| **架构支持** | `linux/amd64`, `linux/arm64` |
| **镜像大小** | 约 3.5 GB（压缩后约 1.2 GB）*估算值* |
| **运行用户** | `sandbox` (UID 501) |
| **工作目录** | `/home/sandbox` |
| **启动命令** | `sleep infinity` |

---

## 🛠️ 预装软件与工具

### 系统工具（apt 安装）

| 工具 | 路径 | 说明 |
|------|------|------|
| `git` | `/usr/bin/git` | 版本控制 |
| `openssh-client` | `/usr/bin/ssh` | SSH 客户端 |
| `curl` | `/usr/bin/curl` | HTTP 客户端 |
| `wget` | `/usr/bin/wget` | 下载工具 |
| `jq` | `/usr/bin/jq` | JSON 处理 |
| `yq` | `/usr/bin/yq` | YAML 处理 |
| `tree` | `/usr/bin/tree` | 目录树显示 |
| `less` | `/usr/bin/less` | 分页查看 |
| `file` | `/usr/bin/file` | 文件类型检测 |
| `nano` | `/usr/bin/nano` | 文本编辑器 |
| `rsync` | `/usr/bin/rsync` | 文件同步 |
| `zip` | `/usr/bin/zip` | 压缩工具 |
| `unzip` | `/usr/bin/unzip` | 解压工具 |
| `bzip2` | `/usr/bin/bzip2` | 压缩工具 |
| `xz-utils` | `/usr/bin/xz` | 压缩工具 |
| `7zip` | `/usr/bin/7z` | 压缩工具 |
| `htop` | `/usr/bin/htop` | 进程监控 |
| `tmux` | `/usr/bin/tmux` | 终端复用 |
| `fzf` | `/usr/bin/fzf` | 模糊搜索 |
| `ffmpeg` | `/usr/bin/ffmpeg` | 多媒体处理 |
| `imagemagick` | `/usr/bin/convert` | 图像处理 |
| `sqlite3` | `/usr/bin/sqlite3` | SQLite 客户端 |
| `postgresql-client` | `/usr/bin/psql` | PostgreSQL 客户端 |
| `default-mysql-client` | `/usr/bin/mysql` | MySQL 客户端 |
| `redis-tools` | `/usr/bin/redis-cli` | Redis 客户端 |
| `openssl` | `/usr/bin/openssl` | 加密工具 |
| `net-tools` | `/usr/bin/netstat` | 网络工具 |
| `dnsutils` | `/usr/bin/dig` | DNS 工具 |
| `gh` | `/usr/bin/gh` | GitHub CLI |
| `ripgrep` | `/usr/bin/rg` | 快速搜索 |
| `fd-find` | `/usr/bin/fd` | 快速查找（链接到 `/usr/local/bin/fd`） |
| `bat` | `/usr/bin/bat` | 增强版 cat（链接到 `/usr/local/bin/bat`） |
| `pandoc` | `/usr/bin/pandoc` | 文档格式转换 |
| `poppler-utils` | `/usr/bin/pdftotext` | PDF 工具 |
| `xvfb` | `/usr/bin/Xvfb` | 虚拟显示器 |
| `xauth` | `/usr/bin/xauth` | X11 认证 |
| `dbus-x11` | `/usr/bin/dbus-launch` | D-Bus X11 支持 |
| `fonts-noto-cjk` | `/usr/share/fonts/` | 中日韩字体 |
| `fonts-noto-color-emoji` | `/usr/share/fonts/` | Emoji 字体 |
| `fonts-liberation` | `/usr/share/fonts/` | Liberation 字体 |

### 额外下载的工具

| 工具 | 路径 | 版本 | 说明 |
|------|------|------|------|
| `glab` | `/usr/bin/glab` | 1.113.0 | GitLab CLI |
| `kubectl` | `/usr/local/bin/kubectl` | v1.36.3 | Kubernetes CLI |
| `grafana` | `/usr/local/bin/grafana` | 13.1.3 | Grafana 服务端（含 CLI） |
| `jenkins-cli` | `/usr/local/bin/jenkins-cli` | 动态下载 | Jenkins CLI（包装脚本） |

### Python 运行时

| 项目 | 路径 | 说明 |
|------|------|------|
| Python 3.13 | `/usr/local/bin/python3` | 基础镜像提供 |
| pip | `/usr/local/bin/pip` | 包管理器 |
| uv | `/usr/local/bin/uv` | 快速包管理器 |

### Node.js 运行时

| 项目 | 路径 | 说明 |
|------|------|------|
| Node.js 26 | `/usr/local/bin/node` | 基础镜像提供 |
| npm | `/usr/local/bin/npm` | 包管理器 |
| npx | `/usr/local/bin/npx` | 包执行器 |

### Python 包（通过 uv 安装）

| 类别 | 包名 | 说明 |
|------|------|------|
| **HTTP 客户端** | `requests`, `httpx`, `aiohttp`, `urllib3` | HTTP 请求库 |
| **数据处理** | `pandas`, `numpy`, `scipy`, `polars`, `pyarrow`, `duckdb` | 数据分析与处理 |
| **金融数据** | `yfinance`, `akshare`, `pandas-datareader` | 金融数据获取 |
| **网页解析** | `beautifulsoup4`, `lxml`, `html5lib` | HTML/XML 解析 |
| **文件处理** | `openpyxl`, `xlsxwriter`, `odfpy`, `python-docx`, `python-pptx`, `Pillow`, `pdfplumber`, `pypdf` | Office 文档与 PDF 处理 |
| **配置格式** | `pyyaml`, `toml`, `python-dotenv` | 配置文件解析 |
| **终端/可视化** | `rich`, `tqdm`, `click`, `typer`, `matplotlib`, `plotly` | 终端美化与数据可视化 |
| **浏览器自动化** | `playwright`, `playwright-stealth`, `selenium`, `scrapy`, `fake-useragent` | 网页抓取与自动化 |
| **加密/SSH** | `cryptography`, `paramiko` | 加密与 SSH |
| **时间处理** | `python-dateutil`, `pytz`, `tzdata` | 日期时间处理 |
| **其他** | `pydantic`, `tenacity`, `cachetools`, `psutil` | 数据验证、重试、缓存、系统监控 |

### Playwright 浏览器

| 浏览器 | 路径 | 说明 |
|--------|------|------|
| Chromium | `/opt/ms-playwright/chromium-*` | 无头浏览器 |
| Firefox | `/opt/ms-playwright/firefox-*` | 无头浏览器 |

---

## 🔧 使用样例

### 1. 数据获取与分析

```python
# 使用 akshare 获取 A 股数据
import akshare as ak

# 获取上证指数实时行情
df = ak.stock_zh_index_spot_em()
print(df.head())

# 获取个股历史数据
df = ak.stock_zh_a_hist(symbol="000001", period="daily", start_date="20240101")
print(df.tail())
```

### 2. 网页抓取

```python
# 使用 Playwright 抓取网页
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)  # headful 模式，自动使用 Xvfb
    page = browser.new_page()
    page.goto("https://example.com")
    print(page.title())
    browser.close()
```

### 3. 文件处理

```python
# 读取 Excel 文件
import pandas as pd

df = pd.read_excel("data.xlsx")
print(df.head())

# 合并 PDF 文件
from pypdf import PdfReader, PdfWriter

output = PdfWriter()
for pdf_path in ["file1.pdf", "file2.pdf"]:
    pdf = PdfReader(pdf_path)
    for page in pdf.pages:
        output.add_page(page)

with open("merged.pdf", "wb") as f:
    output.write(f)
```

### 4. 命令行工具

```bash
# 使用 ripgrep 搜索代码
rg "pattern" /workspace

# 使用 fd 查找文件
fd "*.py" /workspace

# 使用 bat 查看文件（带语法高亮）
bat /workspace/script.py

# 使用 jq 处理 JSON
cat data.json | jq '.items[]'

# 使用 kubectl 查看集群
kubectl get pods -n default

# 使用 gh 查看 GitHub 仓库
gh repo view aosolao/elysia-openclaw-nyandoc
```

### 5. Jenkins CLI

```bash
# 首次使用需要设置 JENKINS_URL 环境变量
export JENKINS_URL="http://jenkins:8080"

# 查看帮助
jenkins-cli help

# 查看任务列表
jenkins-cli list-jobs
```

---

## ⚠️ 注意事项

### 1. 非 root 用户

镜像以 `sandbox` 用户（UID 501）运行，某些系统级操作可能受限。如需 root 权限，可通过 OpenClaw 的 elevated 模式逃逸到宿主机。

### 2. 浏览器自动化

- 浏览器以 **headful 模式**运行（通过 Xvfb 虚拟显示器），消除 headless 指纹
- `DISPLAY=:99` 环境变量已自动设置
- 启动浏览器时使用 `headless=False`：

```python
browser = p.chromium.launch(headless=False)  # 正确
browser = p.chromium.launch(headless=True)   # 不推荐，可能被检测
```

### 3. 镜像源

默认使用中国镜像源（清华 TUNA、npmmirror），海外构建需通过 `--build-arg` 切换：

```bash
docker build \
  --build-arg APT_MIRROR="http://deb.debian.org" \
  --build-arg PYPI_MIRROR="https://pypi.org/simple" \
  --build-arg NPM_MIRROR="https://registry.npmjs.org" \
  -t my-sandbox:latest .
```

### 4. 时区

默认时区为 UTC，运行时可通过环境变量覆盖：

```bash
docker run -e TZ=Asia/Shanghai aosolao/openclaw-sandbox:py313-node26-trixie
```

### 5. Playwright 浏览器路径

浏览器安装在 `/opt/ms-playwright`，环境变量 `PLAYWRIGHT_BROWSERS_PATH` 已设置。使用 `python -m playwright` 调用（不要用 `npx playwright`，版本可能不一致）。

### 6. 无 GPU 支持

镜像未包含 CUDA/cuDNN，不支持 GPU 加速的机器学习任务。

### 7. Jenkins CLI

首次使用 `jenkins-cli` 时，需要设置 `JENKINS_URL` 环境变量，脚本会自动下载 `jenkins-cli.jar`。

---

## 🔨 自行构建

### 单架构构建

```bash
docker build -t my-openclaw-sandbox:latest .
```

### 多架构构建

```bash
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
| `GLAB_VERSION` | GitLab CLI 版本 | `1.113.0` |
| `KUBECTL_VERSION` | kubectl 版本 | `v1.36.3` |
| `GRAFANA_VERSION` | Grafana 版本 | `13.1.3` |

---

## 📚 相关文档

- [Dockerfile](./Dockerfile) - 镜像构建脚本
- [requirements.txt](./requirements.txt) - Python 包清单
- [第 9 章 - 沙箱配置](../09-sandbox.md) - OpenClaw 沙箱配置说明

---

**最后更新**：2026-08-12  
**维护者**：[aosolao](https://github.com/aosolao)
