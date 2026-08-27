# RAG 检索增强生成

## 一、项目现状

当前仓库主工程在 `LangChain-RAG-FastAPI-Service-master/`，实际是：

- Java 后端：Spring Boot 3.4 + Java 17
- 前端：Vue 3 + Vite + TypeScript，主界面在 `front-v2/`
- 主对话入口：`POST /chat/agent/query/stream`
- 兼容直连 RAG 入口：`POST /chat/rag/query`

这项目现在不是”用户提问后直接进 RAG 再直接吐回答”那种纯直线流程。主链路已经变成：

1. 前端发起 Agent 对话请求
2. `ChatController` 进入 `AgentService`
3. Agent Runtime/Loop 决定是否调用 `ragSummary` 工具
4. 工具内部再走 `RagService`
5. `RagService` 调 `HybridRetriever`
6. 检索结果回到 Agent 侧，由 `ResponseComposer` 组织答案
7. `QualityReviewer.reviewAnswer()` 做回答审查
8. 最后通过 SSE 返回前端

所以现在项目里有两条 RAG 路：

- Agent 路：主路径，带规划、工具、反思与重规划、Composer、回答审查
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

这里要分两条，不是所有文件都走同一套解析器。

#### 1. PDF：`PDFBox` 按页抽取，不走 Tika

代码：`DocumentProcessor.extractPdfTextByPage()`

实际流程：

```text
上传 PDF
  │
  ▼
PDDocument.load(file)
  │
  ▼
PDFTextStripper
  │
  ├─ setSortByPosition(true)
  ├─ 按页循环 setStartPage(page) / setEndPage(page)
  ├─ 每页单独 getText(document)
  └─ 跳过空白页
  ▼
Normalizer.normalize(pageText, NFKC)
  │
  ▼
拼成统一文本：
[Page 1/N]
...
[Page 2/N]
...
```

几个关键点：

- `setSortByPosition(true)`：让 PDF 抽出来的文本更接近页面阅读顺序，减少列错乱、段落串行问题。
- 每页单独抽：不是一次性整本提完再切。这样后面切片时可以识别页码边界。
- 每页前面都会插入 `[Page x/y]` 标记。
- 抽出来后做 `NFKC` 归一化。原因是 PDF 里常出现 Unicode 兼容字形，比如 `⽹络` 这种字形看着像“网络”，但码位不同；不归一化会影响检索匹配。

所以这里“PDF 怎么解析”答案很明确：

- 用 `PDFBox`
- 按页抽
- 保留页码标记
- 做 `NFKC` 归一化
- 后面切片再把页码映射成 `pageStart/pageEnd`

#### 2. 非 PDF：`Apache Tika parseToString(file)`

代码：`DocumentProcessor.extractText()`

真实逻辑非常直接：

```java
if (isPdf(originalFilename)) {
    return extractPdfTextByPage(file, originalFilename);
}
return tika.parseToString(file);
```

也就是：

- `.pdf` 不走 Tika
- 其余文件走 `tika.parseToString(file)`

Tika 在这里作用是“统一文件文本抽取入口”，项目里主要拿它解析：

- Word
- txt / md
- 以及其他常见办公文档/文本格式

但当前代码没有自己手写 `AutoDetectParser`、`BodyContentHandler` 那套细粒度流程，而是直接用 `Tika` 封装好的 `parseToString()`。
所以面试里可以讲成：

- 项目里 Tika 负责非 PDF 文件的自动识别和文本抽取
- 代码层没有自定义 parser pipeline
- 走的是 `Tika` 默认探测 + 默认解析器聚合能力

### 切片

知识库不再是旧版简单字符串切片。
现在主逻辑走：

- `RagChunker.splitKnowledge()`

但这块必须讲细，因为它不是“按固定 500 字硬切”那么粗。

#### 1. 知识库文档怎么切

