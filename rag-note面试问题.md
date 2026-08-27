# RAG-Note 项目面试问题总结

---

## 一、AI Agent 基础概念

#### 你理解的 AI Agent 是什么？

我理解的 AI Agent 有三个核心特征。一是**自主规划能力**，给它一个复杂目标它能自己拆解成多步来执行，而不是等着人来一步步指挥。二是**行动能力**，它能通过工具调用跟外部世界真实交互，比如查数据库、调 API、操作文件，而不只是动动嘴皮子。三是**闭环反馈**，每一步执行的结果会反馈回来指导下一步决策，而不是一次性生成完就结束。另外有一个容易混淆的点需要澄清：模型本身只是大脑，真正执行工具操作的是你的业务代码，模型只负责决策"调哪个工具、传什么参数"。

---

## 二、项目整体架构

#### 1. 介绍一下你这个项目的整体架构？

这个项目是一个基于 RAG 架构的智能笔记系统，核心就是"先检索、后生成"。
整体分成三层：存储层用 MySQL 存笔记和聊天记录，ChromaDB 存向量索引做语义搜索，Lucene 做 BM25 关键词索引，三端保持同步；检索层是整套管线的核心，用户提问后先经过 LLM 做 Query 扩展改写多个版本，然后向量检索和 BM25 并行跑，结果通过 RRF 算法融合排序，再用 Cross-Encoder 做一次精排；生成层把检索到的相关文档拼上系统提示词交给 LLM 生成最终回答。用户上传文档时经过 Tika 解析文本、中文分块、批量向量化后同步写入这三端，整个流程是异步处理的，不影响用户正常使用。

Agent 系统采用模块化架构，核心分为九个子包共 69 个 Java 文件。核心执行层包括 AgentService 主入口、AgentLoop 目标驱动循环、AgentState 状态容器、AgentTools 工具定义。运行时子系统包括 AgentRuntime 任务编排、AgentTaskService 生命周期管理、ReflectionService 反思机制、ReplanningService 重规划、ToolExecutionService 工具执行封装。可观测性层包括 AgentTrace 全链路追踪、AgentMetrics 指标统计。策略控制层通过 AgentPolicyService 和 BudgetConfig 限制轮次、时间、Token、工具调用次数。动态工具扩展通过 Skill 包系统支持 Git 导入和 Skill Center 安装。整个系统支持任务持久化、暂停/恢复、全链路追踪，实现了从"黑盒执行"到"全链路可观测"的演进。

---

## 三、技术选型

#### 2. 为什么用 ChromaDB，而不是 Milvus、Pinecone 这些？

我们选择 ChromaDB，主要还是结合项目的**规模、部署成本和开发效率**来考虑。

当时项目的数据量属于中小规模，并没有达到需要专门部署向量数据库集群的程度。ChromaDB 比较轻量，部署和使用都比较简单，Java 端通过 HTTP 就可以调用，而且我们使用的 LangChain4j 也有对应的集成，所以开发成本比较低。

Milvus 更偏向大规模向量检索场景，能力比较完整，但对于我们当前的规模来说，部署和运维成本相对更高。Pinecone 属于云端托管服务，虽然使用方便，但会涉及云服务成本以及企业数据存储的问题。

所以当时的选择不是说 ChromaDB 一定比 Milvus 好，而是**当前项目规模下，ChromaDB 已经能够满足需求，同时开发和维护成本更低**。如果后续数据量和并发规模明显增长，再考虑迁移到更适合大规模场景的方案。

**可能追问：**

- **如果数据量达到百万级，你还会继续用 ChromaDB 吗？**：我会重新评估。如果出现明显的性能、并发或者扩展性瓶颈，会考虑迁移到 Milvus、Elasticsearch/OpenSearch 等更适合生产规模的方案。
    
- **为什么不用 Pinecone？**：主要考虑企业项目的数据存储和成本问题，而且当前规模并不需要额外引入云端托管服务。
    
- **迁移向量数据库困难吗？**：如果业务层不直接依赖具体数据库实现，而是封装统一的向量检索接口，迁移主要就是重新构建 embedding 索引和替换底层实现。
    

**补充细节：**

你可以简单理解成：

```text
ChromaDB
→ 轻量
→ 部署简单
→ 适合中小规模
→ 开发成本低

Milvus
→ 更偏大规模
→ 分布式能力更强
→ 运维成本也更高

Pinecone
→ 云端托管
→ 使用方便
→ 但有成本和数据存储方面的考虑
```

面试的时候重点不要说“ChromaDB 性能比 Milvus 好”，而是说：

> **“不是技术能力的绝对比较，而是根据当前项目规模选择合适的复杂度。”**

#### 3. 为什么用 Lucene 做 BM25，而不是 Elasticsearch？

主要是因为我们这个项目的 BM25 检索规模并不大，而且检索服务和 Java 后端本身结合比较紧密，所以选择了 Lucene。

Lucene 本身就是 Elasticsearch 底层使用的核心搜索库之一，它提供了完整的倒排索引和 BM25 能力。直接使用 Lucene，可以把检索能力集成到 Spring Boot 服务里，不需要额外部署 Elasticsearch 集群，整体架构更简单。

项目中我们还按照用户维度对索引进行隔离，每个用户维护自己的索引目录，服务重启后可以重新加载，所以当前规模下 Lucene 已经能够满足需求。

如果后续数据量、并发量明显增长，或者需要分布式检索、集群管理和更复杂的搜索能力，再考虑迁移到 Elasticsearch 会更合适。

**可能追问：**

- **Lucene 和 Elasticsearch 是什么关系？**：Lucene 是底层搜索引擎库，提供倒排索引、BM25 等核心能力；Elasticsearch 在 Lucene 之上提供了分布式、REST API、集群管理等完整搜索服务能力。
    
- **为什么不直接用 Elasticsearch？**：不是 Elasticsearch 不好，而是当前项目规模不大，直接使用 Lucene 可以减少一个独立服务，降低部署和维护成本。
    
- **Lucene 适合什么场景？**：比较适合单机或者嵌入式搜索场景。如果需要多节点、分片、水平扩展和比较复杂的搜索服务，就更适合 Elasticsearch 这类独立搜索服务。
    

**补充细节：**

可以这样理解：

```text
Lucene：

Spring Boot
   ↓
Lucene
   ↓
倒排索引 + BM25
```

而 Elasticsearch 更像：

```text
Spring Boot
   ↓ HTTP
Elasticsearch 集群
   ↓
Lucene
   ↓
倒排索引 + BM25
```

所以你这里选择 Lucene，本质上是：

> **我们只需要 Lucene 的搜索能力，但暂时不需要 Elasticsearch 提供的分布式服务能力。**

这里还有一个面试注意点：**不要说“百万级一定必须 Elasticsearch”**。是否需要迁移应该看数据量、QPS、P99、内存、索引构建时间以及是否需要水平扩展，而不是只看数据条数。

#### 4. Embedding 模型用的什么？为什么选它？

我们项目使用的是**智谱 AI 的 Embedding-3 模型**，主要考虑三个方面。

第一是**中文语义效果**。我们的知识库主要是中文研发文档，所以比较关注中文技术文本的语义表示效果。
第二是**工程接入成本**。使用 API 的方式接入比较简单，不需要自己部署和维护模型服务，同时支持批量调用，也比较方便放到异步的向量化流程里。
第三是**成本和效果的平衡**。当时对比过本地模型，比如 M3E，本地部署虽然数据可控，但还需要自己承担模型部署、资源占用和调优成本。在当前项目规模下，使用云端 Embedding API 能够比较快地完成知识库向量化，所以最终选择了这个方案。

**可能追问：**

- **为什么不用本地 Embedding 模型？**：本地模型的优势是数据不需要离开自己的环境，而且长期大规模调用可能更可控，但需要承担 GPU/CPU 资源、部署和模型调优成本。当前项目更关注开发效率，所以选择 API。
    
- **怎么判断 Embedding 模型好不好？**：不能只看模型参数或者维度，主要还是放到自己的业务数据上评估，比如 Recall@K、MRR，以及最终问答效果。不同模型可以使用同一批测试数据进行对比。
    
- **Embedding 维度越高是不是越好？**：不一定。维度更高不代表一定检索效果更好，同时还会增加存储和计算成本，所以应该结合实际数据集测试效果，而不是单纯追求高维度。
    

**补充细节：**

Embedding 可以简单理解成：

```text
“Redis 是什么？”
       ↓
Embedding 模型
       ↓
[0.12, -0.31, 0.58, ...]
       ↓
存入 ChromaDB
```

用户以后问：

```text
“Redis 主要有什么作用？”
```

虽然两句话的字面表达不完全一样，但 Embedding 模型可能会把它们映射到比较接近的向量空间，于是向量检索就能把相关 Chunk 找出来。

整个知识库流程就是：

```text
文档
 ↓
解析 / 切片
 ↓
Embedding
 ↓
向量
 ↓
ChromaDB

用户问题
 ↓
Embedding
 ↓
向量相似度检索
 ↓
相关 Chunk
```

这里要注意一个非常容易被追问的问题：

> **Embedding 模型和 LLM 不是一回事。**

Embedding 模型负责的是：

> **“这段文本和另一段文本在语义上有多接近？”**

LLM 负责的是：

> **“结合这些资料，我应该怎么理解并回答用户？”**

所以你的 RAG 链路可以理解成：

```text
用户问题
   ↓
Embedding
   ↓
向量检索 ───┐
            ├→ RRF → Rerank → LLM → 答案
BM25 ───────┘
```

