# Agent 流程

用户消息 → 创建 AgentTask → 规划（可选）→ AgentRuntime 执行 → AgentLoop 循环 → 合成回答 → SSE 流式返回

**SSE（Server-Sent Events，服务器发送事件）**，一种**基于 HTTP 长连接的单向流式推送技术**，允许服务器持续、实时地向客户端（浏览器）推送文本数据，也是大模型对话、实时通知类场景最常用的流式输出方案。

## 项目整体架构

当前项目已发展为模块化 Agent 系统，核心包结构：

```
agent/
├── AgentService.java          # 主入口，协调整体流程
├── AgentLoop.java             # 目标驱动主循环（63KB，核心执行器）
├── AgentState.java            # 执行状态容器（22KB）
├── AgentTools.java            # 工具定义（49KB，@Tool 注解）
├── runtime/                   # 运行时子系统（17个类）
│   ├── AgentRuntime.java      # 任务执行编排（28KB）
│   ├── AgentTaskService.java  # 任务生命周期管理
│   ├── ReflectionService.java # 反思机制
│   ├── ReplanningService.java # 重新规划
│   └── ToolExecutionService.java # 工具执行封装
├── trace/                     # 可观测性（3个类）
│   ├── AgentTrace.java        # 追踪实体
│   └── AgentTraceService.java # 追踪服务（17KB）
├── policy/                    # 策略与预算控制（3个类）
│   ├── AgentPolicyService.java
│   └── BudgetConfig.java
├── metrics/                   # 指标统计
├── entity/                    # 持久化实体
├── repo/                      # 数据访问层
└── tool/                      # 动态工具扩展（Skill 包）
```

**核心能力演进：**
- **任务持久化**：AgentTask 实体 + 暂停/恢复机制
- **反思与重规划**：ReflectionService、ReplanningService（LLM 驱动的自适应）
- **动态工具**：Skill 包系统，支持 Git 导入、Skill Center 安装
- **全链路追踪**：AgentTrace 记录规划、工具调用、反思、质量评审全流程
- **策略控制**：BudgetConfig 限制轮次/时间/Token/工具调用次数

```
用户查询
  │
  ▼
┌────────────────────────────────────────────┐
│ ChatController.agentQueryStream()          │  ← POST /chat/agent/query/stream
│ 校验参数并调用 AgentService                │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│ AgentService.streamAgentResponse()         │  ← 核心主入口
│ 创建 SseEmitter（300秒超时）                │
│ 捕获 SecurityContext（异步线程认证）         │
│ 提交到 taskExecutor 异步执行                │
└────────────────┬───────────────────────────┘
                 │ (异步线程开始)
                 ▼
┌────────────────────────────────────────────┐
│ 1. 创建 AgentTask（持久化任务实体）          │
│ 2. 初始化 AgentTrace（全链路追踪）           │
│ 3. 保存用户消息到数据库（非重新生成时）        │
│ 4. 加载会话历史 + 上下文压缩                 │
│    → ContextManager 32K token 预算          │
│ 5. 根据用户开关过滤可用工具列表               │
│ 6. 构建附件上下文（文件上传场景）             │
│ 7. 设置 AgentTools 检索范围（ThreadLocal）   │
│ 8. 加载动态 Skill 工具（从 skill 包导入）     │
└────────────────┬───────────────────────────┘
                 ▼
┌────────────────────────────────────────────┐
│ AgentRuntime.start()                       │  ← 任务执行编排
│ 决策是否启用 Supervisor 规划                │
└────────────────┬───────────────────────────┘
                 │
                 ▼ (enablePlanning = true)
┌────────────────────────────────────────────┐
│ SupervisorService.plan(query)              │  ← 规划器（LLM，temp=0.1）
│ 返回子任务列表（SubTask[] JSON）            │
│ 单一非写回任务 → 降级为单 Agent             │
│ 空列表 → 走单 Agent                        │
│ 规划结果记录到 AgentTrace                   │
└──────────┬─────────────────────────────────┘
           │
     ┌─────┴──────┐
     ▼            ▼
  单 Agent     多 Agent 流水线
     │            │
     │            ├─ SEQUENTIAL: 顺序执行子任务
     │            │  for each 子任务:
     │            │    → AgentLoop.run()
     │            │    → ResponseComposer.compose()
     │            │    → 上一步结果注入下一步
     │            │  → WriterService.synthesize()
     │            │  → QualityReviewer.reviewAnswer()
     │            │
     │            └─ PARALLEL: 并行执行子任务
     │               CompletableFuture.allOf()
     │               各子任务独立 AgentLoop
     │               → WriterService.synthesize()
     │               → QualityReviewer.reviewAnswer()
     │
     ▼
┌────────────────────────────────────────────┐
│ AgentLoop.run() —— 目标驱动主循环（最多10轮）│
│ Goal → Action → Observation → Goal Check   │
│ （详见下文"AgentLoop 执行细节"）            │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│ composeAndReview() —— 单 Agent 专用        │
│ 1. EvidencePack.from(state)               │
│ 2. ResponseComposer.compose()             │
│ 3. QualityReviewer.reviewAnswer()         │
│ 不达标 → 重新 compose 一次                 │
└────────────────┬───────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────┐
│ 4. 保存 AI 回复到数据库                     │
│ 5. SSE 流式发送（50字符/块，间隔50ms）        │
│ 6. 发送 done 事件（含 trace_id、artifacts） │
│ 7. 更新 AgentTask 状态为 COMPLETED          │
│ 8. 完成 AgentTrace 记录                     │
└────────────────┬───────────────────────────┘
                 │
                 ▼
         finally: 清理 ThreadLocal + emitter
```

