# RAG 检索增强生成

## 一、整体流程总览

### 是什么

RAG（Retrieval-Augmented Generation）= 检索增强生成。核心思想：先从知识库中检索相关文档，再把文档交给 LLM 生成回答，而不是让 LLM 凭空回答。

### 为什么

纯 LLM 有两个致命问题：

1. **知识截止**：LLM 的训练数据有截止日期，无法回答关于最新内容的问题
2. **幻觉**：LLM 会编造看似合理但错误的内容

RAG 通过"先检索、后生成"，让 LLM 基于真实文档回答，大幅降低幻觉率。

### 本项目的 RAG 流程

```
用户提问："什么是向量数据库？"
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Query 扩展（QueryExpander）                                    │
│  └─ LLM 生成 3 个语义等价版本（不超过20字）                       │
│     Q1: "什么是向量数据库？"（原始）                              │
│     Q2: "向量数据库的定义是什么？"                                │
│     Q3: "请解释向量数据库的概念"                                  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  混合检索（HybridRetriever）- 并行执行                            │
│                                                                 │
│  知识库检索（topK = 配置值，如 5）：                              │
│  ├─ 4个查询版本 × 2（向量+BM25）= 8路检索                        │
│  ├─ RRF 融合：合并去重，按 RRF 分数排序，取 topK×2 条            │
│                                                                 │
│  笔记检索（topK = 3）：                                          │
│  ├─ 4个查询版本 × 2（向量+BM25）= 8路检索                        │
│  ├─ RRF 融合：合并去重，按 RRF 分数排序，取 topK×2 条            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ RerankerService (智谱AI Cross-Encoder)   │
│                                          │
│  1. 调用 rerank API 重新打分              │
│  2. 动态 top-N 策略：                    │
│     - score > 0.7 → 全部保留             │
│     - 0.5 ≤ score ≤ 0.7 → 最多保留2条    │
│     - score < 0.5 → 丢弃                 │
└────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  合并结果（按 rerank_score 统一排序）                             │
│  ├─ 笔记结果 + 知识库结果                                        │
│  └─ 按精排分数统一排序                                           │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  系统提示词 + 直接拼接 → LLM 生成回答                            │
│  ├─ 系统提示词：定义回答规则（只基于参考资料、引用来源等）         │
│  ├─ 参考资料：带来源标注的 chunk 内容                             │
│  └─ 一次 LLM 调用生成最终回答                                    │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  返回最终回答 + 检索结果                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、数据存储架构

### 存储层概览

```
┌─────────────────────────────────────────────────────────────────┐
│                          数据存储层                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│   │    MySQL     │    │   ChromaDB   │    │    BM25      │     │
│   │   (内容存储)  │    │  (向量存储)   │    │  (关键词索引) │     │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘     │
│          │                   │                   │             │
│   ┌──────┴───────┐    ┌──────┴───────┐    ┌──────┴───────┐     │
│   │ KnowledgeDoc │    │knowledgeStore│    │  磁盘持久化   │     │
│   │ DocChunk     │    │   noteStore  │    │data/bm25_idx │     │
│   │ Note         │    │              │    │              │     │
│   │ NoteChunk    │    │              │    │              │     │
│   └──────────────┘    └──────────────┘    └──────────────┘     │
│                                                                 │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│   │ md5_hex_store│    │ Redis        │    │ cleanup_task │     │
│   │  (MD5去重)   │    │ (缓存+黑名单) │    │ (删除重试)   │     │
│   └──────────────┘    └──────────────┘    └──────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 向量存储策略

**当前实现**：默认使用 ChromaDB，连接失败时自动降级为 MySQL + 余弦相似度

```
// VectorStoreService 构造函数
try {
    String chromaUrl = props.getChroma().getUrl();
    this.noteStore = ChromaEmbeddingStore.builder()
            .baseUrl(chromaUrl)
            .collectionName(props.getChroma().getNotesCollection())
            .build();
    this.knowledgeStore = ChromaEmbeddingStore.builder()
            .baseUrl(chromaUrl)
            .collectionName(props.getChroma().getCollection())
            .build();
    this.chromaAvailable = true;
} catch (Exception e) {
    log.warn("ChromaDB 连接失败，降级为内存向量存储: {}", e.getMessage());
    this.noteStore = null;
    this.knowledgeStore = null;
}
```

**ChromaDB 不可用时的行为**：

- 笔记添加：跳过向量写入，只写 MySQL 和 BM25
- 知识库添加：跳过向量写入，写入 MySQL 和 BM25，标记 vector_failed
- 知识库检索：自动降级到 MySQL + 余弦相似度（逐条 embedding 后计算）
- 笔记检索：返回空列表，不支持向量检索

### 笔记与知识库存储对比

|   |   |   |
|---|---|---|
|维度|知识库文档|笔记|
|**内容存储**|MySQL（KnowledgeDocument + KnowledgeDocumentChunk）|MySQL（Note + NoteChunk）|
|**向量存储**|ChromaDB（knowledgeStore）|ChromaDB（noteStore）|
|**BM25**|Bm25Service（按 chunk_id）|Bm25Service（按 chunk_id）|
|**切片策略**|按标点层级切分（chunkSize=200, overlap=20）|与知识库一致|
|**删除方式**|REST API（按 doc_id 批量删除）|REST API（按 note_id 批量删除）|
|**重试机制**|chroma_cleanup_task 表|chroma_cleanup_task 表|

---

## 三、文档处理流程（DocumentProcessor）

### 流程概述

