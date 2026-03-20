# 参与 gstack 开发

感谢你愿意改进 gstack。无论你只是修一个提示词拼写错误，还是想新增整条工作流，这份文档都会告诉你怎么快速进入开发状态。

## 快速开始

gstack 的技能本质上是 Claude Code 会自动发现的 Markdown 文档。正常情况下，它们安装在 `~/.claude/skills/gstack/`。开发 gstack 本身时，更方便的方式是让 Claude 直接读取你当前工作树里的技能文件，这样修改后立刻生效，不需要额外复制或发布。

开发模式会把当前仓库通过软链接接入本地 `.claude/skills/` 目录：

```bash
git clone <repo> && cd gstack
bun install
bin/dev-setup
```

之后你修改任意 `SKILL.md` 或模板文件，都可以立即在 Claude Code 里调用对应技能验证效果。完成开发后执行：

```bash
bin/dev-teardown
```

## Contributor mode

Contributor mode 会让 gstack 变成一个“会自我反馈”的工具。开启后，Claude Code 会在每个主要工作流阶段结束时给当前体验打分；只要不是 10 分，就会把问题、复现方式和改进建议写入 `~/.gstack/contributor-logs/`。

开启方式：

```bash
~/.claude/skills/gstack/bin/gstack-config set gstack_contributor true
```

这些日志是给你自己看的。当你决定修某个问题时，复现场景和改进建议往往已经现成写好了。

## 推荐贡献方式

1. 像平时一样使用 gstack
2. 查看 `~/.gstack/contributor-logs/`
3. Fork 并克隆 gstack
4. 在你真正感受到问题的项目里，把 `.claude/skills/gstack` 软链接到你的 fork
5. 直接修复问题
6. 用真实工作流验证修复
7. 提交 PR

这是最有效的贡献方式：在你真实使用 gstack 的地方修 gstack，而不是在脱离上下文的演示环境中修。

## 会话感知

当你同时开了多个 gstack 会话时，提问格式会显式带出项目名、分支名和当前任务，避免“这到底是哪一个窗口”的混乱。这个格式在各技能之间保持一致。

## 在 gstack 仓库里测试 gstack

`bin/dev-setup` 会在仓库内创建 `.claude/skills/` 软链接（已加入 `.gitignore`），让 Claude Code 直接读取当前工作树里的技能：

```text
gstack/
├── .claude/skills/
│   ├── gstack -> ../../
│   ├── review -> gstack/review
│   ├── ship -> gstack/ship
│   └── ...
├── review/
├── ship/
├── browse/
└── ...
```

这样你改完就能立刻在当前仓库里调用 `/review`、`/ship` 等技能验证效果。

## 日常开发流程

```bash
bin/dev-setup

# 修改技能或模板
$EDITOR review/SKILL.md.tmpl

# 重新生成文档
bun run gen:skill-docs
bun run gen:skill-docs --host codex

# 需要时重新编译 browse
bun run build

# 收工时退出开发模式
bin/dev-teardown
```

## 测试与评测

### 环境准备

```bash
cp .env.example .env
# 在 .env 中填写 ANTHROPIC_API_KEY
bun install
```

Bun 会自动加载 `.env`。如果你使用 Conductor，多数情况下工作区也会继承主工作树里的 `.env`。

### 测试层级

| 层级 | 命令 | 成本 | 验证内容 |
|------|------|------|----------|
| Tier 1 | `bun test` | 免费 | 静态校验、命令一致性、模板生成质量、技能引用合法性 |
| Tier 2 | `bun run test:e2e` | 付费 | 基于 `claude -p` 的端到端技能执行 |
| Tier 3 | `bun run test:evals` | 付费 | LLM-as-judge 对生成文档的质量评分 |

常用命令：

```bash
bun test
bun run test:e2e
bun run test:evals
```

### Tier 1：静态校验

`bun test` 会运行：

- 命令解析测试：检查技能里的 `$B` 命令是否都真实存在
- 技能校验测试：检查命令、flags、描述是否有效
- 生成器测试：检查模板占位符和生成结果是否符合预期

### Tier 2：E2E

E2E 会拉起 `claude -p` 子进程，以真实会话形式执行技能，是最接近“这个技能是否真的能端到端工作”的测试。

```bash
EVALS=1 bun test test/skill-e2e.test.ts
```

特点：

- 需要 `EVALS=1`，防止误触发高成本测试
- 如果运行环境嵌套在 Claude Code 内，会自动跳过
- 会把完整的 NDJSON 输出和失败诊断持久化到 `~/.gstack-dev/`

### Tier 3：LLM-as-judge

这一层使用 Claude 对生成的 `SKILL.md` 进行评分，主要考察：

- 清晰度
- 完整性
- 可执行性

每项分数必须达到阈值，且生成结果不能比 `origin/main` 的基线更差。

## 评测产物

E2E 和评测会把机器可读结果写到 `~/.gstack-dev/`，包括：

- 当前执行状态
- 部分结果
- 进度日志
- 每个测试的 NDJSON transcript
- 失败时的诊断 JSON

你可以使用：

```bash
bun run eval:list
bun run eval:compare
bun run eval:summary
```

来查看历史、对比两次运行、统计整体趋势。

## CI

GitHub Actions 会在每次 push 和 PR 时执行 `bun run gen:skill-docs --dry-run`。如果仓库里的生成文件已经过期，CI 会失败，提醒你先重新生成并提交最新文档。

## 编辑 SKILL.md 的正确方式

`SKILL.md` 是由 `.tmpl` 模板生成的。不要直接编辑 `.md`，否则下一次生成时会被覆盖。

正确流程：

```bash
$EDITOR SKILL.md.tmpl
bun run gen:skill-docs
bun run gen:skill-docs --host codex
```

如果你修改的是 browse 相关命令或快照能力，还应运行：

```bash
bun run build
```

## 提交建议

- 模板修改和生成文件更新最好一起提交
- 行为修改与纯机械改动尽量拆开
- 测试基础设施变更与具体测试用例变更尽量拆开
- 如果修改了技能的核心工作流，请至少跑一轮静态测试和必要的 E2E

## 贡献标准

一个高质量 PR 应满足：

- 改动范围清晰
- 模板与生成文件一致
- 测试通过
- 说明用户可感知的收益
- 不引入宿主或框架特定的硬编码假设

如果你不确定某个改动是否合适，最好的办法通常不是先问，而是先在真实项目里用它一轮，再决定要不要提 PR。
