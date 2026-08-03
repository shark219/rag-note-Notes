# RAG 消融实验修改方案（最终定稿 v2）

> 目标：**不是证明某个 RAG 模块理论有效，而是在当前项目场景下找到最优可落地 Pipeline，并判断每个环节是否值得存在。**
>
> 评价体系：效果（综合得分）× 成本（耗时 + Token）双指标驱动，使用 **收益/成本比** 进行模块价值判断。
>
> **本次核心变更**：重新定义 RAG Ablation 实验集（BASELINE + R-1~R-5），移除 R-6~R-9，并补齐实验平台工程能力（run_id、隔离存储、分阶段成本统计）。

---

## 一、实验边界重新定义（本次核心改动）

### 1.1 保留的 RAG 组件消融实验集

| 编号 | 实验 | 关闭组件 | 类型 |
|------|------|---------|------|
| BASELINE | 完整 RAG Pipeline | none | — |
| R-1 | w/o Query Expander | QueryExpander | 查询增强 |
| R-2 | w/o Vector Search | VectorSearch | 检索 |
| R-3 | w/o BM25 Search | BM25Search | 检索 |
| R-4 | w/o RRF Fusion | RRFFusion | 融合排序 |
| R-5 | w/o Reranker | Reranker | 精排 |

> 该集合覆盖在线 RAG 检索链路：**Query理解 → 召回 → 融合 → 排序 → 生成上下文**。5 个组件刚好对应完整链路，规模收敛但实验可信度更高。
>
> 注：现有代码中 `AblationConfig.allRagExperiments()` 的 R-1~R-5 编号已与上表一致，无需重新编号，只需从执行列表中移除 R-6/R-7。

### 1.2 从消融实验集中移除的实验

| 原实验 | 移除原因 | 去向 |
|--------|---------|------|
| R-6 w/o Source Attribution | 属于回答生成/引用机制，影响可解释性而非检索增强核心 | 功能测试 / 未来 Answer Quality Ablation |
| R-7 w/o Retrieval QualityReviewer | 属于 Agent Answer Review，不属于 RAG | 未来 Agent Ablation |
| R-8 ChunkSize | 索引构建参数，不属于在线 RAG Pipeline | 参数实验（Index Experiment，独立） |
| R-9 TopK | 检索参数调优，不属于组件消融 | 参数实验（Index Experiment，独立） |

> 结论：删除不代表"不再研究"，而是**不再纳入 RAG 组件消融实验集**。Source Attribution / QualityReviewer 归入"回答质量 / Agent"类实验；Chunk / TopK 归入"参数实验"，与组件消融分开设计、分开报告。

---

## 二、移除实验的论证

### 2.1 Source Attribution（回答来源引用）

- **情况 A**：只在回答末尾追加来源引用 → 影响可解释性、用户信任，**不影响** retrieval quality / answer correctness，属于 UX / Explainability 实验。
- **情况 B**：影响生成 Prompt（要求"引用来源"）→ 影响 LLM 行为，但仍属 Generation Prompt 范畴。

结论：**不放 RAG Ablation**。未来可做 Answer Quality Ablation（w/o Citation、w/o Reviewer、w/o Reflection）。

### 2.2 Retrieval QualityReviewer

- 属于 Agent 的 Answer Review 环节，不改变检索召回结果，不影响检索增强核心。

结论：**移出 RAG 消融**，未来归入 Agent Ablation（w/o Reviewer、w/o Memory、w/o Tool Router、w/o Reflection）。

### 2.3 ChunkSize / TopK

- 属于索引构建 / 检索参数，不是在线 Pipeline 的组件开关。

结论：**改为参数实验（Index Experiment）**，与组件消融分开。现有代码中的 R-8a~R-8d、R-9a~R-9d 保留为参数对比实验，但不再算作"消融实验"。

---

## 三、已有实验数据（基线，保留参考）

上一轮跑出的结果作为参考基线，被移出集合的实验数据标注去向：

| 实验 | 综合得分 | Δ效果 | 归属 |
|------|---------|-------|------|
| Full（完整流水线） | 96.2 | — | BASELINE |
| w/o Vector Search | 30.2 | -64.7% | R-2（必须保留） |
| w/o BM25 Search | 91.4 | -3.5% | R-3 |
| w/o Reranker | 91.5 | -3.4% | R-5 |
| w/o RRF Fusion | 97.5 | +1.3% | R-4（疑似负收益） |
| w/o Query Expander | 97.0 | +0.8% | R-1（疑似负收益） |
| w/o Retrieval QualityReviewer | — | -9.0% | 移出 → Agent Ablation |
| w/o Source Attribution | — | +0.4% | 移出 → 功能测试 |

