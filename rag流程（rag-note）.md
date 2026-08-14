# RAG 检索增强生成

## 一、项目现状

当前仓库主工程在 `LangChain-RAG-FastAPI-Service-master/`，实际是：

- Java 后端：Spring Boot 3.4 + Java 17
- 前端：Vue 3 + Vite + TypeScript，主界面在 `front-v2/`
- 主对话入口：`POST /chat/agent/query/stream`
- 兼容直连 RAG 入口：`POST /chat/rag/query`

这项目现在不是“用户提问后直接进 RAG 再直接吐回答”那种纯直线流程。主链路已经变成：

1. 前端发起 Agent 对话请求
2. `ChatController` 进入 `AgentService`
3. Agent Runtime/Loop 决定是否调用 `ragSummary` 工具
4. 工具内部再走 `RagService`
5. `RagService` 调 `HybridRetriever`
6. 检索结果回到 Agent 侧，由 `ResponseComposer` 组织答案
7. `QualityReviewer.reviewAnswer()` 做回答审查
8. 最后通过 SSE 返回前端

所以现在项目里有两条 RAG 路：

- Agent 路：主路径，带规划、工具、Composer、回答审查
- 直连路：`chat/rag/query`，直接 `ChatService.ragQuery()`，不经过 Agent

---

## 二、当前真实调用链

### 1. 主链路：Agent + RAG

代码入口：`backend-java/src/main/java/com/rag/notebook/chat/controller/ChatController.java`

```text
前端 AIChat.vue
  │
  ▼
POST /chat/agent/query/stream
  │
  ▼
ChatController.agentQueryStream()
  │
  ▼
AgentService.streamAgentResponse()
  │
  ├─ 保存用户消息
  ├─ 处理附件上下文
  ├─ resolveReferences() 解析会话引用
  ├─ 根据开关过滤工具
  ├─ 判断是否启用 Supervisor
  ▼
AgentRuntime.start()
  │
  ▼
AgentLoop / 运行时多轮工具调用
  │
  ├─ 需要知识检索时调用 AgentTools.ragSummary()
  ▼
RagService.getDocumentsAndSummary()
  │
  ▼
HybridRetriever.searchKnowledge()/searchNotes()
  │
  ▼
QueryExpander
  │
  ▼
向量检索 + BM25 检索 + RRF 融合 + Reranker 精排
  │
  ▼
RagService 生成知识摘要
  │
  ▼
ResponseComposer.compose()
  │
  ▼
QualityReviewer.reviewAnswer()
  │
  ▼
AgentService 按 50 字符切块，通过 SSE 返回
```

### 2. 兼容链路：直连 RAG

```text
POST /chat/rag/query
  │
  ▼
ChatController.ragQuery()
  │
  ▼
ChatService.ragQuery()
  │
  ▼
RagService.ragSummary()/getDocumentsAndSummary()
  │
  ▼
直接返回结果
```

这条链路还在，但不是主入口。

---

## 三、主流程总览

### 1. 用户请求进入 Agent

`ChatController.agentQueryStream()` 会接收这些关键参数：

- `query`
- `sessionId`
- `enableKnowledge`
- `enableNotes`
- `selectedKnowledgeDocs`
- `selectedNotes`
- `fileIds`
- `regenerate`

`AgentService.streamAgentResponse()` 里会先做几件事：

1. 按开关过滤工具。`ragSummary` 只有启用知识库或笔记时才可用。
2. 把附件文本拼到用户问题前面。
3. 结合会话上下文做引用解析。
4. 判断要不要走 Supervisor 拆任务。
5. 创建 AgentTask 和 Trace。
6. 启动 `AgentRuntime.start()`。

### 2. Agent 决定是否调用 RAG

当前不是每次都强制走 RAG。是 Agent 自己在循环里决定要不要调工具。

只要问题需要知识库/笔记证据，通常会调用：

- `ragSummary`

