## 摘要（Summary）

新增 `dsh-beyond-workscope` 插件（超越工作区）：让 Agent 感知工作区之外的环境（`workscope_probe`，白名单根目录最近文件 + 活跃进程，全部标记 untrusted），并在不升级全局沙箱权限的前提下到工作区之外办事——确认制授权（`workscope_grant` / `workscope_read` / `workscope_write` / `workscope_revoke`，右下角确认卡片 + 全程审计 + 会话结束自动撤销），以及会话级「子工作区」（`workscope_workspace` / `workscope_unworkspace`：确认后目录登记为本会话子工作区，read/write 自动放行，管理入口在会话详情区新增的「会话信息」选项卡，位于「对话 / 轨迹」右侧，不进侧边栏、不创建会话）。同时修复 `dsh-git-graph` 中文 locale 下 git 失败分类失效的问题，并清理 `skin-center` 一处与上游 xp 去 bundleWired 修复相矛盾的过时测试。插件已注册进 `dsh-web-ui-all` 聚合包。

## 涉及包（Affected Packages）

- [ ] 任务看板 `packages/dsh-task-board`
- [x] Git 图谱 `packages/dsh-git-graph`（中文 locale 下 git stderr 本地化导致失败分类失效，固定 `LC_ALL=C`）
- [ ] 右侧面板 `packages/dsh-aionui-panel`
- [ ] 远程 Web UI `packages/dsh-remote-web-ui`
- [ ] SSH 远程运维 `packages/dsh-ssh`
- [ ] 实时令牌统计 `packages/dsh-live-stats`
- [ ] 宠物 `packages/dsh-pet`
- [x] 皮肤 / 皮肤中心 `packages/dsh-skins` / `packages/skins`（仅删除 skin-center 一处过时测试用例，无功能改动）
- [x] 聚合包 / 设置 `packages/dsh-web-ui-all` / `packages/dsh-web-ui-settings`（聚合注册 beyond-workscope 行与依赖）
- [x] 其他（请说明）：新增包 `packages/dsh-beyond-workscope`（本 PR 主体）；`docs/publish-prep.md` 清单更新；根 `README.md` 新增「超越工作区」章节

## PR 类型（PR Type）

- [x] 面向用户的功能或行为变更
- [x] Bug 修复
- [ ] 仅文档
- [ ] 维护 / 重构

## 最新代码确认（Latest Codebase Confirmation）

- [x] 我已基于最新 `main` 分支开发，或在提交前已 rebase / 合并最新 `main`。

同步命令：

```bash
git fetch origin && git merge origin/main
```

（本分支已两次合并 `upstream/main` 至 `7fc1d1c`，无冲突遗留。）

## AI 编码披露（AI Coding Disclosure）

- [x] 完全 AI 编码：全部编程改动由 AI 产出，并由贡献者接受 / 审查。
- [ ] 部分 AI 辅助：AI 帮助编写或修改了部分编程改动。
- [ ] 未使用 AI 编码辅助。

使用的 AI 模型：DeepSeek（V4 Flash）

使用的编码 Agent 工具：DeepSeek Harness

## 仓库规范检查（Repo Rules）

- [x] 未修改 DSH 官方源码，仅基于官方 NPM SDK（`@deepseek-ai/*`）开发。
- [x] 未新增指向 DSH 源码 checkout 的 tsconfig `extends` / `paths` / `references`。
- [x] 新增包目录以 `dsh-` 前缀命名（如 `packages/dsh-xxx`）。
- [x] 所有新增 / 修改文件不含任何 emoji 字符。

## 本地验证（Local Validation）

执行的命令：

```bash
pnpm install
node scripts/aggregate.mjs            # 重新生成聚合 cordis.patch.yml（--check 通过）
pnpm -r typecheck                     # 全仓通过（19 包）
pnpm -r test                          # 全仓通过（含新增包 47 项、git-graph 44 项、skin-center 45 项）
pnpm --filter @linxin666/dsh-beyond-workscope build
```

结果摘要：全绿。新增包单测覆盖授权注册表（pending/approve/deny/expire/revoke/会话隔离/上限/审计）、感知（mtime 整数、locale 根目录回退、跨平台进程解析与过滤）、子工作区台账（会话级、去重、covers 自动放行、会话结束释放）、路由（loopback 围栏、工作区确认/移除、session-info）。`pnpm -r build` 通过（构建机绝对路径噪音已还原）。

## 用户可见变更证据（Local Feature Evidence）

本机 GUI（web profile 全家桶，聚合包 link 安装）实测通过，证据链路如下；截图暂缺（本机无截图工具），PR 合并前可补充：

- 插件随聚合包加载：`/plugins/@linxin666/dsh-beyond-workscope/client.js` 200，`/api/dsh-beyond-workscope/*` 路由 200，启动日志无错误；
- 感知：`workscope_probe` 返回桌面 / 文档 / 下载（中文目录回退）最近文件与活跃进程（内核线程已过滤、按内存降序），输出带 `sourceTrust: untrusted`；
- 授权：`workscope_grant` 弹确认卡片（路径 / 级别 / 原因 / 倒计时）→ 允许后 active，`workscope_read` / `workscope_write` 仅在授权目录内可用（越界与 `..` 逃逸被拒），`workscope_revoke` 后立即失效，审计链 `grant_requested → grant_approved → grant_revoked` 完整；
- 子工作区：`workscope_workspace` 确认后登记为会话级子工作区——`workscope_read` 无 grant 直接读取目录内文件（自动放行），宿主持久工作区注册表无任何痕迹（不进侧边栏、不创建会话），「会话信息」选项卡数据源 `/session-info` 返回会话 id / cwd / 创建时间 / 标题 / agent preset；
- 复现步骤：装聚合包后新开会话，对 Agent 说「看看我桌面上最近改了什么」（触发感知）、「把这个目录变成我的子工作区」（触发确认卡片），在会话标题栏「对话 / 轨迹 / 会话信息」第三个页签查看管理。

证据：N/A（截图待补，见上）
