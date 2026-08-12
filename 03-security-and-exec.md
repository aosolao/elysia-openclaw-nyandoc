# 第 3 章 · 安全与执行审批

> 版本：v1.1.0
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [3.1 小白必懂：四层安全模型](#31-小白必懂四层安全模型)
- [3.2 Gateway 绑定与认证](#32-gateway-绑定与认证)
  - [绑定地址（关键）](#绑定地址关键)
  - [认证](#认证)
  - [检查暴露面](#检查暴露面)
- [3.3 命令执行审批（exec approvals）](#33-命令执行审批exec-approvals)
  - [三个一键预设](#三个一键预设)
  - [策略选项详解](#策略选项详解)
  - [推荐配置（cautious）](#推荐配置cautious)
  - [审批弹窗会出现在哪](#审批弹窗会出现在哪)
  - [白名单管理](#白名单管理)
- [3.4 防御内联代码注入（strictInlineEval）](#34-防御内联代码注入strictinlineeval)
- [3.5 命令所有者（Command Owner）](#35-命令所有者command-owner)
- [3.6 工具策略（tool policy）](#36-工具策略tool-policy)
- [3.7 平台注意事项](#37-平台注意事项)
  - [macOS](#macos)
  - [Linux](#linux)
  - [Windows](#windows)
- [3.8 安全审计与排错](#38-安全审计与排错)
- [3.9 下一步](#39-下一步)

<!-- TOC END -->

这是最重要的一章。配得好，AI 既能干活又不会闯祸；配不好，它可能一条命令删光你的文件。

## 3.1 小白必懂：四层安全模型

想象你雇了一个很能干但偶尔会误会意思的助手：

1. **网络层（Gateway 绑定）**：决定谁能连到你的 OpenClaw。
2. **执行层（exec policy）**：决定 AI 能不能在你电脑上跑命令、哪些命令要先问你。
3. **工具层（tool policy）**：决定 AI 能用哪些工具（发消息、读写文件、上网等）。
4. **隔离层（sandbox）**：把工具执行关进 Docker 容器，即使命令通过了审批也不直接触碰宿主机。详见[第 9 章](./09-sandbox.md)。

四层叠加，取更严格者生效。

## 3.2 Gateway 绑定与认证

### 绑定地址（关键）

```bash
# 只允许本机连接（个人使用强烈推荐）
openclaw config set gateway.bind loopback
openclaw config set gateway.port 18789
```

| 绑定值 | 含义 | 风险 |
|---|---|---|
| `loopback` | 仅 127.0.0.1 本机 | 安全 ✅ |
| `lan` | 局域网可连 | 中，需配认证 |
| `public` | 公网可连 | 高，必须强认证+反代 |

个人电脑上用 `loopback` 就对了。

### 认证

```bash
openclaw config set gateway.auth.mode token
```

token 不要明文写配置里，用 SecretRef（见第 2 章）。本机 loopback 绑定下，token 主要防本地其他用户/进程。

### 检查暴露面

```bash
openclaw security audit --deep
```

重点关注：
- `gateway.bind` 是不是 loopback。
- 有没有把服务暴露到公网/`0.0.0.0`。
- token 是否明文存储。
- 有没有启用危险的 debug 开关。

## 3.3 命令执行审批（exec approvals）

这是防止 AI 乱执行命令的核心。OpenClaw 有**两层策略**，最终取更严格的：

- **请求层**（`tools.exec.*`，存在 openclaw.json）
- **宿主层**（`~/.openclaw/exec-approvals.json`，本机审批文件）

用 `openclaw exec-policy` 命令会同步两层，不用手动改文件。

### 三个一键预设

```bash
openclaw exec-policy preset cautious   # 推荐：白名单 + 未命中询问 + 拒绝兜底
openclaw exec-policy preset yolo       # 放飞：全部允许，从不询问
openclaw exec-policy preset deny-all   # 锁死：禁止跑任何命令
```

### 策略选项详解

#### `security`（安全等级）

| 值 | 效果 | 适合 |
|---|---|---|
| `deny` | 禁止所有命令 | 纯聊天/只读场景 |
| `allowlist` | 只允许白名单内的，其余询问 | ✅ 日常推荐 |
| `full` | 允许所有命令 | 完全信任、沙箱环境 |

#### `ask`（何时弹窗确认）

| 值 | 效果 |
|---|---|
| `off` | 从不询问 |
| `on-miss` | 只有不在白名单的命令才问 ✅ |
| `always` | 每条命令都问（最严，包括已信任的） |

#### `askFallback`（你不在时怎么办）

| 值 | 效果 |
|---|---|
| `deny` | 默认拒绝 ✅ 最安全 |
| `allowlist` | 仅白名单内放行 |
| `full` | 默认放行（危险） |

### 推荐配置（cautious）

直接一条命令：

```bash
openclaw exec-policy preset cautious
```

等效于：security=allowlist、ask=on-miss、askFallback=deny。

查看当前生效策略：

```bash
openclaw exec-policy show
```

### 审批弹窗会出现在哪

- 网页控制台（Control UI，`http://127.0.0.1:18789`）
- 已配置的聊天渠道（如 Telegram，需要授权 message 工具，见第 5 章）
- 三个选项：允许一次 / 始终允许（加入白名单）/ 拒绝

### 白名单管理

审批时点"始终允许"会自动加白名单。也可手动管理：

```bash
openclaw approvals get                 # 查看当前策略和白名单
openclaw approvals allowlist list      # 列出白名单
openclaw approvals allowlist add --pattern "ls"     # 加裸命令名
openclaw approvals allowlist add --pattern "~/.local/bin/*"  # 加路径
openclaw approvals allowlist remove <id>
```

**pattern 两种写法：**
- 裸命令名（`rg`、`git`）：只匹配 PATH 里的命令。
- 路径 glob（`/opt/homebrew/bin/rg`）：匹配具体二进制位置，更安全。
- 可选 `argPattern`：正则限制参数，进一步收紧。

## 3.4 防御内联代码注入（strictInlineEval）

即使解释器（python/node 等）在白名单里，攻击者可能通过 `python -c "恶意代码"` 绕过。开启后这类内联 eval 必须单独审批：

```bash
openclaw config set tools.exec.strictInlineEval true
```

会被拦截的形式：`python -c`、`node -e`、`ruby -e`、`osascript -e`、`awk`、`sed`、`find -exec`、`xargs` 内联等。

**强烈建议开启。**

## 3.5 命令所有者（Command Owner）

DM 白名单只证明"某人能跟 bot 说话"，不代表他能执行管理员命令。设置 owner 后，只有你能在聊天里用 `/config`、`/diagnostics`、审批危险操作等：

```bash
# Telegram 示例
openclaw config set commands.ownerAllowFrom '["telegram:你的Telegram数字ID"]' --strict-json
```

怎么拿 Telegram ID？跟 @userinfobot 聊天即可得到。其他渠道前缀：`discord:`、`slack:`、`whatsapp:`（手机号）等。

## 3.6 工具策略（tool policy）

控制 AI 能用哪些工具。配置在 `tools.profile`：

| Profile | 包含 |
|---|---|
| `minimal` | 仅 session_status |
| `coding` | 文件、运行时、网页、会话、记忆、cron 等 |
| `messaging` | 消息发送相关 |
| `full` | 全部 |

默认 `coding`。额外加工具用 `alsoAllow`，例如：

```bash
# 让 AI 能主动发消息/附件到聊天渠道
openclaw config set tools.alsoAllow '["group:messaging"]' --strict-json
```

工具分组速查：

| 组 | 工具 |
|---|---|
| `group:fs` | read/write/edit/apply_patch |
| `group:runtime` | exec/process |
| `group:web` | web_search/web_fetch |
| `group:messaging` | message |
| `group:sessions` | 会话管理 |
| `group:memory` | 记忆搜索 |

拒绝特定工具：`tools.deny: ["browser"]`。

## 3.7 平台注意事项

### macOS
- Gateway 作为 LaunchAgent 运行，只在你登录后生效。
- 弹窗审批可在浏览器控制台处理。
- 涉及系统设置/摄像头/通讯录的命令还会受 macOS TCC 权限限制。

### Linux
- 推荐用 systemd user service 托管 Gateway。
- 不要用 root 跑 Gateway；用普通用户。
- 审批依赖控制台或聊天渠道，无 GUI 服务器注意 `askFallback=deny` 会拦住所有未白名单命令。

### Windows
- 尽量在 PowerShell 7 中操作。
- 命令审批同样适用，但路径用 Windows 格式；白名单 pattern 注意 `.exe` 路径。
- 部分 Unix 工具（bash 管道）在 Windows 上行为不同，建议用 PowerShell 原生命令。
- Mac companion app 不存在；如需 GUI 审批，用网页控制台。

## 3.8 安全审计与排错

```bash
openclaw security audit --deep   # 深度安全检查
openclaw doctor --allow-exec     # 诊断（允许解析密钥类 SecretRef）
openclaw status --deep           # 完整状态
```

如果 `security audit --deep` 报 `probe_failed: missing operator.read`，通常是审计命令用匿名探针连接网关导致的误报（CLI 探针不携带配对设备身份）。确认你的网关是 loopback+token 认证后，可以静默该检查：

```bash
openclaw config set security.audit.suppressions \
  '[{"checkId":"gateway.probe_failed","reason":"loopback+token, CLI probe false positive"}]' \
  --strict-json
```

> 注意：这是基于具体版本的 CLI 探针限制采取的抑制。未来 OpenClaw 版本可能修复该问题；升级后如果 `openclaw security audit --deep` 不再报警，应删除这条 suppression（`openclaw config unset security.audit.suppressions`），保持审计输出真实。

## 3.9 下一步

- 配置记忆 → [第 4 章 记忆系统](./04-memory.md)
