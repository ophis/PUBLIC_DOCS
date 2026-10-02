# Report: TASK-29 AI 时代软件工程师的出路：产品与销售是否比工程更重要

> 2026-09-28。这是一次深度调研（deep research）工作流运行。它抓取了 21 个来源，提取 104 条论断，对排名前 25 条做了 3 票对抗性核验。其中 4 条因会话额度中断没有拿到有效票，之后补投了 3 票。最终结果：18 条确认，7 条驳回。
> 置信度标注：**已验证（高/中）** = 经 3 票核验；**未验证** = 未经核验的线索，包括核验者检索时顺带发现的材料，只能当线索看。结论、建议和对照表是我基于这一次运行做的综合判断。
> 覆盖情况：本次运行的已验证论断几乎都集中在“劳动力市场数据”和“AI 生产率证据”两块。书单、视频、论坛、中文来源、产品/销售岗位数据和转型案例都**没有进入核验**，详见“缺口”。

## 结论与建议

“产品与销售比工程更重要”这个判断，本次调研的数据**不支持它的原始形式**，但支持一个更窄的版本：**贬值的是初级、可替代的实现工作，而不是工程能力整体**。
- AI 对就业的冲击集中在初级：美国 22–25 岁、AI 暴露度最高的职业，就业比对照组低约 19%；有经验的员工没有类似缺口。
- 招聘向资深倾斜：软件开发岗位中资深占 69.3%，入门级只占 4.5%。
- 薪酬同样向资深倾斜：2025 年入门级软件工程师薪酬中位数只涨 1.64%，Staff 级涨 7.52%。
- AI 提效是真实的，但温和且不均：大企业现场随机对照试验约 +26%，远低于早期实验室研究的 58%；METR 的试验一度测出变慢 19%。DORA 发现 AI 提高吞吐，同时降低交付稳定性。

市场仍在为“能驾驭系统、能验证 AI 产出的资深工程能力”付溢价。至于产品和销售是否在**相对工程**升值：唯一一条对比薪资增速的论断在核验中被驳回，销售、解决方案工程师、前线部署工程师（FDE, Forward Deployed Engineer）的数据也没有进入核验。所以这一半目前**没有得到证实，也没有被证伪**。

建议按证据强度排列：
1. **别放弃工程，目标定为“资深”**。数据最硬的一条结论是：冲击集中在入门级和中级，资深岗位的需求在上升，Staff 级的薪酬涨幅最大。可执行的做法是主动承担端到端交付、系统设计、代码评审和故障处理，这些是区分“资深”的工作。
2. **把验证和稳定性当成核心技能**。AI 让变更更多、更快，DORA 发现交付稳定性随之下降。自动化测试、小批量变更和快速反馈回路的价值在上升。Karpathy 的 MenuGen 案例也说明，从演示到上线的最后一段仍是工程活。
3. **用测量而不是感觉判断 AI 帮了你多少**。METR 的试验里，开发者实际变慢了，却自认为快了 20%。建议记录自己的周期时间（cycle time）和返工率，再决定在哪些任务上重度使用 AI。
4. **把产品和销售当作工程的放大器，而不是替代品**。同一批人自报：AI 带来的速度提升中位数是 3 倍，工作价值的提升只有 1.4–2 倍。产出更多代码不等于创造更多价值。“产品判断就是瓶颈”这个推论在核验中被驳回，不过“补上选题和交付能力”仍是低成本、高期权的选择。具体做法：
   - 端到端做一个小产品，并真正收到钱；
   - 定期直接和用户交谈；
   - 在公司内主动承担面向客户的技术工作。
5. **向产品或销售转型，先做内部试验，再换赛道**。产品工程师、FDE、解决方案工程师、开发者关系（DevRel, Developer Relations）、技术创始人等路径，本次运行都没有核验数据或案例。建议先通过轮岗或兼任验证自己的兴趣和能力，再做决定。
6. **如果你是初级工程师**：你正处在受冲击最明显的层级。尽快积累“可证明的交付”，也就是上线过的产品和负责过的系统，比单纯刷算法题更能缩短到中高级的距离（综合判断）。

## 分项发现

### 1. 劳动力市场：冲击集中在初级，资深需求上升

**初级员工就业明显受挤压，有经验的员工没有。** 已验证（高，3-0）。
- 斯坦福数字经济实验室（Stanford Digital Economy Lab）用 ADP 工资单数据做了“煤矿里的金丝雀”（Canaries in the Coal Mine）研究，覆盖 2022 年 11 月到 2026 年 6 月。结果如下：
  - AI 暴露度最高的两个五分位职业中，22–25 岁员工就业下降约 11%；暴露度最低的三个五分位中，同龄人就业增长约 10%。
  - 截至 2026 年 6 月，前者比“若与后者同步”时低约 19%。2025 年 7 月这个缺口是 15%，说明在扩大。
