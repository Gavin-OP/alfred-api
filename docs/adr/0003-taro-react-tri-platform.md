# ADR-0003 · 前端用 Taro + React 一套三端

- 状态：accepted
- 日期：2026-08-01

## 背景（Context）

用户的三条硬约束：

1. **"我最熟的是 React.js，想办法用 React.js"**
2. **"未来不仅是一个小程序，也可以是一个网站，或者被封装成一个 APP"**
3. 起步形态是**微信小程序**

微信小程序不跑标准 DOM，普通 React（Vite/Next）无法直接编译进去。要同时覆盖「小程序 + 网站 + App」，只有两条路：跨端框架，或者维护多套代码。

旧 jobhunter 的 `apps/mobile` 已经是 Taro + React，具备可参考的工程经验（也暴露了坑：`dev:web` 曾因缺 `@pmmmwh/react-refresh-webpack-plugin` 起不来）。

## 决策（Decision）

前端使用 **Taro 3.x + React + TypeScript**，一套代码编译到：

- 微信小程序（`build:weapp`）
- H5 网站（`build:h5`）
- React Native App（`build:rn`，后置）

状态管理与数据层用 TanStack Query（跨端可用），网络层封装 `Taro.request`，**不直接写 `window` / `wx`**，需要平台差异时用 `process.env.TARO_ENV` 分支。

## 理由与代价（Consequences）

**得到**

- 完全满足"用 React 写"+"一套代码三端"
- 复用旧仓库已验证的 Taro 工程经验
- 小程序这个最苛刻的目标从第一天就在编译目标里，不会出现"网站做完了发现搬不进小程序"

**代价**

- Taro 的抽象层意味着某些 Web/RN 生态库不可用，需挑跨端兼容的
- 编译链（webpack）偶发依赖问题，需要在仓库里锁死版本
- RN 端产物需要额外调试，不能假设"编译过就等于能跑"

## 备选方案（Alternatives）

| 方案 | 为何不选 |
|---|---|
| **Next.js（Web）+ Expo（App）+ 原生小程序** | 最"主流 React"，但等于**三套代码**，个人项目维护不起；用户明确要求一套代码覆盖三端 |
| **只做 Web（Vite + React），小程序后补** | 放弃了起步形态（小程序），且后补时大概率整体重写 |
| **uni-app** | 跨端能力相当，但主推 Vue，与用户"最熟 React"冲突 |
| **Flutter** | 三端能力强，但 Dart 与用户技术栈完全不匹配 |

## 备注

`alfred-web` 从**全新脚手架**开始，不直接拷贝旧 `apps/mobile`——旧代码带有 `@claw/*` 命名和手写共享类型的包袱（见 [ADR-0007](./0007-naming-alfred.md)、[ADR-0002](./0002-two-repositories.md)）。可参考其配置，但不继承其结构。
