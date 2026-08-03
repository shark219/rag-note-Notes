# Agent 流程

用户消息 → 加载会话历史 → 构建 System Prompt + 上下文 → Agent 执行 → 生成最终回答 → SSE 流式返回

**SSE（Server-Sent Events，服务器发送事件）**，一种**基于 HTTP 长连接的单向流式推送技术**，允许服务器持续、实时地向客户端（浏览器）推送文本数据，也是大模型对话、实时通知类场景最常用的流式输出方案。

```
用户查询
  │
  ▼
┌─────────────────────────────────────┐
│ ChatController.agentQueryStream()    │  ← POST /chat/agent/query/stream
│ 校验参数并调用 AgentService          │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│ AgentService.streamAgentResponse()   │  ← 核心主入口
│ 创建 SseEmitter（300秒超时）          │
│ 捕获 SecurityContext（异步线程认证）   │
│ 提交到 taskExecutor 异步执行          │
└────────────────┬────────────────────┘
                 │ (异步线程开始)
                 ▼
┌─────────────────────────────────────┐
│ 1. 保存用户消息到数据库（非重新生成时）  │
│ 2. 加载会话历史                        │
│    → ContextManager 滑动窗口+摘要压缩   │
│ 3. 根据用户开关过滤可用工具列表          │
│ 4. 构建附件上下文（文件上传场景）         │
│ 5. 设置 AgentTools 检索范围            │
└────────────────┬────────────────────┘
                 ▼
┌─────────────────────────────────────┐
│ SupervisorService.plan(query)        │  ← 规划器：分析查询复杂度
│ 返回子任务列表                        │
│ 单一非写回任务 → 降级为单 Agent       │
│ 空列表 → 走单 Agent                  │
└──────────┬──────────────────────────┘
           │
     ┌─────┴──────┐
     ▼            ▼
  单 Agent     多 Agent 流水线
     │            │
     ▼            ├─ SEQUENTIAL: runSequentialPipeline()
     │            │  子任务有依赖，逐个执行
     │            │  上一步结果注入下一步
     │            │  AgentLoop → Composer → 注入下一步
     │            │  → WriterService.synthesize()
     │            │  → QualityReviewer.reviewAnswer()
     │            │
     │            └─ PARALLEL: runParallelPipeline()
     │               子任务独立，并行执行
     │               CompletableFuture.allOf 等待全部完成
     │               AgentLoop → Composer → 汇总
     │               → WriterService.synthesize()
     │               → QualityReviewer.reviewAnswer()
     │
     ▼
┌─────────────────────────────────────┐
│ composeAndReview() —— 单 Agent 专用  │
│ 1. EvidencePack.from(state)         │
│ 2. ResponseComposer.compose()       │
│ 3. QualityReviewer.reviewAnswer()   │
│ 不达标 → 重新 compose 一次           │
└────────────────┬────────────────────┘
                 ▼
         最终回答 (SSE 流式返回)
```

---
![[Pasted image 20260802213512.png|932]]
## streamAgentResponse 完整流程

入口：`ChatController.agentQueryStream()` → `AgentService.streamAgentResponse()`（约726行，14个依赖注入）

### (1). 创建 SseEmitter（300秒超时）

实际创建位置：`AgentService.streamAgentResponse()` 第129行

```java
SseEmitter emitter = new SseEmitter(300000L);
```

300秒是一次 SSE 连接从建立到完成的最大存活时间（即AI单条回复消息的处理时间）。每次发消息都会建立一个新的SSE连接。

设置300秒超时是为了防止异常情况导致连接永远不关闭。Agent 多轮工具调用 + LLM 推理可能耗时较长（尤其是多 Agent 流水线），120秒可能不够。

正常情况：

async线程执行完 → emitter.complete() → 连接关闭 ✓

异常情况（没有超时）：

async线程卡死（比如 LLM API 无响应）→ 永远不 complete → 连接永远挂着 → 前端永远在等待 → Tomcat 线程资源泄漏

异常情况（有 300 秒超时）：

