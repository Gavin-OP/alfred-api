# ADR-0006 MVP 延后 LaTeX → PDF 编译

> ⚠️ **状态更新（2026-08-03 V1 重新设计）**：本 ADR 已被 **[ADR-0014 · 简历 Markdown 化 + 版本溯源](./0014-resume-provenance.md)** 取代。简历主信息改为结构化 Markdown（`document_version.markdown_main` + 模板引用 `template_ref` + `checksum`），PDF 编译延后到用户确认后才执行；**不再以 LaTeX 源码作为中间格式**。本文件保留作历史追溯。

- **状态**：superseded by ADR-0014
- **日期**：2026-08-01

## 背景

用户对简历/Cover Letter 的原始要求包含编译能力：

> "我希望给你一些 LaTeX 的模板，你可以帮我自动修改简历……后端需要有运行 LaTeX 的能力，能够直接生成精美的 PDF。"

技术现实：完整 TeX Live 约 1–5 GB，中文简历还要 XeLaTeX + 中文字体；编译是同步阻塞的重任务；不同模板依赖的宏包千差万别，缺包就编译失败。这会让后端镜像和本地开发环境立刻变重。

而 MVP 阶段的核心目标（用户已确认）是**"先把后端 DB 与 API 全套做扎实"**。

在明确权衡后，用户选择了 **"建模型+存版本，PDF 延后"**。

## 决策

MVP **只做数据层与 AI 修改层**，不做编译：

**做**
- `document_asset` / `document_version` 完整版本链模型（LaTeX 源码不可变快照、`based_on_version_id` 版本链、`diff_summary_md`）
- `application.resume_version_id` / `cover_letter_version_id` —— 精确记录每次投递用的哪版
- `POST /v1/documents/{id}/tailor` —— AI 针对某岗位给修改建议 / 直接生成新版本源码
- 版本 diff 端点

**不做（但预留）**
- `document_version.pdf_path` / `compiled_at` / `compile_log` 字段**先建好，值为 NULL**
- `POST /v1/documents/{id}/versions/{no}/compile` 端点**先存在，返回 `501 Not Implemented`**

## 理由与代价

**得到**

- MVP 环境保持轻量，`docker compose up` 秒级可用
- 用户最痛的问题**当下就解决了**：「收到面试时忘了当初用的哪份 CV」——版本链和投递引用已完整可查
- 编译只是"把已有源码渲染出来"，是纯增量功能，不影响任何已有结构

**代价**

- 用户暂时要自己把源码贴进 Overleaf / 本地 TeX 出 PDF
- `compile` 端点先返回 501，属于已知的"故意留白"

## 何时补上（v0.3）

倾向方案：**独立编译容器**（`texlive/texlive` 镜像 + 一个极薄的 HTTP 包装），主后端通过内网调用。

好处：主镜像不被污染；编译崩溃/超时被隔离；可以按需启停这个吃资源的容器。
替代方案（宿主机装 TeX Live）会把开发环境和生产镜像都拖重，且难以复现。

决策留到 v0.3 实际动手时再敲定，届时补一份新的 ADR。

## 备选方案

| 方案 | 为何不选 |
|---|---|
| **MVP 就含编译** | 装 1GB+ 依赖，拖慢 MVP，且与"先把 DB/API 做扎实"的既定优先级冲突 |
| **调用外部 API（如 Overleaf / latexonline）** | 简历是**高度隐私数据**，不应上传到第三方编译服务 |
| **改用 Markdown → PDF（Pandoc/WeasyPrint）** | 用户明确要 LaTeX 模板，排版精度是简历的核心诉求，不能降级 |
| **前端编译（WASM 版 TeX）** | 体积巨大、小程序端不可行 |
