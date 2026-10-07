# Report: Muse 运行时架构与每用户 VM

Web 调研（web）。

Light Research. Angles: 产品识别; 运行时架构; VM 调度与生命周期; 租户与隔离安全; 定价与计算限额. No independent verification stage: each finding is checked only by the agent that found it.

## 结论与建议

Muse 是 Meta 的个人 AI agent（MSL（Meta Superintelligence Labs，Meta 超级智能实验室）出品，2026-09-08 在美国上线）；它为每个用户分配一台专属、长期存在的云端 Linux VM（虚拟机）"Muse Secure VM"，而非按会话或任务分配（整体置信度：高，Meta 官方明确表述）。官方公开的是 VM 内部结构：agent 框架 Hatch 跑在 systemd-nspawn 容器"runtime cell"内，Sentinel（出网审批 agent）和凭据服务 hatch-authd 跑在容器外、同一台 VM 上；底层 hypervisor、云厂商、规格、空闲策略均未披露。以下为综合判断：用户实测显示 VM 跑在 Cloud Hypervisor/KVM（基于内核的虚拟机）上，2 vCPU / 约 8 GB 内存 / 100 GB 持久卷，VM 本身可随时替换、加密持久卷重新挂载到预启动的 VM 上（约 40 秒），这些是实测而非承诺，可能随版本变化。计费按每周"Muse tokens"配额（Free / Power $20 / Maximum $100），不按算力或 VM 时长；建议把官方表述当作可靠事实，把硬件与生命周期细节当作单一来源的观测值使用。

## 发现

### 产品识别

- 目标产品是 Meta 的 Muse（muse.ai），2026-09-08 发布，"The World's First Personal AI Agent Built for Everyone"；替用户处理邮件、订行程、填表、购物，关掉 app 后继续工作。Confidence: high [1]
- 模型为 Muse Spark 系列（MSL 首个模型系列，2026-04-08 发布）；Meta 称 Muse 跑在 Muse Spark 1.3（2026-09-02 发布，闭源权重）。Confidence: high [2] [3] [1]
- 客户端：iOS、Android、Web（muse.ai），可在 WhatsApp 里直接对话，AI 眼镜"即将推出"；约 2026-09-17 推出 macOS 原生 app，可操作本地应用；2026-09-29 推出 Muse for Small Business（美国、加拿大）。Confidence: high [1] [4] [5]
- Mac app 的动作经由云端 Secure VM 执行而非本地算力。Confidence: low [4]，single-source
- 同名/相关产品：Meta Muse Code（终端/CI 编程 agent，二进制 `muse`，2026-08-05 beta、08-31 GA），在用户本机的 OS 沙箱中运行，不是云 VM；E2B 提供 Muse Code 模板，但跑在用户自己的 E2B 账户里。Confidence: high [6] [7]
- obra/superpowers 插件支持的 "Muse" harness（`muse plugins install`）很可能是 Muse Code 而非消费级 Muse。Confidence: medium [8] [6]，unverified（由命令名推断）
- 其他同名产品均不运行每用户云 VM agent：Microsoft Muse（游戏世界模型，2025）、Skiv（原 muse.ai 视频托管，2026-03-03 改名）、museapp.com 画布笔记、Interaxon Muse 脑电头环、Meta Muse Image/Video 生成模型。Confidence: low [9]，部分条目来自背景知识未复核

### 运行时架构（what runs where）