async线程卡死 → 300秒后自动触发超时 → 连接强制关闭 → 前端收到超时错误 → 可以提示用户重试

### (2). 捕获 SecurityContext（解决异步线程认证丢失）

实际执行位置：`AgentService.streamAgentResponse()` 第131行

```java
SecurityContext securityContext = SecurityContextHolder.getContext();
```

为什么要用异步线程？一个线程池不够用吗？

Agent 处理太耗时，不能阻塞 Tomcat 线程。

当多个用户都在聊天界面，Tomcat 线程池的线程都在执行Agent，由于Agent耗时原因会很容易存在没有空闲线程的情况，请求就会进入排队等待，页面卡死。

加入异步线程池之后当需要调用Agent服务时只需要提交任务到 taskExecutor，把任务放到异步线程池队列就可以继续处理下一个请求。

系统涉及两个线程池：

```
┌─────────────────────────────────┐
│ Tomcat 线程池（处理 HTTP 请求） │
│ 默认: 核心200个线程, 最大200个线程 │
│ 线程名: http-nio-8000-exec-1, exec-2, exec-3 ... │
│ 职责: 接收请求 → 执行Controller → 返回响应 │
└─────────────────────────────────┘

┌────────────────────────────────┐
│ taskExecutor 线程池（异步任务） │
│ 核心4个线程, 最大16个线程, 队列容量100 │
│ 线程名: async-1, async-2, async-3, async-4 │
│ 职责: Agent推理、文档处理、笔记自动标签等 │
└────────────────────────────────┘
```

为什么进入异步线程之前需要捕获 SecurityContext？

Spring Security 默认用 ThreadLocal 存储用户的认证信息token，ThreadLocal 是线程私有的，线程之间互相看不到。所以想要让异步线程拿到token，需要有一个传递参数的变量SecurityContext，先在切换线程前get SecurityContext，在异步线程开头手动set，finally 中清除。

必须要有finally清除，因为线程池中的线程是复用的，如果不清除可能会看到该线程处理上一个用户时的数据，发生数据泄露。

### (3). 提交到异步线程池执行

#### 3.1 预处理阶段

1. **保存用户消息**（非重新生成时）：`chatService.addMessage(sessionId, userId, "human", query)`
2. **加载会话历史**：`chatService.getSessionMessages(sessionId)`，获取 `List<ChatMessage>`
3. **上下文压缩**：`ContextManager.buildMessages()` 用 `createBalancedModel()` 生成摘要
   - Token 预算 **32K tokens**（`MAX_CONTEXT_TOKENS = 32000`），预算内直接转换
   - 策略1：压缩早期长工具结果（>500字符的 AI 回复，LLM 压缩至300字符内）——只压缩前70%的消息
   - 策略2：摘要压缩——仍超预算则早期消息整体压缩为摘要（从后往前累加token，保留最近消息原文，至少保留最近5条）
4. **工具过滤**：`filterTools(enableKnowledge, enableNotes)`
   - `ragSummary` 在知识库**或**笔记任一开启时可用
   - 知识库工具（仅 `ragSummary`）受知识库开关控制
   - 笔记工具（`listNotes`、`getNote`、`searchNotes` 等14个）受笔记开关控制
   - 通用工具（`whatTimeIsNow`、`fetchUrl`、`generateDiagram`、`generateMindMap`）始终可用
5. **构建附件上下文**：`chatService.buildAttachmentContext(fileIds, userId)` — 有上传文件时，注入附件内容到查询中
6. **设置 AgentTools 检索范围**：`agentTools.setSearchFilters(enableKnowledge, enableNotes, selectedKnowledgeDocs, selectedNotes)`

#### 3.2 Supervisor 语义分诊

```java
List<SubTask> subTasks = supervisorService.plan(query);
```

SupervisorService 用 LLM（精确模式，`createPreciseModel()`，temperature=0.1）分析查询复杂度：

- **返回空列表** → 简单任务，走单 Agent
- **返回单一子任务且非写回类** → 降级为单 Agent（避免不必要的规划开销）
- **返回 2+ 子任务** → 多 Agent 流水线