> 说明：以上 Δ 来自上轮运行。重新定义实验集后，需在统一实验平台（第五节）下重跑，并采用统一评分口径（第六节）。

---

## 四、实验配置结构重构

现状：`AblationConfig.allRagExperiments()` 混入了 RAG / Index / Agent 三类实验。

建议拆分：

### 4.1 RAG Ablation

```java
RagAblationConfig.ragPipelineExperiments();
// 返回 [BASELINE, R-1, R-2, R-3, R-4, R-5]
```

### 4.2 Agent Ablation（未来）

```java
AgentAblationConfig.agentExperiments();
// 例如 [BASELINE, w/o REVIEWER, w/o MEMORY, w/o TOOL_ROUTER, w/o REFLECTION]
```

### 4.3 Index Experiment（参数实验，独立，不作为消融）

```java
IndexExperimentConfig;
// Chunk / Overlap / TopK 参数对比
```

> 代码调整：
> - `runAllRagAblationExperiments()` 只执行 BASELINE + R-1~R-5（当前跑到 R-7）。
> - `chunkSizeExperiments()` / `topKExperiments()` 移出消融主流程，单独入口（当前 `/run-topk` 已独立）。
> - `/evaluation/ablation/experiments` 的配置列表相应去掉 R-6~R-9。

---

## 五、实验平台工程改造（必须）

### 5.1 run_id 实验运行记录（ablation_runs 表）

```sql
CREATE TABLE ablation_runs (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    run_id VARCHAR(36) NOT NULL,          -- 每次 run-all 生成
    user_id VARCHAR(36),
    status VARCHAR(20),                   -- CREATED / RUNNING / SUCCESS / FAILED / CANCELLED
    started_at DATETIME,
    finished_at DATETIME,
    total_cases INT,
    success_cases INT,
    fixed_config TEXT,                    -- 固定参数快照（embedding / chunk / topK / 测试集等）
    question_snapshot TEXT,               -- 本次 run 使用的测试用例 question 列表
    created_at DATETIME
);
```

### 5.2 隔离存储：不再写 evaluation_reports

消融实验评估结果会污染实时评估报表（趋势 / 分布 / 低分样本）。两种方案：

- **推荐**：新增独立表 `ablation_evaluation_reports`（结构同 EvaluationReport + run_id），消融评估全部写入此表。
- 备选：evaluation_reports 增加 `evaluation_source` 字段（REALTIME / ABLATION）。

### 5.3 ablation_results 细化到 question 粒度

```sql
-- ablation_results 增加/调整字段
run_id VARCHAR(36)
question_id BIGINT                        -- 关联测试用例 id
answer TEXT
score DOUBLE
latency_ms BIGINT
input_tokens INT
output_tokens INT
cost DECIMAL(10,6)
```

> 从"按实验聚合"改为"按 (run_id, experiment_id, question_id) 一行"，实验级指标（综合得分、平均耗时、Token）由明细汇总而来，问题级别可审计、可追溯。现有聚合字段（compositeScore / avgLatencyMs / avgTokenConsumed 等）可保留为汇总冗余或改为视图计算。

### 5.4 实验流程

```
Create Experiment Run（生成 run_id，状态 CREATED）
        ↓
加载当前用户测试用例，快照 question 列表（写入 question_snapshot）
        ↓
执行各实验（BASELINE → R-1~R-5）
   逐 case：执行 RAG → 评估（写 ablation_evaluation_reports）→ 明细写 ablation_results(run_id)
        ↓
状态置 SUCCESS / FAILED（记录 success_cases / total_cases）
```

### 5.5 评分统一口径

- BASELINE 与各消融实验使用**同一套评估逻辑**（EvaluationService）与**同一套权重**（`app.evaluation.*`），保证 Δ 可比。
- 综合得分 = 加权综合（与 EvaluationService 保持一致）；生产/回归口径不混用。

---

## 六、成本统计必须补（分阶段 Token）

**问题**：现有 `tokenConsumed` 只统计最终 LLM 生成，QueryExpander / Reranker 等阶段的 Token 缺失，导致"关闭 QueryExpander"的成本变化恒为 0，收益/成本比失真。

**改造目标**：RAG 各阶段 Token 分别统计，统一汇总。

```json
{
  "retrievalTokens": 0,
  "queryExpansionTokens": 300,
  "rerankerTokens": 500,
  "generationTokens": 800,
  "totalTokens": 1600
}
```