- **客户端**：iOS/Android/Web 客户端"connect directly to your VM via a secure transport layer"（协议未披露）；密码在专用客户端 UI 输入，直送 authd，不进 runtime cell；支付用 Stripe Link 一次性卡号。Confidence: high [10] [1]
- **每用户 VM**：官方描述为"an isolated linux box with a browser and enough storage, CPU, and memory to do real work"，可编译代码、开发自定义 skill、运行并发 sub-agent 和 cron；VM 是数据的权威存储（system of record），持续备份。Confidence: high [10] [1]
- **VM 内两个安全域**：（a）runtime cell = systemd-nspawn 容器，内含 Hatch daemon（agent 主循环）、用户工作区与所有工具二进制；cell 内 root 映射为宿主上的非特权用户，独立 rootfs、veth 虚拟网卡、syscall 过滤（禁用 io_uring）、去掉 CAP_SYS_PTRACE/CAP_NET_ADMIN。（b）cell 外的宿主服务：Sentinel、hatch-safety（独立安全模型与分类器）、按 cgroup 识别的 privsep connector worker、hatch-authd（凭据库、代理 token）、Postgres（持久应用状态）、推理与遥测代理；两域间通过带 SO_PEERCRED 和 peer ACL 的 Unix socket 通信。Confidence: high [10] [11]
- **LLM 推理**在 VM 外，经"inference proxy"调用；推理所在硬件、机房未披露。Confidence: high [10]
- **浏览器**不在 VM 内：官方称 Chromium 浏览器"running behind a virtualization layer"，由独立 broker 持有 CDP（Chrome DevTools Protocol，Chrome 开发者工具协议）连接，browser sub-agent 只拿可访问性树快照、不能执行 JS。用户检查发现 VM 内只有 `browser-broker`，它租用另一台跑浏览器镜像的 VM，浏览器池共享。Confidence: medium [10] [12]（"另租 VM、共享池"为 single-source）
- **虚拟化技术**：Meta 未公开 hypervisor。两位测试者独立观测到 KVM，其中一位从 DMI（桌面管理接口）字符串认出 Cloud Hypervisor；有博主猜 Firecracker，但自承 Meta 未说明。两个调研 angle 结论不一（一个称"未披露"，一个称 Cloud Hypervisor）；我采信 Cloud Hypervisor/KVM 作为观测结果，因其有 DMI 实证。Confidence: medium [13] [14] [15] [16]
- **实测规格**（非官方）：2 vCPU，宿主 AMD EPYC 9D25（Zen 5 "Turin"），约 7.7 GiB 内存、无 swap、无 GPU，100 GB SSD 持久卷 + 7.5 GB 根 overlay，x86_64，Linux 内核 7.0。Confidence: medium [12] [14] [13] [17] [18]
- **OS 不一致**：Meta 博客称 runtime cell 的 rootfs 是"a full debian image"；MSL 工程 VP David Singleton 在 X 上称"a full Ubuntu linux image"，实测为 Ubuntu 24.04。可能 VM 是 Ubuntu、cell 镜像是 Debian（推断）。Confidence: medium [10] [19] [20] [17]
- **VM 内容**（一份 6.8 GB 文件系统导出）：`/home/hatch` 下有 SOUL.md、IDENTITY.md、USER.md、MEMORY.md、AGENTS.md 等 agent 文件和 sub-agent JSONL 轨迹；`/opt/hatch/skills` 约 68 个 skill；`/opt/hatch/runtime-cell` 有 nspawn 启动脚本；Postgres 存 384 维向量记忆；还附带 OpenAI Codex CLI（用其 bubblewrap 沙箱跑 ffmpeg）。用户在 cell 内有 root，可装包。Confidence: medium [11] [21]
- **无 GUI 桌面、无入站 SSH**（VM 在代理后、无公网 IP）；用户可通过 Tailscale connector 让 VM 加入 tailnet，或自建反向隧道。Confidence: medium [22] [21] [20]（"无 GUI"为推断）
- **云厂商未披露**；一次实测的出网 IP 属于 Cloudflare 段；运行在 CoreWeave/Nebius/AWS 的说法无依据（且 AWS Graviton 说法与实测 AMD EPYC 矛盾）。Confidence: medium [1] [12]，unverified

### VM 调度与生命周期

