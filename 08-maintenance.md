# 第 8 章 · 日常维护与排错

> 版本：v1.3.1
> 📖 [elysia-openclaw-nyandoc](https://github.com/aosolao/elysia-openclaw-nyandoc) · [CC BY-NC-SA 4.0](./LICENSE)


<!-- TOC START -->
## 目录

- [8.1 日常检查三件套](#81-日常检查三件套)
- [8.2 升级](#82-升级)
- [8.3 日志](#83-日志)
- [8.4 常见问题排查](#84-常见问题排查)
  - [模型不回复 / 超时](#模型不回复-超时)
  - [命令审批弹窗不出现](#命令审批弹窗不出现)
  - [记忆搜不到](#记忆搜不到)
  - [Telegram 不响应](#telegram-不响应)
  - [Gateway 启动失败](#gateway-启动失败)
  - [配置改坏了打不开](#配置改坏了打不开)
  - [`openclaw config patch` 的三个坑](#openclaw-config-patch-的三个坑)
  - [stale session lock](#stale-session-lock)
  - [Fork 会话没反应（右键 Fork 无新窗口）](#fork-会话没反应右键-fork-无新窗口)
  - [Web 搜索不可用](#web-搜索不可用)
  - [记忆写不进去](#记忆写不进去)
  - [浏览器工具不可用](#浏览器工具不可用)
- [8.5 安全维护清单](#85-安全维护清单)
- [8.6 备份](#86-备份)
  - [8.6.1 备份什么](#861-备份什么)
  - [8.6.2 用备份脚本（推荐）](#862-用备份脚本推荐)
  - [8.6.3 直接用官方命令](#863-直接用官方命令)
- [8.7 资源占用参考](#87-资源占用参考)
- [8.8 跨平台备忘](#88-跨平台备忘)
- [8.9 有用的命令速查](#89-有用的命令速查)
- [结语](#结语)

<!-- TOC END -->

## 8.1 日常检查三件套

```bash
openclaw status          # 整体状态（版本、网关、渠道、会话）
openclaw doctor          # 诊断常见问题
openclaw security audit  # 安全检查
```

建议每隔一段时间，或升级后跑一次。

## 8.2 升级

```bash
npm update -g openclaw
openclaw --version
openclaw gateway restart
openclaw doctor    # 升级后检查
```

升级前建议备份配置和记忆（见 8.6）。第三方插件升级：

```bash
openclaw plugins list
# 在对应插件目录 npm update，或重装
openclaw plugins install @openclaw/memory-lancedb
```

## 8.3 日志

```bash
openclaw logs --follow          # 实时日志
# macOS 直接看日志文件
tail -f /tmp/openclaw/openclaw-$(date +%F).log
# Linux (systemd)
journalctl --user -u openclaw-gateway -f
```

日志里搜 `error`、`timeout`、`failed` 来定位问题。

## 8.4 常见问题排查

### 模型不回复 / 超时

- 现象：`LLM idle timeout` 或一直卡住。
- 处理：
  1. 调大超时：`openclaw config set models.providers.<id>.timeoutSeconds 300`
  2. 检查 API key 是否有效、额度是否用完（百炼等有周配额）。
  3. 检查网络/代理；国内 API 确认没走代理或代理正确。
  4. 配 fallback 模型，主模型挂了自动切换。

### 命令审批弹窗不出现

- 确认 `openclaw exec-policy show` 的 effective 不是 `full/off`。
- 确认网页控制台开着（`http://127.0.0.1:18789`），或 Telegram message 工具已授权。
- 如果没有任何审批客户端且 askFallback=deny，未白名单命令会被直接拒绝。

### 记忆搜不到

```bash
openclaw ltm stats                 # LanceDB 是否有数据
openclaw ltm search "关键词"        # 直接测试
curl http://127.0.0.1:8001/v1/embeddings ...  # embedding 服务是否在跑
```

- embedding 服务挂了：重启它（macOS `launchctl load ...`，Linux `systemctl --user restart embedding`）。
- 新库记忆数为 0 是正常的，开始"记住"东西后才会有。
- 维度要和模型一致（e5=1024）。

### Telegram 不响应

- `openclaw status` 看 Telegram 渠道状态是否 OK。
- 确认 bot token 正确、allowFrom 是你的数字 ID。
- 确认 ownerAllowFrom 已设，否则特权命令无法用。
- 看日志里 telegram 相关报错。

### Gateway 启动失败

```bash
openclaw gateway status
openclaw doctor
# macOS
launchctl print gui/$(id -u)/ai.openclaw.gateway | tail -30
# Linux
systemctl --user status openclaw-gateway
```

常见原因：端口被占（改 `gateway.port`）、配置 JSON 语法错误（检查 `~/.openclaw/openclaw.json`）、密钥脚本没执行权限。

### 配置改坏了打不开

```bash
# 用备份恢复
cp ~/.openclaw/openclaw.json.bak ~/.openclaw/openclaw.json
openclaw gateway restart
# 验证配置
openclaw config validate
```

### `openclaw config patch` 的三个坑

修改嵌套配置（比如某个模型的 `contextWindow`、`timeoutSeconds`）时，`config patch` 很容易误伤，记牢这三点：

1. **不接受位置参数 JSON。**
   `openclaw config patch '{...}'` 会报 `Too many arguments`。必须走标准输入或文件：

   ```bash
   echo '{...}' | openclaw config patch --stdin
   openclaw config patch --file ./patch.json
   ```

2. **数组是整体替换，不是合并。** 这是最危险的一点。
   `models.providers.<id>.models` 是一个数组。如果你只 patch
   `{"id":"Qwen","contextWindow":196608}`，该模型条目的其他字段（`name`、`reasoning`、`input`、`cost`、`maxTokens`、`api`）会被全部清空。**必须为每个数组元素提供完整对象**，并同时用 `--replace-path` 指向该数组，整体原子替换：

   ```bash
   cat <<'EOF' | openclaw config patch --stdin \
     --replace-path 'models.providers.mycompany.models'
   {
     "models": {
       "providers": {
         "mycompany": {
           "models": [
             {
               "id": "Qwen",
               "name": "MyCompany API (Qwen3.6-35B-A3B)",
               "reasoning": false,
               "input": ["text", "image"],
               "cost": {"input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0},
               "contextWindow": 196608,
               "maxTokens": 8192,
               "api": "openai-completions"
             }
           ]
         }
       }
     }
   }
   EOF
   ```

   改多个 provider 的数组就重复传 `--replace-path`。路径里带连字符的（如 `mycompany-out`）要用括号写法：
   `--replace-path 'models.providers["mycompany-out"].models'`。

3. **根键是 `models`，不是 `providers`。** 所有 provider 都挂在 `models.providers.*` 下。顶层写 `{"providers":{...}}` 会报 schema 校验失败：`Unrecognized key: "providers"`。

**经验法则**：改单个标量值（超时、开关、数字、字符串）优先用
`openclaw config set <dot.path> <value>`，没有数组替换风险；只有在需要结构化合并或替换数组时才用 `config patch`，并先用 `openclaw config get` 看清现有结构。

### stale session lock

doctor 报 stale lock 文件，通常是上次异常退出残留：

```bash
openclaw doctor --fix
```

### Fork 会话没反应（右键 Fork 无新窗口）

Fork 失败时前端往往不弹错误，要查后端日志：

```bash
grep sessions.create /tmp/openclaw/openclaw-$(date +%F).log | tail -10
```

常见两种原因：

1. **`git worktree add failed: invalid reference: HEAD`**——workspace 的 Git 仓库没有任何提交。解决：在 workspace 里 `git add -A && git commit`（详见 [第 7 章 7.6 节](./07-workspace.md#76-workspace-应当用-git-管理重要)）。
2. **`parent session is too large to fork (109304/100000 tokens)`**——父会话超过 100k token 硬限制。解决：先 `/compact` 压缩，或开 New Chat。这个限制不可配置。

### Web 搜索不可用

报错 `web_search is disabled or no provider is available`：

```bash
# 查看当前搜索后端
openclaw config get tools.web.search
# 没设 provider 时返回空 {}
```

解决：设一个后端（详见 [第 6 章 6.5 节](./06-local-services.md#65-web-搜索配置)）：

```bash
openclaw config set tools.web.search.provider duckduckgo  # 免 key
```

国内网络还要确认 Gateway 进程走了代理（见 [6.4 节](./06-local-services.md#64-网络代理国内网络环境)）。

### 记忆写不进去

现象：说"记住：我的服务器 IP 是 192.168.1.100"后，下次会话想不起来。按顺序排查：

1. **工具被 profile 屏蔽**——日志里有 `tool policy removed ... memory_store`。解决：把 memory 三件套加进 alsoAllow（见 [第 4 章 4.5 节](./04-memory.md#45-让记忆工具真正可用tool-profile-与钩子权限)）。
2. **agent_end 钩子被拦**——日志里有 `typed hook "agent_end" blocked`。解决：`openclaw config set plugins.entries.memory-lancedb.hooks.allowConversationAccess true` 后重启。
3. **autoCapture 关闭**——如果想让 AI 自动存记忆，设 `autoCapture true`；否则要手动说"记住"。
4. **embedding 服务挂了**——`curl http://127.0.0.1:8001/v1/models` 检查，挂了就重启 embedding 服务。

```bash
# 综合检查
grep -E "memory|tool policy" /tmp/openclaw/openclaw-$(date +%F).log | tail -20
ls -la ~/.openclaw/memory/lancedb/memories.lance/data/
```

### 浏览器工具不可用

报错 `plugins.allow excludes "browser"`：

```bash
openclaw config set plugins.allow '["memory-lancedb","browser"]' --strict-json
openclaw config set browser.enabled true
openclaw config set browser.defaultProfile openclaw
# 同时确保 tools.alsoAllow 里有 browser
openclaw gateway restart
openclaw browser --browser-profile openclaw doctor
```

详见 [第 6 章 6.6 节](./06-local-services.md#66-浏览器自动化工具)。

## 8.5 安全维护清单

定期确认：

- [ ] `gateway.bind` 是 loopback（除非你明确需要远程）。
- [ ] token/API key 没有明文写在配置里。
- [ ] exec policy 是 cautious（allowlist + on-miss + deny fallback）。
- [ ] `commands.ownerAllowFrom` 设的是你自己。
- [ ] Telegram/Discord 等渠道 allowFrom 没有陌生人。
- [ ] `openclaw security audit` 无 critical。
- [ ] 系统和 OpenClaw 都更新到最新补丁。

## 8.6 备份

### 8.6.1 备份什么

需要持久化的内容（含密钥和隐私，备份需加密存储）：

| 路径 | 内容 |
|---|---|
| `~/.openclaw/openclaw.json` | 主配置 |
| `~/.openclaw/exec-approvals.json` | 审批策略/白名单 |
| `~/.openclaw/workspace/` | 人格、记忆、笔记 |
| `~/.openclaw/memory/lancedb/` | LanceDB 向量记忆 |
| `~/.openclaw/agents/` | agent 数据/索引 |
| `~/.openclaw/credentials/` | 凭据目录（若在外部） |

macOS 可用 Time Machine 整体备份 `~/.openclaw`；Linux 用 rsync/restic；Windows 用文件历史或备份软件。

### 8.6.2 用备份脚本（推荐）

项目提供专用脚本：

```
scripts/backup-openclaw.sh
```

脚本底层只调用官方命令 `openclaw backup create --verify`，不自行打包，保证归档格式官方可恢复。纯手动调用，不挂任何定时任务。

**用法**：

```bash
# 完整备份（状态+配置+凭据+工作区），先预览再确认
./scripts/backup-openclaw.sh

# 轻量备份：跳过工作区（工作区已用 Git 管理时日常推荐，小而快）
./scripts/backup-openclaw.sh --light

# 最小备份：只备份活动配置文件
./scripts/backup-openclaw.sh --config-only

# 跳过交互确认，直接执行
./scripts/backup-openclaw.sh --light --yes

# 自定义输出目录
./scripts/backup-openclaw.sh --output ~/Backups
```

**可配置项**（用环境变量覆盖默认值）：

| 变量 | 默认 | 说明 |
|---|---|---|
| `BACKUP_OUTPUT_DIR` | `~/Backups/openclaw` | 备份归档输出目录 |
| `BACKUP_YES` | `0` | 设为 `1` 等同始终传 `--yes` |

**行为说明**：

- 每次先跑 `--dry-run --json` 预览会备份哪些路径，确认后才正式执行。
- 归档为带时间戳的 `.tar.gz`，**同名文件绝不覆盖**。
- `--verify` 写完即校验完整性（manifest 唯一、无越界路径、所有载荷存在）。
- SQLite 用 `VACUUM INTO` 安全快照；自动跳过活动会话、cron 日志、投递队列、socket/pid 等易变文件。
- 插件 `node_modules/` 不打包（可重建）；恢复后若报缺依赖，用 `openclaw plugins update <id>` 或 `openclaw plugins install <spec> --force`。

### 8.6.3 直接用官方命令

```bash
openclaw backup create --verify                          # 完整备份到当前目录
openclaw backup create --output ~/Backups --verify       # 指定输出目录
openclaw backup create --dry-run --json                  # 只预览不生成
openclaw backup create --no-include-workspace            # 不含工作区
openclaw backup create --only-config                     # 只备份配置
openclaw backup verify <归档文件>                         # 校验已有归档
```

配置损坏时仍可备份：用 `--no-include-workspace` 或 `--only-config` 跳过工作区发现。

## 8.7 资源占用参考

| 组件 | 内存 | CPU | 说明 |
|---|---|---|---|
| Gateway（Node） | 150–400 MB | 空闲很低 | 核心服务 |
| 本地 e5 embedding 服务 | ~1.2 GB | 仅查询时短暂占用 | 常驻可释放 |
| LanceDB | 几十 MB | 低 | 嵌入式，随 Gateway |
| 本地 LLM（如 Ollama） | 几 GB | 推理时高 | 用本地模型才有 |

如果内存紧张，embedding 可改为云端 API（百炼 embedding 很便宜），或不用时停止本地服务。

## 8.8 跨平台备忘

- **macOS**：LaunchAgent、Keychain、`/tmp/openclaw/` 日志、iMessage 独占。
- **Linux**：systemd --user、libsecret、journalctl、注意 `loginctl enable-linger`。
- **Windows**：任务计划/NSSM、凭据管理器、PowerShell 7、路径用反斜杠、无 iMessage、审批走网页控制台。

## 8.9 有用的命令速查

```bash
# 状态与诊断
openclaw status --deep
openclaw doctor --allow-exec
openclaw security audit --deep

# 配置
openclaw config get <key>
openclaw config set <key> <value>
openclaw config validate

# 网关
openclaw gateway restart

# 执行策略
openclaw exec-policy show
openclaw exec-policy preset cautious

# 记忆
openclaw ltm stats
openclaw ltm search "关键词"
openclaw ltm list

# 插件
openclaw plugins list
openclaw plugins install <包名>

# 模型
openclaw models list

# 日志
openclaw logs --follow
```

---

## 结语

到这里，你已经有了一个安全、有记忆、能在你电脑上干活、且数据基本本地的 AI 助手。建议按章节逐步配置，每配完一块就用 `openclaw doctor` 验证一次。遇到问题先看日志，再查本文档，最后去 [OpenClaw 文档](https://docs.openclaw.ai) 或 GitHub。祝使用愉快 🎀
