# Claude Code Analysis & Enterprise AI Planning

一个面向 **企业 AI 规划建设、Agent 架构设计、工具治理与权限治理** 的研究型仓库。

本仓库以 Claude Code 为切入点，逐步扩展到企业级 AI 平台总体架构、制药行业 AI Agent 治理框架，以及企业级 MCP/工具接口规范设计，形成一套从 **产品架构研究 → 工程机制拆解 → 企业落地方法论** 的连续知识体系。

---

## 仓库定位

这个仓库不只是对 Claude Code 的拆解记录，更是一个面向企业 AI 建设的参考样板，重点回答三类问题：

1. **先进 Agent 产品是怎么设计的**
2. **这些设计背后的系统化、工程化思想是什么**
3. **如何把这些思想迁移到企业 AI 平台建设中，尤其是高合规行业**

---

## 文档目录

### 1. Claude Code 研究

- [`claude_code_architecture_analysis.md`](./claude_code_architecture_analysis.md)
  - Claude Code 整体架构设计解析
  - 聚焦 Agent Loop、工具系统、权限模型、上下文工程
  - 提炼企业 AI 规划可直接借鉴的方法论

- [`claude_code_sourcecode_analysis.md`](./claude_code_sourcecode_analysis.md)
  - 基于公开传播的 Claude Code 源码泄露相关分析资料
  - 聚焦 query.ts、QueryEngine.ts、Tool.ts 等核心机制
  - 提炼 YOLO Classifier、权限管道、压缩策略、记忆系统等工程思想

### 2. 企业 AI 规划扩展报告

- [`enterprise_ai_platform_architecture.md`](./enterprise_ai_platform_architecture.md)
  - 企业 AI 平台总体架构设计
  - 从能力底座、工具总线、知识体系、治理体系到运营体系做全景规划

- [`pharma_ai_agent_governance_framework.md`](./pharma_ai_agent_governance_framework.md)
  - 制药/生物医药企业 AI Agent 治理框架
  - 聚焦 GxP、数据权限、审计追踪、验证与变更管理

- [`enterprise_mcp_tool_interface_spec.md`](./enterprise_mcp_tool_interface_spec.md)
  - 企业级 MCP / AI 工具接口规范
  - 聚焦工具接入标准、鉴权模型、审计要求、错误码与生命周期管理

---

## 推荐阅读路径

### 路径 A：先理解 Claude Code，再迁移到企业
1. Claude Code 架构设计深度解析
2. Claude Code 源码级架构深度解析
3. 企业 AI 平台总体架构设计

### 路径 B：直接用于企业 AI 建设规划
1. 企业 AI 平台总体架构设计
2. 企业级 MCP / 工具接口规范
3. 制药行业 AI Agent 治理框架

### 路径 C：用于高合规行业（制药/生物医药）
1. Claude Code 源码级分析
2. 制药行业 AI Agent 治理框架
3. 企业级 MCP / 工具接口规范

---

## 核心观点

### 1. 模型不是核心壁垒，Harness 才是
企业 AI 建设的真正竞争力，不在于是否接入了最新模型，而在于是否构建了稳定、可控、可审计的 Harness：
- 工具接口标准
- 权限治理机制
- 上下文管理机制
- 记忆与知识管理机制
- 可观测与审计体系

### 2. 企业 AI 首先要解决“可控”问题
在真实业务环境中，尤其是制药、医疗、金融等高合规行业，AI 系统不能只追求能力，更要追求：
- 可解释
- 可追溯
- 可审计
- 可验证
- 可回退

### 3. 工具标准化是企业 AI 规模化的前提
如果没有统一的工具接口标准，AI 只能停留在“聊天”层；只有把 ERP、LIMS、MES、OA、知识库、流程系统封装成统一工具协议，企业 AI 才能进入真正可执行阶段。

---

## 适用人群

- 企业 CIO / IT 负责人
- AI 平台架构师
- Agent 产品经理
- 数字化/智能化转型负责人
- 制药/生物医药企业 IT 与合规团队

---

## 仓库特点

- **面向企业落地**：不是概念综述，而是面向规划建设的工程化分析
- **兼顾理论与源码**：既有架构抽象，也有源码级机制提炼
- **强调治理与合规**：尤其适合高合规行业参考
- **可持续扩展**：后续可继续增加 AI 治理、提示词工程、知识工程、验证体系等专题报告

---

## 后续规划

- 增补企业 AI 权限模型设计专题
- 增补企业 AI 审计与可观测性体系专题
- 增补 GxP 场景下 AI 验证方法专题
- 增补 AI Agent 组织级运营管理机制专题

---

## License

本仓库文档采用 **CC BY 4.0** 许可协议。

你可以在保留署名的前提下分享和改编这些内容，详见 [`LICENSE`](./LICENSE)。

---

## English

For English readers, see [`README_EN.md`](./README_EN.md).
