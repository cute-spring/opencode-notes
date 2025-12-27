# Prompt Engineering Insights (opencode)

本文基于以下目录中的系统提示词逐个解读与赏析，并抽取可复用的 prompt engineering 规律：

- [prompt](packages/opencode/src/agent/prompt)
  - [compaction.txt](packages/opencode/src/agent/prompt/compaction.txt)
  - [explore.txt](packages/opencode/src/agent/prompt/explore.txt)
  - [summary.txt](packages/opencode/src/agent/prompt/summary.txt)
  - [title.txt](packages/opencode/src/agent/prompt/title.txt)

以及同目录下与 prompt 组合/编排直接相关的两份实现：

- [agent.ts](packages/opencode/src/agent/agent.ts)
- [generate.txt](packages/opencode/src/agent/generate.txt)

这些 prompt 在系统中分别用于：会话压缩（上下文溢出时的“记忆整理”）、代码库探索子代理、PR 风格摘要、对话标题生成。它们共同体现了一种工程化风格：把 LLM 当作“可调用的组件”，用清晰的输出契约与安全边界来提升稳定性。

---

## 1) compaction.txt：会话压缩 / 续聊交接（handoff）

**文件**

- [compaction.txt](packages/opencode/src/agent/prompt/compaction.txt)

**它要解决的真实问题**

- 当上下文窗口接近溢出时，需要把“对话态”压缩成一段可继续工作的摘要。
- 目标不是“写得好看”，而是“把后续继续工作所需的信息打包出来”。

**结构拆解（为什么这样写）**

- 角色定位：开场一句把任务钉死为 summarizing conversations。
- 关注点列表：用项目管理视角列出续聊必需字段：
  - 已完成什么、正在做什么、改哪些文件、下一步做什么
  - 持久化约束（用户偏好/限制）
  - 关键技术决策及理由
- 最后一条定义“信息密度目标”：既要全面又要快速理解。

**赏析：这份 prompt 的工程味**

- 它把“摘要”从文学任务变成“状态快照（state snapshot）”。你可以把它理解为：把 agent 的短期工作记忆写成交接文档。
- 关注点列表里包含“决策与理由”，这是避免后续反复推翻设计的关键；很多摘要 prompt 只写“做了什么”，但不写“为什么”。

**潜在问题 / 失效模式**

- 缺少输出格式约束：模型可能写成长段落、夹叙夹议，导致“可扫描性”下降。
- 缺少“未解决问题 / 风险”字段：当任务卡住或存在不确定性时，后续 agent 可能无法迅速定位阻塞点。
- 没有明确禁止“复制冗余对话”或“粘贴工具输出”，在极端情况下会浪费 token。

**可迁移的技巧（你可以复用到自己的系统）**

- 把“摘要”改写成“交接包”：关注后续行动需要的信息，而不是复述。
- 让摘要包含“持久化约束”：它相当于把用户偏好写入长期记忆。

**一个最小对比实验（建议你自测）**

- 你觉得，如果把“Which files are being modified”删掉，会怎样？
  - 常见后果：后续很难决定该读取哪些文件、diff 范围变大、探索成本上升。

**与实现的对应关系（为什么它尤其重要）**