入口：`RagChunker.splitKnowledge(text, chunkSize, chunkOverlap, isPdf)`

第一步不是直接切字符串，而是先做块级解析：

```text
原始文档文本
  │
  ▼
MarkdownBlockParser.parse(text, isPdf)
  │
  ▼
得到 block 流
  ├─ text
  ├─ code
  ├─ table
  ├─ list
  ├─ image
  └─ page_break 等
  ▼
buildKnowledgeChildren()
  │
  ├─ 普通文本：按 chunkSize/chunkOverlap 累积
  ├─ code/table/list：尽量整块保留
  ├─ 超长 code/table/list：按行切，不在表格行/代码行中间断
  ├─ 超长普通文本：走 `TextChunker.split()` 递归切
  └─ PDF 时保留页码范围
  ▼
生成 RagChunk 列表
```

核心规则：

- 知识库切片是“扁平 chunk”，没有父子 chunk 树。
- 不再依赖标题识别来决定切片边界。
- 原因是文档标题检测不稳定，特别是 PDF 纯文本里很容易误判标题，反而把 chunk 切坏。
- 所以知识库策略更保守：主要按块类型和长度切，不按章节树切。

#### 2. 普通文本怎么切

普通文本走 `TextChunker.split()`。
它不是单纯 substring，而是“按分隔符递归优先切”：

优先级大致是：

1. `\n\n`
2. `\n`
3. `。！？`
4. `.!?`
5. `；;`
6. `，,`
7. 实在不行才按字符长度硬切

所以项目真实策略是：

- 能按段落切，先按段落
- 段落太大，再按句号类标点
- 再不行才按字符长度硬切
- 切完后保留 `chunkOverlap` 尾部重叠文本

这个 overlap 不是摆设。它作用是：

- 避免一句话刚好断在 chunk 边界
- 让相邻 chunk 共享尾部上下文
- 后面检索命中某一片时，不至于丢掉刚好在边界前后的语义

#### 3. code / table / list 怎么切

这类结构化块不会像普通文本那样随便拆。

规则：

- 如果块长度不大，整块保留。
- `code/table/list` 保留阈值：`Math.max(chunkSize * 2, 1200)`。
- 超过阈值时，按“行”切，不按字符直接腰斩。
- 只有单行本身已经超过 `targetChars`，才退化成字符级切分。

这样做原因：

- 表格不能从单元格中间切开
- 代码不能从语句中间切开
- 列表最好保留条目边界

这也是当前实现和很多“纯文本 chunk”方案最大不同点之一。

#### 4. PDF 页码怎么落到 chunk 上

PDF 原文每页前面有 `[Page x/y]` 标记。
`MarkdownBlockParser.parse()` 会把页码边界识别进 block 元数据，后面 `buildKnowledgeChildren()` 在合并 block 时持续维护：

- `pageStart`
- `pageEnd`

所以最终每个知识库 chunk 不只存正文，还会带：

- `content`
- `retrievalText`
- `contentType`
- `pageStart`
- `pageEnd`

后面展示检索证据、拼上下文、标来源时，就能打出：

- `[文件: xxx]`
- `[页码: 3-4]`

#### 5. 知识库 chunk 里 `content` 和 `retrievalText` 区别

这里也要讲，不然很多人会混。

- `content`：干净正文，尽量不预埋来源前缀。
- `retrievalText`：用于向量化 / 检索的文本。

当前知识库场景里，两者基本相同，都是正文内容；只是系统设计上把“存储正文”和“检索文本”拆成两个字段，给后续扩展留口子。

来源信息比如文件名、页码，不是提前烧进 `content`，而是在 `VectorStoreService.joinKnowledgeChunks()` 拼上下文时动态补：

```text
[文件: xxx]
[页码: 2-3]
正文...
```

这样做原因是：

- 存储更干净
- 多 chunk 拼接时来源边界更清楚
- 不会把同样来源前缀重复写进每条存储内容里

