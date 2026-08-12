# 第 10 章 · 常用聊天命令速查

> 版本：v1.0.1
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [10.1 命令类型说明](#101-命令类型说明)
- [10.2 会话管理](#102-会话管理)
- [10.3 模型与思考控制](#103-模型与思考控制)
- [10.4 状态与诊断](#104-状态与诊断)
- [10.5 执行与权限](#105-执行与权限)
- [10.6 命令使用技巧](#106-命令使用技巧)

<!-- TOC END -->

本章汇总日常使用中最常用的聊天命令，方便快速查阅。完整命令列表见 [OpenClaw 官方文档](https://docs.openclaw.ai/tools/slash-commands)。

## 10.1 命令类型说明

OpenClaw 的聊天命令分三种类型：

| 类型 | 格式 | 说明 |
|---|---|---|
| **命令** | `/xxx`（独占消息） | 由 Gateway 处理，必须单独发送 |
| **指令** | `/xxx`（可内联） | 单独发送时持久化设置；与其他文本一起发送时仅作为本次提示 |
| **快捷方式** | `/xxx`（可内联） | 立即执行并返回结果 |

**指令类命令**（可内联）：`/think`、`/fast`、`/verbose`、`/trace`、`/reasoning`、`/elevated`、`/exec`、`/model`、`/queue`

## 10.2 会话管理

| 命令 | 说明 |
|---|---|
| `/new` | 归档当前会话，开始新会话 |
| `/new <model>` | 开始新会话并切换到指定模型 |
| `/reset` | 重置当前会话（保留会话 ID） |
| `/reset soft` | 软重置：保留对话记录，清除后端会话状态 |
| `/compact` | 压缩当前会话上下文 |
| `/compact <指令>` | 带提示的压缩（如 `/compact 保留技术细节`） |
| `/stop` | 中止当前正在执行的请求 |
| `/name <标题>` | 给当前会话命名 |

## 10.3 模型与思考控制

### 模型切换

| 命令 | 说明 |
|---|---|
| `/model` | 显示模型选择器 |
| `/model <名称>` | 切换到指定模型（owner/admin 会同时更新默认配置） |
| `/model <名称> -s` | 仅切换当前会话的模型 |
| `/model default` | 恢复使用配置的默认模型 |
| `/model status` | 查看当前模型详细信息 |
| `/models` | 列出所有可用模型 |

### 思考等级（Thinking）

| 命令 | 说明 |
|---|---|
| `/think <等级>` | 设置思考等级 |
| `/think default` | 清除会话覆盖，使用默认等级 |
| `/thinking` | 别名，同 `/think` |
| `/t` | 简写别名 |

**思考等级选项：**

| 等级 | 效果 |
|---|---|
| `off` | 不思考 |
| `minimal` | 最少思考 |
| `low` | 轻度思考（日常推荐） |
| `medium` | 中等思考 |
| `high` | 深度思考（最大预算） |
| `xhigh` | 超高思考（部分模型支持） |
| `adaptive` | 模型自适应（部分模型支持） |
| `max` | 最大推理努力（部分模型支持） |
| `ultra` | 最大推理 + 主动子代理编排（部分模型支持） |

### 推理内容显示（Reasoning）

| 命令 | 说明 |
|---|---|
| `/reasoning on` | 开启显示推理内容（思考过程会显示在回复中） |
| `/reasoning off` | 关闭显示推理内容（默认状态） |
| `/reasoning stream` | 流式显示推理内容（实时看到思考过程） |
| `/reason` | 别名，同 `/reasoning` |

> ⚠️ **安全提示**：`/reasoning`、`/verbose`、`/trace` 在群聊中可能有风险，会暴露内部推理过程。群聊中建议保持关闭。

### 快速模式

| 命令 | 说明 |
|---|---|
| `/fast` | 显示当前快速模式状态 |
| `/fast on` | 开启快速模式 |
| `/fast off` | 关闭快速模式 |
| `/fast auto` | 自动模式 |
| `/fast default` | 恢复默认设置 |

> **注意**：快速模式的效果取决于模型提供商。部分提供商会映射到 `service_tier=priority`；另一部分会映射到 `service_tier=auto` 或 `standard_only`。

## 10.4 状态与诊断

| 命令 | 说明 |
|---|---|
| `/status` | 显示执行状态、运行时间、插件健康、模型配额等 |
| `/status plugins` | 显示详细插件健康状态 |
| `/help` | 显示帮助摘要 |
| `/commands` | 显示完整命令列表 |
| `/tools` | 显示当前可用的工具 |
| `/tools verbose` | 显示工具及简短描述 |
| `/whoami` | 显示你的发送者 ID |
| `/usage` | 控制每条回复的 token/成本显示 |
| `/usage tokens` | 显示 token 用量 |
| `/usage full` | 显示完整信息 |
| `/usage off` | 关闭显示 |

## 10.5 执行与权限

### Elevated 模式

| 命令 | 说明 |
|---|---|
| `/elevated` | 显示当前状态 |
| `/elevated on` | 开启 elevated 模式（命令直接在宿主机执行） |
| `/elevated off` | 关闭 elevated 模式 |
| `/elevated ask` | 每次都需要确认 |
| `/elevated full` | 完全信任模式 |
| `/elev` | 别名，同 `/elevated` |

### Exec 控制

| 命令 | 说明 |
|---|---|
| `/exec` | 显示当前 exec 设置 |
| `/exec host=<auto\|sandbox\|gateway\|node>` | 设置执行主机 |
| `/exec security=<deny\|allowlist\|full>` | 设置安全级别 |
| `/exec ask=<off\|on-miss\|always>` | 设置询问策略 |

### 其他

| 命令 | 说明 |
|---|---|
| `/verbose on\|off\|full` | 切换详细输出模式 |
| `/trace on\|off` | 切换插件追踪输出 |
| `/queue <模式>` | 管理活跃请求队列行为 |
| `/steer <消息>` | 向正在执行的请求注入指导信息 |
| `/btw <问题>` | 侧问：基于当前会话上下文提问，不影响后续对话 |
| `/skill <名称> [输入]` | 运行指定技能 |
| `/approve <id> <决定>` | 批准执行或插件审批请求 |
| `/loop [间隔] <提示>` | 创建循环任务（owner-only） |
| `/loop status` | 查看当前会话的循环任务 |
| `/loop stop [名称]` | 停止循环任务 |

## 10.6 命令使用技巧

### 命令与文本组合

指令类命令可以与其他文本组合使用：

```text
/think high 帮我分析这个复杂的架构问题...
```

这会：
1. 临时将思考等级设为 `high`
2. 发送后面的问题
3. **不会**持久化思考等级设置（仅对本次消息生效）

### 持久化设置

如果想持久化设置，单独发送命令：

```text
/think high
```

这会：
1. 将思考等级设为 `high`
2. 回复确认信息
3. **持久化**到当前会话

### 命令参数分隔

命令和参数之间可以加 `:` 也可以不加：

```text
/think high
/think: high
```

两种写法效果相同。

### 群聊注意事项

在群聊中使用命令时注意：
- `/reasoning`、`/verbose`、`/trace` 可能暴露内部信息，建议关闭
- `/model` 切换会影响整个会话，群聊中谨慎使用
- 大部分命令仅对授权用户（authorized senders）生效