```
用户上传文件
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  DocumentProcessor.processFile()                                │
│  1. 计算文件 MD5                                                │
│  2. MD5 去重检查（md5Store + MySQL）                             │
│  3. 文本提取（Apache Tika）     
          Tika 是 Apache 的文档解析库，支持 1000+ 种文件格式：
              .pdf   → PDFParser   → 提取文字 + 元数据
              .docx  → OOXMLParser → 提取段落
              .pptx  → OOXMLParser → 提取幻灯片内容
              .txt   → TextParser   → 直接读取
              .md    → TextParser   → 直接读取
              .html  → HtmlParser   → 提取正文
          统一接口: tika.parseToString(file) → 返回纯文本，不需要为每种格式写单独的解析代码
│  4. 文本切分（按段落/句子，支持重叠）
          长文档切成小块 → 每个块添加 [文件: 原始文件名] 前缀（用于来源追溯）
                          → 每个块作为一个检索单位存起来
  				用户提问时 → 找到最相关的块 → 把块的内容喂给 LLM → LLM 基于内容生成回答
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  VectorStoreService.addKnowledgeDocument()                      │
│                                                                 │
│  Step 1: 写入 MySQL（事务保证）                                  │
│  ├─ KnowledgeDocument（文档元数据，status="processing"）         │
│  └─ KnowledgeDocumentChunk（切片内容）                           │
│                                                                 │
│  Step 2: 写入 BM25 索引（同步，事务内）                              │
│  └─ 每个 chunk 写入 Lucene 磁盘索引                                 │
│                                                                 │
│  Step 3: 保存 MD5 记录                                              │
│ └─ md5Store.save(md5, filename, originalFilename, userId)       │
│                                                                 │
│  Step 4: 异步写入 ChromaDB（DocumentTaskExecutor，独立线程）         │
│  ├─ 批量 Embedding（每批 10 个 chunks）                              │
│  ├─ 分批写入 ChromaDB                                               │
│ └─ 成功 → status = "completed"                                    │
│      失败 → status = "vector_failed"（支持重试）                     │
```

### 详细代码流程

```
// DocumentProcessor.processFile()
public CompletableFuture<Void> processFile(File file, String originalFilename,
                                           String userId,
                                           BiConsumer<String, Object> progressCallback) {
    // ① 计算 MD5
    String md5 = computeMd5(file);
    
    // ② MD5 去重检查（双重校验）
    if (md5Store.exists(md5, userId)
            && vectorStoreService.hasKnowledgeDocument(userId, md5)) {
        progressCallback.accept("skipping", originalFilename);
        return CompletableFuture.completedFuture(null);
    }
    // MD5 存在但向量数据丢失 → 清理后重新处理
    if (md5Store.exists(md5, userId)) {
        md5Store.deleteByMd5(md5, userId);
    }
    
    // ③ Tika 解析提取文本
    String content = tika.parseToString(file);
    
    // ④ 中文感知分块（每块添加文件名前缀，便于来源追溯）
    String filePrefix = "[文件: " + originalFilename + "]
";
    List<String> chunks = splitTextWithPrefix(content, chunkSize, chunkOverlap, filePrefix);
    
    // ⑤ MySQL + BM25 同步写入（同一事务）
    String docId = vectorStoreService.addKnowledgeDocument(userId, originalFilename,
            md5, chunks, metadata, progressCallback);
    if (docId == null) return CompletableFuture.completedFuture(null);
    
    // ⑥ 保存 MD5 记录（同步）
    md5Store.save(md5, originalFilename, originalFilename, userId);
    
    // ⑦ ChromaDB 异步写入（DocumentTaskExecutor 独立线程，不占用事务连接）
    return documentTaskExecutor.writeToChromaAsync(userId, originalFilename,
            md5, docId, chunks, progressCallback);
}
```

### 切片策略详解

**问题**：英文 RAG 常用的 `RecursiveCharacterTextSplitter` 按空格/换行切分，对中文效果很差——会把一个句子从中间切断，破坏语义完整性。

**解决**：按中文标点层级切分：

- 优先级：`\n\n` → `\n` → `。` → `！` → `？` → `.` → `!` → `?` → `；` → `;` → `，` → `,`

**为什么需要 overlap（重叠）**：

```
chunk1: "...机器学习是人工智能的一个子集，它通过"
chunk2: "它通过数据训练模型来做出预测..."
        ^^^^^^^^ 重叠部分 ^^^^^^^^
```

没有 overlap，"它通过"这个指代关系就断了。检索到 chunk2 时 LLM 不知道"它"指什么。

### MD5 去重

**MD5 特性**：

- 内容完全相同 → MD5 相同 → 跳过
- 内容有一点点不同（如多一个字）→ MD5 完全不同 → 不会跳过

**双重校验**：

```
if (md5Store.exists(md5, userId) && vectorStoreService.hasKnowledgeDocument(userId, md5)) {
    // 跳过：MD5 存在且向量数据存在
}
```

---

## 四、笔记创建流程（NoteService）

### 流程概述

```
前端 NoteEditor.vue 点击"保存"
│  POST /note/create
│  { title: "RAG学习笔记", content: "# RAG\n\nRAG是..." }
▼
NoteController.createNote()
│  @UserId userId = "user123"
▼
NoteService.createNote(userId, request)                ← @Transactional管理事务
│
├─ ① 创建 Note 实体
│     id = UUID (32位)
│     userId = "user123"
│     title = "RAG学习笔记"
│     content = "# RAG\n\nRAG是..."
│     category = null（用户没传）或 "study"（用户传了）
│     tags = null（还没生成）
│
├─ ② 保存到 MySQL
│     noteRepository.save(note)
│     → INSERT INTO notes (id, user_id, title, content, ...) VALUES (...)
│
├─ ③ 添加向量索引（切片存储）
│     vectorStoreService.addNoteVector(note)
│     → 文本切片（chunkSize=200, overlap=20）
│     → 同步写入 MySQL NoteChunk 表
      → 同步写入 BM25 索引
│     → 异步写入 ChromaDB（noteStore，@Async，失败不影响主流程）
│     → 写入 BM25 索引
│     → try-catch 包裹，失败不影响主流程
│
├─ ④ 注册事务提交后回调
│     TransactionSynchronizationManager.registerSynchronization(
│         afterCommit → self.asyncAutoTagAndReview(noteId, userId, category)
│     )
│     → 不是立即执行，而是等事务提交后才触发
│     → 如果直接调用，会因为当前事务还未提交导致异步任务在MySQL中查不到笔记
│
├─ ⑤ 返回 NoteResponse 给前端
│     { id, title, content, tags:null, category:null, createdAt }
│
│   ===== 事务提交 =====
│
└─ ⑥ 异步执行 asyncAutoTagAndReview()（taskExecutor 线程池） 
    │
    ├─ 6a. 从 MySQL 重新查询笔记（确保数据已提交）
    │
    ├─ 6b. 调用 LLM 自动生成标签和分类
    │       prompt = "请根据以下笔记内容，返回JSON格式的分类结果..."
    │       LLM 返回: {"category":"study","tags":["RAG","向量检索","LLM"]}
    │
    ├─ 6c. 解析 LLM 返回的 JSON
    │       parseCategory() → "study"（四个默认分类"work", "study", "life", "project"）
    │       parseTags() → ["RAG", "向量检索", "LLM"]
    │       如果用户指定了分类 → 优先用用户的
    │
    ├─ 6d. 更新笔记的 category 和 tags
    │       noteRepository.save(note)
    │       → UPDATE notes SET category='study', tags='["RAG","向量检索","LLM"]' WHERE id=...
    │
    └─ 6e. 创建复习记录
            ReviewRecord:
              noteId = 笔记ID
              userId = 用户ID
              nextReviewAt = 明天（now + 1天）
              intervalDays = 1
              reviewCount = 0
            reviewRecordRepository.save(record)
```

