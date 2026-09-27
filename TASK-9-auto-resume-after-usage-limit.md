# Report: TASK-9 Auto-resume Claude Code / Codex sessions after the 5-hour usage limit resets

## 结论与建议

目前没有成熟的方案能在限额重置后自动恢复"所有"中断的会话。Claude Code（v2.1.234 起）已内置自动续跑（auto-continue），默认开启，但只对**仍打开的交互式会话**有效。Codex CLI 未发现原生的等待重置功能，只提供可脚本化的恢复命令（`codex resume <id>` / `codex exec resume <SESSION_ID>`）。第三方工具里，唯一同时覆盖 Claude Code 和 Codex、多会话、且能恢复已关闭会话的是 unsnooze。它很新（约 2.5 个月，154 星），证据只来自它自己的 README。

**建议（macOS，多个 Claude Code + Codex 会话，常用 tmux）：**

1. 保持 Claude Code 内置 auto-continue 开启（`/config` → Continue automatically at usage limit），用它处理所有打开的交互式会话。
2. 在 tmux 下加装 unsnooze，覆盖 Codex 会话和已关闭的 Claude 会话。
3. 预先放行工具权限（permissions allowlist / Codex `--sandbox workspace-write`），避免恢复后停在权限提示上。
4. 前几次重置后检查日志：这个领域没有成熟工具，基于屏幕抓取的方案在文案改版时会失效。

以上建议和下方对比表是基于单次调研结果的综合判断。

## 对比表

| 名称 | 类型 | Claude Code | Codex | 多会话 | 机制 | 平台 | 成熟度 | 注意事项 |
| -- | -- | -- | -- | -- | -- | -- | -- | -- |
| Claude Code 内置 auto-continue | 内置 | ✅ | ❌ | 每个打开的会话各自等待 | 在会话内等待，重置后发送固定续跑提示 | 全平台 | 官方，v2.1.234+ | 不支持 `-p`/后台会话、API key 计费，也不支持已退出的会话；超过 24h 的重置（周限额）不会自动等待；权限提示仍会阻塞 |
| Desktop App "Auto-continue when limits reset" | 内置 | ✅（Desktop 的 Code 标签页） | ❌ | — | 重置后重试被中断的那一轮 | Desktop | 官方 | 只在会话限额卡片上提供，周限额卡片没有；与 CLI 设置分开 |
| Codex CLI | 内置原语 | — | 仅手动/脚本 | 可按 ID 循环 | `codex resume <id>`、`codex exec resume <id>` | 全平台 | 官方 | 未发现原生等待重置；`--last` 查找失败时实际会开一个新会话；需要 Git 仓库 |
| unsnooze | 第三方工具（npm，MIT） | ✅ | ✅（另有 Grok/Qwen/Kimi/OpenCode 等） | ✅ 一个守护进程 + 共享台账 | Claude 用 StopFailure hook + 解析 JSONL；Codex 用抓取窗格 + 读 `~/.codex/sessions` 里的 rate_limits；打开的窗格发送续跑消息，已关闭的用 `--resume` 拉起 | macOS/Linux/Windows（tmux/Zellij） | 154 星、257 次提交、0 open issue，2026-07 创建 | 只有 README 作为证据，没有独立的可靠性数据 |
| claude-auto-retry | 第三方工具 | ✅ | ❌ | 每个 tmux 窗格一个监控 | 抓取屏幕 → 解析重置时间 → 重置后 60s 用 `tmux send-keys` 发 continue；可选 launchd 每 5 分钟补挂监控 | macOS/Linux | 380 星，最受欢迎的单工具 | 无法恢复已关闭的会话；README 说 CLI 没有原生功能，已过时 |
| autoclaude | 第三方 Go TUI | ✅ | ❌ | ✅ 可按窗格或全部开启 | 每 3 秒轮询 tmux 窗格，发送 Esc + continue + Enter | macOS/Linux（Homebrew） | 47 星，最近一次推送 2026-08-10 | 依赖旧文案，可能识别不到当前的 "You've hit your session limit" |
| auto-claude-resume-after-limit-reset（Muminur） | 第三方脚本 | ✅ | ❌ | 宣称支持 | Stop hook 写 status.json，守护进程倒计时后按 tmux → PTY → osascript 的顺序尝试输入 | macOS LaunchAgent / Linux / Windows | 5 星 | README 自相矛盾；在 macOS 上只有 tmux send-keys 可靠 |
| CCAutoRenew | 相关：窗口续期 | ✅ | ❌ | — | 窗口结束时新开一个 `claude` 发 "hi"，以提前开启 5 小时窗口 | macOS/Linux | 289 星 | **不恢复会话**（不用 --resume/--continue） |
| codex-auto-continue（acw） | 相关 | ❌ | ✅ | ✅ | 每轮结束后发送预设的后续提示 | Linux | 小项目 | 遇到限额会暂停，需手动 `acw resume` |
| Claude Squad | 相关：多会话编排 | ✅ | ✅ | ✅ | tmux + git 工作树（git worktree） | macOS/Linux | 约 8.5k 星 | 没有限额恢复逻辑（#114 被拒）；`--autoyes` 只能自动通过提示 |
| ccusage | 相关：用量监控 | ✅ | ❌ | — | `blocks --active` 显示当前 5 小时窗口的剩余时间 | 全平台 | 成熟 | 只能预测重置时间，不恢复会话 |

## 分部分发现

### 1. 内置功能

