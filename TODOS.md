# TODOS

## Browse

### 将 `server.ts` 一并编进二进制

**What:** 去掉 `resolveServerScript()` 的回退链，把 `server.ts` 直接打包进 `browse` 二进制。  
**Why:** 当前依赖运行时路径查找，脆弱且已经引发过问题；单一二进制更简单、更可靠。  
**Context:** Bun 的 `--compile` 支持多入口，理论上可以直接把服务端逻辑一起打进去。  
**Effort:** M  
**Priority:** P2  
**Depends on:** None

### 会话隔离（按名称区分浏览器实例）

**What:** 支持多个相互隔离的浏览器会话，每个会话拥有独立 cookie、storage 和历史记录。  
**Why:** 方便并行测试不同用户角色、不同授权状态和 A/B 验证。  
**Context:** 需要利用 Playwright context 隔离；也是录像、状态持久化和 auth vault 的前置条件。  
**Effort:** L  
**Priority:** P3

### 浏览器交互录像

**What:** 录制浏览器交互视频，支持开始/停止。  
**Why:** QA 报告和 PR 里都需要更直观的证据。  
**Context:** 依赖隔离会话，且需要解决 WebM → GIF 或其他嵌入格式转换。  
**Effort:** M  
**Priority:** P3  
**Depends on:** 会话隔离

### v20 Cookie 加密格式支持

**What:** 为未来 Chromium cookie 数据库的 AES-256-GCM 格式做兼容。  
**Why:** 避免浏览器升级后 cookie 导入失效。  
**Effort:** S  
**Priority:** P3

### 状态持久化

**What:** 将 cookies + localStorage 持久化为 JSON，并支持重新载入。  
**Why:** 让 QA 会话可恢复、认证状态可复现。  
**Context:** handoff 功能里的 `saveState()` / `restoreState()` 已经具备大部分状态抓取能力。  
**Effort:** S  
**Priority:** P3  
**Depends on:** 会话隔离

### 凭据保险库

**What:** 按名称引用的加密凭据存储，避免密码进入 LLM 上下文。  
**Why:** 当前认证信息有机会穿过模型上下文，存在安全隐患。  
**Effort:** L  
**Priority:** P3  
**Depends on:** 会话隔离、状态持久化

### Iframe 支持

**What:** 提供 `frame <sel>` 与 `frame main` 等跨 frame 交互能力。  
**Why:** 许多网页把支付、嵌入和广告放在 iframe 中，目前 browse 看不到。  
**Effort:** M  
**Priority:** P4

### 语义定位器

**What:** 支持按 role / label / text / placeholder / testid 查找元素并附带动作。  
**Why:** 比 CSS 选择器或引用编号更稳。  
**Effort:** M  
**Priority:** P4

### 设备预设

**What:** 支持 `set device "iPhone 16 Pro"` 这类预设。  
**Why:** 响应式测试不必手工调 viewport。  
**Effort:** S  
**Priority:** P4

### 网络 mock / 路由控制

**What:** 拦截、阻断、模拟网络请求。  
**Why:** 用于测试错误态、加载态和离线场景。  
**Effort:** M  
**Priority:** P4

### 下载处理

**What:** 支持点击下载并可控输出路径。  
**Why:** 端到端验证文件下载链路。  
**Effort:** S  
**Priority:** P4

### 内容安全控制

**What:** 增加 `--max-output` 截断和 `--allowed-domains` 域名白名单。  
**Why:** 防止上下文窗口溢出，并限制跳转到不安全域名。  
**Effort:** S  
**Priority:** P4

### 流式预览

**What:** 基于 WebSocket 的实时预览，用于结对浏览。  
**Why:** 让人类可以实时观看 AI 浏览过程。  
**Effort:** L  
**Priority:** P4

### CDP 模式

**What:** 连接已运行的 Chrome / Electron 应用。  
**Why:** 便于测试生产浏览器实例、Electron 或既有会话。  
**Effort:** M  
**Priority:** P4

### Linux / Windows Cookie 解密

**What:** 支持 GNOME Keyring、kwallet、DPAPI 等。  
**Why:** 让 cookie 导入不再局限于 macOS Keychain。  
**Effort:** L  
**Priority:** P4

## Ship

### `/ship` 运行日志

**What:** 每次 `/ship` 结束后，向 `.gstack/ship-log.json` 追加结构化记录。  
**Why:** 让 `/retro` 可以读取真实发版数据，如 PR 速度、review 质量和测试增长。  
**Context:** 项目里已经有类似的 JSON append 模式。  
**Effort:** S  
**Priority:** P2

### 发布后验证（ship + browse）

**What:** push 完后自动打开 staging / preview，截图关键页面、检查 console，并比对 staging 与 prod。  
**Why:** 把部署时回归尽量拦在合并前。  
**Context:** 若要把截图写进 PR，需要额外的图片托管基础设施。  
**Effort:** L  
**Priority:** P2  
**Depends on:** 上传能力、视觉注释能力

### PR 中的视觉验证截图

**What:** `/ship` 在发 PR 前对关键页面截图，并嵌入 PR 正文。  
**Why:** Reviewer 不必本地拉起即可看到变化。  
**Effort:** M  
**Priority:** P2  
**Depends on:** 上传能力

## Review

### 行内 PR 注释

**What:** `/ship` 和 `/review` 通过 `gh api` 直接在 `file:line` 位置发 review comment。  
**Why:** 比起顶层评论，行内评论更可操作，也更适合人机共同审查。  
**Effort:** S  
**Priority:** P2

### Greptile 反馈导出

**What:** 将 `greptile-history.md` 聚合为机器可读 JSON，便于导出给 Greptile 团队。  
**Why:** 让误报模式真正形成反馈闭环。  
**Effort:** S  
**Priority:** P2  
**Depends on:** 积累足够多的 FP 数据

### 视觉评审与标注截图

**What:** `/review` 在预览环境打开改动页面，生成标注截图，和生产环境比对。  
**Why:** 视觉回归是纯代码评审很难发现的一类问题。  
**Effort:** M  
**Priority:** P2

## QA

### 更强的回归测试自动补齐

**What:** 对 `/qa` 修复出的每个问题，自动生成更接近真实场景的回归测试草案。  
**Why:** 目前能补一部分，但还不够稳定和系统。  
**Effort:** M  
**Priority:** P3

### 更完整的 QA 报告索引

**What:** 自动维护按日期、分支和 PR 聚合的 QA 历史索引。  
**Why:** 方便比较同一功能在多次修复后的质量趋势。  
**Effort:** S  
**Priority:** P3

## Infrastructure

### 通用上传能力

**What:** 提供 gstack 统一的截图 / 附件上传能力。  
**Why:** ship、review、qa 都会逐步需要稳定的图片与报告托管。  
**Effort:** L  
**Priority:** P2

### 更细的多宿主适配测试

**What:** 强化 Claude / Codex / 其他宿主的输出一致性检查。  
**Why:** 模板越来越多，共享逻辑越来越复杂，需要更强回归防线。  
**Effort:** M  
**Priority:** P2

## Completed

已完成事项请保留原条目内容，并追加：

```markdown
**Completed:** vX.Y.Z (YYYY-MM-DD)
```