---

## AgentLoop 执行细节

AgentLoop 是目标驱动的主循环，核心理念从"规则机器人"转变为"目标驱动自主决策"。

```
AgentLoop.run(state, historyMessages, tools, chatModel, emitter)

  第 N 轮开始（MAX_ITERATIONS = 10）:
  │
  ├─ 暂停点检测（支持任务暂停/恢复）
  │   检查 AgentTask.status，若为 PAUSED → 保存快照 → 退出循环
  │
  ├─ 策略检查（AgentPolicyService）
  │   检查 BudgetConfig：轮次/时间/Token/工具调用次数是否超限
  │   超限 → 返回 BUDGET_EXCEEDED
  │
  ├─ 连续无进展检测（consecutiveNoProgress >= 2）
  │   → ReflectionService.reflect() — LLM 分析卡住原因
  │   → ReplanningService.replan() — LLM 生成新策略
  │   → 注入反思结果到 workingMemory + 更新 pendingSteps
  │   → 记录到 AgentReflection 实体（持久化）
  │
  ├─ 发送 SSE thinking 事件（轮次/进度/当前目标）
  │
  ├─ LLM 生成（createPreciseModel, temp=0.1）
  │   传入：historyMessages + SystemPrompt + 工具定义
  │   → Response<AiMessage>
  │
  ├─ LLM 要求调用工具
  │   ├─ 防重复规则1：同工具同参数已失败过 → 拦截 + 注入状态
  │   ├─ 防重复规则2：刚执行过完全相同调用 → 拦截 + 注入环境状态
  │   ├─ ToolExecutionService.execute() 执行工具（统一封装）
  │   │   → 更新 AgentTaskStep（持久化每一步）
  │   │   → 记录到 AgentTrace.toolCalls
  │   │   → 检查工具是否被用户开关禁用
  │   ├─ ToolResultEvaluator.evaluate() 质量评估
  │   │   → GOOD / POOR / ERROR 三态
  │   │   → 错误码恢复建议（NOTE_NOT_FOUND/RAG_ERROR/SEARCH_ERROR）
  │   │   → 结构化反思 + 候选策略生成
  │   ├─ 构建 Observation 注入 LLM（含质量/错误码/建议/策略）
  │   ├─ GoalEvaluator.evaluateAfterToolSuccess() 目标检测
  │   │   ├─ 写操作（createNote/editNote/deleteNote/mergeNotes）→ 达成即止
  │   │   ├─ 产物类（generateMindMap/generateDiagram）→ 产物即完成
  │   │   │   └─ 若需写回笔记（关键词匹配）→ 继续执行
  │   │   ├─ 读笔记（getNote）+ 目标仅阅读 → 达成即止
  │   │   └─ 其他工具 → UNDETERMINED，让 LLM 自己判断
  │   └─ 进入第 N+1 轮
  │
  └─ LLM 返回文本（不调工具）
      ├─ 第1轮无工具调用且为反问（isClarificationQuestion）
      │   启发式：含问号 + 长度<200 + 含反问引导词
      │   → NEED_CLARIFICATION
      │
      ├─ 强制 RAG 拦截：未调过工具 + 知识讲解类问题
      │   + ragSummary 可用 → 强制先调 ragSummary
      │
      ├─ GoalEvaluator.evaluateOnTextResponse() 放行判断
      │   ├─ 目标已达成/写操作已确认 → 放行
      │   ├─ 未调过工具 + 通用知识问题 → 放行
      │   ├─ 未调过工具 + 涉及数据操作 → 拦截
      │   ├─ 结果全 POOR/ERROR → 拦截（换策略提示）
      │   ├─ 解释类 + 已有成功结果 → 放行
      │   ├─ 已有成功结果但被拦截过 → 放行（防死循环）
      │   ├─ 有成功 + 有目标 → checkGoalWithLLM（10秒超时）
      │   │   未达成 → 拦截；达成 → 放行
      │   └─ 无明确目标 → 放行
      └─ 放行 → 循环结束 → READY

  最大 10 轮（可通过 BudgetConfig 调整）
```

**新增核心机制：**