### 关键代码

```
@Transactional
public NoteResponse createNote(String userId, NoteCreate request) {
    // ① 创建 Note 实体
    Note note = new Note();
    note.setId(UUID.randomUUID().toString().replace("-", ""));
    note.setUserId(userId);
    note.setTitle(request.getTitle());
    note.setContent(request.getContent());
    if (request.getCategory() != null && !request.getCategory().isEmpty()) {
        note.setCategory(request.getCategory());
    }
    
    // ② 保存到 MySQL
    note = noteRepository.save(note);
    
    // ③ 同步添加向量索引（切片存储）
    try {
        vectorStoreService.addNoteVector(note);
    } catch (Exception e) {
        log.warn("Failed to add note vector: {}", e.getMessage());
    }
    
    // ④ 注册事务提交后回调
    String noteId = note.getId();
    String category = request.getCategory();
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            self.asyncAutoTagAndReview(noteId, userId, category);
        }
    });
    
    // ⑤ 返回响应
    return toResponse(note);
}
```

### @Transactional 工作机制

```
Spring 为标注了 @Transactional 的 Bean 生成代理对象（JDK 动态代理或 CGLIB 代理）

调用流程：
外部调用 → 代理对象 → 开启事务 → 执行目标方法 → 正常完成提交事务
                                              → 抛出异常回滚事务

本质是通过环绕通知（Around Advice）包裹目标方法，把事务管理逻辑和业务逻辑完全解耦
```

---

## 五、向量检索与 BM25 检索

### 5.1 向量检索

**文件路径**：`VectorStoreService.java`

```
查询文本 → Embedding 模型（智谱 embedding-3）→ 查询向量
    │
    ▼
ChromaDB 向量检索（带 user_id 过滤）→ Top K 结果
```

**用户隔离**：

```
Filter filter = new IsEqualTo("user_id", userId);
EmbeddingSearchRequest request = EmbeddingSearchRequest.builder()
        .queryEmbedding(queryEmbedding)
        .maxResults(topK)
        .filter(filter)
        .build();
```

### 5.2 BM25 检索

**文件路径**：`Bm25Service.java`

- 基于 Lucene，每个用户独立的**磁盘持久化索引**（`FSDirectory`）
- 使用 `StandardAnalyzer`（支持中英文分词）
- 查询时自动转义特殊字符，防止 Lucene 查询语法注入

### 5.3 知识库和笔记分开检索

**问题**：知识库和笔记是两种不同性质的数据。笔记是用户自己写的，跟用户意图更相关；知识库是外部上传的文档。

**解决**：分开检索，分开精排，最后按分数统一排序。

```
知识库检索 → RRF 融合 → Cross-Encoder 精排 → topK 条
笔记检索 → RRF 融合 → Cross-Encoder 精排 → topK 条
    │
    ▼
按 RRF 分数统一排序，取最终 topK
```

---

## 六、RRF 融合排序

### 是什么

RRF（Reciprocal Rank Fusion）= 逆排名融合。一种将多路检索结果合并排序的算法。

RRF 的核心思想：

如果一个文档在多路中都排名靠前，它很可能是好文档

如果一个文档只在一路中排名靠前，可能只是那一路的特殊偏好

RRF 通过累加 1/(60+rank) 来奖励"在多路中出现"的文档

### RRF 公式

```
score(d) = Σ 1/(k + rank_i(d))

其中：
- d = 文档
- k = 常数 60（经验值，避免排名靠前的文档分数过高）
- rank_i(d) = 文档 d 在第 i 路检索结果中的排名（从 1 开始）
- Σ = 对所有路的分数求和
```

### 实现细节

```
private List<Map<String, Object>> rrfFusion(List<List<Map<String, Object>>> allRankings, int topK) {
    Map<String, Double> rrfScores = new HashMap<>();
    Map<String, Map<String, Object>> docCache = new HashMap<>();

    for (List<Map<String, Object>> ranking : allRankings) {
        for (int rank = 0; rank < ranking.size(); rank++) {
            Map<String, Object> doc = ranking.get(rank);
            String docKey = getDocKey(doc);

            double rrfScore = 1.0 / (RRF_K + rank + 1);
            rrfScores.merge(docKey, rrfScore, Double::sum);

            docCache.putIfAbsent(docKey, doc);
        }
    }

    return rrfScores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(topK)
            .map(entry -> docCache.get(entry.getKey()))
            .collect(Collectors.toList());
}
```

---

## 七、Cross-Encoder 精排

### 是什么

对 RRF 融合后的结果使用智谱 AI 的 rerank API 进行重新打分和排序，并过滤低相关性结果。