这样面试官如果继续从 **Embedding → ChromaDB → 混合检索 → Rerank → LLM** 往下追，你也能比较自然地接住。

---

## 四、检索流程

#### 5. 详细说一下你的检索流程？

我们的检索流程分了五步。第一步是 Query 扩展，用 LLM 把用户的问题改写成 3 个语义等价的版本，解决同一个意思不同说法的问题。第二步是多路检索，4 个查询版本分别走向量检索和 BM25 检索，一共 8 路并行跑。第三步是 RRF 融合，把 8 路结果合并去重重新排序，出现次数越多、排名越靠前的文档得分越高。第四步是 Cross-Encoder 精排，用智谱的 rerank API 对候选文档重新打分，低于 0.5 的直接丢弃。最后一步才是 LLM 生成，把精排后的结果拼接上系统提示词，调用 LLM 生成最终回答。

#### 6. 如何确保检索的准确性，通过什么方式实现？

我们用了五层机制来保障检索准确性。首先是查询层，通过 LLM 做多 Query 扩展，把一个问题改写成多个语义等价的版本，这样用户和文档用词不一致时也能覆盖到。然后是检索层，向量检索擅长语义匹配，比如"怎么记住东西"能匹配到"间隔重复"，BM25 擅长关键词精确匹配，比如"JVM GC"能精确命中，两者互补。再然后是融合层，多路结果通过 RRF 算法去重排序，一个文档在多路中都排名靠前就说明它确实相关。再往上还有 Cross-Encoder 精排做最后一轮打分过滤。最后兜底层也做了保障，ChromaDB 不可用时会跳过向量写入只写 MySQL 和 BM25，LLM 调用失败时也会返回明确的错误提示，保证系统始终可用。

#### 7. 为什么用 RRF 融合，而不是直接按相似度排序？

因为向量检索和 BM25 返回的分数根本不可比。向量检索给的是余弦相似度，范围在 0 到 1 之间，而 BM25 返回的是关键词匹配分数，可能从 0 到 100 都有，直接把这两种分数放一起比较没有意义。RRF 的好处是只看排名不看分数，公式是 `score = Σ 1/(60 + rank)`，一个文档在多路检索结果里都排在前面的，RRF 分数就高，说明它确实相关。这样就能把不同来源的检索结果公平地融合在一起了。

#### 8. Cross-Encoder 精排是怎么做的？和向量检索有什么区别？

向量检索用的是 Bi-Encoder 模式，查询和文档各自编码成向量再算相似度，速度快但精度一般。Cross-Encoder 不一样，它是把查询和文档一起输入模型，让它们的 token 互相做 attention，能捕捉到更细粒度的语义关系。举个例子，"怎么记住东西"和"间隔重复是一种记忆方法"这两句话，向量检索可能匹配不到，但 Cross-Encoder 能看到"记住"和"记忆"之间的对应关系。我们用的就是智谱的 rerank API，精排完后根据分数过滤，低于 0.5 的直接丢弃。

#### 9. TopK 该取多少，依据是什么？

知识库的 TopK 设的是 5，笔记的 TopK 设的是 3。这个数字主要基于三个考虑：首先是 LLM 的上下文窗口限制，每篇文档大概 200 字，加上系统提示词不能超限；其次是信息密度和噪声的平衡，TopK 太小像 1 到 2 可能会漏掉关键信息，但太大像 10 以上又会引入太多噪声，LLM 容易被无关内容干扰；最后我们在实际检索时每路会多查一些，就是 TopK 乘以 2，给 RRF 融合留点余量，避免某一路排第六但在其他路排很靠前的好文档被漏掉。

#### 10. 精排后的动态 top-N 策略是什么？为什么这样设计？

精排后的结果我们会按分数分成三档来处理：分数大于 0.7 的算高质量结果全部保留，分数在 0.5 到 0.7 之间的中等质量最多保留 2 条，低于 0.5 的直接丢弃。这样设计是为了解决固定 top-N 的问题，比如说只有 1 条高质量结果的时候，强制取 3 条就会把低质量结果也带进来，反而影响最终回答的质量。

#### 11. Chunk 多大，为什么要这么设置？

我们的 chunk 大小是 200 个字符，重叠是 20 个字符。200 这个值太小的话，50 到 100 个字会导致语义碎片化，一句话被切断了检索到也没意义；太大的话，500 字以上一个 chunk 里可能混了好几个主题，检索到了噪声也多。200 字大概就是两到三个完整句子，刚好能表达一个完整的语义单元。重叠 20 个字是为了防止 chunk 边界切断语义关系，比如"它通过数据训练"里的"它"可能指代前一个 chunk 的内容，有重叠就能保留这个上下文衔接。切片的时候也不是按固定字数硬切，而是按中文标点层级来切，从段落、句子到逗号逐级降级，避免在句子中间切断。

#### 12. 批量 Embedding 是怎么做的？效果如何？

之前是逐条调用 Embedding API，400 个 chunk 就要发起 400 次 HTTP 请求，光网络开销就很大。优化后改成了批量调用，每批 10 条一起发，智谱的 API 本身是支持批量处理的。效果很明显，API 请求次数从 400 次降到了 42 次，减少了 90%，大文档的处理时间从 5 分钟缩短到了 1 分钟左右。

#### 13. 检索性能怎么样？有没有做过优化？

单次检索大概在 300 到 500 毫秒，其中 Query 扩展大概 100 毫秒，8 路并行检索大概 100 毫秒，RRF 融合只要 10 毫秒，Cross-Encoder 精排大概 200 毫秒。主要的优化手段有：8 路检索是并行执行的不是串行的，ChromaDB 的 collection UUID 会缓存到 Redis 减少重复调用，BM25 的索引会持久化到磁盘重启不需要重建。

#### 14. 大文档处理很慢怎么办？

最开始逐条向量化再加一次性写入，400 个 chunk 要 5 分钟，用户体验很差。优化做了三件事：一是改成批量 Embedding，每批 10 条减少 API 调用次数；二是分批写入 ChromaDB，每批也是 10 条避免超时；三是把 ChromaDB 写入改成异步执行，用 @Async 注解不阻塞主流程。优化后 5 分钟缩短到了 1 分钟左右。

---

## 五、检索质量与评估

#### 15. 检索命中率怎么样，怎么评估？

我们会准备 50 到 100 个测试问题，人工标注每个问题的标准答案文档，然后跑检索看能不能命中。核心指标有三个：Recall@K 看的是 Top K 结果里包含标准答案的比例，MRR 看的是标准答案排在第几位越靠前越好，Hit Rate 看的是至少命中一个标准文档的比例。从对比实验来看，纯向量检索 Recall@5 是 72%，MRR 是 0.65；加上 BM25 双路之后到了 81% 和 0.72；再加多 Query 和 RRF 融合提升到 85% 和 0.76；最后加上 Cross-Encoder 精排能达到 92% 和 0.85。每加一层优化都有提升。

#### 16. 你是怎么处理幻觉的问题的？

从检索和生成两个层面来处理。检索层主要是提供足够的相关文档作为上下文，让 LLM 有据可依，同时在参考资料上标注来源，像 `[来源：笔记《xxx》]` 这样，让 LLM 能引用具体来源，还有 Cross-Encoder 精排过滤低相关性文档减少噪声干扰。生成层就是在系统提示词里明确约束，要求 LLM 只基于参考资料回答不要编造信息。兜底方面，如果检索不到相关文档就直接告诉用户没找到，不让 LLM 凭空编造；用户也可以追问或查看引用来源来验证回答的真实性。

#### 17. 回答能不能引用来源，思路是什么？

已经实现了来源引用。具体思路分三步：第一步是在检索的时候给每段内容打上来源标签，比如 `[来源：笔记《学习方法总结》]`。第二步是在系统提示词里要求 LLM 在回答时引用来源。第三步是拼接参考资料的时候把来源标注带上。最终的效果就是用户能看到"根据你的笔记《学习方法总结》，间隔重复是提升记忆的核心方法"这样的回答，有据可查。

---

## 六、数据处理与一致性

#### 18. 如果用户修改了笔记内容，向量会更新吗？

会的。笔记更新时会先删除旧的向量，按 note_id 删除 ChromaDB 和 BM25 里的对应数据，然后再写入新的向量和索引。但知识库文档不支持原地更新，因为知识库是按文件名加 MD5 来去重的，如果内容变了需要重新上传，会被当作新文件处理。

#### 19. 如果 AI 找不到答案，怎么降级处理？

我们做了多层降级。第一层是检索降级，如果文档为空就直接返回"未找到相关文档"，不让 LLM 编造。第二层是 Query 扩展降级，如果 LLM 改写失败就用原始查询继续检索。第三层是精排降级，如果 rerank API 调用失败就返回原始 RRF 融合的结果。第四层是 LLM 生成降级，如果最终调用 LLM 出错了就返回错误提示。每一层都有兜底，任何一个环节失败都不会导致整个系统不可用，用户至少能拿到一个结果而不是报错。

#### 20. 三端数据是怎么保证一致的？

MySQL 和 BM25 的写入在同一个事务里，有原子性保证。ChromaDB 是异步写入的，如果失败了文档状态会标记为 vector_failed，用户可以手动重试。删除的时候优先删 MySQL，ChromaDB 删除失败不会阻塞主流程，失败任务会记录到 cleanup_task 表，有定时任务每 5 分钟自动重试，最多重试 10 次。整体上通过状态机和定时任务来保证最终一致性，即使 ChromaDB 临时故障，数据最终也会同步过去。

