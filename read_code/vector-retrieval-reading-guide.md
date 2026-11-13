# seekdb 向量检索 / 混合搜索主线阅读指南

本文按"从用户可见的 SQL 到底层索引库"的顺序，梳理 seekdb 向量检索与混合搜索的完整代码路径。

---

## 一、先从 SQL 语法入手（建立心智模型）

看两个测试用例目录，这是最快的入口：

**向量查询**（`tools/deploy/mysql_test/test_suite/vector_index/t/vector_similarity.test`）：

```sql
-- 建索引：声明距离度量 + 索引类型
create table t1 (c1 int, c3 vector(3), primary key(c1),
  vector index idx1(c3) with (distance=l2, type=hnsw));

-- ANN 查询：approx limit 触发走索引而非全表算距离
select c1 from t1
  where c2 < 4
  order by l2_distance(c3, [1,1,2]) approx limit 5;
```

**全文检索**（`test_suite/fts_index/t/simple_query.test`）：

```sql
select * from articles where match(title, body) against('abc df');
```

**混合向量索引**——text 列 + 自动 embedding（`create_table_with_hybrid_vector_index.test`）：

```sql
create table t(c2 varchar(100),
  vector index idx(c2) with (distance=l2, type=hnsw,
    model=ob_embed, dim=1024, sync_mode=immediate),
  fulltext index idx2(c3));
```

三种能力（向量 / 全文 / 标量过滤）可以出现在同一条 SQL 里，这就是"混合检索"。

---

## 二、整体分层架构

```
SQL 语法:  ORDER BY l2_distance(...) APPROX LIMIT / MATCH...AGAINST
   │
   ▼
┌─ SQL 层  src/sql/ ──────────────────────────────┐
│  parser/resolver   识别 APPROX、vector 类型      │
│  optimizer         选择向量索引扫描路径           │
│  DAS 迭代器         HNSW/IVF/FTS 的物理执行单元   │  ← 读路径核心
└─────────────────────────────────────────────────┘
   │  通过接口缝（seam）解耦
   ▼
┌─ Query 服务层  src/query/api/query/vector/ ─────┐
│  ObIVectorIndexService   SQL↔存储的边界接口      │
│  ObPluginVectorIndexAdaptor  大适配器：管增量+   │  ← 两级索引核心
│    快照两套 vsag 索引、内存、位图                │
└─────────────────────────────────────────────────┘
   │
   ▼
┌─ Data Plane 接口  src/data_plane/api/data_plane/ ┐
│  vector/   距离函数：L2/cosine/IP/L1、PQ 编码    │
│  retrieval/  ObRetrievalProgram、ObSparseRetrieval│  ← 混合融合抽象
└─────────────────────────────────────────────────┘
   │
   ▼
┌─ 存储层  src/storage/ ─────────────────────────┐
│  retrieval/  BM25 迭代器：DaaT/TaaT/BMW          │
│  fts/        分词器：ik/ngram/空格               │
│  vector_index/ kmeans（IVF 聚类）                │
└─────────────────────────────────────────────────┘
   ▲ 写路径异步回填
   │
┌─ 观察者异步任务  src/observer/ ─────────────────┐
│  change_stream/ob_cs_plugin_async_index          │  ← 写路径核心
│  vector_index/*  后台刷新、embedding 任务         │
└─────────────────────────────────────────────────┘
   │
   ▼
  vsag（第三方 HNSW 库，deps 里，cmake/Env.cmake 定义 OB_VSAG_DIR）
```

一个值得注意的工程模式：这个代码库大量使用 **"接口缝 + 组装根"**。比如 `ObIDASTextRetrievalEngine`（`src/sql/das/iter/sparse_retrieval/ob_das_text_retrieval_engine.h`）定义在 SQL 层，头文件注释明说"SQL 拥有请求和 DAS 生命周期，实现不允许出现在这里"；具体实现在 `src/observer/composition/retrieval/` 里通过工厂函数在进程启动时注入。读代码时要意识到这条边界，否则容易在 SQL 层找不到实现。

---

## 三、读路径①：向量查询（HNSW）

### 调用链

```
ORDER BY l2_distance(...) APPROX LIMIT
  → resolver 把 approx limit 标记进语句（src/sql/resolver/dml/）
  → optimizer 识别向量索引可用（src/sql/optimizer/ob_log_table_scan.cpp、ob_join_order.cpp）
  → 生成 DAS_ITER_HNSW_SCAN 迭代器
     ObDASHNSWScanIter（src/sql/das/iter/ob_das_hnsw_scan_iter.cpp）
  → acquire_adapter_guard() 拿到 ObPluginVectorIndexAdaptor
  → vsag_query_vids() 调 vsag 库做图搜索
  → 结果 vid → 回表/回主键拿原始行
```

### 核心文件：`ob_das_hnsw_scan_iter.h`（必读）

`ObDASHNSWScanIterParam` 里的一组成员变量直接暴露了两级索引的设计：

```cpp
ObDASScanIter *delta_buf_iter_;   // 增量缓冲：刚写入还没进索引的数据
ObDASScanIter *snapshot_iter_;    // 快照索引：已构建完成的 HNSW
ObDASScanIter *vid_rowkey_iter_;  // vid ↔ 主键 映射回表
```

对应适配器里的两个枚举值 `VIRT_INC`（增量索引）和 `VIRT_SNAP`（快照索引）。查询时**两边都查再按距离堆合并**，这就是 README 说"新写入的向量立即可搜索"的实现基础。

