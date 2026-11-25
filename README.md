# Hi, I'm Nuwanda.

天津大学计算机科学与技术学院，计算机科学与技术专业本科生（2024–2028）。

Computer Science and Technology Undergraduate at Tianjin University.

关注软件工程、系统设计与 AI 应用开发，喜欢从实际需求出发探索技术方案，在构建软件和解决问题的过程中不断学习。

[个人网站 · nuwanda.space](https://nuwanda.space/) · [项目介绍](projects/README.md)

## Engineering Interests

- **模块设计与开发**：功能抽象、接口设计与可复用模块。
- **系统设计与构建**：需求建模、架构设计与复杂度管理。
- **既有系统理解与扩展**：源码分析、系统集成与功能扩展。
- **AI 辅助开发**：探索 LLM 和 Agent 在软件开发、上下文管理及工程协作中的应用。

## Selected Projects

### CortexFlow

面向长期 AI 辅助开发的上下文管理系统，将分散的对话、任务和工作记录按项目与开发过程组织起来。通过浏览器适配层、消息通信与后端存储分工，减少系统对具体 AI 工具的依赖。

**技术**：Python · WebSocket · SQLite · Vue · TypeScript<br>
**状态**：持续开发中；已有消息接入与持久化，独立界面原型使用模拟数据，项目工作区存储在本地迭代。

[项目介绍](projects/cortexflow.md) · [源代码](https://github.com/Nuwanda218/CortexFlow)

### DeepSeek Web Adapter 与 uXueScript 集成

将网页输入、消息发送、生成状态检测和回复提取封装为独立适配器，再接入已有的 uXueScript 桌面应用，用于网页题目求解。这个项目同时涉及模块接口设计与既有系统集成。

**技术**：JavaScript · ESM / IIFE · Rust · Tauri · WebView<br>
**状态**：完成模块及功能分支集成，有真实网页文本交互验证；功能 PR 待合并。

[适配器介绍](projects/deepseek-web-adapter.md) · [集成贡献](projects/uxuescript-contribution.md) · [功能分支](https://github.com/Nuwanda218/uxuescript/tree/feat/nuwanda-web-solver) · [PR #61](https://github.com/unraous/uxuescript/pull/61)

### API Tools

模型 API 管理与调用工作台，将服务商配置、协议适配、端点测试和工作流执行记录放在同一个界面中。不同协议由独立适配器处理，界面和工作流使用统一调用入口。

**技术**：React · TypeScript · Express · SQLite / sql.js<br>
**状态**：已有前后端与自动化测试，当前面向受控本地使用，网络监听及管理接口认证仍需完善。

[项目介绍](projects/api-tools.md) · [源代码](https://github.com/Nuwanda218/API-TOOLS)

### 校园在线刷题与复习系统

合作开发的校内期末复习网站，本人参与开发。系统提供轮次进度、错题训练和题库更新，通过提交标识处理重复提交，并通过备份、历史归档和事务迁移保留答题数据。

**技术**：Python · Flask · SQLite · JavaScript<br>
**成果**：历史归档记录 **2868 个注册账户**、约 **84.9 万条答题记录**，当前提供脱敏项目介绍。

[项目介绍](projects/campus-quiz.md) · [项目仓库（介绍版本）](https://github.com/Nuwanda218/campus-quiz)

## Other Projects

- [Schedule Studio](projects/schedule.md)：原生 JavaScript 日程应用，包含多视图、快速输入解析和本地存储。[源代码](https://github.com/Nuwanda218/schedule)
- [Nuwanda Space](projects/nuwanda-space.md)：个人网站中的内容组织、图片展示与 Astro 工程实践。[网站](https://nuwanda.space/) · [源代码](https://github.com/Nuwanda218/nuwanda.space)

## Technical Practice

主要使用 Python、JavaScript / TypeScript 开发工具和 Web 功能，有 Rust / Tauri 既有工程的阅读与集成经历。开发中使用 Git、Linux、HTTP、WebSocket、SSE、SQLite 和 Nginx，并结合 AI 辅助实现、代码审查、测试与人工验收推进项目。