#### 21. 如果 ChromaDB 挂了怎么办？

写入的时候 ChromaDB 如果挂了，MySQL 和 BM25 已经写入成功了，文档状态会被标记为 vector_failed，用户可以在界面上手动触发重试。删除的时候如果 ChromaDB 删除失败，MySQL 已经删掉了，失败任务会记到 cleanup_task 表由定时任务自动重试。检索的时候如果 ChromaDB 不可用，向量检索会返回空列表，但 BM25 还能正常用，不过目前代码里还没有做 BM25 降级检索的逻辑，这是一个可以优化的点。

#### 22. MD5 去重是怎么做的？

上传文件的时候先计算文件的 MD5，然后检查两个条件：一是 md5Store 里有没有这个 MD5 记录，二是 MySQL 里有没有对应的向量数据。两个条件都满足才跳过处理。这样设计是防止一种边界情况——应用重启后 ChromaDB 数据丢了但 MD5 记录还在，导致文件没法重新上传。MD5 的特性是内容有任何微小的变化，哪怕多一个字，MD5 值就完全不同，所以不会误判。

---

## 七、AI Agent 深度篇

#### 23. Agent 的核心循环流程是什么样的？

我们的 Agent 循环是目标驱动的，核心流程是 Goal → Action → Observation → State Update → Goal Check → Done / Continue。每次用户请求进来会创建持久化的 AgentTask 记录任务状态，初始化 AgentTrace 开启全链路追踪。然后 AgentRuntime 判断是否启用 Supervisor 规划，如果需要就拆分子任务，否则走单 Agent 路径。AgentLoop 开始执行最多 10 轮（可通过 BudgetConfig 调整），每轮先检查任务是否被暂停、预算是否超限，连续无进展时会触发 ReflectionService 让 LLM 分析卡住原因，然后 ReplanningService 生成新策略。LLM 自主决定调哪个工具，ToolExecutionService 统一执行并记录 AgentTaskStep，ToolResultEvaluator 评估结果质量给出恢复建议。工具成功后 GoalEvaluator 立即判断目标是否达成，达成就直接结束。最后 ResponseComposer 基于证据包生成回答，QualityReviewer 审查质量，不通过就带反馈重新生成。整个过程通过 AgentTrace 记录规划决策、工具调用、反思、质量评审全流程，任务完成后更新 AgentTask 状态。

#### 24. GoalEvaluator 是怎么判断目标达成的？

GoalEvaluator 是我们 Agent 系统的核心组件，它的判断逻辑分几种情况。对于写操作工具比如创建笔记、编辑笔记、删除笔记这些，工具执行成功就直接判定目标达成。对于生成产物的工具比如思维导图、图表，产物生成就算完成，但如果用户还要求把产物写回原笔记，就会标记为未完成继续执行。对于读操作比如 getNote，如果当前目标就是读取笔记，那读取到完整内容就算完成。最难的是 LLM 想直接返回文本不调工具的情况，这时候会用另一个 LLM 来做目标检查，把原始目标、用户问题和 LLM 的回答一起发过去，让 LLM 判断"这个回答完成了目标没有"，返回 YES 就放行，返回 NO 就拦截并提示 LLM 继续操作。这种用 LLM 自检的方式比硬编码关键词匹配要灵活得多。

#### 25. Supervisor 是怎么决定拆不拆任务的？

Supervisor 本质上是用 LLM 做一次任务分诊。它会分析用户的查询，判断是简单任务还是复杂任务，如果是复杂任务就拆成 2 到 4 个子任务，每个子任务有独立的描述、建议工具和目标。返回空列表就表示是简单任务走单 Agent，返回多个子任务就走多 Agent 流水线。不过有些情况我们会强制降级：比如 Supervisor 只返回了一个子任务，而且不是明显的"生成产物并写回"这类复合操作，我们会降级为单 Agent 执行，因为 Supervisor 本身有 LLM 调用延迟，简单任务没必要多此一举。

#### 26. Agent 怎么防止自己陷入死循环？

我们做了好几层防护。第一层是硬限制，最大轮次 10 轮，到了直接结束。第二层是防重复机制，如果 LLM 用相同的工具和参数已经失败过一次，系统会拦截这次调用并告诉 LLM"这个之前就失败了，换条路走"；如果 LLM 连续两轮调用的工具和参数完全相同也会被拦截，注入环境状态让它换策略。第三层是进度追踪，连续两轮没有进展就注入环境状态视图，把当前的目标达成进度、已经做过哪些尝试、哪些路不通都列出来，让 LLM 基于当前状态做出不同选择。最后一层是 GoalEvaluator 在 LLM 想直接回答时会做检查，如果之前工具调用结果都很差或者目标根本没达到，会拦截回答让 LLM 继续干活。

#### 27. 工具调用的结果质量是怎么评估的？

ToolResultEvaluator 负责这块，它优先使用我们定义的结构化结果来判断。如果工具返回了 ToolResult 对象，就根据里面的 status 字段来判断：SUCCESS 就是 GOOD，EMPTY 就是 POOR 表示搜不到结果，ERROR 就是真正的错误。如果没有结构化结果就降级到字符串匹配，看返回值里有没有"未找到"、"失败"、"异常"这些关键词。更重要的是，不同的错误类型会给出不同的恢复建议，比如 NOTE_NOT_FOUND 会告诉 LLM"你传的 noteId 无效，要先调用 searchNotes 或 listNotes 找到正确的 ID"，这样 LLM 知道下一步该怎么做而不是盲目重试。

#### 28. 会话上下文是怎么管理的，怎么解决"这篇笔记"这种指代问题？

我们通过 ConversationContextManager 来管理每个会话的上下文，存在内存的 ConcurrentHashMap 里。每次 Agent 调用 getNote、createNote、appendNote 这些工具时，会自动更新当前会话的活跃笔记 ID 和标题。当用户说"这篇笔记"、"它"、"刚才那篇"的时候，ConversationContextManager 会检测到这些指代词，然后把当前活跃笔记的信息注入到查询里，告诉 LLM "用户刚才操作的笔记是《XXX》，noteId=xxx"。这样 LLM 就能正确理解用户的指代，不用每次都重新搜索。

#### 29. Agent 调用的工具有哪些？

基础工具有 19 个，分四类。笔记检索类有 listNotes、searchNotes、getNote、getRecentNotes、getRelatedNotes，覆盖了浏览、搜索、读取、关联推荐等场景。笔记操作类有 createNote、editNote、appendNote、deleteNote、mergeNotes，基本涵盖了笔记的增删改查合并。知识库类主要是 ragSummary，做 RAG 检索和摘要。复习类有 getTodayReviews、markReviewed、scheduleReview、getNoteStats，跟艾宾浩斯复习系统对接。通用工具包括 whatTimeIsNow、fetchUrl、generateDiagram、generateMindMap。每个工具都有详细的描述和参数说明，LLM 根据这些描述来决定什么时候调哪个工具。

另外我们还实现了动态工具扩展机制，通过 Skill 包系统让用户可以自定义工具。SkillContextResolver 在每次请求时扫描用户已安装的 Skill 包，解析 skill.json 元数据，动态注册为可调用工具。Skill 可以通过 Git 仓库导入或从 Skill Center 一键安装，以独立进程（Python/Shell）执行，失败不影响主服务。这样用户就能根据自己的需求调用企业内部 API、处理特殊文件格式，不需要修改代码重新部署。

#### 30. 多 Agent 流水线什么时候用顺序执行，什么时候用并行执行？

这取决于子任务之间的依赖关系。如果子任务之间有明确的先后顺序，比如"先搜索找到笔记，再读取完整内容，最后生成思维导图"，就用顺序执行，每个子任务的输出会作为下一个子任务的输入上下文。如果子任务之间没有依赖关系，比如"同时查询笔记统计信息和今日复习列表"，就用并行执行，通过 CompletableFuture 并发跑。并行执行完会用 WriterService 把多个子任务的结果合成一个连贯的回答。目前顺序和并行的判断依据是子任务里的 executionMode 字段，由 Supervisor 在拆任务时决定。

---

## 八、Redis 与缓存

#### 31. Redis 在项目里用在了哪些地方？

Redis 主要用在三个地方。首先是 JWT 的黑名单，用户登出的时候把 token 加入 Redis 黑名单，设置跟 token 剩余有效期一致的 TTL，这样登出的 token 就失效了。其次是用户缓存，用户信息会缓存在 Redis 里减少数据库查询。还有一个是缓存防护层，我们封装了一个统一的 CacheProtectionService，所有需要缓存的查询都通过它来读取，同时解决了缓存穿透、击穿和雪崩的问题。

#### 32. 缓存穿透、击穿、雪崩你们是怎么处理的？

我们写了一个统一的 CacheProtectionService，用一个模式同时解决了这三个问题。**缓存穿透**是指查询一个不存在的数据，每次都穿透到数据库，我们的做法是把 null 值也缓存起来，但是给一个很短的 TTL，60 秒，这样相同的查询短时间内不会反复打到数据库。**缓存击穿**是指热点 key 过期瞬间大量并发请求涌入，我们用 Redis 的 SETNX 命令实现互斥锁，同一时刻只有一个线程去重建缓存，其他线程自旋等待，最多等 2 秒。**缓存雪崩**是指大量 key 在同一时间过期，数据库压力骤增，我们的做法是在设置 TTL 的时候加一个正负 25% 的随机抖动，让过期时间自然分散开。这三个问题用一个组件就统一解决了。

