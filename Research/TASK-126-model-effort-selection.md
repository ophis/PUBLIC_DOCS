# Report: TASK-126 如何选择 AI 模型和 effort 等级：效果不降、用量更省

## 结论与建议

在质量不降的前提下，最大的省用量手段是降推理强度（effort），而不是先换模型；在多 agent 的 run 里，第二个手段是给子代理（subagent）和工作流（Workflow）里的 agent 单独指定模型和 effort。依据是：Opus 5.5 在应用程序接口（API, Application Programming Interface）和 Claude Code 里的默认 effort 都是 medium。Anthropic 自测，Opus 5.5 用 medium 就达到或超过 Opus 5 用 high 的编程和知识工作成绩，且步数和 token 更少。同一档位名下，Opus 5.5 比 Opus 5 想得更多，所以用户现在的 high、xhigh 比官方起点高一到两档。官方页面还给出两组数（未经投票核实）：SWE-bench Pro 上，xhigh 比 high 只多约 1.4 分，花费是 2.5 倍；调研和知识工作的曲线几乎是平的，medium 与默认档精度持平，花费约 70%–87%。effort 也不能一路往下降：low 档下 Opus 5.5 和 Sonnet 5.5 都会跳过真正的验证，调试类任务掉得最多（一个调试任务 low 0/5，xhigh 4/5）。agent-pm 现状经核实与 issue 所述一致：deep-research 用 `opus` + `ultracode`（即 Opus 5.5、xhigh，加自动工作流编排），engineering 用 `opus` + `xhigh`，light-research 和 product-design 用 `opus` + `high`，只有用量探测用 Haiku。仓库里没有任何地方给子代理或工作流 agent 指定模型或 effort，本 run 的 159 个工作流 agent 全部跑在 claude-opus-5-5 上。本 run 实测，按 API 单价折算，工作流 agent 占约九成花费，其中约四分之三是每个 agent 写入缓存的上下文，思考和输出只占 14%–17%。由此推断，对 deep-research 来说，减少 agent 数、把 agent 换到更便宜的模型，比降 effort 省得多。订阅额度怎么折算 token，官方没有公布。建议：用户交互会话默认 Opus 5.5 medium，修 bug、审查、验证类工作切到 high，xhigh 和 max 只用在实测有收益的难题上。engineering 主会话从 xhigh 降到 high，实现者子代理用 medium 并加验证指令，审查者用 high。light-research 和 product-design 从 high 降到 medium，产品需求文档（PRD, Product Requirements Document）的终稿审查保留 high。deep-research 先保持 ultracode，但把工作流 agent 改用 Sonnet 5.5，自己写的 Ultra Code 脚本里检索和核实 agent 用 medium。机械性工作用 Haiku 4.5 或 Sonnet 5.5 low，写进子代理定义。每条建议都应先用下文「七」的小规模配对实验验证再改配置；改 `tasks/*.toml` 时要同时改钉住这些值的测试。以上建议、两张表和由实测数据得出的推断，都是本报告对发现的综合。

## 对比表

本节两张表都是本报告对下文发现的综合。「日期」是所依据信息的发布日期或抓取日期（2026-10-01 抓取的是当天的现行版本）。

### 任务类型 × 推荐的模型和 effort

| 任务类型 | 推荐 | 依据（详见「发现」） | 置信度 | 日期 |
|---|---|---|---|---|
| 网上调研与综合 | Opus 5.5 medium；多 agent 深度调研的检索、抓取、核实 agent 先试 Sonnet 5.5 | 知识工作曲线几乎平（Fable 5/5.1 测得）；Opus 5.5 medium ≥ Opus 5 high（知识工作）；本 run 实测花费大头在 agent 数和上下文 | 中 | 2026-09-22 至 10-01 |
| 写 PRD、产品设计 | Opus 5.5 medium 起草；终稿审查 high | 只有「知识工作评测」的间接证据，没有 PRD 专门的评测 | 低–中 | 同上 |
| 编程：规划和设计 | Opus 5.5 high；范围清楚的小设计用 medium | 官方分档：high 用于验证重要、边界情况多的工作 | 中 | 2026-09-25、10-01 |
| 编程：实现（规格清楚） | Opus 5.5 medium，并加验证指令；有测试时可用 low 跑、失败的再用 high 重跑 | 官方：medium 用于范围清楚的日常开发；SWE-bench Pro 上 medium 比 high 低约 2.5 分，花费约 70%；low + 失败重跑的通过率不低于全 high，花费略高于一半 | 中 | 同上 |
| 编程：调试、修老代码的 bug | Opus 5.5 high；难的长程 bug 用 xhigh | low 跳过复现和验证（0/5 → xhigh 4/5）；官方把在老代码（brownfield）里修 bug 归 high | 中–高 | 2026-09-25 |
| 代码审查 | Opus 5.5 high；被审的实现者不要开 xhigh 或 max | 官方：high 用于验证重要的工作；xhigh、max 下模型会自己发起审查轮，和专门的审查者重复 | 中 | 2026-09 |
| 难的无人值守问题（安全审计、端到端构建） | xhigh 或 max，只在 A/B 测出收益后用 | Terminal-Bench 3.0：难任务随 effort 一直提升，但每次尝试的 token 约为 low 的 3 倍 | 中 | 2026-09-25 |
| 机械性工作（检索、格式整理、Linear 状态更新、查状态） | Haiku 4.5（没有 effort 参数）或 Sonnet 5.5 low，写进子代理定义或调用 | 官方成本页：简单子代理任务用 `model: haiku`；effort 文档：low 适合子代理；其余只有个人经验 | 低 | 2026-02 至 10-01 |
| 用户交互会话 | Opus 5.5 medium 为默认，按任务切到 high；xhigh、max 只用于实测有收益的难题 | 同上各行 | 中 | 2026-09-22 至 10-01 |

### agent-pm 各任务

| 对象 | 现状（已核实） | 建议 | 置信度 |
|---|---|---|---|
| deep-research | `opus` + `ultracode`：主会话 Opus 5.5 xhigh，打开自动编排；`/deep-research` 约 100 个 agent，Ultra Code 脚本最多 100 个，全部继承 Opus 5.5 和会话 effort | 保留 ultracode；工作流 agent 改用 Sonnet 5.5（为 run 设 `CLAUDE_CODE_SUBAGENT_MODEL`），自写 Ultra Code 脚本里检索和核实 agent 设 `effort: 'medium'`；能否在 high 或 medium 下打开 ultracode 要先确认（见缺口） | 低–中，先做实验 E3 |
| engineering | `opus` + `xhigh`；`autopilot:build` 派出的实现者和审查者在仓库里没有指定模型 | 主会话 high；实现者 medium 并加验证指令，审查者 high；不要导出 `CLAUDE_CODE_EFFORT_LEVEL` | 中，实验 E2 |
| light-research | `opus` + `high`，一轮 3–6 个检索 agent | 主会话 medium；检索 agent 试 Sonnet 5.5 medium | 中，实验 E1 |
| product-design | `opus` + `high`，最多 2 个子代理（追问、审查） | 起草 medium，审查子代理 high | 低–中，实验 E4 |
| 用量探测 | 已用 Haiku | 不变 | 高 |
| run 内部的子代理（通用） | 都继承主会话 | 每类 agent 显式指定：检索和机械性工作用 Haiku 4.5 或 Sonnet 5.5 low；核实和审查用 Opus 5.5 high 或 medium；实现用 medium | 中 |
| 用户交互会话 | Opus 5.5 high / xhigh | 默认 medium；修 bug、审查、验证类工作切 high；换档放在任务交界处 | 中 |