Rerank模型（Cross-Encoder）会把用户问题和每个候选片段拼在一起输入，深度理解它们之间的语义匹配程度，重新打分排序。最终只保留top-N的高质量片段，把噪声过滤掉。

**那为什么不直接用rerank模型来检索，还要先粗排再精排？**

因为rerank是cross-encoder结构，需要把查询和每个候选拼在一起过模型，计算量比向量检索大得多。如果拿它对百万条数据逐一算分，延迟完全不可接受。所以工程上采用【粗排筛到几十条，精排再从几十条里挑最好的几条】这种两阶段策略，兼顾速度和质量。rerank整体耗时通常在几百毫秒砂以内，对用户体感影响不大，但检索质量的提升非常明显。

### 向量检索 vs Cross-Encoder 的区别

|   |   |   |
|---|---|---|
|维度|向量检索（Bi-Encoder）|Cross-Encoder|
|编码方式|query 和 document 分别编码|query 和 document 一起编码|
|计算过程|Q → 向量 q，D → 向量 d，计算相似度|(Q, D) → 模型 → 相关性分数|
|精度|中等|高|
|速度|快（可预计算 document 向量）|慢（每次都要重新计算）|

**为什么 Cross-Encoder 更准？**

可以捕捉 token 级别的语义交互关系。

例如：查询 "怎么记住东西"，文档 "间隔重复是一种记忆方法"

- 向量检索：二者用词完全不同，可能匹配失败
- Cross-Encoder：可以识别出 "记住" 和 "记忆" 的语义对应关系，准确判定高相关性

```
输入：[CLS] 查询文本 [SEP] 文档文本 [SEP]
      ↓
┌─────────────────────────────────────────┐
│  Transformer Encoder                     │
│  ├─ Self-Attention：query 和 document的每个 token 互相 attend │
│  │                                       │
│  │  例如：                               │
│  │  "什么是向量数据库" 的 "向量"         │
│  │  可以 attend 到文档中的 "高维向量"    │
│  │                                       │
│  └─ 输出：[CLS] token 的表示             │
└─────────────────────────────────────────┘
      ↓
┌─────────────────────────────────────────┐
│  分类头（Linear Layer）                  │
│  └─ 输出：相关性分数（0-1）              │
└─────────────────────────────────────────┘

示例：
输入："什么是向量数据库" + "向量数据库是一种存储高维向量的数据库"
输出：0.95

输入："什么是向量数据库" + "关系型数据库使用SQL查询"
输出：0.12
```

```
RRF 融合结果（如 10 条）
    │
    ▼
调用 rerank API 获取精排分数
    │
    ▼
阈值过滤（score < 0.5 的丢弃）
    │
    ▼
按精排分数重新排序
    │
    ▼
取 topN（如 5 条）
```

动态 top-N 策略：

分数 > 0.7：全部保留

分数 0.5-0.7：最多保留 2 条

分数 < 0.5：丢弃（如果全部被过滤 → 返回RRF原始结果前1条兜底）

### 配置

```
app:
  reranker:
    enabled: true
    type: ZHIPU
    api-key: ${ZHIPU_API_KEY}
    base-url: https://open.bigmodel.cn/api/paas/v4
    model: rerank
    score-threshold: 0.5
    top-n: 5
```

### 兜底策略

rerank API 调用失败时，返回原始 RRF 结果，不阻断流程。

---

## 八、删除流程

### 知识库文档删除

```
deleteKnowledgeByFilename()
    │
    ├─ 1. 尝试删除 ChromaDB
    │      ├─ 成功 → 继续
    │      └─ 失败 → 记录到 chroma_cleanup_task 表
    │
    ├─ 2. 删除 BM25 索引（按 chunk_id）
    │
    ├─ 3. 删除 MySQL（chunks + document）
    │
    └─ 4. 删除 MD5 记录
```

### 笔记删除

```
NoteService.deleteNote()
    │
    ├─ 1. 从 MySQL 删除 Note 记录（@Transactional 事务内）
    │
    ├─ 2. 删除关联的复习记录（ReviewRecord）
    │
    ├─ 3. 事务提交后 → deleteNoteVector()（异步）
    │     ├─ 从 MySQL 删除 NoteChunk
    │     ├─ 删除 BM25 索引（按 chunk_id）
    │     └─ 尝试删除 ChromaDB（按 note_id 批量删除）
    │            ├─ 成功 → 继续
    │            └─ 失败 → 记录到 chroma_cleanup_task 表
    │                      （taskType="note"，定时重试）
    │
    └─ 注意：向量操作失败不阻塞主流程（事务已提交）
```

### 定时清理任务

```
ChromaCleanupScheduler
    │
    ├─ 每 5 分钟执行
    │  └─ 查询 chroma_cleanup_task 表中 status="pending" 的任务
    │     ├─ taskType="knowledge" → REST API 删除 ChromaDB
    │     └─ taskType="note" → noteStore.remove()
    │
    └─ 每天凌晨 3 点
       └─ 清理超过 7 天的 failed 任务
```

---

## 九、间隔复习流程（ReviewService）

### 流程概述

```
笔记创建时（自动）
  │
  └─ NoteService.asyncAutoTagAndReview()
     └─ 创建 ReviewRecord: nextReviewAt=明天, intervalDays=1, reviewCount=0

        ... 1天后 ...

用户打开"每日回顾"页面
  │
  ├─ GET /review/today
  │   └─ 查询 nextReviewAt <= 现在 的所有记录
  │   └─ 返回待复习笔记列表
  │
  ├─ 用户点击某篇笔记
  │   └─ GET /review/question/{noteId}
  │       └─ 生成复习选择题
  │
  ├─ 用户选择答案 → 显示对错
  │
  └─ 用户点击"标记已回顾"
      └─ POST /review/done/{noteId}
          └─ reviewCount + 1 → 查间隔表 → 更新 nextReviewAt
```

### 复习间隔序列

```
1 → 2 → 4 → 7 → 15 → 30 天（艾宾浩斯遗忘曲线，遗忘先快后慢）
```

---



---

## 十、质量审查（QualityReviewer）【可选，默认关闭】