* **Claude Code 内置 auto-continue**（高置信度，3 个官方来源，3-0）：v2.1.234 起，用 [claude.ai](<http://claude.ai>) 订阅登录的交互式会话默认开启。触发限额后在会话内等待，状态行显示 `Usage limit reached · continuing automatically at 3:45pm · esc to cancel`。重置后发送固定的"从停下处继续"提示，**不会重发上一条消息**。再次触发限额最多自动重新等待 2 次，之后停止。设置项为 `autoContinueAtUsageLimit`。来源：[interactive-mode](<https://code.claude.com/docs/en/interactive-mode#wait-for-a-usage-limit-to-reset>)、[errors](<https://code.claude.com/docs/en/errors#youve-hit-your-session-limit>)、CHANGELOG 2.1.234。
  * 实测旁证：本次调研会话本身就触发了限额，CLI 显示 "Claude Code will continue automatically at 3:10am. Keep this session open; it may still pause for permission prompts."
* **内置功能的限制**（高置信度，官方文档）：以下情况完全不提供等待：后台会话、`-p` 无头运行、API key / 云服务商 / 按量计费、没有保存 [claude.ai](<http://claude.ai>) 登录的 LLM 网关。退出、`/login`、`/resume`、回退（rewind）会结束等待，之后恢复会话也不会重新开始等待。电脑睡眠超过约 30 分钟后需要按 Enter。权限提示仍会阻塞。超过 24h 的重置不会自动等待。
* **限额文案**（高置信度）：当前格式为 `You've hit your session|weekly|Opus|Sonnet limit · resets 3:45pm`，第三方工具还会看到时区后缀。旧文案 "Claude usage limit reached" 已被取代。Opus/Sonnet 限额只限制该模型系列，可以用 `/model` 切换后继续工作。
* **Desktop App** 有一个独立的 "Auto-continue when limits reset" 复选框，只在会话限额卡片上提供（高置信度）。
* **Codex CLI**（高置信度，官方文档）：`codex exec resume --last <prompt>` 和 `codex exec resume <SESSION_ID>` 可以脚本化，会话保存在 `~/.codex/sessions`（`--ephemeral` 会话无法恢复）。无人值守时默认是只读沙箱，需要 `--sandbox workspace-write`。官方文档没有提到任何等待重置或重试行为。已知坑：`--last` 按 cwd、来源和模型提供方过滤，查找失败时实际等于新会话（#17302），所以脚本应传明确的会话 ID。

### 2. 第三方自动恢复工具

* **unsnooze**（中置信度，只有 README）：见对比表。它是唯一覆盖"Codex + 已关闭会话 + 多会话"的方案。
* **claude-auto-retry**（中置信度，README 和代码一致）、**autoclaude**（中置信度）、**Muminur 脚本**（低置信度）：都只支持 Claude Code，都依赖 tmux 屏幕抓取 + 按键注入。在内置 auto-continue 出现后，它们的主要价值只剩给 CLI 旧版本兜底。

### 3. 相关工具

CCAutoRenew、codex-auto-continue、Claude Squad、ccusage 都**不会**在限额重置时恢复被中断的会话（高置信度，README + 源码核实）。CCAutoRenew 可以提前开启窗口，Claude Squad 的 `--autoyes` 可以清掉权限提示，两者都可以作为辅助。

### 4. 自建方案（DIY）

没有经过验证的具体结论（见"缺口"）。可行的思路是：用 StopFailure hook 记录会话 ID 和重置时间，由 launchd 在重置后对每个会话执行 `claude --resume <id>` / `codex exec resume <id>`。这正是 unsnooze 已经实现的内容，自建的收益不大。

### 5. 成熟度

没有成熟方案（中置信度）。内置功能最可靠，但覆盖面窄。第三方工具都是新的单人维护项目，大多不到 400 星，且依赖脆弱的屏幕抓取。Muminur 在 v1.24.0（2026-07-14）就因为文案改版漏检过一次真实事件。Codex 没有 hook 或事件，所有 Codex 方案都只能抓屏或读会话文件。

## 对 issue 中已知说法的更正

* issue 说两者"触发 5 小时限额即停止会话"：对当前的 Claude Code 已不完全成立。v2.1.234 起，打开的交互式会话默认会自动等待并续跑。
* 周限额：内置功能不会**自动**等待超过 24h 的重置，但可以通过 `/rate-limit-options` 选 "Wait here, then continue automatically" 手动开启等待（首轮验证据此否决了"周限额完全不提供等待"的说法）。

## 缺口

* **Codex 原生支持现状**（未验证，单一来源）：首轮抽取到多个 openai/codex 的 open issue 在请求此功能（#21073、#34188、#48392 等），维护者均未回复。Codex 内部已有 `UsageLimitReachedError.resets_at` 但没有使用。#8310 报告手动恢复后可能丢失任务意图、重做已完成的步骤。这些都未进入交叉验证。
* **unsnooze 的实际可靠性**：多会话并发恢复、Mac 睡眠后的行为都没有独立证据，也未做端到端测试。它的 Claude StopFailure hook 会携带重置时间，这一点也只来自其 README。
* **服务条款（ToS）**：Anthropic / OpenAI 消费者订阅是否允许自动输入 continue 或自动拉起会话，没有验证到任何结论。
* **自建 hook + launchd 方案**的细节（如何去重、如何识别哪些会话中断在任务中途）没有验证。
* 未验证的其他工具：Codeman（tmux 持久会话，按会话开关自动恢复，仅支持 Claude）、Smart Resume（shell 包装器，重置后执行 `claude --resume <uuid>`，一次只管一个会话）、codex-auto-resume-watchdog（通过 app-server 读取 resetsAt，主要在 Windows 上测试）。
* 本次调研运行曾被本账号的使用限额中断，恢复后补齐了验证和综合步骤。
