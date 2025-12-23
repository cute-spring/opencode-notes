版本：OpenCode AI IDE 架构分析专用指令 (v1.0)

---

角色：你是一名专精于 AI 开发工具（AI IDE）与 Agent 架构的资深技术专家。

背景：我们需要对 OpenCode 这个 AI IDE 项目进行深度架构分析，重点关注其 LLM 集成、Agent 设计、ACP (Agent Client Protocol) 实现以及 IDE 前后端交互机制。

---

输出与文件要求：
- **文件创建**：必须在 `docs/` 目录下新建一个名为 `opencode-architecture-analysis.md` 的文件。
- **文档引用**：引用代码时必须使用 `file_path:line_number` 格式。
- **图表**：使用 Mermaid 绘制架构图、时序图和状态图。

---

工作方法与执行策略：

1. **核心领域扫描**：
   - **Agent 核心 (`packages/opencode/src/agent/`)**：分析 Agent 的定义 (`Info` schema)、权限系统 (`permission`)、工具集 (`tools`)、以及内置 Agent (`build`, `plan`, `explore` 等) 的实现。
   - **ACP 协议 (`packages/opencode/src/acp/`)**：分析 Agent Client Protocol 的实现，包括会话管理 (`session`), 事件订阅 (`setupEventSubscriptions`), 以及工具调用转换逻辑。
   - **LLM 集成 (`packages/core/` 或 `packages/opencode/`)**：查找 `Provider` 抽象、模型选择逻辑 (`ModelKey`)、以及与 Vercel AI SDK (`ai` package) 的集成方式。
   - **IDE 客户端 (`packages/desktop/`)**：分析前端如何与 Agent 交互 (`context/local.tsx`, `hooks/use-providers.ts`)，以及 Agent 状态的 UI 呈现。
   - **SDK (`packages/sdk/`)**：分析客户端 SDK 如何封装核心能力。

2. **文档结构 (必须包含以下章节)**：

   **1. 执行摘要**
   - OpenCode 的核心价值主张。
   - 整体架构风格 (Monorepo, SolidJS + Vite, Node.js Core)。

   **2. 系统全景架构 (System Architecture)**
   - **组件图**：展示 `Desktop` (IDE UI), `OpenCode Core` (Agent Runtime), `Console` (Backend), `ACP` 之间的关系。
   - **数据流**：描述用户输入 -> IDE -> Agent -> LLM -> Tool Execution -> UI Update 的完整链路。

   **3. Agent 设计与实现 (Agent Design)**
   - **Agent 定义**：详细解析 `Agent.Info` 结构 (Name, Mode, Tools, Permissions)。
   - **权限系统**：分析 `edit`, `bash`, `webfetch` 等权限的粒度控制与默认策略 (Allow/Ask/Deny)。
   - **内置 Agent**：列出所有内置 Agent (如 `build`, `plan`, `explore`) 及其专用 Prompt (`packages/opencode/src/agent/prompt/`) 和适用场景。
   - **Subagents**：分析 Subagent 模式 (`mode: "subagent"`) 的工作原理。

   **4. LLM 集成与模型管理 (LLM Integration)**
   - **Provider 抽象**：支持的 Providers (OpenCode Zen, Anthropic, OpenAI 等) 及其配置方式。
   - **模型选择策略**：如何根据任务选择模型 (Default vs Configured vs Recent)。
   - **Prompt Engineering**：分析系统提示词 (`SystemPrompt`) 的构建方式。

   **5. Agent Client Protocol (ACP) 详解**
   - **协议职责**：解释 ACP 在 Client 和 Agent 之间的桥梁作用。
   - **会话生命周期**：`Initialize` -> `NewSession` -> `LoadSession` 流程。
   - **事件驱动机制**：分析 `permission.updated`, `message.part.updated` 等事件的处理逻辑。
   - **工具调用转换**：如何将 LLM 的 Tool Call 转换为 ACP 的 Session Update。

   **6. 核心业务流程 (Key Workflows)**
   - **代码生成/修改流程**：从用户输入 Prompt 到文件被修改 (`edit` tool) 的详细时序图。
   - **上下文收集流程**：Agent 如何读取文件、获取 Diff、执行命令以获取上下文。

   **7. 安全与沙箱 (Security & Sandboxing)**
   - **权限拦截**：分析敏感操作 (如 `rm`, `edit`) 的拦截与用户确认机制。
   - **命令过滤**：分析 `bash` 权限中对特定命令 (如 `rm`, `mkfs`) 的限制逻辑。

   **8. 待确认事项与假设 (Open Questions & Assumptions)**
   - 针对未找到代码或逻辑不清晰部分的合理假设。
   - 需要进一步确认的设计细节。

---

关键代码路径指引 (参考)：
- Agent 定义: `packages/opencode/src/agent/agent.ts`
- ACP 实现: `packages/opencode/src/acp/agent.ts`
- Prompt 模板: `packages/opencode/src/agent/prompt/`
- IDE 模型上下文: `packages/desktop/src/context/local.tsx`
- 权限合并逻辑: `packages/opencode/src/agent/agent.ts` (mergeAgentPermissions)

---

请基于以上要求，生成一份深度技术分析文档。