### 概述

`QualityReviewer` 是一个可选的质量审查组件，可在检索和生成两个阶段对 RAG 输出进行 LLM 驱动的质量审查。默认关闭（`retrieval-review-enabled: false`）。

### 审查流程

```
用户问题 + 检索结果/生成的回答
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  QualityReviewer                                                 │
│                                                                 │
│  reviewRetrieval(query, documents)                               │
│  ├─ 检查文档是否为空 → 空则直接不通过                             │
│  ├─ 调用 LLM（preciseModel）审查文档质量                          │
│  ├─ 返回 ReviewResult(approved, reason, feedback)                │
│  └─ feedback 可用于 rewriteQuery() 改写查询后重试                │
│                                                                 │
│  reviewAnswer(query, documents, answer)                          │
│  ├─ 检查回答是否为空 → 空则直接不通过                             │
│  ├─ 调用 LLM（preciseModel）审查回答忠实度                       │
│  └─ 返回 ReviewResult(approved, reason, feedback)                │
│                                                                 │
│  rewriteQuery(query, feedback)                                   │
│  └─ 根据审查反馈，用 LLM 改写查询后重新检索                       │
└─────────────────────────────────────────────────────────────────┘
```

### 关键特性

- **双阶段审查**：检索阶段审查文档相关性，生成阶段审查回答质量
- **查询改写**：审查不通过时，可基于 LLM 反馈改写查询自动重试
- **默认关闭**：通过 `app.ablation.rag.retrieval-review-enabled` 控制开关（默认 false）
- **异常安全**：LLM 调用异常时默认通过（approved=true），不阻断主流程

### 配置

```yaml
app:
  ablation:
    rag:
      retrieval-review-enabled: false  # 默认关闭
```

---

## 十一、SecurityFilterChain 工作机制

```
HTTP 请求进入 Tomcat
    │
    ▼
┌─ SecurityFilterChain 过滤链 ──────────────────────────────┐
│                                                           │
│  Filter 1: CsrfFilter                                     │
│    └─ 已禁用 (csrf.disable())，直接放行
              CSRF（Cross-Site Request Forgery，跨站请求伪造）：
              典型风险存在于 Cookie + Session 认证场景。攻击者利用浏览器自动携带目标站点 
              Cookie 的特性，诱导用户在已登录状态下，从第三方网站发起非自愿的恶意请求。
              
              本项目采用 JWT 无状态认证模式，不依赖 Cookie 会话，因此禁用 CSRF 防护。
│                                                           │
│  Filter 2: JwtAuthenticationFilter（自定义）               │
│    （插在 UsernamePasswordAuthenticationFilter 之前）      │
│    ├─ 提取 Authorization: Bearer xxx                      │
│    ├─ 解析 JWT                                            │
│    │   ├─ 验证签名（HMAC-SHA256，密钥来自配置文件）        │
│    │   ├─ 检查是否过期                                    │
│    │   └─ 检查 Redis 黑名单（是否已登出）                  │
│    ├─ 有效 → SecurityContext.setAuthentication(userId)     │
│    └─ 无效 → 不设置，交给后续拦截                          │
│                                                           │
│  Filter 3: UsernamePasswordAuthenticationFilter           │
│    └─ 项目中未自定义，使用默认行为                          │
│                                                           │
│  Filter 4: ExceptionTranslationFilter                     │
│    └─ 认证/授权异常时，交给 GlobalExceptionHandler 处理    │
│                                                           │
│  Filter 5: FilterSecurityInterceptor                      │
│    └─ 根据 authorizeHttpRequests 的配置判断权限            │
│       ├─ /user/login → permitAll → 放行                   │
│       ├─ /user/register → permitAll → 放行                │
│       ├─ /health/** → permitAll → 放行                    │
│       └─ 其他 → authenticated                             │
│          ├─ SecurityContext 有认证信息 → 放行              │
│          └─ SecurityContext 无认证信息 → 返回 401          │
│                                                           │
└─────────────────────────────┬─────────────────────────────┘
                              │
                              ▼
                    Controller 方法执行
```

**SSE/异步支持**：`SecurityContextHolder` 设置为 `MODE_INHERITABLETHREADLOCAL`，确保异步线程（SSE、CompletableFuture）也能继承认证信息，避免异步场景下 SecurityContext 丢失。

---


---

## 十二、完整数据流图

```
用户问题: "怎么提升记忆力"
    │
    └─── [Multi-Query] ────────────────────────────────────────
         LLM 改写为 3 个语义等价版本（不超过20字）:
         Q1: "记忆增强的方法有哪些"
         Q2: "提高记忆效率的技巧"
         Q3: "如何改善记忆能力"
         Q0: "怎么提升记忆力"（原始查询，始终保留）
                                                
                         ▼                
┌───────────────────────────────────────────────────────────────────┐
│                  混合检索（HybridRetriever）                        │
│                                                                   │
│  知识库检索:                                                       │
│  Q0 ──┬── 向量检索(topK×2) ── BM25(topK×2)                        │
│  Q1 ──┬── 向量检索(topK×2) ── BM25(topK×2)                        │
│  Q2 ──┬── 向量检索(topK×2) ── BM25(topK×2)                        │
│  Q3 ──┬── 向量检索(topK×2) ── BM25(topK×2)                        │
│          │                                                        │
│          ▼ 最多 8×topK 个候选                                      │
│        RRF 融合 → 去重 → 按 RRF 分数排序 → topK×2 条               │
│          │                                                        │
│          ▼ Cross-Encoder 精排（rerank API）                        │
│          ├─ 动态 top-N：score>0.7 全保留                           │
│          ├─ 0.5≤score≤0.7 最多保留2条                             │
│          └─ score<0.5 丢弃（全部过滤则取 RRF 第1条兜底）           │
│        最终 topK 条知识库文档                                       │
│                                                                   │
│  笔记检索:（同理，topK 可独立配置）                                  │
│        RRF 融合 → 精排 → 最终 topK 条笔记                          │
└──────────────────────────────┬────────────────────────────────────┘
                               │
                               ▼
                      合并结果，按 similarity （余弦相似度）
                      降序排列，统一取 topK
                               │
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│  构建 Prompt（直接拼接，无 MapReduce）                              │
│                                                                   │
│  带来源标注版（默认）:                                              │
│  [来源：笔记《xxx》] 内容...                                       │
│  [来源：知识库《xxx》] 内容...                                     │
│                                                                   │
│  无来源标注版（消融实验 R-6）:                                     │
│  直接拼接文档内容                                                  │
└──────────────────────────────┬────────────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│  系统提示词 + 拼接好的参考资料 + 用户问题 → LLM 一次调用生成回答    │
│                                                                   │
│  System: 你是一个智能笔记助手...                                    │
│  User:   参考资料：...

用户问题：...                             │
│                                                                   │
│  ↓ 一次 LLM 调用                                                   │
│                                                                   │
│  最终回答:                                                         │
│  "提升记忆力有几种经过验证的方法：                                   │
│   1. 间隔重复：按照遗忘曲线安排复习时间...                           │
│   2. 充足睡眠：睡眠期间大脑会巩固记忆...                             │
│   3. 记忆宫殿：将信息与空间位置关联..."                              │
│                                                                   │
│  来源: 笔记《学习方法总结》、知识库《认知心理学》                     │
└───────────────────────────────────────────────────────────────────┘
```

