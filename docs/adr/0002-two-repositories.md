# ADR-0002 前后端拆成两个独立仓库

- **状态**：accepted
- **日期**：2026-08-01

## 背景

旧 jobhunter 是 npm workspaces 单体仓库（`apps/mobile` + `server` + `packages/shared`）。

在最初讨论"把 Node 后端换成 Python"时，我曾建议**保持 monorepo**，理由是前后端强耦合、契约同步方便。但用户提出了不同的出发点：

> "我希望它能前端后端分开，因为未来可能会在上面做拓展。我觉得如果把它分成两个 repository 的话，可能更符合行业的 best practice。"

同时，后端换成 Python 这一事实**改变了 monorepo 的核心收益**：

- npm workspaces 管不了 Python 依赖，`npm install` 装不了 FastAPI
- `packages/shared` 是 TypeScript 类型包，Python 无法 import——**"共享类型单一事实源"这个 monorepo 最大卖点直接失效**

也就是说，语言一分家，monorepo 就只剩"目录挨在一起"这点心理便利了。

## 决策

拆成两个独立仓库：

| 仓库 | 技术栈 | 职责 |
|---|---|---|
| `alfred-api` | Python / FastAPI / PostgreSQL | 数据、AI 解析、Skills、提醒 |
| `alfred-web` | TypeScript / Taro / React | 小程序 + H5 + App 三端 |

**契约对齐方式**：后端 FastAPI 自动产出 `openapi.json` → 前端用 `openapi-typescript` 生成 `schema.d.ts`。
这比手写共享类型包**更强**：后端改字段，前端重新生成后 TypeScript 立刻报错，漂移无处可藏。

## 理由与代价

**得到**

- 两套语言的依赖、lint、CI、测试互不干扰
- 独立部署：后端上服务器/容器，前端发小程序/静态站，各自节奏
- 后端可以被别的客户端复用（CLI、自动化脚本、未来的 AI agent），不被前端仓库绑架
- 契约由 OpenAPI 机器生成，比人工维护的类型包更可靠

**代价**

- 跨仓库改动要开两个 PR（改接口 + 改调用）
- 需要一条约定：**后端先合并、前端再重新生成类型**
- 本地开发要同时开两个终端

缓解：前端提供 `npm run sync:api` 一键拉取并生成类型；接口变更在 API.md 与 CHANGELOG 里显式记录。

## 备选方案

| 方案 | 为何不选 |
|---|---|
| **保持 monorepo（Python + TS 混装）** | 跨语言 monorepo 需要 Nx/Bazel/Turborepo 之类的多语言编排，对个人项目是纯负担；且共享类型收益已因换语言而消失 |
| **Monorepo + git submodule** | submodule 的心智负担远大于两个平级仓库 |
| **三仓库（api / web / contracts）** | 契约已由 OpenAPI 自动生成，单独开一个 contracts 仓库属于过度设计 |
