用户消息 → 加载会话历史 → 构建 System Prompt + 上下文 → Agent 执行 → 生成最终回答 → SSE 流式返回

**SSE（Server-Sent Events，服务器发送事件）**，一种**基于 HTTP 长连接的单向流式推送技术**，允许服务器持续、实时地向客户端（浏览器）推送文本数据，也是大模型对话、实时通知类场景最常用的流式输出方案。

```
用户查询
  │
  ▼
┌─────────────────────────────────────┐
│ ChatController.agentQueryStream()    │  ← /chat/agent/query/stream
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
└────────────────┬────────────────────┘
                 ▼
┌─────────────────────────────────────┐
│ SupervisorService.plan(query)        │  ← 规划器：分析查询复杂度
│ 返回子任务列表                        │
│ 单一任务（非写回类）→ 降级为单 Agent   │
│ 空列表 → 走单 Agent                  │
└──────────┬──────────────────────────┘
           │
     ┌─────┴──────┐
     ▼            ▼
  单 Agent     多 Agent 流水线
     │            │
     ▼            ├─ SEQUENTIAL: runSequentialPipeline()
                 │  子任务有依赖，逐个执行
                 │  上一步结果注入下一步
                 │
                 └─ PARALLEL: runParallelPipeline()
                    子任务独立，并行执行
                    CompletableFuture.allOf 等待全部完成
                    │
                    ▼
              WriterService.synthesize()
              合并多子任务结果
                    │
                    ▼
              QualityReviewer.reviewAnswer()
              质量审查，不达标则重新合成
     │
     ▼
┌─────────────────────────────────────┐
│ QualityReviewer.reviewAnswer()      │  ← 回答质量审查
│ 不达标 → 追加反馈重试一次             │
└────────────────┬────────────────────┘
                 ▼
         最终回答 (SSE 流式返回)
```

---

## streamAgentResponse 完整流程

入口：`ChatController.agentQueryStream()` → `AgentService.streamAgentResponse()`

### (1). 创建 SseEmitter（300秒超时）

```java
SseEmitter emitter = new SseEmitter(300000L);
```

300秒是一次 SSE 连接从建立到完成的最大存活时间（即AI单条回复消息的处理时间）。每次发消息都会建立一个新的SSE连接。

设置300秒超时是为了防止异常情况导致连接永远不关闭。Agent 多轮工具调用 + LLM 推理可能耗时较长（尤其是多 Agent 流水线），120秒可能不够。

正常情况:

async线程执行完 → emitter.complete() → 连接关闭 ✓

异常情况（没有超时）:

async线程卡死（比如 LLM API 无响应）→ 永远不 complete → 连接永远挂着 → 前端永远在等待 → Tomcat 线程资源泄漏

异常情况（有 300 秒超时）:

async线程卡死 → 300秒后自动触发超时 → 连接强制关闭 → 前端收到超时错误 → 可以提示用户重试

### (2). 捕获 SecurityContext（解决异步线程认证丢失）

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

1. **保存用户消息**（非重新生成时） → chatService.addMessage(sessionId, userId, "human", query)
2. **加载会话历史** → chatService.getSessionMessages(sessionId)，获取 List<ChatMessage>
3. **上下文压缩** → ContextManager.buildMessages()
   - 策略1：Token 计数，若在预算内（32K tokens）直接转换
   - 策略2：压缩早期长工具结果（超过500字符的 AI 回复，用 LLM 压缩到300字符内）
   - 策略3：摘要压缩——仍超预算时将早期消息整体压缩为摘要，保留最近消息原文
4. **工具过滤** → filterTools(enableKnowledge, enableNotes)
   - 知识库工具（ragSummary）受知识库开关控制
   - 笔记工具（listNotes, searchNotes, getNote 等）受笔记开关控制
   - 通用工具（whatTimeIsNow, fetchUrl, generateDiagram）始终可用
5. **构建附件上下文** → 如果有上传文件，注入附件内容到查询中

#### 3.2 Supervisor 语义分诊

```java
List<SubTask> subTasks = supervisorService.plan(query);
```

SupervisorService 用 LLM（精确模式，temperature=0.1）分析查询复杂度：

- **返回空列表** → 简单任务，走单 Agent
- **返回单一子任务且非写回类** → 降级为单 Agent（避免不必要的规划开销）
- **返回 2+ 子任务** → 多 Agent 流水线