---

## 九、安全与鉴权

#### 33. 你们是怎么做认证和授权的？

认证用的是 JWT，用户登录后服务端签发一个 token，客户端在每次请求时通过 Authorization 头带回来。不过我们没有用 Spring Security 的自动解析，而是写了一个自定义的 UserId 注解和 ArgumentResolver，在 Controller 参数上直接注入当前用户 ID。JWT 的密钥是 256 位的 HMAC-SHA256 算法，有效期 24 小时。授权方面比较简单，因为这是个人笔记系统，主要是校验用户身份，没有复杂的角色权限体系。登出的时候 token 会被加入 Redis 黑名单，在 JWT 过滤器里会检查黑名单。

#### 34. JWT 过期了怎么处理？

我们目前没有做 refresh token 机制，token 过期了用户就需要重新登录。因为个人笔记系统对用户体验的要求没那么高，重新登录的成本可以接受。不过登出是在 Redis 里维护了一个黑名单，登出的时候把 token 加入黑名单，TTL 设置成跟 token 剩余有效期一致，这样黑名单不会一直膨胀。JWT 过滤器会先检查黑名单，如果在黑名单里就直接拒绝。

---

## 十、架构与配置

#### 35. 你们支持哪些 LLM 供应商，怎么切换？

我们目前支持四个 LLM 供应商：DeepSeek、智谱 AI、阿里云通义千问和本地 Ollama。切换只需要改配置文件里的 `app.llm.type` 字段就行，ModelFactory 会根据 type 创建对应的 LLM 客户端。Embedding 模型也是独立配置的，目前用的智谱的 embedding-3，Reranker 也是智谱的。这个多供应商设计的好处是灵活，用户可以根据自己的需求和预算自由切换，甚至不同的组件可以用不同的供应商。

#### 36. SSE 流式输出是怎么实现的？

我们用的是 Spring 的 SseEmitter，设置 5 分钟超时。整个 Agent 执行是异步的，通过 CompletableFuture.runAsync 配合自定义线程池来跑，避免阻塞 Tomcat 的请求处理线程。前端通过 EventSource 接收事件流，我们会发送多种类型的事件：thinking 事件用于展示 Agent 的思考过程，包括当前是第几轮、正在调什么工具、进度信息等；response 事件是最终的文本回答，按 50 个字一个 chunk 分块推送，前端实现打字机效果；done 事件包含会话 ID、trace ID、token 用量和产物数据。如果客户端断开了连接，SseEmitter 的 send 会抛出 IOException，我们捕获后忽略，不会影响服务端其他请求。

#### 37. 你们的前端架构是什么样的？

前端用的 Vue 3 加 Vite 7，UI 组件库是 Vant 4，主要面向移动端。状态管理用的 Pinia，持久化到 localStorage，分成了 user 和 app 两个 store，分别管理用户状态和应用配置。路由做了全局前置守卫，未登录就重定向到登录页。SSE 流式接收配合打字机效果展示 AI 回答，思考过程也能实时展示。Markdown 编辑用的是 ByteMD，渲染用 marked 加 highlight.js 做代码高亮。国际化用的 vue-i18n，支持中英文切换。整体是个比较标准的 Vue 3 组合式 API 写法。

#### 38. 笔记和知识库有什么区别？

笔记是用户自己手动创建和编辑的内容，有标题、正文、标签、分类这些结构化字段，支持 CRUD 操作，还有艾宾浩斯间隔复习功能。知识库是用户上传的文档，比如 PDF、Word、TXT 等，系统自动解析和分块，用户不能直接编辑，只能删除重传。检索的时候两个来源都会查，但笔记是用户主动整理的知识，权重会更高一些。

#### 39. 间隔复习是怎么实现的？

基于艾宾浩斯遗忘曲线来调度。每次创建笔记或者复习时，系统会根据当前的复习次数计算下一次复习的时间间隔：第一次是 1 天后，第二次 3 天后，第三次 7 天后，第四次 15 天后，第五次 30 天后，后面每次递增。每天系统会查询当天需要复习的笔记，展示给用户。复习的时候会弹出一个 LLM 根据笔记内容自动生成的选择题，用户答对了才算完成复习，答错了会提示正确答案。同时还支持安排复习计划，可以指定笔记和间隔天数。

---

## 十一、性能与优化

#### 40. 你在项目中做过哪些性能优化？

主要在四个方面做了优化。检索方面，8 路检索是并行执行的，不是串行，ChromaDB 的 collection UUID 缓存到 Redis 减少了重复调用。文档处理方面，批量 Embedding 把 400 次请求降到 42 次，分批写入避免超时，异步处理不阻塞主流程。缓存方面，用 Redis 缓存用户信息，用 CacheProtectionService 统一解决了缓存穿透击穿雪崩问题。前端方面，SSE 流式分块输出配合打字机效果，用户不需要等全部生成完就能看到内容。

#### 41. 异步任务你们是怎么管理的？

异步任务主要用 Spring 的 @Async 注解加自定义线程池来处理。线程池的核心线程数 5 个，最大 10 个，队列容量 25 个，拒绝策略是 CallerRunsPolicy，就是如果线程池满了就由调用者线程自己执行。Agent 的 SSE 流式响应也是在 CompletableFuture.runAsync 里跑的，传入 SecurityContext 保证异步线程能拿到用户的认证信息，在 finally 块里清除避免线程复用导致数据泄露。

ChromaDB 的清理任务通过 @Scheduled 定时执行，每 5 分钟跑一次，重试失败的向量删除操作。文档的异步处理状态通过状态机来管理，从 processing 到 completed 或 vector_failed，用户可以手动重试失败的。

Agent 执行引入了更完善的异步管理机制。每个对话请求创建持久化的 AgentTask 记录任务状态，通过 AgentRuntime 编排异步执行流程。任务状态从 CREATED → PLANNING → RUNNING → COMPLETED/FAILED/PAUSED 流转，AgentTaskStep 记录每一步操作详情。支持任务暂停（保存 AgentExecutionSnapshot）和恢复（AgentTaskResumeService 重建上下文），长时间任务不会因为超时或资源限制前功尽弃。AgentTrace 提供全链路追踪，记录规划决策、工具调用、反思、质量评审全过程，方便调试和性能分析。

---

## 十二、项目经验总结

#### 42. 做这个项目你遇到的最大挑战是什么？

最大的挑战是 Agent 的行为可控性和系统可靠性。最开始 LLM 经常陷入死循环，同一个工具反复调，或者目标没达到就擅自回答。后来我们加了好几层防护才解决：一是防重复机制，同参数失败的和与上一轮完全相同的调用都会被拦截；二是进度追踪配合反思与重规划，连续无进展时 ReflectionService 分析卡住原因、ReplanningService 生成新策略；三是 GoalEvaluator 的 LLM 自检，在 LLM 想回答时判断目标到底达成了没有；四是预算控制，通过 AgentPolicyService 限制轮次、时间、Token、工具调用次数，防止资源耗尽。

第二个挑战是任务持久化和容错。早期 Agent 执行完全在内存中，一旦异常整个上下文丢失。引入 AgentTask 实体后实现了任务状态追踪，AgentTrace 提供全链路可观测，AgentTaskStep 记录每一步操作，支持暂停/恢复机制。这样即使遇到异常，用户也能查看执行历史、了解失败原因，长任务可以中断后恢复。

第三个挑战是动态工具扩展的安全性。Skill 包系统允许用户自定义工具很灵活，但执行用户代码有安全风险。我们通过独立进程隔离（Python/Shell）、资源限制、Skill Center 审核机制来缓解，失败不影响主服务。

#### 43. 如果重新做这个项目，你会怎么做？

如果重新做，有几点我会改进。一是向量数据库，当初用内存模拟导致重启后数据丢失，后面才接 ChromaDB，一开始就应该直接上正式的向量库。二是 Agent 的架构设计，现在的任务持久化、反思与重规划、预算控制这些机制都是后来逐步加上的，如果一开始就考虑到可观测性和容错性，代码结构会更清晰。三是 Prompt 工程，目前是把所有工具描述和约束都写在一个大的系统提示词里，其实可以分层的，通用的放系统提示词，具体的放工具描述里。四是测试体系，RAG 系统的评测比较滞后，现在才加上回归测试和消融实验，应该一开始就建立评估基线、持续监控检索质量和 Agent 表现。五是动态工具的安全机制，Skill 包系统的沙箱隔离、权限控制可以做得更严格，比如限制网络访问、文件系统访问范围。六是代码质量，有些地方异常处理不够精细，比如 BM25 的降级策略、ChromaDB 不可用时的兜底检索，这些都可以做得更健壮。

---

## 十三、Agent 规划器与流水线设计（深挖）

#### 44. 为什么要有 Supervisor 规划器这一环节？简单的 AgentLoop 不够吗？

因为 AgentLoop 内部的 GoalEvaluator 判断"目标完不完成"，用的是针对**单一目标**设计的规则，比如写操作成功就算完成、产物生成就算完成。这个前提是一次循环只服务一个目标。如果用户的请求里其实包含两个不相关的诉求，比如"总结一下这周笔记，同时查一下 JVM 资料"，不拆分直接丢给一个 AgentLoop，第一个诉求的写操作一旦成功，GoalEvaluator 很可能直接判定"目标达成、结束循环"，第二个诉求根本没执行就被误判完成了。所以真正独立的多个目标必须先拆开，每个子任务各自跑一次 AgentLoop.run()，维护独立的 AgentState 和 goal，这样目标判定才不会串。

