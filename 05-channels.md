# 第 5 章 · 渠道配置（Channels）

> 版本：v1.0.0
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [5.1 网页控制台（开箱即用）](#51-网页控制台开箱即用)
- [5.2 配置 Telegram（最常用）](#52-配置-telegram最常用)
  - [准备工作](#准备工作)
  - [配置](#配置)
  - [字段说明](#字段说明)
  - [把自己设为命令所有者](#把自己设为命令所有者)
- [5.3 授权消息工具（让 AI 能主动发消息/附件）](#53-授权消息工具让-ai-能主动发消息附件)
- [5.4 Telegram 高级互动：表情回应、贴纸、GIF](#54-telegram-高级互动表情回应贴纸gif)
  - [Emoji Reaction（表情回应）](#emoji-reaction表情回应)
  - [Sticker（贴纸/表情包）](#sticker贴纸表情包)
  - [动作开关汇总](#动作开关汇总)
  - [Reaction 通知](#reaction-通知)
  - [GIF](#gif)
- [5.5 其他渠道概览](#55-其他渠道概览)
- [5.6 渠道安全原则](#56-渠道安全原则)
- [5.7 平台差异](#57-平台差异)
- [5.8 下一步](#58-下一步)

<!-- TOC END -->

渠道就是你和 AI 对话的入口：Telegram、Discord、网页控制台、WhatsApp、Signal 等。本章以最常用的 Telegram 为例。

## 5.1 网页控制台（开箱即用）

启动 Gateway 后，浏览器打开 `http://127.0.0.1:18789/` 即可对话，无需额外配置。本机 loopback 绑定下只有你能访问。

## 5.2 配置 Telegram（最常用）

### 准备工作
1. 在 Telegram 找 @BotFather，发 `/newbot`，按提示创建机器人，拿到 **bot token**。
2. 找 @userinfobot 拿到你自己的 **数字 user id**（用来限制只有你能用）。

### 配置

```bash
openclaw config set plugins.entries.telegram.enabled true
openclaw config set channels.telegram.enabled true

# bot token（建议用 SecretRef，别明文；下面是明文示例）
openclaw config set channels.telegram.botToken '"你的bot-token"'

# 只允许你自己私聊
openclaw config set channels.telegram.dmPolicy allowlist
openclaw config set channels.telegram.allowFrom '[你的数字ID]' --strict-json

# 群聊默认关闭（个人使用推荐）
openclaw config set channels.telegram.groupPolicy disabled

openclaw gateway restart
```

### 字段说明

| 字段 | 作用 | 推荐 |
|---|---|---|
| `dmPolicy` | 私聊策略：`allow`/`allowlist`/`deny` | `allowlist` |
| `allowFrom` | 允许的用户 ID 列表 | 只有你自己 |
| `groupPolicy` | 群聊策略 | `disabled`（个人用） |
| `groupAllowFrom` | 允许触发的群内用户 | 按需 |

### 把自己设为命令所有者

```bash
openclaw config set commands.ownerAllowFrom '["telegram:你的数字ID"]' --strict-json
openclaw gateway restart
```

这样你才能在 Telegram 里用 `/config`、`/diagnostics`、审批危险命令等特权功能。

## 5.3 授权消息工具（让 AI 能主动发消息/附件）

默认 AI 只能"回复"，不能主动发消息或发文件。开启：

```bash
openclaw config set tools.alsoAllow '["group:messaging"]' --strict-json
```

| 能力 | 开启前 | 开启后 |
|---|---|---|
| 回复文字 | ✅ | ✅ |
| 主动发消息 | ❌ | ✅ |
| 发文件/附件 | ❌ | ✅ |
| 处理 exec 审批 | ❌ | ✅（在 Telegram 里批准命令） |

只要你的 Telegram 是 allowlist 且只有你，风险很低——AI 只能发给你。

> 这是工具层配置，原理见 [3.6 工具策略](./03-security-and-exec.md#36-工具策略tool-policy)。

## 5.4 Telegram 高级互动：表情回应、贴纸、GIF

除了文字，Telegram 渠道还支持更丰富的互动方式，但这些动作默认是关闭的，需要按需开启。

### Emoji Reaction（表情回应）

让 AI 能在你的消息上贴一个 emoji 反应（👍❤️😂😻 这种飘在消息上的小表情），比发文字更轻量自然：

```bash
openclaw config set channels.telegram.actions.reactions true
```

AI 调用 message 工具的 `react` 动作即可给指定消息加 emoji。适合表示“收到”、“好笑”、“赞”等轻量反馈。

### Sticker（贴纸/表情包）

让 AI 能发送 Telegram 贴纸。默认关闭：

```bash
openclaw config set channels.telegram.actions.sticker true
```

发送贴纸需要贴纸的 `fileId`（Telegram 内部 ID）。有两种用法：
- 发已知 fileId 的贴纸：`action: "sticker"` + `fileId`。
- 搜索已缓存贴纸：`action: "sticker-search"`——你之前给 AI 发过的贴纸会被缓存（含 emoji、setName 等），AI 可以按描述搜索后重发。

⚠️ **限制**：Bot API 不能上传新贴纸，只能发已有 fileId 的贴纸。也就是说 AI 不能“从网上找个表情包发你”，只能发你之前发过的、或你告诉它 fileId 的贴纸。静态 WEBP 贴纸能识别；动态 TGS 和视频 WEBM 会被跳过。

### 动作开关汇总

相关开关在 `channels.telegram.actions` 下：

| 动作 | 开关 | 默认 |
|---|---|---|
| 发消息 | `sendMessage` | 开（受工具白名单控制） |
| 表情回应 | `reactions` | 关 |
| 发贴纸 | `sticker` | 关 |
| 删除消息 | `deleteMessage` | 关 |
| 编辑消息 | `editMessage` | 开 |
| 创建论坛话题 | `createForumTopic` | 开 |

### Reaction 通知

别人对你机器人发的消息加 reaction 时，可以让 OpenClaw 把它作为系统事件通知 AI：

```bash
# own = 只通知别人对机器人消息的反应（默认）
openclaw config set channels.telegram.reactionNotifications own
# off = 不通知；all = 所有反应都通知
openclaw config set channels.telegram.reactionLevel minimal
```

### GIF

有 `gif-search` 技能可搜 GIF 动图发送，但需要配 GIF 搜索源（如 Tenor/Giphy API key）。按需启用。

## 5.5 其他渠道概览

配置方式类似，都是 `plugins.entries.<渠道>.enabled true` + `channels.<渠道>.*`。常用：

| 渠道 | 插件 | 说明 |
|---|---|---|
| Discord | `plugins.entries.discord` | 需要 bot token，支持群聊 |
| Slack | `plugins.entries.slack` | 工作区集成 |
| WhatsApp | `plugins.entries.whatsapp` | 需要配对扫码 |
| Signal | `plugins.entries.signal` | 需 signal-cli |
| iMessage | `plugins.entries.imessage` | 仅 macOS |
| Matrix | `plugins.entries.matrix` | 自托管友好 |
| Feishu/Lark | `plugins.entries.feishu` | 国内常用 |

查看已加载插件：

```bash
openclaw plugins list
```

## 5.6 渠道安全原则

1. **DM allowlist 只放你自己**，别开放给陌生人。
2. **群聊默认关闭**，除非你明确需要在群里用。
3. **bot token 当密码保管**，泄露了立即去 BotFather 重置。
4. 用 SecretRef 存 token，别明文写配置（见第 6 章 macOS Keychain 示例）。
5. 配置完跑 `openclaw doctor` 确认无渠道安全警告。

## 5.7 平台差异

- **macOS**：iMessage 渠道独占；通知可走 Apple Notification。
- **Linux**：桌面通知依赖 libnotify；WhatsApp/Signal 需要各自的后台服务。
- **Windows**：iMessage 不可用；推荐 Telegram/Discord/网页控制台。

## 5.8 下一步

- 本地服务与密钥管理 → [第 6 章 本地服务与插件](./06-local-services.md)