- compaction agent 在系统里是“隐藏的 primary agent”，且工具被全禁用（只输出文本）[agent.ts](packages/opencode/src/agent/agent.ts#L166-L177)。
- 系统在触发 compaction 时，会额外注入一条 user 指令要求提供可续聊的详细提示 [compaction.ts](packages/opencode/src/session/compaction.ts#L109-L175)，这会与系统 prompt 形成“同向加固”（redundant alignment）。

---

## 2) explore.txt：代码库探索子代理（search specialist）

**文件**

- [explore.txt](packages/opencode/src/agent/prompt/explore.txt)

**它要解决的真实问题**

- 在大型代码库里快速定位文件、符号与实现路径，并把结果以可用的形式汇报出来。

**结构拆解（为什么这样写）**

- 角色定位：file search specialist，明确“你不是写业务逻辑的 agent”。
- 能力清单：Glob / regex search / Read & analyze。
- 操作指南：把“任务类型 → 工具选择”编码为可执行策略。
- 安全约束：
  - 结果输出必须是绝对路径
  - 不用 emoji
  - 不创建文件，不执行会修改系统状态的命令

**赏析：这是一个“策略 prompt”，不是一个“任务 prompt”**

- 它的价值不在于一次性回答具体问题，而在于把“探索行为”固化成可复用流程。
- 这种 prompt 适合被主 agent 当作子模块调用：主 agent 把问题交给它，它返回路径与发现，主 agent 再做决策。

**潜在问题 / 失效模式**

- 指令存在轻微张力：
  - 一方面说可以用 Bash 做 copy/move/list；另一方面又禁止任何会修改系统状态的 bash（copy/move 就会修改）。
  - 这会让模型在边界条件下不确定到底能不能“移动/复制”。
- “thoroughness level”只被提及但没定义：缺少 quick/medium/very thorough 的可操作差异，会降低一致性。

**可迁移的技巧**

- 把“工具选型”写进 prompt，相当于把经验变成策略表。
- 明确输出契约（绝对路径、清晰汇报）是让探索结果可消费的关键。

**一个最小对比实验（建议你自测）**

- 你觉得如果删掉“Return file paths as absolute paths”，会怎样？
  - 常见后果：下游无法直接点击/定位，结果可用性下降，集成成本上升。

**与实现的对应关系**

- explore 是 subagent，并且禁用了 edit/write 等能力，突出“只读探索”的定位 [agent.ts](packages/opencode/src/agent/agent.ts#L150-L165)。

---

## 3) summary.txt：PR 风格摘要（变更面向）

**文件**

- [summary.txt](packages/opencode/src/agent/prompt/summary.txt)

**它要解决的真实问题**

- 用户完成一次交互后，需要一个“像 PR 描述一样”的简短总结，便于回溯与归档。

**结构拆解（为什么这样写）**

- 首句定义体裁：Write like a pull request description。
- 强输出约束（Rules）让它更像“编译器”而不是“写作助手”：
  - 2-3 句
  - 只描述变更，不写过程
  - 不提测试/构建
  - 不复述用户需求
  - 第一人称
  - 不提问、不引入新问题
  - 唯一例外：保留对用户的未回答问题

**赏析：把摘要当作“面向读者的变更日志”**

- 这份 prompt 的核心是把“对话摘要”改造成“变更摘要”。
- “不写过程”与“不解释用户问了什么”会显著减少噪声，让内容更接近版本库中的变更记录。

**潜在问题 / 失效模式**

- 当对话没有产生实际变更时，模型可能被迫编造“改动”，或写出空泛句子。
- “2-3 sentences max”对复杂变更可能不够，可能牺牲关键信息。

**可迁移的技巧**

- 强约束 + 明确体裁，能把 LLM 的输出从“散文”收敛为“标准件”。
- “唯一例外条款”是很好的工程折中：它允许系统保留关键交互闭环。

**与实现的对应关系**

- summary agent 会在满足一定条件时用于生成 message 级摘要正文 [summary.ts](packages/opencode/src/session/summary.ts#L113-L154)。

---

## 4) title.txt：对话标题生成（极强输出契约）

**文件**

- [title.txt](packages/opencode/src/agent/prompt/title.txt)

**它要解决的真实问题**

- 把一段对话/输入压缩成一个“可检索的线程标题”，方便历史搜索与回溯。

**结构拆解（为什么这样写）**

- 第一层硬约束：You output ONLY a thread title. Nothing else.
- 第二层任务块：<task> 明确输入输出关系与长度限制（单行、≤50 chars、无解释）。
- 第三层规则块：<rules> 定义“信息选择策略”和“风格策略”：
  - 聚焦主问题
  - 使用 -ing 动词（Debugging/Implementing/Analyzing）
  - 保留技术名词、数字、文件名、HTTP code 的精确性
  - 删除 stop words（the/this/my/a/an）
  - 不假设技术栈
  - 不用工具
  - 不回应问题，只产出标题
  - 禁止输出“summarizing/generating”等元动词
  - 输入很短时用“意图/语气”命名
- 第四层示例：<examples> 提供 few-shot 的风格锚点。

**赏析：这是“输出契约最强”的一类 prompt**

- 这类 prompt 非常像接口定义：约束层层叠加，几乎不给模型自由发挥空间。
- few-shot 示例不是为了教学“语义理解”，而是为了钉住“标题风格”。

**潜在问题 / 失效模式**

- 规则内部可能出现风格冲突：
  - 强制 -ing 动词，但示例里也存在非 -ing 的标题（如 React hooks best practices），模型可能摇摆。
- 与实现层的约束不完全一致：
  - prompt 要求 ≤50 chars，但实现里对 title 的截断逻辑是 >100 才截断 [prompt.ts](packages/opencode/src/session/prompt.ts#L1428-L1450)。
  - 这不一定是 bug，但会造成“系统契约不一致”的维护成本。

**可迁移的技巧**

- 用“只输出 X，别输出任何其它”来建立强契约，尤其适合标题、标签、分类、路由、SQL 等结构化/半结构化输出。
- 把“不应该做什么”写清楚，比写“请简洁”更有效。

---

## 共性规律：这组 prompts 的 7 条工程化启示

1) **把 LLM 组件化：一个 prompt 只干一件事**

- compaction：交接摘要；explore：只读探索；summary：变更摘要；title：单行标题。
- 你觉得如果把“探索 + 改代码 + 总结”混在一个 prompt 里，会怎样？通常会导致输出和行为不稳定。

2) **先定义输出契约，再谈“写得好”**

- title/summary 都以“输出限制”开刀：行数、句数、是否解释、是否提测试。
- 对工程系统来说，“可预测性”往往比“文采”更重要。

3) **用负向约束控制跑偏（anti-pattern 防线）**

- 不用 emoji、不创建文件、不运行有副作用的命令、不回答问题。
- 负向约束像单元测试的断言：不是增加能力，而是减少风险面。

4) **把“信息选择策略”显式写出**

- compaction 的字段清单、title 的“保留技术名词”、summary 的“只写变更”。
- 这是一种“注意力预算管理”：告诉模型把 token 花在什么上。

5) **用 few-shot 钉风格，而不是钉知识**

