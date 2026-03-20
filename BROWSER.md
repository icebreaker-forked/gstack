# Browser：技术说明

本文介绍 gstack 无头浏览器的命令参考与内部实现。

## 命令概览

| 分类 | 命令 | 用途 |
|------|------|------|
| 导航 | `goto`、`back`、`forward`、`reload`、`url` | 进入或切换页面 |
| 读取 | `text`、`html`、`links`、`forms`、`accessibility` | 提取页面内容 |
| 快照 | `snapshot [-i] [-c] [-d N] [-s sel] [-D] [-a] [-o] [-C]` | 获取引用、对比变化、生成标注 |
| 交互 | `click`、`fill`、`select`、`hover`、`type`、`press`、`scroll`、`wait`、`viewport`、`upload` | 操作页面 |
| 检查 | `js`、`eval`、`css`、`attrs`、`is`、`console`、`network`、`dialog`、`cookies`、`storage`、`perf` | 调试与验证 |
| 视觉 | `screenshot`、`pdf`、`responsive` | 查看页面视觉结果 |
| 对比 | `diff <url1> <url2>` | 比较两个环境差异 |
| 弹窗 | `dialog-accept [text]`、`dialog-dismiss` | 控制 alert/confirm/prompt |
| 标签页 | `tabs`、`tab`、`newtab`、`closetab` | 多页面工作流 |
| Cookie | `cookie-import`、`cookie-import-browser` | 从文件或真实浏览器导入 cookie |
| 批量执行 | `chain`（从 stdin 读取 JSON） | 一次调用执行多个步骤 |
| 交接 | `handoff [reason]`、`resume` | 切换到可见 Chrome 让用户接手 |

所有选择器参数都支持 CSS 选择器、`snapshot` 之后的 `@e` 引用，以及 `snapshot -C` 生成的 `@c` 引用。全部命令总数超过 50 个。

## 工作原理

