# 伴居（暂定名）

面向私聊、长期记忆与家庭环境行动的陪伴 Agent。项目范围、软件结构和设计取舍以[软件架构](docs/overview.md)为准。

本仓库维护架构与设备契约文档；同一工作区中的 `runtime/` 提供共享运行能力和正式实现，`experiments/` 承载按名称组织的实验 Agent 与验证工作，目录与职责见[工作区与仓库分工](docs/index.md#工作区与仓库分工)。当前已有本地实现和 SimuHome 接口验证，实际覆盖见[本地验证](docs/simuhome.md#证据覆盖)。

## 阅读入口

- [软件架构](docs/overview.md)：角色、共用执行内核、任务、动态工具、记忆与反馈。
- [家庭设备工具契约](docs/device-contract.md)：统一设备能力与后端适配边界。
- [主 Agent 详细设计](docs/main-agent.md)：决策、委派、子任务结果处理、回复子 Agent 与验证计划。
- [执行子 Agent 详细设计](docs/executor-agent.md)：任务输入、结构化结果、范围检查与发现/暴露候选策略。
- [工具暴露与设备发现对照实验](docs/exposure-experiment.md)：对照组、规模设置与追加判定。
- [SimuHome 本地验证](docs/simuhome.md)：运行命令、证据覆盖和实验观察层。
- [文档地图](docs/index.md)：阅读路线、实验 Agent 目录约定、权威位置及状态定义。
- [协作规范](AGENTS.md)：架构修改与文档维护规则。