#### 6. 知识库切片为什么不按 section 切

当前代码注释写得很直白：

- 文档标题检测不可靠
- 容易产生错误 section 归属
- 尤其 PDF 更明显

所以知识库这里基本放弃“章节树驱动切分”，改成：

- 扁平 block 流
- 结构块整块保留
- 普通文本按长度 + 标点 + overlap 切
- 页码作为可靠元信息保留

这是偏工程稳健性方案，不追求理论上最漂亮章节树。

### section 干嘛的

`section` 在当前项目里，主要是给笔记用，不是知识库主切分边界。

字段名实际是：

- `sectionPath`

它作用有两层。

#### 1. 入库时：给笔记 chunk 标“章节路径”

笔记走 `RagChunker.splitNote()`。
它会解析 Markdown 标题：

- `#`
- `##`
- `###`
- ...

然后维护一个 `headingStack`，把层级标题拼成：

```text
总标题 > 一级标题 > 二级标题
```

这个字符串就是 `sectionPath`。

比如笔记标题是“Redis”，正文里有：

```md
# 持久化
## RDB
```

最终某个 chunk 的 `sectionPath` 可能是：

```text
Redis > 持久化 > RDB
```

#### 2. 检索后扩上下文时：决定扩“邻居”还是扩“整节”

代码：`VectorStoreService.expandRetrievedContexts()`

当前逻辑：

- 知识库命中后，只扩 `chunk_index ± 1` 邻居。
- 笔记命中后，如果同一个 `sectionPath` 下命中超过 1 个 chunk，就直接把整节 `findByNoteIdAndSectionPath()` 拉出来拼上下文。
- 如果没有多命中，再退回邻居扩展。

也就是：

```text
知识库：neighbor 扩展
笔记：section 命中明显时整节扩展，否则 neighbor 扩展
```

所以 `sectionPath` 最大价值不是展示，而是：

- 帮笔记把检索结果提升到“章节级上下文”
- 避免只命中小片段时丢掉同节里的关键定义、步骤、例子

### 笔记与文档切片策略有什么不一样

这个问题面试很容易问，必须分清。

#### 1. 知识库文档：扁平切片，稳妥优先

特点：

- 入口 `splitKnowledge()`
- 先 `MarkdownBlockParser.parse()` 做块识别
- 不依赖标题树切边界
- 普通文本按标点/段落/长度递归切
- code/table/list 尽量整块保留
- PDF 额外保留 `pageStart/pageEnd`
- 检索后只做邻居扩展为主

适用原因：

- 上传文档格式杂
- PDF 标题检测不稳定
- 更适合“保守切法 + 页码元信息”

#### 2. 笔记：按 Markdown section 切，语义组织优先

特点：

- 入口 `splitNote()`
- 先解析 Markdown 标题层级
- 用 `headingStack` 生成 `sectionPath`
- 再按 section 内 block 切 chunk
- `retrievalText` 会显式拼上：标题、章节、类型、正文
- 检索后可按整节扩展上下文

`retrievalText` 真实结构是：

```text
标题: <title>
章节: <sectionPath>
类型: <contentType>
正文: <text>
```

这比知识库 chunk 更“语义增强”。
原因是笔记天然有作者手写结构，标题层级更可信。

#### 3. 短笔记还有特殊策略

如果 `title + content` 总长度 `<= 600`，`splitNote()` 直接整篇作为一个 chunk：

- 不再继续细切
- `sectionPath` 直接用笔记标题

原因很简单：

- 很短笔记硬切没有意义
- 反而会破坏语义完整性

#### 4. 两者本质差异总结

一句话概括：

- 文档切片更偏“格式复杂、来源异构，所以保守扁平切”。
- 笔记切片更偏“作者结构明确，所以按 section 做语义切”。

如果再口语一点：

- 文档信页码和块结构，不太信标题。
- 笔记信 Markdown 标题，所以 section 很重要。

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

