# gstack 开发说明

## 常用命令

```bash
bun install              # 安装依赖
bun test                 # 运行免费测试（browse + snapshot + skill 校验）
bun run test:evals       # 运行付费评测：LLM judge + E2E（按 diff 选择，单次最高约 $4）
bun run test:evals:all   # 无视 diff，运行全部付费评测
bun run test:e2e         # 仅运行 E2E（按 diff 选择，单次最高约 $3.85）
bun run test:e2e:all     # 无视 diff，运行全部 E2E
bun run eval:select      # 查看基于当前 diff 会触发哪些测试
bun run dev <cmd>        # 以开发模式运行 CLI，例如：bun run dev goto https://example.com
bun run build            # 生成文档并编译二进制
bun run gen:skill-docs   # 由模板重新生成 SKILL.md
bun run skill:check      # 技能健康检查面板
bun run dev:skill        # 监听模式：变更后自动重新生成并校验
bun run eval:list        # 列出 ~/.gstack-dev/evals/ 下的评测记录
bun run eval:compare     # 对比两次评测结果（默认取最近两次）
bun run eval:summary     # 汇总所有评测统计
```

`test:evals` 需要 `ANTHROPIC_API_KEY`。Codex 的 E2E 测试（`test/codex-e2e.test.ts`）直接使用 `~/.codex/` 中的认证，不需要额外设置 `OPENAI_API_KEY`。E2E 测试会实时流式输出进度，结果持久化到 `~/.gstack-dev/evals/`，并自动与上一次运行做比较。

**按 diff 选择测试：** `test:evals` 和 `test:e2e` 会基于与基线分支的 `git diff` 自动挑选测试。每个测试通过 `test/helpers/touchfiles.ts` 声明依赖文件。若修改了全局 touchfiles（如 `session-runner`、`eval-store`、`llm-judge`、`gen-skill-docs`），会触发全部测试。可用 `EVALS_ALL=1` 或 `:all` 脚本强制全量执行；`eval:select` 可用于预览。

## 测试要求

```bash
bun test             # 每次提交前运行，免费，通常 <2 秒
bun run test:evals   # 发布前运行，付费，按 diff 选择
```

`bun test` 会运行技能校验、`gen-skill-docs` 质量检查和 browse 集成测试。`bun run test:evals` 会运行 LLM 评测与基于 `claude -p` 的 E2E 测试。创建 PR 前，这两类测试都应通过。

## 项目结构

```text
gstack/
├── browse/               # 基于 Playwright 的无头浏览器 CLI
│   ├── src/              # CLI、服务端与命令实现
│   │   ├── commands.ts   # 命令注册表，单一事实来源
│   │   └── snapshot.ts   # SNAPSHOT_FLAGS 元数据
│   ├── test/             # 集成测试与夹具
│   └── dist/             # 编译产物
├── scripts/              # 构建与开发工具
│   ├── gen-skill-docs.ts # 模板转 SKILL.md
│   ├── skill-check.ts    # 健康检查面板
│   └── dev-skill.ts      # 监听模式
├── test/                 # 技能校验与评测
│   ├── helpers/          # skill-parser、session-runner、llm-judge、eval-store
│   ├── fixtures/         # 真值数据、缺陷夹具、评测基线
│   ├── skill-validation.test.ts
│   ├── gen-skill-docs.test.ts
│   ├── skill-llm-eval.test.ts
│   └── skill-e2e.test.ts
├── qa-only/              # /qa-only 技能
├── plan-design-review/   # /plan-design-review 技能
├── design-review/        # /design-review 技能
├── ship/                 # 发布工作流技能
├── review/               # PR 评审技能
├── plan-ceo-review/      # /plan-ceo-review 技能
├── plan-eng-review/      # /plan-eng-review 技能
├── office-hours/         # /office-hours 技能
├── investigate/          # /investigate 技能
├── retro/                # 复盘技能
├── document-release/     # /document-release 技能
├── setup                 # 一次性安装脚本：构建二进制并注册链接
├── SKILL.md              # 由 SKILL.md.tmpl 生成，不要直接改
├── SKILL.md.tmpl         # 模板文件：修改这里，然后重新生成
└── package.json          # browse 的构建脚本
```

## SKILL.md 工作流

`SKILL.md` 文件是由 `.tmpl` 模板**生成**的。更新文档时：

1. 修改对应的 `.tmpl` 文件，例如 `SKILL.md.tmpl` 或 `browse/SKILL.md.tmpl`
2. 运行 `bun run gen:skill-docs`，或直接运行会自动执行该步骤的 `bun run build`
3. 同时提交 `.tmpl` 和生成后的 `.md`

新增 browse 命令时，请修改 `browse/src/commands.ts` 并重新构建。新增 snapshot 标志位时，请修改 `browse/src/snapshot.ts` 中的 `SNAPSHOT_FLAGS` 并重新构建。

## 与平台无关的设计原则

技能绝不能写死框架专属命令、文件模式或目录结构。正确流程是：

1. 先读取 `CLAUDE.md` 中的项目配置，例如测试命令、评测命令等
2. 如果缺失，则通过 `AskUserQuestion` 向用户确认，或让 gstack 自行在仓库中搜索
3. 将结果写回 `CLAUDE.md`，避免下次重复询问

这条规则适用于测试、评测、部署以及所有项目私有行为。配置归项目所有，gstack 只读取。