- **按用户常驻**：Meta 称 Muse"keeps working after people close the app"并支持 cron；未公布空闲关机、挂起或缩容至零策略。测试者观察到 memory balloon 空闲页回收，"an idle hatchling costs its ~2 GB working set rather than its 8 GB allocation"，并据此判断 VM 常驻在线。Confidence: medium [1] [13] [16]，常驻为推断
- **预启动 VM + 挂卷（疑似 warm pool）**：VM 启动时无用户身份，`hatch-prewarm` 先运行，再把用户持久卷（RV，Reliable Volume）"graft"到预启动的运行时上；无来源使用"warm pool"一词，Meta 未说明。Confidence: medium [13]，single-source，池化为推断
- **启动时间**：一次替换 VM 的观测——内核启动 0 s → cell default.target ~13 s → 挂卷 ~21 s → execution-ready ~39 s，发布期间用户约 40 s 不可用。Meta 无官方启动时间。Confidence: medium [13]，single-source
- **VM 可丢弃、磁盘持久**：新 harness 版本发布时拉起新 VM（"hatchling"）、挂上用户卷、销毁旧 VM，一晚观察到 3 次。Confidence: medium [13] [15]（xeno 引用 ad，独立性有限）
- **持久化内容**：100 GB 用户卷（LUKS2 加密的 btrfs，挂在 `/home/hatch`、`/var/lib/hatch`，含 Postgres 数据）跨 VM 替换保留；7.5 GB 根 overlay 用一次性密钥、随 VM 丢弃；agent 装的包记入卷上的 append-only ledger，在新 VM 上重放。Confidence: medium [13] [12]（ledger 为 single-source）
- **备份**：官方称"backed up continuously so you can restore it"；二进制字符串显示用 btrfs snapshot 实现；恢复粒度和保留期未公开。Confidence: medium（官方表述部分为 high）[10] [13]
- **会话跨 VM 替换存活**：各 harness 阶段在 Postgres 中有重启检查点和超时环境变量。Confidence: medium [13]，single-source
- **用户可见快照/fork/挂起功能**：无。Cloud Hypervisor 本身支持 snapshot/restore，Meta 工程师参与其开发，但 Muse 是否用 VM 级快照未知。Confidence: low [13] [23]，unverified
- **并发上限**（黑盒测试，非官方配额）：sub-agent 峰值并发 33–72；同时启动 40 个失败 1 个、80 个失败 5 个、120 个失败 87 个（Postgres 锁超时），错峰启动 80 个零失败。Confidence: medium [14]，single-source
- **文件可见性**：约 2026-09-24 起可下载整个文件系统的 zip（去除密钥）并按 `/` 浏览；Meta 称这是"your own computer in the cloud"的预期行为。Confidence: medium [24] [10]

### 租户模型与隔离、安全

