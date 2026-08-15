# dsh-beyond-workscope 设计文档

> 状态：设计定稿，待开发
> 日期：2026-08-13
> 开发基地：`~/桌面/codes/dsh-web-ui`（IjalG fork，分支 `feat/beyond-workscope`，成型后 PR 到上游 zhu1090093659/dsh-web-ui）
> 本文档存放于仓库之外，不入库。

## 1. 定位（已与用户对齐）

一个 DSH Web UI 的 hot-pluggable 插件，做两件事：

1. **感知意图的方法**：给 agent 一个 read 级工具，让它能看到"工作区之外"的环境（白名单根目录的最近文件、活跃进程），从而感知用户当下在干什么。不提供任何"意图推断引擎"——agent 自己看图说话。
2. **超越工作区办事的权限**：细粒度、需确认、有审计、可撤销的"越界授权"。用户确认后，agent 获得在指定目录（工作区之外）的受限操作权限（read/write 分级），并只能通过插件自带的受限工具在该授权范围内执行。任务完成可释放，会话结束自动清。

**明确不做**（MVP）：
- 不做截图感知（二期）
- 不做模型意图推断/置信度排序（agent 自行判断）
- 不碰 DSH 源码（纯 bundle patch + client inject，与 dsh-ssh / dsh-task-board 同路线）
- 不改会话 cwd（DSH 的 `SessionHeader.cwd` 创建时冻结）——MVP 用"逻辑绑定"（授权注册表 + 注入上下文），二期再考虑会话级绑定
- 不重写 dsh-web-ui 项目本身，只按仓库惯例新增一个包

## 2. 背景事实（已核实）

### DSH 沙箱模型
- 三档：`read-only` / `workspace-write` / `danger-full-access`
- 写入边界 = 会话 cwd（workspaceRoot），`workspace-write` 下 cwd 之外一律拒绝
- 切档只有粗粒度 per-session 事件 `sandbox/mode`，无"指定目录"细粒度授权
- 模型侧可见 `sandbox:policy` 上下文贡献

→ 本插件补的正是"细粒度、有审计、可撤销的指定目录授权"。

### DSH 工具管道（可挂载点）
- `ctx.tools.register(defineTool({...}))`：注册即自动进 system prompt
- `tools/pre-execute` 为 allow/deny/ask 门；`ctx.tools.guard()` 为单调守卫
- `ctx.approval`（dsh-user-approval，SDK 包）承载 ask；未挂载时降级 deny
- agent 作用域 `agent.ctx` 支持 per-agent 注册/限制

### dsh-web-ui 仓库惯例（以 dsh-ssh 为模板）
- 双半区：host 半区 `src/index.ts`（引擎/路由/工具/设置/系统提示词公告），browser 半区 `src/client/index.ts`（slots + locale + DOM 挂载）
- `cordis.patch.yml`：`- insert: - id: <id>, name: '<pkg>'`
- `package.json`：`dsh.bundle.patch` + `dsh.client { inject: [client-runtime, client-connection, client-ui-settings], platform: 'web' }`
- 路由：`webServer` 注册 `/api/<name>/*`，loopback-only 信任围栏 + 同源标记（dsh-ssh 模式）
- 客户端通信：同源 `fetch`（`src/client/api.ts`）
- 构建：`shared/tsdown.client.ts` 的 `clientBundle` 预设；`tsc -p tsconfig.build.json` 出类型
- 失败策略：client DOM 挂载失败只 warn 不 throw（外部插件不得击穿 GUI boot）
- 全部依赖官方 NPM SDK `@deepseek-ai/*@0.1.0-rc.6`（与本机一致）

## 3. 包结构

```
packages/dsh-beyond-workscope/
├── cordis.patch.yml        # - insert: - id: beyond-workscope / name: '@deepseek-ai/dsh-beyond-workscope'
├── package.json            # @deepseek-ai/dsh-beyond-workscope, version 0.1.1, private
├── tsconfig.json / tsconfig.build.json / tsdown.config.ts / vitest.config.ts
├── src/
│   ├── index.ts            # host 入口：插件行、配置、设置节、组装
│   ├── protocol.ts         # 与 client 共享的类型 + API 前缀（纯类型/常量，client 可引用）
│   ├── perceive.ts         # 感知：白名单根最近文件 + 进程快照（untrusted）
│   ├── grants.ts           # 授权注册表：pending/active/审计、路径校验、自动释放
│   ├── tools.ts            # workscope_probe / workscope_grant / workscope_revoke /
│   │                       #   workscope_list / workscope_read / workscope_write
│   ├── routes.ts           # /api/dsh-beyond-workscope：pending 查询、确认/拒绝、授权列表、撤销
│   └── client/
│       ├── index.ts        # client 入口：locale + 挂载确认浮层/侧边入口
│       ├── api.ts          # 同源 fetch 客户端
│       ├── locales.ts      # zh/en
│       ├── mount.tsx       # 授权确认卡片 + 活跃授权列表（注入方式待定：slots 或直接 DOM）
│       └── css-modules.d.ts
├── tests/                  # grants/perceive/routes 单测
└── README.md               # 中文为主（仓库惯例双语可后补）
```

