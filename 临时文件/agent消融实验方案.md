# Agent 消融实验完整方案

> 基于当前项目实际代码架构重新设计，参考 `note.md` 第十章消融实验框架，补充 Agent 层详细设计与 RAG 层实现状态对照。

---

## 一、当前项目 Agent 架构速览（截至 2026-07）

```
用户查询
  │
  ▼
ChatController.agentQueryStream()
  ├─ 创建 SseEmitter（300秒超时）
  ├─ 捕获 SecurityContext（异步线程认证）
  └─ 提交到 taskExecutor 异步线程
        │
        ▼
  ┌─────────────────────────────────────┐
  │ AgentService.streamAgentResponse()   │
  ├─ 1. 保存用户消息                      │
  ├─ 2. ContextManager 三层上下文压缩      │
  ├─ 3. filterTools() 按用户开关过滤工具    │
  ├─ 4. SupervisorService.plan() 语义分诊  │
  └──┬──────────────────────────────────┘
     │
     ├─── [子任务为空] ─── 单 Agent 路径
  │     AgentLoop.run() 目标驱动循环
  │     ├─ LLM 自主选工具（Goal→Action→Observation→Goal Check）
  │     ├─ ToolResultEvaluator 质量评估 + 反思注入
  │     ├─ GoalEvaluator 目标检测（产物/写操作自动终结）
  │     ├─ 防重复调用拦截（同参数/同失败）
  │     ├─ 最大 10 轮 + 连续2轮无进展自动反思
  │     └─ ResponseComposer.compose() 基于 EvidencePack 生成回答
  │         └─ QualityReviewer.reviewAnswer() 质量审查
  │
     └─── [子任务 2+] ─── 多 Agent 流水线
           ├─ SEQUENTIAL: runSequentialPipeline() 依赖链
           └─ PARALLEL: runParallelPipeline() 独立并行
                 └─ WriterService.synthesize() 合成
                     └─ QualityReviewer.reviewAnswer() 审查
```

### 涉及核心类

| 类 | 当前行数 | 核心职责 |
|----|---------|---------|
| AgentService.java | ~725 | 主入口：协调 Supervisor→AgentLoop→Composer→Reviewer |
| AgentLoop.java | ~565 | 目标驱动主循环，GoalEvaluator 自检，ToolResultEvaluator 评估 |
| AgentState.java | ~385 | 执行状态：目标/证据级别(LIST→SEARCH→CONTENT→ARTIFACT)/工作记忆 |
| AgentTools.java | ~550 | 17 个 @Tool 注解工具 |
| SupervisorService.java | ~223 | LLM 规划器，拆分子任务 |
| WriterService.java | ~105 | 多 Agent 结果合成 |
| ResponseComposer.java | ~202 | 基于 EvidencePack 生成最终回答 |
| GoalEvaluator.java | ~350 | LLM 自检目标达成 + 写操作/产物自动判定 |
| ToolResultEvaluator.java | ~180 | 工具结果质量评估 + 错误恢复建议 |
| ContextManager.java | ~245 | 三层压缩（Token计数→工具结果压缩→摘要压缩） |
| ConversationContextManager.java | ~74 | 代词引用解析 |
| QualityReviewer.java | ~185 | 检索质量/回答质量双重审查 |
| ModelFactory.java | ~188 | 模型工厂（智谱/DeepSeek/Ollama/阿里云，4种温度预设） |

---

## 二、RAG 消融实验实现状态

> RAG 消融实验（R-1~R-9）已全部实现，以下为对照 note.md 设计文档的完成度。

### 2.1 已实现验证

| 实验 | 设计文档 | 实现状态 | 实现位置 |
|------|---------|---------|---------|
| R-1 w/o QueryExpander | 10.3 实验 R-1 | ✅ 已实现 | `HybridRetriever.java` queryExpansionEnabled 开关 |
| R-2 w/o Vector Search | 10.3 实验 R-2 | ✅ 已实现 | `HybridRetriever.java` vectorSearchEnabled 开关 |
| R-3 w/o BM25 Search | 10.3 实验 R-3 | ✅ 已实现 | `HybridRetriever.java` bm25SearchEnabled 开关 |
| R-4 w/o RRF Fusion | 10.3 实验 R-4 | ✅ 已实现 | `HybridRetriever.java` rrfFusionEnabled→simpleMerge() 降级 |
| R-5 w/o Reranker | 10.3 实验 R-5 | ✅ 已实现 | `HybridRetriever.java` rerankEnabled→topK 截取 |
| R-6 w/o Source Attribution | 10.3 实验 R-6 | ✅ 已实现 | `RagService.java` buildContextPlain() |
| R-7 w/o Retrieval QualityReviewer | 10.3 实验 R-7 | ✅ 已实现 | 配置开关就绪（当前管道中未实际调用 QualityReviewer） |
| R-8 Chunk Size 对比 | 10.3 实验 R-8 | ✅ 配置定义 | `AblationConfig.chunkSizeExperiments()` 参数就绪 |
| R-9 Top-K 对比 | 10.3 实验 R-9 | ✅ 配置定义 | `AblationConfig.topKExperiments()` 参数就绪 |