然后 RAG 返回知识摘要，进入最终答案合成。

### 3. 最终不是 RAG 直接输出

这点要记住。

当前主流程里，`RagService` 生成的是一份“知识摘要/证据总结”，最终面向用户的输出通常还会再经过：

- `ResponseComposer.compose()`
- `QualityReviewer.reviewAnswer()`

所以用户最终看到的内容，常常不是 RAG 单次直出原文，而是 Agent 层加工后的答案。

---

## 四、当前 RAG 检索与生成流程

### 流程图

```text
用户问题
  │
  ▼
QueryExpander.expand()
  │
  ├─ 原始查询必保留
  └─ 最多再生成 3 个扩展查询
  ▼
HybridRetriever
  │
  ├─ 知识库：向量检索 + BM25
  ├─ 笔记：向量检索 + BM25
  ├─ 异常时降级到关键词检索
  └─ 多路结果做融合
  ▼
RRF 融合 / simpleMerge
  │
  ▼
RerankerService.rerank()
  │
  ▼
RagService 合并知识库和笔记结果
  │
  ▼
expandRetrievedContexts() 扩展邻域上下文
  │
  ▼
拼接参考资料上下文
  │
  ▼
调用 LLM 生成 summary
  │
  ▼
Agent 侧再组织最终回答
```

---

## 五、Query 扩展

代码：`backend-java/src/main/java/com/rag/notebook/rag/QueryExpander.java`

当前实现要点：

- 原始查询始终保留
- 最多额外生成 3 个扩展查询
- 扩展走 `creativeModel`
- 查询缓存 key 前缀：`query-cache:query-expansion`
- 缓存 30 分钟
- 输入过长时，只截前 100 个字符喂给扩展模型
- 单条扩展结果超过 50 字会被截断
- 失败时只用原始查询

### 实际行为

```text
原始 query
  │
  ├─ 查缓存，命中直接返回
  ├─ 未命中则调用 creativeModel
  ├─ 解析返回的多行文本
  ├─ 去重、去空、最多保留 3 条扩展
  └─ 和原始 query 一起组成最多 4 条查询
```

### 和旧笔记不同地方

旧笔记里“10~30 字”是提示词要求，不是硬校验规则。
真正代码约束是：

- 输入最多 100 字
- 扩展结果最多 50 字
- 总数最多 4 条（含原始查询）

---

## 六、混合检索

代码：`backend-java/src/main/java/com/rag/notebook/rag/HybridRetriever.java`

当前检索分两支：

- `searchKnowledge()`
- `searchNotes()`

两支逻辑基本同构。

### 1. 检索组件开关

`HybridRetriever` 支持这些组件开关：

- Query Expansion
- Vector Search
- BM25 Search
- RRF Fusion
- Rerank

开关来源：

- 有消融配置时优先用 `AblationConfig`
- 没有时用 `application.yml` 默认配置

### 2. 知识库检索

每个 query 版本，最多走两路：

1. 向量检索
2. BM25 检索

每路取：

- `effectiveTopK * 2`

如果指定了 `selectedKnowledgeDocs`，还会先把用户传入的 ID、md5、filename、originalFilename 归一化，再做过滤。

### 3. 笔记检索

笔记也一样：

1. 向量检索
2. BM25 检索

每路同样默认取 `topK * 2` 候选，再做融合和精排。

### 4. 为什么知识库和笔记分开检索

现在项目里不是把所有内容先扔一起再搜。
而是：

- 知识库单独检索一套
- 笔记单独检索一套
- 最后 `RagService` 才把两边结果合并排序

这样做原因很直接：

- 笔记是用户自己的内容，语义通常更贴近个人表达
- 知识库是上传文档，规模更大、噪声更高
- 分开粗排和精排，最后再统一排序，更容易控质量

---

## 七、向量检索、BM25、关键词兜底

### 1. 向量检索