## 编写 SKILL 模板

`SKILL.md.tmpl` 是给 Claude 读取的**提示模板**，不是 Bash 脚本。每个 Bash 代码块都在独立 shell 中执行，变量不会跨代码块保留。

规则：

- 逻辑与状态优先用自然语言描述，不要依赖 shell 变量跨块传递。
- 不要写死分支名。通过 `gh pr view` 或 `gh repo view` 动态检测 `main`、`master` 等。面向 PR 的技能优先使用 `{{BASE_BRANCH_DETECT}}`。
- 保持 Bash 代码块自包含。如果某个块依赖前文上下文，就在块前的说明文字中重新表述。
- 条件判断尽量写成英文步骤说明，而不是在 Bash 里堆复杂 `if/elif/else`。

## 浏览器交互

凡是需要浏览器操作的场景，例如 QA、dogfooding、cookie 导入，都应使用 `/browse` 技能，或直接通过 `$B <command>` 调用 browse 二进制。不要使用 `mcp__claude-in-chrome__*` 工具；它们又慢又不稳定，也不是本项目的标准做法。

## Vendored symlink 注意事项

开发 gstack 时，`.claude/skills/gstack` 可能是一个回指当前工作目录的符号链接。这意味着技能变更会**立刻生效**。它适合快速迭代，但在大改期间也意味着半成品可能影响别的 Claude Code 会话。

每个会话至少检查一次：

```bash
ls -la .claude/skills/gstack
```

如果它是指向当前工作目录的软链接，需要注意：

- 模板改动加上 `bun run gen:skill-docs` 会立刻影响所有 gstack 调用
- 对 `SKILL.md.tmpl` 的破坏性修改可能同时影响其他会话
- 大规模重构时，可以先移除该软链接，让系统回退到全局安装目录 `~/.claude/skills/gstack/`

对 plan review 而言，如果改动涉及技能模板或 `gen-skill-docs` 流水线，应考虑是否先在隔离环境验证，再推广到活动技能。

## 提交风格

**始终做可二分的提交。** 每个 commit 只包含一个逻辑变更。若同时做了重命名、重写和新增测试，应拆成多个独立提交，以便理解和回滚。

好的拆分示例：

- 重命名/移动 与 行为变更分开
- 测试基础设施 与 具体测试实现分开
- 模板变更 与 生成文件更新分开
- 机械性重构 与 新功能分开

当用户说 “bisect commit” 或 “bisect and push” 时，应将已暂存/未暂存改动拆成逻辑清晰的多个提交，再推送。

## CHANGELOG 写法

`CHANGELOG.md` 面向**用户**，不是贡献者。应写成产品发布说明：

- 先写用户现在**能做什么**，而不是实现细节
- 用自然语言表达，“你现在可以……” 比 “重构了……” 更合适
- 不要提 `TODOS.md`、内部跟踪、评测基础设施或只对贡献者有意义的内容
- 贡献者向的改动应单独放在文末 “For contributors” 小节
- 每一条都应让读者觉得“这个值得试一下”

## AI 压缩后的工作量估算

估算工作量时，同时给出人工团队时间与 CC+gstack 时间：

| 任务类型 | 人工团队 | CC+gstack | 压缩倍数 |
|-----------|-----------|-----------|-----------|
| 脚手架/样板代码 | 2 天 | 15 分钟 | ~100x |
| 写测试 | 1 天 | 15 分钟 | ~50x |
| 功能开发 | 1 周 | 30 分钟 | ~30x |
| 修 bug + 回归测试 | 4 小时 | 15 分钟 | ~20x |
| 架构/设计 | 2 天 | 4 小时 | ~5x |
| 调研/探索 | 1 天 | 3 小时 | ~3x |

完整性很便宜。只要问题还是“湖”，而不是“海”，就不要为了省几分钟推荐投机取巧的方案。

## 本地计划文档

贡献者可以把长期规划和设计文档放在 `~/.gstack-dev/plans/`。这里的文件仅保存在本机，不会提交到仓库。审阅 `TODOS.md` 时，可以顺便看看 `plans/` 里是否有已经成熟、适合升级成 TODO 或直接实现的方案。

## E2E 评测失败归因协议

当 `/ship` 或其他流程中的 E2E 评测失败时，**不要在没有证据的情况下说“与本次改动无关”**。这类系统存在大量隐性耦合：前言文本变化会影响代理行为，新的 helper 会改变时序，重新生成的 `SKILL.md` 会改变提示上下文。

在将失败归因为“原本就有”之前，必须做到：

1. 在 `main` 或基线分支上运行同样的评测，并证明那里也失败
2. 如果 `main` 通过、当前分支失败，那就是你的改动引起的，必须追查
3. 如果无法在 `main` 上验证，就明确写成“未验证，可能相关也可能无关”，并在 PR 描述里标为风险

没有证据就说“原本就坏”是偷懒，不成立。

## 部署到当前生效的技能目录

当前生效的技能目录通常位于 `~/.claude/skills/gstack/`。完成修改后：

1. 推送你的分支
2. 在技能目录中拉取并重置到目标分支：
   `cd ~/.claude/skills/gstack && git fetch origin && git reset --hard origin/main`
3. 重新构建：
   `cd ~/.claude/skills/gstack && bun run build`

也可以直接复制二进制：

```bash
cp browse/dist/browse ~/.claude/skills/gstack/browse/dist/browse
```
