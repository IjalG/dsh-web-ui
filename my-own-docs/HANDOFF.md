# dsh-beyond-workscope 交接文档（给下一个会话）

> 交接日期：2026-08-14
> 交接原因：HMR 实验遇权限/机制疑云，用户要求交给权限更充足的会话继续
> **2026-08-14 接手增补**：HMR 一事已被用户明确排除（前会话权限问题所致，新会话
> 无此问题，无需理会本文 HMR 相关内容）。本会话已完成 E2E 全流程验收 + 4 处修复
> 落地 + 全仓回归全绿 + PR 准备更新，详见 §2 增补与 §8。

## 0. 一句话现状

插件 **已开发完成、已上线运行、已提交推送**；本会话补完了 E2E 验收并修复了验收
中发现的 4 个真实问题（全部已提交推送，分支 HEAD 见 §8）。剩余工作只剩远期
「上游 PR 提交」动作本身（需维护者拍板）。

---

## 1. 项目在哪里

| 项 | 路径 |
|---|---|
| **插件代码**（fork 仓库） | `~/桌面/codes/dsh-web-ui`，分支 `feat/beyond-workscope` |
| 插件包目录 | `~/桌面/codes/dsh-web-ui/packages/dsh-beyond-workscope/` |
| 设计文档（仓库外） | `~/桌面/codes/agent/dsh-beyond-workscope/DESIGN.md` |
| 本交接文档（仓库外） | `~/桌面/codes/agent/dsh-beyond-workscope/HANDOFF.md` |
| 上游仓库 | `github.com/zhu1090093659/dsh-web-ui`（PR 目标；fork 是 `IjalG/dsh-web-ui`，已配 SSH） |
| 已推送 | `origin/feat/beyond-workscope`（自 `f827c22` 起 + 4 个新提交，HEAD `d36c5a8`，见 §8） |

## 2. 已完成（可验证）

- **插件本体**：`workscope_probe`（感知：白名单根最近文件+进程，untrusted）/ `workscope_grant`（确认制授权，超时自动拒）/ `workscope_revoke` / `workscope_list` / `workscope_read` / `workscope_write`（授权边界强制）
- **授权注册表** `src/grants.ts`：per-session、realpath 规范化+前缀边界校验（防符号链接逃逸）、pending/active/denied/revoked/expired、上限、审计、会话结束自动撤销
- **路由** `/api/dsh-beyond-workscope/{pending,grants,audit,pending/approve,pending/deny,grants/revoke}`，id 走请求体（webServer exact 匹配不支持路径参数——已踩过坑），loopback 围栏
- **Browser 半区**：右下角确认卡片（倒计时/级别收紧/允许/拒绝）+ 授权管理（撤销/审计），zh/en
- **测试**：28 项全绿（`pnpm --filter @deepseek-ai/dsh-beyond-workscope test`），typecheck/build 通过
- **家族注册**：已进 `dsh-web-ui-all/aggregate.yml`（10 行 10 依赖），`node scripts/aggregate.mjs --check` 通过；根 README 已加「超越工作区」章节
- **本机已安装运行**：用户自己装了全家桶 `dsh-web-ui-all`（profile bundles 已含），插件路由实测 200、client bundle 已进 boot manifest、无启动错误
- **repo 内已知噪音**：任何 pnpm install/build 会重建 `packages/skins/*/lib/client.js`（构建机绝对路径差异），提交前必须 `git checkout -- packages/skins/`（除非真的想提交这些改动）

### 2.1 本会话新增（2026-08-14 接手后，全部实测通过）

- **E2E 验收完成**（场景 A/B/C）：probe 感知、grant 卡片确认 -> 写/追加/读 ->
  越界/路径逃逸拒绝 -> revoke -> 撤销后拒绝 -> 审计链完整；deny/超时路径由单测覆盖
- **修复 1（验收必现）**：`probe` 输出 `mtime` 为浮点（`stat.mtimeMs`），工具输出
  schema 要求 integer -> 工具直接报错。已 `Math.trunc` 截断 + 单测断言整数
- **修复 2（验收必现）**：`workscope_grant` 成功路径返回 `error: undefined` 字段 ->
  DSH 工具管道报「not lossless JSON」。已改为可选字段条件展开（同 dsh-ssh
  `ssh_list` 先例）+ 注释
- **修复 3（验收发现）**：默认感知根目录只有英文 XDG 名，中文系统（桌面/文档/下载）
  下 Documents/Downloads 缺失被跳过。已改为 locale 自适应（英文存在优先，否则中文
  回退，两者皆无保留英文以便 warning 可见）+ 2 条单测（`src/index.ts` 的
  `defaultScanRoots` 已导出）
