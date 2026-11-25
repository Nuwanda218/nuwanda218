# uXueScript：DeepSeek 网页求解器集成

[uXueScript](https://github.com/unraous/uxuescript) 是基于 Vue、Rust 和 Tauri 的学习通辅助客户端。本次贡献为其题目求解流程增加 DeepSeek 网页模式：读取原有题目数据，通过网页生成答案，再解析为应用已有的答案格式，供后续处理使用。

我负责这一功能分支的方案设计、AI 辅助开发、集成与验证。独立模块的网页操作细节见 [DeepSeek Web Adapter](deepseek-web-adapter.md)，本文主要说明如何在已有应用中接入它。

[功能分支](https://github.com/Nuwanda218/uxuescript/tree/feat/nuwanda-web-solver) · [上游 PR #61](https://github.com/unraous/uxuescript/pull/61) · [首页入口改进提案 #62](https://github.com/unraous/uxuescript/issues/62)

## 复用原有题目处理链

题目提取、材料组织、批处理、提示词和答案结构已经由上游应用提供。新增网页模式的任务，是增加一种生成方式，而不是重新实现整条题目业务链。因此，集成保留原求解器选择接口，让网页模式与原有模型 API 模式分别处理生成请求，再返回相同的答案数据结构。

网页模式先检查整卷输入并准备所有批次，再逐批发送，只有所有批次都通过严格 JSON 和题目对应关系校验才返回答案。这样既复用现有处理规则，也避免前一批已生成、后一批失败时把不完整结果交给后续填写流程。网页模式不会自动重试或回退到模型 API。

主要代码入口是 `src-tauri/src/core/quiz/web/deepseek.rs`；求解器设置与登录引导位于 `src/components/TheConfigPanel/TheSolverPanel.vue`。

## 将独立模块接入原构建流程

集成采用 vendor 方式，将 Adapter 固定版本的 14 个 JavaScript 运行模块纳入 `injected/deepseek-web/adapter-core/`。模块保留相对目录与运行逻辑，少数文件仅增加宿主类型检查和 lint 所需的注解；`UPSTREAM.md` 记录来源提交、差异和更新规则。

这种方式让页面脚本跟随应用版本构建，避免发布时还要另行下载或手工替换一份外部 IIFE。`scripts/build-scripts.mjs` 复用既有打包流程生成独立脚本，Rust 再将产物注入 DeepSeek WebView。更新 Adapter 时需要显式选择版本、同步源码并检查差异，目前不自动同步。

## Rust 与 JavaScript 如何交接异步结果

网页生成通常需要数秒，而 Rust 的一次 `eval_with_callback` 只能读取脚本当次同步返回的内容。直接把网页中的 Promise 当作同步结果，无法可靠判断生成何时结束，也无法处理刷新和会话切换后的迟到回调。

因此，页面控制器负责启动和跟踪异步生成；Rust 通过短调用读取同步 JSON 状态。控制接口为 `bindDocument / start / read / cancel / ack`：先确认页面身份，启动请求，然后读取状态，最后确认接收结果。

```mermaid
sequenceDiagram
    participant Quiz as 题目求解流程
    participant Rust as Rust 请求管理器
    participant JS as 页面控制器
    participant Adapter as Adapter
    participant Page as DeepSeek 网页
    Quiz->>Rust: 提交一个纯文本批次
    Rust->>JS: bindDocument（文档标识）
    JS-->>Rust: 绑定确认
    Rust->>JS: start（请求 ID、文档标识、会话版本、文本）
    JS->>Adapter: generate（请求与当前会话）
    JS-->>Rust: 已接收请求
    Adapter->>Page: 填写、发送、观察回复
    loop 生成尚未结束
        Rust->>JS: read（同一请求身份）
        JS-->>Rust: pending 同步快照
    end
    Page-->>Adapter: 回复完成
    Adapter-->>JS: 完整结果或结构化错误
    Note over JS: 保存最终结果，等待确认接收
    Rust->>JS: read（同一请求身份）
    JS-->>Rust: 最终结果
    Rust->>JS: ack（同一请求身份）
    JS-->>Rust: 确认有效并清理结果槽
    Rust-->>Quiz: 返回结果，继续严格答案解析
```

三种身份信息各自处理一种变化：

| 信息 | 解决的问题 | 代码名称 |
| --- | --- | --- |
| 请求 ID | 区分每次生成，识别重复启动和迟到结果 | `requestId` |
| 页面文档标识 | 区分刷新、导航前后的文档，避免旧文档回调影响新页面 | `documentToken` |
| 会话重新绑定时更新的版本号 | 即使没有整页刷新，也能识别单页应用内的会话切换；切回原路径也不能复用旧结果 | `bindingGeneration` |

页面只保留一个活动请求及其结果。同一身份重复 `start` 只确认原请求，不再次调用生成。最终结果留在页面内存中，直到 Rust 读到并 `ack`，从而避免一次回调延迟就丢失结果；确认时还会检查会话版本，阻止在读取与接收之间发生的会话切换。相关实现位于 `injected/deepseek-web/bootstrap.js` 和 `src-tauri/src/app/webview/deepseek/request.rs`。

## WebView 创建与实例状态

应用启动时不必创建 DeepSeek WebView。选择网页模式后再按需创建，后续复用页面和已有登录状态；切换回课程页面时保留网页，减少重复加载。界面同时提供登录引导、页面查看和登录状态更新。

每次 WebView 创建过程拥有独立、共享的请求状态，配置入口、页面回调和生成入口都引用同一份状态。这样能让创建、就绪、关闭和失败恢复围绕实际实例协调，避免旧实例的回调改变新实例状态。相关入口为 `src-tauri/src/app/webview/solvers.rs` 及 `deepseek/` 下的实例与请求管理模块。

关闭过程中，应用先停止接收新任务，再取消并等待正在运行的生成与观察任务退出。这只能证明本地异步任务已收尾，不能证明操作系统和 WebView2 已经销毁原生页面资源。因此，本地任务完成与原生资源释放分开管理，不能仅凭关闭调用返回就立即用同一标签重建 WebView。

## 测试与验证

已有真实页面记录包括独立 Adapter 同实例连续两轮文本生成，以及应用内合成单题从提示词、网页生成到原答案结构解析的验证。

受控测试覆盖重复启动、结果读取与确认、错误身份、迟到回调、导航中断、批处理解析失败，以及 WebView 创建和本地任务收尾。这些测试用于稳定复现状态变化；真实网页生成用于检查网站行为，两者互相补充。真实网页结论对应已有验证版本，网站或 WebView 环境变化后需要重新验收。

## 本地构建与定向检查

在功能分支根目录，按项目构建文档准备 Node.js、pnpm 与 Rust stable。Windows 还需要 Microsoft C++ Build Tools、WebView2 Runtime，以及项目配置要求的链接器和 x86-64-v3 环境。

```sh
pnpm install --frozen-lockfile
pnpm build
pnpm tauri dev
```

请求控制器和题目生成链的定向检查：

```sh
pnpm exec vitest run tests/scripts/deepseek-bootstrap.test.js
pnpm sync:config
pnpm build:scripts
cargo test --manifest-path src-tauri/Cargo.toml -p uxuescript app::webview --locked --lib
cargo test --manifest-path src-tauri/Cargo.toml -p uxuescript core::quiz::web::deepseek::tests --locked --lib
```

更完整的环境与操作步骤见功能分支的 [构建说明](https://github.com/Nuwanda218/uxuescript/blob/feat/nuwanda-web-solver/docs/development/build.md)和[测试说明](https://github.com/Nuwanda218/uxuescript/blob/feat/nuwanda-web-solver/tests/README.md)。

## 当前状态与限制

- 截至 2026-11-25，[PR #61](https://github.com/unraous/uxuescript/pull/61)仍为 Open，尚未合并到上游；公开功能分支对应提交 `af75d03`。
- 网页模式处于开发预览，仅支持纯文本。用户需要登录并打开已有 DeepSeek 会话，保持输入框为空、无附件且没有其他生成任务。
- 真实学习通课程回填尚未验收。合成单题生成与解析成功，不代表真实课程页面的填写、暂存或评分均已通过。
- 本地取消不保证远端停止。不确定投递或请求发出后失联时，会阻止后续发送；操作者需检查网页，必要时手动停止并重启应用。
- 独立销毁 DeepSeek WebView 及同标签重建尚未开放。本地任务测试不等于原生资源销毁验收。

## 上游协作与个人贡献

本人负责功能分支的方案设计、AI 辅助开发、集成与验证，主要包括 Adapter 接入、跨语言请求管理、按需 WebView 与登录/页面交互。题目提取、原有模型 API 协议、批处理、提示词基础及答案结构来自上游项目。

除功能 PR 外，我还提交了 [Issue #62](https://github.com/unraous/uxuescript/issues/62)，提出保留默认学习通首页地址，同时允许用户选择其他门户入口，避免为不同入口修改源码并重新编译。上游作者已于 2026-11-17 回复解决，并将其以 completed 关闭；这是独立的改进提案，不等同于 PR #61 已被合并，也不表示相关修复由我实现。

uXueScript 根目录包含 GNU GPL v3 许可证，用户脚本构建元数据标注 `GPL-3.0-only`。Adapter 来源固定为 `f4fdafe`，记录在 [UPSTREAM.md](https://github.com/Nuwanda218/uxuescript/blob/feat/nuwanda-web-solver/injected/deepseek-web/adapter-core/UPSTREAM.md)；其独立许可证尚未明确，来源说明不替代授权说明。

更新于 2026-11-25。
