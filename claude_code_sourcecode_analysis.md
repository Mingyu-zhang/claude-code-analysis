# Claude Code 源码级架构深度解析
## 基于 512,000 行泄露源码的工程化思想提炼

> **文档类型**：源码级研究报告  
> **版本**：2.1（源码级增强版）  
> **发布日期**：2026-04-02  
> **适用对象**：工程负责人、AI 平台架构师、安全治理团队、Agent 基础设施研发团队  
> **数据来源**：Claude Code v2.1.88 泄露源码（2026-03-31 npm Source Map 事件）  
> **核心主题**：QueryEngine、Tool 框架、权限管道、上下文压缩、系统提示缓存、记忆系统

### 摘要
本文基于公开传播的 Claude Code 源码泄露分析资料，从工程实现层面对其 Agent Loop、工具体系、权限模型、上下文压缩、系统提示缓存和记忆机制进行拆解。相较于理论层报告，本报告更强调“代码里到底是怎么实现的”，以及这些实现方式对企业 AI 平台设计意味着什么。

### 建议阅读方式
- 如果你想看 **源码级工程信号**，优先阅读本报告
- 如果你想先建立整体认知，再深入实现细节，建议先读《Claude Code 架构设计深度解析》
- 如果你要把这些机制迁移到企业场景，建议继续阅读权限、审计和平台架构专题

---

## 零、泄露事件背景


### 事件经过
2026 年 3 月 31 日，Anthropic 旗下明星产品 Claude Code 因构建配置失误，将完整的 **TypeScript Source Map 文件**打包进了 npm 发布包。Source Map 文件指向了一个可公开访问的 Cloudflare R2 存储桶 URL，导致 **51.2 万行完整未混淆 TypeScript 源代码**被任何人下载。

### 泄露规模

| 指标 | 数量 |
|------|------|
| 源码文件数 | 1,900+ 个 TypeScript 文件 |
| 代码总量 | 512,664 行 |
| 运行时 | Bun |
| UI 框架 | React + Ink（终端 UI） |
| 核心单文件最大 | QueryEngine.ts（46,000 行 / 785KB） |
| 工具数量 | ~40 种工具 |
| 命令数量 | ~85 条斜杠命令 |

### 失误根因（对企业 IT 的安全启示）
```
构建配置错误
   ↓ 生产包包含了 *.map 文件
   ↓ map 文件指向可公开访问的 R2 存储桶
   ↓ 任意用户可通过 npm 包的 sourceMappingURL 字段下载完整原始代码
```
> **教训**：Source Map 是调试利器，但属于开发环境专属资产。生产构建必须明确剔除或不生成 map 文件。这是 SDLC（软件交付生命周期）中的基础安全卫生要求。

---

## 一、源码目录全景

```
claude-code/
├── src/
│   ├── query.ts           # 785KB，Agent Loop 核心心脏（"胖核心"设计）
│   ├── QueryEngine.ts     # 46,000 行，LLM API 引擎 + 流式处理 + 工具循环
│   ├── Tool.ts            # 29,000 行，工具系统 + 权限模型
│   ├── commands.ts        # 25,000 行，斜杠命令实现
│   ├── forkSubagent.ts    # Fork 子代理管理
│   ├── AgentTool.ts       # 子代理作为工具的实现
│   ├── tools/             # 各工具具体实现
│   ├── services/
│   │   └── compact/       # 三层上下文压缩策略
│   ├── skills/            # Skills 系统
│   ├── buddy/             # 未发布 BUDDY 数字宠物（特性标志控制）
│   └── memory/            # AutoDream 记忆巩固系统
├── package.json
└── ...
```

**设计信号**：`query.ts` 约 785KB 是最大文件，采用"胖核心"模式——核心循环与调度逻辑集中于一处，避免了多模块调用的间接性，显著降低了调试复杂度。

---

## 二、Agent Loop 源码级解析

### 2.1 核心数据结构：消息即状态

```typescript
// 系统唯一状态：一个消息数组
type ConversationState = {
  messages: Message[];   // 追加式，永不原地修改
  systemPrompt: string;  // 分层缓存（静态前缀 + 动态后缀）
}

// 静态/动态边界标记（发现于源码）
const BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__';
// 静态前缀：可跨会话 cache → 降低 API 成本
// 动态后缀：当前会话特定信息（git状态、CLAUDE.md、记忆文件等）
```