- **修复 4（全仓回归发现）**：`dsh-git-graph` 1 条测试失败——中文 locale 下 git 输出
  本地化 stderr，`classifySwitchFailure` 只匹配英文文案 -> 全部归类 `internal`。
  已在生产 runner（`SubprocessSpawnSpec.env`）与测试 runner（`execFile` env）固定
  `LC_ALL=C`/`LANG=C`（core 新增 `GIT_LOCALE_ENV` 共享常量），44/44 回归全绿
- **PR 准备**：`docs/publish-prep.md` 清单 18 -> 19 包（新增 dsh-beyond-workscope）；
  `pnpm -r typecheck` / `pnpm -r test` 全仓全绿（git-graph locale 修复后）
- **重启 GUI 的方式**：`/tmp/restart-dsh-web.sh`（setsid 脱离进程树：sleep 5 ->
  pkill "bin/dsh web" -> 用绝对 node 路径重启 -> 轮询 3080 健康检查）；重启会中断
  当前会话，用户在 GUI 刷新后说「继续」即可恢复（会话持久化在 `~/.dsh/sessions/`）
- **已知机制事实**：profile boot 的 HMR 插件以 `config: { root: [] }` 创建——源码级
  热载 watch 默认关闭（与权限无关，是 boot 代码写死的）；patch 配置热载只做
  `fiber.update` 配置合并，不会重 import 包模块。所以插件 host 半区代码改动生效
  的唯一可靠路径是重启 GUI

## 3. 进行中/未完成

### 3.2 E2E 验收（用户 GUI 测试）— 2026-08-14 已完成

- 场景 A（probe 感知桌面最近文件）：实测通过（修复 1/3 后）
- 场景 B（grant 卡片确认 -> 读写 -> revoke）：实测通过（修复 2 后），审计链完整
- 场景 C（越界/逃逸拒绝、撤销后拒绝）：实测通过；deny/超时路径由单测覆盖
- 验收发现并修复 4 个真实问题（见 §2.1），全部回归后提交推送

### 3.3 上游 PR 准备（只剩提交动作本身）

- [已更新] `docs/publish-prep.md` 快照 18 -> 19 包（新增 dsh-beyond-workscope），
  版本 0.1.1/private 字段已核对一致
- [已通过] 聚合包 `--check`、全仓 `pnpm -r typecheck` / `pnpm -r test` 全绿
  （git-graph 中文 locale 失败已修，见 §2.1 修复 4）
- [待办] 提交 PR 到 `zhu1090093659/dsh-web-ui` 前需维护者确认；PR 前确认
  `dsh plugin remove` 单独包条目避免混淆（当前无害）
- [注意] 推送前 AGENTS.md 要求核验目标仓库为 PRIVATE；经 GitHub API 实测
  origin（IjalG）与 upstream（zhu1090093659）均为 **public**，已与用户确认
  继续推送到 origin（用户已拍板，2026-08-14）

## 4. 关键机制备忘（踩坑记录）

- `dsh plugin --profile web <args>` = 在 `~/.dsh/profiles/web` 跑 pnpm；`add link:<path>` 改 profile package.json bundles + node_modules 链接，**需重启**才加载
- patch 热载机制：`watchUserPatches`（app-boot）-> `hmr.registerConfig` -> 文件变更 -> include 更新。**web profile 默认不可用**（见 §3.1）
- webServer 路由匹配是 **exact 字面匹配**，路径参数用不了——id 走 body/query（dsh-ssh 惯例）
- tsconfig 里 `"types": ["node"]` 是必须的（本包直接 import node 内建，`types: []` 下构建会报"找不到 node:fs"——dsh-ssh 没写是因为它被 @types/ssh2 传递引入，本包没有这个运气）
- 插件 client 半区打包：`shared/tsdown.client.ts` 的 `clientBundle` 预设；client bundle 有纯度门（跨插件 value import 会被拒）
- `ApprovalOutcome = 'allowed-once' | ...`——DSH 审批是**单次放行**，做不了持久授权，所以本插件用自建 pending+轮询通道
- 皮肤/其他包 `lib/` 是提交物，重建会因绝对路径产生 diff（见 §2 噪音）

## 5. 探针插件源码（HMR insert 实验复现用）