## 4. Host 半区设计

### 4.1 插件行与配置

```
id: beyond-workscope, name: '@deepseek-ai/dsh-beyond-workscope'
inject: ['webServer', 'tools', 'systemPrompt']
Config（schemastery）:
  announceToAgent: boolean = true      # 是否注入系统提示词公告
  enabled: boolean = true
  scanRoots: string[] = [桌面, 文档, 下载]   # 感知白名单根（懒展开，不含整盘扫描）
  maxRecentFiles: number = 20
  maxProcesses: number = 30
  confirmTimeoutMs: number = 120000    # 授权确认超时，超时即拒绝
  autoRevokeOnSessionEnd: boolean = true
```

设置节：`settingsNamespace('dsh-beyond-workscope')` + `installSettingsSection`（与 dsh-ssh 同法）。

系统提示词公告（对齐 dsh-ssh 的 SECTION_ORDER≈150 引导带）：
- 说明 workscope_* 工具的用途与触发词（"桌面/文档/下载/最近文件/看下我电脑/这个文件夹/那个目录"等）
- 明确：感知数据 untrusted；越界操作必须先 grant 且经用户确认；只能在授权路径内操作；完成或任务结束要 revoke

### 4.2 感知 `perceive.ts`

```
probe(ctx, opts) → PerceptionReport
  recentFiles: 对 scanRoots 下非隐藏目录递归扫描（深度≤3，跳过 node_modules/.git 等），
               按 mtime 取 top maxRecentFiles，输出 {path, mtime, size, kind(文件/目录/项目[含.git])}
  processes:   ps 枚举（Linux: ps -eo pid,comm,args；macOS 同；Windows: tasklist+wmic 降级），
               过滤自身与 shell 噪声，取 top maxProcesses {pid, name, args}
  sourceTrust: 'untrusted'（固定）
  scannedAt:   ISO 时间
```

- 感知输出**固定标记 untrusted**：不进长期记忆；工具输出 schema 里带 `sourceTrust` 字段，README/公告写明该约束
- 懒加载：只在工具被调用时才执行扫描；无桌面会话时静默降级（MVP 本就不含截图）

### 4.3 授权注册表 `grants.ts`

```
Grant { id, sessionId, path(规范化), scope: 'read'|'write', reason, status: 'pending'|'active'|'denied'|'revoked'|'expired',
        requestedAt, decidedAt?, decidedBy: 'user'|'timeout'|'auto', evidence?: string }
```

- **路径校验**：绝对路径 → `fs.realpath` 规范化；拒绝工作区自身（无意义）与根目录 `/`；拒绝非目录路径
- **pending**：工具发起后进入 pending 队列 → client 确认 → active；超时 → denied（timeout）
- **active 检查**：`isAllowed(sessionId, path, scope)` —— 路径 ∈ 任一 active 授权且级别覆盖（write 可读，read 不可写）
- **生命周期**：显式 revoke；会话结束自动 revoke（`session/event` 或宿主收尾钩子，按 SDK 可用面实现）；支持按 session 维度隔离（两个会话互不共享授权）
- **审计**：每次 grant/revoke/deny 记结构化日志 + 供 client 查询的历史列表

### 4.4 工具 `tools.ts`（全部走 `defineTool`）

| 工具 | 风险语义 | 行为 |
|---|---|---|
| `workscope_probe` | read | 返回 PerceptionReport（含 untrusted 标记）。触发词见公告 |
| `workscope_grant` | write（需确认） | 参数 `{path, scope: read\|write, reason}` → 创建 pending → 等待用户确认（阻塞至 confirmTimeoutMs）→ 返回 `{grantId, status, path, scope}`；确认期间工具结果说明"等待用户确认" |
| `workscope_revoke` | write | `{grantId?或 path?}` → 撤销 active/pending → 返回结果 |
| `workscope_list` | read | 当前会话 active + pending 授权列表 |
| `workscope_read` | read | `{path}` 读文件（文本，限 1MB），**仅当 isAllowed(session, path, 'read')**；否则拒绝并提示先 grant |
| `workscope_write` | write | `{path, content, mode?: append\|overwrite}` 写文件，**仅当 isAllowed(session, path, 'write')**；先创建父目录 |