| 机制 | 说明 |
|------|------|
| **任务暂停/恢复** | 每轮检查 AgentTask.status，支持暂停 → 保存快照 → 后续恢复执行 |
| **反思与重规划** | 连续无进展 → ReflectionService（LLM 分析原因）→ ReplanningService（LLM 生成新策略） |
| **预算控制** | AgentPolicyService + BudgetConfig 限制轮次/时间/Token/工具调用，超限自动停止 |
| **全链路追踪** | 每次工具调用、反思、重规划都记录到 AgentTrace，可回溯全过程 |
| **工具执行封装** | ToolExecutionService 统一处理工具调用，记录 AgentTaskStep 持久化 |
| **动态工具加载** | SkillContextResolver 从 skill 包加载额外工具（Git 导入/Skill Center） |

---

## streamAgentResponse 完整流程

入口：`ChatController.agentQueryStream()` → `AgentService.streamAgentResponse()`（约726行，14个依赖注入）

### (1). 创建 SseEmitter + AgentTask（300秒超时）

实际创建位置：`AgentService.streamAgentResponse()` 第129行

```java
SseEmitter emitter = new SseEmitter(300000L);
String taskId = UUID.randomUUID().toString();
agentTaskService.createTask(taskId, sessionId, userId, query);
agentTraceService.startTrace(taskId, sessionId, query);
```

300秒是一次 SSE 连接从建立到完成的最大存活时间（即AI单条回复消息的处理时间）。每次发消息都会建立一个新的SSE连接。
设置300秒超时是为了防止异常情况导致连接永远不关闭。Agent 多轮工具调用 + LLM 推理可能耗时较长（尤其是多 Agent 流水线），120秒可能不够。

正常情况：
	async线程执行完 → emitter.complete() → 连接关闭 ✓

异常情况（没有超时）：
	async线程卡死（比如 LLM API 无响应）→ 永远不 complete → 连接永远挂着 → 前端永远在等待 → Tomcat 线程资源泄漏

异常情况（有 300 秒超时）：
	async线程卡死 → 300秒后自动触发超时 → 连接强制关闭 → 前端收到超时错误 → 可以提示用户重试

**AgentTask 持久化**：每个对话请求创建一个 AgentTask 实体，记录：
- taskId, sessionId, userId, query
- status（CREATED/PLANNING/RUNNING/PAUSED/COMPLETED/FAILED）
- startTime, endTime, totalSteps
- 支持任务暂停/恢复（通过 AgentTaskResumeService）

**AgentTrace 追踪**：全链路可观测，记录：
- 规划决策（SupervisorService.plan 结果）
- 工具调用详情（name/args/result/duration）
- 反思记录（ReflectionService.reflect 输出）
- 质量评审（QualityReviewer.reviewAnswer 结果）
- Token 消耗统计

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

1. **创建 AgentTask**：`agentTaskService.createTask(taskId, sessionId, userId, query)` — 持久化任务实体
2. **初始化 AgentTrace**：`agentTraceService.startTrace(taskId, sessionId, query)` — 开始追踪记录
3. **保存用户消息**（非重新生成时）：`chatService.addMessage(sessionId, userId, "human", query)`
4. **加载会话历史**：`chatService.getSessionMessages(sessionId)`，获取 `List<ChatMessage>`
5. **上下文压缩**：`ContextManager.buildMessages()` 用 `createBalancedModel()` 生成摘要
   - Token 预算 **32K tokens**（`MAX_CONTEXT_TOKENS = 32000`），预算内直接转换
   - 策略1：压缩早期长工具结果（>500字符的 AI 回复，LLM 压缩至300字符内）——只压缩前70%的消息
   - 策略2：摘要压缩——仍超预算则早期消息整体压缩为摘要（从后往前累加token，保留最近消息原文，至少保留最近5条）
6. **工具过滤**：`filterTools(enableKnowledge, enableNotes)`
   - `ragSummary` 在知识库**或**笔记任一开启时可用
   - 知识库工具（仅 `ragSummary`）受知识库开关控制
   - 笔记工具（`listNotes`、`getNote`、`searchNotes` 等14个）受笔记开关控制
   - 通用工具（`whatTimeIsNow`、`fetchUrl`、`generateDiagram`、`generateMindMap`）始终可用
7. **动态工具加载**：`skillContextResolver.resolveActiveTools(userId)` — 从 skill 包加载额外工具
   - 支持从 Git 仓库导入 Skill 包（Python/Shell 脚本）
   - 支持从 Skill Center 安装公共技能包
   - 动态注册为 @Tool 方法，LLM 可调用
8. **构建附件上下文**：`chatService.buildAttachmentContext(fileIds, userId)` — 有上传文件时，注入附件内容到查询中
9. **设置 AgentTools 检索范围**：`agentTools.setSearchFilters(enableKnowledge, enableNotes, selectedKnowledgeDocs, selectedNotes)`

#### 3.2 AgentRuntime 编排

```java
RuntimeResult result = agentRuntime.start(
    taskId, query, queryWithContext, sessionId, userId,
    systemPrompt, activeTools, emitter, enablePlanning,
    artifactWriteBack, chatModel
);
```

AgentRuntime 负责：
- 更新 AgentTask 状态（RUNNING/PLANNING/COMPLETED/FAILED）
- 决策是否启用 Supervisor 规划
- 协调单 Agent / 多 Agent 流水线
- 发送 SSE 事件（thinking/response/done/error）
- 捕获异常并记录到 AgentTrace