```bash
mkdir -p /tmp/hmr-probe/lib
cat > /tmp/hmr-probe/package.json <<'EOF'
{
  "name": "@deepseek-ai/hmr-probe",
  "version": "0.0.1",
  "private": true,
  "type": "module",
  "main": "lib/index.js",
  "exports": { ".": "./lib/index.js", "./package.json": "./package.json" }
}
EOF
cat > /tmp/hmr-probe/lib/index.js <<'EOF'
export const name = 'hmr-probe'
export const inject = ['webServer']
export function apply(ctx) {
  ctx.webServer.register({
    kind: 'exact',
    path: '/api/hmr-probe/ping',
    handler: (req, res) => {
      res.writeHead(200, { 'content-type': 'application/json' })
      res.end(JSON.stringify({ ok: true, ts: Date.now() }))
    },
  })
}
EOF
ln -sfn /tmp/hmr-probe ~/.dsh/profiles/web/node_modules/@deepseek-ai/hmr-probe
# patch 里加：
# - insert:
#     - id: hmr-probe
#       name: '@deepseek-ai/hmr-probe'
# 轮询 http://127.0.0.1:3080/api/hmr-probe/ping —— 200 = insert 热载成功
```

## 8. 2026-08-14 接手会话新增提交（已推 origin/feat/beyond-workscope）

```text
d36c5a8 fix: dsh-beyond-workscope 感知根目录 locale 自适应（桌面/文档/下载 中文目录回退）+ 单测
022f765 docs: publish-prep 清单更新为 19 包（新增 dsh-beyond-workscope）
f56ffa3 fix: dsh-git-graph git 调用固定 C locale，失败分类不再依赖英文 stderr（中文 locale 下回归全绿）
5135a73 fix: dsh-beyond-workscope 工具输出修复（mtime 整数化、grant 可选字段条件展开）+ 回归测试
```

## 6. 常用命令

```bash
cd ~/桌面/codes/dsh-web-ui
pnpm --filter @deepseek-ai/dsh-beyond-workscope typecheck
pnpm --filter @deepseek-ai/dsh-beyond-workscope test
pnpm --filter @deepseek-ai/dsh-beyond-workscope build
node scripts/aggregate.mjs --check
dsh --profile web --dump-config | grep -A3 beyond-workscope   # 插件行
# 重启 GUI（会中断当前对话）：setsid nohup bash /tmp/restart-dsh-web.sh &（脚本已按绝对路径幂等重启）
```

## 7. 安全与边界提醒

- 授权只约束本插件的工具；DSH 沙箱管其余
- 感知数据固定 untrusted
- 管理路由 loopback-only 围栏（已有测试覆盖）
- 不要修改 `~/.dsh/settings.yaml`、`storages/` 等用户数据；patch 文件实验后必须还原

## 9. 2026-08-14 阶段二增补：第二个工作区（workspace 注册）

- 目标（用户原话）：不需要给 full access，就可以在工作区之外有一个额外的工作区
- 机制：`workscope_workspace`（确认制，复用卡片）-> 批准后 `ctx.workspaceRegistry.create` 注册为
  持久工作区 -> GUI 工作区切换器可见 -> 切换后新建会话即以该目录为沙箱工作区（bash/fs/git 全量）
- `workscope_unworkspace`：非破坏移除（目录与已有会话保留）；台账未命中时回退直查宿主持久注册表
  （重启后插件内存台账失忆，注册本身持久）
- 工作区持久语义：不随会话结束自动移除（与 grant 的自动撤销不同），显式移除
- 关键实现事实：`ctx.get('workspaceRegistry')` 在插件 apply 时必为 undefined（宿主 apiproxy 硬注入、
  注册晚于插件）——必须惰性 provider（执行时再取），routes/tools 全部 provider 化
- 持久注册表位置：`~/.dsh/storages/workspace.json`（6 个工作区，含 codes 项目集 9d8ad028）
- 已知限制（决策点）：ledger 与审计均内存态，重启丢失；审计后续可用 storage domain 持久化
- 新增提交（自 9a4052c 起 5 个）：阶段二功能 / 惰性解析 / list schema status / 回退移除 / list schema sessionId
- 测试 47 项全绿；实测闭环：注册 -> 持久化 -> 重启存活 -> 回退移除 -> 重注册 -> list 正常

## 10. 2026-08-14 v3 重构：子工作区（用户明确方向后）

- 用户定义：工作区确切说是「子工作区」，隶属会话；不进侧边栏；不创建会话；
  管理入口 = 会话详情区「对话/轨迹」右侧新增「会话信息」选项卡（也为用户未来生态铺路）
