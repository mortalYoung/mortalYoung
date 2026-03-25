# Harness Project：基于 Agent 的项目长时间持续迭代的最佳实践

## 前言

随着 Claude 等大模型能力的不断提升，越来越多的团队开始尝试构建 Agent 应用。然而，当一个基于 Agent 的项目从原型走向长期运营，真正的挑战才刚刚开始：**如何让 Agent 应用在数周、数月乃至数年的持续迭代中始终保持可控、可维护、可演进？**

大多数团队在落地过程中都会遇到同样的困境：Agent 能力强大，却难以驾驭——它可能在某个环节偏离预期，也可能在没有约束的情况下产生不可预测的行为。随着项目持续迭代，这一问题会愈发严峻：需求在变，上下文在积累，Agent 的行为边界也需要不断调整。

**本文的核心观点是：基于 Agent 的项目，长时间持续迭代的最佳实践是将该项目 Harness 化。**

所谓"Harness 化"，即将项目纳入 **Harness Project** 架构范式进行管理。其核心定义为：

$$
\textit{Harness} = \textit{Agent} + \textit{State} + \textit{Reins}
$$

> 其中，$\textit{Agent}$ 是执行任务的智能体，$\textit{State}$ 是任务的上下文与记忆，$\textit{Reins}$ 是对 Agent 行为的约束与引导机制。三者缺一不可。

Harness 在英文中意为"马具"，其本质是将马的力量以可控的方式加以利用。这个比喻恰如其分：Agent 如同一匹强健的马，拥有强大的推理与执行能力；而 Harness Project 所做的，就是为这匹马配上合适的马具，让它在正确的方向上发挥力量。

## Agent：Harness 的执行核心

