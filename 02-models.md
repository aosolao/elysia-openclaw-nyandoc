# 第 2 章 · 模型配置

> 版本：v1.2.1
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [2.1 基本概念（小白先看）](#21-基本概念小白先看)
- [2.2 配置主模型与备用模型](#22-配置主模型与备用模型)
  - [给模型起别名（可选，方便切换）](#给模型起别名可选方便切换)
- [2.3 配置自定义/国内提供商](#23-配置自定义国内提供商)
  - [示例：豆包（火山方舟）](#示例豆包火山方舟)
  - [示例：阿里百炼](#示例阿里百炼)
  - [提供商字段说明](#提供商字段说明)
- [2.4 超时设置（重要，避免长任务中断）](#24-超时设置重要避免长任务中断)
- [2.5 安全地存放 API Key（不要明文）](#25-安全地存放-api-key不要明文)
  - [方式 A：环境变量（最简单，全平台通用）](#方式-a环境变量最简单全平台通用)
  - [方式 B：系统密钥库（更安全，推荐）](#方式-b系统密钥库更安全推荐)
- [2.6 模型选择器与别名（重要）](#26-模型选择器与别名重要)
  - [登记模型并起别名](#登记模型并起别名)
  - [默认模型与 fallback](#默认模型与-fallback)
- [2.7 心跳模型与其他专用模型（可选）](#27-心跳模型与其他专用模型可选)
  - [2.7.1 修改默认模型](#271-修改默认模型)
  - [2.7.2 给心跳单独配模型](#272-给心跳单独配模型)
  - [2.7.3 其他可单独覆盖模型的场景](#273-其他可单独覆盖模型的场景)
- [2.8 思考过程（Thinking / Reasoning）](#28-思考过程thinking-reasoning)
  - [思考等级](#思考等级)
  - [设置默认思考等级](#设置默认思考等级)
  - [对话中临时切换](#对话中临时切换)
  - [让模型“支持思考”的配置](#让模型支持思考的配置)
  - [显示效果](#显示效果)
- [2.9 常用模型相关命令](#29-常用模型相关命令)
- [2.10 配置建议（不同场景）](#210-配置建议不同场景)
- [2.11 下一步](#211-下一步)

<!-- TOC END -->

## 2.1 基本概念（小白先看）

OpenClaw 自己没有模型，它把你的话转发给"大模型提供商"（豆包、百炼、OpenAI 等），拿到回复再展示给你。你需要：

1. **主模型（primary）**：平时对话用的。
2. **备用模型（fallbacks）**：主模型挂了/超时时自动顶上。
3. **API key**：调用模型的钥匙。

## 2.2 配置主模型与备用模型

配置写在 `~/.openclaw/openclaw.json` 的 `agents.defaults.model` 下，推荐用命令改：

```bash
# 主模型
openclaw config set agents.defaults.model.primary "doubao/ark-code-latest"

# 备用模型（可多个，按顺序尝试）
openclaw config set agents.defaults.model.fallbacks '["bailian/qwen3.7-plus"]' --strict-json
```

模型引用格式是 `提供商/模型ID`，例如：
- `openai/gpt-4o`
- `doubao/ark-code-latest`
- `bailian/qwen3.7-plus`
- `anthropic/claude-sonnet-4`
- 本地模型：`ollama/llama3`（需先装 Ollama）

### 给模型起别名（可选，方便切换）

别名和模型选择器的完整说明在 [2.6 节](#26-模型选择器与别名重要)。最简单的方式：

```bash
openclaw config set agents.defaults.models.doubao/ark-code-latest.alias "doubao"
```

之后在对话里用 `/model doubao` 就能快速切换。

## 2.3 配置自定义/国内提供商

如果用的是 OpenAI 兼容接口（豆包、百炼、本地 vLLM 等），在 `models.providers` 里加一个提供商。

### 示例：豆包（火山方舟）

```bash
openclaw config set models.providers.doubao.baseUrl "https://ark.cn-beijing.volces.com/api/plan/v3"
openclaw config set models.providers.doubao.api "openai-responses"
```

API key 不要明文写进配置（见 2.5）。

### 示例：阿里百炼

```bash
openclaw config set models.providers.bailian.baseUrl "https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1"
openclaw config set models.providers.bailian.api "openai-completions"
```

### 提供商字段说明

| 字段 | 作用 | 常见值 |
|---|---|---|
| `baseUrl` | 接口地址 | 各厂商的 endpoint |
| `api` | 接口协议 | `openai-completions`、`openai-responses`、`anthropic-messages`、`ollama` |
| `apiKey` | 密钥 | 建议用 SecretRef 或环境变量，不写明文 |
| `timeoutSeconds` | 请求超时（秒） | 默认约 120，慢模型调大 |
| `models` | 该提供商下的模型列表 | 数组，含 id、contextWindow、maxTokens 等 |

## 2.4 超时设置（重要，避免长任务中断）

默认模型空闲超时约 120 秒，写长文档、跑复杂任务时容易触发。建议调大：

```bash
# 豆包超时设为 300 秒
openclaw config set models.providers.doubao.timeoutSeconds 300

# 百炼也设上
openclaw config set models.providers.bailian.timeoutSeconds 300
```

**影响说明：**
- 调大后，模型思考/生成时间更长也不会被中断。
- 但如果模型真的卡死，你要等更久才会报错并切到备用模型。
- 建议值：普通对话 120s，长文档/编码任务 300s，本地慢模型可设 600s。

> 还有一个整轮运行上限 `agents.defaults.timeoutSeconds`（默认 600），提供商超时不能超过它；如果你把提供商超时设得很大，也要相应调大这个值：
> ```bash
> openclaw config set agents.defaults.timeoutSeconds 900
> ```

## 2.5 安全地存放 API Key（不要明文）

### 方式 A：环境变量（最简单，全平台通用）

```bash
# macOS/Linux：写进 ~/.zshrc 或 ~/.bashrc
export DOUBAO_API_KEY="你的key"
```

配置里引用：
```bash
openclaw config set models.providers.doubao.apiKey '${DOUBAO_API_KEY}'
```

Windows (PowerShell)：
```powershell
setx DOUBAO_API_KEY "你的key"
```

### 方式 B：系统密钥库（更安全，推荐）

- **macOS**：用 Keychain，配合一个读取脚本（本指南第 6 章有示例）。
- **Linux**：用 `secret-tool`（libsecret）或 `pass`。
- **Windows**：用凭据管理器（Credential Manager），通过 PowerShell `Get-StoredCredential`。

OpenClaw 支持 `exec` 类型的 SecretRef，即"运行一条命令来获取密钥"，这样配置文件里永远不出现明文：

```json
"apiKey": {
  "source": "exec",
  "provider": "keychain_doubao",
  "id": "value"
}
```

## 2.6 模型选择器与别名（重要）

网页控制台和聊天窗口的**模型下拉选择器**默认由 `agents.defaults.models` 这个白名单驱动：只有登记在这里的模型才会出现在选择器里。很多人配置了一堆 `models.providers.*.models` 却发现选择器里只显示一个，原因就是忘了在这里登记。

### 登记模型并起别名

```bash
openclaw config set agents.defaults.models '{
  "doubao/ark-code-latest": {"alias": "doubao"},
  "bailian/qwen3.7-plus": {"alias": "qwen"},
  "bailian/qwen3.6-flash": {"alias": "qwen-flash"},
  "bailian/glm-5.2": {"alias": "glm-5.2"},
  "bailian/deepseek-v4-pro": {"alias": "ds-v4-pro"}
}' --strict-json
```

- key 是完整的 `提供商/模型ID`。
- `alias` 是别名，之后用 `/model <别名>` 可以快速切换。
- 刷新网页后，选择器就会列出所有登记的模型。
- 如果不设 `agents.defaults.models`，选择器会显示所有有可用认证的提供商的显式模型条目（但可控性差，建议显式登记）。

### 默认模型与 fallback

```bash
openclaw config set agents.defaults.model.primary "doubao/ark-code-latest"
openclaw config set agents.defaults.model.fallbacks '["bailian/qwen3.7-plus"]' --strict-json
```

fallback 在主模型超时、报错或配额用完时自动顶上（按顺序尝试）。建议配置一个不同提供商的备用模型，避免单一服务故障。

## 2.7 心跳模型与其他专用模型（可选）

### 2.7.1 修改默认模型

默认主模型由 `agents.defaults.model.primary` 控制。用 `config set` 改最安全（标量值，不涉及数组替换，见 [第 8 章 8.4 节](./08-maintenance.md#84-常见问题排查) 的 config patch 注意事项）：

```bash
# 改成百炼 qwen3.8-max 为例
openclaw config set agents.defaults.model.primary "bailian/qwen3.8-max"

# 备用模型链（主模型失败时依次 fallback）
openclaw config set agents.defaults.model.fallbacks '["bailian/qwen3.7-plus"]' --strict-json
```

- 模型名必须是 `提供商/模型ID` 格式，且已在 `models.providers.<提供商>.models` 中注册。
- 改完即时生效，新会话即用；当前会话可发 `/new` 开新会话，或在网页右上角切换。
- **只临时切换当前会话、不改默认**：对话里发 `/model <别名>`，或用网页会话的模型选择器。
- 项目仓库提供交互式脚本 `scripts/set-model-interactive.sh`：自动列出可用模型，依次选择默认模型、心跳模型、心跳间隔、isolatedSession、lightContext，确认后才写入配置。适合不想手写 JSON/命令的用户。

### 2.7.2 给心跳单独配模型

心跳（heartbeat）是 OpenClaw 定时唤醒助手做检查的机制（邮件、日历、提醒等，见工作区 `AGENTS.md` 的 Heartbeat 一节）。它默认用主模型，但可以单独指定一个更快/更省的模型：

```bash
# 单独设置心跳模型
openclaw config set agents.defaults.heartbeat.model "bailian/qwen3.6-flash"

# 心跳频率
openclaw config set agents.defaults.heartbeat.every "30m"
```

心跳的两个关键开关：

```bash
# 在独立会话里跑心跳，不污染主会话（推荐）
openclaw config set agents.defaults.heartbeat.isolatedSession true

# 用最小上下文，省 token
openclaw config set agents.defaults.heartbeat.lightContext true
```

> ⚠️ **心跳模型串味（bleed）警告**：如果不开 `isolatedSession`，心跳和主对话共用同一个会话，心跳跑完可能把该会话“留在”心跳用的小模型上。若小模型的上下文窗口比主模型小，下一轮主对话就可能报上下文溢出。官方建议：要么开 `isolatedSession: true`，要么确保心跳模型的 `contextWindow` 足够大。开了 isolated 还可以配 `lightContext: true` 进一步省 token。

### 2.7.3 其他可单独覆盖模型的场景

除了心跳，以下场景也支持单独指定模型（需要时再查官方文档）：

- **子代理（subagents）**：派生的子任务可用不同模型。
- **上下文压缩（compaction）**：长对话压缩摘要可指定模型。
- **渠道覆盖（channel overrides）**：某个聊天渠道可用不同模型。

这些都遵循“不设则继承默认主模型”的原则，日常只配主模型 + 心跳模型即可。

## 2.8 思考过程（Thinking / Reasoning）

很多新一代模型（豆包 ark-code、Qwen3、DeepSeek V4、GLM-5 等）支持“思考”——在正式回答前先输出一段推理过程。OpenClaw 可以把这段思考以可折叠块显示在回复上方。

### 思考等级

| 等级 | 效果 |
|---|---|
| `off` | 不思考 |
| `minimal` | 最少思考（类似 think） |
| `low` | 轻度思考（日常推荐） |
| `medium` | 中等思考 |
| `high` | 深度思考（类似 ultrathink） |
| `xhigh` / `max` | 最大努力（仅部分模型支持） |
| `adaptive` | 模型自适应（仅部分模型支持） |

等级越高，思考越久、token 消耗越多，复杂问题质量更好。

### 设置默认思考等级

```bash
# 全局默认
openclaw config set agents.defaults.thinkingDefault low

# 给每个模型单独设默认（会覆盖全局）
# 在 agents.defaults.models 里加 params.thinking
openclaw config set agents.defaults.models '{
  "doubao/ark-code-latest": {"alias":"doubao","params":{"thinking":"low"}},
  "bailian/qwen3.7-plus": {"alias":"qwen","params":{"thinking":"low"}}
}' --strict-json
```

### 对话中临时切换

只发一条命令（不带其他内容）会记住当前会话的设置：

```text
/think medium      # 本会话用中等思考
/think high        # 深度思考
/think off         # 关闭思考
/think             # 查看当前等级
```

单条消息临时指定（在消息开头加）：

```text
/t high 帮我分析这个复杂问题……
```

网页控制台顶部工具栏也有思考等级选择器，可直接点选。

### 让模型“支持思考”的配置

模型 catalog 里需要 `reasoning: true`，OpenClaw 才会给它发送思考参数并渲染思考块。豆包 ark-code 默认就是 true；但用 OpenAI 兼容接口接入的模型（如百炼上的 Qwen3/GLM/DeepSeek）可能需要手动标记：

```bash
# 把百炼模型的 reasoning 设为 true 并指定思考格式
# Qwen3 系列需要 compat.thinkingFormat = "qwen"（发送 chat_template_kwargs.enable_thinking）
openclaw config set models.providers.bailian.models '[
  {
    "id": "qwen3.7-plus",
    "name": "Qwen3.7 Plus",
    "reasoning": true,
    "input": ["text","image"],
    "contextWindow": 1048576,
    "maxTokens": 8192,
    "compat": { "thinkingFormat": "qwen" }
  }
]' --strict-json
```

> 如果多行 JSON 的 `config set` 命令报错（shell 转义问题），可以改用 `openclaw config edit` 直接打开配置文件编辑，或把 JSON 存成文件后用 `--strict-json` 传入。

不同模型的思考格式：

| 模型 | 思考格式 | 说明 |
|---|---|---|
| 豆包 ark-code | 内置（openai-responses） | reasoning: true 即可 |
| Qwen3 系列 | `compat.thinkingFormat: "qwen"` | 发送 enable_thinking 参数 |
| DeepSeek V4 | 原生 reasoning_content | reasoning: true 即可 |
| GLM-5.2 | 原生（off/low/high/max） | reasoning: true 即可 |
| Claude | adaptive / 等级 | 需官方 API |

> 注意：部分代理/套餐端点可能不支持思考参数，配置后如果 API 报错或没有思考内容，说明该端点不支持，换回普通模式即可。

### 显示效果

- 思考内容以可折叠块显示在正式回复**上方**。
- 简单问答可 `/think off` 关闭以省 token、降延迟。
- 如果开了思考却看不到块，先确认模型 catalog 的 `reasoning: true` 和提供商端点支持。

## 2.9 常用模型相关命令

```bash
openclaw models list              # 列出所有可用模型
openclaw models auth login        # 登录某个提供商（OAuth 类）
# 对话中临时切换模型：
/model <provider/model 或别名>
# 查看当前会话用的什么模型：
/status
```

## 2.10 配置建议（不同场景）

| 场景 | 建议 |
|---|---|
| 日常对话 | 选择响应快、成本适中的模型 |
| 复杂任务 | 选择推理能力强的模型，超时设长 |
| 本地隐私 | 使用本地模型（如 Ollama），数据不出本机 |
| 成本控制 | 选择性价比高的模型，或按需切换 |

具体模型选择请参考各厂商官方文档，根据实际需求测试后决定。

## 2.11 下一步

- 上锁安全 → [第 3 章 安全与执行审批](./03-security-and-exec.md)