### 2.2 核心循环：QueryEngine 的 run_turn

源码揭示的实际循环结构（伪代码还原）：

```typescript
// QueryEngine.ts 核心 while 循环（约88行关键路径）
async function* runTurn(
  messages: Message[],
  tools: Tool[],
  config: QueryConfig
): AsyncGenerator<TurnEvent> {

  // 安全阀：防止无限循环
  const safetyConfig = {
    maxTurns: config.maxTurns ?? DEFAULT_MAX_TURNS,
    maxBudgetUsd: config.maxBudgetUsd,  // 预算上限（美元）
  };

  while (true) {
    // 1. 构建请求：系统提示 + 消息历史 + 工具 Schema
    const request = buildRequest(messages, tools, systemPrompt);

    // 2. 流式调用 LLM API
    const stream = await callClaudeAPI(request);

    // 3. 实时解析流式响应（流式优先设计）
    for await (const chunk of stream) {
      if (chunk.type === 'content_block') {
        yield { type: 'text', content: chunk.text };
      }
      if (chunk.type === 'tool_use') {
        // 工具调用：不等待整条消息，立即执行！
        const result = await executeToolWithPermission(chunk);
        messages.push({ role: 'tool', result });
      }
    }

    // 4. 检查终止条件（由 LLM 的 stop_reason 决定）
    if (stream.stop_reason === 'end_turn') break;
    if (messages.length >= safetyConfig.maxTurns) break;
  }
}
```

**关键设计决策揭示**：
1. **流式优先**：工具调用在模型响应流式返回时就立即执行，不等完整消息
2. **安全阀双保险**：`maxTurns` + `maxBudgetUsd` 双重防止失控循环
3. **中断控制**：每个工具调用绑定独立 `AbortController`，用户 `Esc` 可随时取消

### 2.3 系统提示词的分层缓存架构

```
System Prompt 结构（源码发现）
├── [静态前缀] ── 可跨会话 Cache（缓存命中 → 降低 Token 成本）
│   ├── 角色定义与核心约束
│   ├── 工具使用规范
│   └── 安全行为约束
├── ── BOUNDARY 标记 ──
└── [动态后缀] ── 会话特定，每次重建
    ├── 当前 git 状态
    ├── CLAUDE.md 内容（项目级配置）
    ├── 记忆文件内容
    └── 当前会话上下文
```

**工程价值**：静态前缀不变可被缓存，每次 API 调用节省大量 token 费用。这是 Anthropic 在生产环境中的成本优化实践。

---

## 三、工具系统源码深度解析

### 3.1 工具接口契约（Tool.ts 揭示）

```typescript
// 每个工具通过 buildTool() 工厂函数创建
interface Tool {
  name: string;
  description: string;       // LLM 理解工具用途的关键
  inputSchema: JSONSchema;   // 契约：LLM ↔ 实现层的接口规范

  // 生命周期方法（安全默认值设计）
  isEnabled: () => boolean;
  isConcurrencySafe: () => boolean;  // 默认 false → 串行，安全优先
  isReadOnly: () => boolean;         // 默认 false → 触发权限检查
  checkPermissions: (args) => 'allow' | 'deny' | 'ask';
  execute: (args: ValidatedArgs) => Promise<ToolResult>;
}
```

**"安全默认值"哲学**：新增工具时，默认不并发安全、默认非只读，确保新工具不会意外绕过安全机制。要获取更高权限必须显式声明。

### 3.2 并行执行引擎：StreamingToolExecutor

```typescript
// 源码揭示的并行调度逻辑
class StreamingToolExecutor {
  async execute(toolCalls: ToolCall[]): Promise<ToolResult[]> {
    const groups = this.groupByConcurrencySafety(toolCalls);

    const results = [];
    for (const group of groups) {
      if (group.isConcurrencySafe) {
        // 只读工具 → 并行执行，最大10个并发
        results.push(...await Promise.all(group.calls.map(exec)));
      } else {
        // 写操作 → 串行执行，保证顺序一致性
        for (const call of group.calls) {
          results.push(await exec(call));
        }
      }
    }
    return results;
  }
}
```