## 发现

仓库 `ophis/agent-pm`，提交 `c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca`。下文代码行的链接都指向这个提交。

### 方法与来源

- **两轮调研。** 本 issue 是 mixed 类型。第一轮是 `/deep-research`，负责网上部分，共 100 个 agent：1 个拆题、5 个检索、18 个抓取、75 个核实票、1 个综合。90 条说法里，按重要性挑出 25 条，每条由 3 个独立 agent 投票，全部 3:0 成立。第二轮是 Ultra Code，负责本地部分，共 59 个 agent（上限 60）：5 个读仓库的 agent 提出 60 条说法，按重要性挑出 18 条，每条 3 票。结果 17 条成立，1 条被推翻并改正（见「五」），其余 42 条只有单个 agent 的读取结果。第二轮之前做了用量刹车检查：`five_hour=0.11`，低于 0.8，没有触发。
- **网上来源。** 共抓取 18 个来源，除 3 篇个人博客外，全部是 Anthropic 的文档、博客和工程文章。Anthropic 的文档页大多不显示日期，下文写「无日期，2026-10-01 抓取」，指当天的现行版本。Opus 5.5 于 2026-09-22 发布，Sonnet 5.5 于 2026-09-28 发布，默认值和行为可能还会调整。
- **置信度。**
  - 高：官方文档原文，且经 3 票核实。
  - 中：经核实，但只有厂商内部测试或代理模型的数据；或者是官方页面原文，由一个抓取 agent 带原文摘录、没有经过投票。后一种标「未投票核实」。
  - 低：个人博客、单个实例，或本报告的推断。
- **综合 agent 的一处说法不准。** `/deep-research` 的综合 agent 说价格、15 倍 token、顾问模式和失败重跑这几组数「没有来源证实」。实际上它抓取的官方页面都有原文，只是这几条不在被投票的 25 条之内。本报告把它们标为「未投票核实」，不当作缺失。
- **本 run 实测。** 本 run 自己的会话记录、工作流日志和用量探测，作为本地观测数据使用，见「三」。

### 一、effort 到底改变什么

