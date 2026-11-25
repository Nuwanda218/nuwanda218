# DeepSeek Web Adapter

> 项目说明草稿，整理于 2026-11-25。实现与验证范围对应当时的代码快照，可在此继续修改和完善。

把已有 DeepSeek 网页会话封装成可调用的 JavaScript 模块：调用方提交文本请求，Adapter 负责操作页面、确认是否发送、关联回复，并返回完整结果或结构化失败状态。

这是一个独立模块与客户端集成实践。当前版本为 `0.1.0`，包配置为 `private: true`，未发布到 npm；截至 2026-11-25，本地独立仓库尚未配置公开远端。公开可查的实际消费方是 [uXueScript 开发分支](https://github.com/Nuwanda218/uxuescript/tree/feat/nuwanda-web-solver)，相关贡献见 [上游 PR #61](https://github.com/unraous/uxuescript/pull/61)。截至同日，该 PR 为 Open，尚未合并。

## 解决的问题

网页生成是异步过程。点击发送、看见回复和确认当前请求完成是不同阶段；页面切换、虚拟列表滚动、控件异步更新也会影响判断。若只拼接点击脚本，调用方很难判断请求究竟未发送、已确认发送，还是投递状态不确定。

本模块将这些过程放进统一请求状态机，并隔离页面选择器和 DOM 操作，供宿主通过稳定接口消费。它操作网页公开界面，不依赖私有 DeepSeek API，也不依赖 Tauri 或 Tampermonkey。

## 实现与结构

| 文件或目录 | 职责 |
| --- | --- |
| `src/index.js`、`registry.js` | 三个公共导出与页面内单个存活实例管理 |
| `src/adapter.js` | 初始化、生成、取消、事件与资源清理 |
| `src/request.js` | 请求状态、绝对截止时间和投递状态 |
| `src/page/` | 选择器、输入、开关、附件、消息关联和正文提取 |
| `tests/unit/` | 状态与包消费契约；其余页面测试见 `tests/` |
| `vite.config.js` | 构建单文件 IIFE，供页面注入 |

主要设计：

- 请求携带 `requestId`，以消息签名和出现次序关联新回复，适应历史消息虚拟列表的可见窗口变化。
- 各阶段共用一个绝对截止时间；发送确认后才进入等待回复阶段，不把点击成功当成生成完成。
- 投递状态区分 `not_sent`、`unknown`、`confirmed`。发送结果不确定时返回明确状态，不自动重发。
- 取消区分本地等待结束与网页停止生成；后者只能尽力操作并观察，不能据此保证远端停止。
- 模块导入没有 DOM 操作或网络副作用；调用方显式执行 `initialize()` 和 `generate()`。

## 交付与使用

包根提供 ESM `import`，公共导出为 `VERSION`、`createDeepSeekAdapter`、`getDeepSeekAdapterInstance`；没有 CommonJS 接口。构建后得到 `dist/deepseek-web-adapter.iife.js`，安装全局对象 `DeepSeekWebAdapter`，加载本身不会初始化或发送消息。

以下是 ESM 调用顺序示意，需要消费环境已链接或安装本地模块；不是可从 npm 直接安装的说明：

```javascript
import { createDeepSeekAdapter } from 'deepseek-web-adapter';

const adapter = createDeepSeekAdapter();
// 在用户已登录且已打开的受支持会话页面执行：
await adapter.initialize();
const session = adapter.getSession();
if (!session) throw new Error('会话未初始化');
// 后续按 generate 的输入契约显式提交请求。
```

实际 uXueScript 集成采用 **vendor 源码**：将固定提交的 runtime 模块放入 `injected/deepseek-web/adapter-core/`，由宿主自己的构建管线生成 IIFE。它不是 npm 安装或自动同步；ESM 包根消费还另有独立消费者契约测试。

## 构建与检查

在独立模块根目录使用 Node.js、pnpm，依据锁文件安装开发依赖：

```sh
pnpm install --frozen-lockfile
pnpm test
pnpm typecheck
pnpm lint
pnpm format:check
pnpm build
```

命令来自当前 `package.json`；**本轮文档盘点未运行上述安装、测试或构建**。自动化检查覆盖本地状态、Mock DOM 与包/IIFE 契约，不能替代真实网页验收。首次接入时应先检查当前页面结构与会话条件。

## 已有证据与技术边界

- 历史真实网页记录：同一实例连续两轮文本请求完成，并验证销毁后重新初始化；本轮未重新访问真实会话。
- 页面要求已登录、已有 UUID 会话；不自动登录、创建或切换会话。
- 单张 PNG 的输入、挂载及 Mock DOM 流程已实现；历史记录仅确认真实页面挂载，尚未验收真实图片发送及模型理解，不能称为已验证的多模态能力。
- 真实取消、超时与会话切换未完成全面验收；页面改版可能要求更新 DOM 适配。
- uXueScript 已有纯文本消费链；真实学习通课程回填尚未验收。此模块返回完整结果，不提供 token 流接口。

## 个人贡献与来源

本人负责方案设计、AI 辅助开发、宿主集成和验证，包括模块交付契约、请求状态边界、网页适配问题定位及跨仓消费。该表述表示技术决策与交付责任，不表示全部代码均由本人逐行独立手写。

独立模块来源版本为 `f4fdafef1f3a7589a04f4384732a3a63c83f762c`；宿主的 `adapter-core/UPSTREAM.md` 记录了来源、工具注解和人工更新规则。当前独立模块 `package.json` 没有 `license` 字段，Git 根目录没有 LICENSE/COPYING 文件，许可证状态尚待明确；本草稿不替项目指定许可证。宿主 uXueScript 的上游与许可证见其原仓库，不能据此自动推定独立模块采用同一许可。