## 二十二、反思与重规划机制

### 1. 触发条件

反思触发条件（`AgentState.needsReflection()`）：
- **连续无进展次数 ≥ 2**
- **或同一工具连续失败 2 次**

重规划触发条件有两处：

#### (1) AgentLoop 中的软重规划（AgentLoop.java:198-232）
- 触发时机：`state.needsReflection()` 返回 true
- 执行流程：
  1. `ReflectionService.reflect()` 分析失败原因
  2. `ReplanningService.replan()` 生成策略调整
  3. **注入状态视图到 messages**，引导 LLM 换策略
  4. **继续下一轮执行**
- 特点：不替换 SubTask 列表，而是通过提示引导 LLM 自主调整

#### (2) AgentRuntime 中的硬重规划（AgentRuntime.java:253-302）
- 触发时机：关键前置步骤（FETCH）达到 MAX_ROUNDS 且失败
- 执行流程：
  1. 检测到 `shouldTriggerReplan()` 返回 true
  2. 调用 `reflectionService.reflect()` 分析原因
  3. 记录重规划 trace
  4. **直接返回 NEED_CLARIFICATION，终止任务**
  5. 提示用户提供备用方案或切换策略
- 特点：强制终止，不生成新计划继续执行

### 2. 反思判断逻辑

`ReflectionResult.shouldReplan()` 是一个布尔字段，由两种方式生成：

#### (1) LLM 判断（主要方式）
```java
// ReflectionService.java:199
boolean shouldReplan = node.has("shouldReplan") && node.get("shouldReplan").asBoolean();
```
LLM 根据系统提示分析当前状态，返回 JSON：
```json
{
  "goalAchieved": false,
  "shouldReplan": true,
  "failureType": "URL_BLOCKED | NO_PROGRESS | REPEATED_TOOL_FAILURE | ...",
  "rootCause": "根本原因描述",
  "confidence": 0.75
}
```

#### (2) 回退逻辑判断（ReflectionService.java:247-267）
```java
// URL 白名单拦截 → 必须重规划
if (isUrlBlocked) {
    shouldReplan = true;
}
// 连续无进展 ≥ 2 → 必须重规划
else if (state.getConsecutiveNoProgress() >= 2) {
    shouldReplan = true;
}
// 工具重复失败 → 根据无进展次数决定
else {
    shouldReplan = state.getConsecutiveNoProgress() >= 2;
}
```

### 3. 重规划后效果不好的兜底机制

**当前缺少防止无限重规划的保护机制**：

- AgentLoop 有 `MAX_ITERATIONS = 10` 限制总轮次
- **但没有 `MAX_REPLAN_COUNT` 限制重规划次数**
- AgentState 中没有 `replanCount` 字段追踪重规划次数
- 理论上可能在 10 轮内反复：失败 → 反思 → 重规划 → 失败 → ...
- 最终靠 **达到 MAX_ITERATIONS** 强制终止

**实际运行中的保护**：
1. AgentLoop 软重规划通过注入提示引导 LLM，而非无限循环
2. 关键步骤失败时，硬重规划直接终止并要求用户介入
3. 10 轮迭代上限作为最终兜底

**改进建议**：
在 AgentState 中增加：
```java
private int replanCount = 0;
private static final int MAX_REPLAN_ATTEMPTS = 2;
```
并在触发重规划时检查：
```java
if (state.getReplanCount() >= MAX_REPLAN_ATTEMPTS) {
    return NEED_CLARIFICATION; // 强制用户介入
}
```

### 4. 两处重规划的区别

| 位置 | 触发条件 | 行为 | 是否继续执行 | 是否替换计划 |
|------|---------|------|------------|------------|
| **AgentLoop:212** | `state.needsReflection()` (连续失败2次) | 注入状态视图到 messages，引导 LLM 换策略 | **是**，继续下一轮 | **否**，通过提示引导 |
| **AgentRuntime:253** | 关键步骤达到 MAX_ROUNDS | 直接终止，返回澄清问题 | **否**，立即返回 | **否**，TODO 未实现 |