第二个原因是执行预算。AgentLoop 有硬性的最大轮次限制（10轮），多个独立诉求硬塞进一个循环会互相挤占这个预算，工具调用历史也会搅在一起，模型容易顾一头忘一头。拆成子任务后每个子任务是独立的一次调用，预算不共享。

第三个原因是并行能力。单个 AgentLoop 结构上是顺序推进的，一轮只能往前走一步。如果多个子任务互相没有依赖，比如同时查知识库和查笔记统计，规划器可以判断出 PARALLEL 模式，用 CompletableFuture 把子任务分到不同线程并发跑，这是单个 AgentLoop 做不到的——它一轮内可以让模型一次调用多个工具，但那是同一个上下文里的决策，不是两条独立推理链路的真正并发。

不过要澄清一点：规划器这次 LLM 调用，对所有请求都是无条件发生的，不存在"简单请求跳过规划器"。真正被省掉的是规划器**之后**的开销——如果规划器返回的子任务只有1个、且不是"生成产物再写回笔记"这种复合场景，会强制把子任务清空，直接走单 Agent 路径，跳过每个子任务单独 compose、最后 WriterService 合成、合成结果再审查这一整套流水线。所以这本质上是"先分诊、按需升级复杂度"的思路，不会让占大多数的简单请求为复杂机制买单，但分诊本身这次 LLM 调用的成本是省不掉的，好在用的是低成本的 precise 模型档位（temperature=0.1），边际成本可控。

#### 45.为什么单任务场景不直接走 pipeline？

技术上可以这么做，效果不会错，但会多付出一次没必要的开销，还有一定的质量风险，所以没有这样设计。

关键差异在最后一步。单 Agent 路径是 `agentLoop.run()` → `composeAndReview()`，compose 只调一次 LLM，review 直接拿 EvidencePack 里完整的证据（笔记原文、写操作确认、知识库摘要）去审查，没通过就重新 compose 一次，链路很直接。如果把单任务塞进 pipeline，流程会变成 `agentLoop.run()` → `responseComposer.compose()` 生成子任务片段 → `synthesizeResults()` 里再调 `writerService.synthesize()` 把"子任务结果"重新合成一遍，合成完才做审查。问题就出在这个 synthesize 环节——它是为了把**多个**子任务的结果风格统一、逻辑衔接起来设计的，只有一个子任务的时候，它其实在把一个已经完整的答案再拿去给 LLM"复述"一遍，这纯粹是浪费，多了一次调用、加了延迟和成本，却没有任何"缝合多个来源"的实际需求。而且这次多余的复述还有质量风险，容易让原本精确引用笔记细节的回答，在复述后变得笼统、丢细节，语气也变得像在总结多个来源，跟单任务场景应有的直接聚焦风格不符。另外 pipeline 里做审查时看到的证据也变薄了——只是子任务输出截断到 500 字符的预览，不是完整的 EvidencePack，审查的准确度会打折扣。

所以核心原因是：pipeline 里的"合成"环节专门解决"调解多个独立结果"的问题，子任务数为 1 时这个问题根本不存在，这个环节就变成了一次多余且有风险的操作。保留一条更短直接的 compose+review 链路，就是为了不让占大多数的单任务请求承担只有多任务场景才需要的合成成本。这也是为什么"子任务数=1"会被提前拦下来，不让它流入 pipeline 逻辑。

#### 46. 温度（Temperature）参数是什么？为什么要设置成 0.1？

温度是控制大模型生成文本时"随机性/确定性"的采样参数，取值一般在 0 到 1 之间。原理上，LLM 每生成一个 token，会先算出词表里每个候选词的概率分布，再从分布里采样下一个词，温度就是在采样前对这个概率分布做软化或锐化：
**温度越低，分布越尖锐，模型几乎总选概率最高的词，输出稳定、可复现；温度越高，分布越平缓，低概率词也有机会被选中，输出更有多样性但也更容易跑偏**。

项目里按职责分了三档温度：
`TEMP_PRECISE=0.1` 用在 AgentLoop 工具决策、Supervisor 任务规划、GoalEvaluator 目标判定；
`TEMP_BALANCED=0.5` 用在 ResponseComposer 生成回答、历史摘要；
`TEMP_CREATIVE=0.8` 用在 QueryExpander 查询扩展。

用 0.1 的场景要的是**稳定可复现的结构化决策**，不是文采。具体原因三点：一是 Supervisor 要求严格按 JSON 格式返回子任务，温度高的话 LLM 更容易在 JSON 前后夹带自然语言说明或者格式写错，代码里也确实为此写了专门的清洗兜底逻辑，温度调低能减少这种情况；二是 AgentLoop 每轮的工具选择是逻辑判断题，不是开放创作，同样的上下文应该做出同样的判断，如果温度高、决策随机性大，前面提到的防重复防死循环机制的效果会被削弱，因为这些机制建立在"模型决策相对稳定"的前提上；三是 GoalEvaluator 判断目标是否达成本质是 YES/NO 判定题，需要准确对齐语义而不是发散。反过来 QueryExpander 用 0.8 高温度，是因为它的任务恰恰需要把一个查询改写出几个不同表述版本，需要多样性，温度太低几个改写版本会长得几乎一样，失去多角度覆盖检索的意义。

一句话总结：温度是"确定性"和"多样性"之间的取舍旋钮，项目里的做法是按调用职责分别配置合适的档位——需要精确决策的地方用低温换稳定，需要发散表达的地方用高温换多样性。

#### 47. 多 Agent 流水线中每个子任务有独立的上下文和记忆吗？顺序执行时结果是怎么传递给下一个任务的？R1、R2、R3 之间是不是只能看到上一个的结果？

是独立的，但独立程度要分清楚——有的字段共享，有的不共享。

**独立体现在哪**：顺序流水线里每个子任务单独调一次 `agentLoop.run()`，这个方法内部第一件事就是 `new AgentState(userQuery)`，创建全新的状态对象。这意味着每个子任务的工作记忆（workingMemory）、工具调用历史（toolHistory）、观察记录（observations）、目标（goal），都是从零开始的，子任务 A 执行过程中调了什么工具、失败过什么、积累了哪些"已知事实"，子任务 B 完全看不到、也不受影响。这个隔离是刻意设计的——如果状态不隔离，子任务 B 可能在还没开始跑的时候，就被子任务 A 遗留下来的"目标已达成"标记误判成结束了。但有一处是共享的：对话历史 `historyMessages` 是在流水线外层加载一次，原样传给每一个子任务的 `agentLoop.run()` 调用，所以每个子任务都能看到同一份"用户之前跟AI聊过什么"的历史。

**结果怎么传递**：不是把上一个子任务的 AgentState 对象直接传给下一个，而是把它"翻译"成一段文本，塞进下一个子任务的用户提问里。每跑完一个子任务，会用 `ResponseComposer.compose()` 把这个子任务的 AgentState 转成一段自然语言回答文本，连同这个子任务积累的"已知事实"、如果生成了产物（比如思维导图内容，截断到8000字符）打包成一个条目，追加到一个叫 `completedContext` 的字符串里。这个字符串在 `runSequentialPipeline` 外层声明，贯穿整个顺序流水线的执行过程，是**持续累积、从不清空**的——执行完 R1 把内容 append 进去，执行 R2 时提示词里已经带着含 R1 内容的 `completedContext`，执行完 R2 再把内容追加进去，到 R3 执行时，`completedContext` 里已经是 R1+R2 的内容拼在一起了。所以 **R3 能同时看到 R1 和 R2 的结果，不是只有上一步（R2）的**——`completedContext` 这个命名和提示词里"[已完成的上一步结果]"的措辞容易让人误以为只装了最近一条，但实际存的是从第一个子任务开始的全部累积历史。下一个 AgentLoop 第一轮看到的提示词里就带着"前面做了什么、结果是什么"这段文字，是靠读这段文本知道前情，不是靠共享内存对象感知前一个 Agent 的状态，这个设计类似多 Agent 协作里常见的"黑板模式"：每个 Agent 做完事往共享文本记录里写一份总结，下一个 Agent 读总结继续干。

还有一点要注意：流水线外层维护了一个 `mergedState`，每跑完一个子任务会合并进它的产物、证据级别、工作记忆，但这个 `mergedState` 只是给最后返回结果用的（比如前端展示的产物列表），不会喂回下一个子任务的 `agentLoop.run()` 调用，跟"结果怎么传给下一步"没关系。

**潜在的设计影响**：如果顺序流水线的子任务数量较多（比如5、6个），`completedContext` 会越滚越大，虽然每个子任务的产物内容部分做了截断（最多8000字符），但整体仍是线性增长的，子任务越靠后拿到的上下文就越长，一方面保证信息不丢失，另一方面会推高后面子任务的 prompt token 消耗，理论上链条特别长时有可能顶到 `ContextManager` 的 32K token 预算。目前项目里 Supervisor 一般只拆 2 到 4 个子任务，这个问题在实际场景里还不明显，但如果以后支持更长链条的任务拆解，是需要留意的潜在瓶颈。

一句话总结：每个子任务的内部执行状态完全隔离、独立创建；子任务之间的信息传递靠把每一步的回答文本+关键事实+产物内容持续累积成一段说明文字，注入到下一步的用户提问里，是文本层面的交接、且是全量累积不是只传上一步，不是对象层面的共享。

