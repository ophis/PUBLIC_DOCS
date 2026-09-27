# Report: TASK-9 Auto-resume Claude Code / Codex sessions after the 5-hour usage limit resets

> 修订 2026-09-26：在首版基础上补做了四项定向核查：Codex 源码与 issue、unsnooze 及其他工具的源码、服务条款（ToS）、Claude Code 自建原语。原"缺口"中的大部分项目已经核实，并更正了若干说法。

## 结论与建议

目前没有成熟的方案能在限额重置后自动恢复"所有"中断的会话。

* **Claude Code**：v2.1.234（2026-08-17）起内置自动续跑（auto-continue），默认开启，但只覆盖**仍打开的交互式会话**。`/goal`（2.1.269）和动态工作流（2.1.271）后来也支持了；`-p` 无头运行和后台会话不支持。
* **Codex CLI**：最新稳定版 0.157.1（2026-09-26）**没有**原生等待重置功能。源码核实：触发限额后直接返回错误，不重试。至少 7 个请求此功能的 issue 仍为 open，OpenAI 均未回复。
* **第三方工具**：唯一同时覆盖 Claude Code 和 Codex、多会话、且能拉起已关闭会话的是 unsnooze。已按源码核实其机制，但项目很新：154 星、周下载约 438、9 名贡献者，没有独立的使用口碑。Codeman（770 星）支持的 agent 也包括 Codex，不过它是自托管 Web UI 形态。

**建议（macOS，多个 Claude Code + Codex 会话，常用 tmux）：**

1. **Claude 会话：用内置功能。** 保持 `/config` → Continue automatically at usage limit 开启，并确认没有项目级设置把它关掉（见下文）。
2. **Codex 会话，以及已关闭或** `-p` **的 Claude 会话：用 unsnooze，在 tmux 下运行。** 它是目前唯一现成的多会话 + 双 agent 方案。安装前要知道它会改动的地方：`~/.claude/settings.json` 里的 StopFailure hook、`~/.zshrc` 包装函数、LaunchAgent。它与内置功能同时作用于 Claude 会话时的交互没有验证过（见"缺口"），可以考虑只让它接管 Codex。
3. **权限**：恢复后的任务仍可能卡在权限提示上。给 Claude 预先配好 allowlist（无人值守时用 `--permission-mode dontAsk` + `--allowedTools`）；Codex 用 `--sandbox workspace-write`。官方文档说 `--dangerously-skip-permissions` 只应在容器/VM 中使用，不推荐。
4. **合规**：内置续跑是官方功能，没有问题。第三方的按键注入、自动拉起、定时 `claude -p` 在消费者订阅下属于灰色地带（见第 6 节），不要做 24/7 全天候运行。
5. **不建议自建**，除非 unsnooze 不合用。自建需要的原语都已核实（第 4 节），但工作量与 unsnooze 相当。

以上建议与下表是基于调研结果的综合判断。

## 对比表