1. **effort 是行为信号，不是硬性的 token 上限。** 它作用于全部输出 token，包括文本、工具调用及其参数，以及思考。
   - 低 effort：工具调用更少，会合并，也更简短；思考更少，简单问题直接不思考。
   - 高 effort：工具调用更多，会先说明计划，总结更详细，代码注释更多。
   - 各档：max 不限思考长度，xhigh 用于长时间探索，low 尽量少思考。
   - 没有哪一档保证每次请求都思考：只处理工具结果的后续请求，即使在 xhigh、max 也可能不思考。
   - 在部分模型上，降 effort 不一定缩短可见回复，回复长度要在提示里要求。

   置信度：高。来源：[effort 文档](https://platform.claude.com/docs/en/build-with-claude/effort)、[thinking-steering-and-cost 文档](https://platform.claude.com/docs/en/build-with-claude/thinking-steering-and-cost)（均无日期，2026-10-01 抓取）。

2. **各模型支持的档位和默认值。**

   | 模型 | 支持的档位 | API 默认 | Claude Code 默认 | 说明 |
   |---|---|---|---|---|
   | Opus 5.5 | low、medium、high、xhigh、max | medium（Opus 5 及更早的 Opus 是 high） | medium | 自适应思考（adaptive thinking）始终开着，关掉思考的请求返回 400，所以 effort 是主要的成本控制 |
   | Sonnet 5.5 | 同上，各档已重新校准，同名档位的思考量与 Sonnet 5 不同 | high | medium | 官方：规格清楚的智能体编程和多步工具调用从 medium 起，更难更长的用 high |
   | Haiku 4.5 | 没有 effort 参数，只能用 `budget_tokens` 的扩展思考 | — | — | 不支持 xhigh，所以不能开 ultracode |
   | Opus 4.7 | 五档 | — | xhigh | 「xhigh 是 Claude Code 默认档」只对 Opus 4.7 成立 |

   一个坑：用户设置里的顶层 `effortLevel` 对 Opus 5.5 不生效，Opus 5.5 会从 medium 开始。要用 `/effort`、`/model` 选单、`--effort`、环境变量、按模型的 `modelSettings`，或项目、本地、托管设置里的 `effortLevel` 来设。

   置信度：高。来源：[effort 文档](https://platform.claude.com/docs/en/build-with-claude/effort)（无日期，2026-10-01 抓取）、[Sonnet 5.5 提示指南](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-sonnet-5-5)（无日期，引用 2026-08-18 的 beta 头）、[Claude Code 模型配置](https://code.claude.com/docs/en/model-config)（2026-10-01 修改）、[Sonnet 5.5 发布页](https://www.anthropic.com/claude-sonnet-5-5)（2026-09-28）。

3. **同名档位，Opus 5.5 想得比 Opus 5 多，xhigh、max 尤其明显。** 沿用 Opus 5 的设置，回合会更长、输出 token 更多。官方建议从 medium 起步，在自己的评测上逐档对比（effort sweep），xhigh 和 max 只留给测出收益的工作。置信度：高。来源：[Opus 5.5 提示指南](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5)（无日期，约 2026-08 下旬后）、[Claude Code 模型配置](https://code.claude.com/docs/en/model-config)（2026-10-01 修改）。

4. **Ultra Code（ultracode）不是 effort 档位，是 Claude Code 的一个设置。**
   - 打开后，Claude Code 对每个实质性任务自动编排工作流，按会话当前的 effort 运行。一个请求可能拆成理解、修改、验证几个工作流。
   - 用 `claude --effort ultracode` 启动时，会同时打开 ultracode，并把 effort 设成 xhigh。
   - 打开后，每个请求用的 token 更多、耗时更长，订阅会更快碰到 5 小时或每周上限。
   - 它还会关掉「Large workflow」警告和 Agent 工具子代理的并发上限。工作流运行时默认最多 16 个 agent 并发，每个 run 最多 1,000 个 agent。

   置信度：高。来源：[Claude Code 工作流文档](https://code.claude.com/docs/en/workflows)（2026-09-29 修改）、[Claude Code 模型配置](https://code.claude.com/docs/en/model-config)（2026-10-01 修改）。

5. **少想的办法，以及换档的缓存代价。**
   - 要少想，先降 effort：降 effort 减少思考、成本和延迟，比提示词里写「少想」可靠。
   - 在 API 上，两次请求之间改顶层 effort，会让提示缓存（prompt caching）失效；按消息调整 effort（per-message effort，beta）不会。
   - 思考 token 按输出计费，可以从 `usage.output_tokens_details.thinking_tokens` 读出来。

   置信度：中（官方原文，部分经投票核实）。来源：[Opus 5.5 提示指南](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5)、[thinking-steering-and-cost 文档](https://platform.claude.com/docs/en/build-with-claude/thinking-steering-and-cost)（均无日期，2026-10-01 抓取）。

### 二、质量和档位的关系

**总体证据。**

1. **Opus 5.5 medium 对 Opus 5 high（厂商自测）。**
   - 编程和知识工作评测上，Opus 5.5 medium 达到或超过 Opus 5 high。在真实仓库的智能体编程上，步数和 token 都更少。在几项编程评测上，low 也接近。
   - 核实 agent 从发布页记下的数：Terminal-Bench 4.0 上，Opus 5.5 默认档胜过 Opus 5 max，花费约五分之一；CursorBench 4.0 上，Opus 5.5 medium 52.5%，Opus 5 max 46.6%。
   - 二手来源 Orcarouter（2026-09-22）：FrontierCode 上 max 54.4%、medium 54.6%，几乎不随 effort 变；CursorBench 上两者差约 5 分（57.8 对 52.5）。可见差距大小取决于评测。
   - 没有独立复现。

   置信度：厂商结论为高，具体分数为中。来源：[Opus 5.5 发布页](https://www.anthropic.com/claude-opus-5-5)（2026-09-22）、[Claude Code 模型配置](https://code.claude.com/docs/en/model-config)（2026-10-01 修改）、[Opus 5.5 提示指南](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5)。

2. **SWE-bench Pro 上 Opus 5.5 的 effort 曲线（以 high 为基准）。**
   - medium（API 默认）：低约 2.5 分，花费约 70%。
   - low：低约 8 分，花费约三分之一。
   - xhigh：高约 1.4 分，花费是 high 的 2.5 倍。
   - 另外，Opus 5.5 medium 与 Fable 5.1 默认档成绩持平（92.8% 对 92.3%），每个解出的任务花费约五分之一（0.22 美元对 1.19 美元）。

   置信度：中（官方原文，未投票核实）。来源：[Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)（无日期，2026-10-01 抓取）。

3. **调研和知识工作：曲线几乎是平的。**
   - 在 WideSearch、DeepWideSearch、BrowseComp、GDPval 四项上：low 少 1–3 分，每个任务省三分之一到一半；medium 与默认档精度持平，花费约 70%–87%；默认档比 medium 没有可测的提升。
   - 这组测量用的是 Fable 5，不是 Opus 5.5。
   - DeepResearch Bench II（Fable 5.1）上，提高 effort 让每个任务的花费从 4.66 美元升到 7.12 美元，分数没有提高。

   置信度：中（官方原文，未投票核实；代理模型）。来源：同上页。

4. **长程编程：难任务随 effort 一直提升，代价陡峭。**
   - Terminal-Bench 3.0 上的 Fable 5.1，各 370 次尝试：每次尝试的 token 中位数从 low 的 73k 涨到 max 的 222k，约 3 倍。
   - 通过数从 140/370（约 38%）升到 214/370（约 58%）。由此算出，每解出一题，max 用的 token 约为 low 的 2 倍。
   - 各领域从 low 到最高三档（合并统计）都在提升：安全 64%→87%，硬件 34%→75%，机器学习 54%→73%，科学 41%→61%，软件 43%→56%，媒体 18%→30%，运维 12%→22%。
   - 局限：代理模型；各类题量很小（媒体 4 题、硬件 5 题、安全 7 题）；关闭了安全干预；统计的是 token 中位数，不是计费用量。

   置信度：中。来源：[Spending your effort](https://claude.dev/blog/spending-your-effort/)（Anthropic Claude Code 团队 Thariq Shihipar，2026-09-25）。

5. **low 会跳过验证，这是无人值守编程不该用 low 的主要理由。**
   - Sonnet 5.5 在 low 下，有时没跑任何能检验改动的检查就报告完成，例如依赖没装、就跳过了测试。在系统提示里加 Anthropic 给的验证段落后，这种情况变得很少，质量没有可测变化，每个任务的花费略增（只在 low 下测过）。
   - 在长的智能体任务上，Sonnet 5.5 在 low 和 medium 下更容易中途停下来请示，这对无人值守的 run 不利。
   - Opus 5.5 的调试任务 mvcc-lsm-compaction：low 0/5，每次约 1 分钟，先改代码、没复现 bug，也没检查新测试能否抓住它；xhigh 4/5，每次约 11 分钟，先复现，写了随机化测试，还检查了测试在半成品修复上会失败。
   - HTML 过滤任务：low 1/5，每次约 2 分钟；xhigh 5/5；一次 high 的 run 约 33 分钟，做了对抗式自查、读了解析器源码、跑了跨站脚本（XSS, Cross-Site Scripting）测试集、写了模糊测试。
   - 同一篇文章里，规格清楚的实现任务在各档成绩差不多。
   - Sonnet 的结论来自未公开样本量的内部测试；Opus 的例子是单个任务各 5 次，很可能是特意挑来展示差距的。

   置信度：高（现象），具体幅度为中。来源：[Sonnet 5.5 提示指南](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-sonnet-5-5)（无日期）、[Spending your effort](https://claude.dev/blog/spending-your-effort/)（2026-09-25）。

6. **xhigh 和 max 的额外开销：模型会自己发起审查。**
   - Sonnet 5.5 在 xhigh、max 下做完后，会自己再做审查和加固，有时还会派审查子代理。
   - Anthropic 的编程测试里，在 max 下往系统提示加一句「Don't start extra rounds of review or hardening on your own, and don't launch reviewer sub-agents unless the user asked for a review」，会话花费降了约三分之一，质量不变。
   - 二手报道：Sonnet 5.5 在 FrontierCode 上 max 46.2%，低于 xhigh 的 52.1%，归因于 max 会派审查子代理。
   - 这一点只在 Sonnet 5.5 的文档里写明，套到 Opus 5.5 上是外推。对 engineering 的含义：autopilot 已经有专门的审查者，实现者开 xhigh 可能和它重复。

   置信度：中。来源：[Sonnet 5.5 提示指南](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-sonnet-5-5)（无日期）；buda.im 二手文章（2026-09-29，核实 agent 未记下 URL）。

7. **max 收益递减、容易想太多。**
   - Claude Code 模型配置页：max「may show diminishing returns and is prone to overthinking」，要先测再推广。
   - effort 文档里关于 Opus 4.7 的说明：在多数工作上，max 花很多成本换不大的质量提升；在结构化输出和不太吃智力的任务上会想太多。

   置信度：高。来源：[Claude Code 模型配置](https://code.claude.com/docs/en/model-config)（2026-10-01 修改）、[effort 文档](https://platform.claude.com/docs/en/build-with-claude/effort)。

8. **官方按档位给的建议（Claude Code 文档，与 Claude Code 团队成员的文章几乎逐字一致）。**
   - low：你会自己检查的、在环的快速工作，如头脑风暴、草图、改名、简单改动。
   - medium：范围清楚的日常开发，如实现新功能。
   - high：验证重要、边界情况多的工作，如在老代码里修 bug。
   - xhigh：更深的推理，token 花费更高。两个来源都没给具体任务建议。
   - max：要模型独立解决的难题，如找安全漏洞、端到端构建并验证一个应用。
   - 高档位下，Opus 5.5 测了更多边界情况，验证了更多自己的工作，也更多地自己做决定；低档位下更快交出一个起点。
   - 作者本人的做法：先让模型采访自己，用 low 实现，自己审，再用 high 做验证。这是个人做法，不是产品指南。

   置信度：高。来源：[Claude Code 模型配置](https://code.claude.com/docs/en/model-config)（2026-10-01 修改）、[Spending your effort](https://claude.dev/blog/spending-your-effort/)（2026-09-25）。

9. **两份官方页面对默认模型的侧重不同。**
   - Claude Code 成本页：Sonnet 能做好大多数编程，比 Opus 便宜，Opus 留给复杂的架构决策和多步推理；简单子代理任务指定 `model: haiku`。
   - 成本优化指南：大多数 agent 工作从 Opus 5.5 默认档（medium）起步，难的推理和长程智能体工作再用 Fable 5.1。
   - 两者都没有给出 Opus 5.5 与 Sonnet 5.5 在订阅用量下的质量对比。

   置信度：中（未投票核实）。来源：[Claude Code 成本页](https://code.claude.com/docs/en/costs)（无日期，提到 v2.1.271）、[Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)（无日期，2026-10-01 抓取）。

**按任务类型。** 下面每条的结论都是本报告的综合。

- **网上调研与综合。** 知识工作曲线平（第 3 条），Opus 5.5 medium 已经不低于 Opus 5 high（第 1 条），所以 medium 足够。
  - 多 agent 的另一面（2025 年数据，Claude 4 时代模型）：单个 agent 用的 token 约为聊天的 4 倍，多 agent 系统约 15 倍。在 BrowseComp 上，三个因素解释了 95% 的成绩差异，其中 token 用量单独解释 80%，另两个是工具调用次数和模型选择。编排者-工作者（orchestrator-worker）结构（Opus 4 主导、Sonnet 4 子代理）在内部调研评测上比单个 Opus 4 高 90.2%；升级模型的收益大于把 token 预算翻倍。这种结构只适合可并行、价值高的工作，如调研；多数编程任务不适合。置信度：中（未投票核实；年代较早）。来源：[How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)（2025-06-13）。
  - 推论：在调研里，多花 token 确实买到质量，但应该靠更多并行的、便宜的工作者去花，而不是靠每个 agent 都开高 effort 的 Opus。这是推断，置信度低。
- **写 PRD、产品设计。** 没有找到专门的评测。只有「知识工作」的间接证据：第 1、3 条；另有 Claude Code 内置 claude-api skill 的参考文档说，Opus 5.5 medium 写长篇分析交付物比 Opus 5 high 好，输出 token 少约 40%（单一来源，随 Claude Code v2.1.287 附带，缓存日期 2026-09-25）。置信度：低–中。
- **编程。**
  - 规划和设计：high，范围清楚时 medium（第 8 条）。
  - 实现：medium，加验证指令（第 2、5、8 条）。
  - 调试：high，难的长程 bug 用 xhigh（第 4、5 条）。
  - 代码审查：high，被审的实现者避免 xhigh、max（第 6、8 条）。
  - 置信度：中。
- **机械性工作。** 没有基准测试（benchmark）。证据如下：
  - effort 文档说 low 适合「such as subagents」这类简单、看重速度和成本的任务；Claude Code 成本页说简单子代理任务用 `model: haiku`。
  - 个人报告一：把分诊和路由子代理从 medium 降到 low，两周里没有可测的质量下降，但没给数据（[luonghongthuan.com](https://luonghongthuan.com/en/blog/claude-code-effort-dial-subagent-cost-control/)，2026-08-14）。
  - 个人报告二：Max 订阅用户把约 95% 的执行类工作交给 Haiku 后，周五时每周额度用量从 70%–80% 降到约 40%；作者说 Opus 在简单执行任务上会过度设计，Haiku 不适合创作、架构、复杂调试（[thoughts.jock.pl](https://thoughts.jock.pl/p/claude-model-optimization-opus-haiku-ai-agent-costs-2026)，2026-02-13，Opus 4.5/4.6 时代）。
  - 置信度：低。

### 三、用量怎么算

1. **订阅额度：官方说法。**
   - 5 小时和每周额度是跨模型共享的，用 `/model` 换模型不能恢复。只有「Opus 上限」「Sonnet 上限」这类按模型族的上限，可以换到别的族继续用。
   - 子代理的请求和主对话计入同一份额度。
   - 置信度：中（未投票核实）。来源：[Claude Code 成本页](https://code.claude.com/docs/en/costs)（无日期，提到 v2.1.271）、[Claude Code 子代理文档](https://code.claude.com/docs/en/sub-agents)（无日期，提到 v2.1.257）。
2. **没有官方的 token 到订阅额度的换算。**
   - 没找到任何官方来源说明 token 怎么折算成 5 小时和每周额度，也没有 Opus、Sonnet、Haiku 之间的折算比例。
   - 一篇博客明确说这个换算没有公开（[openclawdc.com](https://openclawdc.com/blog/do-claude-code-subagents-burn-your-limit/)，2026-08-19）。
   - 置信度：中（是「找不到」的结论）。
3. **API 单价（不是订阅换算，只能当相对参照）。**
   - 每百万 token 的输入/输出价格：Opus 5.5 4/20 美元，Sonnet 5.5 2/10 美元，Haiku 4.5 1/5 美元，比例为 4:2:1。
   - 缓存读取：Opus 5.5 为 0.20 美元，按例外的 0.05 倍计价，与 Sonnet 5.5 相同；Haiku 4.5 为 0.10 美元。
   - 缓存写入：5 分钟有效期按输入价的 1.25 倍，1 小时按 2 倍。
   - Claude 4.7 及以后的分词器（tokenizer），同样的文本约多出 30% 的 token。
   - 置信度：中（官方原文，未投票核实；与 Claude Code 内置 claude-api skill 的价目表一致，缓存日期 2026-09-25）。来源：[Pricing](https://platform.claude.com/docs/en/about-claude/pricing)（无日期，晚于 2026-09-01）。
4. **用量为什么会成倍放大（官方）。**
   - 每个子代理、每个工作流 agent 都发自己的请求。队员（teammate）在计划模式下运行时，agent 团队（agent team）用的 token 约为普通会话的 7 倍。
   - 工作流 agent 只有在模型、effort、agent 类型、工具、输出结构和工作目录都相同时，才共享提示缓存；即使在订阅下，缓存默认也只保留 5 分钟（可用 `subagentPromptCacheTtl` 改成 1 小时）。所以各阶段混用不同模型或 effort，会减少缓存共享。
   - 一个工作流超过 25 个 agent，或预计超过 150 万 token，会显示「Large workflow」警告。默认规模建议是 medium（少于 10 个 agent），Pro 计划是 small（少于 5 个）。
   - 置信度：中（未投票核实）。来源：[Claude Code 成本页](https://code.claude.com/docs/en/costs)、[Claude Code 工作流文档](https://code.claude.com/docs/en/workflows)（2026-09-29 修改）。
   - 二手说法：每个非分叉（fork）子代理都要为自己的启动上下文付费，包括系统提示、整套 CLAUDE.md、git 状态快照和 skill 内容；默认最多 20 个子代理并发；fork 复用父会话的缓存。置信度：低。来源：[openclawdc.com](https://openclawdc.com/blog/do-claude-code-subagents-burn-your-limit/)（2026-08-19）。
5. **本 run 实测（2026-10-01，本地观测）。** 数据来自会话记录，每条助手消息带 `model` 和 `usage`：

   | 部分 | agent 数 | 模型 | 写入缓存 | 读取缓存 | 输出（其中思考） | 按 Opus 5.5 API 单价折算 |
   |---|---|---|---|---|---|---|
   | `/deep-research` 轮 | 100 | 全部 claude-opus-5-5 | 3.19M | 11.57M | 0.144M（0.026M） | 约 21.2 美元：写入 75%、读取 11%、输出 14% |
   | Ultra Code 轮 | 59 | 全部 claude-opus-5-5 | 1.52M | 5.86M | 0.088M（0.036M） | 约 10.5 美元：写入 72%、读取 11%、输出 17% |
   | 主会话（截至写报告前） | 1 | claude-opus-5-5 | 0.23M（1 小时缓存） | 5.39M | 0.051M（0.032M） | 约 3.9 美元 |

   其他观测：
   - `/deep-research` 轮的 harness 统计是 3,743,039 个子代理 token、595 次工具调用、11.2 分钟；Ultra Code 轮是 1,849,681 个、290 次、3.4 分钟。
   - 用量探测：第一轮之后 `five_hour=0.11`、`seven_day=0.54`；第二轮之后 `five_hour=0.15`、`seven_day=0.55`。
   - `/deep-research` 的 100 个 agent 里，75 个是核实票。这个内置工作流的脚本里，按 `model|effort` 搜索没有匹配，即不给任何阶段指定模型或 effort。
   - 局限：会话记录里的流式用量记录可能少计输出；记录里看不到每个 agent 实际用的 effort；订阅额度是否按 API 单价的比例计算缓存读写，未知；5 小时窗口里还可能有本 run 之外的用量。

   推断（置信度低）：
   - 这类多 agent 的 run，花费大头是 agent 的数量乘以每个 agent 的上下文；思考和输出只占 14%–17%。
   - 所以对 deep-research 来说，减少 agent 数，或把 agent 换到单价减半的 Sonnet 5.5，省的比降 effort 多。降 effort 主要减少思考、输出和工具调用的轮数。
   - [light-research.md 第 20 行](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/light-research.md#L20)写的「网上深度调研约占 5 小时窗口的 40%」，比本次观测（第一轮后 `five_hour` 只有 0.11）高得多。不过这只是一个数据点。

### 四、省用量的做法：证据与代价

1. **降 effort。** 证据见「二」第 1–5 条，是首选手段。代价：low 会跳过验证。缓解办法：加 Anthropic 的验证段落（Sonnet 5.5 提示指南给出原文），或把验证单独放到 high。置信度：中–高。
2. **给子代理和工作流 agent 单独指定模型和 effort。**
   - 模型的决定顺序：先看调用时传入的 `model` 参数（工作流脚本给某个阶段指定的模型也算这一项），再看子代理定义的 `model` 字段（`inherit` 表示用主会话模型），再看环境变量 `CLAUDE_CODE_SUBAGENT_MODEL`，最后才用主会话模型。v2.1.251 之前，环境变量排在最前。
   - 设 `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1`（v2.1.257 起）会把一个模型强加给所有子代理、teammate 和工作流 agent，包括内置的 Explore 和 Plan。
   - 内置 Explore 从 v2.1.198 起继承主会话模型（上限 Opus），不再固定用 Haiku。
   - effort：子代理前置元数据（frontmatter）里的 `effort` 覆盖会话档位，不写就继承。但如果设了 `CLAUDE_CODE_EFFORT_LEVEL`，前置元数据里的 effort 不生效。
   - 子代理继承会话的思考开关（v2.1.198 起），没有按子代理单独设的思考开关。
   - 工作流文档建议大型 run 前先看 `/model`，并让 Claude 给不需要最强模型的阶段用小模型。
   - 置信度：高。来源：[Claude Code 子代理文档](https://code.claude.com/docs/en/sub-agents)（无日期，提到 v2.1.257）、[Claude Code 模型配置](https://code.claude.com/docs/en/model-config)（2026-10-01 修改）、[Claude Code 工作流文档](https://code.claude.com/docs/en/workflows)（2026-09-29 修改）、anthropics/claude-code 的 CHANGELOG（2.1.257 条目；最新为 2.1.287）。
   - 代价：便宜的模型能否胜任，没有找到证据；不同模型之间不共享缓存。
3. **按阶段选模型。** `opusplan` 在计划模式下用 Opus，执行时用 Sonnet（Claude Code 模型配置页）。证据只有官方的定性建议（「二」第 9 条），没有找到 5.5 代 Sonnet 与 Opus 按用量计的质量对比。代价：会话中途换模型，缓存要冷启动，新模型也读不到旧模型的思考块。置信度：低–中。
4. **顾问模式（advisor）：便宜的执行者遇到难题时请教更强的模型。**
   - 4.6 代：Sonnet 执行、Opus 顾问，在 SWE-bench Multilingual 上比 Sonnet 单独高 2.7 分，每个智能体任务的花费低 11.9%（300 题 × 5 次）。Haiku 配 Opus 顾问，在 BrowseComp 上 41.2%，Haiku 单独 19.7%；仍比 Sonnet 单独低 29%，但花费少 85%。顾问每次只写约 400–700 token 的计划。来源：[The advisor strategy](https://claude.com/blog/the-advisor-strategy)（2026-04-09）。
   - 5.5 代：Opus 5.5 high 执行、Fable 5.1 顾问，得 90.1%，每次尝试 2.92 美元，比 Opus 5.5 单独用 high 高 1.7 分，花费约 2.1 倍。这正好落在执行者自己的 effort 曲线上：Opus 5.5 单独用 xhigh 是 91.1%、4.11 美元。另外，low 档的执行者几乎不请教顾问（300 题里 1 次），成绩反而比单独跑低 7 分。来源：[Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)（无日期，2026-10-01 抓取）。
   - 这些来源讲的都是 API 和托管 agent，没有找到在 Claude Code 会话里配置顾问的方法。
   - 置信度：中（未投票核实）。结论：对 agent-pm 暂不适用。
5. **先用 low 跑，失败的再用 high 重跑。**
   - Opus 5.5 在 SWE-bench Pro 上用 low，13% 失败；失败的用 high 重跑后，总通过率约 97%，每题约 0.17 美元。全部用 high 是 95.3%、0.29 美元。
   - 前提是有可检查的失败信号；代价是失败的任务要多花一倍时间。
   - 对 agent-pm：有测试的 engineering 可以用；调研和 PRD 没有自动的失败信号。
   - 置信度：中（未投票核实）。来源：[Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)。
6. **编排者-工作者。** 只有在有大量可并行、彼此独立的工作时才划算（「二」调研一条）。Claude Code 内置 claude-api skill 的成本优化文档说：工作是一条前后依赖的链，或一个上下文就装得下时，在实测的每个案例里，都是编排者所用的模型单独以较低 effort 跑更划算（单一来源，缓存日期 2026-09-25）。置信度：低–中。
7. **缓存和上下文管理。**
   - 提示缓存是最大的成本手段，在官方指南的基准上让智能体循环便宜 2.7–5.3 倍（未投票核实，[Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence)）。
   - 订阅下主对话的缓存保留 1 小时；一旦动用用量额度（usage credits）就变成 5 分钟（[Claude Code 成本页](https://code.claude.com/docs/en/costs)）。工作流 agent 默认 5 分钟。
   - CLAUDE.md 越短，每个子代理付出的启动成本越低（二手，见「三」第 4 条）。
   - 多 agent 设时间预算时，小团队的回答质量与单个 agent 相当，完成得快很多，但时间紧时会少搜、少验一点（Opus 5.5 提示指南）。
   - 置信度：中。
8. **在 xhigh、max 下禁止自发审查。** 一句系统提示，会话花费降约三分之一（Sonnet 5.5 实测，见「二」第 6 条）。置信度：中。

### 五、agent-pm 现状

除非另行标注，每条都经 3 票核实成立。

1. **模型和 effort 只在 `tasks/<task>.toml` 里设，每个任务必填，没有默认值。** 角色的 toml 和 `pipeline.toml` 都不能设这两项：`ROLE_KEYS` 里没有，`TASK_KEYS` 里有。各任务的值：
   - deep-research：`opus`、`ultracode`（[deep-research.toml#L1-L2](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/deep-research.toml#L1-L2)；注释写着 `claude --help` 里没有 ultracode，但 `claude -p` 接受）
   - light-research：`opus`、`high`（[light-research.toml#L1-L2](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/light-research.toml#L1-L2)）
   - product-design：`opus`、`high`（[product-design.toml#L1-L2](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/product-design.toml#L1-L2)）
   - engineering：`opus`、`xhigh`（[engineering.toml#L1-L2](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/engineering.toml#L1-L2)）
   - 规则所在（单个 agent 读取，未投票）：[pipeline.py#L142-L143](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/pipeline.py#L142-L143)、[pipeline.py#L335-L336](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/pipeline.py#L335-L336)。
   - `opus` 是别名。本 run 的会话记录显示它解析为 claude-opus-5-5（2026-10-01）。
2. **启动器怎么用这两个值。** 启动器把它们原样拼成 `--model`、`--effort`，新 run 和续跑都一样，同时加上 `--permission-mode auto --setting-sources user --strict-mcp-config`（[launch.py#L89-L92](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/launch.py#L89-L92)）。
   - 这两个参数只作用于顶层的 `claude -p` 会话，子代理没有单独的设置。
   - 启动器导出的环境变量里，没有任何子代理模型或 effort 变量。
   - 有一条说法被 3:0 推翻并改正：它说环境里只有 `PATH` 和 `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS`。实际上还有 `LINEAR_KEYCHAIN_SERVICE`，read_repo 和 repo_from_issue 类任务还有 `AGENT_PM_ISSUE`（[launch.py#L33](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/launch.py#L33)、[launch.py#L211-L212](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/launch.py#L211-L212)）。
   - 因为传了 `--setting-sources user`，用户级的 `~/.claude` 设置仍然会生效。这部分不在仓库里，没有读。
3. **各任务派出多少 agent。**
   - deep-research：`/deep-research` 轮约 100 个 agent，规模是它自己的，「never change or cap it」（[deep-research.md#L21](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/deep-research.md#L21)）。Ultra Code 轮在脚本里限制最多 100 个 agent，每条关键说法 3 票（[deep-research.md#L20](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/deep-research.md#L20)）。
   - light-research：一轮 3–6 个并行的检索 agent（[light-research.md#L30-L32](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/light-research.md#L30-L32)），目标是 10 分钟以内、不超过 5 小时窗口的 10%；网上深度调研的目标是约 20–30 分钟、约 40%（[light-research.md#L19-L20](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/light-research.md#L19-L20)）。
   - product-design（单个 agent 读取，未投票）：总有一个新起的审查子代理，必要时再加一个追问子代理（[product-design.md#L30](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/product-design.md#L30)）。
   - engineering（单个 agent 读取，未投票）：把构建交给外部的 `autopilot:build` skill，仓库里没写它派多少 agent、用什么模型（[engineering.md#L29](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/engineering.md#L29)）。
   - 所有任务和章程文件都没有给子代理或工作流 agent 指定模型或 effort。仓库里唯一用到的便宜模型是探测用的 Haiku，全仓库没有出现 Sonnet（两个读取 agent 的结论）。
4. **用量门控。**
   - router 每个周期（tick）在启动 run 之前跑一次 `claude -p "Reply with OK." --model haiku --output-format stream-json --verbose`，工作目录是 `work/`（[router.py#L434-L438](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/router.py#L434-L438)）。
   - 放行条件：状态不是 rejected、`five_hour` 小于 0.9、每个 `seven_day*` 窗口都小于 1；缺少 `five_hour` 就拦下（[router.py#L195-L202](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/router.py#L195-L202)）。
   - deep-research 在 mixed issue 的第二轮前另做一次刹车检查，阈值是 0.8（[deep-research.md#L22](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/deep-research.md#L22)）。
   - 门控放行时不记录用量；只有试运行（dry run）或被拦下时才写日志（未投票，[router.py#L439-L445](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/router.py#L439-L445)）。
5. **每个 run 不记录 token 或用量（单个 agent 读取，未投票）。**
   - 启动命令不传 `--output-format`（[launch.py#L89-L92](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/launch.py#L89-L92)）。
   - `runs.log` 的 start 行只记任务名，不记模型和 effort（[router.py#L457](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/router.py#L457)），7 天后会被清理。
   - 会话评论只有起止时间和退出码，可以算出时长（[sessions.py#L57-L63](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/sessions.py#L57-L63)）。
   - 会话记录本身带有每条消息的模型和 token 用量，见「三」第 5 条。
6. **改配置和做实验的约束（单个 agent 读取，未投票）。**
   - 启动器没有覆盖模型或 effort 的参数（[launch.py#L152-L155](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/launch.py#L152-L155)）。issue 只能通过 `Tasks` 标签选非默认任务，标签经 `pipeline.toml` 的 `[task_labels]` 映射（[README.md#L79](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/README.md#L79)）。所以每个实验组都要一个变体任务：新的 `.md` 和 `.toml`、加进角色的 `tasks`、一个新标签（[README.md#L83](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/README.md#L83)）。
   - 测试钉住了现有的值：`test_launch.py` 写死了 `--model opus` 和每个任务的 effort（[test_launch.py#L678](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/tests/test_launch.py#L678)、[#L698-L701](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/tests/test_launch.py#L698-L701)）；`test_router.py` 钉住 researcher 的任务列表（[#L988-L990](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/tests/test_router.py#L988-L990)）；`test_pipeline.py` 钉住 `[task_labels]`（[#L540](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/tests/test_pipeline.py#L540)）。
   - 同一个 issue 重跑会接着上次做，而不是从头来：调研报告复用同一个文件（[deep-research.md#L24](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/deep-research.md#L24)），已完成构建的 engineering issue 只会提一个问题（[engineering.md#L26](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/tasks/engineering.md#L26)）。所以对照实验要用 issue 的副本。
   - 每个角色同时只跑一个 run，router 在每小时的第 0 分和第 30 分各跑一次（[router.py#L407-L410](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/router.py#L407-L410)、[router plist#L19-L29](https://github.com/ophis/agent-pm/blob/c55bc56148aed3e2e24e72533fd0d0b1f7e8c2ca/scripts/com.ophis.agent-pm.router.plist#L19-L29)）。

### 六、建议

本节全部是本报告的综合；依据见各条括号。

1. **deep-research。**
   - 现状：deep-research 用 `--effort ultracode` 启动，等于 Opus 5.5 xhigh 加自动编排。两轮共约 160–200 个工作流 agent，没有一个指定模型，全部是 Opus 5.5，effort 跟随会话（「一」第 4 条、「五」第 1、3 条、「三」第 5 条）。
   - 建议一：在 run 的环境里设 `CLAUDE_CODE_SUBAGENT_MODEL=sonnet`，让没有指定模型的工作流 agent 用 Sonnet 5.5，包括 `/deep-research` 内部的拆题和综合 agent；主会话（编排、写报告）仍用 Opus。这不改变 `/deep-research` 的规模。实现上要给任务加一个键（例如 `subagent_model`），并由启动器导出为环境变量。现有的 `TASK_KEYS` 和测试都要跟着改（「五」第 1、6 条）。
   - 建议二：在 deep-research.md 的 Ultra Code 规则里，要求脚本给检索和核实 agent 设 `effort: 'medium'`（或显式的模型）。
   - 待确认：ultracode 能否在 medium 或 high 下打开。如果可以，主会话也从 xhigh 降到 high。
   - 证据：知识工作曲线平；工作流 agent 的花费大头在上下文；Sonnet 5.5 的单价是 Opus 5.5 的一半，缓存读取同价。
   - 风险：Sonnet 核实票的辨别力没有证据。
   - 置信度：低–中。改之前先做实验 E3。
2. **engineering。**
   - 主会话从 xhigh 降到 high。理由：规划、协调和审查都属于「验证重要」的工作；xhigh 比 high 只多约 1.4 分，花费是 2.5 倍；xhigh、max 会自己发起审查，与 autopilot 的审查者重复。
   - `autopilot:build` 的实现者用 medium，并在要求里加验证指令；审查者用 high。这要在 autopilot 的 agent 定义里写 `model` 和 `effort`。该插件不在本仓库，未核实它现在怎么写（见缺口）。
   - 不要导出 `CLAUDE_CODE_EFFORT_LEVEL`，它会盖掉 agent 定义里的 effort。
   - 有测试的任务可以考虑「low 先跑、失败再用 high」。
   - 置信度：中。实验 E2。
3. **light-research。** 主会话从 high 降到 medium（知识工作曲线平；Opus 5.5 medium 不低于 Opus 5 high）。3–6 个检索 agent 在 Agent 调用里传 `model: "sonnet"` 试一试。置信度：中。实验 E1。
4. **product-design。** 起草用 medium；必定会派的审查子代理用 high，因为它承担验证。没有 PRD 的专门证据。置信度：低–中。实验 E4。
5. **用量探测。** 已经用 Haiku，不变。
6. **run 内部的子代理（通用规则）。**
   - 每类 agent 都显式写模型和 effort，不靠继承：检索、格式整理、Linear 状态更新、查状态用 Haiku 4.5 或 Sonnet 5.5 low；核实、审查用 Opus 5.5 high，或经实验后用 medium；实现用 medium，并加验证指令。
   - 注意：Explore 在 Opus 会话里也跑 Opus。工作流脚本可以给每个 `agent()` 传 `opts.model`、`opts.effort`，按阶段设置。
   - 依据：「四」第 2 条。置信度：中。
7. **用户交互会话。**
   - 默认用 Opus 5.5 medium，这也是 Claude Code 的默认。注意顶层 `effortLevel` 设置对 Opus 5.5 不生效。
   - 修 bug、代码审查、需要验证的工作切到 high。xhigh 和 max 只用在实测有收益、可以放手让它跑的难题上。
   - 在环的头脑风暴和小改动可以用 low。
   - 换档尽量放在任务交界处，例如新开会话，因为 API 上改顶层 effort 会让缓存失效；Claude Code 的 `/effort` 是否同样如此，未查到（见缺口）。
   - 大量检索交给指定了便宜模型的子代理。
   - 置信度：中。

### 七、对照实验方案

本节是本报告的设计，以「二」第 2–5 条和下列来源为依据：

- Anthropic 建议，起步时用 20–50 个来自真实失败的任务，因为早期改动的效果大，小样本就够；每个任务要多次试验，并区分 pass@k 和 pass^k（每次 75% 成功，3 次全过约 42%）。每次试验都要从干净的环境开始；评分要看结果，并读记录核对评分器（[Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)，2026-01-09，未投票核实）。
- Anthropic 自己的 effort 对比用了 70 个任务，每档每题 5 次（2026-09-25 的文章）。
- 「约 50 个案例、每种配置至少 5 次」这条说法没有找到出处。

1. **实验组。** 每组只改一处。
   - E1 light-research：`opus/high` 对 `opus/medium`。
   - E2 engineering：`opus/xhigh` 对 `opus/high`。
   - E3 deep-research：现状，对加上 `CLAUDE_CODE_SUBAGENT_MODEL=sonnet`。
   - E4 product-design：`opus/high` 对 `opus/medium`。
2. **样本。**
   - 从已完成、结果被接受的 issue 里挑，用副本，因为重跑原 issue 会接着上次做（「五」第 6 条）。
   - 试点规模：E1 和 E4 各 8 对，E2 用 6 个有测试的 issue，E3 用 4 个。每对两组各跑 1 次；E2 预算允许时各跑 2 次。
   - 实施：在一个实验用的 agent-pm 工作树里加变体任务和标签，把 `pipeline.toml` 的 `[docs]` 指到草稿仓库或草稿分支，以免污染正式文档。
3. **控制变量。**
   - 同一对的两组在同一天按 ABBA 交替运行，减少网上内容变化和时段差异。
   - 每次在干净的工作目录里跑；记下 `claude --version`；会话中途不换档。
   - 需要用 `five_hour` 差值时，暂停 router，不让别的 run 并行。
4. **指标。**
   - 质量：用户盲评，即两组产出去掉组别、随机排序后两两比较。另加检查表：
     - 调研：子问题覆盖；抽 5 条说法，对照来源计错；引用是否有效、是否支持说法；缺口是否写明。
     - PRD：模板完整；未决问题；审查意见数。
     - engineering：测试和持续集成（CI, Continuous Integration）是否通过；审查意见数和严重度；审查后的修复提交数；用户是否原样接受。
   - 用量：按模型汇总每个 run 的 token，包括主会话、子代理和工作流，按会话记录里的 `message.usage` 分输入、写缓存、读缓存、输出、思考；每个 run 前后各跑一次用量探测，记 `five_hour` 和 `seven_day`，只有两位小数，所以只适合用量 ≥ 5% 的 run；agent 数取 `journal.jsonl` 里的 `started` 条目数。也可以在 run 环境里设 `CLAUDE_CODE_ENABLE_TELEMETRY=1` 打开遥测（OpenTelemetry, OTel）：`claude_code.token.usage` 和 `claude_code.cost.usage` 带 `model`、`effort`（v2.1.274 起）和 `query_source` 属性，`workflow.run_id` 可以把整个工作流的请求归到一起（[Monitoring usage](https://code.claude.com/docs/en/monitoring-usage)，无日期，2026-10-01 抓取，未投票核实）。订阅用户还可以用 `/usage` 看近 24 小时和 7 天按 skill、子代理、插件、MCP（Model Context Protocol）服务器的用量占比；这个数是近似值，只来自本机的会话记录（[Claude Code 成本页](https://code.claude.com/docs/en/costs)，未投票核实）。
   - 时长：会话评论里的起止时间。
5. **判定规则。** 满足以下全部条件，就改用便宜的一组：
   - 盲评中，至少 7/8 对（E2 是 5/6）便宜组不差。
   - 没有一对出现「便宜组有事实错误或测试失败、另一组没有」。
   - 用量的配对比值的中位数至少下降 25%。
6. **样本量能说明什么。** 下面是单侧 95% Clopper–Pearson 下界，由本报告计算，表示「便宜组不差」的比例至少有多高：

   | 「不差」的对数 | 下界 |
   |---|---|
   | 8/8 | 69% |
   | 7/8 | 53% |
   | 10/10 | 74% |
   | 20/20 | 86% |
   | 28/30 | 81% |

   所以试点只能排除大的退步。最常用的任务（light-research）要想最终定案，应扩到 20 对以上。符号检验（sign test）里，8 对中 7 对便宜组更好时 p≈0.035。
7. **顺带测订阅换算。** 把每个 run 的 `five_hour` 增量，对按 API 单价折算的花费做回归，按模型分开。这可以在本地估出「每 1% 的 5 小时额度对应多少 token」，补上官方没有公布的换算。

## 对已知说法的更正

1. **「effort 等级包括 Ultra Code」不对。** Ultra Code（ultracode）是 Claude Code 的设置，不是 effort 档位。`--effort ultracode` 会打开它，并把 effort 设成 xhigh。所以 deep-research 实际上是 Opus 5.5 xhigh 加自动编排（「一」第 4 条）。
2. **issue「现状」里关于 agent-pm 配置的四条都成立。** 补充两点：
   - `opus` 是别名，本 run 解析为 claude-opus-5-5。
   - deep-research 的刹车检查也用 Haiku。
   - 见「五」第 1、4 条。
3. **「subagent 的模型和 effort 多数默认跟主会话一样」：实际比「多数」更彻底。**
   - 仓库里没有任何地方给子代理或工作流 agent 指定模型或 effort，本 run 的 159 个工作流 agent 全是 claude-opus-5-5。
   - 另外，内置的 Explore 从 v2.1.198 起不再固定用 Haiku，而是继承主会话模型（「四」第 2 条、「五」第 3 条）。
4. **「用订阅，有 5 小时窗口和每周额度」：成立。** router 的门控读的就是 `five_hour` 和 `seven_day*` 两种窗口。这两个窗口跨模型共享，换模型不能恢复额度（「三」第 1 条、「五」第 4 条）。
5. **调研简报中待核实说法的结果。**
   - 「Sonnet 5.5 默认 high」：只在 API 上成立，Claude Code 里是 medium。
   - 「xhigh 是 Claude Code 默认档」：只对 Opus 4.7 成立。
   - 子代理模型的决定顺序：调用时传入的模型 > 定义里的 `model` > 环境变量 > 主会话。环境变量排第一是 v2.1.251 之前的旧顺序，2026-08-19 的一篇博客仍按旧顺序写。
   - 「约 50 个案例、至少 5 次」：没找到。Anthropic 说的是起步 20–50 个任务、多次试验，没有固定 5 次的规定。
   - 「顾问模式落在执行者自己的 effort 曲线上」：官方成本页对 Opus 5.5 执行、Fable 5.1 顾问的组合这样写，但未经投票核实。2026-04 的博客讲的是 4.6 代，结果是 +2.7 分、花费 −11.9%。

## 缺口

1. **订阅换算。** 没有官方的 token 到 5 小时和每周额度的换算，也没有 Opus、Sonnet、Haiku 之间的比例；订阅是否像 API 那样计算缓存读写，也不知道。本 run 只有两个数据点（`five_hour` 0.11、0.15），而且 5 小时窗口里还可能有别的用量。
2. **便宜模型的质量。** 没有证据比较 Sonnet 5.5 与 Opus 5.5 在调研、PRD、核实投票上按用量计的质量。Haiku 4.5 用于机械性工作只有个人经验；它没有 effort 参数，也不能开 ultracode。
3. **PRD 和产品设计。** 没有专门的评测。
4. **未经投票核实的说法。** 下列内容只经单个抓取 agent 带原文摘录，未投票：
   - SWE-bench Pro 的 effort 曲线；
   - 知识工作曲线平（Fable 5 测得）；
   - DeepResearch Bench II；
   - 低档跑、失败重跑；
   - 两组顾问模式的数据；
   - API 价格；
   - 4 倍和 15 倍 token、BrowseComp 80% 方差；
   - agent team 7 倍 token；
   - `/usage` 和 OTel 的属性；
   - 工作流缓存的共享条件；
   - 起步 20–50 个任务。
5. **Claude Code 的未知点。**
   - 在 `claude -p` 下能否在 medium 或 high 打开 ultracode，没有核实。文档只说 ultracode 按会话 effort 运行，以及 `--effort ultracode` 会设成 xhigh。
   - `/effort` 在会话中途换档是否让缓存失效，没查到。
   - 没有找到在 Claude Code 里配置顾问模式的方法。
   - 会话记录里看不到每个 agent 实际用的 effort，只能依据文档说它继承会话。
6. **仓库之外的配置。** 本次只读目标仓库，没有读：
   - autopilot 插件里实现者和审查者的 agent 定义，即它们的 `model` 和 `effort` 前置元数据；
   - 用户级的 `~/.claude` 设置；
   - `/deep-research` 内置脚本，只搜了 `model|effort`。
7. **本地读取的局限。**
   - 第二轮 60 条说法里，只有 18 条投了票（上限所限），其余 42 条作为单个 agent 的读取结果使用，已逐条标注。
   - 读取 agent 报告没有 Grep 和 Glob 可用，只能逐个文件读，可能漏掉其他文件里的相关代码；`prune.py` 没有读。
8. **实测数据的局限。** 会话记录里的流式用量可能少计输出。按 API 单价折算只是相对参照。「多 agent 的花费大头在上下文」这个推断只基于本 run 两轮的数据。
9. **时效。** Opus 5.5（2026-09-22 发布）和 Sonnet 5.5（2026-09-28 发布）的默认值和行为可能还会变；文中涉及的 Claude Code 版本最新到 2.1.287。本报告的信息截至 2026-10-01。
