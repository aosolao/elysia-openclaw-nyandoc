# 第 6 章 · 本地服务与插件

> 版本：v1.1.0
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [6.1 插件管理基础](#61-插件管理基础)
- [6.2 本地 Embedding 服务（以 vLLM-MLX 为例）](#62-本地-embedding-服务以-vllm-mlx-为例)
  - [6.2.1 准备模型](#621-准备模型)
  - [6.2.2 用 LaunchAgent 托管（macOS，推荐）](#622-用-launchagent-托管macos推荐)
  - [6.2.3 Linux 等价方案（systemd user service）](#623-linux-等价方案systemd-user-service)
  - [6.2.4 Windows 等价方案](#624-windows-等价方案)
  - [6.2.5 验证](#625-验证)
- [6.3 安全存放密钥：macOS Keychain 示例](#63-安全存放密钥macos-keychain-示例)
  - [Linux 等价](#linux-等价)
  - [Windows 等价](#windows-等价)
- [6.4 网络代理（国内网络环境）](#64-网络代理国内网络环境)
- [6.5 Web 搜索配置](#65-web-搜索配置)
  - [选项对比](#选项对比)
  - [方案 A：DuckDuckGo（免 key，最快上手）](#方案-aduckduckgo免-key最快上手)
  - [方案 B：Brave Search（稳定推荐）](#方案-bbrave-search稳定推荐)
- [6.6 浏览器自动化工具](#66-浏览器自动化工具)
  - [前置条件](#前置条件)
  - [配置三步](#配置三步)
  - [启动与验证](#启动与验证)
  - [三种 profile 模式](#三种-profile-模式)
- [6.7 常用插件推荐](#67-常用插件推荐)
- [6.8 下一步](#68-下一步)

<!-- TOC END -->

## 6.1 插件管理基础

```bash
openclaw plugins list                    # 列出已安装插件
openclaw plugins install <包名>           # 安装插件
openclaw plugins inspect <插件id>         # 查看插件详情
```

OpenClaw 的插件通过 npm 安装到 `~/.openclaw/npm/projects/`。装好第三方插件后，建议显式信任，避免启动警告：

```bash
openclaw config set plugins.allow '["memory-lancedb"]' --strict-json
```

## 6.2 本地 Embedding 服务（以 vLLM-MLX 为例）

如果你想用本地 embedding 模型（不走云端、隐私好），可以用 vLLM-MLX、Ollama 或 LM Studio 起一个 OpenAI 兼容的 `/v1/embeddings` 服务。

### 6.2.1 准备模型

以 multilingual-e5-large（1024 维，多语言，中文好）为例，用 MLX 版权重。建立一个符号链接和注册表：

```bash
mkdir -p ~/ai-workspace/vllm-mlx
cd ~/ai-workspace/vllm-mlx
python3.12 -m venv .venv
.venv/bin/pip install vllm-mlx
# 模型软链
ln -s /路径/to/multilingual-e5-large-mlx e5-local
```

`embedding-only-registry.yaml` 示例：

```yaml
manager:
  memory_budget_gb: 2
  contention_policy:
    strategy: fail
models:
  - name: e5-local
    path: /绝对路径/to/multilingual-e5-large-mlx
    estimated_memory_gb: 1.2
```

### 6.2.2 用 LaunchAgent 托管（macOS，推荐）

让服务开机自启、崩溃自动重启，且**不阻止 Mac 睡眠**。创建 `~/Library/LaunchAgents/ai.embedding.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.embedding</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USER/ai-workspace/vllm-mlx/.venv/bin/vllm-mlx</string>
        <string>serve</string>
        <string>--models-config</string>
        <string>/Users/YOUR_USER/ai-workspace/vllm-mlx/embedding-only-registry.yaml</string>
        <string>--embedding-model</string>
        <string>e5-local</string>
        <string>--host</string>
        <string>127.0.0.1</string>
        <string>--port</string>
        <string>8001</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USER/ai-workspace/vllm-mlx</string>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USER/ai-workspace/vllm-mlx/.embed.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USER/ai-workspace/vllm-mlx/.embed.log</string>
</dict>
</plist>
```

加载/卸载：

```bash
launchctl load ~/Library/LaunchAgents/ai.embedding.plist     # 启动+开机自启
launchctl unload ~/Library/LaunchAgents/ai.embedding.plist   # 停止
launchctl list | grep embedding                                    # 查状态
```

> ⚠️ **不要加 `caffeinate -i`**：那会阻止 Mac 空闲睡眠。embedding 服务睡眠时没人用，唤醒后自动恢复，不需要防睡眠。

### 6.2.3 Linux 等价方案（systemd user service）

创建 `~/.config/systemd/user/embedding.service`：

```ini
[Unit]
Description=Local Embedding Service
After=network.target

[Service]
Type=simple
WorkingDirectory=%h/ai-workspace/vllm-mlx
ExecStart=%h/ai-workspace/vllm-mlx/.venv/bin/vllm-mlx serve \
  --models-config %h/ai-workspace/vllm-mlx/embedding-only-registry.yaml \
  --embedding-model e5-local --host 127.0.0.1 --port 8001
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now embedding
systemctl --user status embedding
# 允许开机自启（即使没登录）：
sudo loginctl enable-linger $USER
```

### 6.2.4 Windows 等价方案

用任务计划程序或 [NSSM](https://nssm.cc/) 把启动命令注册为服务。最简单的方式是写一个 `start-embedding.bat` 放到开机启动文件夹（`Win+R` 输入 `shell:startup`）。把下面内容存成该目录下的 `start-embedding.bat`（按实际路径修改）：

```bat
@echo off
cd /d C:\path\to\vllm-mlx
.venv\Scripts\vllm-mlx.exe serve --models-config embedding-only-registry.yaml --embedding-model e5-local --host 127.0.0.1 --port 8001
pause
```

### 6.2.5 验证

```bash
curl http://127.0.0.1:8001/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"e5-local","input":"测试"}'
```

返回向量数组即成功。Ollama 用户：`ollama pull mxbai-embed-large`，端口 11434，baseUrl 用 `http://127.0.0.1:11434`，dimensions=1024。

## 6.3 安全存放密钥：macOS Keychain 示例

不要把 token/API key 明文写进 openclaw.json。用 exec SecretRef 从 Keychain 读取。

**第 1 步：写读取脚本** `~/.openclaw/secrets/kc-read.sh`：

```bash
#!/usr/bin/env bash
# 用法: kc-read.sh <keychain-item-name>
security find-generic-password -s "$1" -w 2>/dev/null
```

```bash
chmod 700 ~/.openclaw/secrets/kc-read.sh
```

**第 2 步：把密钥存进 Keychain**

```bash
security add-generic-password -s "openclaw.telegram.botToken" -w "你的bot-token" -U
security add-generic-password -s "openclaw.gateway.auth.token" -w "你的网关token" -U
security add-generic-password -s "openclaw.your-provider.apiKey" -w "你的模型API key" -U
```

**第 3 步：在 openclaw.json 里引用**（通过命令或直接编辑）

```json
"botToken": {
  "source": "exec",
  "provider": "keychain_telegram",
  "id": "value"
}
```

并在 `secrets.providers` 定义 provider：

```json
"secrets": {
  "providers": {
    "keychain_telegram": {
      "source": "exec",
      "command": "/Users/YOUR_USER/.openclaw/secrets/kc-read.sh",
      "args": ["openclaw.telegram.botToken"],
      "passEnv": ["HOME","USER","PATH"],
      "timeoutMs": 8000
    }
  }
}
```

### Linux 等价

用 `secret-tool`（libsecret）或 `pass`：

```bash
secret-tool store --label="OpenClaw Telegram" openclaw telegram-bot
# 脚本里：secret-tool lookup openclaw telegram-bot
```

### Windows 等价

用 PowerShell 读取凭据管理器：

```powershell
# 存（一次性）
cmdkey /generic:openclaw-telegram /user:bot /pass:你的token
# 读脚本里用 PowerShell：
# (Get-StoredCredential -Target openclaw-telegram).GetNetworkCredential().Password
```

## 6.4 网络代理（国内网络环境）

如果模型 API 或插件下载需要代理：

```bash
# 环境变量（写进 ~/.zshrc / 环境变量）
export HTTP_PROXY=http://127.0.0.1:6152
export HTTPS_PROXY=http://127.0.0.1:6152
# 如果本地/国内服务不需要代理，加 no_proxy
export NO_PROXY=127.0.0.1,localhost
```

但注意：Gateway 作为 LaunchAgent 运行时不读你的 shell 配置。OpenClaw 提供了一个**不会被升级覆盖**的包装脚本 `~/.openclaw/gateway-wrapper.sh`，在里面设置代理环境变量和 Node 引导即可。典型写法：

```bash
#!/bin/sh
set -eu

# 加载 OpenClaw 生成的环境变量
GENERATED_ENV="/Users/YOUR_USER/.openclaw/service-env/ai.openclaw.gateway.env"
if [ -f "$GENERATED_ENV" ]; then . "$GENERATED_ENV"; fi

export HTTP_PROXY='http://127.0.0.1:6152'
export HTTPS_PROXY='http://127.0.0.1:6152'
export ALL_PROXY='socks5://127.0.0.1:6153'
export NO_PROXY='localhost,127.0.0.1'
# Node 的 undici fetch 不会自动读 HTTP_PROXY，需要 bootstrap 注入
export NODE_OPTIONS="--import /Users/YOUR_USER/.openclaw/proxy-bootstrap.mjs"

OPENCLAW_BIN="$(command -v openclaw)"
exec "$OPENCLAW_BIN" "$@"
```

配套的 `proxy-bootstrap.mjs`（用 undici 的 `EnvHttpProxyAgent` 设置全局 dispatcher）：

```js
import { createRequire } from 'module';
const require = createRequire(import.meta.url);
try {
  const undici = require('/Users/YOUR_USER/.local/lib/node_modules/openclaw/node_modules/undici');
  const proxyUrl = process.env.HTTPS_PROXY || process.env.HTTP_PROXY;
  if (proxyUrl && undici.EnvHttpProxyAgent) {
    undici.setGlobalDispatcher(new undici.EnvHttpProxyAgent());
  }
} catch (err) {
  console.warn('[proxy-bootstrap] skipped:', err.message);
}
```

> ⚠️ 不要直接改 `service-env/ai.openclaw.gateway.env`——文件头注明了"Do not edit while the gateway service is installed"，升级或重装时会被重新生成。改 `gateway-wrapper.sh` 才是持久化方案。

国内 API 通常不需要代理，靠 `NO_PROXY` 排除即可。

## 6.5 Web 搜索配置

`web_search` 工具需要显式选择搜索后端，否则报错 `web_search is disabled or no provider is available`。

### 选项对比

| 后端 | API Key | 质量 | 备注 |
|---|---|---|---|
| **DuckDuckGo** | 不需要 | 一般 | 实验性 HTML 抓取，免配置，可能被验证码挡 |
| **Brave Search** | 免费额度 1000 次/月 | 好 | 结构化结果，推荐长期使用 |
| Perplexity / Exa / Tavily | 需要 | 好 | 付费或试用额度 |
| 其他 AI 搜索引擎 | 需要 | 好 | 国内可用多种选择 |

### 方案 A：DuckDuckGo（免 key，最快上手）

```bash
openclaw config set tools.web.search.provider duckduckgo
# 无需重启，立即生效
```

可选区域和安全搜索：

```bash
openclaw config set plugins.entries.duckduckgo.config.webSearch.region '"us-en"'
openclaw config set plugins.entries.duckduckgo.config.webSearch.safeSearch '"moderate"'
```

⚠️ DuckDuckGo 是实验性集成，抓取 HTML 搜索页，高频使用可能遇到 CAPTCHA。国内网络需确保 Gateway 已配代理（见 6.4）。

### 方案 B：Brave Search（稳定推荐）

1. 到 <https://brave.com/search/api/> 注册，选择 Search 套餐（含每月 $5 免费额度，约 1000 次）。
2. 生成 API key，存进 Keychain（见 6.3），**不要把真 key 明文写进 openclaw.json**。

```bash
# 把 YOUR_BRAVE_KEY 换成实际 key；更安全的做法是用 SecretRef 从 Keychain 读取
openclaw config set plugins.entries.brave.config.webSearch.apiKey '"YOUR_BRAVE_KEY"'
openclaw config set plugins.entries.brave.config.webSearch.mode '"web"'
openclaw config set tools.web.search.provider brave
openclaw config set tools.web.search.maxResults 5
openclaw gateway restart
```

验证：直接对 AI 说"搜一下 XXX"，或 `openclaw config get tools.web.search`。

## 6.6 浏览器自动化工具

OpenClaw 可以跑一个**隔离的 Chrome/Chromium profile**（默认叫 `openclaw`），AI 用它打开网页、点击、填表、截图，与你的日常浏览器完全分开。

### 前置条件

- 已安装 Chrome / Brave / Edge / Chromium（macOS 上检测 `/Applications/Google Chrome.app`）。
- `plugins.allow` 白名单里必须包含 `browser`，否则 `openclaw browser` 命令和工具都不存在。
- `tools.profile: "coding"` 默认不含 browser，需要在 `alsoAllow` 里加。

### 配置三步

```bash
# 1. 允许 browser 插件
openclaw config set plugins.allow '["memory-lancedb","browser"]' --strict-json
# 2. alsoAllow 让 AI 能调用 browser 工具
openclaw config set tools.alsoAllow '["group:messaging","browser"]' --strict-json
# 3. 启用并设默认 profile
openclaw config set browser.enabled true
openclaw config set browser.defaultProfile openclaw
openclaw gateway restart
```

> 如果 `plugins.allow` 是严格白名单（你显式设过），必须把 `browser` 加进去；完全删掉 `plugins.allow` 也会恢复默认全部允许。有显式 root `browser` 配置块时也会激活插件。

### 启动与验证

```bash
openclaw browser --browser-profile openclaw doctor    # 诊断
openclaw browser --browser-profile openclaw start     # 启动 Chrome
openclaw browser --browser-profile openclaw status    # 查状态
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot  # 读取页面
```

`doctor` 全绿即可。隔离 profile 的数据在 `~/.openclaw/browser/openclaw/user-data/`。

### 三种 profile 模式

| profile | 作用 | 适合 |
|---|---|---|
| `openclaw`（默认） | 隔离的托管浏览器，无个人登录 | 自动化、抓取、测试 |
| `user` | 通过 Chrome DevTools 附加到你**真实的** Chrome 会话 | 需要登录态，且人在电脑前（首次会弹"允许远程调试"） |
| `chrome` | 通过 OpenClaw Chrome 扩展驱动真实 Chrome | 需要登录态，人不在电脑前（无弹窗） |

日常自动化用默认 `openclaw` 就够了。需要登录网站时再考虑 `user`/`chrome`。

## 6.7 常用插件推荐

| 插件 | 作用 | 安装 |
|---|---|---|
| memory-lancedb | 本地向量记忆 | `openclaw plugins install @openclaw/memory-lancedb` |
| @openclaw/llama-cpp-provider | 本地 GGUF embedding（无需另起服务） | `openclaw plugins install @openclaw/llama-cpp-provider` |
| active-memory | 主动召回（内置，直接启用） | 配置即开 |
| obsidian | 读写 Obsidian 笔记 | 按需 |
| github/gh-issues | GitHub 集成 | 按需 |

## 6.8 下一步

- 工作区与项目目录 → [第 7 章 工作区与项目目录](./07-workspace.md)
