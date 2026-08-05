# Issue 追踪：GitHub

> **用途**：说明本仓库的 issue 与 PRD 如何以 GitHub Issues 形式存在，以及技能与人工应如何用 `gh` CLI 创建、读取、分诊与关闭工单（含 PR 作为分诊面的规则与 wayfinding 寻路操作）。

本仓库的 issue 与 PRD 以 GitHub Issues 形式存在。所有操作均使用 `gh` CLI。

## 约定

- **创建 issue**：`gh issue create --title "..." --body "..."`（多行正文用 heredoc）。
- **读取 issue**：`gh issue view <编号> --comments`，用 `jq` 过滤评论并拉取标签。
- **列出 issue**：`gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'`，按需加 `--label` 与 `--state` 过滤。
- **评论 issue**：`gh issue comment <编号> --body "..."`
- **加 / 去标签**：`gh issue edit <编号> --add-label "..."` / `--remove-label "..."`
- **关闭**：`gh issue close <编号> --comment "..."`

仓库由 `git remote -v` 推断；在 clone 内运行时 `gh` 会自动识别。

## Pull Request 作为分诊面

**PR 是否作为需求入口：否。** _（若本仓库把外部 PR 当作功能请求，则设为 `yes`；`/triage` 会读取此标志。）_

设为 `yes` 时，PR 走与 issue 相同的标签与状态，使用对应的 `gh pr` 命令：

- **读取 PR**：`gh pr view <编号> --comments` 与 `gh pr diff <编号>`。
- **列出待分诊的外部 PR**：`gh pr list --state open --json number,title,body,labels,author,authorAssociation,comments`，仅保留 `authorAssociation` 为 `CONTRIBUTOR` / `FIRST_TIME_CONTRIBUTOR` / `NONE` 的（丢弃 `OWNER`/`MEMBER`/`COLLABORATOR`）。
- **评论 / 打标签 / 关闭**：`gh pr comment`、`gh pr edit --add-label`/`--remove-label`、`gh pr close`。

GitHub 的 issue 与 PR 共用同一编号空间，裸 `#42` 可能指任一者——用 `gh pr view 42` 解析，失败再回退 `gh issue view 42`。

## 技能说「发布到 issue 追踪器」时

创建一个 GitHub issue。

## 技能说「拉取相关工单」时

运行 `gh issue view <编号> --comments`。

## 寻路操作（Wayfinding）

供 `/wayfinder` 使用。**地图（map）** 是一个带**子** issue 作为工单的单一 issue。

- **地图**：一个标了 `wayfinder:map` 标签的 issue，承载 Notes / 已决决策 / Fog 主体。`gh issue create --label wayfinder:map`。
- **子工单**：通过 GitHub 子 issue 机制链接到地图的 issue（用 `gh api` 操作子 issue 端点）。子 issue 未启用时，在地图主体用任务列表加入子项，并在子项顶部写 `Part of #<地图>`。标签：`wayfinder:<type>`（`research`/`prototype`/`grilling`/`task`）。认领后指派给驱动开发者。
- **阻塞**：用 GitHub 原生 issue 依赖——可见的规范表达。用 `gh api --method POST repos/<owner>/<repo>/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>` 加边，其中 `<blocker-db-id>` 是阻塞者的**数据库 id**（`gh api repos/<owner>/<repo>/issues/<n> --jq .id`，**不是** `#编号` 或 `node_id`）。GitHub 报告 `issue_dependencies_summary.blocked_by`（仅未关闭的阻塞，即实时闸门）。依赖不可用时，回退到子项顶部的 `Blocked by: #<n>, #<n>` 行。工单在全部阻塞关闭后解锁。
- **前沿查询**：列出地图的未关闭子项（`gh issue list --state open`，限定地图子 issue / 任务列表），丢弃仍有未关闭阻塞者（`issue_dependencies_summary.blocked_by > 0`，或 `Blocked by` 行中有未关闭 issue）或已有指派的；地图顺序中靠前者优先。
- **认领**：`gh issue edit <n> --add-assignee @me`——会话的首次写入。
- **解决**：`gh issue comment <n> --body "<answer>"`，随后 `gh issue close <n>`，再把上下文指针（gist + 链接）追加到地图的「已决决策」。