#### 48. 单 Agent 和多 Agent 都有"组织回答 + 质量审查"这一步吧？两者具体有什么区别？

两条路径确实都有这一步，但composed 的对象、审查看到的证据、以及审查不通过后的重试方式，三处都不一样。

**第一个区别：compose 调用的次数和对象不同。** 单 Agent 路径是 `composeAndReview()`，整个请求从头到尾只调一次 `ResponseComposer.compose()`，直接基于这一次 AgentLoop 循环积累的完整 `EvidencePack`（笔记原文、写操作确认、知识库摘要、产物）生成回答。多 Agent 路径是先给**每个子任务**都单独调一次 `compose()`，生成的是这个子任务自己的回答片段，然后这些片段还要再经过 `WriterService.synthesize()` 额外一次 LLM 调用合并成最终答案——相当于多 Agent 路径的"组织回答"其实是两层：先各自 compose 出片段，再对片段做一次二次合成，不是直接从原始证据一步生成最终回答。

**第二个区别：质量审查看到的证据丰富程度不同。** 单 Agent 的审查（`composeAndReview` 里）拿到的是完整证据——笔记内容单条最多截到700字符、写操作确认原文、知识库摘要原文，是比较扎实的证据链。多 Agent 最终审查（在 `synthesizeResults` 里）拿到的证据是每个子任务输出文本截断到**500字符的预览**，不是原始的 EvidencePack，证据变薄了很多，审查的时候更多是在看"合成的答案跟子任务片段是否大致对得上"，而不是"答案跟检索到的原始证据是否吻合"，准确度会打一些折扣。

**第三个区别，也是比较隐蔽的一点：审查不通过之后的重试方式不一样。** 单 Agent 的重试是"盲重试"——审查不通过后代码里直接又调了一次 `responseComposer.compose(pack, loopResult.outcome())`，参数跟第一次完全一样，并**没有把审查给出的 `reason`/`feedback` 传回去**，能不能改好完全靠 0.5 温度的随机性去撞运气。多 Agent 的重试反而是"带反馈重试"——`synthesizeResults` 里审查不通过后，会把 `review.feedback()` 拼到 query 里再传给 `writerService.synthesize()`，让第二次合成时明确知道"哪里不对、要往哪个方向改"。这个地方其实是反直觉的：单 Agent 路径整体上证据更扎实、逻辑更直接，但它的审查重试机制反而比多 Agent 路径更简陋，是一个可以指出来的改进点——理想情况下单 Agent 的重试也应该把 review 的反馈传回 compose，而不是指望模型自己再蒙一次。

两条路径共同点是：都只重试一次，不会形成无限循环，这个上限是一致的。

| 缓存 Key                                        | 存储内容                 | TTL         | 目的                                                                                                 |
| --------------------------------------------- | -------------------- | ----------- | -------------------------------------------------------------------------------------------------- |
| `blacklist:{jti}`                             | `"1"`                | Token 剩余有效期 | JWT 黑名单。用户登出后将 Token 的 `jti` 加入黑名单，后续请求校验时拒绝已登出的 Token                                             |
| `user:{userId}`                               | `UserResponse JSON`  | 30 分钟       | 用户信息缓存。避免每次请求都查询 DB 加载用户资料。`AuthService` 读取，更新/删除用户时执行 evict                                       |
| `note:{noteId}`                               | `Note JSON`          | 30 分钟       | 单条笔记缓存。减少数据库查询。笔记更新/删除时执行 evict                                                                    |
| `note:list:{userId}`                          | （未写入）                | —           | 笔记列表缓存 Key。代码中仅在更新/删除笔记时执行 evict，但 `listNotes()` 未实现缓存写入，属于预留功能                                    |
| `review:today:{userId}`                       | `Map JSON`           | 10 分钟       | 当日复习数据缓存。保存用户当天待复习内容列表，避免每次打开复习页重新计算                                                               |
| `chroma:collection:id:{collectionName}`       | Collection ID 字符串    | 24 小时       | Chroma 向量库集合 ID 缓存。避免每次操作向量库时调用 Chroma API 查询 Collection ID                                        |
| `query-cache:supervisor:{sha256(query)}`      | `List<SubTask>` JSON | 10 分钟       | Agent 任务规划缓存。相同查询复用 LLM 生成的子任务拆分结果，减少 Token 消耗和响应延迟                                                |
| `query-cache:query-expansion:{sha256(query)}` | `List<String>` JSON  | 30 分钟       | 查询扩展缓存。复用 LLM 生成的扩展查询词列表，避免重复调用 LLM                                                                |
| `cache:lock:{key}`                            | `"1"`（SETNX 锁标记）     | 10 秒        | 防缓存击穿锁。缓存未命中时，只有获取锁的线程查询 DB 并回写缓存，其他线程等待后读取缓存                                                      |
| `__NULL__`                                    | `__NULL__` 空值标记      | 60 秒        | 防缓存穿透。数据库不存在的数据也缓存空标记，避免大量不存在的 Key 重复查询 DB                                                         |
| `__health_check__`                            | `"1"`                | 用完即删        | Redis 健康检查探针。`HealthController` 执行写入、读取、删除流程验证 Redis 连通性                                           |
| **缓存保护机制**                                    | **组合策略**             | **—**       | **`CacheProtectionService` 提供三层防护：① `cache:lock:{key}` 防缓存击穿；② `__NULL__` 防缓存穿透；③ 随机 TTL 抖动防缓存雪崩** |


---

## 十六、任务持久化与可观测性

#### 52. Agent 任务可以暂停和恢复吗？怎么实现的？

可以。我们实现了完整的任务暂停/恢复机制，通过 AgentTask 实体和 AgentTaskResumeService 来支持。每个对话请求创建一个 AgentTask 记录持久化到数据库，包含 taskId、sessionId、userId、query、status、startTime、totalSteps 等字段。状态机从 CREATED → PLANNING → RUNNING → PAUSED/COMPLETED/FAILED 流转。

用户可以通过前端调用 `POST /agent/tasks/{taskId}/pause` 暂停任务，系统会将 AgentTask.status 更新为 PAUSED。AgentLoop 每轮开始前都会检查任务状态，如果检测到 PAUSED，就保存当前执行快照（AgentExecutionSnapshot），包含当前轮次、已调用工具、AgentState、消息历史等，然后退出循环。

恢复时调用 `POST /agent/tasks/{taskId}/resume`，AgentTaskResumeService 从数据库加载 AgentExecutionSnapshot，重建 AgentState 和消息上下文，然后从暂停的轮次继续执行。这样长时间任务可以中断后续执行，不会因为超时或资源限制导致前功尽弃。实现上用到了 AgentResumeContext 来封装恢复上下文，通过 JSON 序列化存储到数据库。

#### 53. AgentTrace 全链路追踪记录了哪些信息？有什么用？

AgentTrace 提供全链路可观测性，每个 AgentTask 对应一条 AgentTrace 记录。记录的信息包括：规划决策（SupervisorService.plan 返回的子任务列表、执行模式 SINGLE/SEQUENTIAL/PARALLEL、规划耗时）；工具调用链（每次调用的 name/args/result/duration/quality 评级）；反思记录（ReflectionService.reflect 的 LLM 分析输出、触发原因、建议策略）；重规划记录（ReplanningService.replan 的新策略、调整的 pendingSteps）；质量评审（QualityReviewer.reviewAnswer 的 verdict/reason/feedback、是否重试）；Token 消耗统计（累计 LLM 调用的 Token 使用量）；任务总耗时（startTime/endTime）。

用途主要有三个方面。一是调试排查，可以回放完整执行过程，定位"为什么某个工具没调用"、"为什么质量审查不通过"、"在哪一步卡住的"。二是性能分析，统计"哪些工具成功率低"、"哪些查询需要反思"、"平均耗时分布"，发现系统瓶颈。三是效果评估，对比不同 Prompt 模板、不同模型、不同规划策略的表现，基于历史 trace 数据做 A/B 测试。

前端可以通过 `GET /agent/traces/{traceId}` 查询追踪详情，展示完整的执行时间线和每一步的输入输出，方便用户理解 Agent 的决策过程。AgentTrace 数据采用 JSON 字段存储工具调用、反思等详细信息，查询时按需加载，避免内存膨胀。

#### 54. 如果 Agent 执行过程中服务重启了怎么办？

服务重启会导致内存中的执行上下文丢失，但不会完全丢失任务信息。AgentTask 实体已经持久化到数据库，包含任务状态和基本信息。AgentTaskStep 记录了每一步操作的详情，可以看到重启前已经完成了哪些步骤。AgentTrace 记录了完整的执行链路，包括工具调用、反思、质量评审等。

重启后用户有两个选择。一是查看任务状态，通过 `GET /agent/tasks/{taskId}` 可以看到任务停在哪个状态、已执行步骤、失败原因等，AgentTrace 可以回溯完整过程。二是手动重试，对于失败或中断的任务，用户可以发起新的请求重新执行，因为之前的对话历史、笔记、知识库数据都还在。

目前没有实现自动从断点恢复的机制，因为 AgentExecutionSnapshot 只在用户主动暂停时保存，服务重启属于异常中断。如果要实现自动恢复，需要在每轮结束时都持久化快照，但这会增加数据库写入开销。我们的设计权衡是：正常暂停支持恢复，异常中断支持查询历史重新执行，在可靠性和性能间取了平衡。

#### 55. AgentPolicyService 的预算控制具体怎么工作的？