---

## 十三、三端数据同步流程详解

---

### 同步上传流程

#### 知识库文档上传

```
用户上传文件
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  DocumentProcessor.processFile()                                │
│  ├─ 计算 MD5                                                    │
│  ├─ MD5 去重检查（跳过已存在的文档）                              │
│  ├─ Tika 解析提取文本                                            │
│  └─ 中文感知切片                                                 │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  VectorStoreService.addKnowledgeDocument()  【同一事务】         │
│                                                                 │
│  ① 写入 MySQL（同步，事务保证）                                  │
│     ├─ INSERT knowledge_document（status="processing"）         │
│     └─ INSERT knowledge_document_chunk（多条）                  │
│                                                                 │
│  ② 写入 BM25 索引（同步，同一事务）                              │
│     └─ 逐个 chunk 写入 Lucene 磁盘索引                          │
│                                                                 │
│  ③ 保存 MD5 记录（同步）                                         │
│     └─ md5Store.save(md5, filename, userId)                     │
│                                                                 │
│  ===== 事务提交 =====                                            │
│                                                                 │
│  ④ 异步写入 ChromaDB（DocumentTaskExecutor，独立线程）          │
│     ├─ 批量 Embedding（每批 10 条）                              │
│     ├─ 分批写入 ChromaDB                                         │
│     ├─ 成功 → 更新 status = "completed"                         │
│     └─ 失败 → 更新 status = "vector_failed"                     │
└─────────────────────────────────────────────────────────────────┘
```

**关键点**：

- MySQL 和 BM25 写入在**同一事务**中，保证原子性
- MD5 记录也在事务后同步保存
- ChromaDB 写入是**异步**的（DocumentTaskExecutor），失败不影响主流程
- 状态在 `addKnowledgeDocument` 中设为 `processing`，由 `DocumentTaskExecutor` 异步更新为 `completed` 或 `vector_failed`

---

#### 笔记创建

