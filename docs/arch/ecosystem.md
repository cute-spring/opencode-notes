# 生态映射与技术栈 (Ecosystem & Tech Stack)

本模块探讨了如何在其他生态（如 Python）中复刻 OpenCode 的架构，并详细解析了项目所依赖的关键技术栈。

## 6.1 跨语言架构映射：Python 生态实现方案

### 6.1.1 核心推荐方案：PydanticAI + LiteLLM

对于希望在 Python 生态中复刻 OpenCode 架构的开发者，**PydanticAI** 是目前最接近的设计方案。

- **模型中立性**：结合 **LiteLLM** 统一调用 100+ 模型提供商。
- **结构化对话与状态**：使用 Pydantic 模型定义工具参数和 Agent 响应。
- **依赖注入与上下文管理**：利用 `Deps` 机制模拟 `TaskTool` 的上下文注入。

### 6.1.2 维度对比

| 维度 | OpenCode (TS/Node.js) | Python 生态推荐实现 |
| :--- | :--- | :--- |
| **工具协议** | MCP | **MCP Python SDK** |
| **核心编排** | ACP + Processor | **PydanticAI** |
| **模型调用** | Vercel AI SDK | **LiteLLM** |
| **类型约束** | Zod | **Pydantic v2** |
| **异步/流式** | Node.js Streams | **Asyncio + HTTPX** |

### 6.1.3 代码示例

```python
from pydantic_ai import Agent, RunContext
from pydantic import BaseModel

class FileSearchArgs(BaseModel):
    pattern: str
    path: str = "."

search_agent = Agent(
    'openai:gpt-4o',
    deps_type=dict,
    result_type=str,
    system_prompt="你是一个文件搜索专家..."
)

@search_agent.tool
async def glob_search(ctx: RunContext[dict], args: FileSearchArgs) -> list[str]:
    return ["file1.ts", "file2.ts"]
```

## 6.2 关键技术栈与依赖解析 (Key Tech Stack & Dependencies)

### 6.2.1 AI 与智能体核心 (AI & Agent Core)
- **`ai` (Vercel AI SDK)**: 处理 LLM 流式响应、工具调用的核心引擎。
- **多模型适配器 (@ai-sdk/*)**: 模型无关架构。
- **`zod`**: 声明式 Schema 验证。
- **`ulid`**: 唯一标识符生成。

### 6.2.2 前端展现层 (Frontend & UI)
- **`solid-js`**: 核心 UI 库，极致响应式性能。
- **`@solidjs/start`**: 全栈框架。
- **`virtua`**: 高性能虚拟列表。
- **`@pierre/diffs`**: 代码对比渲染引擎。

### 6.2.3 后端与运行时 (Backend & Runtime)
- **`hono`**: 高性能 Web 框架。
- **`@aws-sdk/client-s3`**: 存储服务。
- **内部工作空间组件**: `sdk`, `plugin`, `script`。

### 6.2.4 辅助工具与基础设施 (Utilities & Infra)
- **`fuzzysort`**: 模糊搜索。
- **`turbo` (Turborepo)**: Monorepo 构建系统。
- **`sst` (Serverless Stack)**: IaC 框架。

---

> **教授箴言**
>
> “技术栈的选择应服务于架构模式，而非相反。OpenCode 选择了 SolidJS 和 Node.js，是为了在 Web 生态中实现极致的响应速度与高并发处理。理解了背后的‘分治’与‘协议驱动’思想，你可以在任何现代编程语言中复刻这种智能。”