| 名称 | 类型 | Claude Code | Codex | 多会话 | 恢复已关闭会话 | 机制 | 平台 | 成熟度（2026-09-26） | 注意事项 |
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| Claude Code 内置 auto-continue | 内置 | ✅ | ❌ | 每个打开的会话各自等待 | ❌ | 会话内等待，重置后发送固定续跑提示 | 全平台 | 官方，v2.1.234+ | 不支持 `-p`/后台会话/API key；周限额需手动开启；最多自动重新等待 2 次 |
| Claude Desktop 复选框 | 内置 | ✅（Code 标签页） | ❌ | — | ❌ | 重置后重试被中断的那一轮 | Desktop | 官方 | 只在会话限额卡片上提供；与 CLI 设置分开 |
| Codex CLI | 内置原语 | — | 仅手动/脚本 | 可按 ID 循环 | 可脚本化 | `codex resume <id>`、`codex exec resume <id>` | 全平台 | 官方 0.157.1 | 无等待重置；无触发于错误的 hook；`/goal` 触发限额后需手动 `/goal resume` |
| **unsnooze** | 第三方（npm，MIT） | ✅ | ✅ | ✅ 一个守护进程 + 台账 | ✅ | Claude：StopFailure hook + 读 JSONL + 抓屏；Codex：读 rollout 的 `rate_limits.resets_at` + 抓屏；发送续跑用 `tmux send-keys` 或 `--resume` | macOS/Linux/Windows；tmux/Zellij 等 | 154★，438 次/周下载，9 名贡献者，0 open / 13 closed issue | 会改 shell rc 和 Claude 设置；不阻止 Mac 睡眠；有过安全审计 issue（#17） |
| Codeman | 第三方 Web UI | ✅ | ✅（另有 OpenCode/Gemini 等） | ✅ | 未核实 | 限额计时器 → 发送续跑提示 | 自托管 | 770★，最近推送 2026-09-26 | 需要在 Codeman 中运行会话；按会话开关，默认关闭 |
| claude-auto-retry | 第三方 | ✅ | ❌ | 每个 tmux 窗格一个 | ❌ | tmux 包装器抓屏 + send-keys | macOS/Linux | 380★，最近推送 2026-08-26 | README 说 CLI 无原生功能，已过时；安装 unsnooze 会移除它的 hook |
| autoclaude | 第三方 Go TUI | ✅ | ❌ | ✅ | ❌ | 每 3 秒扫描 tmux 窗格，发送 continue | macOS/Linux | 47★，最近推送 2026-08-10 | 源码已能识别当前文案（更正首版的疑虑） |
| smart_resume | 第三方 shell 包装器 | ✅ | ❌ | ❌ 每次一个会话 | ✅ | 退出后读 JSONL 里的重置时间，再 `claude --resume` | macOS/Linux/WSL | 10★ | 包装器进程须一直存活 |
| Muminur 脚本 | 第三方 | ✅ | ❌ | 宣称支持 | ❌ | hook + 守护进程，按键注入 | 以 Linux/Windows 为主 | 5★，最近推送 2026-07-14 | 在 macOS 上只有 tmux 可靠 |
| vigil | 第三方 Go | ✅（仅 `-p`） | ❌ | ❌ 每次一个会话 | ✅ | `claude -r <id> -p` 循环续跑，最多 20 轮 | 跨平台 | 0★ | 正则匹配不到当前文案，大概率不可用 |
| claude-usage-watchdog | Claude Code 插件 | ✅ | ❌ | 每个项目一次一个 | ✅ | 未公开的 OAuth 用量接口 + 转录检测 → `--resume -p` | 只在 Linux 测过 | 0★ | 每 30 分钟轮询；macOS 凭据在钥匙串中，大概率读不到 |
| codex-auto-resume-watchdog | 第三方 Python | ❌ | ✅ | 单会话 | ✅ | 读 app-server 的 `primary.resetsAt` → `codex resume` | 主要在 Windows 上测试 | 2★，无许可证 | 实验性质 |
| codex-auto-continue（acw） | 相关 | ❌ | ✅ | ✅ | ❌ | 每轮结束后发送预设提示 | Linux | 25★ | 遇到限额会**暂停**，需手动 `acw resume` |
| CCAutoRenew | 相关：窗口续期 | ✅ | ❌ | — | ❌ | 新开一个 `claude` 来开启新的 5 小时窗口 | macOS/Linux | 289★ | 不恢复会话 |
| Claude Squad | 相关：编排 | ✅ | ✅ | ✅ | ❌ | tmux + 工作树（git worktree） | macOS/Linux | 约 8.5k★ | 没有限额逻辑（#114 被拒） |
| ccusage / Claude-Code-Usage-Monitor | 相关：监控 | ✅ | ❌ | — | ❌ | 显示窗口剩余时间和重置时间 | 全平台 | 成熟（8.7k★） | 只能看，不恢复会话 |

## 分部分发现

### 1. Claude Code 内置功能（高置信度，官方文档 + CHANGELOG + npm 发布日期）

