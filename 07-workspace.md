# 第 7 章 · 工作区与项目目录

> 版本：v1.2.1
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)

> ⚠️ **沙箱用户注意**：启用沙箱后（第 9 章），bind mount 只允许源路径在 `~/.openclaw/workspace` 内，且软链会被 realpath 校验拒绝。项目目录必须放在 workspace 内，或在 workspace 内建目录后软链到外部。


<!-- TOC START -->
## 目录

- [7.1 两个目录的区别（小白必懂）](#71-两个目录的区别小白必懂)
- [7.2 推荐做法：workspace 不动，项目单独放](#72-推荐做法workspace-不动项目单独放)
  - [配置步骤](#配置步骤)
- [7.3 （可选）方案二：符号链接](#73-可选方案二符号链接)
- [7.4 （可选）方案三：直接换 workspace](#74-可选方案三直接换-workspace)
- [7.5 workspace 里的关键文件](#75-workspace-里的关键文件)
- [7.6 workspace 应当用 Git 管理（重要）](#76-workspace-应当用-git-管理重要)
  - [初始化步骤](#初始化步骤)
  - [关于 Fork 的 token 限制](#关于-fork-的-token-限制)
  - [Git 提交的额外好处](#git-提交的额外好处)
- [7.7 平台路径速查](#77-平台路径速查)
- [7.8 下一步](#78-下一步)

<!-- TOC END -->

## 7.1 两个目录的区别（小白必懂）

OpenClaw 有两个容易混淆的目录概念：

| 目录 | 作用 | 类比 |
|---|---|---|
| **workspace（工作区）** | AI 的"家"，放人格、记忆、规则、笔记 | 你家的书房和日记本 |
| **项目目录** | 实际写代码、产出文件的地方 | 你上班的工位 |

默认 workspace 是 `~/.openclaw/workspace`。建议**把记忆和项目分开**：让 workspace 保持干净（只放 AI 自己的东西），项目产出放到另一个固定目录。

## 7.2 推荐做法：workspace 不动，项目单独放

最简单有效的方案（本指南作者也这么用）：

- workspace 保持默认 `~/.openclaw/workspace`
- 建一个项目目录，如 `~/ai-workspace/openclaw-work`
- 在 workspace 的 `AGENTS.md` 里写清楚规则，让 AI 记住

### 配置步骤

**第 1 步：创建项目目录**

- macOS/Linux：
  ```bash
  mkdir -p ~/ai-workspace/openclaw-work
  ```
- Windows (PowerShell)：
  ```powershell
  New-Item -ItemType Directory -Force -Path "$HOME\ai-workspace\openclaw-work"
  ```

**第 2 步：在 workspace 的 `AGENTS.md` 里加一段**

macOS/Linux 示例：

```markdown
## Project Directory

The workspace (~/.openclaw/workspace) is home for memory, personality, and notes only.
For actual project work (code, files, builds, scripts, outputs), use:

    /Users/YOUR_USER/ai-workspace/openclaw-work

Rules:
- When running project-related commands, set workdir to that path or cd there first.
- When reading/writing project files, use absolute paths under that directory.
- Do NOT create project files in the workspace unless explicitly asked.
- Memory files (MEMORY.md, memory/, SOUL.md, etc.) stay in the workspace.
```

Windows 示例：

```markdown
## Project Directory

For actual project work, use:

    C:\Users\YOUR_USER\ai-workspace\openclaw-work

Rules:
- When running project-related commands, set workdir to that path or cd there first.
- When reading/writing project files, use absolute paths under that directory.
- Do NOT create project files in the workspace unless explicitly asked.
- Memory files (MEMORY.md, memory/, SOUL.md, etc.) stay in the workspace.
```

`AGENTS.md` 每次会话自动加载，AI 会永久遵守。

## 7.3 （可选）方案二：符号链接

如果你想让 AI 在 workspace 里用相对路径访问项目，可以建软链：

- macOS/Linux：
  ```bash
  ln -s ~/ai-workspace/myproject ~/.openclaw/workspace/myproject
  ```
- Windows（管理员命令行）：
  ```cmd
  mklink /D C:\Users\YOUR_USER\.openclaw\workspace\myproject C:\path\to\myproject
  ```

## 7.4 （可选）方案三：直接换 workspace

如果你确实想让 workspace 指向别处：

```bash
openclaw config set agents.defaults.workspace "/新的/绝对路径"
openclaw gateway restart
```

⚠️ **注意：这会让 AI 找不到原 workspace 里的 `MEMORY.md`、`SOUL.md`、`AGENTS.md` 等，需要手动把这些文件复制到新目录，否则 AI 会"失忆"且丢失人格。** 一般不推荐。

## 7.5 workspace 里的关键文件

| 文件 | 作用 | 谁维护 |
|---|---|---|
| `AGENTS.md` | AI 的行为规则 | 你和 AI 共同编辑 |
| `SOUL.md` | 人格/语气设定 | 你定义 |
| `USER.md` | 关于你的信息 | 你和 AI |
| `IDENTITY.md` | AI 身份 | 你定义 |
| `MEMORY.md` | 长期记忆（精炼） | AI 维护 |
| `memory/YYYY-MM-DD.md` | 每日笔记 | AI 维护 |
| `TOOLS.md` | 本机工具备注（SSH、设备名等） | 你和 AI |
| `HEARTBEAT.md` | 定时巡检清单 | 按需 |

## 7.6 workspace 应当用 Git 管理（重要）

OpenClaw 的会话 Fork 功能底层调用 `git worktree` 给分支会话创建独立工作目录。如果 workspace 的 Git 仓库**没有任何提交**（`git status` 显示 `No commits yet`），Fork 会失败：

```
git worktree add failed:
fatal: invalid reference: HEAD
```

而且前端不会提示错误，表现就是"右键 Fork 什么都没发生"。

### 初始化步骤

```bash
cd ~/.openclaw/workspace

# 1. 写 .gitignore（排除临时文件和系统垃圾）
cat > .gitignore <<'EOF'
# macOS
.DS_Store

# 临时文件（用完即删）
tmp/

# 日志
*.log
EOF

# 2. 首次提交
git add -A
git commit -m "Initial commit: workspace bootstrap"

# 3. 验证 worktree 机制可用
git worktree add /tmp/openclaw-fork-test HEAD && echo WORKTREE_OK
git worktree remove /tmp/openclaw-fork-test --force
```

看到 `WORKTREE_OK` 就说明 Fork 的底层依赖正常了。

### 关于 Fork 的 token 限制

Fork 还有一个**硬编码的 100,000 token 上限**，不可通过配置修改。父会话超过这个大小时，后端报错：

```
parent session is too large to fork (109304/100000 tokens)
```

这个限制写在 OpenClaw 源码里（`DEFAULT_PARENT_FORK_MAX_TOKENS = 1e5`），是保护机制，防止分支会话继承接近满的上下文。应对方法：

- 大会话先 `/compact` 压缩后再 Fork。
- 或直接开 New Chat，AI 会从 MEMORY.md/AGENTS.md 继承上下文。
- 不要直接改 dist 源码去调大限制——升级会被覆盖，且失去保护意义。

### Git 提交的额外好处

即使不用 Fork，给 workspace 建 Git 也很值：
- 人格、记忆、规则文件的改动有历史可查、可回滚。
- 误删 MEMORY.md 或 SOUL.md 时能恢复。
- 纯本地仓库不需要远程（`git remote` 为空即可）。

## 7.7 平台路径速查

| 系统 | workspace 默认路径 |
|---|---|
| macOS | `/Users/YOUR_USER/.openclaw/workspace` |
| Linux | `/home/YOUR_USER/.openclaw/workspace` |
| Windows | `C:\Users\YOUR_USER\.openclaw\workspace` |

## 7.8 下一步

- 日常维护与排错 → [第 8 章 日常维护与排错](./08-maintenance.md)