**实际效果**：多个 `GlobTool`/`GrepTool` 搜索调用并行执行，文件搜索速度显著提升。写操作严格串行，保证数据一致性。

### 3.3 完整工具生态（源码揭示的约 40 种工具）

```
┌─────────────────────────────────────────────────┐
│  文件操作层                                       │
│  FileReadTool / FileEditTool / MultiFileTool     │
├─────────────────────────────────────────────────┤
│  搜索发现层                                       │
│  GlobTool / GrepTool / LSPTool                  │
├─────────────────────────────────────────────────┤
│  系统执行层                                       │
│  BashTool（核心）/ PowerShellTool（Windows预览） │
├─────────────────────────────────────────────────┤
│  网络交互层                                       │
│  WebFetchTool / WebSearchTool                   │
├─────────────────────────────────────────────────┤
│  任务管理层                                       │
│  TodoWriteTool / DiagnosticsTool                │
├─────────────────────────────────────────────────┤
│  多代理层                                         │
│  AgentTool（子代理）/ MCPTool（外部集成）         │
└─────────────────────────────────────────────────┘
```

**发现亮点**：`LSPTool` 是源码中发现的一个重要工具——集成了语言服务器协议（Language Server Protocol），意味着 Claude Code 可以利用 IDE 级别的代码理解能力（类型检查、定义跳转等），而不仅仅是文本搜索。

---

## 四、权限系统源码级还原

### 4.1 六层权限管道（纵深防御）

```
工具调用请求
     ↓
[Layer 1] 规则匹配（settings.json allow/deny/ask 规则）
     ↓ 未匹配
[Layer 2] 工具特定 checkPermissions（每个工具自定义）
     ↓ 未决
[Layer 3] PreToolUse Hooks（用户/系统自定义钩子介入）
     ↓ 通过
[Layer 4] 硬编码安全检查（.git目录、系统配置等，不可绕过）
     ↓ 不触发
[Layer 5] YOLO Classifier AI分类器（auto模式，两阶段审查）
     ↓ 不确定
[Layer 6] 用户提示（最终人工确认）
```

### 4.2 五种权限模式（源码发现）

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `default` | 敏感操作询问用户 | 日常交互（最安全） |
| `acceptEdits` | 自动批准文件编辑，其他询问 | 开发加速 |
| `bypassPermissions` | 自动批准一切（硬编码安全检查仍生效） | CI/CD 自动化 |
| `dontAsk` | 自动拒绝所有需要询问的操作 | 严格受控环境 |
| `auto` | **AI 分类器自动审批**（YOLO Classifier） | 平衡安全与自动化 |

### 4.3 YOLO Classifier：AI 审批 AI（源码揭示）

这是源码中最有趣的发现之一：

```typescript
// auto 模式下，使用独立的、更保守的 AI 模型来审批工具调用
class YOLOClassifier {
  async approve(toolCall: ToolCall): Promise<'allow' | 'deny' | 'ask'> {
    // 两阶段审查
    const quickJudgment = await this.fastModel.classify(toolCall);
    if (quickJudgment.confident) return quickJudgment.decision;

    // 快速判断不确定时，启用深度思考
    const deepAnalysis = await this.thinkingModel.analyze(toolCall);
    return deepAnalysis.decision;
  }
}
```

**设计哲学**：用 AI 来监督 AI，在安全性与自动化效率之间找到平衡点。

### 4.4 权限邮箱模式（Mailbox Pattern）

```typescript
// Tool.ts 中的权限邮箱机制
// 防止多个 Worker 子代理同时处理同一权限请求（原子性）
const createResolveOnce = () => {
  let resolved = false;
  return (decision: PermissionDecision) => {
    if (resolved) return;  // 幂等：已决定不重复处理
    resolved = true;
    mailbox.send(decision);
  };
};
```

在多代理协作时，权限决策需要原子性：一旦某个子代理对某个操作做出了权限决定，其他子代理不能重复发起相同请求。

---

## 五、上下文压缩引擎：三层策略源码解析

### 5.1 三层压缩架构（services/compact/ 目录）