### 2.2 已实现文件清单

| 文件 | 说明 |
|------|------|
| `evaluation/dto/AblationConfig.java` | 消融实验配置 DTO，Builder + 预设实验工厂方法 |
| `evaluation/entity/AblationResult.java` | 消融实验结果实体，映射 `ablation_results` 表 |
| `evaluation/repository/AblationResultRepository.java` | JPA Repository |
| `evaluation/service/AblationExperimentService.java` | 核心服务：运行实验、生成报告、自动关键发现 |
| `evaluation/controller/AblationController.java` | REST API |
| `config/ApplicationProperties.java` | `Ablation.RagAblation` 配置类 |
| `rag/HybridRetriever.java` | 各检索组件插入配置开关 + simpleMerge() 降级 |
| `rag/RagService.java` | buildContextPlain() + getDocumentsAndSummary() 重载 |

### 2.3 配置传递路径

```
AblationController
  └─> AblationExperimentService.runExperiment(config, testCases, userId)
        └─> RagService.getDocumentsAndSummary(userId, query, config)
            ├─> retrieveDocuments(userId, query, config)
            │   └─> HybridRetriever.searchKnowledge(userId, query, topK, config)
            │       ├─ queryExpansionEnabled? → QueryExpander.expand()
            │       ├─ vectorSearchEnabled?   → VectorStoreService.searchKnowledge()
            │       ├─ bm25SearchEnabled?     → Bm25Service.search()
            │       ├─ rrfFusionEnabled?      → rrfFusion() / simpleMerge()
            │       └─ rerankEnabled?         → RerankerService.rerank() / topK limit
            └─> buildContext / buildContextPlain (sourceAttributionEnabled)
```

---

## 三、Agent 消融实验设计（未实现）

> 以下 8 个实验基于当前实际代码架构重新设计，与 note.md 10.2 的旧描述有重大差异。

### 3.1 当前 Agent 架构的关键变更（与 note.md 旧设计的差异）

| 旧设计（note.md 10.2） | 当前代码 | 对消融实验的影响 |
|------------------------|---------|----------------|
| `processWithFunctionCalling()` 原生循环 | 自定义 `AgentLoop` 目标驱动循环 | A-1/A-5 的参照对象变更 |
| `ClarifierService` 独立服务 | 已合并到 `AgentLoop.isClarificationQuestion()` 内联判断 | note.md 中无对应实验 |
| `CompletionGate` 硬编码拦截 | `GoalEvaluator` LLM 自检 + `ToolResultEvaluator` 评估 | A-2 涉及范围变更 |
| 单 Agent 最多 3 轮 | `AgentLoop` 最多 10 轮 + 连续 2 轮无进展反思 | A-2 影响更大 |
| 质量审查 `reviewAndRetryIfNeeded()` | `composeAndReview()` 整合在 AgentService 中 | A-4 实施位置变更 |
| 并行子任务最多 2 轮工具调用 | 每个子任务独立 AgentLoop（最多 10 轮） | A-5 影响程度加大 |
| SseEmitter 120 秒超时 | SseEmitter 300 秒超时 | 无影响 |

### 3.2 实验 A-1：移除 Supervisor 规划

| 维度 | 说明 |
|------|------|
| **实验编号** | A-1 |
| **消融内容** | 跳过 `SupervisorService.plan()`，所有查询直接走单 `AgentLoop` |
| **当前代码位置** | `AgentService.java:165-169` — `List<SubTask> subTasks = supervisorService.plan(query)` |
| **对照基线** | 完整 Agent 流水线（Supervisor 自动判断简单/复杂，多任务走流水线） |
| **预期影响** | 复杂多步骤查询的处理质量下降；简单查询无影响。多 Agent 流水线的分治优势丧失 |
| **关注指标** | Context Recall（复杂查询信息遗漏）、Answer Relevancy（遗漏子问题） |
| **实施方式** | 在 `supervisorService.plan()` 调用处插入开关，`false` 时返回空列表强制走单 Agent |
| **实施代码** | `AgentService.java` streamAgentResponse() 方法中 |
| **开关定义** | `app.ablation.agent.supervisor-enabled` |