- **一用户一台专属 VM**（非每会话/任务）；help center："Every Muse user's VM is isolated so no one else's agent can access it." sub-agent、cron、skill 都跑在同一台用户 VM 内。Confidence: high [1] [10] [25]
- **跨用户隔离边界**为 VM（hypervisor）；物理宿主是否多租户共享、用何种 hypervisor，Meta 均未说明。Confidence: medium [10]，unverified（多租户共享宿主为推断，2 vCPU 切片的实测支持这一推断 [12]）
- **出网控制**：Sentinel 是 connector 动作和所有出网的唯一审批者（"Nothing Muse does reaches the internet unless the Sentinel approves it"）；通过 forward proxy + user namespace + veth + eBPF（扩展伯克利包过滤器）cgroup 程序强制，在 L4/L7 检查主机名、解析 IP、端口、协议、方法、路径和解码后的 body，拦截 SSRF（服务端请求伪造）；Meta 新增的 LSM（Linux 安全模块）hook 做"tainted egress"污点跟踪，读过用户数据的进程不能自动放行；审批范围可为一次、会话、任务、限时或永久。Confidence: high [10] [1]
- **凭据**：hatch-authd 把 OAuth token 等凭据存在用户 VM 内（"not in centralized Meta infrastructure"），cell 内代码只见代理 token，Sentinel 在网络边界替换为真实凭据；邮件 connector 过滤 OTP、重置密码链接。Confidence: high [10] [25] [1]
- **Meta 员工访问**：当前 Secure VM 仅以运营政策限制员工访问，"does not prevent Meta from accessing data when necessary"。Confidence: high [10] [26]
- **Muse Confidential VM**（计划"later this year"，2026）：整台 VM 用仅用户持有的密钥加密，承诺"cryptographically and verifiably"阻止 Meta 访问；新闻称基于 TEE（可信执行环境）并有 Moxie Marlinspike 参与（转引 WIRED，未能直读）。截至 9 月底未上线，是否默认开启未知。Confidence: medium [1] [10] [26] [27]
- **数据删除**：可删除消息、文件或"Reset Muse"（不可撤销）；无公开保留期，未说明重置后备份留存多久；VM 数据不用于广告。Confidence: high [28] [25]
- **合规**：未找到 Muse 的 SOC 2 / ISO 27001 公开证明；有公开漏洞赏金，最高 $300,000。Confidence: medium [10]，"无合规证明"为推断
- **上线后事件**未涉及云 VM 边界：macOS 客户端隐藏调试偏好可被本地进程劫持听写流量、截获 token（Patrick Wardle 披露，约一天内修复）；Reuters 报道内测期 guardrail 被绕过等问题。Confidence: medium [29] [30]

### 定价与计算相关限额

- **套餐**：Free（有会刷新的用量上限）；Power $20/月，每周 500M Muse tokens；Maximum $100/月，每周 3B Muse tokens；按月自动续订。Confidence: high [31] [32] [33]
- **免费额度约每周 100M tokens**，Gizmodo 称出自 Zuckerberg 的 Threads 帖，Meta help center 未公布。Confidence: medium [34] [33]
- **按 token 计量，不按算力**：官方页面未列 vCPU 时长、VM 时长、并发 VM/任务、存储配额或分档 VM 规格；用尽只能升级或等刷新，无公开超额计费，据称不结转。Confidence: medium [31] [35]，基于"未提及"推断
- **付费档只差额度、不差功能或 VM 大小**（第三方称；另一来源称高档有更大 VM，未能核实）。Confidence: medium [33] [36]
- 每周额度似从注册日起算 7 天重置；注册即使免费也需绑卡；邀请码双方各得 1B tokens。Confidence: low [34] [32] [35]，多为 single-source
- Muse Spark 开发者 API 单独定价（$1.25/M 输入、$4.25/M 输出），未说明与消费级 Muse tokens 的关系。Confidence: medium [37]，single-source

## 缺口

- **Hypervisor 与云厂商**：Meta 未公开；Cloud Hypervisor 仅来自一位测试者的 DMI 字符串，云厂商只有一个 Cloudflare 出口 IP。可能改变"运行在什么之上"的结论。
- **生命周期细节**（预启动挂卷、40 s 启动、package ledger、btrfs 备份、会话检查点）几乎都出自同一篇实测博文 [13]；Meta 未确认，任何一次发布都可能改变。
- **空闲策略**：是否在空闲时挂起/缩容、cron 在 app 关闭时如何触发，均无官方说明；"常驻"为推断。
- **配额**：无官方 CPU/内存/磁盘规格、并发上限、最长运行时长，也不知各档 VM 规格是否不同。
- **未能直读的原始来源**：WIRED（Confidential VM、员工访问）、Tom's Hardware 正文、Singleton 与 Zuckerberg 的 X/Threads 帖、TechCrunch Mac app 原文、Phoronix；相关结论依赖转引。
- muse.ai 跳转到登录墙，未见任何公开 help/docs/changelog 以外的官方页面；未检索 Hacker News 讨论与招聘信息。

## 来源

