# 第 4 章 · 记忆系统

> 版本：v1.1.0
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [4.1 小白先懂：AI 为什么会"失忆"](#41-小白先懂ai-为什么会失忆)
- [4.2 选择记忆后端](#42-选择记忆后端)
  - [推荐：memory-lancedb](#推荐memory-lancedb)
- [4.3 Embedding（把文字变成向量）](#43-embedding把文字变成向量)
  - [选项对比](#选项对比)
  - [方案 A：本地 embedding 服务（隐私首选）](#方案-a本地-embedding-服务隐私首选)
  - [方案 B：云端 embedding（OpenAI 兼容）](#方案-b云端-embeddingopenai-兼容)
- [4.4 autoRecall 与 autoCapture](#44-autorecall-与-autocapture)
- [4.5 让记忆工具真正可用（tool profile 与钩子权限）](#45-让记忆工具真正可用tool-profile-与钩子权限)
  - [4.5.1 coding profile 默认屏蔽记忆工具](#451-coding-profile-默认屏蔽记忆工具)
  - [4.5.2 agent_end 钩子被安全策略拦截](#452-agent_end-钩子被安全策略拦截)
  - [4.5.3 自检清单](#453-自检清单)
- [4.6 怎么手动存记忆](#46-怎么手动存记忆)
- [4.7 文件记忆（MEMORY.md / daily notes）](#47-文件记忆memorymd-daily-notes)
- [4.8 关于"做梦"（dreaming）](#48-关于做梦dreaming)
- [4.9 Active Memory（主动召回插件，可选）](#49-active-memory主动召回插件可选)
- [4.10 记忆数据位置与备份](#410-记忆数据位置与备份)
- [4.11 下一步](#411-下一步)

<!-- TOC END -->

## 4.1 小白先懂：AI 为什么会"失忆"

每次对话默认是独立的——上次告诉它的事，这次可能就忘了。记忆系统就是让 AI 能把重要信息存下来、需要时想起来。OpenClaw 的记忆分几种：

1. **工作区文件记忆**：`MEMORY.md`、`memory/YYYY-MM-DD.md` 这些文件，每次自动加载。
2. **语义搜索记忆**：把笔记/事实转成向量，按意思搜索（需要 embedding 模型）。
3. **长期向量记忆库**（LanceDB）：存原子化的事实/偏好/决定，自动召回。
4. **做梦（dreaming）**：内置 memory-core 的定时整理功能（注意：LanceDB 插件下不适用）。

## 4.2 选择记忆后端

| 后端 | 安装 | 适合 | 自动召回 | 自动捕获 |
|---|---|---|---|---|
| **内置 memory-core** | 自带 | 简单、文件为主 | 需主动搜 | 否 |
| **memory-lancedb** ✅ | `openclaw plugins install @openclaw/memory-lancedb` | 想要自动记忆、向量库 | ✅ | 可选 |
| **QMD** | `npm i -g @tobilu/qmd` | 搜工作区外文档、重排序 | 需配置 | 否 |
| **Honcho** | `openclaw plugins install @honcho-ai/openclaw-honcho` | 跨会话/跨渠道用户画像 | ✅ | ✅ |

### 推荐：memory-lancedb

本地向量库、零云依赖、能自动召回，是个人使用的最佳平衡。

```bash
openclaw plugins install @openclaw/memory-lancedb
openclaw config set plugins.slots.memory '"memory-lancedb"'
```

## 4.3 Embedding（把文字变成向量）

语义搜索需要 embedding 模型。**这一步必须配，否则只能关键词搜索。**

### 选项对比

| 方案 | 本地 | 开销 | 中文 | 推荐度 |
|---|---|---|---|---|
| 本地 vLLM-MLX/Ollama + bge/e5 | ✅ | 占 1-2GB 内存 | 好 | ⭐⭐⭐⭐⭐ |
| OpenAI text-embedding-3 | ❌ 云 | 极低 | 好 | ⭐⭐⭐⭐（不介意云） |
| 阿里百炼 text-embedding-v3 | ❌ 云 | 极低 | 很好 | ⭐⭐⭐⭐（国内） |
| 内置 local GGUF | ✅ | 需装插件 | 一般 | ⭐⭐⭐ |

### 方案 A：本地 embedding 服务（隐私首选）

以 vLLM-MLX 跑 multilingual-e5-large 为例（1024 维，多语言含中文）。模型可换成任意 OpenAI 兼容 embedding 服务（Ollama: `ollama pull mxbai-embed-large`，端口 11434）。

**第 1 步：确保服务在跑**，监听如 `http://127.0.0.1:8001/v1`，模型 id 如 `e5-local`。

**第 2 步：配置 LanceDB 用它**

```bash
openclaw config set plugins.entries.memory-lancedb.config.embedding.baseUrl '"http://127.0.0.1:8001/v1"'
openclaw config set plugins.entries.memory-lancedb.config.embedding.model '"e5-local"'
openclaw config set plugins.entries.memory-lancedb.config.embedding.apiKey '"not-needed"'
openclaw config set plugins.entries.memory-lancedb.config.embedding.dimensions 1024
openclaw config set plugins.entries.memory-lancedb.config.autoRecall true
openclaw config set plugins.entries.memory-lancedb.config.autoCapture false
openclaw gateway restart
```

> 维度要和模型一致：e5-large/mxbai-embed-large/bge-m3 = 1024；bge-small = 512；OpenAI text-embedding-3-small = 1536；text-embedding-3-large = 3072。

**第 3 步：验证**

```bash
curl http://127.0.0.1:8001/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"e5-local","input":"测试"}'
openclaw ltm stats   # 应显示 LanceDB 已注册
```

### 方案 B：云端 embedding（OpenAI 兼容）

```bash
openclaw config set plugins.entries.memory-lancedb.config.embedding.baseUrl '"https://api.openai.com/v1"'
openclaw config set plugins.entries.memory-lancedb.config.embedding.model '"text-embedding-3-small"'
openclaw config set plugins.entries.memory-lancedb.config.embedding.apiKey '"$OPENAI_API_KEY"'
openclaw config set plugins.entries.memory-lancedb.config.embedding.dimensions 1536
```

## 4.4 autoRecall 与 autoCapture

```bash
# 每轮对话前自动搜索相关记忆并注入（推荐开启）
openclaw config set plugins.entries.memory-lancedb.config.autoRecall true

# 自动把对话里的事实存进记忆库（建议先关，用熟了再开）
openclaw config set plugins.entries.memory-lancedb.config.autoCapture false
```

| 设置 | 效果 | 建议 |
|---|---|---|
| autoRecall=true | AI 回复前自动想起相关记忆 | ✅ 开 |
| autoCapture=false | 不自动存，你说"记住"才存 | ✅ 初期推荐，可控 |
| autoCapture=true | 自动捕获对话事实 | 后期可开，但会存进碎片 |

## 4.5 让记忆工具真正可用（tool profile 与钩子权限）

配好后端和 embedding 后，记忆仍可能写不进去。常见有三个坑，按顺序检查：

### 4.5.1 coding profile 默认屏蔽记忆工具

OpenClaw 的 `tools.profile: "coding"` 会移除 `memory_recall`、`memory_store`、`memory_forget` 三个工具。日志里会出现：

```
tool policy removed 7 tool(s) via tools.profile (coding):
... memory_forget, memory_recall, memory_store ...
```

被移除后 AI 无法主动调用记忆工具。必须在 `alsoAllow` 里显式加回：

```bash
openclaw config set tools.alsoAllow '["group:messaging","memory_recall","memory_store","memory_forget"]' --strict-json
```

> 用 `openclaw config get tools.alsoAllow` 确认。已加过 `browser` 等其他工具时，把它们一并写进同一个 JSON 数组，不要漏掉。

### 4.5.2 agent_end 钩子被安全策略拦截

memory-lancedb 是第三方插件，默认不允许访问对话内容。日志里会出现：

```
typed hook "agent_end" blocked because non-bundled plugins must set
plugins.entries.memory-lancedb.hooks.allowConversationAccess=true
```

不开这个权限，autoCapture 和每轮结束后的记忆处理都不会执行：

```bash
openclaw config set plugins.entries.memory-lancedb.hooks.allowConversationAccess true
openclaw gateway restart
```

重启后日志里不应再出现 `blocked` 字样。

### 4.5.3 自检清单

配完后逐项验证：

```bash
# 1. 插件已注册并初始化（日志应有 "initialized"）
grep memory-lancedb /tmp/openclaw/openclaw-$(date +%F).log | tail -5

# 2. 工具对 AI 可见（不应出现在 removedTools 里）
grep "tool policy" /tmp/openclaw/openclaw-$(date +%F).log | tail -3

# 3. 数据库有数据片段
ls -la ~/.openclaw/memory/lancedb/memories.lance/data/

# 4. 直接对 AI 说"记住：测试记忆 XYZ"，然后问"我刚才让你记住什么"
```

## 4.6 怎么手动存记忆

**最简单：直接对 AI 说。**

- "记住：我喜欢喝美式咖啡，不加糖"
- "记一下：我的服务器是 192.168.1.10"
- "把这个决定记下来：周末去杭州"
- 触发词：记住、记一下、别忘了、以后都、remember、don't forget

AI 会调用 `memory_store`，自动分类（偏好/事实/决定/实体）、去重、打重要度。

**命令行查询（不能直接写入，写入走对话）：**

```bash
openclaw ltm list                 # 列出所有记忆
openclaw ltm search "关键词"       # 语义搜索
openclaw ltm stats                # 统计
openclaw ltm query --filter "category = 'preference'"  # 条件查询
```

**删除：** 对 AI 说"忘掉关于 X 的记忆"，或用 `memory_forget`。

## 4.7 文件记忆（MEMORY.md / daily notes）

除了向量库，传统文件仍然重要：

- `MEMORY.md`：长期、精炼的"人生记忆"，主会话才加载。
- `memory/YYYY-MM-DD.md`：每日原始日志。
- 这些文件由 AI 主动维护，你说"记住"时 AI 也可能写文件。

向量库擅长"精准召回一句话事实"；文件擅长"承载长上下文和详细经过"。两者配合最好。

## 4.8 关于"做梦"（dreaming）

做梦是**内置 memory-core** 的功能：每天定时把短期记忆整理提升进 `MEMORY.md`。默认凌晨 3 点，默认关闭。

⚠️ **如果你切换到了 memory-lancedb，memory-core 被禁用，做梦功能不再生效。** LanceDB 本身存的每条记忆都能直接被检索，不需要"提升"这一步。

如果你坚持用内置 memory-core 并想开做梦：

```bash
openclaw config set plugins.entries.memory-core.config.dreaming.enabled true
openclaw config set plugins.entries.memory-core.config.dreaming.frequency "0 10 * * *"
openclaw config set plugins.entries.memory-core.config.dreaming.timezone "Asia/Shanghai"
```

**关机/睡眠会错过定时任务且不补跑。** 如果你每晚关机，把时间设在白天开机时段，或手动整理：

```bash
openclaw memory promote --apply   # 手动提升候选记忆
```

## 4.9 Active Memory（主动召回插件，可选）

默认 AI 要自己决定何时搜记忆。active-memory 插件在每次回复前自动跑一个轻量子任务搜记忆，让回忆更自然。会稍微增加延迟（约 1-2 秒）。

```bash
openclaw config set plugins.entries.active-memory.enabled true
openclaw config set plugins.entries.active-memory.config.agents '["main"]' --strict-json
openclaw config set plugins.entries.active-memory.config.allowedChatTypes '["direct"]' --strict-json
openclaw config set plugins.entries.active-memory.config.queryMode '"recent"'
openclaw config set plugins.entries.active-memory.config.timeoutMs 15000
```

用 `/verbose on` 和 `/trace on` 可在对话里看到它召回了什么。

## 4.10 记忆数据位置与备份

- LanceDB：`~/.openclaw/memory/lancedb/`
- 工作区文件：`~/.openclaw/workspace/MEMORY.md`、`memory/`
- 内置 SQLite 索引：`~/.openclaw/agents/main/agent/openclaw-agent.sqlite`

**备份这三个地方**就能保住所有记忆。Windows 路径为 `%USERPROFILE%\.openclaw\`。

## 4.11 下一步

- 配聊天渠道 → [第 5 章 渠道配置](./05-channels.md)