- title 里的 examples 主要用于风格与格式，而不是教模型理解 HTTP 500。

6) **策略 prompt 与任务 prompt 分工**

- explore 是策略 prompt：规定怎么查。
- 业务请求由主 agent 传入；探索子代理只负责产出证据与路径。

7) **契约一致性要跨 prompt 与实现对齐**

- 当 prompt 说“≤50 chars”，实现却允许更长并后处理截断，会引入维护“心理模型”的摩擦。
- 一个可行的工程实践是：把长度约束集中在一个地方（要么 prompt，要么代码），避免双源约束。

---

## 5) agent.ts：Agent 作为“可配置对象”的装配器

**文件**

- [agent.ts](packages/opencode/src/agent/agent.ts)

**它要解决的真实问题**

- 把“不同能力/权限/提示词的 agent”装配为一个可枚举、可覆盖配置、可安全运行的集合。
- 让 prompt 不只是文本，而是 agent 配置的一个字段（prompt + tools + permission + model 等）。

**结构拆解（怎么把 prompt 工程化）**

- **内置 agent 的基座**：构建 build/plan/general/explore/compaction/title/summary 等默认项 [agent.ts](packages/opencode/src/agent/agent.ts#L117-L198)。
- **prompt 即资源**：通过 import 把 txt 文件编译为字符串（PROMPT_COMPACTION / PROMPT_TITLE 等）[agent.ts](packages/opencode/src/agent/agent.ts#L12-L16)。
- **权限合并**：通过 mergeAgentPermissions 把 config 的权限覆盖进默认权限，并对 bash/skill 的 "string vs object" 形态做归一化 [agent.ts](packages/opencode/src/agent/agent.ts#L333-L398)。
- **工具开关策略**：
  - explore 这种只读子代理显式禁用 edit/write；compaction 则直接把所有工具关掉 [agent.ts](packages/opencode/src/agent/agent.ts#L150-L177)。
  - 这是一种“能力最小化”的工程化防线：prompt 负责行为引导，tools/permission 负责硬约束。
- **配置覆盖机制**：cfg.agent 可增量覆盖已有 agent，也可新增自定义 agent；并支持 disable 删除 [agent.ts](packages/opencode/src/agent/agent.ts#L199-L255)。

**赏析：这里的设计模式味道**

- 你可以把 agent 看作一个“策略对象”（Strategy）：prompt/工具/权限/模型参数决定了策略如何执行。
- agent.ts 做的事情很像“装配器 + 配置驱动的工厂”（Factory + Configuration）：把多个策略实例化、合并覆盖、再选择默认策略。

**潜在问题 / 失效模式**

- **软约束与硬约束不一致**：例如 prompt 里要求某种格式，但代码里又做二次清洗/截断，最终有效契约会变得难以直觉把握。
- **权限合并的可理解性成本**：mergeDeep + 多层默认值让权限推理需要读代码；这在多人维护时容易出现“我以为允许/禁止”的误解。

**可迁移的技巧**

- 把 prompt 放进配置对象，并提供覆盖机制：这能让 prompt 迭代更像“改配置”而不是“改代码逻辑”。
- 让 prompt 与 tool/permission 形成“双保险”：prompt 负责把模型朝正确方向拉，permission 负责防越权。

---

## 6) generate.txt：生成“新 Agent 的系统提示”的元提示词

**文件**

- [generate.txt](packages/opencode/src/agent/generate.txt)

**它要解决的真实问题**

- 当用户说“我想要一个新 agent 做 X”，系统需要稳定地产出一个新 agent 规格（identifier / whenToUse / systemPrompt）。
- 这是一个典型的“prompt 写 prompt”（meta-prompting）场景：LLM 的输出不是业务答案，而是下一层 agent 的系统提示。

**结构拆解（为什么稳定）**

- **分步法**：把生成过程拆成 5+1 步（意图提取、人格设定、指令架构、性能优化、identifier、示例）[generate.txt](packages/opencode/src/agent/generate.txt#L5-L58)。
- **输出强约束**：明确要求输出必须是包含 3 个字段的 JSON，且“只能有这些字段”[generate.txt](packages/opencode/src/agent/generate.txt#L59-L64)。
- **示例要求**：要求在 whenToUse 里包含 <example>，用来约束“触发条件描述”的风格和可操作性 [generate.txt](packages/opencode/src/agent/generate.txt#L34-L58)。

**赏析：它把“提示词写作”变成可执行清单**

- generate.txt 的价值在于把经验（如何写一个好 system prompt）结构化成 checklist。
- 其中第 4 步“Optimize for Performance”很关键：它要求写入决策框架、自检步骤、fallback，这会显著降低 agent 在边界条件下的不可控输出。

**潜在问题 / 失效模式**

- 示例段落里有一处“since the user is greeting”的文案与上下文不一致（示例一是 prime function），这可能会把模型的解释模板带歪。
- prompt 提到了 CLAUDE.md，但在本仓库结构里并不一定存在；如果不存在，该指令会变成噪声。

**与实现的对应关系（为什么它更稳）**

- agent.ts 的 generate() 会把 generate.txt 作为 system prompt，并使用结构化 schema 强制 JSON 形状 [agent.ts](packages/opencode/src/agent/agent.ts#L294-L330)。
- 生成时会把“已存在的 identifiers 列表”注入，防止命名冲突 [agent.ts](packages/opencode/src/agent/agent.ts#L301-L320)。
- 你可以把它理解为：prompt 做“语义引导”，schema 做“结构约束”，二者叠加形成工程上的确定性。

---

## 7) 额外 tips：让这些 prompts 更稳、更好用

### Prompt 内部的“隐性技巧”（已经在这些 prompt 里出现）

- **角色收敛（Role Narrowing）**：开场就限定身份与任务边界（title generator / file search specialist / summarizer），把输出空间做小。
- **体裁锁定（Genre Lock）**：用“PR 描述”“线程标题”等成熟体裁作为风格锚点，减少漂移。
- **输出契约优先（Contract First）**：先约束“只能输出什么/最多几句/不能提什么”，再谈内容质量。
- **负向约束优先（Don’ts First）**：明确禁止项（不创建文件、不改系统状态、不回应问题、不用工具），比“请小心”更有效。
- **注意力预算清单（What to focus on）**：用条目列出必须覆盖的信息字段，降低“漏关键信息”的概率。
- **清单化（Checklist）**：像 generate.txt 这样把生成过程拆步骤，能显著提升一致性。

### Prompt 与实现协同的“工程秘诀”（更关键）

- **双层对齐（redundant alignment）**：系统 prompt + 运行时 user 指令同向重复强调（compaction 场景），比单层 prompt 更稳。
- **硬约束兜底（Schema / Post-process）**：用 schema 强制结构，用清洗/截断保证落库格式；prompt 负责语义引导，代码负责结构确定性。
- **能力最小化（Least Capability）**：通过 tools/permission 关掉越权能力，避免纯靠 prompt 说教。

### 5 个立刻可复用的增强建议

- **为 compaction 增加固定模板**：Done / Doing / Files / Next / Constraints / Decisions / Risks，用可扫描结构提升交接质量。
- **消除 explore 的指令张力**：将 Bash 能力表述统一为“只读命令”（ls/find/rg/cat），避免 copy/move 这类副作用动词。
- **统一 title 的长度契约来源**：要么完全靠 prompt，要么完全靠代码截断，避免“双源约束”造成维护摩擦。
- **为 summary 增加“无变更兜底”**：明确当无代码变更时输出固定句式，减少被迫编造。
- **为 generate 增加自检门槛**：要求在 systemPrompt 中包含自检条目（字段齐全、只输出 JSON、无多余文本），配合 schema 会更稳定。
