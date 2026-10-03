# Report: TASK-138 调研：把 agent 的调研结果做成可交互、可视化的网页（开源工具）

## 结论与建议

以下是本文对发现的综合判断：**Markdown 报告照旧，旁边再放一个由 agent 写的自包含 `.html`，在本地打开。**
1. IndyDevDan（GitHub 账号 disler）没有发布做 cmux guide 的工具或 prompt。他让前沿 agent 按“分层递进”写一份 HTML 学习指南；产物公开在 disler/learning-cmux-with-agents，生成方法没有公开。
2. 他发布过同类生成器：Claude Code skill（技能）`planf3`（disler/planf3）和 `htmlspec` / `htmlvspec`（disler/pi-agent-observability），都让 agent 按模板写单个自包含 HTML。它们写的是开发实施计划，不是学习指南，用于调研报告要改模板。
3. 其他开源打包：nicobailon/visual-explainer（MIT，约 1.02 万 star，活跃）；html-explainer（默认先问一轮问题，不适合无人值守）。
4. 确定性兜底：`markmap --no-open --offline` 把 Markdown 转成离线可用的交互式思维导图；Quarto 加 `embed-resources: true` 输出单个自包含 HTML。
5. 接入方式（最小可行）：
   - researcher 发布 `Research/<同名>.md` 时，同目录再写 `Research/<同名>.html`。内容和 Markdown 报告相同，按层组织：结论、对比表、图表、可展开的细节。CSS、JS、SVG 全部内联，不依赖 CDN。
   - 起点用 disler 的 `htmlspec`：它已要求 CSS、JS 内联、不用 CDN，不需要 API key。把计划模板（Purpose、Relevant Files、Implementation Phases 等）换成上面的分层结构，输入改为报告路径（未测试）。visual-explainer 的页面依赖 CDN，只作版式参考。
   - 生成失败或不想花 token 时，改用 `npx -y markmap-cli --no-open --offline -o <同名>.mindmap.html <同名>.md`。
   - issue 上挂 `.md` 链接，再挂 `.html` 链接。GitHub 页面只显示 HTML 源码（未核实），所以打开方式是在用户的 docs clone 里 `git pull` 后，用浏览器 `open Research/<同名>.html`。
   - 想在浏览器里直接私密访问，GitHub Pages 需要 Enterprise Cloud；Pro 和 Team 发布的 Pages 是公开的，不能用。

## 对比表

本表是综合各发现的结果。“无人值守”指 agent 可以不经人工直接生成。

**候选**

