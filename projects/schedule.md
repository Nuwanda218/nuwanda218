# Schedule Studio

> 项目说明草稿，整理于 2026-11-25。实现与验证范围对应当时的代码快照，可在此继续修改和完善。

使用原生 JavaScript ES Modules 构建的日程管理应用，包含年/月/周/日视图、任务编辑、筛选和本地保存。

## 工程重点

- 将日期处理、快速输入解析、任务存储、主题和语言切换从页面入口中分离。
- 使用 localStorage 保存任务，支持 JSON 导入导出；没有描述为多人协作后端。
- Quick Add 将输入中的日期、时间和时长转换为任务字段，检查脚本覆盖对应解析行为。
- 样式按基础、组件和视图拆分，页面通过 DOM 与事件更新。

## 本地运行

```bash
npm ci
npm run dev
```

开发服务器只监听本机，默认端口 5173。

```bash
npm run check
npm run build
```

`check` 检查页面资源和 Quick Add 解析，`build` 检查生产打包与模块导入。本轮整理未重新执行这些命令，也未核验公网部署。

## 主要目录

```text
assets/js/app.js                页面入口
assets/js/modules/              任务、日期、语言和主题模块
assets/js/core/storage.js       本地存储
assets/css/                     基础、组件和视图样式
scripts/                        资源及输入解析检查
```

项目用于前端模块化和交互练习。公开仓库尚未明确许可证，开发来源及 AI 辅助分工应在后续贡献记录中说明。