### 3.3 实验 A-2：移除 AgentLoop 反思机制

| 维度 | 说明 |
|------|------|
| **实验编号** | A-2 |
| **消融内容** | 移除 `ToolResultEvaluator` 评估 + 反思注入 + `needsReflection()` 环境状态注入 |
| **当前代码位置** | `AgentLoop.java:89-96` — `needsReflection()` 环境注入；`AgentLoop.java:152-159` — `evaluator.evaluate()` |
| **对照基线** | 完整 AgentLoop（质量评估 + POOR/ERROR 时反思消息注入 + 连续2轮无进展注入环境视图） |
| **预期影响** | 工具返回空/低质量时 LLM 缺乏纠正线索，可能编造回答或重复无效调用 |
| **关注指标** | Faithfulness（编造风险上升）、Answer Relevancy（答非所问）、工具调用轮次（浪费轮次） |
| **实施方式** | 三点消融：① 跳过 `evaluator.evaluate()`，全部返回 GOOD；② 跳过 `buildObservationMessage()` 反思注入；③ 跳过 `needsReflection()` 环境状态注入 |
| **实施代码** | `AgentLoop.java` 中 3 处改动 |
| **开关定义** | `app.ablation.agent.reflection-enabled` |

### 3.4 实验 A-3：移除上下文压缩

| 维度 | 说明 |
|------|------|
| **实验编号** | A-3 |
| **消融内容** | `ContextManager.buildMessages()` 仅做消息格式转换，不执行三层压缩 |
| **当前代码位置** | `AgentService.java:150-151` — `contextManager.buildMessages(history, chatModel)` |
| **对照基线** | 完整三层压缩（Token 计数→工具结果压缩→摘要压缩，32K 预算） |
| **预期影响** | 长对话（20+轮）早期关键信息可能被截断丢失；短对话（<5轮）无明显影响 |
| **关注指标** | Context Recall（长对话信息丢失）、Answer Relevancy（缺少上下文导致偏差） |
| **实施方式** | 开关 `false` 时调用 `ContextManager` 的直通方法，仅做 `toLcMessages()` 转换 |
| **实施代码** | `AgentService.java` 或 `ContextManager.java` 新增 bypass 模式 |
| **开关定义** | `app.ablation.agent.context-compression` |

### 3.5 实验 A-4：移除回答质量审查

| 维度 | 说明 |
|------|------|
| **实验编号** | A-4 |
| **消融内容** | 跳过 `QualityReviewer.reviewAnswer()`，直接返回首次生成结果 |
| **当前代码位置** | `AgentService.java:442-475` — `composeAndReview()` 方法 |
| **对照基线** | 完整流程：Composer 生成回答 → QualityReviewer 审查 → 不达标带反馈重试 |
| **预期影响** | 幻觉、答非所问、信息遗漏无法自动纠正 |
| **关注指标** | Faithfulness（幻觉无法纠正）、Answer Relevancy（偏题无法纠正） |
| **实施方式** | 在 `composeAndReview()` 的 `reviewAnswer()` 调用处插入开关，不通过时直接返回 answer |
| **实施代码** | `AgentService.java` composeAndReview() 方法 |
| **开关定义** | `app.ablation.agent.quality-review-enabled` |

### 3.6 实验 A-5：移除多 Agent 流水线

| 维度 | 说明 |
|------|------|
| **实验编号** | A-5 |
| **消融内容** | 即使 Supervisor 返回 2+ 子任务，也合并为单 AgentLoop 执行 |
| **当前代码位置** | `AgentService.java:178-210` — `subTasks.isEmpty()` 分支判断 |
| **对照基线** | 完整流水线：复杂查询走 `runSequentialPipeline()` / `runParallelPipeline()` |
| **预期影响** | 复杂查询在单 AgentLoop 中可能步骤混乱、工具调用互干扰、遗漏子任务 |
| **关注指标** | Context Recall（多子任务场景遗漏）、Answer Relevancy、Token 消耗量（单 Agent 可能更多轮次） |
| **实施方式** | 开关 `false` 时强制 `subTasks` 为空列表，走单 Agent 路径 |
| **实施代码** | `AgentService.java` streamAgentResponse() 中 |
| **开关定义** | `app.ablation.agent.multi-agent-enabled` |

### 3.7 实验 A-6：移除附件上下文

