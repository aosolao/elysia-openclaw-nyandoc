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
  - [本地方案完整对比](#本地方案完整对比)
  - [方案 A：本地 embedding 服务（隐私首选）](#方案-a本地-embedding-服务隐私首选)
  - [方案 B：云端 embedding（OpenAI 兼容）](#方案-b云端-embeddingopenai-兼容)
- [4.4 autoRecall 与 autoCapture](#44-autorecall-与-autocapture)
  - [存储增长与清理机制](#存储增长与清理机制)
    - [auto-capture 写入原理](#auto-capture-写入原理)
    - [存储增长模型](#存储增长模型)
    - [memory_forget 原理](#memory_forget-原理)
    - [清理频率建议](#清理频率建议)
- [4.5 让记忆工具真正可用（tool profile 与钩子权限）](#45-让记忆工具真正可用tool-profile-与钩子权限)
  - [4.5.1 coding profile 默认屏蔽记忆工具](#451-coding-profile-默认屏蔽记忆工具)
  - [4.5.2 agent_end 钩子被安全策略拦截](#452-agent_end-钩子被安全策略拦截)
  - [4.5.3 自检清单](#453-自检清单)
- [4.6 怎么手动存记忆](#46-怎么手动存记忆)
- [4.7 文件记忆（MEMORY.md / daily notes）](#47-文件记忆memorymd-daily-notes)
- [4.8 关于"做梦"（dreaming）](#48-关于做梦dreaming)
  - [适用前提](#适用前提)
  - [Dreaming 详细原理](#dreaming-详细原理)
  - [是否需要嵌入模型](#是否需要嵌入模型)
  - [开启做梦](#开启做梦)
  - [Dreaming 的安全保护](#dreaming-的安全保护)
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
| **Honcho** | `openclaw plugins install @honcho-ai/openclaw-honcho` | 跨会话/跨渠道用户画像 | ✅ | ✅ |

> ⚠️ QMD 后端已被官方移除，不再可用。

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
| 云端 text-embedding-3 | ❌ 云 | 极低 | 好 | ⭐⭐⭐⭐（不介意云） |
| 国内大模型服务商 Embedding API | ❌ 云 | 极低 | 很好 | ⭐⭐⭐⭐（国内） |
| 内置 local GGUF | ✅ | 需装插件 | 一般 | ⭐⭐⭐ |

### 本地方案完整对比

OpenClaw 原生支持 3 种本地 embedding adapter，另可通过 `openai-compatible` 万能接口接入更多本地服务：

**原生 adapter（开箱即用）：**

| Provider ID | 运行方式 | 嵌入模型举例 | 特点 |
|---|---|---|---|
| `local` | 进程内 GGUF（llama.cpp） | embeddinggemma-300m (~0.6GB) | 零外部进程，最轻量，需装 `@openclaw/llama-cpp-provider` 插件 |
| `ollama` | 本地 Ollama 服务 | mxbai-embed-large, nomic-embed-text, qwen3-embedding | 模型选择最多，本地质量最高 |
| `lmstudio` | 本地 LM Studio | 取决于加载的模型 | GUI 友好，适合不熟悉命令行的用户 |

**通过 `openai-compatible` 接入的本地服务：**

任何实现了 `/v1/embeddings` 的本地服务都能接入：

| 本地服务 | 特点 | 推荐嵌入模型 |
|---|---|---|
| vLLM | 高性能推理引擎，支持批量 | BAAI/bge-m3, E5-Mistral |
| TEI (HuggingFace) | 专为嵌入优化，Rust 实现 | 任意 HF 模型 |
| Xinference | 模型管理平台 | BAAI/bge-large-zh-v1.5 |
| FastEmbed (Qdrant) | 轻量快速，ONNX 推理 | BAAI/bge-small-zh |

**嵌入质量排名（从高到低）：**

1. 🥇 某旗舰级云端 embedding (3072维) — 最强，多语言优异
2. 🥈 某云端多模态 embedding (3072维) — 很强，支持多模态
3. 🥉 Voyage / BAAI/bge-m3 (1024维) — 优秀，中文表现好
4. Ollama + mxbai-embed-large (1024维) — 本地方案中最强
5. llama.cpp GGUF embeddinggemma (300M参数) — 轻量可用，质量中等

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

> 维度要和模型一致：e5-large/mxbai-embed-large/bge-m3 = 1024；bge-small = 512；text-embedding-3-small = 1536；text-embedding-3-large = 3072。

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
| autoCapture=false | 不自动存，你说“记住”才存 | ✅ 初期推荐，可控 |
| autoCapture=true | 自动捕获对话事实 | 后期可开，但会存进碎片 |

### 存储增长与清理机制

#### auto-capture 写入原理

```
对话发生 → agent_end 事件触发 → 检查是否满足 capture 条件 → 写入 LanceDB
```

**写入限制（防止爆炸）：**

| 限制项 | 默认值 | 作用 |
|---|---|---|
| `captureMaxChars` | 500 | 超过 500 字符的消息**不触发** auto-capture |
| 每轮上限 | 3 条 | 一次对话轮次最多捕获 3 条记忆 |
| 去重检测 | 内置 | 相似度高的内容不会重复写入 |
| 内容过滤 | 内置 | 拒绝 prompt injection、元数据、已注入的召回上下文 |

**触发条件：**
- 用户消息长度 ≤ `captureMaxChars`
- 匹配内置触发词（`记住`、`remember`、`prefer` 等）或 `customTriggers` 自定义词
- 不是信封/传输元数据
- 不是已注入的召回上下文

#### 存储增长模型

以下为估算值，实际取决于对话频率和触发词匹配情况：

| 使用强度 | 每日新增 | 每月新增 | 每年新增 |
|---|---|---|---|
| 轻度（偶尔说“记住”） | ~3-5 条 | ~100-150 条 | ~1200-1800 条 |
| 中度（日常偏好自动捕获） | ~10-20 条 | ~300-600 条 | ~3600-7200 条 |
| 重度（频繁触发） | ~30+ 条 | ~900+ 条 | ~10000+ 条 |

每条记忆存储：文本 + 1024 维向量 + 元数据 ≈ **几 KB**

> 即使每年 10000 条，总存储也就 **几十 MB**，不会爆炸。

#### memory_forget 原理

```
memory_forget(memoryId) → 按 ID 精确删除
memory_forget(query="关键词") → 向量搜索 → 找到相似度 >90% 的唯一匹配 → 删除
                            → 多个匹配 → 返回候选列表让你选
```

**注意**：query 模式要求**单一高匹配**（>90%），否则会列出来让你确认，防止误删。

#### 清理频率建议

| 使用强度 | 建议清理频率 | 清理内容 |
|---|---|---|
| 轻度 | **不需要主动清理** | 自然增长，几年都不用管 |
| 中度 | 每 3-6 个月 | 删除过时偏好、已不再适用的决定 |
| 重度 | 每月 | 定期审计，合并重复，删除失效条目 |

**什么时候该清理？**
- ⚠️ 发现召回了明显过时的信息
- ⚠️ 同一件事有多条重复/矛盾的记忆
- ⚠️ 换了工作/地址/偏好等根本性变化

**怎么清理？**

```bash
# 查看所有记忆
openclaw ltm list --limit 50

# 按条件筛选
openclaw ltm query --filter "category = 'preference'" --order-by createdAt:desc

# 对话中让 AI 帮你删
memory_forget(query="旧的地址")
```

> 💡 **大多数情况下不需要主动清理**。原因：
> 1. 存储开销很小（几十 MB 级别）
> 2. 去重机制防止重复
> 3. `memory_recall` 有 `minScore` 阈值，过时记忆自然沉底不会被召回
> 4. 真正需要清理的场景是**内容过时**而非空间不足

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

## 4.8 关于“做梦”（dreaming）

做梦是**内置 memory-core** 的功能：每天定时把短期记忆整理提升进 `MEMORY.md`。默认凌晨 3 点运行，默认开启。

### 适用前提

⚠️ **如果你切换到了 memory-lancedb，memory-core 被禁用，做梦功能不再生效。** LanceDB 本身存的每条记忆都能直接被检索，不需要“提升”这一步。

做梦功能仅在使用内置 memory-core 后端时可用。

### Dreaming 详细原理

Dreaming 模拟人类睡眠时的记忆整理过程，分三个阶段：

| 阶段 | 作用 | 是否写入 MEMORY.md |
|---|---|---|
| **Light（浅睡）** | 整理最近短期记忆，去重暂存 | ❌ |
| **REM（快速眼动）** | 反思主题和反复出现的想法 | ❌ |
| **Deep（深睡）** | 评分筛选，提升到长期记忆 | ✅ |

**短期记忆来源：** daily notes（`memory/YYYY-MM-DD.md`）、会话转录（redacted session transcripts）、以及短期召回状态（recall store）。

**工作流程：**
1. Light 阶段读取最近的短期记忆和 daily notes，去重并暂存候选
2. REM 阶段反思主题，生成反思摘要
3. Deep 阶段对候选评分，超过阈值的提升到 `MEMORY.md`

### 是否需要嵌入模型

**Dreaming 本身不需要嵌入模型。** 它的 Deep 阶段评分排名基于 6 个**确定性信号**（不依赖向量）：

| 信号 | 权重 | 说明 |
|---|---|---|
| 相关性 (Relevance) | 0.30 | 检索质量均值 |
| 频率 (Frequency) | 0.24 | 短期信号累积次数 |
| 查询多样性 (Query diversity) | 0.15 | 不同查询/天上下文出现次数 |
| 新近度 (Recency) | 0.15 | 时间衰减新鲜度 |
| 巩固度 (Consolidation) | 0.10 | 多日反复出现强度 |
| 概念丰富度 (Conceptual richness) | 0.06 | 概念标签密度 |

但 **memory-core 的记忆搜索功能**（`memory_recall`）需要嵌入模型：
- 关键词搜索（BM25/FTS5）→ 不需要嵌入模型
- 向量搜索 → 需要嵌入模型
- 默认混合搜索 → 需要嵌入模型

**没有嵌入模型时**，只有关键词搜索可用，向量搜索会降级。

### 开启做梦

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

### Dreaming 的安全保护

- 提升前会备份旧的 `MEMORY.md` 到 SQLite
- 有 `maxPriorEntryLossFraction`（默认 0.25）防止删除过多旧条目
- 有 bootstrap-safe 文件预算，防止 `MEMORY.md` 无限增长
- 来源为 `untrusted` 或 `system` 的候选会被结构性排除

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