代码：`VectorStoreService.searchKnowledge()`、`VectorStoreService.searchNotes()`

基本流程：

```text
查询文本
  │
  ▼
EmbeddingModel 生成 query embedding
  │
  ▼
ChromaDB collection 检索
  │
  ▼
按 user_id 过滤
  │
  ▼
返回 TopK 结果
```

知识库和笔记各用一个 collection：

- `knowledgeStore`
- `noteStore`

### 2. BM25 检索

代码：`Bm25Service`

当前是 Lucene 磁盘索引，不是内存版。

特点：

- 每个用户独立索引
- 支持中英文分词
- 文档写入和删除都跟业务数据同步走
- 检索异常时不会直接炸主流程

### 3. 关键词检索兜底

`HybridRetriever` 里已经加了关键词兜底：

- 知识库：`keywordFallbackKnowledge()`
- 笔记：`keywordFallbackNotes()`

触发场景：

1. Chroma 不可用，且向量路关闭
2. BM25 检索异常
3. 各路结果全空，需要退到纯关键词检索

这点比旧笔记更准确。旧笔记主要写了“MySQL + 余弦相似度”降级，但当前主检索流程里，真正挂在 `HybridRetriever` 上的兜底更直接的是 `KeywordSearchService`。

---

## 八、RRF 融合

代码：`HybridRetriever.rrfFusion()`

当前默认常量：

- `DEFAULT_RRF_K = 30`

不是 60。

### 公式

```text
score(d) = Σ 1 / (rrfK + rank + 1)
```

### 实际行为

- 只在多路结果大于 1 路时用 RRF
- 如果关闭 RRF，走 `simpleMerge()`
- `simpleMerge()` 是去重后按原始分数排序，不做融合分累加

### simpleMerge 不是 RRF

这个要单独记。

当前项目里，RRF 可以被消融关闭。
关闭后不是“还用别的高级融合算法”，而是很朴素地：

- 去重
- 按 similarity 排序
- 截断 topK

---

## 九、Cross-Encoder 精排

代码：`backend-java/src/main/java/com/rag/notebook/rag/RerankerService.java`

当前接智谱 rerank API。

### 流程

```text
融合后的候选文档
  │
  ▼
按文档内容拼成 documents 数组
  │
  ▼
POST {baseUrl}/rerank
  │
  ▼
拿到每条候选的 relevance_score
  │
  ▼
按 rerank_score 降序重排
  │
  ▼
动态 top-N 过滤
```

### 动态 top-N 规则

当前真实规则：

- `score > 0.7`：全部保留
- `0.5 <= score <= 0.7`：最多保留 2 条
- `score < 0.5`：丢弃

注意一处和旧笔记不同：

旧笔记写了“如果全部被过滤，返回 RRF 原始结果前 1 条兜底”。
当前 `RerankerService.dynamicTopN()` 代码里没有这段兜底逻辑。

真实情况是：

- rerank API 调用失败，返回原始 documents
- rerank API 成功但所有分数都低于阈值，可能直接返回空列表

这点很关键，旧笔记写错了。

---

## 十、RagService 如何合并结果并生成摘要

代码：`backend-java/src/main/java/com/rag/notebook/rag/RagService.java`

### 1. 检索

`RagService.retrieveDocuments()` 会按开关决定是否搜索：

- 知识库
- 笔记

然后把两边结果合并。

### 2. 排序规则

合并后排序优先级实际是：

1. `similarity`
2. 没有时看 `rerank_score`
3. 再没有时看 `rrf_score`

代码在 `scoreOf()`。

### 3. 截断

合并后统一截断到：

- `topK`

### 4. 邻域扩展

不是直接把命中 chunk 原样拿去拼 Prompt。
还会调用：

- `vectorStoreService.expandRetrievedContexts(allResults)`

也就是命中后还会扩展相邻 chunk 或同 section 上下文，减少信息断裂。

