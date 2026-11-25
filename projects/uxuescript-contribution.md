# uXueScript：DeepSeek 网页求解器集成贡献

> 项目说明草稿，整理于 2026-11-25。实现与验证范围对应当时的代码快照，可在此继续修改和完善。

在既有 Vue + Rust/Tauri 客户端中接入独立 DeepSeek 网页 Adapter，让原有题目处理链可以选择网页生成，并沿用原答案数据结构。工作重点是模块消费、跨语言请求桥、WebView 生命周期和交互集成。

上游项目为 [unraous/uXueScript](https://github.com/unraous/uxuescript)。本人的工作位于 [Nuwanda218 开发分支](https://github.com/Nuwanda218/uxuescript/tree/feat/nuwanda-web-solver)，对应 [PR #61](https://github.com/unraous/uxuescript/pull/61)。截至 2026-11-25，PR 为 Open，尚未合并；本稿以 `af75d0385992c749684df13530c61ed3d3cc884f` 为本地审阅版本。

## 问题与结果

原生模型接口与网页操作有不同的完成和失败语义。网页脚本的异步 Promise 不能直接当作一次同步 `eval` 的返回值；刷新、换会话与迟到回调也可能让旧结果进入当前任务。

本次贡献将网页调用拆为“绑定文档、启动请求、读取状态、取消、确认消费”，使 Rust 宿主能够读取明确终态，并把纯文本结果接回既有批处理和严格答案解析链。网页模式仍是开发预览，真实课程回填尚未验收。

## 集成链与代码入口

```text
求解器设置 → Rust 题目路由 → 整卷预检与串行批次
    → RequestState → 页面 bootstrap → Adapter → DeepSeek 页面
    ← 完整结果与严格 JSON 解析 ← 同步状态快照与 ack
```

| 入口 | 本次工作涉及的职责 |
| --- | --- |
| `injected/deepseek-web/adapter-core/` | 固定提交的独立 Adapter runtime 源码及来源记录 |
| `injected/deepseek-web/adapter.js`、`bootstrap.js` | 暴露 API、唯一实例管理与请求控制器 |
| `scripts/build-scripts.mjs` | 复用现有管线将模块打包为页面注入 IIFE |
| `src-tauri/src/app/webview/deepseek/` | 请求桥、页面观测、登录检查、样式与实例状态 |
| `src-tauri/src/app/webview/solvers.rs` | 按需创建、复用及归位协调 |
| `src-tauri/src/core/quiz/web/deepseek.rs` | 纯文本预检、原 Prompt/批处理复用和答案解析 |
| `src/components/TheConfigPanel/TheSolverPanel.vue` | 求解器选择、登录引导与页面操作 |

## 关键决策

1. **源码消费与构建统一。** vendor 14 个 runtime 模块，来源固定为 `f4fdafe`。宿主只加必要的类型/lint 注解，通过原构建入口生成 IIFE；更新为人工同步，规则记录在 `UPSTREAM.md`。
2. **三层身份隔离。** `requestId` 区分请求，文档 token 区分宿主绑定的文档，`binding generation` 区分同文档内的会话绑定代次。控制器校验身份，防止旧页面或旧请求的迟到结果被当前任务消费。
3. **完整终态交付。** `bindDocument/start/read/cancel/ack` 中每次 `eval_with_callback` 只返回同步 JSON；网页异步生成结果保存在单个槽中，直到宿主读到并 `ack`。同身份重复启动只确认原请求，没有请求队列或自动重试。
4. **先预检，再串行生成。** 整卷图片和不支持输入在发送前检查；复用既有材料、Prompt 和答案结构。任一批失败均不交付部分答案，不回退 Native API。
5. **按需加载与本地收尾。** 选择网页模式时创建并复用 WebView，协调层级、登录引导与页面切换；本地 drain 等待在途生成和观察任务退出，不把请求槽清空当作任务已经结束。

## 构建、运行与定向检查

以仓库构建文档和锁文件为准，准备对应 Node.js/pnpm、Rust stable。Windows 还需 Microsoft C++ Build Tools、WebView2 Runtime，以及项目要求的链接器与 x86-64-v3 环境。

以下命令均从宿主仓库根目录执行，来自当前脚本和测试说明；**本轮文档盘点未运行这些安装、构建、应用或测试命令**：

```sh
pnpm install --frozen-lockfile
pnpm build:scripts
pnpm build
pnpm tauri dev
```

定向检查请求桥与网页纯文本业务入口：

```sh
pnpm exec vitest run tests/scripts/deepseek-bootstrap.test.js
pnpm sync:config
pnpm build:scripts
cargo test --manifest-path src-tauri/Cargo.toml -p uxuescript app::webview --locked --lib
cargo test --manifest-path src-tauri/Cargo.toml -p uxuescript core::quiz::web::deepseek::tests --locked --lib
```

启动应用后，由操作者登录并打开已有 DeepSeek 会话，在明确选择网页模式后发起任务。页面需保持输入为空、无附件且未在生成。项目测试说明中的手动清单是实际验收入口；不要把构建通过或静态“开发预览”提示当作网页已就绪。

## 验证状态与边界

- 历史证据包括独立 Adapter 同实例两轮真实文本生成，以及宿主单题网页生成与原答案结构解析；当前源码和公开提交可追溯，本轮未重跑真实页面。
- 已有本地测试资产覆盖重复启动、终态读取与确认、错误身份、迟到结果、导航中断、批处理失败和本地生命周期；Mock 与状态测试不证明实际 WebView2 或网站行为。
- 宿主网页入口只支持纯文本；独立 Adapter 的 PNG 实验能力未接入本链路。
- 真实学习通课程回填尚未验收，不能称为已验证的完整答题闭环。
- 取消保证本地等待结束，不能保证 DeepSeek 远端已经停止；不确定投递会隔离后续发送，需要操作者检查页面并按文档恢复。
- 本地 drain 不证明原生 WebView 已销毁；独立销毁及同标签重建仍未开放。此限制是当前能力边界。

## 贡献归属、上游与许可证

本人负责方案设计、AI 辅助开发、集成与验证，主要贡献为独立模块消费、网页请求桥、按需 WebView 与登录/页面交互。现有课程任务处理、Native 模型协议、原批处理和答案结构属于上游基础；本稿不将整个 fork 或整个应用表述为个人原创，也不表示全部代码均由本人独立手写。

uXueScript 根目录包含 GNU GPL v3 许可证，构建脚本的用户脚本元数据标注 `GPL-3.0-only`；上游版权及许可文件应保留。独立 Adapter 的来源记录显示其 `package.json` 无 license 字段、根目录无许可证文件，来源与授权状态需分别说明，本稿不替其指定许可。