在 Anthropic 的 [agentic settings](https://docs.anthropic.com/en/docs/build-with-claude/agentic-and-multi-agent-frameworks) 定义中，Agent 是能够在较长时间范围内自主规划和执行多步骤任务的系统。它与传统 LLM 调用的根本区别在于：

- **Tool Use（工具调用）**：Agent 可以调用外部工具（搜索、代码执行、文件读写等）来完成任务，而不仅仅是生成文本。
- **Multi-step Reasoning（多步推理）**：Agent 能够将复杂任务拆解为子任务，并通过多轮循环逐步完成。
- **Autonomy（自主性）**：Agent 在给定目标后，能够自主决策每一步的行动，无需人工干预每个细节。

在 Harness 架构中，Agent 承担的是**执行**职责。它接收来自 Reins 的指令和约束，读取 State 中的上下文，并通过 Tool Use 完成具体操作。

值得注意的是，Anthropic 在其 [multi-agent frameworks](https://docs.anthropic.com/en/docs/build-with-claude/agentic-and-multi-agent-frameworks) 中提出了 **Orchestrator-Subagent** 模型：一个 Orchestrator Agent 负责任务规划与分发，多个 Subagent 负责具体执行。Harness Project 天然兼容这种模式——Orchestrator 和 Subagent 均受 Reins 约束，并共享同一套 State 体系。

## State：Harness 的记忆与上下文

Anthropic 将 Agent 的记忆形式分为四类：

| 类型 | 描述 | 示例 |
|------|------|------|
| **In-context storage** | 当前对话窗口内的临时信息 | 本次任务的输入、中间结果 |
| **External storage** | 持久化的外部存储 | 数据库、文件系统、知识库 |
| **In-weights storage** | 模型训练时内化的知识 | 模型的通用世界知识 |
| **In-cache storage** | KV Cache 缓存的计算结果 | 长文档的 Prompt Cache |

在 Harness Project 中，**State** 主要对应前两类：In-context storage 和 External storage。State 的作用是为 Agent 提供执行任务所需的全部上下文，包括：

- **Task State（任务状态）**：当前任务的目标、进度、中间产物。
- **World State（世界状态）**：与任务相关的外部环境信息，如文件内容、API 返回结果、数据库查询结果。
- **Conversation State（对话状态）**：用户与 Agent 之间的历史交互记录，用于维护连贯的对话上下文。
- **Agent State（智能体状态）**：Agent 自身的工作记忆，包括已完成的子任务、待执行的计划等。

State 的质量直接决定了 Agent 的决策质量。一个设计良好的 State 管理机制，能够让 Agent 在长时间、多步骤的任务执行中始终保持对全局目标的清晰认知，避免因上下文丢失而导致的"迷失"问题。

## Reins：Harness 的约束与引导

"Reins"（缰绳）是 Harness Project 中最关键、也最容易被忽视的部分。它回答的是：**如何确保 Agent 在正确的方向上行动？**

Anthropic 在其安全框架中特别强调了 Agent 的**最小权限原则（Principle of Least Privilege）**和**人机协作（Human-in-the-Loop）**机制。这些思想正是 Reins 的核心设计哲学。

Reins 的实现包含以下几个层面：

### 1. Prompt-level Reins（提示词层约束）

通过精心设计的 System Prompt，为 Agent 设定明确的角色边界、行为准则和禁止事项。这是 Reins 的第一道防线。

Anthropic 建议在 System Prompt 中明确定义：
- Agent 的目标和职责范围
- 允许调用的工具及其使用场景
- 遇到不确定情况时的处理策略（如：主动寻求人工确认）

### 2. Tool-level Reins（工具层约束）

对 Agent 可调用的工具进行严格的权限控制。Anthropic 的 **Computer Use** 功能展示了工具层约束的重要性——一个能够操作鼠标键盘的 Agent，如果缺乏工具层约束，其风险是难以想象的。

工具层约束的关键设计：
- **工具白名单**：只暴露当前任务所需的最小工具集。
- **操作审计**：记录每一次工具调用的输入与输出，便于追溯和审计。
- **副作用隔离**：对于具有写操作（如文件修改、数据库写入）的工具，增加确认或沙箱机制。

### 3. State-level Reins（状态层约束）

通过对 State 的读写权限控制，限制 Agent 可以访问和修改的信息范围。不同的 Subagent 只能访问其职责范围内的 State，避免信息越权访问。

### 4. Flow-level Reins（流程层约束）

在任务执行流程的关键节点设置**检查点（Checkpoint）**，触发人工确认或自动验证：

```
Task Start → [Plan Review] → Execute → [Result Verification] → [Human Approval] → Complete
```

Anthropic 将这种机制称为 **interruption and verification**，即在 Agent 执行高风险操作前主动暂停并等待人工确认。这是构建可信 Agent 系统的重要实践。

## Harness 的整体架构

将三个组件整合在一起，Harness Project 的架构如下：

```
┌─────────────────────────────────────────────────────────┐
│                      Harness Project                     │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │    Reins    │───▶│    Agent    │───▶│    State    │  │
│  │             │    │             │    │             │  │
│  │ • Prompt    │    │ • Orchestr. │    │ • Task      │  │
│  │ • Tools     │◀───│ • Subagents │◀───│ • World     │  │
│  │ • Flow      │    │ • Tool Use  │    │ • Convers.  │  │
│  └─────────────┘    └─────────────┘    └─────────────┘  │
│         │                  │                  │          │
│         └──────────────────┴──────────────────┘          │
│                    Human-in-the-Loop                      │
└─────────────────────────────────────────────────────────┘
```

三者形成一个闭环：
- **Reins** 约束 Agent 的行为边界，并通过 State 的权限控制限制其信息访问。
- **Agent** 读取 State 中的上下文，在 Reins 的约束下执行任务，并将结果写回 State。
- **State** 为 Agent 的决策提供依据，同时记录执行历史供 Reins 进行审计和验证。

## Advanced：Git Native Project

上述 Harness 架构解决了 Agent 的可控性问题，但还有一个深层次的工程挑战尚未解决：**State 的版本管理**。

在传统软件开发中，代码的版本管理由 Git 完成，这让我们可以追踪每一次变更、回滚到任意历史状态、并行开发多个特性。然而，在 Agent 应用中，State 的变更同样频繁且关键——但我们却缺乏对 State 进行版本管理的有效手段。

**Git Native Project** 提出了一个大胆的解法：

$$
\textit{State} = \textit{GitHub} \mid \textit{GitLab}
$$

即，**将 GitHub 或 GitLab 作为 Agent 的 State 存储后端**。这意味着：

### State 即 Repository

每一个 Harness Project 对应一个 Git 仓库。Agent 对 State 的每一次写操作，对应一次 Git Commit。Task State、World State、Conversation State 均以文件的形式存储在仓库中。

```
project-repo/
├── .harness/
│   ├── state/
│   │   ├── task.json          # Task State
│   │   ├── conversation.md    # Conversation State
│   │   └── agent-memory.json  # Agent State
│   ├── reins/
│   │   ├── system-prompt.md   # Prompt-level Reins
│   │   └── tools.json         # Tool-level Reins
│   └── audit/
│       └── tool-calls.log     # 工具调用审计日志
└── workspace/                 # Agent 的工作产物
```

### 版本管理带来的核心价值

**1. 完整的变更追踪（Full Audit Trail）**

每一次 Agent 行动都对应一个 Commit，Commit Message 记录了 Agent 的意图和执行的操作。这为 Reins 的审计功能提供了天然支撑：

```
feat(agent): 分析用户需求并生成初步方案

- 读取用户输入: "优化数据库查询性能"
- 调用 search_tool 检索相关文档
- 生成优化方案草稿，写入 workspace/proposal.md
```

**2. 状态回滚（State Rollback）**

当 Agent 执行出现偏差时，可以通过 `git revert` 将 State 回滚到任意历史节点，而无需重新开始整个任务。这极大降低了 Agent 执行失败的成本。

**3. 并行任务探索（Branch-based Exploration）**

利用 Git Branch，可以让 Agent 同时探索多个解决方案，最终通过 Pull Request 的形式合并最优结果。这种模式天然契合 Anthropic 的 **multi-agent parallelization** 实践——多个 Subagent 在各自的 Branch 上并行工作，Orchestrator 负责最终的合并决策。

```
main
├── feature/solution-a  ← Subagent A 探索方案 A
├── feature/solution-b  ← Subagent B 探索方案 B
└── feature/solution-c  ← Subagent C 探索方案 C
```

并行管理的关键在于：**多个 Agent 如何在同一仓库的不同 Branch 上同时工作，而互不干扰？** 答案是 **Git Worktree**。

`git worktree` 允许将同一个仓库的不同分支同时 checkout 到不同的文件系统目录，每个目录拥有独立的工作区，但共享同一套 Git 对象存储：

```
project-repo/              ← main 分支（主工作区）
project-repo-solution-a/   ← feature/solution-a（Subagent A 工作区）
project-repo-solution-b/   ← feature/solution-b（Subagent B 工作区）
project-repo-solution-c/   ← feature/solution-c（Subagent C 工作区）
```

每个 Subagent 在自己的 Worktree 目录中独立运行，无需切换分支，也不会相互阻塞。这使得并行 Agent 的资源隔离变得简单而高效。

**4. 协作与审查（Collaboration & Review）**

Pull Request 本身就是一种天然的 **Human-in-the-Loop** 机制：Agent 完成任务后提交 PR，人工审查员（或另一个 Agent）对变更进行 Code Review，确认无误后再合并到主分支。

这将软件工程中成熟的 Code Review 文化引入了 Agent 任务管理，使得人机协作更加自然和高效。

**5. 权限管理（Access Control）**

GitHub/GitLab 本身提供了完善的权限控制体系（Branch Protection、CODEOWNERS、Required Reviews 等），这些机制可以直接复用为 Reins 的权限层。

### Git Native Project 的适用场景

Git Native Project 特别适合以下场景：

- **代码生成与重构**：Agent 直接在代码仓库中工作，每次修改即为一个 Commit，天然融入现有的开发流程。
- **文档生成与维护**：将文档仓库作为 State，Agent 的每次更新都有完整的变更记录。
- **数据分析报告**：分析过程和结果以文件形式存储在仓库中，每一步分析都可追溯。
- **长期运行的自动化任务**：需要持久化 State 且要求高可靠性的 Agent 任务。

## 总结

**将 Agent 项目 Harness 化，是基于 Agent 的项目长时间持续迭代的最佳实践。** 这一结论来自于工程实践中的核心洞察：Agent 应用的生命周期远不止于初次搭建，更在于数月乃至数年的持续演进——而 Harness 架构正是为这种长周期迭代而设计的。

Harness Project 为 Agent 应用提供了一套完整的架构范式：

| 组件 | 职责 | 对持续迭代的价值 |
|------|------|---------|
| **Agent** | 执行任务 | 提供智能推理与工具调用能力，随模型升级持续增强 |
| **State** | 存储上下文 | 保证跨迭代周期的决策质量与任务连贯性 |
| **Reins** | 约束行为 | 在需求变化时灵活调整 Agent 的行为边界 |

而 **Git Native Project** 则在此基础上更进一步，将成熟的版本控制体系引入 Agent 的 State 管理，解决了版本追踪、状态回滚、并行探索和人机协作等一系列工程挑战——这些恰恰是长期迭代项目最需要的能力。

正如 Anthropic 在其 Agent 构建指南中所强调的：

> *"The key question is not whether to use AI agents, but how to build them in a way that is safe, reliable, and aligned with human intent."*

将项目 Harness 化，就是对这个问题的工程回答：通过 State 给 Agent 以记忆，通过 Reins 给 Agent 以边界，通过 Git Native 给 Agent 以历史——让 Agent 应用在长期迭代中始终可控、可维护、可演进。