### 5. 拼接 Prompt

`RagService` 会把最终文档拼成：

- 带来源版 `buildContext()`
- 不带来源版 `buildContextPlain()`

然后构造：

```text
参考资料：
<context>

用户问题：<query>
```

再交给聊天模型生成 summary。

### 6. 结果为空时

如果检索结果为空，直接返回：

- `未找到相关文档。`

不会再强行让 LLM 编。

---

## 十一、Agent 侧为什么和纯 RAG 不一样

当前主链路里，RAG 只是证据生产环节，不是最终话术输出环节。

`AgentService.composeAndReview()` 真实流程：

1. 从 Agent 状态抽取证据 `EvidencePack`
2. `ResponseComposer.compose()` 生成回答
3. 组装审查证据列表
4. `QualityReviewer.reviewAnswer()` 审查回答
5. 审查不通过时，再次 compose 一次

所以现在主流程更像：

```text
RAG 提供证据
  +
Agent 负责表达
  +
LLM 负责忠实度复核
```

不是老式“RAG 检索完，Prompt 一拼，最终答案直接回用户”。

---

## 十二、文档入库流程

代码：`backend-java/src/main/java/com/rag/notebook/rag/DocumentProcessor.java`

### 当前真实流程

```text
用户上传文件
  │
  ▼
DocumentProcessor.processFile()
  │
  ├─ 1. 计算 MD5
  ├─ 2. 查 knowledge_document 是否已存在相同 md5
  ├─ 3. 提取文本
  ├─ 4. RagChunker.splitKnowledge() 切片
  ├─ 5. MySQL + BM25 同步写入
  └─ 6. DocumentTaskExecutor 异步写入 ChromaDB
```

### 文本提取

- PDF：`PDFBox` 按页抽取，保留 `[Page x/y]`
- 其他格式：`Apache Tika`
- PDF 文本会做 `NFKC` 归一化，避免兼容字形影响检索

### 切片

知识库不再是旧版简单字符串切片。
现在走：

- `RagChunker.splitKnowledge()`

切片里会保留：

- `content`
- `retrievalText`
- `contentType`
- `pageStart/pageEnd`
- metadata

### 写入顺序

同步阶段：

1. MySQL 写 `KnowledgeDocument`
2. MySQL 写 `KnowledgeDocumentChunk`
3. BM25 写 Lucene 索引

异步阶段：

4. `DocumentTaskExecutor.writeToChromaAsync()` 写 ChromaDB
5. 成功则文档状态完成，失败则标记 `vector_failed`

### 去重

当前代码直接查：

- `documentRepository.existsByUserIdAndMd5(userId, md5)`

旧笔记里“md5Store + MySQL 双重校验”已经不是当前主实现描述重点。
当前看代码，核心去重依据就是知识文档表里的用户级 MD5 唯一性。

---

## 十三、笔记入库流程

代码：`backend-java/src/main/java/com/rag/notebook/note/service/NoteService.java`

### 创建笔记

```text
POST /note/create
  │
  ▼
NoteService.createNote()
  │
  ├─ 1. 保存 Note 到 MySQL
  ├─ 2. 清理列表缓存
  ├─ 3. vectorStoreService.addNoteVector(note)
  ├─ 4. 注册 afterCommit 回调
  └─ 5. 事务提交后异步执行自动打标和复习记录创建
```

### addNoteVector 实际做什么

代码：`VectorStoreService.addNoteVector()`

```text
Note 内容
  │
  ▼
RagChunker.splitNote()
  │
  ├─ 写 NoteChunk 到 MySQL
  ├─ 写 BM25 索引
  └─ Chroma 可用时异步写入 noteStore
```

### 自动标签和复习

事务提交后，`asyncAutoTagAndReview()` 会：

1. 重新查 note
2. 调 LLM 生成分类和 tags
3. 更新 note
4. 创建 `ReviewRecord`

默认复习起点：