每个子任务包含：

- **id**：唯一标识（R-1, R-2...）
- **goal**：任务目标（如"搜索古诗词笔记并读取完整内容"）
- **successCriteria**：成功标准列表（如 ["mindmap"]）
- **executionMode**：执行模式（SEQUENTIAL / PARALLEL）
- **dependsOn**：依赖关系（数字索引列表）
- **toolHint**：建议工具（可选参考）
- **mustUseTool**：是否必须使用工具（布尔值）

#### 3.3 单 Agent 路径

```
AgentLoop.run() — 目标驱动的主循环

  第 N 轮开始:
  │
  ├─ 连续2轮无进展 → 注入环境状态视图（反思）
  ├─ 发送 SSE thinking 事件（轮次/进度信息）
  ├─ LLM 生成（createPreciseModel, temperature=0.1），传入消息列表 + 工具定义
  │
  ├─ LLM 要求调用工具
  │   ├─ 防重复规则1：同工具同参数已失败过 → 拦截 + 注入状态+建议
  │   ├─ 防重复规则2：刚执行过完全相同调用 → 拦截 + 注入环境状态
  │   ├─ 执行工具 → 获取 ToolResult
  │   ├─ 检查工具是否被禁用（开关控制）
  │   ├─ ToolResultEvaluator 评估质量（GOOD / POOR / ERROR）   
  │   ├─ 构建 Observation 注入给 LLM（含质量状态、错误码、恢复建议、替代策略）
  │   ├─ GoalEvaluator 目标检测（工具成功后立即调用）：
  │   │   ├─ 读笔记类（getNote）+ 目标仅是阅读 → 达成即止
  │   │   ├─ 产物类（思维导图/图表）→ 产物即完成
  │   │   │   └─ 若还需写回笔记（关键词匹配）→ 继续执行（注入写回提示）
  │   │   ├─ 写操作类（createNote/editNote/deleteNote/mergeNotes/markReviewed/scheduleReview）→ 完成
  │   │   └─ 其他工具 → UNDETERMINED，让 LLM 自己判断
  │   └─ 进入第 N+1 轮
  │
  └─ LLM 返回文本（不调工具）
      ├─ 第1轮无工具调用且为反问（isClarificationQuestion）→ NEED_CLARIFICATION
      │   启发式规则：含问号 + 长度<200 + 无Markdown标题 + 含反问引导词
      ├─ 强制 RAG 拦截：未调过工具 + 知识讲解类问题（isKnowledgeSeekingQuery）
      │   + ragSummary 可用 → 强制先调 ragSummary，禁止直接用通用知识回答
      ├─ GoalEvaluator 判断是否放行（evaluateOnTextResponse）：
      │   ├─ 目标已达成/写操作已确认 → 放行
      │   ├─ 未调过工具 + 通用知识问题 → 放行
      │   ├─ 未调过工具 + 涉及数据操作 → 拦截（注入"必须调工具"提示）
      │   ├─ 结果全 POOR/ERROR → 拦截（换策略提示）
      │   ├─ 解释/总结类 + 已有成功工具结果 → 放行（避免过度追问细节）
      │   ├─ 已有成功结果但被拦截过 → 放行（避免无限循环）
      │   ├─ 有成功 + 有目标 → LLM 自检目标是否达成 checkGoalWithLLM（10秒超时）
      │   │   未达成 → 拦截（注入未达成原因）；达成 → 放行
      │   └─ 无明确目标 → 放行（自由对话）
      └─ 放行 → 循环结束

  最大 10 轮
```
![[Pasted image 20260731154533.png|1049]]
![[Pasted image 20260731161455.png|706]]
![[Pasted image 20260731154646.png|1060]]


**AgentLoop 架构核心要素：**