```
上下文使用量监控
        │
        ├─ 达到 MicroCompact 阈值
        │      ↓
        │  [MicroCompact]：本地化清理
        │  - 删除旧工具输出（FRC：Function Result Clearing）
        │  - 无 API 调用，零成本
        │
        ├─ 达到 AutoCompact 阈值（~92% 使用率）
        │      ↓
        │  [AutoCompact]：摘要式压缩
        │  - 保留 13,000 Token 缓冲区
        │  - 调用 Claude API 生成最多 20,000 Token 摘要
        │  - 内置熔断机制（3次失败后停止，防止死循环）
        │
        └─ 紧急情况
               ↓
           [FullCompact]：深度压缩
           - 保留最近访问的文件（每文件 ≤ 5,000 Token）
           - 保留活跃计划和 TodoList
           - 保留已使用的技能模式
           - 深度重构上下文组织
```

### 5.2 FRC（Function Result Clearing）机制

```typescript
// 旧工具结果清除，只保留最近 N 个
// 强制模型主动将关键信息"记入"响应文本，而非依赖工具历史
function clearOldFunctionResults(
  messages: Message[],
  keepLast: number = 10
): Message[] {
  const toolResults = messages.filter(m => m.role === 'tool');
  const toRemove = toolResults.slice(0, -keepLast);
  return messages.filter(m => !toRemove.includes(m));
}
```

**设计意图**：强制模型不能"懒惰地"依赖历史工具输出，必须主动将重要信息提炼到回复中。这降低了上下文膨胀的速度。

---

## 六、多智能体架构：四种 Spawn 模式

### 6.1 四种子代理生成模式

| 模式 | 隔离级别 | 上下文共享 | 适用场景 |
|------|----------|-----------|---------|
| `default` | 同进程 | 共享 messages[] | 简单子任务委派 |
| `fork` | 独立进程 | 全新 messages[] | 研究探索（不污染主上下文） |
| `worktree` | Git Worktree + fork | 独立文件系统 | 并行功能开发 |
| `remote` | 容器隔离 | 完全独立 | Claude Code Remote / CI |

### 6.2 Fork 子代理的隔离设计

```typescript
// forkSubagent.ts 核心思想
class ForkSubagent {
  constructor(task: Task) {
    // 关键：全新的、空白的消息历史
    // 探索性任务的工具输出永远不会污染主代理上下文
    this.messages = [];
    this.parentMessages = null;  // 刻意不继承父上下文
  }

  async run(): Promise<SubagentResult> {
    // 在独立进程中执行
    // 主代理可以继续与用户交互（非阻塞）
    return await executeInBackground(this.task);
  }
}
```

### 6.3 Swarm 模式：去中心化多代理协作

```typescript
// Swarm 模式（源码中发现，通过特性标志控制）
class SwarmCoordinator {
  async orchestrate(tasks: Task[]): Promise<void> {
    const taskBoard = new SharedTaskBoard(tasks);

    // 子代理自主认领任务（非中央分配）
    await Promise.all(
      this.agents.map(agent => agent.scanAndClaim(taskBoard))
    );
  }
}

// 代理间通信：Send Message Protocol
// 统一的请求-响应模式驱动代理间协商
interface AgentMessage {
  type: 'permission_request' | 'status_update' | 'result';
  agentId: string;
  payload: unknown;
}

// 进程内上下文隔离（关键技术）
// AsyncLocalStorage 让每个代理拥有独立的上下文，即使运行在同一进程内
const agentContext = new AsyncLocalStorage<AgentContext>();
```

---

## 七、Skills 系统：知识的工程化封装

### 7.1 Skills 的本质

Skills 是将复杂领域知识和多步操作封装为**可按需加载的知识单元**：

```typescript
// Skills 不是预加载到 System Prompt，而是在需要时通过 tool_result 注入
// 这保持了 System Prompt 的简洁，节约 token 成本

interface Skill {
  name: string;
  description: string;           // 触发条件描述
  validation: () => boolean;     // 环境适用性检查
  examples: Example[];           // 少样本示例
  executionSteps: Step[];        // 可执行步骤序列
}

// 加载时机：模型识别到需要某个 Skill 时，动态注入到上下文底部
// 上下文底部 = 高注意力区域 → 模型更容易遵循
```