| 候选 | 形态 | 输入 | 无人值守 | 完全离线自包含 | 许可与活跃度（2026-10-03） |
|---|---|---|---|---|---|
| Dan 的 cmux guide（产物，[disler/learning-cmux-with-agents](https://github.com/disler/learning-cmux-with-agents)） | 单页 HTML 学习指南 | agent 直接生成 | 是，但生成器没有公开 | 否，引用 10 张本地图片 | MIT，115 star，唯一一次提交在 2026-06-29 |
| [planf3](https://github.com/disler/planf3)（disler） | 单页 HTML 实施计划，每节一张 AI 图 | prompt + 代码库 | 是；没有 `OPENAI_API_KEY` 时图片位留空 | 是，只有内联 `<style>` | MIT，148 star，2026-06-21 推送 |
| [htmlspec / htmlvspec](https://github.com/disler/pi-agent-observability/tree/main/.claude/skills)（disler） | 单页 HTML 计划；Freeform 区可放交互或动画的内联 HTML/SVG | prompt + 代码库 | htmlspec 是；htmlvspec 缺 `OPENAI_API_KEY` 时停下问人 | htmlspec 是（无图、无 CDN） | MIT，145 star，2026-05-31 推送 |
| [visual-explainer](https://github.com/nicobailon/visual-explainer) | 单页 HTML 或幻灯片 | agent 直接生成；`--quick` 模式用 JSON spec 渲染 | 是（skill、Claude Code plugin） | 否，从 CDN 加载字体、Mermaid、Chart.js、three.js | MIT，10,216 star，2026-10-02 推送 |
| [handbook-visual-explainer](https://github.com/nikiforovall/claude-code-rules/blob/main/plugins/handbook-visual-explainer/README.md) | 同上，7 个命令 | agent 直接生成 | 是（Claude Code plugin） | 否，依赖 CDN | 2026-09-22 有提交 |
| [html-explainer](https://github.com/ZBQtesla/html-explainer) | 滚动页或幻灯片讲解页 | agent 直接生成 | 默认否，先做一轮问答 | 是，默认单文件、原生 JS | 原仓库 ds-vibe/html-explainer：MIT，15 star |
| [codebase-explainer](https://github.com/tpgiv1995/codebase-explainer) | 带动画的代码库讲解页 | agent 直接生成 | 是（skill） | 基本是，只依赖 Google Fonts | 0 star，许可有争议；只作 prompt 参考 |
| [markmap-cli](https://github.com/markmap/markmap) | 交互式思维导图 | Markdown | 是（CLI） | 是，加 `--offline` | MIT，约 1.31 万 star，2026-09-12 推送 |
| [Quarto](https://quarto.org/docs/output-formats/html-basics.html) | HTML 文档 | `.qmd` / Markdown | 是（CLI） | 是，加 `embed-resources: true` | 未核实 |
| [Observable Framework](https://observablehq.github.io/framework/) | 数据报告、仪表盘静态站点 | Markdown + 响应式 JS + 数据加载器 | 是（构建） | 未核实 | ISC，约 3.7k star；维护状况存疑 |
| [marimo](https://docs.marimo.io/guides/exporting/) | notebook 导出的 HTML | marimo `.py` notebook | 是（`marimo export html` / `html-wasm`） | WASM 版要通过 HTTP 访问 | 未核实 |
| [DeerFlow 2.0](https://github.com/bytedance/deer-flow) | 自带网页、图表、PPT 等 skill 的调研 agent | 由 agent 生成 | 是（`--print`、`--json`） | 未核实 | MIT，约 8.3 万 star，2026-10-03 有推送 |
| [STORM](https://github.com/stanford-oval/storm) | 类似 Wikipedia 的文字文章 | 网络检索 | 是 | 不适用，不产出可视化 | 未核实 |
| [OpenDeepResearch](https://github.com/sgauravm/OpenDeepResearch) | Markdown 或 PDF 报告，配 PNG 图表；可选生成 Plotly 交互图 | 网络检索 | 是 | 不适用 | 1 star，无许可 |

**做法**

| 做法 | 效果 | 成本 | 维护负担 | 无人值守可靠性 |
|---|---|---|---|---|
| 现成框架（Quarto、Observable Framework、marimo、静态站点、图表库） | 版式固定，可读性中等 | 要装 CLI 或构建工具链；需要运行代码时还要内核 | 中，工具链要升级 | 高，确定性渲染 |
| agent 直接写单页 HTML | 最丰富：动画、自定义布局 | 输出 token 多（未测量） | 低，只维护一份 prompt 或 skill | 中，可能出现版式错乱、图表数据编造（推断） |
| 思维导图（markmap） | 只能体现结构，没有图表 | 几乎为零 | 低 | 高 |

## 发现

类型：web。

**问题 2：IndyDevDan 的工具，以及他的 GitHub 有没有发布**
- 出处是视频 [SEE CMUX SOLVE Multi-Agent Orchestration](https://www.youtube.com/watch?v=WAFUMBLOjHo)（2026-07-06）。原话：“I have a state-of-the-art agent build out a comprehensive HTML file that I can use to visually understand, study, and master new tools”，并要求“incremental tier-based explanation”。他没有提到任何产品、skill 或工具名。高（3-0，上一轮核实，本轮未重核）。
- disler 没有发布生成 `guide/index.html` 的工具、prompt 或 skill。learning-cmux-with-agents 只有一个分支、一次提交（6eaacab，“🚀”，2026-06-29），之后没有推送。高（3-0）。[来源](https://github.com/disler/learning-cmux-with-agents)
- 仓库里也找不到生成方法：
  - `.claude/` 只有 cmux 编排用的 agents（build-be、build-fe、lead、plan、test）、commands（cmux-did-spawn、cmux-fresh、prime、spawn-fs-team）、skill `cmux`、settings.json 和 status line 脚本；唯一提到 guide 的是 prime.md，让 agent 去读它。
  - justfile 的 `guide` 只是在仓库根目录跑 `python3 -m http.server`。
  - `prompts/`（31 条练习加 PATTERNS-read-and-notify.md）和 `ai_docs/` 里没有生成 guide 的 prompt。

  高（3-0）。[来源](https://github.com/disler/learning-cmux-with-agents)
- `guide/index.html` 995 行，标题“Orchestrate Agents”，head 里只有 charset 和 viewport；没有任何注释或文字提到生成器。`guide/` 另有 cmux-guide.pdf 和 10 张图片（webp、svg）。CSS 里有“animated orchestration diagram”和 flip cards，注释（“COMPONENT: figure (ONE per section)”、“the ONE accent”）像是一套可复用的设计模板，但没有名字。高（3-0）。[来源](https://github.com/disler/learning-cmux-with-agents/blob/main/guide/index.html)
- disler 的 55 个公开仓库里只有这一个 learning-* 仓库，没有后续的 learning-<tool>-with-agents。之后推送的仓库（ten-levels-of-jev、self-compact-pi-agent、fusion-harness、fixing-smartass-opus-5、inkwell-agent-sandboxes-and-software-factory、super-simple-software-factory、pi-vs-claude-code）里的 HTML 是应用界面、可视化器或 planf3 式的计划，不是调研转指南的生成器。高（3-0）。[来源](https://github.com/disler?tab=repositories)
- 他发布过的同类生成器：
  - `planf3`：Claude Code skill，用法 `/planf3 <user prompt> [questionable]`。agent 读 prompt、代码库（以及 AI_DOCS/、APP_DOCS/），按 `.claude/skills/planf3/SKILL.md` 的模板把单个自包含 HTML 计划写进 `specs/`，每节用 gpt-image-2 生成一张图。prompt 为空才会停；没有 `OPENAI_API_KEY` 时计划照写，图片位留空。它和 cmux guide 的模板结构不同，不是做 guide 的那个工具。高（3-0；“最接近”是判断，2-1）。[来源](https://github.com/disler/planf3)
  - `htmlspec` / `htmlvspec`（在 pi-agent-observability 的 `.claude/skills`）：输入 USER_PROMPT 加代码库，没有 prompt 就停。htmlspec 写纯文字计划到 `specs/htmlspec-<slug>.html`，没有图片，CSS、JS 全部内联，除非确有必要不用 CDN。htmlvspec 必用 gpt-image-2 生成图片（封面加每节一张，最多 10 张），缺 `OPENAI_API_KEY` 就停下问人，并假定装在 `~/.claude/skills/htmlvspec`。两者都有 Freeform 区，可放切换、动画 SVG 流程、对比矩阵、决策树等内联交互内容。高（3-0；与 cmux guide 的相似度 2-1，不能说明 guide 的做法）。[来源](https://github.com/disler/pi-agent-observability/tree/main/.claude/skills)
- 复用方式（推断，未测试）：把 htmlspec（无需 API key，可无人值守）或 planf3 复制到 `~/.claude/skills`，把计划模板换成指南分节，输入改为“读 `$1` 处的报告”。直接把报告路径当 USER_PROMPT 传进去，内容会被塞进开发计划的结构。中。

**问题 1：候选工具**
- visual-explainer：agent skill 兼 Claude Code plugin，生成单文件 HTML 页面或幻灯片（`--slides`）。安装：`/plugin marketplace add nicobailon/visual-explainer`，再 `/plugin install visual-explainer@visual-explainer-marketplace`；命令为 `/visual-explainer:<cmd>`：generate-web-diagram、diff-review、plan-review、generate-slides、project-recap、fact-check、generate-visual-plan。另有 Codex CLI、Cursor、OpenCode、Pi 和 MCP 版本。输出到 `~/.agent/diagrams/`（用 `VISUAL_EXPLAINER_OUTPUT_DIR` 改）。页面引用 Google Fonts，并视内容从 CDN 加载 Mermaid、Chart.js 或 three.js，需要联网才能完整显示。高（3-0）。[来源](https://github.com/nicobailon/visual-explainer)
- handbook-visual-explainer：visual-explainer 的 Claude Code plugin（插件）分叉，提供 web-diagram、visual-plan、slides、diff-review、plan-review、project-recap、fact-check 七个命令，另有一个遇到表格数据时自动触发的 skill；都输出单文件 HTML，用 Mermaid 画图。高（3-0）。[来源](https://github.com/nikiforovall/claude-code-rules/blob/main/plugins/handbook-visual-explainer/README.md)
- html-explainer：Claude Skill，生成滚动页或幻灯片形式的交互讲解页，默认单个自包含 `.html`、原生 JS。默认先做一轮编号问答（选项带字母）再写 HTML；prompt 里已回答或明确交给 agent 决定的问题可跳过。中（3-0，采用度低）。[来源](https://github.com/ZBQtesla/html-explainer)
- codebase-explainer：SKILL.md 规定输出单个 HTML 讲解页，带滚动分节、进度、数据流动画、术语提示。它面向代码库，写死了公司品牌，只能当 prompt 参考。中（3-0，单一来源，个人项目）。[来源](https://github.com/tpgiv1995/codebase-explainer)
- markmap-cli：把 Markdown 的标题和列表结构转成交互式思维导图 HTML。实测 `npx -y markmap-cli --no-open --offline -o out.html t.md`（v0.18.12）：退出码 0，生成约 344 KB 文件，没有外部引用；不加 `--offline` 要从 CDN 加载 d3 和 markmap。默认会打开浏览器，无人值守时要加 `--no-open`。高（3-0）。[来源](https://markmap.js.org/docs/packages--markmap-cli)、[仓库](https://github.com/markmap/markmap)
- Quarto：`quarto render report.qmd --to html`；在 `format: html` 下设 `embed-resources: true`，脚本、CSS、图片都以 data URI 嵌入单个文件。前提是装了 Quarto CLI；含可执行代码时还要 Jupyter 或 R 内核。高（3-0）。[来源](https://quarto.org/docs/output-formats/html-basics.html)
- Observable Framework：开源静态站点生成器，做数据应用、仪表盘和报告。输入是带响应式 JS 图表和控件的 Markdown 页面，加上构建时生成数据快照的数据加载器（任何语言）。2026-03 有社区讨论问它是否还在维护，最近一次更新约在 2026-05。中（3-0）。[来源](https://observablehq.github.io/framework/)
- marimo：CLI 可无人值守导出，`marimo export html` 得到静态快照，`marimo export html-wasm` 得到在浏览器里跑 Python 的交互 HTML；WASM 版输出一个目录，要通过 HTTP 访问。输入是 marimo `.py` notebook，不是 Markdown。中（3-0）。[来源](https://docs.marimo.io/guides/exporting/)
- 调研 agent：
  - DeerFlow 2.0 自带 frontend-design（生成 HTML 页面）、chart-visualization、ppt-generation、deep-research、vercel-deploy 等 skill，有 `deerflow --print` / `--json` 无头模式。高（3-0）。[来源](https://github.com/bytedance/deer-flow)
  - STORM 只产出带引用的文字文章，演示界面是一个简单的 Streamlit 查看器。高（3-0）。[来源](https://github.com/stanford-oval/storm)
  - OpenDeepResearch 默认让 LLM 填一个类型化的图表参数，再由 matplotlib 画成 PNG，作者称接近 100% 可靠；可选的 code 模式让 LLM 写 Plotly 代码，得到交互图，但作者说在小模型上经常失败。中（3-0，单一来源，1 star）。[来源](https://github.com/sgauravm/OpenDeepResearch)

**问题 3：可靠性的取舍**
- 本文综合：确定性渲染器（markmap、Quarto、填参数的图表渲染器）可靠，但形式有限；agent 自由写 HTML 或代码，效果更丰富，但可靠性低、输出 token 多。直接证据只有 OpenDeepResearch 两种模式的对比；visual-explainer 的 `--quick` JSON spec 模式、disler 的固定模板 skill 体现的是同一取舍：用模板约束 agent。中，推断。

**问题 4：私有仓库里的 HTML 怎么打开**
- GitHub Pages 要做访问控制，需要 GitHub Enterprise Cloud，并且是组织名下的项目站点，只有对源仓库有读权限的人能看。Pro 和 Team 可以从私有仓库发布 Pages，但站点在互联网上公开。高（3-0）。[来源](https://docs.github.com/en/enterprise-cloud@latest/pages/getting-started-with-github-pages/changing-the-visibility-of-your-github-pages-site)
- 本文综合：我们的 docs 仓库是私有的，用户本机有 docs clone。所以零成本、私密的打开方式是本地打开文件；这要求 HTML 内联所有资源，不能像 Dan 的 guide 那样依赖本地图片或 http.server。

## 对已知说法的更正

- “IndyDevDan 用自己的工具把调研结果做成网页”：没有公开这样一个工具。他是让 agent 按 prompt 写一份 HTML 指南；指南公开了（MIT），生成用的 prompt 或 skill 没有公开。他公开的 planf3、htmlspec、htmlvspec 是同一路数的 HTML 生成 skill，但面向开发计划。
- “里面有动画、图表”：动画部分成立，CSS 里有动画编排图和 flip cards；没有核实到图表。
- 上一版本报告说 guide 948 行、6 张本地图片：实际 995 行、10 张图片（webp、svg）。

## 缺口

- disler 的 gist、agenticengineer.com、indydevdan.com、视频简介没有确认查过；付费课程里可能有生成 prompt。“没有发布”只覆盖他的公开 GitHub 仓库。
- 待解：guide 的 CSS 模板是不是一个没公开的 disler skill，与 htmlvspec / planf3 同源。
- 改造后的 htmlspec 或 visual-explainer 把长篇 Markdown 报告转成分层指南的效果和 token 成本没有测试。
- 以下 GitHub 行为都没有通过核实：GitHub 网页显示 `.html` 源码、raw.githubusercontent.com 以 text/plain 返回、htmlpreview.github.io 打不开私有仓库。Cloudflare Pages 配 Cloudflare Access、Netlify 密码保护、secret gist 这几种私密托管方式没有调研。
- 问题 1 仍有未核实的候选：Mermaid（包括 GitHub 在 Markdown 里是否原生渲染 Mermaid）、Evidence.dev、Slidev、reveal.js、Streamlit / Gradio、MkDocs / Docusaurus / Astro、Jupyter、Obsidian 知识图谱、D3 / Vega-Lite / ECharts、GPT-Researcher 的输出格式、NotebookLM 的开源替代。两轮 workflow 共抽出 168 条说法，各只核实了重要性最高的 25 条，其余没有进入本文。
- Quarto、STORM 的许可和活跃度没有核实。
- 没有测量任何做法的 token 成本、生成时间和质量；问题 3 的对比是定性的。
- 已推翻：“visual-explainer 平常不依赖浏览器以外的任何东西，用手绘内联 SVG”（0-3）；“Co-STORM 的思维导图是它唯一的可视化输出”（0-3）；marimo“静态导出完全不能交互，WASM 导出保留控件交互”这一细节说法（1-2），marimo 的交互细节要谨慎看待。
- 待解：agent 写 HTML 在大量报告上的稳定性如何（版式错乱、图表数据编造）；要求内联全部 JS 和 CSS 时，文件大小和 token 成本会增加多少。
