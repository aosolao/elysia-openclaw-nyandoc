# OpenClaw 安装与配置完全指南

> 版本：v1.4.0
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)

> 本指南记录了从"刚装好 OpenClaw"到"配置成一个安全、好用、有记忆的个人 AI 助手"的全过程。
> 兼顾技术小白和硬核用户：每一节先讲"是什么、为什么"，再给"怎么配、有哪些选项、推荐值、影响"。
> 覆盖 **macOS / Linux / Windows** 三大系统，平台差异会单独标注。

<!-- TOC START -->
## 目录

- [文档版本约定](#文档版本约定)
- [这台机器的最终配置快照（参考）](#这台机器的最终配置快照参考)
- [章节导航](#章节导航)
- [阅读建议](#阅读建议)
- [通用约定](#通用约定)

<!-- TOC END -->

## 文档版本约定

每份文档标题下方标注版本号，遵循语义化版本：

- **修订号（v1.0.x）**：错别字、格式、注释等不改变含义的修改。
- **次版本（v1.x.0）**：新增内容、补充说明、向后兼容的改动。
- **主版本（vx.0.0）**：大重构、章节重组或不兼容的改写。

修改任何文档时，必须同步递增其版本号。

## 这台机器的最终配置快照（参考）

本文档基于以下实际环境整理，可作为你的配置参考：

| 项目 | 选择 |
|---|---|
| 系统 | macOS（Apple Silicon） |
| 主模型 | doubao/ark-code-latest（豆包） |
| 备用模型 | bailian/qwen3.7-plus（阿里百炼） |
| 模型超时 | 300 秒 |
| Gateway 绑定 | 127.0.0.1（仅本机） |
| 认证 | token（存在 macOS Keychain） |
| 执行审批 | cautious（白名单 + 未命中询问，拒绝兜底） |
| 内联代码防御 | strictInlineEval 开启 |
| 记忆后端 | memory-lancedb（本地向量库） |
| Embedding | 本地 vLLM-MLX 跑 multilingual-e5-large（1024 维） |
| Web 搜索 | DuckDuckGo（免 key）；稳定可换 Brave Search API |
| 浏览器 | Chrome 隔离 profile（openclaw，CDP 18800） |
| 渠道 | Telegram（DM 白名单 + owner） |
| 工作区 | `~/.openclaw/workspace`（记忆/人格） |
| 项目目录 | `~/.openclaw/workspace/openclaw-work`（原路径软链兼容） |
| 沙箱 | 全部会话 Docker 隔离（mode=all, scope=agent） |
| 沙箱镜像 | `YOUR_USER/openclaw-sandbox:py313-node26-trixie` |

## 章节导航

1. [安装与初始化](./01-installation.md)
2. [模型配置](./02-models.md)
3. [安全与执行审批](./03-security-and-exec.md)
4. [记忆系统](./04-memory.md)
5. [渠道配置（Telegram 等）](./05-channels.md)
6. [本地服务与插件](./06-local-services.md)
7. [工作区与项目目录](./07-workspace.md)
8. [日常维护与排错](./08-maintenance.md)
9. [沙箱与隔离环境](./09-sandbox.md)

## 阅读建议

- **小白用户**：按章节顺序读，重点看每节的"推荐设置"和"复制即用"命令。
- **硬核用户**：直接看每节的"选项详解"表格和"影响说明"。
- **Windows 用户特别注意**：OpenClaw 核心跨平台，但部分能力（如 macOS  companion app、Keychain）在 Windows 上有差异，相关章节会标注。

## 通用约定

- 命令前的 `$` 表示在终端执行，不需要输入 `$` 本身。
- 配置大多通过 `openclaw config set <key> <value>` 修改，**改完大多需要 `openclaw gateway restart`**（命令会提示）。
- 查看当前配置：`openclaw config get <key>`。
- 做任何大改动前，建议先备份配置：`cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak`。