| 维度 | 说明 |
|------|------|
| **实验编号** | A-6 |
| **消融内容** | 用户上传的文件附件内容不再拼入 query |
| **当前代码位置** | `AgentService.java:159-163` — `chatService.buildAttachmentContext(fileIds, userId)` |
| **对照基线** | 附件内容通过 `buildAttachmentContext()` 注入 query |
| **预期影响** | 文件相关查询无法获取文件信息，回答偏离 |
| **关注指标** | Context Recall（缺少文件内容）、Answer Relevancy（无法基于文件回答） |
| **实施方式** | 开关 `false` 时 `attachmentContext` 设为 null，query 不附加文件内容 |
| **实施代码** | `AgentService.java` 预处理阶段 |
| **开关定义** | `app.ablation.agent.attachment-enabled` |

### 3.8 实验 A-7：移除工具过滤

| 维度 | 说明 |
|------|------|
| **实验编号** | A-7 |
| **消融内容** | 全部 17 个工具始终可用，忽略用户的知识库/笔记开关 |
| **当前代码位置** | `AgentService.java:98-109` — `filterTools(enableKnowledge, enableNotes)` |
| **对照基线** | 按用户偏好过滤工具：知识库/笔记开关分别控制对应工具集 |
| **预期影响** | 用户禁用的工具仍可被调用，可能导致隐私问题或意外行为 |
| **关注指标** | 工具调用分布、用户授权合规性 |
| **实施方式** | 开关 `false` 时跳过 `filterTools()`，传入完整 `toolSpecifications` |
| **实施代码** | `AgentService.java` 工具过滤阶段 |
| **开关定义** | `app.ablation.agent.tool-filter-enabled` |

### 3.9 实验 A-8：移除 Writer 合成

| 维度 | 说明 |
|------|------|
| **实验编号** | A-8 |
| **消融内容** | 多 Agent 子任务结果不经过 `WriterService.synthesize()` 合成，改为直接拼接 |
| **当前代码位置** | `AgentService.java:641-667` — `synthesizeResults()` 调用 `writerService.synthesize()` |
| **对照基线** | WriterService 用 LLM 合并多个子任务结果为连贯回答 |
| **预期影响** | 回答结构松散、信息重复、逻辑衔接差 |
| **关注指标** | Answer Relevancy（回答连贯性）、用户可读性评分 |
| **实施方式** | 开关 `false` 时直接 `\n\n---\n\n` 拼接各子任务结果 |
| **实施代码** | `AgentService.java` synthesizeResults() 方法 |
| **开关定义** | `app.ablation.agent.writer-synthesize` |

### 3.10 实验 A-9：移除 GoalEvaluator 目标检测（新增）

| 维度 | 说明 |
|------|------|
| **实验编号** | A-9（新增，note.md 未覆盖。当前代码的新架构引入的新组件） |
| **消融内容** | 移除 `GoalEvaluator` 的所有调用：工具成功后的目标达成检测 + LLM 返回文本时的拦截/放行判断 |
| **当前代码位置** | `AgentLoop.java:174-193` — `goalEvaluator.evaluateAfterToolSuccess()` + `AgentLoop.java:220-228` — `goalEvaluator.evaluateOnTextResponse()` |
| **对照基线** | GoalEvaluator 在每次工具成功后检测目标是否达成、在 LLM 想直接回答时判断是否放行 |
| **预期影响** | ① 目标已达成后 Agent 仍可能继续调用工具（过度执行）；② LLM 可能偷懒（未调工具直接回答）；③ 写操作/产物工具无法自动终结循环 |
| **关注指标** | 工具调用轮次（过度执行浪费）、Faithfulness（无工具结果时编造）、Answer Relevancy |
| **实施方式** | 两处消融：① 跳过 `evaluateAfterToolSuccess()`，默认返回 undetermined；② 跳过 `evaluateOnTextResponse()`，默认返回 allowed |
| **实施代码** | `AgentLoop.java` 中两处 GoalEvaluator 调用 |
| **开关定义** | `app.ablation.agent.goal-evaluator-enabled` |

### 3.11 实验 A-10：移除防重复调用拦截（新增）

| 维度 | 说明 |
|------|------|
| **实验编号** | A-10（新增，note.md 未覆盖。当前代码的新架构引入的新组件） |
| **消融内容** | 移除 `hasCalledWithArgsAndFailed()` 和 `hasSameAction()` 防重复拦截逻辑 |
| **当前代码位置** | `AgentLoop.java:129-145` — 两个防重复规则 |
| **对照基线** | LLM 重复调用相同工具+参数时被拦截，附带环境状态注入 |
| **预期影响** | LLM 可能陷入死循环：反复调用同一工具+参数，浪费 Token 和轮次 |
| **关注指标** | 工具调用轮次（最大轮次占比）、Token 消耗量、失败率 |
| **实施方式** | 开关 `false` 时跳过防重复检查代码块，直接执行工具 |
| **实施代码** | `AgentLoop.java` 中两处防重复拦截 |
| **开关定义** | `app.ablation.agent.duplicate-guard-enabled` |