| 组件                         | 职责                                                               |
| -------------------------- | ---------------------------------------------------------------- |
| AgentState                 | 执行状态：目标、证据级别（LIST/SEARCH/CONTENT/ARTIFACT）、工作记忆、工具调用审计、产物列表、观察记录 |
| ToolResultEvaluator        | 工具结果质量评估（GOOD/POOR/ERROR）+ 错误码恢复建议 + 结构化反思/替代策略生成                |
| GoalEvaluator              | 工具成功后自动判定（写操作/产物自动完成）+ LLM 自检目标达成 + 文本响应时拦截/放行判断                 |
| ConversationContextManager | 代词引用解析（"这篇笔记"、"它"注入当前 noteId、title）                              |

**执行要点：**

- AgentLoop 不生成最终回答，只返回 `Outcome`（READY / MAX_ROUNDS / NEED_CLARIFICATION）和 `AgentState`
- 每次工具成功后立即调用 `GoalEvaluator`——目标达成则直接结束循环（不等 LLM 决策）
- `Observation` 消息包含质量状态、错误码、恢复建议、替代策略、已知事实，引导 LLM 自主决策
- 强制 RAG 机制：对知识讲解类问题，即使 LLM 想直接回答也会被拦截，强制先调 `ragSummary`
- 最终回答由 `ResponseComposer` 根据 `EvidencePack` 独立生成（另一轮 LLM 调用，`createBalancedModel()`）

#### 3.4 多 Agent 流水线

**顺序执行（SEQUENTIAL）：** 子任务间有依赖（如"读取笔记A → 生成思维导图 → 写回原笔记"）

```
for each 子任务:
  1. 构建顺序任务 Prompt（含上一步完成上下文）
  2. agentLoop.run() 执行当前子任务（注入目标 + successCriteria）
  3. responseComposer.compose() 生成子任务结果片段
  4. 将上一步结果注入下一步的 completedContext
  5. 合并 AgentState（产物、证据等）

writerService.synthesize() 整合全部结果
qualityReviewer.reviewAnswer() 质量审查（不达标 → 带反馈重新合成一次）
```

**并行执行（PARALLEL）：** 子任务相互独立（如同时搜索知识库和笔记）

```
CompletableFuture.runAsync(taskExecutor) 提交全部子任务
CompletableFuture.allOf().join() 等待全部完成
  每个子任务独立 agentLoop.run()
  独立 responseComposer.compose()
合并所有 AgentState

writerService.synthesize() 整合全部结果
qualityReviewer.reviewAnswer() 质量审查（不达标 → 带反馈重新合成一次）
```

#### 3.5 composeAndReview（单 Agent 合成 + 审查）

```java
private String composeAndReview(AgentLoopResult loopResult, String query, SseEmitter emitter) {
    // 1. 发送 composing 思考事件
    // 2. EvidencePack.from(state) 提取结构化证据
    // 3. ResponseComposer.compose(pack, outcome) 生成回答
    //    用 createBalancedModel()，temperature=0.5
    //    模板: prompt/compose_answer.txt
    // 4. QualityReviewer.reviewAnswer(query, docs, answer)
    //    → ReviewResult{approved, reason, feedback}
    // 5. 不达标 → 重新 compose 一次（只重试一次）
    // 6. 返回最终回答
}
```

#### 3.6 质量审查 — QualityReviewer

```java
QualityReviewer.reviewAnswer(query, documents, answer)
  → ReviewResult{approved, reason, feedback}
```

审查内容（基于 `prompt/review_answer.txt` 模板）：

- 回答是否基于提供的证据，有无编造
- 是否完整回答了用户问题
- 审查结果用 JSON 格式解析：`{"verdict":"approved/rejected","reason":"...","feedback":"..."}`

不达标时追加反馈重试一次（单 Agent：重新 compose；多 Agent：带 feedback 重新合成）。

另有 `reviewRetrieval()` 方法审查检索质量（基于 `prompt/review_retrieval.txt`），`rewriteQuery()` 根据审查反馈改写查询（基于 `prompt/query_rewrite.txt`）。

#### 3.7 证据包体系 — EvidencePack

`ResponseComposer` 根据 `EvidencePack`（从 `AgentState` 提取的结构化证据包）生成最终回答。

证据包结构（`EvidencePack` record）：