[1] https://about.fb.com/news/2026/09/introducing-muse-personal-ai-agent/ — primary
[2] https://about.fb.com/news/2026/04/introducing-muse-spark-meta-superintelligence-labs/ — primary
[3] https://research.meta.ai/blog/introducing-muse-spark-1-3 — primary
[4] https://aiweekly.co/alerts/meta-ships-muse-for-mac-agent-now-acts-in-native-macos-apps — secondary
[5] https://about.fb.com/news/2026/09/introducing-muse-small-business/ — primary
[6] https://dev.meta.ai/docs/muse-code/ — primary
[7] https://docs.e2b.dev/agents/muse — primary
[8] https://raw.githubusercontent.com/obra/superpowers/main/README.md — primary
[9] https://www.getaiperks.com/en/ai/meta-muse-vs-muse-ai-skiv — secondary
[10] https://research.meta.ai/blog/security-and-safety-for-ai-agents-our-approach-with-muse — primary
[11] https://mouse.dev/blog/muse-runtime-export/ — secondary
[12] https://www.starkinsider.com/2026/09/meta-muse-specs-what-it-runs-on.html — secondary
[13] https://rohanadwankar.github.io/posts/sandbox2.html — secondary
[14] https://blog.cygankiewicz.com/en/meta-muse-black-box-testing/ — secondary
[15] https://xenospectrum.com/en/meta-muse-cloud-runtime-observed-resources/ — secondary
[16] https://dev.to/devopsdaily/meta-says-every-muse-user-gets-their-own-vm-2dmj — secondary
[17] https://www.tomshardware.com/pc-components/cpus/meta-muse-runs-agents-on-amd-epyc-turin-hosts-with-two-cores-and-8gb-of-memory-ai-agent-can-pass-terminal-commands-to-ubuntu-host-system — secondary
[18] https://runtimewire.com/article/meta-muse-linux-sandbox-ssh-security — secondary
[19] https://x.com/dps/status/2103161493722419334 — primary
[20] https://the-decoder.com/metas-muse-agent-gives-every-user-a-full-cloud-computer-running-ubuntu-linux/ — secondary
[21] https://securityboulevard.com/2026/10/metas-muse-agent-will-help-you-root-around-the-computer-it-gives-every-user/ — secondary
[22] https://tailscale.com/blog/meta-muse-ai-agent-tailscale — primary
[23] https://www.phoronix.com/news/Cloud-Hypervisor-53 — secondary
[24] https://aichief.com/news/meta-makes-the-muse-filesystem-even-more-accessible/ — secondary
[25] https://www.meta.com/help/artificial-intelligence/1047255454427887/ — primary
[26] https://tech.yahoo.com/ai/meta-ai/articles/building-owner-doesn-t-key-110745773.html — secondary
[27] https://www.startuphub.ai/ai-news/artificial-intelligence/2026/meta-s-private-muse-pitch-leaves-trust-gaps — secondary
[28] https://www.meta.com/help/artificial-intelligence/2225571704857152/ — primary
[29] https://www.infoq.com/news/2026/09/meta-muse-zeroday/ — secondary
[30] https://www.implicator.ai/meta-muse-ai-agent-internal-security-flaws/ — secondary
[31] https://www.meta.com/help/subscriptions/1021145227643680/ — primary
[32] https://techcrunch.com/2026/09/08/meta-debuts-its-muse-ai-agent-will-consumers-trust-it/ — secondary
[33] https://openclawdatabase.com/meta-muse/pricing/index.md — secondary
[34] https://gizmodo.com/metas-muse-let-me-waste-a-mind-boggling-amount-of-free-compute-on-nothing-in-particular-2000808945 — secondary
[35] https://www.layer3labs.io/guides/meta-muse-pricing — secondary
[36] https://techcrunch.com/2026/09/23/everything-new-coming-to-metas-ai-agent-muse/ — secondary
[37] https://dev.meta.ai/docs/pricing-rate-limits — primary