**评价指标定义（沿用）：**

```
效果收益（ΔScore） = 模块开启得分 - 模块关闭得分

成本增量（ΔCost）  = latency增比例 + token增比例
                    = (L_open - L_close) / L_close + (T_open - T_close) / T_close

模块收益率          = ΔScore / ΔCost
```

> token 增量使用 totalTokens（各分阶段之和）；latency 使用分阶段（retrieval / queryExpansion / rerank / generation）合计，不再只取 `totalLatencyMs`。

---

## 七、测试集：继续使用当前用户测试用例（明确决定）

- **不引入独立 EvaluationDataset / evaluation_questions 表**。
- 每次 run-all 时对"当前用户的测试用例"做一次**快照**：将 question 列表写入本次 run 的记录（ablation_runs.question_snapshot），ablation_results 记录 question_id。
- 好处：既保持"用现在的用户的测试用例"，又保证单次 run 内 question 集合一致、可追溯；不同 run 间测试集变化可通过"用例数 / 快照"识别，报告中显示本次用例数。
- 注意：跑实验前先去重（已有 dedup 功能）并固定测试用例，避免重复用例稀释指标。

---

## 八、整体路线（重新编排）

```
Step 0: 实验平台改造（run_id / 隔离存储 / 分阶段 Token）
  ↓
Phase 1: RAG 组件消融（BASELINE + R-1~R-5）
  逐模块收益/成本比
  ↓
Phase 2: 问题模块优化
  RRF（参数调整 + 架构调整）
  QE（降温 + 条件触发 + 实体保持约束）
  ↓
Phase 3: 候选 Pipeline 对比（决策矩阵）
  ↓
（独立工作流）参数实验：Chunk / Overlap / TopK —— 不属于消融，可并行
```

---

## 九、RAG 组件消融实验设计（BASELINE + R-1~R-5）

固定参数：Embedding 模型 / Chunk / TopK（写入 ablation_runs.fixed_config，报告页面展示）。

每组输出价值评估卡：

```
模块名称: BM25 Search
ΔScore:   +3.5
ΔLatency: +300ms
ΔToken:   +200（含各分阶段合计）
ΔCost:    0.15  (latency增比例 + token增比例)
收益率:    0.233 (ΔScore / ΔCost)
决策:      保留（收益率中等，精确匹配场景不可替代）
```

---

## 十、问题模块优化（RRF / QE）

### 10.1 RRF 优化验证

**Step 1：场景价值分析**

抽 20~30 个 case，按问题类型分类：

| 问题类型 | 数量 | RRF 有帮助 | RRF 有伤害 | 无影响 | 分析 |
|---------|------|-----------|-----------|--------|------|
| 专业术语/代码 | N | ? | ? | ? | 高价值场景 |
| 语义理解类 | N | ? | ? | ? | Vector 主导 |
| 混合查询 | N | ? | ? | ? | — |

**判断逻辑**：不机械使用"80% 无影响则删除"，而是看 RRF 在**高价值场景**（专业术语、配置、异常排查等）是否有不可替代的帮助。

**Step 2：两种优化方案**

| 方案 | 改动 | 说明 |
|------|------|------|
| **RRF-v1** | 调 BM25/Vector 各路 topK 配比、k 值参数 | 在当前框架内优化 RRF |
| **Fusion-v2** | 改为 BM25→Rerank + Vector→Rerank，最后结果合并 | 跳过 RRF，改变融合架构 |

### 10.2 QE 优化验证

**Step 1**：抽 10 个 QE 导致下降的 case，分析原因。

**Step 2：三种优化方案**

| 方案 | 改动 | 说明 |
|------|------|------|
| **QE-v1** | temperature 0.8 → 0.3 | 减少随机性，降低离谱扩展 |
| **QE-v2** | 条件触发：检索结果不足时才扩展 | 默认不扩展，需要时启用 |
| **QE-v3** | 增加实体保持约束 Prompt | 强制保留产品名、版本号、类名、方法名、错误信息等关键实体 |

---

## 十一、候选 Pipeline 对比

Reviewer 移出 RAG 范围后，候选管线只针对 RAG 组件：

| 方案 | Pipeline | 说明 |
|------|----------|------|
| **A** | Full（QE+V+BM25+RRF+Rerank） | 当前完整方案（对照） |
| **B** | 优化 RRF 后的完整版 | RRF-v1 |
| **C** | 去 RRF（QE+V+BM25+Rerank） | 核心候选 |
| **D** | 条件 QE（QE-v2+V+BM25+RRF+Rerank） | 条件 QE 方案 |
| **F** | **最优核心（QE+V+BM25+Rerank）** | **最可能最终方案** |
| **E** | 低成本（V+Rerank） | 极致精简 |