---

## 四、联合消融实验设计（未实现）

### 4.1 当前系统组件交互图

```
┌────────── Agent Layer ──────────┐   ┌────────── RAG Layer ────────────┐
│                                  │   │                                 │
│  Supervisor Plan ───┐           │   │   QueryExpander                  │
│                     │           │   │       │                          │
│  AgentLoop          │           │   │   ┌───┴───┐                      │
│  ├─ Reflection      │           │   │   │       │                      │
│  ├─ GoalEvaluator   │           │   │ Vector  BM25                     │
│  ├─ DuplicateGuard  │           │   │ Search  Search                   │
│  └─ ToolCall        │           │   │   │       │                      │
│                     │           │   │   └───┬───┘                      │
│  ContextManager ────┤           │   │       │                          │
│                     │           │   │   RRF Fusion                     │
│  Multi-Agent Pipeline           │   │       │                          │
│  ├─ WriterService   │           │   │  Cross-Encoder Rerank            │
│  └─ Sequential/Parallel         │   │       │                          │
│                     │           │   │  Source Attribution               │
│  QualityReviewer ───┤           │   │       │                          │
│  (Answer)           │           │   │  LLM Generation                  │
│                     │           │   │       │                          │
│  Attachment Context │           │   │  QualityReviewer                 │
│  Tool Filtering     │           │   │  (Retrieval)                     │
│                      │           │   │                                 │
└──────────────────────┘           └─────────────────────────────────────┘
```

### 4.2 实验 C-1：Agent 反思 + RAG 检索审查联合消融

| 变体 | Agent Reflection | RAG Retrieval Review | 说明 |
|------|-----------------|---------------------|------|
| 完整 | ✅ | ✅ | 两层质量控制 |
| 消融 A | ❌ | ✅ | 仅 RAG 层检错（检索失败可改写查询重试，但 Agent 不会反思换策略） |
| 消融 B | ✅ | ❌ | 仅 Agent 层反思（工具执行失败会反思，但检索质量差不会触发查询改写） |
| 消融 C | ❌ | ❌ | 无质量控制 |

**分析目标**：两层质量控制覆盖的错误类型是否有重叠。Agent 反思能否补偿 RAG 检索审查缺失？

### 4.3 实验 C-2：Supervisor + Multi-Agent 联合消融

| 变体 | Supervisor Plan | Multi-Agent Pipeline | 说明 |
|------|---------------|---------------------|------|
| 完整 | ✅ | ✅ | 完整流水线 |
| 消融 A | ❌ | ✅ | 无规划（不影响流水线判断，因为流水线依赖 Supervisor 输出） |
| 消融 B | ✅ | ❌ | 有规划但不执行（Supervisor 输出子任务但强制走单 Agent = 实验 A-5） |
| 消融 C | ❌ | ❌ | 全部移除（简单单 Agent） |

**分析目标**：Supervisor 规划和 Multi-Agent 执行是强依赖关系还是可以独立评估？

### 4.4 实验 C-3：GoalEvaluator + QualityReviewer 联合消融

| 变体 | GoalEvaluator | Answer QualityReviewer | 说明 |
|------|--------------|----------------------|------|
| 完整 | ✅ | ✅ | 两层回答质量保障 |
| 消融 A | ❌ | ✅ | Agent 可能过度执行或偷懒，但最终回答会被审查 |
| 消融 B | ✅ | ❌ | 目标检测保障任务方向正确，但回答质量无兜底 |
| 消融 C | ❌ | ❌ | 无任何质量保障 |

**分析目标**：GoalEvaluator（过程质量）和 QualityReviewer（结果质量）的互补关系。

### 4.5 实验 C-4：Context 压缩 + Top-K 联合消融

| 变体 | Context Compression | Top-K | 说明 |
|------|--------------------|-------|------|
| 完整 | ✅ | 5 | 当前设置 |
| 消融 A | ❌ | 5 | 无压缩 + 标准 TopK |
| 消融 B | ✅ | 10 | 有压缩 + 大 TopK（验证压缩是否能支撑更大召回） |
| 消融 C | ❌ | 10 | 无压缩 + 大 TopK（可能超 Token 预算被截断） |

**分析目标**：上下文压缩能否有效扩展可用的检索结果数量。在何种 TopK 下压缩策略收益最大？

---

## 五、评测指标体系

### 5.1 核心指标（LLM-as-a-Judge）

