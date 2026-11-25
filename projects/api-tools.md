# API Tools

> 项目说明草稿，整理于 2026-11-25。实现与验证范围对应当时的代码快照，可在此继续修改和完善。

一个用于管理模型 API 接入、测试端点和执行工作流的本地开发工作台。

## 解决什么问题

不同模型服务商的协议、模型配置和调用记录分散时，可以在同一个界面中配置 Provider、模型及端点，通过适配器调用，并查看工作流执行步骤和用量信息。

该项目与 Python 论文写作系统中的模型调用模块分别维护，不把它们视为已经打通的一套服务。

## 已有源码

- **协议适配**：OpenAI Chat Completions、Responses 和 Claude Messages 分别实现适配器，通过注册表选择对应实现。
- **接入配置**：管理 Provider、模型及端点，凭证通过环境变量名引用；配置导入导出和真实 Key 的保存分开处理。
- **工作流**：按步骤执行调用，保存 run、run step、状态和错误；当前检查到的执行路径为顺序步骤，不描述成智能路由或任意并行调度。
- **数据与界面**：Express API、sql.js/SQLite 存储和 React/TypeScript 页面分别组织；源码含接口、适配器与页面测试。
- **扩展入口**：仓库包含 MCP、Skill 和用量统计模块；其存在不代表所有外部服务均已完成本轮实测。

## 技术栈与结构

```text
client/                         React、TypeScript、Vite
server/src/routes/              Express HTTP API
server/src/adapters/            模型协议适配
server/src/apiProtocol/         操作与输入契约
server/src/workflows/           工作流与执行记录
server/src/db/                  sql.js / SQLite
server/src/configuration/       配置导入导出
```

## 启动与检查

```bash
npm ci
```

复制 `.env.example` 为本机 `.env`，按需要配置服务商凭证。不要把真实 Key 写入 README 或演示配置。

```bash
npm run dev
npm run typecheck
npm test
npm run build
```

开发界面默认使用 127.0.0.1:5173，后端默认端口为 8787。端口配置以当前源码和本机环境为准。

## 运行边界

当前后端有写入 API Key 配置的管理接口，启动代码未显式限定回环监听地址；网络监听限制及管理接口认证需要完善后再考虑部署。请在受控本机或隔离环境使用，不能把当前版本直接当作公网多用户服务。

历史 v0.3 验证报告记录过后端 118 项、前端 40 项测试及构建检查；本轮未重跑，也不将其数字直接当作当前分支结果。当前分支还包含后续用量统计代码，功能与测试范围应按实际版本分别确认。

项目个人贡献与 AI 辅助分工尚需进一步整理；当前文档按已见源码介绍实现，不声称全部独立手写。仓库尚未明确许可证。