- 明天
- `intervalDays = 1`
- `reviewCount = 0`

---

## 十四、删除与一致性流程

### 1. 知识库删除

代码：`VectorStoreService.deleteKnowledgeByFilename()`

当前真实顺序：

```text
按 filename 查文档
  │
  ├─ 先删 BM25
  ├─ 再删 MySQL 文档
  └─ 最后删 ChromaDB
       ├─ 成功：结束
       └─ 失败：写 chroma_cleanup_task
```

这里和很多笔记里“先删 Chroma 再删 MySQL”不一样。
当前代码是：

- 主流程优先保证 MySQL + BM25 清掉
- Chroma 失败走补偿任务

### 2. 笔记删除

代码：`NoteService.deleteNote()`

真实顺序：

```text
删除 Note
  │
  ├─ 清缓存
  ├─ 删除 review_record
  └─ afterCommit 后调用 vectorStoreService.deleteNoteVector()
         ├─ 先查 NoteChunk
         ├─ 删除 BM25
         ├─ 删除 MySQL NoteChunk
         └─ 删除 ChromaDB，失败写 cleanup task
```

### 3. 为什么这样设计

因为当前项目明确采用：

- MySQL/BM25 优先保证主流程一致
- Chroma 用异步补偿追最终一致性

这是典型“主存储强一致，向量库最终一致”。

---

## 十五、Chroma 清理重试

代码：`backend-java/src/main/java/com/rag/notebook/rag/ChromaCleanupScheduler.java`

### 定时任务

1. 每 5 分钟跑一次 pending 任务
2. 每天凌晨 3 点清理 7 天前 failed 任务

### pending 任务处理

- `taskType = note`：删笔记向量
- 其他：按知识库文档删 Chroma 记录

失败时：

- `retryCount + 1`
- 达到 `maxRetry` 后标记 `failed`

所以当前删除策略很清楚：

- 不因为 Chroma 失败卡死业务删除
- 后台反复补偿

---

## 十六、Review 复习流程

代码：`backend-java/src/main/java/com/rag/notebook/review/service/ReviewService.java`

### 今日复习列表

接口：`GET /review/today`

逻辑：

- 查 `nextReviewAt <= now` 的记录
- 拼出笔记标题、预览、标签、分类、复习次数等
- 带缓存，TTL 10 分钟

### 标记已复习

接口：`POST /review/done/{noteId}`

当前间隔数组：

```text
[1, 2, 4, 7, 15, 30]
```

逻辑：

1. `reviewCount + 1`
2. 取下一复习间隔
3. 更新 `lastReviewedAt`
4. 更新 `nextReviewAt`
5. 清掉今日复习缓存

### 生成题目

接口：`GET /review/question/{noteId}`

流程：

- 校验 note 权限
- 截取最多 2000 字内容
- 异步调用 LLM 生成 4 选 1 单选题
- 优先按行文本格式解析，JSON 作为兜底

---

## 十七、质量审查现状

代码：`backend-java/src/main/java/com/rag/notebook/rag/QualityReviewer.java`

当前要点：

- `reviewRetrieval()` 组件存在
- 但主 RAG 流程里没有自动接入“检索失败后重写 query 再检索”闭环
- 主链路里真正用到的是 `reviewAnswer()`
- 它挂在 `AgentService.composeAndReview()` 里

所以现在“质量审查”实际含义是：

- 回答阶段有审查
- 检索阶段预留了审查能力，但还不是默认主链路

---

## 十八、评估链路现状

代码：

- `RegressionTestService`
- `EvaluationService`

当前评估不是从前端 Agent 全链路采样。
而是：

1. 回归测试直接调用 `RagService.getDocumentsAndSummary()`
2. 手动构造 `RagTrace`
3. 再交给 `EvaluationService.evaluate()`

所以它覆盖的是：

- 检索
- 生成

不覆盖：