| 指标 | 计算公式 | 权重 | 说明 |
|------|---------|------|------|
| Faithfulness | SUPPORTED 声明数 / 总声明数 | 0.35 | 回答是否忠实于检索文档，评估幻觉程度 |
| Answer Relevancy | 反向生成 3 个问题与原始问题的平均语义相似度 | 0.25 | 回答是否切题 |
| Context Precision | Σ(Precision@k × rel(k)) / 相关文档数 | 0.25 | 检索文档的相关性 |
| Context Recall | 标准答案句子被上下文支持的比例 | 0.15 | 检索是否覆盖答案所需信息 |

### 5.2 Agent 专用指标

| 指标 | 说明 | 采集方式 |
|------|------|---------|
| 工具调用轮次 | AgentLoop 最大循环轮数 | `AgentState.toolHistory.size()` |
| 工具成功率 | GOOD 结果数 / 总调用数 | `AgentState.toolHistory` 统计 |
| 反思触发次数 | `needsReflection()` 触发次数 | `AgentState.consecutiveNoProgress` 追踪 |
| 防重复拦截次数 | 被 `hasCalledWithArgsAndFailed` / `hasSameAction` 拦截次数 | 新增计数器 |
| GoalEvaluator 拦截次数 | `evaluateOnTextResponse()` 返回 BLOCKED 次数 | 新增计数器 |
| 最大轮次到达率 | 达到 MAX_ITERATIONS=10 的请求占比 | `AgentLoopResult.MAX_ROUNDS` 统计 |
| 反问澄清率 | 返回 NEED_CLARIFICATION 的请求占比 | `AgentLoopResult.NEED_CLARIFICATION` 统计 |
| Token 消耗 | Agent 流程总 Token 使用量 | `TokenCounter.estimateEntityTokens()` |
| 端到端延迟 | 从请求到 SSE done 的总耗时 | `System.currentTimeMillis()` 差值 |

### 5.3 综合评分公式

```
LLM Score = Faithfulness × 0.35 + AnswerRelevancy × 0.25 + ContextPrecision × 0.25 + ContextRecall × 0.15

Rule Score = 100 - latency_penalty - empty_penalty - short_answer_penalty - low_similarity_penalty

Total Score = HarmonicMean(LLM Score, Rule Score / 100) × 100
```

---

## 六、测试数据集构建

### 6.1 数据来源

| 来源 | 数量 | 用途 |
|------|------|------|
| 线上真实 RagTrace 采样 | 200 条 | 真实请求分布 |
| LLM 基于知识库文档生成 | 300 条 | 覆盖边界场景 |
| 人工标注 Ground Truth | 100 条（从中精选） | Context Recall 评估 |

### 6.2 测试用例分层

| 难度 | 比例 | 特征 |
|------|------|------|
| L1 简单 | 30% | 事实查询，单个文档/单工具即可回答 |
| L2 中等 | 40% | 需要跨文档整合或多步工具调用链 |
| L3 困难 | 30% | 多步骤推理，需要综合多知识源 + 复杂 Agent 流程 |

### 6.3 查询类型分布

| 类型 | 比例 | 示例 | 依赖的 Agent 组件 |
|------|------|------|------------------|
| simple_query | 25% | "什么是Java的垃圾回收" | AgentLoop（单轮工具） |
| multi_step | 30% | "搜索笔记A，读取内容，生成思维导图" | Supervisor + AgentLoop + GoalEvaluator |
| comparison | 15% | "对比笔记A和笔记B中关于JVM的论述" | Supervisor + Multi-Agent + Writer |
| creation | 15% | "创建一个新笔记总结我所有关于Spring的笔记" | Supervisor + Multi-Agent + Writer |
| file_related | 10% | "总结我上传的这个PDF文件" | Attachment Context |
| clarification | 5% | 模糊查询测试反问 | AgentLoop.isClarificationQuestion |

---

## 七、未实现部分的代码实现计划

### 7.1 新增配置属性

在 `ApplicationProperties.java` 的 `Ablation` 类中新增 `AgentAblation`：

```java
@Data
public static class Ablation {
    private RagAblation rag = new RagAblation();
    private AgentAblation agent = new AgentAblation();

    @Data
    public static class AgentAblation {
        private boolean supervisorEnabled = true;         // A-1
        private boolean reflectionEnabled = true;         // A-2
        private boolean contextCompression = true;        // A-3
        private boolean qualityReviewEnabled = true;      // A-4
        private boolean multiAgentEnabled = true;         // A-5
        private boolean attachmentEnabled = true;          // A-6
        private boolean toolFilterEnabled = true;          // A-7
        private boolean writerSynthesize = true;           // A-8
        private boolean goalEvaluatorEnabled = true;       // A-9
        private boolean duplicateGuardEnabled = true;      // A-10
    }
}
```