| 字段 | 来源 | 说明 |
|------|------|------|
| userQuery | AgentState.originalQuery | 用户原始查询 |
| notes | 工具调用历史解析 | 去重后的笔记证据列表，同笔记只保留深度最高的 |
| knowledgeBaseSummary | ragSummary 工具结果 | 知识库检索摘要 |
| noteStats | getNoteStats 工具结果 | 笔记统计数据 |
| todayReviews | getTodayReviews 工具结果 | 今日复习数据 |
| writeConfirmation | 写操作工具结果 | 创建/编辑/删除等操作确认 |
| goal | AgentState.goal | 任务目标 |
| artifacts | AgentState.artifacts | 产物（思维导图、图表等） |

**证据深度级别**：`LIST_EVIDENCE`(列表摘要) → `SEARCH_EVIDENCE`(搜索预览) → `CONTENT_EVIDENCE`(完整正文) → `ARTIFACT_EVIDENCE`(产物)

去重规则：同一 noteId 只保留深度最高的证据。

### (4). SSE 响应发送

1. 保存 AI 回复到数据库：`chatService.addMessage(sessionId, userId, "ai", response)`
2. 发送 SSE thinking 事件（`stage: "complete"` 已处理完成）
3. 分块流式发送响应：每50字符一条 response 事件（含 session_id），间隔50ms
   - 提升用户体验，方便前端模拟打字机效果，逐字符显示
4. 发送 SSE done 事件，包含：
   - **session_id**：当前会话 ID
   - **trace_id**：RAG trace 追踪 ID（优先使用 RagService 的 trace_id）
   - **token_used / token_max**：Token 使用统计（estimateEntityTokens 估算）
   - **artifacts**：产物数据列表（type/id/label + 元数据，供前端渲染思维导图、图表等）

### (5). 清理（finally 块）

- 清除 AgentTools 检索范围配置（`agentTools.clearSearchFilters()` — ThreadLocal）
- 清除 SecurityContext（线程复用安全，防止数据泄露）
- 如未发完成事件 → 发送 error 事件（含异常信息 + session_id）
- 安全关闭 emitter

### (6). 立即返回 emitter（释放 Tomcat 线程）

---

- **Clarifier 集成到 AgentLoop**：`isClarificationQuestion()` 方法内联在循环中（启发式规则：问号+长度+反问引导词）
- **自定义 AgentLoop**：目标驱动（Goal → Action → Observation → State Update → Goal Check）
- **GoalEvaluator**：LLM 自检目标达成 + 写操作/产物自动判定 + LLM 调用 `checkGoalWithLLM()` 兜底
- **最多10轮**：Agent 循环最大10轮，配合反思机制
- **300秒超时**：SseEmitter 300秒超时（多 Agent 流水线更耗时）
- **证据级别**：LIST → SEARCH → CONTENT → ARTIFACT，四级证据深度 + 自动去重
- **产物追踪**：Artifact 系统追踪思维导图、图表等产物，输出到前端 SSE done 事件
- **多 Agent 流水线**：支持 SEQUENTIAL 和 PARALLEL 两种执行模式
- **19个工具**：新增 `getNoteStats`、`scheduleReview`
- **强制 RAG 拦截**：知识讲解类问题强制先检索知识库/笔记，禁止直接用通用知识回答
- **环境状态视图**：连续无进展时注入状态视图（非"请反思"，而是"这是状态，请决策"）
- **Observation 体系**：结构化 Observation 包含质量评估、错误码、恢复建议、替代策略、已知事实
- **composeAndReview 一体化**：单 Agent 路径将 compose 和 review 合并为一个方法，不达标自动重试

---

## 设计取舍与演进（为什么这样设计）

上面的流程图和步骤拆解回答的是"每一步做了什么"，这一节补充"为什么这么做、之前踩过什么坑"，方便回顾关键组件背后的取舍。

### Supervisor 语义分诊：为什么不是所有请求都跑一遍规划

如果每个请求都先过一遍 Supervisor 拆分子任务，简单问题（"JVM有哪些垃圾回收器"）也要多等一次 LLM 调用；但完全不分诊的话，复合任务（"搜索笔记A、生成思维导图、再写回笔记A"）塞进单个 AgentLoop 的 10 轮里容易轮次不够或语义漂移。