#### 3.3 Supervisor 语义分诊

```java
List<SubTask> subTasks = supervisorService.plan(query);
agentTraceService.recordPlanning(taskId, subTasks, executionMode);
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

规划结果记录到 AgentTrace.planningDecision，包含：
- 是否触发规划（enablePlanning）
- 子任务列表 JSON
- 执行模式（SINGLE/SEQUENTIAL/PARALLEL）
- 规划耗时

#### 3.4 单 Agent 路径

AgentLoop 执行细节已在上文"AgentLoop 执行细节"章节详细说明，这里补充关键点：

**AgentLoop 架构核心要素：**

| 组件                         | 职责                                                               |
| -------------------------- | ---------------------------------------------------------------- |
| AgentState                 | 执行状态：目标、证据级别（LIST/SEARCH/CONTENT/ARTIFACT）、工作记忆、工具调用审计、产物列表、观察记录、任务状态机 |
| ToolResultEvaluator        | 工具结果质量评估（GOOD/POOR/ERROR）+ 错误码恢复建议 + 结构化反思/替代策略生成                |
| GoalEvaluator              | 工具成功后自动判定（写操作/产物自动完成）+ LLM 自检目标达成 + 文本响应时拦截/放行判断                 |
| ConversationContextManager | 代词引用解析（"这篇笔记"、"它"注入当前 noteId、title）                              |
| ReflectionService          | 连续无进展时 LLM 分析卡住原因，生成 ReflectionResult 持久化到 AgentReflection 表           |
| ReplanningService          | 基于反思结果 LLM 生成新策略，更新 AgentState.pendingSteps                            |
| ToolExecutionService       | 统一工具调用封装，记录 AgentTaskStep（每一步持久化），更新 AgentTrace.toolCalls            |
| AgentPolicyService         | 预算控制，检查 BudgetConfig（轮次/时间/Token/工具调用次数），超限返回 BUDGET_EXCEEDED     |

**执行要点：**

- AgentLoop 不生成最终回答，只返回 `AgentLoopResult`（包含 Outcome 和 AgentState）
- 每次工具成功后立即调用 `GoalEvaluator`——目标达成则直接结束循环（不等 LLM 决策）
- `Observation` 消息包含质量状态、错误码、恢复建议、替代策略、已知事实，引导 LLM 自主决策
- 强制 RAG 机制：对知识讲解类问题，即使 LLM 想直接回答也会被拦截，强制先调 `ragSummary`
- 最终回答由 `ResponseComposer` 根据 `EvidencePack` 独立生成（另一轮 LLM 调用，`createBalancedModel()`）
- **任务暂停支持**：每轮检查 AgentTask.status，若为 PAUSED → 保存 AgentExecutionSnapshot → 后续可通过 AgentTaskResumeService 恢复
- **全链路追踪**：每次工具调用、反思、重规划都记录到 AgentTrace，可回溯完整执行过程

#### 3.5 多 Agent 流水线

**顺序执行（SEQUENTIAL）：** 子任务间有依赖（如"读取笔记A → 生成思维导图 → 写回原笔记"）

```
for each 子任务:
  1. 更新 AgentTask 状态（RUNNING_STEP_N）
  2. 构建顺序任务 Prompt（含上一步完成上下文）
  3. agentLoop.run() 执行当前子任务（注入目标 + successCriteria）
  4. 记录 AgentTaskStep（stepId/goal/toolsCalled/result/duration）
  5. responseComposer.compose() 生成子任务结果片段
  6. 将上一步结果注入下一步的 completedContext
  7. 合并 AgentState（产物、证据等）

writerService.synthesize() 整合全部结果
qualityReviewer.reviewAnswer() 质量审查（不达标 → 带反馈重新合成一次）
agentTraceService.recordQualityReview(taskId, reviewResult)
```

**并行执行（PARALLEL）：** 子任务相互独立（如同时搜索知识库和笔记）

```
List<CompletableFuture<SubTaskResult>> futures = new ArrayList<>();
for each 子任务:
  futures.add(CompletableFuture.supplyAsync(() -> {
    更新 AgentTask 状态
    agentLoop.run() 独立执行
    responseComposer.compose() 独立合成
    记录 AgentTaskStep
    返回 SubTaskResult
  }, taskExecutor));

CompletableFuture.allOf(futures).join()
合并所有 AgentState