每个子任务包含：
- goal：任务目标（如"搜索古诗词笔记并读取完整内容"）
- successCriteria：成功标准列表（如 ["mindmap"]）
- executionMode：执行模式（SEQUENTIAL / PARALLEL）
- dependsOn：依赖关系
- toolHint：建议工具（可选参考）

#### 3.3 单 Agent 路径

```
┌───────────────────────────────────────────────┐
│ AgentLoop.run() — 目标驱动的主循环               │
│                                                 │
│  第 N 轮开始:                                    │
│  │                                               │
│  ├─ 连续2轮无进展 → 注入环境状态视图（反思）       │
│  ├─ 发送 SSE thinking 事件（轮次/进度信息）        │
│  ├─ LLM 生成（精确模式），传入消息列表 + 工具定义    │
│  │                                               │
│  ├─ LLM 要求调用工具                              │
│  │  ├─ 防重复规则1：同工具同参数已失败过 → 拦截     │
│  │  ├─ 防重复规则2：刚执行过完全相同调用 → 拦截     │
│  │  ├─ 执行工具 → 获取 ToolResult                 │
│  │  ├─ ToolResultEvaluator 评估质量（GOOD/POOR/ERROR）│
│  │  ├─ 构建 Observation 注入给 LLM                │
│  │  ├─ GoalEvaluator 目标检测：                   │
│  │  │  ├─ 读笔记类目标（getNote）→ 达成即止        │
│  │  │  ├─ 产物类工具（思维导图/图表）→ 产物即完成    │
│  │  │  │  └─ 若还需写回笔记 → 继续执行             │
│  │  │  ├─ 写操作工具（create/edit/delete）→ 完成   │
│  │  │  └─ 其他工具 → 继续，让 LLM 自己判断         │
│  │  └─ 进入第 N+1 轮                              │
│  │                                               │
│  └─ LLM 返回文本（不调工具）                       │
│     ├─ 第1轮无工具调用且为反问 → NEED_CLARIFICATION│
│     ├─ GoalEvaluator 判断是否放行                  │
│     │  ├─ 目标已达成 → 放行                       │
│     │  ├─ 未调过工具 + 通用知识 → 放行             │
│     │  ├─ 未调过工具 + 笔记操作 → 拦截（注入提示）  │
│     │  ├─ 结果全 POOR/ERROR → 拦截（换策略）       │
│     │  ├─ 有成功结果 + 有目标 → LLM 自检目标是否达成│
│     │  └─ 无明确目标 → 放行（自由对话）            │
│     └─ 放行 → 循环结束                            │
│                                                 │
│  最大 10 轮                                      │
└───────────────────────────────────────────────┘
```

**AgentLoop 架构核心要素：**

| 组件 | 职责 |
|------|------|
| AgentState | 执行状态：目标、证据级别(LIST/SEARCH/CONTENT/ARTIFACT)、工作记忆、工具调用审计 |
| ToolResultEvaluator | 工具结果质量评估 + 错误码恢复建议 + 结构化反思 |
| GoalEvaluator | LLM 自检目标达成 + 写操作/产物工具自动判定完成 |
| ConversationContextManager | 代词引用解析（"这篇笔记"、"它"注入当前 noteId） |

**执行要点：**

- AgentLoop 不生成最终回答，只返回"任务是否完成"和 AgentState
- 每次工具成功后立即调用 GoalEvaluator——目标达成则直接结束循环
- 最终回答由 ResponseComposer 根据 EvidencePack 独立生成（另一轮 LLM 调用）

#### 3.4 多 Agent 流水线

**顺序执行（SEQUENTIAL）**：

适用于子任务间有依赖的场景（如"读取笔记A → 生成思维导图 → 写回原笔记"）。

```
for each 子任务:
  1. agentLoop.run() 执行当前子任务
  2. responseComposer.compose() 生成子任务结果
  3. 将上一步结果注入下一步的 query 上下文
  4. 合并 AgentState（产物、证据等）
writerService.synthesize() 整合全部结果
qualityReviewer.reviewAnswer() 质量审查
```

**并行执行（PARALLEL）**：

适用于子任务相互独立的场景（如同时搜索知识库和笔记）。