* **版本**：2.1.234（2026-08-17）引入，CHANGELOG 原文："Claude Code now continues your session automatically when a [claude.ai](<http://claude.ai>) usage limit resets"。当前最新版为 2.1.283（2026-09-25）。2.1.269 起 `/goal` 在限额期间会暂停并等待；2.1.271 起动态工作流也会在限额后自动继续。
* **行为**：等待时状态行显示 `Usage limit reached · continuing automatically at 3:45pm · esc to cancel`。重置后发送固定的"从停下处继续"提示，**不重发**上一条消息。再次触发限额最多自动重新等待两次，之后显示 `Automatic continue stopped after repeated usage-limit hits`。电脑睡眠超过约 30 分钟后需要按 Enter。
* **等待会被取消的情况**：发送新提示、退出（之后恢复会话也不会重新等待）、`/login`、清空/回退（rewind）、`/resume`、`/teleport`、转交给 Desktop/后台/云端、重置时间被推迟到 24h 以后、续跑被 `UserPromptSubmit` hook 拦截。
* **不自动开始，但可以手动开启**（`/rate-limit-options` → "Wait here, then continue automatically"）：Remote Control 与 agent team 队友会话、24h 以后才重置的周限额、当前使用的是另一模型系列时触发的 Opus/Sonnet 限额。
* **完全不提供**：后台会话、`-p` 运行、API key / 云服务商 / 按量计费、没有保存 [claude.ai](<http://claude.ai>) 登录的 LLM 网关。
* **设置陷阱**：`autoContinueAtUsageLimit` 只从用户设置、`--settings` 和托管设置（managed settings）读取；如果在项目或本地设置文件中把它设为 false，就会关闭该功能。
* **限额文案**：`You've hit your session|weekly|Opus|Sonnet limit · resets 3:45pm`（后面可能带时区）。
* **通知**：2.1.234 起新增 Notification 类型 `quota_auto_resume_fired` / `_stale` / `_disabled`，可以接到手机推送。

### 2. Codex CLI（高置信度，源码核实，main 分支 985cf47）

* **无原生等待**：`core/src/session/turn.rs` 在限额错误时直接返回，不重试；错误分支的注释是 "let the user continue the conversation"。`/goal` 触发限额后进入 "Goal hit usage limits (/goal resume)" 状态，需要手动恢复。
* **限额文案**：`You've hit your usage limit.` + ` Try again at 3:51 PM.`（源码 `protocol/src/error.rs` 中的 `UsageLimitReachedError { resets_at, ... }`）。
* **重置时间可以从磁盘读到**：rollout 文件 `~/.codex/sessions/YYYY/MM/DD/rollout-<ts>-<uuid>.jsonl` 里总会持久化 `token_count` 事件，其中的 `rate_limits.primary/secondary.resets_at` 是 Unix 秒。**但限额错误本身不会写入 rollout**（它属于非持久化事件）。
* **hook**：更正首版，Codex **有**生命周期 hook（PreToolUse、Stop、SessionEnd 等 12 种），但没有任何一种在错误或限额时触发；`Stop` 只在正常完成时触发，`notify` 只发送 turn 完成事件。
* `codex exec resume`：不在可信目录时需要 `--skip-git-repo-check`。`--last` 按 cwd + 模型提供方 + 非归档过滤（`--all` 可以去掉 cwd 过滤），所以脚本应该传明确的会话 ID。
* **issue 状态**（均为 open，OpenAI 均未回复）：#21073 CLI 自动恢复（2026-05，15 条用户评论）、#34188 挂起并在重置时自动恢复（CLI）、#43605 goal/长任务持续自动恢复（CLI）、#31386 goal 在额度刷新后恢复、#28931 / #48392（App）、#34053 重试策略。#8310（bug）：限额后恢复会话会丢失任务意图；维护者回复认为限额边界不是原因，而是压缩（compaction）导致的。

### 3. unsnooze 深入核实（源码核实；可靠性为中低置信度）

* **Claude**：`unsnooze install` 在 `~/.claude/settings.json` 中加入 StopFailure hook（matcher 为 `overloaded|server_error|rate_limit`，写入前会先备份）。触发时从会话 JSONL 读取重置时间，读不到就抓屏。另有每个窗格一个监控进程，识别 `/rate-limit-options` 菜单并选择 "Stop and wait for limit to reset"，不会盲按 Enter。
* **Codex**：tail rollout 文件，读取 `rate_limits.resets_at` 和 `task_complete.error`（codex-cli 0.145+），同时抓取窗格上的横幅。
* **恢复方式**：仍打开的窗格用 `tmux send-keys -l` 输入文字再单独发 Enter；已关闭的 Claude 会话用 `claude --resume <id>`；Codex 用 `codex resume <id> "<msg>"`，无头时用 `codex exec --skip-git-repo-check resume`。
* **Mac 睡眠**：守护进程由 LaunchAgent（KeepAlive）常驻，每 30 秒对照绝对重置时间检查一次，睡眠醒来后会补做；**不会**阻止睡眠（没有 caffeinate）。
* **权限**：自身不加绕过权限的参数，但可以通过 `resumeExtraArgs` 让用户自己加。
* **系统改动**：`~/.claude/settings.json`、shell rc 中的包装函数块、LaunchAgent、`~/.unsnooze/`（权限 0700/0600）。网络请求只有 npm 更新检查和可选的 ntfy 推送，没有遥测。
* **已修复的问题**：#8 重置后没有恢复（1.14.4 修复）；#20 / #27 Codex 漏检；#30 Codex 文字已输入但没有提交（2026-09-26 关闭）；#17 第三方安全审计指出的命令注入、符号链接、锁竞争问题（issue 已关闭，未逐项核实修复）。
* **口碑**：只找到一条 X 帖子和一个目录收录，HN 上没有；Reddit 无法检索。

### 4. 自建方案（原语均已核实）

1. Claude：StopFailure hook（matcher `rate_limit`）。payload 里有 `session_id`、`transcript_path`、`cwd`、`error`、`last_assistant_message`，**没有**重置时间字段（更正 unsnooze README 的说法），需要从文本 "resets …" 中解析。实测转录中这条消息带有 `isApiErrorMessage: true, error: "rate_limit"`，但这是未公开的内部格式。
2. 写入台账（一个会话一行），用 launchd `StartCalendarInterval` 在最早的重置时间加几分钟后触发。
3. 逐个恢复：`cd "$cwd" && claude -p "Continue the task where you stopped" --resume "$session_id" --permission-mode dontAsk --allowedTools …`。`--resume <id>` 自 v2.1.223 起可以跨目录使用。失败时返回非零退出码。
4. Codex：没有限额 hook，只能在包装脚本中检测，或者轮询 rollout 中的 `resets_at`，然后 `codex exec --skip-git-repo-check resume <id> "continue"`。
5. 坑：限额是多个会话共享的，一起恢复会立刻再次耗尽，应该逐个恢复；不要与内置等待重复续跑；用 `-p` 恢复是追加一轮新对话，而不是重放被中断的那一轮；Mac 睡眠时 launchd 任务会延迟到唤醒后才执行。

### 5. 成熟度

没有成熟方案（中置信度）。内置功能可靠但覆盖面窄。第三方工具中只有 unsnooze 功能完整，但项目只有约 2.5 个月历史，也没有独立口碑。其余工具要么只支持 Claude，要么不恢复会话。GitHub 上 2026 年新建、超过 100 星的仓库里，没有遗漏其他自动恢复工具。

### 6. 服务条款（ToS）

* **Anthropic**：消费者条款（2025-10-08）禁止"通过自动化或非人工方式（bot、脚本等）访问服务"，**除非**使用 API key 或 Anthropic 明确允许。Claude Code 文档说 Pro/Max 的限额"以普通的个人使用为前提"，并保留不经通知执法的权利。内置 auto-continue 是官方功能，没有问题；第三方按键注入、自动拉起、定时 `claude -p` 没有明确许可，属于灰色地带。据媒体报道，2025-07 引入周限额时针对的正是 24/7 后台运行和账号共享（一手公告为图片，未能核实）。
* **OpenAI**：使用条款（2026-01-01）禁止"自动或程序化提取数据或输出"以及"规避速率限制"。Codex 文档建议程序化流程（如 CI/CD）使用 API key。等重置后再恢复不算规避，但用脚本做同样属于灰色地带。

### 7. 补充核实：vigil、claude-usage-watchdog 与预热（prewarm）技巧

* **vigil**（[alim596/vigil](<https://github.com/alim596/vigil>)，0★，最近推送 2026-07-18，Go）：只支持 Claude Code 的 `-p` 无头模式。流程是 `claude -p … --output-format stream-json` 取得会话 ID，再循环执行 `claude -r <id> -p "Continue…"`，`-max-cycles` 默认 20；每次调用只管一个会话，不支持交互式 TUI 和 Codex。**缺陷（读源码得出）**：它的限额正则只匹配字面量 `hit your limit`，匹配不到当前的 `You've hit your session limit` / `weekly limit`，会被当成非限额失败而直接退出循环。现状下大概率不可用。
* **claude-usage-watchdog**（[shou-dev19/claude-usage-watchdog](<https://github.com/shou-dev19/claude-usage-watchdog>)，Claude Code 插件，0★，最近推送 2026-07-19）：用 `SessionStart` hook 启动守护进程，调用**未公开**的 OAuth 用量接口 `api.anthropic.com/api/oauth/usage` 读取 `five_hour.resets_at` / `seven_day.resets_at`。再在转录里查找 `isApiErrorMessage` + `hit your (session|weekly) limit`（能识别当前文案），用 `claude --resume <id> -p … --permission-mode acceptEdits` 恢复。**并不"精确"**：它每 30 分钟轮询一次，重置后可能要多等最多 30 分钟；每个项目每次只恢复一个会话。只在 Linux 上测过；它从 `~/.claude/.credentials.json` 读取 token，而 macOS 把凭据存在钥匙串（Keychain），所以在普通 Mac 登录环境下大概率取不到用量数据。另外，使用未公开接口有随时失效和合规方面的风险。
* **预热技巧**（[spareloop](<https://github.com/VinayJogani14/spareloop>)，7★，与 CCAutoRenew 同理）：假设 5 小时窗口从第一条消息开始计时，所以提前发一条极简 prompt，就能把重置时间"钉"在想要的时刻，公式是 `预热时间 = 通常耗尽时间 − 5h − 余量`。**官方未证实**：Anthropic 帮助中心只说"每 5 小时重置 / 滚动窗口"，没有写窗口从第一条消息开始。ccusage 的 blocks 模型也是同样的假设，还把起点向下取整到整点，属于启发式做法。它可以作为调度层面的辅助手段，并不恢复会话；实际效果应以 `/usage` 或状态行显示的重置时间为准。
* **"Claude Code 自己醒不来，外部 watchdog 是唯一可靠解法"**：**部分错误。** 打开着的交互式会话从 v2.1.234 起可以自己等待并续跑；本次调研会话被限额中断后，正是由内置功能在重置后自动发出续跑提示的。这句话只对以下情况成立：已退出的会话、`-p` 无头运行和后台会话（官方明确不提供等待）、24h 以后才重置的周限额（不会自动等待）。`/loop` 和 cron 定时任务"只在 Claude Code 运行且空闲时触发"，限额期间触发时会怎样，文档没有说明，所以"定时唤醒也会被 block"未能核实。结论：外部 watchdog 只是上述情况下的必要补充，对打开的交互式会话并不是必需的。
* spareloop 的对比表里还列了 [claude-queue](<https://github.com/vasiliyk/claude-queue>)，没有核实。

## 对 issue 中已知说法的更正

* 说"触发限额即停止会话"：对当前的 Claude Code 已不完全成立，打开的交互式会话会默认自动续跑。对 Codex 成立。
* 首版说"周限额完全不提供等待"：错误，可以通过 `/rate-limit-options` 手动开启。
* 首版说"Codex 没有 hook"：不准确，有生命周期 hook，只是没有针对错误或限额的 hook。
* 首版说"autoclaude 可能识别不到新文案"：已被源码否定，它能识别。
* 首版说"Codeman 仅支持 Claude"：错误，它的支持列表包括 Codex。
* unsnooze README 说"StopFailure payload 带有重置时间"：官方文档里没有这个字段，实际是从转录文本中解析出来的。
* 另一 agent 提出"Claude Code 被限流时自己醒不来，外部 watchdog 是唯一可靠解法"：对打开的交互式会话不成立，详见第 7 节。
* "claude-usage-watchdog 到点精确 `--resume`"：实际是每 30 分钟轮询一次，最多会晚 30 分钟。
* "vigil 在 reset 后续跑同一 session"：机制属实，但它的正则匹配不到当前的限额文案。

## 缺口

* unsnooze 与 Claude 内置 auto-continue **同时启用**时是否会重复续跑或互相干扰（unsnooze 会在菜单里选 "Stop and wait…"，这一行在官方文档中没有出现）：没有验证。
* 所有第三方工具都没有做端到端实测；本机也没有安装 Codex。
* unsnooze 安全审计（#17）中各项问题的修复没有逐项核实。
* Anthropic 2025-07 周限额公告原文（图片）没有核实；Anthropic / OpenAI 对自动续跑类第三方工具是否执法，没有公开案例。
* Codeman 能否拉起已关闭的会话，没有核实。