### 7.2 AblationConfig 扩展 Agent 字段

在 `AblationConfig.java` 中新增 Agent 开关字段，扩展 `allAgentExperiments()` 工厂方法：

```java
// 新增 Agent 开关字段
@Builder.Default private boolean supervisorEnabled = true;
@Builder.Default private boolean reflectionEnabled = true;
@Builder.Default private boolean contextCompression = true;
@Builder.Default private boolean qualityReviewEnabled = true;
@Builder.Default private boolean multiAgentEnabled = true;
@Builder.Default private boolean attachmentEnabled = true;
@Builder.Default private boolean toolFilterEnabled = true;
@Builder.Default private boolean writerSynthesize = true;
@Builder.Default private boolean goalEvaluatorEnabled = true;
@Builder.Default private boolean duplicateGuardEnabled = true;

// 预设 Agent 消融实验
public static AblationConfig[] allAgentExperiments() {
    return new AblationConfig[] {
        AblationConfig.builder().experimentId("A-1").experimentName("w/o Supervisor Plan")
            .ablationComponent("Supervisor").supervisorEnabled(false).build(),
        AblationConfig.builder().experimentId("A-2").experimentName("w/o Reflection")
            .ablationComponent("Reflection").reflectionEnabled(false).build(),
        AblationConfig.builder().experimentId("A-3").experimentName("w/o Context Compression")
            .ablationComponent("ContextCompression").contextCompression(false).build(),
        AblationConfig.builder().experimentId("A-4").experimentName("w/o Answer QualityReview")
            .ablationComponent("QualityReview").qualityReviewEnabled(false).build(),
        AblationConfig.builder().experimentId("A-5").experimentName("w/o Multi-Agent Pipeline")
            .ablationComponent("MultiAgent").multiAgentEnabled(false).build(),
        AblationConfig.builder().experimentId("A-6").experimentName("w/o Attachment Context")
            .ablationComponent("Attachment").attachmentEnabled(false).build(),
        AblationConfig.builder().experimentId("A-7").experimentName("w/o Tool Filtering")
            .ablationComponent("ToolFilter").toolFilterEnabled(false).build(),
        AblationConfig.builder().experimentId("A-8").experimentName("w/o Writer Synthesis")
            .ablationComponent("Writer").writerSynthesize(false).build(),
        AblationConfig.builder().experimentId("A-9").experimentName("w/o GoalEvaluator")
            .ablationComponent("GoalEvaluator").goalEvaluatorEnabled(false).build(),
        AblationConfig.builder().experimentId("A-10").experimentName("w/o Duplicate Guard")
            .ablationComponent("DuplicateGuard").duplicateGuardEnabled(false).build(),
    };
}
```

### 7.3 AgentService 改造清单

| 实验 | 修改方法 | 插入开关位置 | 改动量 |
|------|---------|-------------|--------|
| A-1 | `streamAgentResponse()` | `supervisorService.plan(query)` 调用处 | ~2 行 |
| A-3 | `streamAgentResponse()` | `contextManager.buildMessages()` 调用处 | ~5 行（新增 bypass 方法） |
| A-4 | `composeAndReview()` | `qualityReviewer.reviewAnswer()` 调用处 | ~2 行 |
| A-5 | `streamAgentResponse()` | `subTasks.isEmpty()` 分支判断 | ~2 行 |
| A-6 | `streamAgentResponse()` | `buildAttachmentContext()` 调用处 | ~2 行 |
| A-7 | `filterTools()` | 方法入口 | ~2 行 |
| A-8 | `synthesizeResults()` | `writerService.synthesize()` 调用处 | ~5 行（降级拼接） |

### 7.4 AgentLoop 改造清单

| 实验 | 修改位置 | 插入开关 | 改动量 |
|------|---------|---------|--------|
| A-2 | `evaluator.evaluate()` 调用 + `buildObservationMessage()` 反思注入 + `needsReflection()` 环境注入 | 3 处 | ~10 行 |
| A-9 | `goalEvaluator.evaluateAfterToolSuccess()` 调用 + `evaluateOnTextResponse()` 调用 | 2 处 | ~6 行 |
| A-10 | `hasCalledWithArgsAndFailed()` 拦截 + `hasSameAction()` 拦截 | 2 处 | ~4 行 |

### 7.5 AblationExperimentService 扩展

在现有 `AblationExperimentService` 中新增 Agent 消融实验执行方法：