- 确认机制：优先走 `ctx.approval`（若 SDK 暴露 request 且挂载）；否则走 host 内部 pending + client 路由确认。**以 SDK 实际 API 为准**（实现前查 `@deepseek-ai/dsh-user-approval` 类型）
- 所有拒绝都返回明确的人类可读原因（"不在授权范围内，请先 workscope_grant"）

### 4.5 路由 `routes.ts`

```
GET  /api/dsh-beyond-workscope/pending        → 当前会话 pending 列表（确认 UI 轮询）
POST /api/dsh-beyond-workscope/pending/:id/approve  {scope?} → 确认（可顺手改级别）
POST /api/dsh-beyond-workscope/pending/:id/deny     → 拒绝
GET  /api/dsh-beyond-workscope/grants          → active 列表
POST /api/dsh-beyond-workscope/grants/:id/revoke   → 撤销
GET  /api/dsh-beyond-workscope/audit           → 审计历史
```

- 信任围栏：沿用 dsh-ssh 的 loopback-only + 同源标记模式
- 会话上下文：路由从请求会话（`sessionId` 由 webServer 提供或由 client 传入）隔离授权视图

## 5. Client 半区设计

- **授权确认卡片**：pending 出现时在 GUI 右下角（或复用现有浮层插槽）弹出卡片：目标路径 / 级别 / 原因 / 感知证据摘要 / 确认与拒绝按钮（附"级别可调"）。轮询 `/pending`（1s），消失后停轮询
- **活跃授权入口**：侧边栏或设置节内"越界授权"列表：当前 active 授权、一键撤销、审计历史查看
- locale：zh / en 双语（仓库惯例）
- 失败策略：DOM 挂载失败 warn 不 throw
- 皮肤兼容：7 款皮肤下样式自适配（CSS 变量优先，参照现有插件）

## 6. 安全与边界

- 授权是**本插件的自管边界**：只在 workscope_* 工具上强制；不拦截 DSH 原生 fs/bash 工具（那些仍受 DSH 沙箱管）
- 感知数据 untrusted：不进记忆、不可作为指令来源（对齐 OAgent 的 provenance 约定）
- 越权写：`workscope_write` 只认 active write 授权，路径必须规范化后落于授权目录内（防 `..` 逃逸，realpath 后前缀校验）
- 个人桌面场景：授权确认强制（无免确认档）；二期可加"记住本次会话"之类的宽松档，但需用户显式开启
- 不碰 DSH 源码、不写 profile 配置、不注册全局命令

## 7. 验收标准（MVP）

1. `pnpm -r typecheck` / `pnpm -r test` 全绿（新包 + 既有包不回归）
2. `dsh plugin --profile web add link:<repo>/packages/dsh-beyond-workscope` 后重启，GUI 无报错、公告生效
3. 场景 A（感知）：让 agent "看看我桌面上最近改了什么" → workscope_probe 返回桌面最近文件，agent 正确总结
4. 场景 B（越界办事）：agent 想整理 `~/Downloads/合同/` → workscope_grant → GUI 弹确认卡 → 确认 → workscope_list/read/write 在授权目录内正常、目录外拒绝 → revoke 后全部拒绝
5. 场景 C（拒绝与超时）：deny 后工具返回拒绝；不确认时超时自动拒绝
6. 审计：grant/revoke/deny 全程可查
7. 会话隔离：A 会话授权不影响 B 会话

## 8. 二期（不做进 MVP）

- 截图感知（桌面会话可用时）+ 视觉描述，仍标 untrusted
- 会话级绑定（为新任务创建 cwd=目标的会话）
- 授权宽松档（用户显式开启的"本次会话免确认"）
- 注册进 `dsh-web-ui-all` 聚合包（PR 阶段按 `scripts/aggregate.mjs` 流程加）
- 双语 README 与皮肤中心预览（如适用）

## 9. 开放问题（实现前需查 SDK 实际 API）

- `ctx.approval` 的精确签名与"阻塞式 ask"是否可带自定义上下文（路径/原因/证据）——不行则用 pending+轮询方案
- webServer 路由如何拿到当前 sessionId（dsh-ssh 路由无 session 概念，需查 host-webserver / client-connection 的会话标识通道）
- 会话结束钩子：如何可靠收到"会话结束"（查 session 事件面），用于自动 revoke
- 系统提示词公告的注入方式（dsh-ssh 用固定字符串 SECTION；我们的公告需引用当前授权状态吗？MVP 不需要，保持静态）