- 机制：workscope_workspace（确认卡）-> 会话级台账记录（WorkspaceLedger，纯内存）->
  「会话信息」选项卡（conversation.view id='session-info'，order 20）展示会话元信息 +
  子工作区列表/移除 + 会话内审计；子工作区内 workscope_read/write 自动放行（ledger.covers，write 级）
- 与 v2 差异：完全移除 workspaceRegistry 依赖（不再注册宿主工作区，侧边栏无痕、不建会话）；
  生命周期=会话（会话结束自动释放，与 grant 一致）；v2 遗留的宿主记录已用临时动态插件清理
- 关键实现事实：
  - 选项卡 = slots.register({name:'conversation.view', id, order, locale, label, inject}, View)，
    需 devDep @deepseek-ai/dsh-client-ui-conversation 拉入 SlotMap 增强 + type-only import
  - LocaleNamespaceMap 的值必须是「键联合类型」而非对象类型（trajectory 的 TrajectoryKey 范式）
  - ctx.sessions 直接属性访问需 inject 声明；插件内一律 ctx.get('sessions')（可选访问）
  - webServer 按 pathname 匹配（query 剥离），handler 抛异常 -> 空 400（webServer catch），
    插件路由必须防御化
- 提交：1ba98c6（v3 主体）/ a918964（文案）/ 78ccd8d（防御化）/ 5e22c22（ctx.get）
- 47 项测试全绿；实测：登记 -> 自动放行读文件 -> 宿主注册表无记录 -> session-info 元数据完整

## 11. 2026-08-14 可逆操作 + 感知时间线 + 操作回滚（goal-f2839884）

- 新工具：workscope_move / workscope_copy / workscope_delete / workscope_ops / workscope_rollback
- delete 移入插件回滚区（~/.dsh/dsh-beyond-workscope/rollback/<session>/<op>/artifact）而非销毁；
  write 前快照原内容；move 反向；copy 移除产物；回滚路径经授权门控；回滚区随会话清理
- probe 时间线：会话级缓存上次报告（内存），二次探测输出 delta（新增/变化/消失）
- 单测 58 项全绿；提交：7704cca（功能）、f168076（move/copy mkdir 父目录）
- 实测闭环：move（自动建目录）-> delete（移入回滚区）-> ops 列表 -> rollback 恢复 -> 内容完整、审计链完整
- 已知上游坑：skin-center 切换皮肤时写 ~/.dsh/cordis.patch.yml，首次会 append 到 dsh 默认的
  `[]` 后面形成坏格式（`[]` + 列表 -> YAML 解析失败，重启脚本崩溃）；后续切换写为纯列表则正常。
  属上游与 dsh 默认格式的兼容问题，遇崩溃恢复该文件为纯列表即可
- 注意：改 src 后必须 build（lib 是产物），否则重启加载旧代码

## §12 dsh-engineering 项目落地（goal-a289847d 完成）

- 仓库 `~/桌面/codes/dsh-web-ui/dsh-engineering/`（git@github.com:IjalG/dsh-engineering.git，
  main 分支，首提交 b6e0b10 已推 origin；包无 emoji、基于官方 SDK、shared/tsdown 预设）
- `packages/dsh-beyond-workscope-eng/`：beyond-workscope 副本，改名 `@linxin666/dsh-beyond-workscope-eng`、
  行 id `beyond-workscope-eng`；host apply 内 loader 组合检测到 dsh-web-ui 版行（id `beyond-workscope`）
  则 no-op，仅注册 `/api/dsh-beyond-workscope-eng/ping` 返回 `{provider:'dsh-web-ui'}`；
  否则完整注册 + ping 返回 `{provider:'dsh-engineering'}`；client 半区 8 次×500ms 轮询 ping 决定是否挂 surface
- `packages/dsh-engineering-all/`：聚合包 + 管理面板（设置 > 插件 > 插件配置，slot
  `settings.plugin.item` 分组 id `engineering-plugins` order 95；dsh-web-ui 家族分组
  `web-ui-plugins` order 90）；面板检测到 `web-ui-plugins` 在场时隐藏 beyond-workscope 成员条目
  （显示 "已由 dsh-web-ui 家族托管" 提示），避免重复
- profile 挂载：`~/.dsh/profiles/web/package.json` bundles 含 `@linxin666/dsh-engineering-all`，
  deps 顶层 link `dsh-engineering-all` + `dsh-beyond-workscope-eng`（loader 按顶层包名解析，
  嵌套依赖不生效）；`pnpm install --no-frozen-lockfile`（CI=true 下 frozen 会挡）