### 7.2 CLAUDE.md：活文档作为记忆系统

```
CLAUDE.md 的四种记忆层级（源码揭示）

[L1] ~/.claude/CLAUDE.md         ← 用户全局记忆（跨所有项目）
[L2] ./CLAUDE.md                  ← 项目级记忆（团队共享，版本控制）
[L3] ./CLAUDE.local.md            ← 本地个人记忆（.gitignore排除）
[L4] 运行时 memory 文件            ← 会话级动态记忆
```

---

## 八、AutoDream：仿生记忆巩固系统

### 8.1 触发条件（精确还原）

```typescript
// 后台自动触发，所有条件必须同时满足
const shouldRunDream = (state: MemoryState): boolean => {
  return (
    timeSinceLastConsolidation >= 24 * HOURS &&  // 距上次巩固 ≥ 24小时
    newSessionsSinceLastRun >= 5 &&               // 新会话 ≥ 5次
    !isOtherConsolidationRunning() &&             // 无并发巩固进程
    timeSinceLastScan >= 10 * MINUTES             // 距上次扫描 ≥ 10分钟
  );
};
```

### 8.2 四阶段巩固流程（Orient-Gather-Consolidate-Prune）

```
[Orient]   → 读取 MEMORY.md，扫描现有记忆文件，确定巩固范围
     ↓
[Gather]   → 检查日志，找出过时记忆，汇总相关会话
     ↓
[Consolidate] → 合并、更新、解决矛盾记忆（LLM辅助摘要）
     ↓
[Prune]    → 裁剪：MEMORY.md 保持 ≤ 200 行 / 25KB
```

**仿生学灵感**：模拟人类睡眠时的记忆巩固过程（慢波睡眠）——在低活跃期整理、压缩、强化重要记忆，清除无关记忆。

---

## 九、未发布功能揭示（特性标志控制）

### 9.1 内部特性标志（Feature Flags）

```typescript
// 源码发现的未发布功能开关
const FEATURE_FLAGS = {
  PROACTIVE: false,      // 主动模式：AI 主动发现问题并提出
  VOICE_MODE: false,     // 语音输入支持
  BRIDGE_MODE: false,    // IDE 桥接模式
  KAIROS: false,         // 未知高级功能（名字来自希腊语"关键时刻"）
  SWARM: false,          // Swarm 多代理协作模式
  BUDDY: false,          // 数字宠物系统
};
```

### 9.2 BUDDY 数字宠物系统

```
buddy/
├── pet.ts         # 宠物状态机（稀有度、闪光变种、属性）
├── display.ts     # 终端渲染
└── events.ts      # 触发事件（代码提交、任务完成等）
```

发布计划（源码注释发现）：2026年4月1-7日预热，5月正式发布。

---

## 十、从源码提炼的工程化铁律（升级版）

### 对比：理论 vs 源码实证

| 设计铁律 | 理论层面 | 源码实证 |
|---------|---------|---------|
| 单一主循环 | 提倡单一主循环 | `query.ts` 785KB 单文件"胖核心"，所有调度在此 |
| 消息即状态 | 状态=消息数组 | `ConversationState = {messages: Message[]}` 追加式，永不原地修改 |
| 流式优先 | 提及流式 | `StreamingToolExecutor` 模型返回流中立即执行工具，不等完整响应 |
| 安全默认值 | 权限分级 | `isConcurrencySafe: () => false`，`isReadOnly: () => false`，所有工具默认最保守 |
| 失败即反馈 | 错误作反馈 | 熔断机制（autoCompact 3次失败后停止），错误作为 ToolResult 继续循环 |
| 上下文工程 | 六大支柱 | 三层压缩 + FRC + 分层 System Prompt 缓存 + AutoDream 记忆巩固 |

### 新发现：源码独有揭示

1. **AI 监督 AI**（YOLO Classifier）：用独立保守的 AI 模型审批另一个 AI 的工具调用
2. **AsyncLocalStorage 进程内隔离**：同一进程多代理运行，通过 Node.js 异步本地存储实现上下文隔离
3. **System Prompt 静/动分层**：通过 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记分割，静态部分可跨会话缓存
4. **LSP 集成**：工具层集成语言服务器协议，AI 可使用 IDE 级代码理解能力
5. **仿生记忆**：AutoDream 模拟人类睡眠记忆巩固，有精确的触发条件和四阶段流程

