# Rin 手动更新

上游：`https://github.com/openRin/Rin.git`。

## 手动策略

本个人 Fork 的上游同步只在本地执行；GitHub Actions 只保留 `workflow_dispatch`。
上游更新由使用者或受其明确指示的 AI 发起；没有定时同步、机器人推送或 GitHub 工作流连锁部署。
允许保留 Cloudflare 原生 Git 自动构建；推送是否同时部署按既有 CF 配置执行。
GitHub Actions 账号权限受限时，手动按钮也可能无法运行，使用下面的本地检查命令。

## 同步前准备

1. 先检查 `git status --short`，妥善保存已有修改；不要覆盖业务配置、数据迁移或其他未提交工作。
2. 核实 Cloudflare 当前连接、构建分支及部署命令，明确推送的线上影响；允许保留原生自动构建，无需关闭或断开连接。
3. `git remote -v` 核实自己的 `origin` 与本项目的 `upstream`；已有 upstream 时不要重复添加。

```powershell
git remote -v
git status --short
git switch main
git fetch upstream main
git log --oneline HEAD..upstream/main
git diff HEAD...upstream/main
```

确认上游版本和变更范围后，再手动合并。为先检查结果，使用 `git merge --no-commit --no-ff upstream/main`。
发生冲突时只处理已了解的改动；不确定时 `git merge --abort` 退出，不能使用 force push 或覆盖当前定制。

合并后必须检查 `.github/workflows`：上游可能重新引入自动工作流，也可能产生新的工作流文件。
保留本 Fork 的手动策略，删除上游同步入口；每个保留入口的 `on` 必须只有 `workflow_dispatch`。
核查 `schedule`、`cron` 触发器、`SYNC_PAT`、Git 推送、其他工作流调用/dispatch 和同步后的部署钩子。
服务运行所需的业务定时任务不属于源码更新，不要误删。

## 检查后决定推送与发布

先运行下面的本地检查并看 `git diff`，再由使用者明确决定是否提交、推送。
推送前说明 CF 既有流程是否会自动部署，并取得涵盖本次推送及其线上影响的授权。
CF 已完成部署时不要再次触发备用部署入口；独立部署命令仅在明确请求时执行。
部署前备份既有实例的数据和配置，并核对 Worker/Pages、D1、KV、R2 和域名；不要复制上游默认资源覆盖现有实例。
凭据保留在本地受控配置或既有 Secrets 中，不创建新的 PAT，不把值写入聊天、仓库或说明文档。

## Cloudflare 原生 Git 构建

- 2026-10-07 用户明确接受 CF 自动构建；保留既有 Pages / Workers Git 连接与构建配置。
- 自动构建与是否部署由当前项目配置决定。推送前核对分支和部署命令，含迁移时先按既有流程备份。
- 本次仅修改本地说明与工作流，没有核验或变更控制台设置；不删除 GitHub OAuth/App 授权。

参考：[Pages 分支构建控制](https://developers.cloudflare.com/pages/configuration/branch-build-controls/) ·
[Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/)。

不存在 upstream 时只需添加一次：`git remote add upstream https://github.com/openRin/Rin.git`。

## 本地检查与独立部署

```powershell
bun install --frozen-lockfile
bun run check
bun run format:check
bun run test
```

服务构建需要本项目 CLI 生成的实例配置。先按 README 准备本地配置（可用 `bun run dev:setup`），
核实 Wrangler 资源绑定后再运行 `bun run build`。不要把业务 `dev:cron` 或后台定时处理当作上游自动同步。

独立部署命令是 `bun run deploy`，它会构建并进行数据库迁移，只有明确请求部署时才执行。
手动 Actions 中：Build 成功后不会触发 Deploy；查看 Build 的 Run ID，另行打开 Deploy，
在 `build_run_id` 输入该 ID，或只填写已审阅的 Release `artifact_url`，两者只能选一。
该工作流会核查手动 Build 来源和部署脚本摘要。Release 需要填写已经存在的 `release_tag`（如 `v0.3.0`）；
Cleanup Preview 需要填写已关闭的 PR 编号，仅清理该 PR 的预览实例。
Coverage 留在 GitHub 制品中，不再使用旧版 Codecov 上传步骤。