- 实测验证（重启脚本 + curl + Slots 查询）：
  - `GET /api/dsh-beyond-workscope-eng/ping` -> `{"provider":"dsh-web-ui"}`（dsh-web-ui 在场 -> 副本 no-op）
  - `settings.plugin.item` slot occupants 含 `engineering-plugins`(95) 与 `web-ui-plugins`(90) 共存
  - dsh-web-ui 版 beyond-workscope 无 ping 端点（该路径 404 属正常）
- 本次重启再遇 `~/.dsh/cordis.patch.yml` 坏格式：dsh 报 "failed to parse patches ... (4:1)"，
  日志上下文显示首行 `[]` 残留，但磁盘文件实际已是纯列表（mtime 20:16 修复过）——报错来自
  skin-center 期间旧崩溃日志，新进程正常（HTTP 200）。若再现：确保该文件为纯 `- id:` 列表即可

## §13 dsh-nas M1 落地（goal-9c219dbb 进行中，commit 83dd77e 已推）

- 包：`packages/dsh-nas`（系统）——host：应用注册表 nas.apps、文件 API（根=会话 cwd，路径边界，
  写审计）、回收站（.nas/trash 可恢复）、配置 ~/.dsh/dsh-nas.json(0600)；client：两态桌面
  （右侧停靠 768px 平板竖屏比例 + 全屏占满）、自研窗口系统、文件管理器、回收站、设置、搜索、
  preview；侧边栏入口 sidebar.footer.action（nas-entry）+ 桌面 shell.overlay（nas-desktop）
- 开关（省 token 诉求）：settings 配置 enabled/announceToAgent；**管理路由（apps.list/settings.get/
  settings.set）常驻，数据路由（fs/trash/prefs）随开关注册/注销**——关闭后 prompt section + 数据
  面消失，面板仍可显示状态并重新打开（不要重蹈"关了就开不了"的坑）
- UI 兼容：桌面/把手自动避让 dsh-web-ui aionui-panel 右列（测量 [data-aionui-explorer-col]/
  [data-aionui-preview-col] 宽度）；右侧把手可沿边上下拖拽（localStorage 持久化 dsh.nas.handleY）
- 排查笔记：bundle patch 生效（dump-config 可见 nas 行）但曾全 404——根因是 settings 持久化里
  enabled=false 残留 + management 路由被误删；profile 顶层 cordis.patch.yml 会被 dsh 重置为 [],
  别依赖它；验证实例注意端口残留与 pgrep 自杀（命令行含匹配串）

## §14 dsh-nas M2-M5 完成（goal-9c219dbb 收尾，commit 059e590..d0a4e48）

- M2 检索/任务/通知：SQLite FTS5 + CJK bigram 分词（search.ts，事务 rescan + 异步渐进避免首请求阻塞、
  跳过 node_modules 等大目录）；node-cron 计划任务（CRUD/启停/手动触发，notify/log 动作）；Webhook 通知
  幂等账本（pending/sending/succeeded/failed/uncertain 状态机 + 指数退避 + 人工裁决）
- M3 dsh-office：Word（TipTap 富文本 + mammoth/docx 往返）、Excel（Univer 在线表格引擎 +
  exceljs 桥接；公式 '=SUM(...)' 存真公式可往返，多 sheet 映射；旧自研网格保留备选）、PDF
  （pdf-lib 合并/拆分 + LibreOffice 转 PDF）、OCR（OpenAI 兼容视觉端点，配置 0600，结果 untrusted）；
  apps 注册用**延迟重试**（loader 并发 apply，nas.apps 可能未就绪——不要硬 inject 导致降级卡死）
- M4 dsh-mail：IMAP 收/阅读 + SMTP 发（mock 先行：.eml 出入工作区 .nas/mail-in|out），凭据
  ~/.dsh/dsh-mail.json 0600，不回显密码
- M5 审阅协同流：nas_edit 工具（工作区相对路径，写入即生成 pending 变更）+ review 台账
  （.nas/nas.db 表 nas_reviews，快照旧/新内容）+ 桌面「变更」窗口（行级 diff + 接受/拒绝，
  拒绝回滚旧内容）；公告已告知 agent 用 nas_edit 而非直接写文件
- 坑：webServer 路由 body 数字字段要用 num() 解析（str() 只收 string，number 会变 NaN）；
  profile 顶层 patch 会被 dsh 重置为 []，别依赖；新包挂载必须加 profile 顶层 link（loader 按包名
  从 profile 根解析）
- 主 GUI 规则：**不要重启/抓取 3080 主进程**（用户终端控制，重启会出乱子）；host 侧改动需用户
  方便时自行重启，client 侧改动刷新浏览器即生效