### 5. 重规划生成的新计划是否真正执行

**AgentLoop 中的软重规划**：
- 不生成新的 `SubTask` 列表
- 不替换 `remainingSteps`
- 而是注入 `buildStateViewForReflection(state)` 到 messages
- LLM 在下一轮根据状态视图自主选择不同工具/策略
- **本质是"提示引导"而非"硬性替换计划"**

**AgentRuntime 中的硬重规划**：
- 调用了 `agentTraceService.recordReplanning()` 记录重规划
- 但注释明确写了 `TODO: 后续可以调用 SupervisorService 重新规划`
- **当前简化处理：直接返回 NEED_CLARIFICATION，不执行新计划**
- 新规划逻辑存在但未激活

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
AgentLoop（最多 10 轮）
  │
  ├─ 简单问题可直接答
  ├─ 需要知识证据时调用 ragSummary
  └─ 连续失败 2 次触发反思与重规划
      ├─ ReflectionService 分析失败原因
      ├─ ReplanningService 生成策略调整
      ├─ 注入状态视图到 messages
      └─ 继续下一轮（LLM 自主调整策略）
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

不是”前端直接调 RAG”。
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

不是只写”MySQL + 余弦相似度降级”就够。
当前 `HybridRetriever` 真正挂载的是 `KeywordSearchService` 兜底。

### 7. 删除流程是主存储优先，Chroma 补偿

不是所有地方都”先删 Chroma 再删业务数据”。
当前核心设计是：

- MySQL/BM25 优先
- Chroma 失败进 cleanup task

### 8. Agent 主路径增加了反思与重规划机制

当前 Agent 执行失败时会触发：

- 连续失败 2 次 → 反思分析原因
- 生成策略调整 → 注入提示引导 LLM
- 关键步骤失败 → 终止并要求用户介入

---

## 二十一、面试时可直接复述版

这项目现在是 Agent 驱动 RAG，不是传统纯 RAG 直出。主入口在 `/chat/agent/query/stream`。请求先进 `AgentService`，Agent 在运行时判断要不要调用 `ragSummary` 工具；一旦调用，就进入 `RagService`。`RagService` 先做 Query 扩展，再由 `HybridRetriever` 对知识库和笔记分别做向量检索与 BM25 检索，多路结果用 RRF 融合，再交给 reranker 做 Cross-Encoder 精排。之后 `RagService` 把命中 chunk 的邻域上下文扩展开，拼参考资料 Prompt，调用模型生成知识摘要。这个摘要回到 Agent 层后，不是直接返回，而是再经过 `ResponseComposer` 组织成最终回答，并由 `QualityReviewer.reviewAnswer()` 做回答忠实度审查，最后通过 SSE 分块发给前端。

Agent 执行过程中如果连续失败 2 次，会触发反思机制：`ReflectionService` 使用 LLM 分析失败原因，判断是否需要重规划；如果需要，`ReplanningService` 生成策略调整建议，注入状态视图到对话上下文中，引导 LLM 在下一轮自主选择不同工具或参数。对于关键前置步骤（如网页抓取）失败时，系统会直接终止任务并返回澄清问题，要求用户提供备用方案。整个 AgentLoop 最多执行 10 轮，通过迭代上限防止无限循环。

入库这边，知识库文档先抽文本再走 `RagChunker.splitKnowledge()`，同步写 MySQL 和 BM25，Chroma 向量异步写；笔记则在保存 Note 后走 `splitNote()`，同步写 `NoteChunk` 和 BM25，Chroma 异步写。删除时以 MySQL 和 BM25 为主，Chroma 删除失败写入 `chroma_cleanup_task`，由定时任务做最终一致性补偿。
