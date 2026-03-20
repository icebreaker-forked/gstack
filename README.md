# gstack

我是 [Garry Tan](https://x.com/garrytan)，Y Combinator 的总裁兼 CEO。在 YC 这些年，我和成千上万家创业公司合作过，其中不少后来成长为 Coinbase、Instacart、Rippling 这样市值数百亿美元的公司。更早之前，我参与过 Palantir 的早期设计，也联合创办过 Posterous，还在 2013 年做了 YC 的内部社交网络 Bookface。

而现在，我正处在一个完全不同的新阶段。

过去 60 天里，我写了 **60 多万行生产代码**，其中约 35% 是测试；即使在继续履行 YC CEO 工作的前提下，我仍然能做到 **每天 1 万到 2 万行可用代码**。最近一次 `/retro` 的 7 天统计，跨 3 个项目合计 **140,751 行新增、362 次提交、约 11.5 万净新增代码**。模型每周都在显著变强，我们已经站在一个真实拐点上：以前需要二十人团队才能完成的工作，现在一个人就可以完成。

**gstack 就是我这样工作的方式。**

它是我的开源“软件工厂”。它把 Claude Code 变成一支真正可管理的虚拟工程团队：有重新定义问题的 CEO、有锁架构的工程经理、有识别 AI slop 的设计师、有专抓生产事故的偏执 reviewer、有会打开真实浏览器点完整条流程的 QA 负责人，还有负责测试、推送和发 PR 的发布工程师。

15 个专业角色，加上 6 个增强工具。全部通过 slash command 使用。全部是 Markdown。**全部免费，MIT 协议，立刻可用。**

我正在把它作为 2026 年 3 月这个时间点上，对 agentic software workflow 的一场公开实验。我把它开源出来，是因为我希望更多人一起把这条路走通。

Fork 它，改进它，变成你自己的工作流。

## 适合谁

- **创始人和 CEO**：尤其是仍然亲自下场写东西的技术创始人。它让你用一人的规模，像二十人团队那样推进。
- **第一次认真使用 Claude Code 的人**：比起空白输入框，结构化角色更容易上手。
- **Tech Lead / Staff Engineer**：把评审、测试、QA 和发布自动化带进每个 PR。

## 10 分钟上手

1. 安装 gstack
2. 运行 `/office-hours`，先说清楚你要做什么，再决定写什么
3. 用 `/plan-ceo-review` 审需求
4. 用 `/review` 审当前分支
5. 用 `/qa` 跑一遍 staging
6. 到这里你基本就能判断它是否适合你

只要你的仓库已经具备基本测试环境，通常 5 分钟内就能跑出第一次有价值的结果。

## 安装

**依赖：** [Claude Code](https://docs.anthropic.com/en/docs/claude-code)、[Git](https://git-scm.com/)、[Bun](https://bun.sh/) v1.0+

### 1. 安装到本机

打开 Claude Code，直接粘贴下面这段，让 Claude 代你完成安装：

> Install gstack: run **`git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && cd ~/.claude/skills/gstack && ./setup`** then add a "gstack" section to CLAUDE.md that says to use the /browse skill from gstack for all web browsing, never use mcp__claude-in-chrome__* tools, and lists the available skills: /office-hours, /plan-ceo-review, /plan-eng-review, /plan-design-review, /design-consultation, /review, /ship, /browse, /qa, /qa-only, /design-review, /setup-browser-cookies, /retro, /investigate, /document-release, /codex, /careful, /freeze, /guard, /unfreeze, /gstack-upgrade. Then ask the user if they also want to add gstack to the current project so teammates get it.

### 2. 安装到当前仓库（可选）

如果你希望团队成员克隆仓库后也能直接使用 gstack，可将其 vendoring 到项目里：

> Add gstack to this project: run **`cp -Rf ~/.claude/skills/gstack .claude/skills/gstack && rm -rf .claude/skills/gstack/.git && cd .claude/skills/gstack && ./setup`** then add a "gstack" section to this project's CLAUDE.md that says to use the /browse skill from gstack for all web browsing, never use mcp__claude-in-chrome__* tools, lists the available skills: /office-hours, /plan-ceo-review, /plan-eng-review, /plan-design-review, /design-consultation, /review, /ship, /browse, /qa, /qa-only, /design-review, /setup-browser-cookies, /retro, /investigate, /document-release, /codex, /careful, /freeze, /guard, /unfreeze, /gstack-upgrade, and tells Claude that if gstack skills aren't working, run `cd .claude/skills/gstack && ./setup` to build the binary and register skills.

仓库里保存的是真实文件，而不是子模块，因此 `git clone` 后即可工作。所有东西都在 `.claude/` 下，不会污染你的 `PATH`，也不会常驻后台。

### Codex、Gemini CLI、Cursor

gstack 适用于支持 [SKILL.md 标准](https://github.com/anthropics/claude-code) 的代理。技能位于 `.agents/skills/`，会被自动发现。

```bash
git clone https://github.com/garrytan/gstack.git ~/.codex/skills/gstack
cd ~/.codex/skills/gstack && ./setup --host codex
```

也可以让安装脚本自动检测本机可用代理：

```bash
git clone https://github.com/garrytan/gstack.git ~/gstack
cd ~/gstack && ./setup --host auto
```

这样会根据环境将技能安装到 `~/.claude/skills/gstack` 和/或 `~/.codex/skills/gstack`。全部 21 个技能都能在支持的代理上工作。`careful`、`freeze`、`guard` 这类依赖 hook 的安全技能，在非 Claude 宿主上会退化为内联安全提醒。

## 它是怎么工作的

下面是一个典型流程：

```text
你：    我想做一个给日历生成每日简报的应用。
你：    /office-hours
代理：  先不急着写代码，先追问具体痛点和真实场景。

你：    我有多个 Google Calendar，信息总过期，地点常错，准备工作很慢。
代理：  你说的是“每日简报应用”，但你真正描述的是“个人幕僚 AI”。
        接着它会：
        - 抽取你没明说但已经隐含的能力需求
        - 质疑前提，逼你缩小或重构问题
        - 给出 2-3 套实现路线和投入评估
        - 推荐最窄、最快能上线验证的一条路
        - 写出设计文档，供后续技能直接消费

你：    /plan-ceo-review
代理：  从产品视角重新审题。

你：    /plan-eng-review
代理：  给出数据流、状态机、失败路径和测试矩阵。

你：    Approve plan. Exit plan mode.
代理：  开始写代码。

你：    /review
代理：  自动修正明显问题，并对高风险问题征求确认。

你：    /qa https://staging.myapp.com
代理：  打开真实浏览器点流程，发现并修复 bug。

你：    /ship
代理：  跑测试、推送并创建 PR。
```

你说的是“功能”，代理理解的是“真实问题”。这不是副驾驶，而是一支有分工、有节奏、有交接的团队。

## 一次完整 sprint

gstack 不是一组零散工具，而是一条完整流程：

**思考 → 规划 → 开发 → 审查 → 测试 → 发布 → 复盘**

每个技能都会把产出交给下一个技能。`/office-hours` 生成的设计文档会被 `/plan-ceo-review` 和 `/plan-eng-review` 继续消费；`/review` 的发现会被 `/ship` 校验；`/qa` 会根据已有计划和实现补齐回归测试。

一个人完成一个 feature 的完整 sprint，通常只要 30 分钟左右。真正改变规模的是：你可以并行跑 10 到 15 个这样的 sprint。

## 核心技能

| 技能 | 扮演角色 | 作用 |
|------|----------|------|
| `/office-hours` | YC Office Hours | 用 6 个高压问题逼你把产品说清楚，重构需求，并形成后续计划输入。 |
| `/plan-ceo-review` | CEO / Founder | 从更高层级重新定义问题，找出真正值得做的版本。 |
| `/plan-eng-review` | Eng Manager | 固化架构、数据流、边界条件和测试方案。 |
| `/plan-design-review` | Senior Designer | 按维度打分，指出设计距离“10 分”还缺什么。 |
| `/design-consultation` | Design Partner | 从零建立设计系统，研究竞品并产出设计方向。 |
| `/review` | Staff Engineer | 找出 CI 抓不到但线上会出事的结构性问题。 |
| `/investigate` | Debugger | 根因导向的调试流程；没有调查就没有修复。 |
| `/design-review` | Designer Who Codes | 审视觉质量并直接动手修。 |
| `/qa` | QA Lead | 用真实浏览器测试、修复、补回归。 |
| `/qa-only` | QA Reporter | 只做 QA 报告，不修改代码。 |
| `/ship` | Release Engineer | 同步主分支、跑测试、推送并创建 PR。 |
| `/document-release` | Technical Writer | 根据实际改动自动更新 README、ARCHITECTURE、CLAUDE 等文档。 |
| `/retro` | Eng Manager | 周复盘、交付节奏、测试健康度与成长建议。 |
| `/browse` | QA Engineer | 为代理提供“眼睛”，用真实 Chromium 读页面、点页面、截图。 |
| `/setup-browser-cookies` | Session Manager | 从 Chrome/Arc/Brave/Edge 导入登录态 cookie。 |

### 增强工具

| 技能 | 作用 |
|------|------|
| `/codex` | 让 OpenAI Codex CLI 做第二意见评审，支持审查、对抗性挑战和开放咨询。 |
| `/careful` | 在破坏性命令前发出警告。 |
| `/freeze` | 将编辑范围锁定在某个目录。 |
| `/guard` | 同时开启 `/careful` 与 `/freeze`。 |
| `/unfreeze` | 解除 `/freeze` 限制。 |
| `/gstack-upgrade` | 升级 gstack 并展示变化。 |

更详细的理念、案例和工作方式见 [docs/skills.md](docs/skills.md)。

## 为什么它重要

- **`/office-hours` 会在你写代码前重构问题。** 它不是替你扩写需求，而是判断你真正要解决的痛点。
- **设计是系统中心，而不是最后补丁。** `/design-consultation`、`/design-review` 和 `/plan-eng-review` 会把设计选择贯穿到开发全链路。
- **`/qa` 是能力跃迁点。** AI 真正“看到”页面之后，可以跑完整交互、修复 bug、补回归，从而支持更多并行 worker。
- **路由式 review。** 不是每次都让所有角色介入，而是按改动类型选择合适的评审路径。
- **测试优先。** `/ship` 能在没有测试框架时帮你补基础；`/qa` 的每次修复都应伴随回归测试。
- **文档持续对齐。** `/document-release` 会把 README、ARCHITECTURE、CONTRIBUTING、CLAUDE、TODOS 等同步到真实代码状态。
- **卡住时支持浏览器交接。** 遇到验证码、MFA 或复杂授权时，可将浏览器转交给人类完成，再无缝还给代理继续。
- **跨模型复核。** `/review` 加 `/codex` 可以形成不同模型的交叉视角。
- **按需安全护栏。** `/careful`、`/freeze`、`/guard` 可以在涉及生产环境或危险命令时强制降风险。

## 并行跑 10 到 15 个 sprint

单个 sprint 已经很强，并行 sprint 才是规模化拐点。

[Conductor](https://conductor.build) 可以并行运行多个 Claude Code 会话，每个会话都在独立工作区中。你可以同时让一个会话跑 `/office-hours`，另一个审 PR，第三个开发功能，第四个在 staging 跑 `/qa`，其余会话处理别的分支和任务。

并行之所以可控，关键不是代理数量，而是流程结构。没有流程，10 个代理就是 10 个混乱源；有了“思考、规划、开发、审查、测试、发布”的节奏，它们才像真正的团队。

## 这是一扇窗口期

gstack 是 **免费、MIT 协议、开源、现在就能用** 的。我把自己的开发方法完整公开，是因为我相信：未来优势不只属于会用模型的人，而属于会把模型组织成系统的人。

同样是 agent，结果会非常不同。gstack 的价值不只是“快”，而是它提供了结构化角色、评审门槛和交付节奏，避免把高速开发变成高速失控。

模型正在快速进步。真正提前建立工作方法论的人，会拥有明显优势。

15 个专业角色，6 个增强工具，全部用 slash command 驱动，全部用 Markdown 组织，全部免费。项目地址：**[github.com/garrytan/gstack](https://github.com/garrytan/gstack)**。

## 文档索引

| 文档 | 内容 |
|------|------|
| [Skill Deep Dives](docs/skills.md) | 每个技能的理念、案例与工作方式 |
| [Architecture](ARCHITECTURE.md) | 系统设计与内部实现 |
| [Browser Reference](BROWSER.md) | `/browse` 的完整命令参考 |
| [Contributing](CONTRIBUTING.md) | 开发环境、测试与贡献方式 |
| [Changelog](CHANGELOG.md) | 各版本新增内容 |

## 隐私与遥测

gstack 支持可配置的遥测模式，用于了解哪些技能被使用、运行时长和崩溃情况，以帮助改进产品。不会上传代码内容、仓库路径或文件内容。你可以随时通过 `gstack-config set telemetry off` 关闭。