### 自适应计划（很精彩的部分）

`vec_index_type_` 有几种模式：`VEC_INDEX_ADAPTIVE_SCAN / PRE / POST_ITERATIVE_FILTER`。看 `ob_das_hnsw_scan_iter.cpp:754` 的 `check_pre_filter_need_retry()`：

- 标量过滤的选择性太差（过滤后剩很多行）→ 走暴力扫比走索引+回表更划算 → 动态切换
- 数据量小于 `MAX_HNSW_BRUTE_FORCE_SIZE` → 直接暴力算距离

这是"ANN 索引不一定总是最优"的经典问题的工程解法。

---

## 四、读路径②：全文检索

```
MATCH(coupon_name) AGAINST("...")
  → SQL 层解析为 FTS 求值节点（ObFtsEvalNode）
  → ObIDASTextRetrievalEngine 接口（SQL 层只有接口）
  → 组装根注入的实现（observer/composition/retrieval/）
  → 分词：src/storage/fts/（ik 中文分词 / ngram / 空格）
  → 倒排表扫描 → BM25 打分 → DaaT/BMW 归并取 top-k
```

### 关键模块

**分词器** `src/storage/fts/`：`ob_ik_ft_parser.cpp`（中文 IK 分词）、`ob_ngram_ft_parser.cpp`、`ob_whitespace_ft_parser.cpp`，统一实现 `ob_fts_parser.h` 接口。

**BM25 检索引擎** `src/storage/retrieval/`（33 个文件，是检索算法最密集的地方）：

| 文件 | 算法 |
|---|---|
| `ob_sparse_daat_iter` | Document-at-a-Time：逐文档打分，精确 |
| `ob_sparse_taat_iter` | Term-at-a-Time：逐词项累积分数 |
| `ob_sparse_bmw_iter` + `ob_block_max_iter` | Block-Max WAND：跳过不可能进 top-k 的文档，高性能 top-k |
| `ob_text_daat_iter` / `ob_text_taat_iter` | 文本模式的具体实现 |
| `ob_spiv_daat_iter` / `ob_spiv_dim_iter` | 稀疏向量（sparse vector）模式的实现 |

这些迭代器的统一接口在 `ob_i_sparse_retrieval_iter.h`：`get_next_row / get_next_batch / advance_to / get_dim_max_score`，`advance_to` 和 `get_dim_max_score` 就是为 WAND/MaxScore 动态剪枝准备的。

---

## 五、混合融合层：ObRetrievalProgram

`src/data_plane/api/data_plane/retrieval/ob_retrieval_program.h` 是一个刻意设计的通用"检索程序"抽象：先 `compile` 请求（声明要哪些维度：向量维度、文本维度、标量过滤），再 `run` 输出归并后的 match 流。`ObSparseRetrievalCursor`（`ob_sparse_retrieval.h`）定义了 source → cursor 的流式接口，支持多源 DaaT 归并 + top-k 堆。

这层就是"向量 + 全文 + 标量统一在一个执行计划里"的融合点——各路检索产出统一的 `ObRetrievalDocId`（注意它的设计：文档 ID 以 datum 形式跨界，**不假设整数表示**，这是接口防腐的细节）。

---

## 六、写路径：为什么 P99 平

```
INSERT/UPDATE (DML 提交即返回，不等索引)
  → redo log
  → Change Stream 流水线（src/observer/change_stream/）
      fetcher 取行 → dispatcher 分发 → worker 处理
  → ObCSPluginAsyncIndex::process(rows)   ← 挂接点
  → 更新内存中的增量 vsag HNSW（VIRT_INC）+ vbitmap 记录 vid
  → 后台任务（observer/vector_index/*）定期把增量 merge 进快照索引（VIRT_SNAP）
```

查询侧同时读增量 + 快照（`inner_get_next_row` 里 `adaptor_vid_iter_` 统一吐行），所以写完立即可查，又不用在写事务里同步构建索引。

带 `model=ob_embed` 的混合索引还有一条额外的异步链：`ob_hybrid_vector_refresh_task.h` 里的 `ObVecEmbeddingAsyncTaskExecutor`，状态机 `PREPARE_EMBEDDING → WAITING_EMBEDDING → FINISH`，调 AI 服务生成向量后再进索引管道。

---

## 七、推荐阅读顺序（带验收标准）

1. **SQL 语义**（半天）：跑读 `vector_similarity.test`、`simple_query.test`，回答"approx limit 和普通 limit 的区别是什么" ✔
2. **DAS 迭代器**（1-2 天）：精读 `ob_das_hnsw_scan_iter.h` 的 param 结构和 `inner_get_next_row`，回答"一次 HNSW 查询要访问哪几张内部表" ✔（提示：增量表、快照表、vbitmap、vid↔rowkey 映射表）
3. **适配器**（2-3 天）：`ob_vector_index_adaptor.h` 中的 `ObPluginVectorIndexAdaptor`，重点看 `vsag_query_vids`、`complete_index_mem_data_incremental`、`update_incr_bitmap`，回答"增量转快照时查询怎么保证不丢不重" ✔
4. **FTS/BM25**（2-3 天）：`ob_i_sparse_retrieval_iter.h` → `ob_sparse_bmw_iter.cpp`，回答"BMW 怎么用 block max 上界剪枝" ✔
5. **异步写路径**（2 天）：`ob_cs_plugin_async_index.cpp`，回答"为什么 async 模式可以没有 delta buffer 表" ✔（提示：`skip_delta_buffer_` 的注释）

