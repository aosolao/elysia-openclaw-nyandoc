# 第 9 章 · 沙箱与隔离环境

> 版本：v1.2.1
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [9.1 沙箱是什么、不是什么](#91-沙箱是什么不是什么)
- [9.2 三个核心参数](#92-三个核心参数)
  - [mode（何时沙箱化）](#mode何时沙箱化)
  - [scope（容器粒度）](#scope容器粒度)
  - [backend（运行后端）](#backend运行后端)
- [9.3 工作区访问](#93-工作区访问)
- [9.4 这台机器的沙箱配置](#94-这台机器的沙箱配置)
  - [镜像](#镜像)
  - [完整配置](#完整配置)
  - [PATH 修复（必做）](#path-修复必做)
  - [exec host 必须改](#exec-host-必须改)
- [9.5 自定义镜像构建](#95-自定义镜像构建)
  - [关键规则](#关键规则)
  - [构建与部署](#构建与部署)
  - [验证](#验证)
- [9.6 Bind mount 安全根限制](#96-bind-mount-安全根限制)
- [9.7 Elevated 逃逸](#97-elevated-逃逸)
- [9.8 沙箱内不可用的功能](#98-沙箱内不可用的功能)
- [9.9 常见问题](#99-常见问题)
  - [Q: 改了配置但 exec 还在宿主机跑？](#q-改了配置但-exec-还在宿主机跑)
  - [Q: 容器内 python3 找不到包？](#q-容器内-python3-找不到包)
  - [Q: Chromium 启动崩溃或白屏？](#q-chromium-启动崩溃或白屏)
  - [Q: 容器内访问不了外网？](#q-容器内访问不了外网)
  - [Q: 新移入的 skill 在容器内看不到？](#q-新移入的-skill-在容器内看不到)

<!-- TOC END -->

前 8 章里，exec 命令直接跑在你的宿主机上——即使有审批白名单，模型一旦被诱导或误判，命令仍以你的用户身份触碰所有文件。沙箱（Sandbox）是第四层防护：把工具执行关进容器，缩小"模型搞砸了"时的爆炸半径。

## 9.1 沙箱是什么、不是什么

**是什么**：OpenClaw 把 `exec`/`read`/`write`/`edit`/`apply_patch`/`process` 等工具的执行搬进 Docker 容器（或 SSH/OpenShell 远程环境），容器有独立的文件系统、进程空间和网络栈。

**不是什么**：它不是完美的安全边界。Docker 容器逃逸漏洞、挂载进来的宿主机目录、显式开启的 elevated 逃逸，都能突破容器隔离。沙箱的定位是**降低意外和低级别恶意的破坏面**，不能替代审批制和工具权限控制。

**不受沙箱影响的**：

- Gateway 进程本身始终在宿主机上。
- `tools.elevated` 是显式逃逸出口——审批通过后命令直接在宿主机执行，跳过沙箱。
- 浏览器工具走宿主机 Chrome CDP（见第 6 章），不是容器内浏览器。

## 9.2 三个核心参数

| 参数 | 可选值 | 默认 | 作用 |
|---|---|---|---|
| `mode` | `off` / `non-main` / `all` | `off` | **何时**沙箱化 |
| `scope` | `agent` / `session` / `shared` | `agent` | **容器粒度** |
| `backend` | `docker` / `ssh` / `openshell` | `docker` | **在哪跑** |

### mode（何时沙箱化）

- `off`：全部跑宿主机（无隔离）。
- `non-main`：除主会话（你和 AI 的私聊）外，群聊/频道/子会话全部进沙箱。
- `all`：所有会话都进沙箱，包括主会话。

推荐：单人本地使用选 `all`，隔离最彻底；宿主机操作通过 elevated 逃逸完成。

### scope（容器粒度）

- `agent`：每个 agent 一个容器，该 agent 的多个会话复用同一个容器。启动快，但会话间有文件/进程残留。
- `session`：每个会话一个全新容器，最干净，但每次新会话启动慢几秒。
- `shared`：所有沙箱会话共用一个容器。资源最省，隔离最弱。

推荐：`agent`——平衡了启动速度和隔离度。

### backend（运行后端）

- `docker`：本地 Docker 容器（推荐，功能最全，支持沙箱浏览器）。
- `ssh`：任意 SSH 可达的远程主机，适合把计算卸载到另一台机器。
- `openshell`：OpenShell 托管的远程沙箱，支持镜像同步。

macOS 本地使用选 `docker`（需要 OrbStack 或 Docker Desktop）。

## 9.3 工作区访问

`agents.defaults.sandbox.workspaceAccess` 控制容器能看到什么：

| 值 | 行为 | 适用场景 |
|---|---|---|
| `none` | 容器用独立的临时目录，完全看不到你的 workspace | 最高隔离，跑不信任的代码 |
| `ro` | workspace 只读挂载到 `/agent`，写工具被禁用 | 需要读但不希望被改 |
| `rw` | workspace 读写挂载到 `/workspace` | 日常使用，AI 需要读写记忆和项目文件 |

推荐 `rw`，否则 AI 没法更新记忆、编辑项目文件。

## 9.4 这台机器的沙箱配置

### 镜像

使用自定义镜像 `aosolao/openclaw-sandbox:py313-node26-trixie`（作者的公开镜像，读者可根据自身需求构建或替换），相比 OpenClaw 默认镜像（`debian:bookworm-slim + python3`）额外预装：

- Python 3.13 + Node 26（基于 `nikolaik/python-nodejs`）
- 金融数据栈：pandas、akshare、yfinance、polars、duckdb 等
- 爬虫/自动化：Playwright（Chromium + Firefox）、Selenium、Scrapy、playwright-stealth
- 反检测：Xvfb 虚拟显示器（headful 模式）、Noto CJK 字体、Liberation 字体
- 文档处理、多媒体、数据库客户端、现代 CLI 工具等
- 时区：镜像默认 UTC，运行时由 `docker.env.TZ` 注入（这台机器设为 `Asia/Shanghai`）
- 编码：`LANG=C.UTF-8`，中文文件名和输出正常

Dockerfile 和 requirements.txt 在 `~/.openclaw/workspace/openclaw-work/docs/docker-sandbox/`。

### 完整配置

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
        backend: "docker",
        workspaceAccess: "rw",
        docker: {
          image: "aosolao/openclaw-sandbox:py313-node26-trixie",
          network: "bridge",
          readOnlyRoot: false,
          tmpfs: ["/tmp", "/dev/shm:size=2g"],
          env: {
            HTTP_PROXY: "http://host.docker.internal:6152",
            HTTPS_PROXY: "http://host.docker.internal:6152",
            NO_PROXY: "localhost,127.0.0.1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16",
            TZ: "Asia/Shanghai"
          }
        }
      }
    }
  }
}
```

逐项说明：

| 配置 | 值 | 为什么 |
|---|---|---|
| `network` | `bridge` | 默认 `none` 完全无网络，金融数据接口和爬虫无法工作 |
| `readOnlyRoot` | `false` | 先关闭以保证 pip/npm 临时写正常；稳定后可改 true 收紧 |
| `tmpfs` | `/tmp` + `/dev/shm:2g` | Playwright/Chromium 在容器里默认 64MB /dev/shm 会崩溃，必须加大 |
| `env.HTTP_PROXY/HTTPS_PROXY` | `http://host.docker.internal:6152` | Mac/Windows Docker Desktop/OrbStack 下容器通过此域名访问宿主机代理；Linux 原生 Docker 需额外加 `--add-host=host.docker.internal:host-gateway` |
| `NO_PROXY` | 含私有网段 | 访问内网/自建 GitLab 等不走代理 |
| `env.TZ` | `Asia/Shanghai` | 容器时区；镜像默认 UTC，运行时注入，不同国家用户改自己的配置即可 |
| `LANG`/`LC_ALL` | `C.UTF-8`（镜像内置） | 全局 UTF-8，中文文件名和输出不乱码 |

### PATH 修复（必做）

OpenClaw 会把宿主机 PATH 注入容器，导致 Debian 系统自带的 `/usr/bin/python3`（空包，没装 pandas 等）抢在 `/usr/local/bin/python3`（我们装了所有包）前面。必须配置：

```bash
openclaw config set tools.exec.pathPrepend '["/usr/local/bin"]' --strict-json
```

### exec host 必须改

`tools.exec.host` 默认为 `gateway` 时，沙箱配置不生效。必须改为 `auto`：

```bash
openclaw config set tools.exec.host auto
```

改后需要重启 gateway 或等新会话才生效。

## 9.5 自定义镜像构建

### 关键规则

1. **sandbox 用户 UID 必须匹配宿主机**（Mac 默认 501），否则挂载的 workspace 文件权限错乱：

   ```dockerfile
   RUN useradd --uid 501 --create-home --shell /bin/bash sandbox
   ```

2. **CMD 必须是常驻进程**，不能是 `bash`（无 TTY 时立即退出导致容器停止）：

   ```dockerfile
   CMD ["sleep", "infinity"]
   ```

3. **Python 包分层利用缓存**：uv 升级层、requirements.txt 安装层、Playwright 浏览器层各自独立，改一个不牵连其他重装。

4. **`uv pip install` 必须加 `--system`**：容器内没有激活虚拟环境，不加的话包装到 venv 里，运行时找不到。

5. **uv requirements.txt 每行只能一个包**：`requests httpx` 这种空格分隔写法 pip 容忍但 uv 严格拒绝。

6. **关键环境变量写进 `/etc/profile.d/`**：`su -` / `sh -lc` 登录 shell 会清掉 Docker `ENV`，只靠 `ENV PLAYWRIGHT_BROWSERS_PATH=...` 不够。

7. **Playwright headful 反检测**：容器无显示器，用 Xvfb 在内存里渲染虚拟桌面，Chromium 以 `headless=False` 启动，消除 `HeadlessChrome` UA 等指纹。启动脚本必须 `mkdir -p /tmp/.X11-unix && chmod 1777`。

### 构建与部署

```bash
cd ~/.openclaw/workspace/openclaw-work/docs/docker-sandbox
DOCKER_BUILDKIT=1 docker build -t aosolao/openclaw-sandbox:py313-node26-trixie .

# 换镜像或改配置后，重建容器
openclaw sandbox recreate --all
```

### 验证

```bash
# 查看沙箱状态和挂载
openclaw sandbox explain

# 列出运行中的沙箱容器
openclaw sandbox list
docker ps --filter "name=openclaw-sbx"

# 进入容器调试
docker exec -it openclaw-sbx-main bash
```

容器内验证清单：

```bash
id                                          # uid=501(sandbox)
whoami                                      # sandbox
which python3                               # /usr/local/bin/python3
python3 -c "import pandas, akshare; print('OK')"
. /etc/profile.d/xvfb.sh && echo $DISPLAY   # :99
pgrep -f "Xvfb :99"                         # 有进程
df -h /dev/shm                              # 2.0G
env | grep -i proxy                         # 代理变量
date +%Z                                    # 时区（CST 或 Asia/Shanghai）
echo $LANG                                  # 编码（C.UTF-8）
touch /workspace/.test && rm /workspace/.test
```

## 9.6 Bind mount 安全根限制

默认情况下 workspace（`~/.openclaw/workspace`）会自动挂载到容器内 `/workspace`，其下所有子目录均可直接访问，**无需额外配置 bind mount**。

如果你需要挂载 workspace 之外的目录，注意 OpenClaw 出于安全考虑，**只允许 bind mount 的源路径在 `~/.openclaw/workspace` 内**。它会对路径做 `realpath` 解析后二次校验——这意味着**软链方案无效**：即使你在 workspace 内建一个软链指向外部目录，OpenClaw 解析真实路径后仍会拒绝。

正确做法是**把真实目录搬进 workspace，再在原位置建软链**：

```bash
# 真实目录搬进 workspace
mv ~/your-projects/some-project ~/.openclaw/workspace/

# 原位置建软链指回，保持旧路径可用
ln -s ~/.openclaw/workspace/some-project ~/your-projects/some-project
```

这样 OpenClaw 校验时真实路径在允许根内，而你在终端里用旧路径也不受影响。

## 9.7 Elevated 逃逸

沙箱启用后，容器内没有 `openclaw` CLI，也看不到宿主机路径。需要执行宿主机操作时有两种方式：

1. **直接在宿主机终端运行**（最简单安全）。
2. **开启 elevated 逃逸**，让 AI 请求在宿主机执行（需你审批）。

elevated 是**持久配置**但**每次调用都需审批**，不是自动放行。

```bash
# 开启（webchat 本地使用，"*" 表示所有发送者）
openclaw config set tools.elevated.allowFrom.webchat '["*"]' --strict-json

# 关闭（删除配置项）
❯ echo '{ "tools": { "elevated": { "allowFrom": { "webchat": null } } } }' | openclaw config patch --stdin

Applied 1 config update(s). No gateway restart needed.

# 验证
❯ openclaw config get tools.elevated.allowFrom

{}

# 改完重启
openclaw gateway restart
```

建议平时关闭，需要时临时开启、用完即关。

## 9.8 沙箱内不可用的功能

- `/learn`（skill_workshop 工具被禁，因为 skills 目录只读挂载）。需要创建 skill 时，让 AI 写到 workspace 暂存目录，再手动 `mv` 到 `~/.openclaw/skills/`。
- `openclaw` CLI（容器内未安装）。
- 宿主机浏览器 CDP（浏览器工具走宿主机 Chrome，不经过容器）。
- 直接访问宿主机文件系统（除了挂载进来的 `/workspace`）。

## 9.9 常见问题

### Q: 改了配置但 exec 还在宿主机跑？

`tools.exec.host` 改动后当前会话不会立即生效，需要重启 gateway 或开新会话。用 `openclaw sandbox explain` 确认 `runtime: sandboxed`。

### Q: 容器内 python3 找不到包？

检查 `which python3`。如果是 `/usr/bin/python3`，说明 PATH 顺序有问题，配置 `tools.exec.pathPrepend`。

### Q: Chromium 启动崩溃或白屏？

检查 `/dev/shm` 大小（`df -h /dev/shm`）。Playwright 需要至少 512MB，推荐 2GB，通过 `docker.tmpfs` 配置。

### Q: 容器内访问不了外网？

默认 `network: "none"`，必须改为 `"bridge"`。如果需要代理，在 `docker.env` 里设置 `HTTP_PROXY`/`HTTPS_PROXY` 指向 `host.docker.internal:<端口>`（Mac/Windows 开箱即用；Linux 原生 Docker 需在 `docker.runArgs` 加 `--add-host=host.docker.internal:host-gateway`）。

### Q: 新移入的 skill 在容器内看不到？

skills 目录在容器启动时挂载，运行中移入的新 skill 需要 `openclaw sandbox recreate --all` 重建容器才能看到。