```
用户创建笔记
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  NoteService.createNote()  【@Transactional】                   │
│                                                                 │
│  ① 写入 MySQL                                                   │
│     ├─ INSERT note（标题、内容、分类）                            │
│     └─ INSERT note_chunk（切片内容，多条）                       │
│                                                                 │
│  ② 添加向量索引（addNoteVector）                                │
│     ├─ 文本切片（chunkSize=200, overlap=20）                     │
│     ├─ 写入 MySQL NoteChunk（同步）                              │
│     ├─ 写入 BM25 索引（同步）                                     │
│     └─ 异步写入 ChromaDB（writeNoteToChromaAsync）               │
│                                                                 │
│  ③ 注册事务提交后回调                                            │
│     └─ afterCommit → asyncAutoTagAndReview()                    │
│        ├─ LLM 自动生成标签和分类                                 │
│        └─ 创建复习记录                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

### 同步删除流程

#### 知识库文档删除

```
用户删除文档
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  VectorStoreService.deleteKnowledgeByFilename()  【@Transactional】
│                                                                 │
│  ① 查询 MySQL 获取文档信息                                       │
│     ├─ docId（文档ID）                                           │
│     ├─ md5（内容哈希）                                           │
│     └─ userId（用户ID）                                          │
│                                                                 │
│  ② 尝试删除 ChromaDB                                            │
│     ├─ 调用 REST API：POST /api/v1/collections/{id}/delete      │
│     ├─ 请求体：{"where": {"doc_id": "xxx"}}                     │
│     ├─ 成功 → 继续                                               │
│     └─ 失败 → 记录到 chroma_cleanup_task 表                     │
│        └─ INSERT cleanup_task (docId, userId, taskType="knowledge")
│                                                                 │
│  ③ 删除 BM25 索引（同步）                                       │
│     └─ 遍历所有 chunk，逐个删除                                  │
│        for (chunk : chunks) {                                   │
│            chunkKey = md5 + "_" + chunkIndex                    │
│            bm25Service.deleteDocument(userId, chunkKey)         │
│        }                                                        │
│                                                                 │
│  ④ 删除 MySQL（同步）                                           │
│     ├─ DELETE knowledge_document_chunk WHERE document_id = ?    │
│     └─ DELETE knowledge_document WHERE id = ?                   │
│                                                                 │
│  ⑤ 删除 MD5 记录                                                │
│     └─ md5Store.deleteByMd5(md5, userId)                        │
└─────────────────────────────────────────────────────────────────┘
```

**关键点**：

- **ChromaDB 删除（REST API）**首先尝试，失败不阻塞主流程，记录到 cleanup_task 表待重试
- ChromaDB 失败时记录到 `cleanup_task` 表，定时重试
- BM25 删除是**同步**的，按 chunk_id 逐个删除

---

#### 笔记删除

```
用户删除笔记
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  NoteService.deleteNote()（外层事务）                             │
│                                                                 │
│  ① 删除 MySQL 中的 Note 记录                                    │
│     └─ DELETE note WHERE id = ?                                 │
│                                                                 │
│  ② 删除关联的复习记录（ReviewRecord）                            │
│     └─ DELETE review_record WHERE note_id = ?                   │
│                                                                 │
│  ===== 事务提交 =====                                            │
│                                                                 │
│  ③ 事务提交后 → deleteNoteVector()                              │
│                                                                 │
│  VectorStoreService.deleteNoteVector()：                        │
│  ├─ ④ 查询 MySQL 获取 chunks（用于 BM25 删除键）                │
│  │     └─ SELECT note_chunk WHERE note_id = ?                   │
│  ├─ ⑤ 删除 BM25 索引（按 chunkIndex）                           │
│  │     └─ bm25Service.deleteDocument(userId, chunkKey)          │
│  ├─ ⑥ 从 MySQL 删除 NoteChunk                                    │
│  │     └─ DELETE note_chunk WHERE note_id = ?                   │
│  └─ ⑦ 尝试删除 ChromaDB（REST API）                             │
│        ├─ 调用 POST /api/v1/collections/{id}/delete             │
│        ├─ 成功 → 继续                                            │
│        └─ 失败 → 记录到 chroma_cleanup_task 表                  │
│           └─ INSERT cleanup_task（taskType="note"）             │
└─────────────────────────────────────────────────────────────────┘
```

---

### 同步更新流程

#### 笔记更新

```
用户编辑笔记
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  NoteService.updateNote()  【@Transactional】                   │
│                                                                 │
│  ① 检查内容是否变化                                              │
│     ├─ 标题变化 → 需要更新向量                                   │
│     ├─ 内容变化 → 需要更新向量                                   │
│     └─ 都没变 → 跳过向量更新                                     │
│                                                                 │
│  ② 如果内容变化：删除旧向量                                      │
│     ├─ deleteNoteVector(noteId, userId)                         │
│     │   ├─ 删除 MySQL NoteChunk                                 │
│     │   ├─ 删除 ChromaDB 向量                                   │
│     │   └─ 删除 BM25 索引                                       │
│     └─ addNoteVector(note)                                      │
│         ├─ 切片 → 写入 MySQL NoteChunk                          │
│         ├─ 异步写入 ChromaDB                                     │
│         └─ 写入 BM25 索引                                       │
│                                                                 │
│  ③ 更新 MySQL Note 表                                           │
│     └─ UPDATE note SET title=?, content=?, ... WHERE id=?       │
└─────────────────────────────────────────────────────────────────┘
```

**关键点**：

- 笔记更新是**先删后增**，不是直接更新向量
- 只有内容变化才触发向量更新
- 标签和分类更新不触发向量重建

---

#### 知识库文档更新

**当前实现**：没有直接的更新逻辑

```
用户重新上传同名文件
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  DocumentProcessor.processFile()                                │
│                                                                 │
│  ① 计算 MD5                                                     │
│     └─ 新文件的 MD5 = "abc123"                                  │
│                                                                 │
│  ② MD5 去重检查                                                 │
│     ├─ md5Store.exists(md5, userId) → true（旧文件 MD5 相同）    │
│     └─ vectorStoreService.hasKnowledgeDocument() → true         │
│     → 跳过，不处理                                               │
│                                                                 │
│  如果内容不同（MD5 不同）：                                       │
│     ├─ md5Store.exists() → false                                │
│     └─ 当作新文件处理，旧数据不会被删除                           │
└─────────────────────────────────────────────────────────────────┘
```

**问题**：

- 同名文件、内容不同 → MD5 不同 → 当作新文件，旧数据残留
- 没有按 filename 更新的逻辑

---

### 定时重试流程

```
ChromaCleanupScheduler（每 5 分钟执行）
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  查询 cleanup_task 表                                            │
│  └─ SELECT * FROM chroma_cleanup_task WHERE status = 'pending'  │
│                                                                 │
│  遍历每个任务：                                                   │
│  ├─ taskType = "knowledge"                                      │
│  │   └─ vectorStoreService.deleteFromChromaByDocId(docId)       │
│  │       └─ 调用 REST API 删除 ChromaDB                         │
│  │                                                              │
│  └─ taskType = "note"                                           │
│      └─ vectorStoreService.deleteNoteFromStore(noteId)          │
│          └─ noteStore.remove(noteId)                            │
│                                                                 │
│  结果处理：                                                       │
│  ├─ 成功 → DELETE cleanup_task WHERE id = ?                     │
│  └─ 失败 → UPDATE retry_count + 1                              │
│          └─ retry_count >= 10 → status = 'failed'              │
└─────────────────────────────────────────────────────────────────┘

每天凌晨 3 点清理过期任务
    │
    ▼
DELETE FROM chroma_cleanup_task 
WHERE status = 'failed' AND created_at < NOW() - 7 DAY
```

---

### 三端数据状态对照表

|   |   |   |   |
|---|---|---|---|
|场景|MySQL|ChromaDB|BM25|
|**正常上传**|✅ 已写入|✅ 已写入|✅ 已写入|
|**ChromaDB 写入失败**|✅ 已写入（status=vector_failed）|❌ 未写入|✅ 已写入|
|**正常删除**|✅ 已删除|✅ 已删除|✅ 已删除|
|**ChromaDB 删除失败**|✅ 已删除|❌ 残留（待重试）|✅ 已删除|
|**重试成功**|✅ 已删除|✅ 已删除|✅ 已删除|
|**重试失败（超过10次）**|✅ 已删除|❌ 残留（标记failed）|✅ 已删除|

---

### 数据一致性保障机制

```
┌─────────────────────────────────────────────────────────────────┐
│                     数据一致性保障                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【写入时】                                                      │
│  ├─ MySQL + BM25：同一事务，原子性保证                            │
│  ├─ ChromaDB：异步写入，失败标记 vector_failed                    │
│  └─ 状态机：processing → completed / vector_failed              │
│                                                                 │
│  【删除时】                                                      │
│  ├─ MySQL：优先删除，保证主流程可用                               │
│  ├─ BM25：同步删除                                               │
│  ├─ ChromaDB：失败记录到 cleanup_task 表                         │
│  └─ 定时重试：每 5 分钟重试，最多 10 次                          │
│                                                                 │
│  【最终一致性】                                                  │
│  ├─ ChromaDB 写入失败：用户手动重试                              │
│  ├─ ChromaDB 删除失败：定时任务自动重试                           │
│  └─ 残留数据：超过 7 天的 failed 任务自动清理                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 流程图总结