### 决策矩阵

| 方案 | Pipeline | 综合得分 | Δ效果 | Latency | Token | 收益率 | 结论 |
|------|----------|---------|-------|---------|-------|--------|------|
| A | Full | baseline | — | baseline | baseline | — | 对照 |
| B | 优化后完整 | ? | ? | ? | ? | ? | 候选 |
| C | 去 RRF | ? | ? | ? | ? | ? | 候选 |
| D | 条件 QE | ? | ? | ? | ? | ? | 候选 |
| **F** | **QE+V+BM25+Rerank** | **?** | **?** | **?** | **?** | **?** | **★ 最可能** |
| E | 低成本 | ? | ? | ? | ? | ? | 候选 |

### 最终决策原则

- **效果优先**：如果某方案效果与最高分差距 < 1%，选成本最低的
- **成本敏感**：如果某方案效果提升 < 3% 但耗时翻倍，放弃
- **场景适配**：如果某模块只在特定场景有效（如专业术语），通过条件启停保留而非全局删除

---

## 十二、报告页面调整

"实验配置说明"改为 **"RAG Pipeline 消融实验"**，展示：

```
固定配置：
Embedding: xxx
Chunk: xxx
TopK: xxx
测试集: 当前用户测试用例（N cases，快照于本次 run）

实验:
BASELINE  完整 RAG Pipeline
R-1       去除 Query Expansion
R-2       去除 Vector Search
R-3       去除 BM25 Search
R-4       去除 RRF Fusion
R-5       去除 Reranker
```

并展示 run 状态（RUNNING / SUCCESS / FAILED）与历史 run 结果（按 run_id 切换），而非只显示"最新一次"。

---

## 十三、架构总结

```
Evaluation Platform
 ├─ RAG Ablation      BASELINE + R-1~R-5（本阶段核心）
 ├─ Agent Ablation    未来：Reviewer / Memory / Tool Router / Reflection
 └─ Index Experiment  未来：Chunk / TopK 参数（独立）
```

---

## 十四、实施时间线

| 阶段 | 内容 | 预计耗时 |
|------|------|---------|
| Step 0 | 实验平台改造（run_id / ablation_runs / 隔离存储 / 分阶段 Token） | 1 天 |
| Phase 1 | RAG 组件消融 BASELINE + R-1~R-5（含成本分析） | 半天 |
| Phase 2 | RRF 场景分析 + 2 种优化 + QE case 分析 + 3 种优化 | 半天 |
| Phase 3 | 6 个候选 Pipeline 对比 + 决策矩阵 | 半天 |
| 参数实验 | Chunk / Overlap / TopK（可选，独立） | 半天 |
| **总计** | | **约 3 天** |

---

## 十五、不纳入本次 RAG Ablation 的内容

| 原计划项 | 处理 | 理由 |
|---------|------|------|
| Source Attribution 消融 | 移出，改为功能测试 | 影响可解释性而非准确率 |
| Retrieval QualityReviewer 消融 | 移出，归入未来 Agent Ablation | 属于 Agent Answer Review |
| Chunk / TopK 消融 | 移出，改为独立参数实验 | 索引/检索参数，非组件消融 |
| Agent 消融 A-1~A-8 | 降为低优先级 | 当前焦点是 RAG Pipeline 决策 |
| 联合消融 C-1~C-5 | 取消 | 工程决策不需要交互效应分析 |
| Recall@K / nDCG | 不加入 | 最终决策只看综合得分 + 成本 |
| P95/P99 延迟 | 不加入 | 平均耗时足够做工程决策 |
| 统计显著性/多轮运行 | 不加入 | 工程决策不需要 mean±std |
| 固定测试数据集（EvaluationDataset） | 不加入 | 继续使用当前用户测试用例（见第七节） |

---

## 十六、最终能回答的问题

1. **Vector 是否必须？** → 是（-64.7%，已有定论）
2. **BM25 是否值得？** → Phase 1 看收益率
3. **Reranker 是否值得？** → Phase 1 看收益率
4. **RRF 是删除还是优化？** → Phase 2 先优化再决策
5. **QE 是删除还是优化？** → Phase 2 先优化再决策
6. **最终上线 Pipeline 是什么？** → Phase 3 决策矩阵输出
7. **Chunk / TopK 最佳参数？** → 参数实验（独立工作流）