gstack 的浏览器是一个编译后的 CLI 二进制。CLI 自身很薄，只负责读取状态文件、向本地持久化 Chromium 守护进程发送 HTTP 请求，再把结果输出到 stdout。真正执行浏览器操作的是基于 [Playwright](https://playwright.dev/) 的本地服务端。

```text
Claude Code
   │
   ▼
browse CLI（二进制）
   │  HTTP + Bearer Token
   ▼
Bun HTTP server
   │
   ▼
Playwright API
   │
   ▼
Chromium（无头）
```

### 生命周期

1. **第一次调用**：CLI 检查项目根目录下的 `.gstack/browse.json`。如果没有运行中的服务，就在后台启动 `browse/src/server.ts`。服务会拉起无头 Chromium、随机端口、生成 bearer token，并写入状态文件。首次启动通常约 3 秒。
2. **后续调用**：CLI 读取状态文件，发出带 token 的 HTTP POST，请求服务端执行命令。往返一般约 100 到 200ms。
3. **空闲关闭**：30 分钟无命令后，服务自动退出并清理状态文件。
4. **崩溃恢复**：如果 Chromium 崩溃，服务立即退出。下次调用时 CLI 会自动重新启动新实例。

## 主要组件

```text
browse/
├── src/
│   ├── cli.ts                     # 读取状态文件、发 HTTP、打印结果
│   ├── server.ts                  # Bun HTTP 服务端
│   ├── browser-manager.ts         # Chromium 生命周期、标签页、引用映射、崩溃处理
│   ├── snapshot.ts                # 可访问性树转 @ref、差异对比、标注
│   ├── read-commands.ts           # text/html/js/css/is/dialog 等只读命令
│   ├── write-commands.ts          # click/fill/select/upload 等写命令
│   ├── meta-commands.ts           # chain/diff/snapshot 路由与服务管理
│   ├── cookie-import-browser.ts   # 从真实 Chromium 浏览器解密并导入 cookie
│   ├── cookie-picker-routes.ts    # cookie 选择器 UI 的 HTTP 路由
│   ├── cookie-picker-ui.ts        # 自包含的 cookie 选择器界面
│   └── buffers.ts                 # console/network/dialog 的环形缓冲区
├── test/                          # 集成测试与 HTML 夹具
└── dist/
    └── browse                     # 编译后的二进制
```

## 快照系统

浏览器交互的核心能力是基于 `@ref` 的元素引用机制，它建立在 Playwright 的可访问性树 API 之上：

1. `page.locator(scope).ariaSnapshot()` 生成类似 YAML 的可访问性树
2. 解析器为每个元素分配引用，例如 `@e1`、`@e2`
3. 系统为每个引用构造对应的 Playwright `Locator`
4. 这个映射保存在 `BrowserManager`
5. 后续执行 `click @e3` 时，就能直接取回对应 `Locator` 并操作

优势是：

- 不需要修改 DOM
- 不需要注入脚本
- 直接复用 Playwright 原生能力

### 失效引用检测

SPA 页面在不跳转的情况下也会改变 DOM，比如切 tab、弹窗、路由切换。旧快照里的引用可能已经失效。为避免 Playwright 默认 30 秒超时，`resolveRef()` 会在真正执行前先做一次快速 `count()` 检查；如果结果为 0，就立即报错并提示重新执行 `snapshot`。

### 扩展能力

- `--diff` / `-D`：把快照存成基线。下一次带 `-D` 的调用会返回 unified diff，便于验证点击、输入等操作是否真的生效。
- `--annotate` / `-a`：在元素边界框上临时叠加标签层，截图后再移除，用于可视化引用位置。配合 `-o <path>` 指定输出路径。
- `--cursor-interactive` / `-C`：通过 `page.evaluate` 额外扫描 ARIA 树之外、但用户仍能点击的元素，比如 `cursor:pointer` 的 `div`、带 `onclick` 的节点或 `tabindex>=0` 的元素，并为其分配 `@c1`、`@c2` 引用。

## 截图模式

| 模式 | 语法 | 对应 Playwright API |
|------|------|---------------------|
| 整页截图（默认） | `screenshot [path]` | `page.screenshot({ fullPage: true })` |
| 仅视口 | `screenshot --viewport [path]` | `page.screenshot({ fullPage: false })` |
| 元素裁剪 | `screenshot "#sel" [path]` 或 `screenshot @e3 [path]` | `locator.screenshot()` |
| 区域裁剪 | `screenshot --clip x,y,w,h [path]` | `page.screenshot({ clip })` |

元素裁剪支持 CSS 选择器和 `@e` / `@c` 引用。命令会自动判断参数类型：`@e`/`@c` 视为引用，`.`/`#`/`[` 开头视为 CSS 选择器，`--` 开头视为 flag，其余视为输出路径。

## 认证与安全

每个服务会话都会生成随机 UUID 作为 bearer token，并以 `chmod 600` 权限写入状态文件 `.gstack/browse.json`。所有 HTTP 请求都必须带 `Authorization: Bearer <token>`，从而避免同机其他进程直接控制浏览器。

## 控制台、网络与弹窗捕获

服务端会监听 Playwright 的 `console`、`response` 和 `dialog` 事件，并把结果写入 O(1) 环形缓冲区，同时异步刷盘到：

- `.gstack/browse-console.log`
- `.gstack/browse-network.log`
- `.gstack/browse-dialog.log`

`console`、`network`、`dialog` 命令默认读取内存缓冲，而不是磁盘文件。

## 用户接管

当无头浏览器无法继续，例如遇到验证码、MFA 或复杂登录流程时，可以执行：

```bash
$B handoff "卡在登录页验证码"
# 用户在可见 Chrome 中手工完成操作
$B resume
```

`handoff` 会打开一个可见的 Chrome 窗口，并保留当前页面、cookie、localStorage 和标签页。用户手动完成后，`resume` 会把控制权交回代理，并附带新的快照。

系统在连续 3 次失败后会主动建议使用 `handoff`。

## 弹窗处理

为防止浏览器被 alert/confirm/prompt 卡死，系统默认自动接受弹窗。`dialog-accept` 和 `dialog-dismiss` 可覆盖这一行为；对 prompt 来说，`dialog-accept <text>` 会把文本作为输入值。所有弹窗都会记录类型、文本和最终动作。

## JavaScript 执行

- `js`：执行单条表达式
- `eval`：执行一个 JS 文件

两者都支持 `await`。如果表达式里包含 `await`，系统会自动包装成异步上下文：

```bash
$B js "await fetch('/api/data').then(r => r.json())"
$B js "document.title"
$B eval my-script.js
```

## 多工作区隔离

每个工作区拥有独立浏览器实例、独立 Chromium 进程、独立标签页、cookie 和日志。状态文件存放在当前项目根目录下的 `.gstack/` 中，因此不同项目互不干扰，也不会端口冲突。

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `BROWSE_PORT` | 0（随机 10000-60000） | 指定 HTTP 服务固定端口 |
| `BROWSE_IDLE_TIMEOUT` | 1800000 | 空闲自动关闭时间（毫秒） |
| `BROWSE_STATE_FILE` | `.gstack/browse.json` | 状态文件路径 |
| `BROWSE_SERVER_SCRIPT` | 自动检测 | `server.ts` 的路径 |

## 性能

| 工具 | 首次调用 | 后续调用 | 每次调用的上下文开销 |
|------|-----------|-----------|----------------------|
| Chrome MCP | ~5s | ~2-5s | ~2000 tokens |
| Playwright MCP | ~3s | ~1-3s | ~1500 tokens |
| **gstack browse** | **~3s** | **~100-200ms** | **0 tokens** |

在一个包含 20 条浏览器命令的会话里，MCP 方案会把大量 token 浪费在协议封装上，而 gstack 只走纯文本 stdout。

## 为什么用 CLI，而不是 MCP

对于本地浏览器自动化，MCP 额外引入了很多纯开销：

- 每次调用都要传 JSON schema 和协议包装，造成上下文膨胀
- 长连接更脆弱，掉线后恢复成本高
- Claude Code 本身已经有 Bash 能力，再叠一层协议没有必要

gstack 的做法更直接：编译后二进制，纯文本输入，纯文本输出，不需要额外连接管理。

## 致谢

浏览器自动化层建立在 [Playwright](https://playwright.dev/) 之上。可访问性树 API、Locator 系统和 Chromium 管理能力，是 `@ref` 交互模型成立的基础。感谢 Playwright 团队提供这样稳定扎实的底层能力。

## 开发说明

### 前置条件

- [Bun](https://bun.sh/) v1.0+
- Playwright Chromium（`bun install` 时会自动安装）

### 快速开始

```bash
bun install              # 安装依赖和 Playwright Chromium
bun test                 # 运行集成测试
bun run dev <cmd>        # 直接从源码运行 CLI
bun run build            # 编译到 browse/dist/browse
```

### 开发模式与编译二进制

开发期间优先使用 `bun run dev`，它会直接执行 `browse/src/cli.ts`，无需每次编译：

```bash
bun run dev goto https://example.com
bun run dev text
bun run dev snapshot -i
bun run dev click @e3
```

发布时再执行 `bun run build`，生成 `browse/dist/browse`。

### 运行测试

```bash
bun test
bun test browse/test/commands
bun test browse/test/snapshot
bun test browse/test/cookie-import-browser
```

测试会拉起本地 HTTP 服务 `browse/test/test-server.ts`，从 `browse/test/fixtures/` 提供 HTML 页面，再用 CLI 对这些页面执行命令验证。

### 源码索引

| 文件 | 作用 |
|------|------|
| `browse/src/cli.ts` | 入口。读取 `.gstack/browse.json`，向服务端发请求，并打印结果。 |
| `browse/src/server.ts` | Bun HTTP 服务。分发命令并管理空闲超时。 |
| `browse/src/browser-manager.ts` | Chromium 生命周期、标签页和引用映射。 |
| `browse/src/snapshot.ts` | 解析可访问性树、分配 `@e`/`@c` 引用，并处理 diff/annotate。 |
| `browse/src/read-commands.ts` | 只读命令：`text`、`html`、`links`、`js`、`css`、`is` 等。 |
| `browse/src/write-commands.ts` | 写命令：`goto`、`click`、`fill`、`upload` 等。 |
| `browse/src/meta-commands.ts` | chain、diff、snapshot 路由与服务管理。 |
| `browse/src/cookie-import-browser.ts` | 从真实 Chromium 浏览器解密 cookie。 |
| `browse/src/cookie-picker-routes.ts` | `/cookie-picker/*` HTTP 路由。 |
| `browse/src/cookie-picker-ui.ts` | cookie 选择器界面。 |
| `browse/src/buffers.ts` | 环形缓冲区与 console/network/dialog 捕获。 |

## 部署到当前技能目录

当前生效的技能通常位于 `~/.claude/skills/gstack/`。修改后：

1. 推送你的分支
2. 在技能目录执行 `git pull`
3. 重新构建：

```bash
cd ~/.claude/skills/gstack && bun run build
```