```
                    上传流程                    删除流程
                        │                           │
                        ▼                           ▼
              ┌─────────────────┐         ┌─────────────────┐
              │   MySQL 写入    │         │  ChromaDB 删除  │
              │  (事务保证)     │         │  (尝试删除)     │
              └────────┬────────┘         └────────┬────────┘
                       │                           │
              ┌────────┴────────┐         ┌────────┴────────┐
              │                 │         │                 │
              ▼                 ▼         ▼                 ▼
    ┌─────────────┐   ┌─────────────┐   ┌─────────┐   ┌─────────┐
    │ ChromaDB    │   │   BM25      │   │  成功   │   │  失败   │
    │ (异步写入)  │   │  (同步写入) │   │  继续   │   │记录重试 │
    └─────────────┘   └─────────────┘   └─────────┘   └────┬────┘
                                                           │
              ┌────────────────────────────────────────────┘
              ▼
    ┌─────────────────┐         ┌─────────────────┐
    │   BM25 删除     │ ──────▶ │   MySQL 删除    │
    │   (同步删除)    │         │  (优先删除)     │
    └─────────────────┘         └─────────────────┘
```

## 十四、各环节的兜底策略汇总

|   |   |   |
|---|---|---|
|环节|可能的故障|兜底方案|
|Query 扩展|LLM API 失败|仅使用原始查询|
|向量检索（知识库）|ChromaDB 连接失败|降级为 MySQL + 余弦相似度|
|向量检索（笔记）|ChromaDB 连接失败|返回空列表|
|BM25 检索|Lucene 异常|返回空列表，不阻断流程|
|Cross-Encoder 精排|rerank API 失败|返回原始 RRF 结果|
|笔记向量写入|ChromaDB 不可用|跳过向量写入，只写 MySQL 和 BM25|
|知识库向量写入|ChromaDB 失败|标记 status="vector_failed"，支持重试|
|整个检索|没搜到任何文档|返回"未找到相关文档"|
|删除 ChromaDB|删除失败|记录到 cleanup_task 表，定时重试|
|删除 ChromaDB|重试超过 10 次|标记 status="failed"，不再重试|
|删除 ChromaDB|残留超 7 天|每天凌晨 3 点自动清理 failed 记录|

## 十五、Ragas评估流程

```
RagTrace (查询记录)
  │
  │ 包含: query, retrievedDocs, finalAnswer, groundTruth(可选)
  │
  ▼
┌──────────────────────────────────────────────────┐
│           EvaluationService.evaluate()            │
│                                                   │
│  并行执行 LLM 评估 (CompletableFuture)：           │
│                                                   │
│  ┌─────────────────┐  ┌─────────────────────┐   │
│  │ 1. Faithfulness │  │ 2. Answer Relevancy │   │
│  │    忠实度        │  │    答案相关性        │   │
│  │                 │  │                     │   │
│  │ 声明拆分+逐条   │  │ 反向生成3个问题     │   │
│  │ SUPPORTED/      │  │ 计算与原问题的      │   │
│  │ NOT_SUPPORTED   │  │ 语义相似度平均值    │   │
│  └────────┬────────┘  └──────────┬──────────┘   │
│           │                      │               │
│  ┌────────┴────────┐  ┌─────────┴───────────┐   │
│  │ 3. Context      │  │ 4. Context Recall   │   │
│  │    Precision    │  │    上下文召回率      │   │
│  │    上下文精确度  │  │    (需Ground Truth) │   │
│  │                 │  │                     │   │
│  │ 位置加权精确度  │  │ 标准答案拆句        │   │
│  │ Precision@k     │  │ 判断上下文是否支持   │   │
│  └────────┬────────┘  └─────────┬───────────┘   │
│           │                      │               │
│  ┌────────┴──────────────────────┴────────────┐ │
│  │           规则评估 (ruleEvaluate)            │ │
│  │                                             │ │
│  │  - 耗时 > 15s → -20分                       │ │
│  │  - 检索为空 → -30分                          │ │
│  │  - 回答 < 20字 → -15分                       │ │
│  │  - 平均相似度 < 0.3 → -15分                  │ │
│  └──────────────────┬────────────────────────┘ │
│                     ▼                           │
│  ┌──────────────────────────────────────────┐  │
│  │          综合评分计算                      │  │
│  │                                           │  │
│  │  LLM加权分 = faithfulness×W1 + relevancy×W2│  │
│  │             + precision×W3 + recall×W4     │  │
│  │                                           │  │
│  │  最终分 = 调和平均数(LLM加权分, 规则分)     │  │
│  │                                           │  │
│  │  评级: ≥90优秀, ≥75良好, ≥60及格, <60不及格│  │
│  └──────────────────┬───────────────────────┘  │
│                     ▼                           │
│  ┌──────────────────────────────────────────┐  │
│  │          自动诊断 (diagnose)               │  │
│  │                                           │  │
│  │  - 精度+召回都低 → Embedding/Chunk/召回策略│  │
│  │  - 精度低 → 优化Reranker或TopK             │  │
│  │  - 召回低 → 优化Query扩展或降低阈值        │  │
│  │  - 忠实度低 → 幻觉，优化Prompt或换模型      │  │
│  │  - 相关性低+忠实度高 → 检索文档不对         │  │
│  └──────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---