---

## 十一、对企业 AI 规划建设的源码级启示

### 11.1 架构层：可直接复用的模式

```
复用优先级 A（直接可用）：
├── 三层上下文压缩策略（MicroCompact/AutoCompact/FullCompact）
├── 分层记忆体系（全局/项目/本地/会话四级）
├── 工具接口契约（buildTool 工厂 + JSON Schema）
└── 安全默认值原则（新工具默认最保守权限）

复用优先级 B（需适配）：
├── 权限管道六层设计 → 适配企业 RBAC/ABAC 体系
├── Fork 子代理隔离 → 适配企业多系统隔离需求
├── AutoDream 记忆巩固 → 适配企业知识库更新机制
└── 特性标志系统 → 适配企业灰度发布流程

复用优先级 C（参考思想）：
├── YOLO Classifier → 企业 AI 操作的合规审批自动化
├── Swarm 模式 → 企业大规模并行 AI 任务编排
└── LSP 集成 → 专业领域工具的深度集成（如 LIMS 系统的语义理解）
```

### 11.2 制药行业专项映射

| Claude Code 机制 | 制药 IT 应用 |
|----------------|-------------|
| 权限六层管道 | GxP 数据操作的多层审批链 |
| YOLO Classifier | AI 操作的合规性自动分类（GMP/GxP）|
| FRC（旧结果清除）| 监管提交历史的版本管理策略 |
| AutoDream 记忆巩固 | 知识管理系统的定期归档与整理 |
| CLAUDE.md 分层 | 企业规范文档的分层（公司级/部门级/项目级）|
| 特性标志 | 合规验证前的功能隔离（IQ/OQ/PQ 阶段管控）|

### 11.3 安全卫生：从泄露事件学到什么

```
Source Map 泄露事件 → 对应企业 AI 建设的安全卫生清单：

□ AI 系统的 System Prompt 是否有防泄露保护？
□ AI 工具接口是否有完整的审计日志？
□ AI 产出物（代码/文档）是否有版本控制和变更追踪？
□ AI 使用的凭证（API Key）是否与代码严格分离？
□ AI Agent 的权限边界是否经过定期审查？
□ 构建和发布流程是否有安全检查门禁？
```

---

## 十二、总结：源码揭示的底层真相

> **"模型是代理（Agent），代码是驾驭（Harness）。构建好的 Harness，代理自会完成其余。"**  
> —— Claude Code 源码注释

### 三个底层真相

```
┌──────────────────────────────────────────────────────────────────┐
│  真相一：复杂来自边界，不来自核心                                   │
│  核心 Agent Loop 极简（88行关键路径），复杂性在12层 Harness 包装中   │
├──────────────────────────────────────────────────────────────────┤
│  真相二：安全来自纵深，不来自单点                                   │
│  六层权限管道 + AI 分类器 + 硬编码安全检查，任何单层被绕过不致命      │
├──────────────────────────────────────────────────────────────────┤
│  真相三：可靠来自工程，不来自模型                                   │
│  上下文压缩 + FRC + 记忆巩固 + 分层缓存，工程基础设施决定了可靠性    │
└──────────────────────────────────────────────────────────────────┘
```

**对企业 AI 规划的终极建议**：先花 80% 精力构建 Harness（权限、上下文、工具、记忆），再花 20% 精力选模型。顺序错了，再好的模型也不可靠。

---

*数据来源：*
- *Claude Code v2.1.88 泄露源码（2026-03-31）*
- *CSDN：逆向深扒 Claude Code 源码（2026-04-01）*
- *HuggingFace Forums：Production AI Architecture Patterns（2026-03-31）*
- *DEV Community：Claude Code Source Leak Analysis（2026-03-31）*
- *青稞社区：Claude Code 源码逆向工程与系统性分析（2026-04-02）*
- *GitHub hjdhnx/analysis_claude_code（Claude Code v1.0.33 逆向研究）*