- 同一研究还指出：“没有看到全经济范围的 AI 岗位替代……有经验的员工没有类似缺口。”
- 来源：[Stanford Canaries 2026-08 更新](https://digitaleconomy.stanford.edu/news/canariesaug26/)

**软件开发岗位总量仍低于疫情前，结构向资深倾斜。** 已验证（高，3-0）。
- Indeed 招聘实验室（Indeed Hiring Lab）的数据：
  - 美国软件开发职位发布量仍比 2020 年 2 月基线低约 27.5%。
  - 以 2025 年 1 月为基准，到 2026 年 5 月，资深岗位发布量 +13.5%，中级 −6.7%，入门级 −6.3%。资深岗位同比增长 14.7%。
  - 各行业中，软件开发的资深化程度最高：2026 年一季度资深岗位占 69.3%，入门级只占 4.5%。
  - 科技行业资深岗位占比上升约 9 个百分点，主要挤占的是中级岗位（−7.7 个百分点），入门级占比只降了 1 个多百分点。
- 解读：被挤压的不只是新人，还有“中级、执行型”岗位。
- 来源：[Indeed：AI 与职位发布](https://hiringlab.indeed.com/2026/07/08/ai-and-job-postings-from-destruction-to-creation/)、[Indeed：劳动力市场向资深倾斜](https://hiringlab.indeed.com/2026/07/23/the-labor-market-is-tilting-toward-seniority/)

**薪酬增长集中在资深和 Staff 级。** 已验证（高，3-0；Levels.fyi 为用户自报数据）。
- 2025 年美国软件工程师总薪酬中位数：

  | 级别 | 2024 | 2025 | 变化 |
  |---|---|---|---|
  | 入门级（Entry Level） | $152,500 | $155,000 | +1.64% |
  | Staff | $425,500 | $457,500 | +7.52% |
  | Principal | $590,000 | $551,151 | −6.58% |

- Principal 级下降，说明“越资深涨得越多”并不是单调关系。
- 来源：[Levels.fyi 2025 年终报告](https://levels.fyi/2025)

### 2. 生产率：AI 提效真实但温和、不均，并有代价

**大规模现场随机对照试验：约 +26%。** 已验证（高，3-0）。
- 在微软（Microsoft）、埃森哲（Accenture）和一家匿名财富 100 强公司做了三项随机对照试验，共 4,867 名开发者。使用 GitHub Copilot 的开发者完成任务数（PR）增加约 26%（标准误 10.3%）。
- 这远小于早期实验室研究（Peng 等，2023）测得的任务耗时减少 58%。作者给出的解释是：编码只是开发者工作的一部分，省下的编码时间只有一部分会变成更多编码。
- 来源：[Management Science 论文](https://pubsonline.informs.org/doi/10.1287/mnsc.2025.00535)

**METR 的随机对照试验：先测出变慢，后来转为可能加速，但置信区间都跨零。**
- 2025 年初的试验：资深开源开发者使用 AI 工具后，完成真实任务**多花 19% 时间**。已验证（中，2-1）。
- 同一试验中，开发者事前预期提速 24%；即使实际变慢，事后仍认为 AI 让自己快了 20%。**自报的提效数字不可靠。** 已验证（高，3-0）。
- 2026 年 2 月的跟进试验（2025 年 8 月起，57 名开发者）：老参与者估计耗时 −18%（置信区间 −38% 到 +9%），新参与者 −4%（−15% 到 +9%）。METR 认为 2026 年初的加速“很可能”比 2025 年初大，但也说自己的数据只是“很弱的证据”，中心估计很可能不是真实影响的好代理。已验证（高，3-0）。
- 来源：[METR 2025-07](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)、[METR 2026-02 更新](https://metr.org/blog/2026-02-24-uplift-update/)

**速度提升大于价值提升（自报）。** 已验证（中，2-0）。
- METR 在 2026 年 2–4 月调查了 349 名技术工作者，其中 87 名软件工程师。自报 AI 带来的工作价值提升中位数为 1.4–2 倍，速度提升中位数为 3 倍。
- 注意：这是自报数据。“差距来自产品判断是瓶颈”这一推论在核验中被驳回，见“修正”。
- 来源：[METR AI 使用调查](https://metr.org/blog/2026-05-11-ai-usage-survey/)

**DORA 2025：采用几乎普及，吞吐上升，稳定性下降。** 已验证（高，3-0）。
- DORA（DevOps Research and Assessment）2025 报告调查了近 5,000 名技术从业者：
  - 90% 在工作中使用 AI，超过 80% 认为 AI 提高了生产率（自报）；
  - 30% 对 AI 生成的代码几乎不信任或完全不信任。
- AI 采用与交付吞吐、产品绩效正相关，与交付**稳定性负相关**。没有强自动化测试、版本控制实践和快速反馈，更多变更会带来更多不稳定。
- 来源：[DORA 2025](https://dora.dev/dora-report-2025/)、[Google Cloud 公告](https://cloud.google.com/blog/products/ai-machine-learning/announcing-the-2025-dora-report)

### 3. 代表人物观点：Karpathy

**“人人都是程序员”：门槛降低，但这是观点，不是测量。** 已验证（高，3-0，观点转述准确）。
- 2025 年 6 月，Andrej Karpathy 在 YC AI 创业学校（YC AI Startup School）的主题演讲《Software Is Changing (Again)》中说：自然语言编程（他称为“软件 3.0”，Software 3.0）让“人人都是程序员”；过去要学五到十年才能做软件，“现在不是这样了”。
- 核验者一致提醒，这句话只支持“实现变便宜”，不支持“工程技能的市场价值下降”。同一场演讲中，他还主张：
  - 做“钢铁侠战衣”（Iron Man suit，部分自主），而不是完全自主的代理；
  - “这是代理的十年，而不是代理之年”（decade of agents）。

**MenuGen：写代码只用了几小时，上线又花了一周。** 已验证（高，3-0，单一个人案例）。
- Karpathy 用 AI 几小时就做出了 MenuGen 的演示。要把它变成真实产品，还得接入认证、支付、域名和 Vercel 部署，这些“不是代码”的工作又花了一周。
- 核验者一致指出三点限制：
  - 这里的“交付”指 DevOps 和集成，本身仍是工程工作，**不能拿来支持“产品/销售更重要”**；
  - Karpathy 自称不是 Web 开发者；
  - 他认为这个瓶颈是工具缺口，会被代理自动化。
- 来源：[YC 演讲页面](https://www.ycombinator.com/library/MW-andrej-karpathy-software-is-changing-again)、[MenuGen 博文](https://karpathy.bearblog.dev/vibe-coding-menugen/)

**“人仍是瓶颈”并不等于反驳 C2。** 驳回（补投 0-3）。
- 原论断把下面这句话解读为反驳 C2：“给我一个一万行的 diff 没用，我仍是瓶颈，尽管这一万行瞬间就能生成，我得确保它没引入 bug”，也就是“要把 AI 拴在绳子上”（on the leash）。
- 三位核验者都认为原话属实，但解读有两处错误：
  - 原话本身承认 AI 生成代码远快于人。它说的是人转向规格和验证的角色，而不是人在实现上能与 AI 竞争；
  - 解读已经过时。据核验者检索，Karpathy 2026 年 1 月称自己已转为“80% 由代理写代码”（未验证），同时仍强调重要代码要“像鹰一样盯着”。
- 更准确的表述：AI 负责大部分生成，人负责定义与验证。

**核验者检索到的后续表态。** 未验证（线索）。
- 2025 年 10 月，Karpathy 在 Dwarkesh 播客中说，写 nanochat 这类新颖代码时代理“净无帮助”。
- 2026 年 1 月，他说自己从“80% 手写”变成“80% 由代理写”，同时担心手写能力退化。
- 2026 年 4 月在红杉（Sequoia）Ascent 大会上，他说“氛围编程抬高了下限”（vibe coding raises the floor）。他认为人要负责品味、工程、设计和理解，顶尖工程师的差距被放大了。
- 这些表态更接近“实现变便宜，但资深判断更值钱”，而不是“工程不再重要”。
- 来源：[Dwarkesh 访谈](https://www.dwarkesh.com/p/andrej-karpathy)、[X 2026-01](https://x.com/karpathy/status/2015883857489522876)、[Sequoia Ascent 2026](https://karpathy.bearblog.dev/sequoia-ascent-2026/)

### 4. 发展方向与转型路径

本次运行抓取了下列材料，但没有任何相关论断进入核验。以下只列线索：
- **前线部署工程师（FDE）**：驻在客户一侧、把产品落到客户场景里的工程岗位，是“工程 + 销售/交付”最直接的交叉点。线索：
  - [The Pragmatic Engineer：Forward Deployed Engineers](https://newsletter.pragmaticengineer.com/p/forward-deployed-engineers)
  - [Paraform：FDE 需求增长四倍](https://www.paraform.com/blog/forward-deployed-engineer-demand-quadrupled)（招聘平台博客，需谨慎看待）
  - [a16z：服务驱动增长](https://a16z.com/services-led-growth/)
- **产品经理（PM, Product Manager）就业市场**：[Lenny's Newsletter：产品岗位市场现状](https://www.lennysnewsletter.com/p/state-of-the-product-job-market-in-ee9)
- **AI 写大部分代码后工程师做什么**：
  - [The Pragmatic Engineer：When AI writes almost all code](https://newsletter.pragmaticengineer.com/p/when-ai-writes-almost-all-code-what)
  - [The Pragmatic Engineer：The future of software engineering with AI](https://newsletter.pragmaticengineer.com/p/the-future-of-software-engineering-with-ai)
  - [Kent Beck：90% of my skills are now worth $0](https://newsletter.kentbeck.com/p/90-of-my-skills-are-now-worth-0)
- **悲观派观点**：[Forbes：Dario Amodei 重申 AI 就业警告](https://www.forbes.com/sites/kolawolesamueladebayo/2026/02/21/dario-amodei-doubled-down-on-his-ai-jobs-warning-heres-whats-different-now/)
- **斯坦福 10 万开发者生产率研究（二手转述）**：[Proxify 摘要](https://proxify.io/articles/stanford-study-of-100000-developers-on-engineering-productivity)
- **论坛**：[V2EX 讨论](https://www.v2ex.com/t/1215275)、[Hacker News 讨论](https://news.ycombinator.com/item?id=48037249)

## 对已知判断的修正（对照表，综合判断）

| 判断 | 结论 | 依据 |
|---|---|---|
| C1：AI 让实现大幅变容易，工程实现技能的市场价值在下降 | **部分成立**。“变容易”成立，但幅度比宣传的温和：现场约 +26%，METR 结果跨零。“价值下降”只对入门级和中级成立，资深岗位的需求和薪酬在上升 | Copilot 随机对照试验、METR、Indeed、Levels.fyi、Stanford |
| C2：人在工程实现上无法与 AI 竞争 | **只对“生成代码”成立**。对“把代码变成可靠系统”不成立：AI 提高吞吐，同时降低稳定性；资深经验反而更值钱。“Karpathy 反驳 C2”的论断以 0-3 被驳回，因为他的原话承认 AI 生成更快，把人定位为验证者 | DORA、Stanford、Indeed、Karpathy 核验记录 |
| C3：产品判断力是 AI 没有的，而且在升值 | **未证实**。“速度提升大于价值提升”只是自报数据；“差距来自产品判断”的推论被 3-0 驳回。Karpathy 2026 年“人负责品味与理解”的说法未验证 | METR 调查；核验记录 |
| C4：销售/分发相对工程更重要 | **未证实**。本次没有核验到任何销售、解决方案工程师或 FDE 的岗位和薪酬数据。PM 薪酬增速快于工程师的论断被驳回。MenuGen 的“最后一公里”是 DevOps，不是销售 | 核验记录 |

补充修正：常被引用的“AI 放大器”框架（DORA“AI 放大组织已有的强项和弱项”）在本次核验中以 0-3 被驳回。驳回的具体理由没有记录在运行结果里，引用这句话前建议查阅原文。

## 缺口

**被驳回的论断**（运行结果只记录票数，不含驳回理由）：
- “Claude Code 发布后美国软件开发职位增长近 15%，同期总职位下降 7%”（Indeed，1-2）；
- “2025-05 到 2026-05 的增量中 71% 来自资深岗位，37% 来自标题含 AI 的岗位”（Indeed，0-3）；
- “2025 年软件工程师薪酬中位数同比 +2.67%”（Levels.fyi，0-3）；
- “PM（+4.55%）和工程经理（+9.64%）薪酬增速快于工程师（+2.67%）”（Levels.fyi，0-3）；
- “DORA：AI 是放大器，回报主要来自组织系统”（DORA，0-3）；
- “Karpathy 说‘人仍是瓶颈’，因此反驳了 C2”（YC 演讲，补投 0-3）。理由见第 3 节。
- “速度提升高估价值提升，部分是因为人们用 AI 去做本不值得做的事，这说明‘做什么’才是瓶颈”（METR 调查，补投 0-3）。三位核验者都指出：METR 只把替代效应列为可能原因之一，而且作者说差距比预期小；原文完全没有提到产品判断。有核验者提到 Faros AI 的遥测数据：PR 合并量 +98%，评审时间 +91%。这指向“评审/验证是瓶颈”，本身未验证。

**未覆盖的部分**：
- 产品/销售岗位数据：PM、销售、解决方案工程师、FDE 的职位和薪酬趋势都没有进入核验，这是回答 C3/C4 最关键的缺口。
- 转型案例：没有核验到具体人物的转型经历。
- 代表人物：只核验了 Karpathy。黄仁勋（Jensen Huang）、Sam Altman、Dario Amodei、Paul Graham、Marty Cagan、DHH 等人的观点都没有核验。
- 中文来源：V2EX 和知乎只抓到一个 V2EX 帖子，内容没有进入核验；没有覆盖中文博客和播客。
- 书单和视频：本次运行没有产出，下面的清单来自通用知识，**未经本次调研核实**。

## References

### 书单（未经本次调研核实，仅作起点）

| 书 | 简介 | 推荐理由 |
|---|---|---|
| *Inspired*（Marty Cagan） | 科技公司如何做产品：产品发现、赋能团队 | 工程师理解“PM 到底做什么”的标准入门 |
| *The Mom Test*（Rob Fitzpatrick） | 怎样和用户交谈，才不会被客气话误导 | 篇幅短，马上能用；补的正是“想清楚人真正需要什么” |
| *Continuous Discovery Habits*（Teresa Torres） | 每周接触用户、用机会解决方案树（opportunity solution tree）做持续发现 | 把产品判断变成可练习的习惯 |
| *Obviously Awesome*（April Dunford） | 产品定位方法 | 连接产品和销售：先讲清楚“为什么是你” |
| *Traction*（Gabriel Weinberg、Justin Mares） | 19 种获客渠道和“靶心”（Bullseye）筛选法 | 技术人补分发能力的系统框架 |
| *Founding Sales*（Pete Kazanjy） | 写给创始人和早期团队的销售手册 | 面向“没做过销售的技术人” |
| *SPIN Selling*（Neil Rackham） | 基于大量销售访谈研究的提问法 | 理解复杂 B2B 销售，适合走解决方案工程师 / FDE 路线 |
| *The Sales Acceleration Formula*（Mark Roberge） | HubSpot 用数据化方法扩张销售团队 | 工程师思维看销售：可度量、可迭代 |
| *The Staff Engineer's Path*（Tanya Reilly） | Staff+ 工程师的职责、影响力和技术领导 | 对应“走向资深”这条数据最硬的路线 |

### 视频与播客
- [Andrej Karpathy：Software Is Changing (Again)](https://www.ycombinator.com/library/MW-andrej-karpathy-software-is-changing-again)，YC AI 创业学校，2025-06。提出“软件 3.0”、部分自主，以及 MenuGen 案例。本报告已验证其中两条论断。
- [Dwarkesh Podcast：Andrej Karpathy](https://www.dwarkesh.com/p/andrej-karpathy)，2025-10。谈代理的局限和“AGI 还要十年”。未验证。
- [Karpathy：Sequoia Ascent 2026 讲稿](https://karpathy.bearblog.dev/sequoia-ascent-2026/)，2026-04。谈“抬高下限”、代理式工程（agentic engineering）和人的判断。未验证。

### 其他资料（本次运行抓取的全部来源）
- 一手数据：
  - [Stanford Canaries 2026-08](https://digitaleconomy.stanford.edu/news/canariesaug26/)
  - [Indeed 2026-07-08](https://hiringlab.indeed.com/2026/07/08/ai-and-job-postings-from-destruction-to-creation/)
  - [Indeed 2026-07-23](https://hiringlab.indeed.com/2026/07/23/the-labor-market-is-tilting-toward-seniority/)
  - [Levels.fyi 2025](https://levels.fyi/2025)
  - [METR 2025-07](https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/)
  - [METR 2026-02](https://metr.org/blog/2026-02-24-uplift-update/)
  - [METR 2026-05 调查](https://metr.org/blog/2026-05-11-ai-usage-survey/)
  - [DORA 2025](https://dora.dev/dora-report-2025/)
  - [Management Science：Copilot 现场试验](https://pubsonline.informs.org/doi/10.1287/mnsc.2025.00535)
- 二手报道与博客：第 4 节列出的 Pragmatic Engineer、Lenny's Newsletter、Kent Beck、a16z、Paraform、Forbes、Proxify。
- 论坛：[V2EX t/1215275](https://www.v2ex.com/t/1215275)、[HN 48037249](https://news.ycombinator.com/item?id=48037249)