writerService.synthesize() 整合全部结果
qualityReviewer.reviewAnswer() 质量审查（不达标 → 带反馈重新合成一次）
agentTraceService.recordQualityReview(taskId, reviewResult)
```

**子任务执行追踪**：
- 每个子任务创建 AgentTaskStep 记录（stepId/goal/toolsCalled/result/startTime/endTime）
- 并行执行时各 future 独立记录，最终汇总到 AgentTask.totalSteps
- 失败子任务记录到 AgentTrace.errors，不中断整体流程（部分失败容忍）

#### 3.6 composeAndReview（单 Agent 合成 + 审查）

```java
private String composeAndReview(AgentLoopResult loopResult, String query, SseEmitter emitter) {
    // 1. 发送 composing 思考事件
    // 2. EvidencePack.from(state) 提取结构化证据
    // 3. ResponseComposer.compose(pack, outcome) 生成回答
    //    用 createBalancedModel()，temperature=0.5
    //    模板: prompt/compose_answer.txt
    // 4. QualityReviewer.reviewAnswer(query, docs, answer)
    //    → ReviewResult{approved, reason, feedback}
    // 5. agentTraceService.recordQualityReview(taskId, reviewResult)
    // 6. 不达标 → 重新 compose 一次（只重试一次）
    // 7. 返回最终回答
}
```

质量审查追踪到 AgentTrace.qualityReview，包含：
- verdict（approved/rejected）
- reason（审查理由）
- feedback（改进建议）
- retriedCompose（是否触发重试）

#### 3.7 质量审查 — QualityReviewer

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

审查结果记录到 AgentTrace.qualityReview，包含：
- verdict（approved/rejected）
- documents（证据来源）
- answer（待审查的回答）
- retriedWithFeedback（是否触发重试）

#### 3.8 证据包体系 — EvidencePack

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

证据提取过程记录到 AgentTrace，便于分析 Agent 基于哪些信息生成了回答。

### (4). SSE 响应发送

1. 更新 AgentTask 状态为 COMPLETED
2. 保存 AI 回复到数据库：`chatService.addMessage(sessionId, userId, "ai", response)`
3. 发送 SSE thinking 事件（`stage: "complete"` 已处理完成）
4. 分块流式发送响应：每50字符一条 response 事件（含 session_id），间隔50ms
   - 提升用户体验，方便前端模拟打字机效果，逐字符显示
5. 发送 SSE done 事件，包含：
   - **session_id**：当前会话 ID
   - **task_id**：AgentTask ID（支持任务追踪）
   - **trace_id**：AgentTrace ID（全链路追踪）
   - **token_used / token_max**：Token 使用统计（estimateEntityTokens 估算）
   - **artifacts**：产物数据列表（type/id/label + 元数据，供前端渲染思维导图、图表等）
   - **total_steps**：AgentTask.totalSteps（子任务/工具调用总数）
   - **duration_ms**：任务总耗时

AgentTrace 最终完成记录，包含：
- 完整工具调用链（name/args/result/duration）
- 规划决策（是否分任务、执行模式）
- 反思记录（连续无进展时的 LLM 分析）
- 质量评审（是否通过、反馈内容）
- Token 消耗统计
- 任务总耗时

### (5). 清理（finally 块）

- 清除 AgentTools 检索范围配置（`agentTools.clearSearchFilters()` — ThreadLocal）
- 清除 SecurityContext（线程复用安全，防止数据泄露）
- 清除 SkillContext（动态工具上下文 — ThreadLocal）
- 如未发完成事件 → 发送 error 事件（含异常信息 + session_id + task_id）
- 更新 AgentTask 状态为 FAILED（异常情况）
- AgentTrace 记录错误信息
- 安全关闭 emitter

### (6). 立即返回 emitter（释放 Tomcat 线程）

---

## 核心特性与演进

- **Clarifier 集成到 AgentLoop**：`isClarificationQuestion()` 方法内联在循环中（启发式规则：问号+长度+反问引导词）
- **自定义 AgentLoop**：目标驱动（Goal → Action → Observation → State Update → Goal Check）
- **GoalEvaluator**：LLM 自检目标达成 + 写操作/产物自动判定 + LLM 调用 `checkGoalWithLLM()` 兜底
- **最多10轮**：Agent 循环最大10轮，配合反思机制（可通过 BudgetConfig 调整）
- **300秒超时**：SseEmitter 300秒超时（多 Agent 流水线更耗时）
- **证据级别**：LIST → SEARCH → CONTENT → ARTIFACT，四级证据深度 + 自动去重
- **产物追踪**：Artifact 系统追踪思维导图、图表等产物，输出到前端 SSE done 事件
- **多 Agent 流水线**：支持 SEQUENTIAL 和 PARALLEL 两种执行模式
- **工具数量**：19个基础工具 + 动态 Skill 包工具（支持用户自定义扩展）
- **强制 RAG 拦截**：知识讲解类问题强制先检索知识库/笔记，禁止直接用通用知识回答
- **环境状态视图**：连续无进展时注入状态视图（非"请反思"，而是"这是状态，请决策"）
- **Observation 体系**：结构化 Observation 包含质量评估、错误码、恢复建议、替代策略、已知事实
- **composeAndReview 一体化**：单 Agent 路径将 compose 和 review 合并为一个方法，不达标自动重试

**新增核心能力（任务持久化与运行时）：**

- **AgentTask 实体**：每个对话请求创建持久化任务记录，支持状态追踪（CREATED/PLANNING/RUNNING/PAUSED/COMPLETED/FAILED）
- **任务暂停/恢复**：AgentTaskResumeService + AgentExecutionSnapshot，支持长任务中断后恢复执行
- **AgentTrace 追踪**：全链路可观测，记录规划决策、工具调用、反思、重规划、质量评审全过程
- **反思机制**：ReflectionService — 连续无进展时 LLM 分析卡住原因，持久化到 AgentReflection 表
- **重规划机制**：ReplanningService — 基于反思结果 LLM 生成新策略，动态调整 pendingSteps
- **预算控制**：AgentPolicyService + BudgetConfig 限制轮次/时间/Token/工具调用次数，防止资源耗尽
- **ToolExecutionService**：统一工具调用封装，每次调用记录 AgentTaskStep，追踪每一步操作
- **AgentRuntime**：任务执行编排层，协调 Supervisor → AgentLoop → Composer → Reviewer 全流程
- **动态工具扩展**：SkillContextResolver 从 skill 包加载额外工具，支持 Git 导入、Skill Center 安装
- **指标统计**：AgentMetrics + AgentMetricsService 记录工具调用频率、成功率、平均耗时等指标

---

## 设计取舍与演进（为什么这样设计）

上面的流程图和步骤拆解回答的是"每一步做了什么"，这一节补充"为什么这么做、之前踩过什么坑"，方便回顾关键组件背后的取舍。

### 任务持久化：为什么需要 AgentTask 实体

早期版本 Agent 执行完全在内存中，一旦异常（LLM API 超时、服务重启、OOM）整个执行上下文丢失，用户得不到任何反馈。引入 AgentTask 持久化后：

- 每个对话请求创建一个 AgentTask 记录（taskId/sessionId/userId/query/status/startTime/totalSteps）
- 状态机：CREATED → PLANNING → RUNNING → COMPLETED/FAILED/PAUSED
- 异常时记录到数据库，前端可查询任务状态，提供"任务失败，点击重试"交互
- 支持长任务暂停/恢复：用户可主动暂停任务 → 保存 AgentExecutionSnapshot → 后续恢复执行
- AgentTaskStep 记录每一步操作（stepId/goal/toolsCalled/result/duration），便于调试"第3步卡在哪"

设计权衡：增加了数据库写入开销（每次工具调用写一条 AgentTaskStep），但换来了任务可追溯性和容错能力。对于短任务（<3轮）开销可接受，对于长任务（规划+多工具调用）追溯能力是必需的。

### AgentTrace：从"黑盒执行"到"全链路可观测"

AgentTask 解决"任务状态追踪"，但不记录执行细节（LLM 决策、工具调用参数、反思内容）。AgentTrace 补齐这一层：

- **规划决策**：SupervisorService.plan 返回的子任务列表 + 执行模式
- **工具调用链**：每次工具调用记录 name/args/result/duration/quality
- **反思记录**：ReflectionService.reflect 的 LLM 输出（为什么卡住、建议策略）
- **质量评审**：QualityReviewer.reviewAnswer 的 verdict/reason/feedback
- **Token 消耗**：累计 LLM 调用的 Token 使用量

用途：
- 调试：回放完整执行过程，定位"为什么工具没调用"、"为什么质量审查不通过"
- 分析：统计"哪些工具成功率低"、"哪些查询需要反思"
- 评估：对比不同 Prompt 模板/模型的表现（基于历史 trace 数据）

设计权衡：trace 数据可能很大（工具返回长文本），采用 JSON 字段存储，查询时按需加载。

### 反思与重规划：从"卡住就放弃"到"自适应调整"

早期版本 AgentLoop 连续无进展（2轮无新工具调用/证据增长）只会注入"状态视图"，让 LLM 自己想办法。但 LLM 可能反复尝试同一失败路径（如笔记不存在时反复 getNote）。

引入 ReflectionService + ReplanningService：
- **ReflectionService.reflect()**：LLM 分析"为什么卡住"（工具不可用？参数错误？目标不明确？）
- **ReplanningService.replan()**：基于反思结果，LLM 生成新策略（换工具？换参数？分解子目标？）
- 反思结果持久化到 AgentReflection 表，避免重复分析
- 新策略注入到 AgentState.pendingSteps，引导后续轮次执行

效果：对于"笔记标题记错"类错误，旧版会卡满10轮；新版第3轮触发反思 → 建议"先 searchNotes 模糊搜索" → 第4轮成功找到正确笔记。

设计权衡：每次反思需额外 LLM 调用（增加延迟+成本），但只在连续无进展时触发（大部分任务不触发），整体收益大于成本。

### 预算控制：为什么需要 BudgetConfig

没有预算限制时，极端情况下 Agent 可能：
- 陷入工具调用死循环（反复调同一工具，每次失败）
- 消耗大量 Token（每轮 LLM 调用累加，上下文越来越长）
- 长时间挂起（单个任务占用线程池资源，阻塞其他请求）

AgentPolicyService + BudgetConfig 限制：
- **maxIterations**：最大轮次（默认10，可调整）
- **maxTimeSeconds**：最大执行时间（默认300秒）
- **maxTokens**：最大 Token 消耗（默认50K，防止成本失控）
- **maxToolCalls**：最大工具调用次数（默认30，防止死循环）

每轮开始前检查预算，超限立即返回 BUDGET_EXCEEDED，避免资源浪费。

设计权衡：严格预算可能中断合法的复杂任务（如"分析50篇笔记生成报告"），需根据业务场景调整阈值。当前默认值覆盖95%的正常任务。

### Supervisor 语义分诊：为什么不是所有请求都跑一遍规划

如果每个请求都先过一遍 Supervisor 拆分子任务，简单问题（"JVM有哪些垃圾回收器"）也要多等一次 LLM 调用；但完全不分诊的话，复合任务（"搜索笔记A、生成思维导图、再写回笔记A"）塞进单个 AgentLoop 的 10 轮里容易轮次不够或语义漂移。

`SupervisorService.plan()`（第46行）用精确模式 LLM（temperature=0.1）返回 JSON 数组，但 LLM 的 JSON 输出并不总是"纯净"——经常夹带"根据分析："之类的中文前缀或 \`\`\`json\`\`\` 代码块标记。为此做了两层兜底：`cleanLlmJson()`（第151行）先按前缀/代码块规则清洗，失败再用 `extractJsonArray()`（第186行）按括号计数精确提取第一个数组，避免因为格式漂移直接判定规划失败、退化为单 Agent。

即便 Supervisor 成功拆出子任务，`AgentRuntime` 里还有一层二次降级：只有 1 个子任务、且不是"生成产物+写回笔记"这种强需要顺序编排的场景，也直接退化为单 Agent。"要不要走多 Agent"被拆成两层过滤，规划开销尽量只花在真正复杂的请求上。

规划结果记录到 AgentTrace.planningDecision，便于分析"哪些查询触发了多 Agent"、"规划准确率如何"。

### 动态工具扩展：为什么需要 Skill 包系统

硬编码 AgentTools 的19个 @Tool 方法无法满足所有场景：
- 用户需要调用企业内部 API（如 JIRA、GitLab、内部知识库）
- 临时需求（如"分析 CSV 文件"）不值得改代码重新部署

Skill 包系统允许：
- **Git 导入**：用户提供 Git 仓库 URL，后端 clone 并解析 skill.json 元数据，动态注册为工具
- **Skill Center 安装**：公共技能市场，用户一键安装（如"PDF 解析"、"代码静态分析"）
- **运行时注册**：SkillContextResolver 在每次请求时加载用户已安装的 Skill，合并到 activeTools
- **隔离执行**：Skill 以独立进程（Python/Shell）执行，失败不影响主服务

技术实现：
- Skill 包结构：skill.json（元数据）+ 可执行脚本（Python/Shell）
- SkillContextResolver 扫描 `~/.rag-note/skills/` 目录，解析 skill.json
- 动态构造 ToolSpecification（name/description/parameters），注入到 AgentLoop
- 工具调用时通过 ToolExecutionService 启动子进程，捕获 stdout/stderr

设计权衡：增加了安全风险（执行用户代码），通过沙箱隔离（独立进程、资源限制）+ 权限审核（Skill Center 审核机制）缓解。

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

| 类                          | 职责                                                                       | 文件大小  |
| -------------------------- | ------------------------------------------------------------------------ | ----- |
| AgentService               | 主入口，协调 *Supervisor → AgentRuntime → AgentLoop → Composer → Reviewer 全流程* | 33KB  |
| AgentLoop                  | 目标驱动的主循环，*Goal → Action → Observation → Goal Check*                      | 63KB  |
| AgentState                 | 执行状态容器：目标、证据级别、工作记忆、工具调用历史、产物、观察记录、任务状态机                                 | 22KB  |
| AgentTools                 | @Tool 注解定义的所有可用工具（19个基础工具），含检索范围过滤（setSearchFilters）                     | 49KB  |
| AgentRuntime               | 任务执行编排层，协调规划、单/多 Agent 流水线、SSE 事件发送                                      | 28KB  |
| AgentTaskService           | AgentTask 生命周期管理（创建/更新状态/查询/删除）                                          | 13KB  |
| AgentTaskResumeService     | 任务暂停/恢复机制，保存/加载 AgentExecutionSnapshot                                   | 5KB   |
| AgentEventService          | SSE 事件发送 + AgentTaskEvent 持久化                                            | 2.5KB |
| ToolExecutionService       | 工具执行统一封装，记录 AgentTaskStep，更新 AgentTrace.toolCalls                        | 9KB   |
| SupervisorService          | LLM 规划器，分析查询是否需拆分子任务                                                     | 9KB   |
| ReflectionService          | 连续无进展时 LLM 分析卡住原因，生成 ReflectionResult 持久化                                | 15KB  |
| ReplanningService          | 基于反思结果 LLM 生成新策略，更新 AgentState.pendingSteps                              | 12KB  |
| AgentPolicyService         | 预算控制，检查 BudgetConfig（轮次/时间/Token/工具调用次数）                                 | 8KB   |
| AgentTraceService          | 全链路追踪服务，记录规划/工具调用/反思/质量评审全流程                                             | 17KB  |
| WriterService              | 多 Agent 结果合成器，基于 prompt/writer_synthesize.txt                            | 3.8KB |
| ResponseComposer           | 基于证据包生成最终回答（独立 LLM 调用，prompt/compose_answer.txt）                         | 9KB   |
| GoalEvaluator              | 工具成功后自动判定 + LLM 自检目标达成 + 文本响应拦截/放行                                       | 22KB  |
| ToolResultEvaluator        | 工具结果质量评估（GOOD/POOR/ERROR）+ 错误恢复建议 + 候选策略生成                               | 9KB   |
| ContextManager             | 对话上下文压缩（Token计数32K预算 + 工具结果压缩 + 摘要压缩）                                    | 10KB  |
| ConversationContextManager | 代词引用解析 + 会话笔记上下文追踪                                                       | 2.5KB |
| EvidencePack               | 从 AgentState 提取的结构化证据包（隔离 Agent 内部上下文）                                   | 12KB  |
| QualityReviewer            | 回答质量审查（reviewAnswer）+ 检索质量审查（reviewRetrieval）+ 查询改写（rewriteQuery）        | -     |
| ModelFactory               | LLM 模型工厂（支持智谱/DeepSeek/Ollama/阿里云，三种温度预设：精确/平衡/聊天）                       | 10KB  |
| TokenCounter               | Entity Token 估算                                                          | 3.6KB |
| SkillContextResolver       | 动态工具加载，从 skill 包解析并注册额外工具                                                | -     |
| AgentMetrics               | 工具调用指标统计（频率/成功率/平均耗时）                                                    | -     |
| AgentMetricsService        | 指标收集与查询服务                                                                | -     |
| ChatController             | SSE 入口 POST /chat/agent/query/stream                                     | -     |

**实体类（持久化）：**

| 实体 | 说明 |
|------|------|
| AgentTask | 任务实体（taskId/sessionId/userId/query/status/startTime/endTime/totalSteps） |
| AgentTaskStep | 子任务步骤（stepId/taskId/goal/toolsCalled/result/duration） |
| AgentTaskEvent | 任务事件（eventType/eventData/timestamp），用于前端实时追踪 |
| AgentReflection | 反思记录（taskId/iteration/problem/suggestedStrategies/timestamp） |
| AgentTrace | 全链路追踪（taskId/sessionId/query/planningDecision/toolCalls/reflections/qualityReview/tokenUsage） |
| AgentToolMetric | 工具调用指标（toolName/callCount/successCount/avgDuration/lastCallTime） |

---

## 前端对接

前端通过 SSE（Server-Sent Events）接收实时更新：

**事件类型：**

| 事件类型 | 数据结构 | 说明 |
|---------|---------|------|
| thinking | {stage, message, iteration, goal} | Agent 执行进度（planning/running/reflecting/composing/complete） |
| response | {content, session_id} | 流式回答内容（50字符/块） |
| done | {session_id, task_id, trace_id, token_used, artifacts, total_steps, duration_ms} | 任务完成 |
| error | {message, session_id, task_id} | 任务失败 |
| task_created | {task_id, session_id, status} | 任务创建 |
| tool_call | {tool_name, args} | 工具调用开始 |
| tool_result | {tool_name, result, quality} | 工具调用完成 |

**任务追踪 API：**

- `GET /agent/tasks/{taskId}` — 查询任务详情（AgentTaskDetailResponse）
- `POST /agent/tasks/{taskId}/pause` — 暂停任务
- `POST /agent/tasks/{taskId}/resume` — 恢复任务（AgentTaskResumeRequest）
- `GET /agent/tasks` — 查询用户任务列表（AgentTaskSummaryResponse[]）

**追踪查询 API：**

- `GET /agent/traces/{traceId}` — 查询执行追踪详情
- `GET /agent/metrics/tools` — 查询工具调用指标

---

## 配置项

**BudgetConfig（预算控制）：**

```java
maxIterations: 10          // 最大轮次
maxTimeSeconds: 300        // 最大执行时间
maxTokens: 50000          // 最大 Token 消耗
maxToolCalls: 30          // 最大工具调用次数
```

**ContextManager（上下文压缩）：**

```java
MAX_CONTEXT_TOKENS: 32000  // Token 预算
COMPRESSION_THRESHOLD: 500 // 工具结果压缩阈值（字符）
MIN_RECENT_MESSAGES: 5     // 最少保留最近消息数
```

**AgentLoop（执行控制）：**

```java
MAX_ITERATIONS: 10         // 最大循环轮次
CONSECUTIVE_NO_PROGRESS: 2 // 触发反思的无进展轮次
GOAL_CHECK_TIMEOUT: 10000  // GoalEvaluator LLM 超时（毫秒）
```

**SseEmitter（连接超时）：**

```java
SSE_TIMEOUT: 300000        // 300秒（5分钟）
CHUNK_SIZE: 50             // 流式发送块大小（字符）
CHUNK_INTERVAL: 50         // 流式发送间隔（毫秒）
```