AgentPolicyService 配合 BudgetConfig 实现预算控制，防止 Agent 资源耗尽。BudgetConfig 定义了四个维度的限制：maxIterations（最大轮次，默认10）、maxTimeSeconds（最大执行时间，默认300秒）、maxTokens（最大 Token 消耗，默认50K）、maxToolCalls（最大工具调用次数，默认30）。

AgentLoop 每轮开始前调用 `agentPolicyService.checkBudget(state)`，检查当前累计的轮次、耗时、Token、工具调用次数是否超限。如果任意一项超限，返回 PolicyCheckResult{allowed=false, reason="..."}，AgentLoop 立即返回 BUDGET_EXCEEDED 结果，不再继续执行。

Token 统计通过 TokenCounter 估算，每次 LLM 调用后累加到 AgentState.totalTokensConsumed。工具调用次数从 AgentState.toolHistory.size() 获取。耗时通过 AgentTask.startTime 和当前时间差计算。这样即使 LLM 陷入循环或任务过于复杂，也能在超限时及时停止，避免成本失控或占用资源过久阻塞其他请求。

BudgetConfig 可以按用户、按任务类型配置不同的阈值。比如普通用户 maxIterations=10、VIP 用户可以放宽到15；简单查询类任务 maxTokens=20K、复杂分析类任务可以放宽到50K。目前代码里是全局配置，后续可以扩展为动态策略。

---

## 十七、反思与重规划机制

#### 56. ReflectionService 什么时候触发？反思结果怎么用？

ReflectionService 主要在 AgentLoop 执行过程中出现**连续无进展或者工具重复失败**时触发。目前的判断条件包括连续两轮没有明显进展，或者同一个工具连续失败。

触发后，ReflectionService 会把当前任务的 goal、工具调用历史、失败记录和观察结果交给 LLM，让它分析当前为什么没有继续推进，并返回问题原因、建议策略以及是否需要调整执行策略。

反思结果主要有两个作用：一是记录到 AgentState 的工作记忆 `AgentState.workingMemory`中，让后续轮次知道之前哪里失败过；二是触发后续的策略调整，让 Agent 下一轮尽量避免重复之前的失败路径。反思结果和重规划过程也会记录到 Trace，方便后面排查 Agent 为什么卡住。

这个机制主要解决的是**Agent 陷入局部循环的问题**。比如之前一直用 `getNote` 获取内容但参数始终不对，普通 Agent 可能会重复尝试；反思后可以明确发现问题，并提示下一轮改用 `searchNotes` 或先搜索候选内容。

**可能追问：**

- **ReflectionResult 的 `shouldReplan` 怎么判断？**：主要由 LLM 根据当前执行状态判断是否需要调整策略，同时代码对 URL 被拦截、连续无进展等明确场景也有回退判断，避免完全依赖模型。
    
- **如果反思之后还是没有效果怎么办？**：当前 AgentLoop 本身有最大执行轮次限制，达到上限后会终止执行；对于关键前置步骤失败的情况，Runtime 还可以直接进入 `NEED_CLARIFICATION`，让用户补充信息。目前没有单独实现 `MAX_REPLAN_COUNT`，所以不能说已经有完整的重规划次数控制。
    

**补充细节：**

这里的“反思”可以简单理解成：

```text
正常执行
   ↓
连续失败 / 无进展
   ↓
ReflectionService
   ↓
分析：为什么失败？
   ↓
ReflectionResult
   ↓
告诉 Agent：
“之前这条路不行，可以尝试另一种方式”
   ↓
继续执行
```

需要特别注意：**当前实现中的 Replanning 并不是重新生成一套完整的 SubTask 计划。** AgentLoop 中的重规划主要是把反思结果和当前状态重新注入上下文，让 LLM 在下一轮选择不同的工具或策略。

#### 57. ReplanningService 跟 Supervisor 有什么区别？

两者最大的区别是**触发时机和负责的层级不同**。

Supervisor 是任务开始时的初始规划，主要根据用户的问题判断是否需要拆分任务，并生成多个 SubTask，同时确定任务之间的依赖关系和执行模式。它解决的是**整个任务应该怎么拆、怎么组织**。

ReplanningService 则是在 Agent 执行过程中发现问题后，对当前执行策略进行调整。它会结合当前 AgentState 和 ReflectionResult，告诉 Agent 当前哪里出了问题、后面可以尝试什么不同的策略。它解决的是**当前这个子任务接下来应该怎么走**。

所以两者的粒度也不同。Supervisor 是比较高层的任务编排，比如：

```text
用户问题
  ↓
Supervisor
  ↓
Task A：搜索相关资料
Task B：分析资料
Task C：生成结果
```

而 Replanning 更像是：

```text
Task A 执行
  ↓
发现 searchNotes 失败
  ↓
Reflection
  ↓
Replanning
  ↓
改用其他搜索策略
```

所以我一般会把它理解成：**Supervisor 是事前的战略规划，Replanning 是执行过程中的战术调整。**

**可能追问：**

- **Replanning 会重新生成 SubTask 吗？**：目前不会。Supervisor 负责生成和拆分 SubTask，Replanning 主要调整当前 AgentLoop 的执行策略，不会重新编排整个 Supervisor-Worker 任务树。
    
	**那为什么Replanning 不设置成重新生成一套完整的 SubTask 计划呢？**：**主要是因为 Supervisor 和 Replanning 解决的问题不同。** Supervisor 负责的是任务级别的拆分，如果执行过程中只是某一个 Worker 的工具调用失败或者策略不合适，没有必要把整个任务重新拆一遍。
	比如 Supervisor 已经把任务拆成“搜索资料 → 分析资料 → 生成结果”，如果只是“搜索资料”这个子任务里的某个工具调用失败，这时候更合理的是在当前 Worker 内部换一种搜索方式，而不是重新让 Supervisor 把整个任务重新规划一次。
	另外，重新生成完整 SubTask 计划会带来更大的上下文和执行成本，还可能导致原来已经完成的任务被重复执行。所以目前设计成**局部策略调整**，尽量复用已经完成的结果，只调整当前卡住的执行路径。
	如果以后出现的是**任务目标发生变化、多个 Worker 都失败，或者当前任务依赖关系本身有问题**，那时候再考虑升级到 Supervisor 级别的全局 Replanning 会更合适。
	
- **如果重规划之后还是失败怎么办？**：当前主要依靠 AgentLoop 的最大轮次限制；如果是关键的 FETCH 前置步骤失败，Runtime 会直接进入 `NEED_CLARIFICATION`，让用户介入，而不是继续无限尝试。
    
- **为什么不每次失败都重规划？**：因为反思和重规划本身也需要额外的 LLM 调用。如果只是一次临时的工具失败就重新规划，会增加 Token、延迟和系统开销，所以目前是在连续无进展或明确失败的情况下触发。
    

**补充细节：**

你可以把两者简单记成：

```text
Supervisor
   ↓
“这个大问题应该怎么拆？”
   ↓
SubTask A / B / C
```

而：

```text
Replanning
   ↓
“当前这个子任务走不通了，下一步换什么方法？”
```

还有一个非常重要的代码层面区别：

**AgentLoop 里的 Replanning 和 AgentRuntime 里的关键步骤失败处理不是完全一回事。**

AgentLoop 中：

```text
反思
 ↓
ReplanningService
 ↓
更新执行上下文 / 注入状态
 ↓
继续下一轮 Agent Loop
```

也就是说，它主要是**调整当前 Agent 的行为，引导下一轮换策略**。

而 AgentRuntime 对关键 FETCH 步骤达到 `MAX_ROUNDS` 并确认失败时，目前的处理更加保守：

```text
FETCH 失败
 ↓
Runtime 判断关键步骤失败
 ↓
NEED_CLARIFICATION
 ↓
任务暂停 / 用户介入
```

因此面试时最好不要说：

> “重规划失败后会不断生成新的计划。”

更准确的说法是：

> **“当前版本的重规划主要是执行策略层面的调整，不是重新生成整个 SubTask 计划；如果继续执行仍然无法推进，会受到最大轮次限制，关键前置步骤失败则会进入 NEED_CLARIFICATION。”**

---

## 十八、Agent 范式深度对比

#### 49. 这个 Agent 流程是更接近于 ReAct 范式还是 Plan-Execute 范式？

**结论：不是纯 ReAct，也不是纯 Plan-Execute。是 Hierarchical ReAct（也叫 Goal-Conditioned ReAct）——外层 Plan-then-Solve，内层 ReAct。**

**两个范式的本质区别：**

纯 ReAct 是"没有地图，走一步看一步"。每轮 Thought → Action → Observation，下一步往哪走完全靠当前 Observation 推导。好处是灵活，坏处是复杂任务容易迷路——LLM 可能在第三步忘了第一步的意图，或者提前宣布"我做好了"。

纯 Plan-Execute 是"先画完整地图，然后照着走"。先 LLM 生成完整的步骤序列，然后逐个机械执行。好处是稳定不走偏，坏处是僵化——执行到第三步发现第一步拿到的结果跟预想不一样，计划后半段全废了。

**这个项目的三层混合设计：**

| 层级 | 范式 | 做什么 |
|------|------|--------|
| 外层 Supervisor | Plan（轻量） | 把复杂意图拆成独立子目标，定 executionMode |
| 内层 AgentLoop | ReAct + GoalEvaluator | 每个子目标内自主探索，达成即停 |
| 收尾 Writer/Composer | Synthesize | 把各子目标结果合成一个回答 |