`SupervisorService.plan()`（第46行）用精确模式 LLM（temperature=0.1）返回 JSON 数组，但 LLM 的 JSON 输出并不总是"纯净"——经常夹带"根据分析："之类的中文前缀或 \`\`\`json\`\`\` 代码块标记。为此做了两层兜底：`cleanLlmJson()`（第151行）先按前缀/代码块规则清洗，失败再用 `extractJsonArray()`（第186行）按括号计数精确提取第一个数组，避免因为格式漂移直接判定规划失败、退化为单 Agent。

即便 Supervisor 成功拆出子任务，`AgentService` 里还有一层二次降级（对应本文档开头流程图"单一非写回任务 → 降级为单 Agent"）：只有 1 个子任务、且不是"生成产物+写回笔记"这种强需要顺序编排的场景，也直接退化为单 Agent。"要不要走多 Agent"被拆成两层过滤，规划开销尽量只花在真正复杂的请求上。

### AgentLoop：从"规则机器人"到"目标驱动"的范式转变

代码注释里记录了这段演进：旧版本是"要求 LLM 调指定工具 → 执行 → CompletionGate 拦截 → ping-pong"——本质是外部规则强推 LLM 按脚本走，LLM 换个说法就会被规则打断。现在的 `AgentLoop.run()`（第64行）改成"给 LLM 目标 → 自主选工具 → 执行 → Observation 注入 → GoalEvaluator 判断"，规则退居为"防止跑偏的护栏"，而不再是"驱动流程的主干"。

这个转变最直接的体现是 `GoalEvaluator.evaluateAfterToolSuccess()`（第54行）：工具成功后不等 LLM 表态，直接判断目标是否达成——写操作工具（`createNote`/`editNote`/`deleteNote`/`mergeNotes`/`markReviewed`/`scheduleReview`）成功即完成，产物工具（思维导图/图表）生成即完成（除非还要求写回原笔记）。"确定性高的场景"走硬编码规则秒判，省掉一次 LLM 调用；"确定性低的场景"才留给 LLM 自己判断，是效率和准确性的折中，而不是把所有判断都甩给 LLM。

### GoalEvaluator 的三代演进与防死循环安全阀

`evaluateOnTextResponse()`（第234行）判断"LLM 不调工具、直接想回答"时该不该放行，这段逻辑本身经历了三代迭代：v1 用硬编码字符串匹配判断目标达成，LLM 换个表达就误判；v2 退化成只看"有没有调过工具"，结果调了工具但答非所问也会被放行；v3（现在的版本）在调过工具且有明确目标时，单独用低温度（0.0）LLM 跑一次 `checkGoalWithLLM()`（第261行，10秒超时）做语义判断，才算真正理解目标本身而不是关键词对不对得上。

但纯 LLM 判断也有风险——如果 LLM 自己的判断标准偏严，可能反复拦截，导致 AgentLoop 在 10 轮预算内空转。所以加了一道安全阀（第143行）：只要已经有过成功的工具结果、又被拦截过一次，第二次直接放行，不再等 LLM 二次确认。相当于用"轮次成本"给"判断严格度"设了个上限。

### ToolResultEvaluator：把"失败原因"翻译成"下一步策略"

工具"没抛异常"不代表"结果有用"——搜索返回空列表、检索到不相关内容，都不是异常，但如果原样喂给 LLM，很容易导致编造答案。`ToolResultEvaluator.evaluate()`（第21行）优先用结构化 `ToolResult` 的三态（SUCCESS/EMPTY/ERROR）判断，字符串关键词匹配只是降级兜底（对应 3.3 节 Observation 构建里的质量分级）。

区别于笼统的"重试"，`suggestFixByErrorCode()`（第71行）按错误码给出差异化建议：`NOTE_NOT_FOUND` 提示"先用 searchNotes/listNotes 定位正确 ID 再调 getNote"，`RAG_ERROR` 提示"换用笔记搜索"，`SEARCH_ERROR` 提示"换关键词或换工具"。这些反思结果不是一次性提示，而是通过 `buildStructuredReflection()`（第106行）写进 `AgentState.workingMemory`，变成后续轮次里 LLM 能持续看到的"已知失败经验"，避免同一个死胡同被反复尝试。

### ContextManager 两级压缩：为什么不是"一刀切摘要"

32K token 预算（第31行）超限时，直接把全部历史压成一段摘要最简单，但会丢失最近几轮的细节，而这些细节往往才是当前对话真正需要的上下文。所以拆成两级：先只压"早期、超过500字符的 AI 长回复"（多数是工具返回的长文本，比普通对话消息占 token 更多），用 LLM 摘成300字符以内（`compressToolResults()`，第91行）；如果压完仍超预算，才做整体摘要（`compressWithSummary()`，第159行），并强制至少保留最近5条消息用原文。两级递进，只在真正必要时才牺牲更多信息量。

### 多温度模型分工 + Compose/Review 双重把关

`ModelFactory` 按用途预设了三档温度（`TEMP_PRECISE=0.1`/`TEMP_BALANCED=0.5`/`TEMP_CREATIVE=0.8`，第24/26/28行）：AgentLoop 的工具决策、Supervisor 规划、GoalEvaluator 判定都用精确模式，要的是稳定不跑偏；ResponseComposer 生成回答、历史摘要用平衡模式，要的是表达自然；QueryExpander 的多 Query 扩展用创意模式，要的是覆盖面广。评估类场景（RAGAS 消融实验）单独用 `createEvaluationModel()`（第112行），温度锁定为 0，保证同一输入多次评估结果可复现——这跟"聊天/决策"场景对温度的需求完全不同，所以没有共用同一个模型实例。

最后一道关卡是 3.5 节 `composeAndReview` 里 Compose 和 Review 的组合：`ResponseComposer` 基于证据包生成回答后，`QualityReviewer.reviewAnswer()` 拿着同一份证据重新核对"有没有编造""是否完整回答了问题"，不通过则重新 compose 一次。相当于用另一次独立的 LLM 调用给第一次生成结果做"背对背复核"，弥补单次生成可能出现的幻觉，但只重试一次，避免陷入无限修正的循环。

---

## 涉及类说明

| 类 | 职责 |
|----|------|
| AgentService | 主入口，协调 Supervisor → AgentLoop → Composer → Reviewer 全流程 |
| AgentLoop | 目标驱动的主循环，Goal → Action → Observation → Goal Check |
| AgentState | 执行状态容器：目标、证据级别、工作记忆、工具调用历史、产物、观察记录 |
| AgentTools | @Tool 注解定义的所有可用工具（19个），含检索范围过滤（setSearchFilters） |
| SupervisorService | LLM 规划器，分析查询是否需拆分子任务 |
| WriterService | 多 Agent 结果合成器，基于 prompt/writer_synthesize.txt |
| ResponseComposer | 基于证据包生成最终回答（独立 LLM 调用，prompt/compose_answer.txt） |
| GoalEvaluator | 工具成功后自动判定 + LLM 自检目标达成 + 文本响应拦截/放行 |
| ToolResultEvaluator | 工具结果质量评估（GOOD/POOR/ERROR）+ 错误恢复建议 + 候选策略生成 |
| ContextManager | 对话上下文压缩（Token计数32K预算 + 工具结果压缩 + 摘要压缩） |
| ConversationContextManager | 代词引用解析 + 会话笔记上下文追踪 |
| EvidencePack | 从 AgentState 提取的结构化证据包（隔离 Agent 内部上下文） |
| QualityReviewer | 回答质量审查（reviewAnswer）+ 检索质量审查（reviewRetrieval）+ 查询改写（rewriteQuery） |
| ModelFactory | LLM 模型工厂（支持智谱/DeepSeek/Ollama/阿里云，三种温度预设：精确/平衡/聊天） |
| TokenCounter | Entity Token 估算 |
| ChatController | SSE 入口 POST /chat/agent/query/stream |