```
CompletableFuture.allOf(全部子任务).join()
  每个子任务独立 agentLoop.run()
  独立 responseComposer.compose()
合并所有 AgentState
writerService.synthesize() 整合全部结果
qualityReviewer.reviewAnswer() 质量审查
```

#### 3.5 质量审查

```java
QualityReviewer.reviewAnswer(query, documents, answer) → ReviewResult{approved, reason, feedback}
```

审查内容：
- 回答是否基于提供的证据
- 是否完整回答了用户问题
- 是否有幻觉（编造不存在的信息）

不达标时追加反馈重试一次。

#### 3.6 生成最终回答

ResponseComposer 根据 EvidencePack（从 AgentState 提取的结构化证据包）生成最终回答。

证据包结构：
- 笔记证据（去重：同笔记只保留深度最高的——完整正文 > 搜索预览 > 列表摘要）
- 知识库摘要
- 产物（思维导图、图表等）
- 操作结果（创建/编辑/删除确认）
- 笔记统计、今日复习等

### (4). SSE 响应发送

1. 保存 AI 回复到数据库 → chatService.addMessage(sessionId, userId, "ai", response)
2. 发送 SSE thinking 事件（stage: "complete" 已处理完成）
3. 分块流式发送响应（每50字符一条 response 事件，间隔50ms）

提升用户体验，方便用于前端模拟打字机效果，逐字符显示

4. 发送 SSE done 事件，包含：
   - session_id
   - trace_id（RAG trace 追踪ID）
   - token_used / token_max（Token 使用统计）
   - artifacts（产物数据：思维导图、图表等，供前端渲染）

### (5). 清理

finally 块中：
- 清除 SecurityContext（线程复用安全）
- 清除 AgentTools 检索范围配置（ThreadLocal）
- 发送错误事件（如有异常）

### (6). 立即返回 emitter（释放 Tomcat 线程）

---

## 关键设计演进

### 旧架构（文档版）
- **ClarifierService**：独立的查询澄清服务
- **processWithFunctionCalling**：LangChain4j 原生函数调用循环
- **CompletionGate**：硬编码 CompletionGate 拦截
- **最多3轮**：Agent 循环最大3轮
- **120秒超时**：SseEmitter 120秒超时

### 新架构（当前代码）
- **Clarifier 集成到 AgentLoop**：`isClarificationQuestion()` 方法内联在循环中
- **自定义 AgentLoop**：目标驱动（Goal → Action → Observation → State Update → Goal Check）
- **GoalEvaluator**：LLM 自检目标达成，替代硬编码规则
- **最多10轮**：Agent 循环最大10轮，配合反思机制
- **300秒超时**：SseEmitter 300秒超时（多 Agent 流水线更耗时）
- **证据级别**：LIST → SEARCH → CONTENT → ARTIFACT，四级证据深度
- **产物追踪**：Artifact 系统追踪思维导图、图表等产物
- **多 Agent 流水线**：支持 SEQUENTIAL 和 PARALLEL 两种执行模式

---

## 涉及类说明

| 类 | 职责 |
|----|------|
| AgentService | 主入口，协调 Supervisor → AgentLoop → Composer → Reviewer 全流程 |
| AgentLoop | 目标驱动的主循环，Goal → Action → Observation → Goal Check |
| AgentState | 执行状态容器：目标、证据级别、工作记忆、工具调用历史 |
| AgentTools | @Tool 注解定义的所有可用工具（17个） |
| SupervisorService | LLM 规划器，分析查询是否需拆分子任务 |
| WriterService | 多 Agent 结果合成器 |
| ResponseComposer | 基于证据包生成最终回答（独立 LLM 调用） |
| GoalEvaluator | LLM 自检目标达成 + 写操作/产物自动判定 |
| ToolResultEvaluator | 工具结果质量评估 + 错误恢复建议 |
| ContextManager | 对话上下文压缩（Token 计数 + 工具结果压缩 + 摘要压缩） |
| ConversationContextManager | 代词引用解析 + 会话笔记上下文追踪 |
| EvidencePack | 从 AgentState 提取的结构化证据包（隔离 Agent 内部上下文） |
| QualityReviewer | RAG 检索质量和回答质量双重审查 |
| ModelFactory | LLM 模型工厂（支持智谱/DeepSeek/Ollama/阿里云，不同温度预设） |
| TokenCounter | Token 估算 |
| ChatController | SSE 入口 /chat/agent/query/stream |