**外层 Supervisor = 轻量 Plan，只定目标，不定具体操作：**

Supervisor 不告诉你"第一步调 searchNotes、第二步调 getNote、第三步调 generateMindMap"。它拆的是语义级别的子目标：

```json
[
  {"goal": "搜索JVM内存结构笔记并读取完整内容", "toolHint": "searchNotes + getNote"},
  {"goal": "基于笔记内容生成思维导图", "toolHint": "generateMindMap"},
  {"goal": "将思维导图追加到原笔记", "toolHint": "appendNote"}
]
```

toolHint 只是建议，AgentLoop 可以自己换工具。Supervisor 定的是"做什么"，不是"怎么做"。

**内层 AgentLoop = 纯 ReAct，但加了 GoalEvaluator 收敛边界：**

每个子目标的执行仍然是标准 ReAct：LLM 每轮自主选工具、观察结果、调整策略。GoalEvaluator 的作用不是推着 LLM 按计划走，而是判断"这个子目标够了，别再绕了"。

**GoalEvaluator 的阻断方向跟 Plan-Execute 的执行方向是反的：**

- Plan-Execute 是**正向推动**："计划说第三步要做 X，所以你现在必须做 X"
- GoalEvaluator 是**反向收敛**："不管你怎么做，只要满足了这些条件，立即停"

**总结：ReAct 解决单目标内的灵活性，Plan 解决多目标间的复杂度，GoalEvaluator 解决 ReAct 不知道什么时候该停的问题。** 三个东西拼在一起，缺一个都不完整。

---

## 十九、GoalEvaluator 的停止判断与兜底机制

#### 50. GoalEvaluator 怎么判断什么时候该停？如果检索召回了无关内容但 ToolResult 是 SUCCESS，会误判为目标达成吗？

**GoalEvaluator 的判断分两个入口：**

**入口一：工具成功后（evaluateAfterToolSuccess）**

只有三种情况直接判定 ACHIEVED：

| 场景 | 触发条件 |
|------|---------|
| 写操作 | createNote/editNote/appendNote/deleteNote/mergeNotes/markReviewed/scheduleReview 成功 |
| 产物生成 | generateMindMap/generateDiagram 成功（且不需要写回笔记） |
| 读取笔记 | getNote 成功 + goal 里明确是"读取"类目标 |

**关键点：searchNotes、ragSummary、listNotes 这些检索工具成功执行后，GoalEvaluator 返回的是 UNDETERMINED，不是 ACHIEVED。** 它不会因为"搜到了东西"就判目标达成。

**入口二：LLM 想直接回答时（evaluateOnTextResponse）**

判断链：
1. 一个工具都没调过 → 拦截，强制调工具（除非纯通用知识问题）
2. 调了工具但全部 POOR/ERROR → 拦截，换策略重试
3. 调了工具且有成功的 → 分叉：如果是"总结/解释/概述"类问题 → 直接放行；否则 → 调另一个 LLM 做目标检查

**关于"检索到无关内容"的边界情况：**

以"读取线程相关的笔记并总结"为例，如果 searchNotes 返回了不相关的笔记（ToolResult=SUCCESS），链路是：

1. ToolResultEvaluator → 看到 SUCCESS → GOOD（不做语义相关性判断）
2. GoalEvaluator.evaluateAfterToolSuccess → searchNotes → UNDETERMINED（继续循环，正确）
3. 如果 LLM 决定直接回答 → isInformationalAnswerQuery 检测到"总结"关键词 → **直接放行**

**坦率说，这里确实存在一个漏洞。** 对于总结/解释类问题，GoalEvaluator 会跳过 LLM 目标检查直接放行，这是为了减少早期版本中频繁误拦截问题做的权衡。如果 LLM 自己没有意识到搜索结果不相关，就可能基于无关内容生成回答。

**四层兜底：**

1. **LLM 自身判断力**：ReAct 循环内 LLM 应该能识别搜索结果相关性，自己换关键词重搜
2. **AgentLoop 防偷懒机制**：shouldForceRagBeforeDirectAnswer 检测到知识类问题且未调工具时强制注入检索提示
3. **QualityReviewer**：最终回答生成后把回答跟检索证据做对比，不一致时驳回
4. **ToolResultEvaluator 的 EMPTY 分支**：真正空结果会被标记 POOR，驱动换策略

**ToolResultEvaluator 的 POOR 含义澄清：**

POOR 不是"LLM 判断内容质量差"，而是"工具正常跑完了但返回空结果"。ToolResultEvaluator 是纯规则判断，跟 LLM 无关：

| ToolResult.status | → ResultQuality | 含义 |
|---|---|---|
| SUCCESS | GOOD | 工具跑通，拿到了数据 |
| EMPTY | POOR | 工具跑通，但结果是空的 |
| ERROR | ERROR | 工具执行异常 |

POOR 的作用是驱动恢复——给出策略建议（如 searchNotes 无结果 → "换关键词或改用 listNotes"），防止 LLM 在空结果上反复重试。

**根本局限：** ToolResultEvaluator 做的是信号层面的质量判断（有没有结果、有没有报错），不是语义层面的相关性判断（结果对不对口）。要做语义相关性判断需要在每次工具调用后额外调 LLM 或 Cross-Encoder，有成本和延迟代价，目前没做。

**改进方向：** 要么在 ToolResultEvaluator 里加入语义相关性评估（用轻量规则或 embedding 相似度快速判断），要么去掉 isInformationalAnswerQuery 的快速放行、改为必须通过 LLM 目标检查但优化检查 prompt 减少误判。

---

## 二十、Agent 通信与 A2A 协议

#### 51. 你的多 Agent 之间是怎么通信的？了解 A2A 协议吗？为什么不用这个？

在这个项目中，多 Agent 不是通过 Kafka、RabbitMQ 或 A2A 这类独立通信协议通信，而是采用 Spring 服务内部编排、Java 方法调用和共享上下文对象的方式实现。

项目入口主要是 `AgentService`。用户请求进入后，`AgentService.streamAgentResponse()` 调用 `SupervisorService.plan(query)`。Supervisor 使用 LLM 分析用户问题，判断是否需要拆分任务，然后返回 JSON 数组形式的 `SubTask` 列表。每个子任务包含任务 ID、描述、目标、建议工具、成功标准、依赖关系和执行模式。这里的通信载体实际是 `List<SubTask>` 和 Java 对象，不是网络协议。

如果是顺序执行，项目调用 `runSequentialPipeline()`，依次运行多个 `AgentLoop`。前一个子 Agent 执行完后，系统使用 `ResponseComposer.compose()` 生成结果文本，再把结果、关键事实和产物内容追加到 `completedContext`，通过 `buildSequentialTaskQuery()` 注入下一个 Agent 的用户消息。因此，顺序 Agent 之间通过累积文本上下文传递信息，后面的 Agent 可以看到前面所有子任务的结果。

如果是并行执行，项目调用 `runParallelPipeline()`，通过 `CompletableFuture.runAsync()` 和 `taskExecutor` 线程池并发运行多个子 Agent。每个子 Agent 独立调用 `AgentLoop.run()`，执行结束后生成 `SubResult`，包含任务 ID、标签、回答片段和 `AgentState`。系统等待全部任务完成后，合并各个 `AgentState`，再由 `WriterService.synthesize()` 统一合成最终答案，并由 `QualityReviewer` 做质量审查。

单个 `AgentLoop` 内部，Agent 之间更准确地说是 Agent 与模型、工具之间通过 LangChain4j 消息对象通信。消息包括 `SystemMessage`、`UserMessage`、`AiMessage` 和 `ToolExecutionResultMessage`。LLM 决定调用工具后，`AgentTools` 执行知识库检索、笔记查询、笔记修改或导图生成等操作，工具结果被包装成 Observation，重新加入当前消息列表，供下一轮 LLM 决策。每个子任务还会创建独立的 `AgentState`，记录工具调用历史、Observation、已知事实、产物和证据等级，最后再由父流程合并。

A2A 协议我了解。A2A，也就是 Agent-to-Agent 协议，主要解决不同厂商、不同框架、不同进程中的 Agent 如何通过标准协议发现、调用、传递任务状态和返回结果的问题。它更适合跨系统、跨服务、跨厂商的 Agent 协作。

当前项目没有使用 A2A，主要有四个原因：

1. 当前系统是单体 Spring Boot 应用，Supervisor、AgentLoop、WriterService、QualityReviewer 和 AgentTools 都在同一个 JVM 内，直接 Java 方法调用比 HTTP 协议通信更简单、延迟更低。
2. 当前这些 Agent 角色是系统内部固定模块，不是需要被外部发现和调用的独立 Agent。
3. 项目当前重点是任务拆分、工具调用、RAG 检索和结果合成，使用 `SubTask`、`AgentState`、消息列表和 `completedContext` 已经能满足需求。
4. 引入 A2A 需要额外处理服务发现、协议适配、鉴权、序列化、超时、重试和版本兼容，当前没有跨系统协作需求，投入产出比不高。

所以我的总结是：当前项目采用单体应用内的多 Agent 编排。Supervisor 通过 `SubTask` 分发任务，顺序任务通过 `completedContext` 传递结果，并行任务通过 `CompletableFuture` 和线程池执行，最终由 WriterService 汇总、QualityReviewer 审查。由于 Agent 都在同一个 Spring Boot 进程内，暂时不需要 A2A。未来如果把各个 Agent 拆成独立服务，或需要接入外部、异构 Agent，再考虑 A2A 会更合适。