- Supervisor
- AgentLoop
- ResponseComposer
- WriterService
- SSE 输出
- Agent 侧回答审查

这个边界要分清。

---

## 十九、当前项目最准确的一版总流程

```text
用户在前端发起问题
  │
  ▼
/chat/agent/query/stream
  │
  ▼
ChatController
  │
  ▼
AgentService
  │
  ├─ 处理会话、附件、工具开关、任务追踪
  └─ 启动 Agent Runtime
  ▼
AgentLoop
  │
  ├─ 简单问题可直接答
  └─ 需要知识证据时调用 ragSummary
  ▼
RagService
  │
  ├─ QueryExpander 生成多查询版本
  ├─ HybridRetriever 分别检索知识库和笔记
  │    ├─ 向量检索
  │    ├─ BM25
  │    ├─ 关键词兜底
  │    ├─ RRF 融合或 simpleMerge
  │    └─ Reranker 精排
  ├─ 合并知识库和笔记结果
  ├─ expandRetrievedContexts 扩展邻域上下文
  ├─ 拼接参考资料 Prompt
  └─ 调 LLM 生成 summary
  ▼
Agent 收到 summary 作为证据
  │
  ▼
ResponseComposer.compose()
  │
  ▼
QualityReviewer.reviewAnswer()
  │
  ▼
按 50 字符切块，通过 SSE 发送给前端
```

---

## 二十、和旧笔记相比，必须修正点

### 1. 主入口变了

不是“前端直接调 RAG”。
主入口是：

- `POST /chat/agent/query/stream`

### 2. 最终回答不是 RagService 直接给用户

主链路里还会经过：

- `ResponseComposer`
- `QualityReviewer.reviewAnswer()`

### 3. RRF 常量是 30，不是 60

真实代码：

- `DEFAULT_RRF_K = 30`

### 4. rerank 全部低分时，没有代码级前 1 条兜底

真实代码没有这个兜底。
只在 rerank API 失败时返回原始结果。

### 5. 文档切片不是旧版简单文本切片

当前知识库走：

- `RagChunker.splitKnowledge()`

笔记走：

- `RagChunker.splitNote()`

### 6. 主检索兜底已经落到关键词检索服务

不是只写“MySQL + 余弦相似度降级”就够。
当前 `HybridRetriever` 真正挂载的是 `KeywordSearchService` 兜底。

### 7. 删除流程是主存储优先，Chroma 补偿

不是所有地方都“先删 Chroma 再删业务数据”。
当前核心设计是：

- MySQL/BM25 优先
- Chroma 失败进 cleanup task

---

## 二十一、面试时可直接复述版

这项目现在是 Agent 驱动 RAG，不是传统纯 RAG 直出。
主入口在 `/chat/agent/query/stream`。请求先进 `AgentService`，Agent 在运行时判断要不要调用 `ragSummary` 工具；一旦调用，就进入 `RagService`。`RagService` 先做 Query 扩展，再由 `HybridRetriever` 对知识库和笔记分别做向量检索与 BM25 检索，多路结果用 RRF 融合，再交给 reranker 做 Cross-Encoder 精排。之后 `RagService` 把命中 chunk 的邻域上下文扩展开，拼参考资料 Prompt，调用模型生成知识摘要。这个摘要回到 Agent 层后，不是直接返回，而是再经过 `ResponseComposer` 组织成最终回答，并由 `QualityReviewer.reviewAnswer()` 做回答忠实度审查，最后通过 SSE 分块发给前端。

入库这边，知识库文档先抽文本再走 `RagChunker.splitKnowledge()`，同步写 MySQL 和 BM25，Chroma 向量异步写；笔记则在保存 Note 后走 `splitNote()`，同步写 `NoteChunk` 和 BM25，Chroma 异步写。删除时以 MySQL 和 BM25 为主，Chroma 删除失败写入 `chroma_cleanup_task`，由定时任务做最终一致性补偿。