```java
/**
 * 异步运行所有 Agent 消融实验（A-1 ~ A-10）
 */
@Async
public CompletableFuture<List<AblationResult>> runAllAgentAblationExperiments(String userId) {
    // 1. 获取测试用例
    // 2. 跑基线（完整 Agent 流水线）
    // 3. 遍历 allAgentExperiments() 依次执行
    // 4. 报告生成
}

/**
 * 运行单次 Agent 消融实验
 * 不同之处：通过 AgentService 执行（非 RagService），需要非流式模式
 */
public AblationResult runAgentExperiment(AblationConfig config, List<TestCase> testCases, String userId) {
    // 1. 根据 config 设置 AgentService 的开关
    // 2. 逐条调用非流式 Agent 入口
    // 3. 评估回答质量
    // 4. 统计 Agent 专用指标（工具调用轮次、反思次数等）
}
```

### 7.6 非流式 Agent 执行

Agent 消融实验需要批量执行（50+ 测试用例），无法使用 SSE 流式模式。需要新增非流式入口：

```java
// AgentService 新增非流式方法
public String executeAgent(String query, String sessionId, String userId,
                           AblationConfig config) {
    // 与 streamAgentResponse() 相同的流程，但返回 String 而非 SseEmitter
    // 去掉 SSE 事件发送，只返回最终回答
}
```

### 7.7 实现优先级

| 优先级 | 内容 | 预估工时 |
|--------|------|---------|
| P0 | AgentAblation 配置属性 + AblationConfig Agent 字段 | 0.5 天 |
| P0 | AgentService/AgentLoop 开关插入（A-1~A-10） | 1 天 |
| P0 | AblationExperimentService Agent 实验扩展 | 1 天 |
| P1 | 非流式 Agent 执行入口 | 0.5 天 |
| P1 | 联合消融实验（C-1~C-4） | 1 天 |
| P2 | 测试数据集构建 | 1 天 |
| P2 | 实验执行与报告生成 | 0.5 天 |

---

## 八、完整代码修改清单

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `config/ApplicationProperties.java` | 修改 | Ablation 中新增 `AgentAblation` 内部类 |
| `evaluation/dto/AblationConfig.java` | 修改 | 新增 Agent 开关字段 + `allAgentExperiments()` |
| `evaluation/service/AblationExperimentService.java` | 修改 | 新增 `runAllAgentAblationExperiments()` |
| `agent/AgentService.java` | 修改 | 7 处插入配置开关 + 非流式执行入口 |
| `agent/AgentLoop.java` | 修改 | 3 处插入配置开关 |
| `evaluation/controller/AblationController.java` | 修改 | 新增 Agent 实验 API 端点 |

---

## 九、note.md 与当前代码的差异对照（消融实验相关）

| note.md 章节 | 描述内容 | 当前状态 | 差异说明 |
|-------------|---------|---------|---------|
| 10.1.1 完整流水线图 | 混合使用旧名 processWithFunctionCalling | ❌ 过时 | 当前为 AgentLoop 目标驱动循环 |
| 10.1.1 RAG 流水线图 | 14 步 RAG 流水线（A~J） | ✅ 基本一致 | QualityReviewer 当前在 Agent 层使用，非 RAG 层 |
| 10.2 Agent 消融实验 A-1 | "processWithFunctionCalling" 引用 | ❌ 过时 | 改为 "单 AgentLoop" |
| 10.2 Agent 消融实验 A-2 | "ToolResultEvaluator 规则评估和反思消息注入" | ✅ 当前代码存在 | AgentLoop.java 中已实现 |
| 10.2 Agent 消融实验 A-4 | "reviewAndRetryIfNeeded()" 方法名 | ❌ 过时 | 实际方法名为 `composeAndReview()` |
| 10.2 Agent 消融实验 A-5 | "并行子任务最多2轮工具调用" | ❌ 过时 | 当前每个子任务独立 AgentLoop（最多10轮） |
| 10.3 RAG 消融实验 R-1~R-9 | 7 个开关 + 2 个参数对比 | ✅ 全部已实现 | HybridRetriever + RagService 已改造 |
| 10.5 评测指标体系 | RAGAS 四指标 + 辅助指标 | ✅ 基本实现 | Agent 专用指标需补充 |
| 10.7.3 配置管理 | `ablation.agent.*` 和 `ablation.rag.*` 配置 | ⚠️ 部分实现 | RAG 已配，Agent 未配 |
| 10.9 代码实现清单 | `AblationExperimentService` 等 | ⚠️ 部分实现 | RAG 已实现，Agent 未实现 |
| 10.11 RAG 消融实验实现记录 | 已实现的 RAG 消融实验 | ✅ 已实现 | 与实际代码一致 |
| 10.11.5 待完成事项 | "Agent 消融实验 A-1~A-8" | ❌ 未实现 | 本方案的目标 |
