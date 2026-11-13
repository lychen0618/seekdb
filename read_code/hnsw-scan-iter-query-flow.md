# HNSW 查询执行核心：ObDASHNSWScanIter 精读

本文是阅读指南（vector-retrieval-reading-guide.md）第 2 步的落地，回答四个问题：

1. 一条 `ORDER BY l2_distance(...) APPROX LIMIT` 在 DAS 层的**完整执行流**是什么？
2. 一次 HNSW 查询**实际访问哪几张内部表**、各在什么时机？
3. 增量图（VIRT_INC）+ 快照图（VIRT_SNAP）两套 vsag 索引的结果**怎么合并不丢不重**？
4. 标量过滤（pre / iterative / post filter）的**自适应切换**怎么实现？

行号基于 v1.4.0，均经 grep 验证。文件：`src/sql/das/iter/ob_das_hnsw_scan_iter.h`（701 行）/ `.cpp`（3815 行）。

---

## 一、总览：一次查询的调用栈

```
ObDASHNSWScanIter::inner_get_next_row            .cpp:565
  └─ (首次调用，adaptor_vid_iter_==null 时) process_adaptor_state      :869
       └─ inner_process_adaptor_state                                :901
            └─ process_adaptor_state_hnsw                             :915
                 ├─ acquire_adapter_guard(index_ctx, adaptor_guard)    :934   ← 拿内存图适配器
                 ├─ set_vector_query_condition(query_cond_)            :940   ← 组装查询条件
                 ├─ RWLock::RLockGuard(adaptor->get_query_lock())      :948   ← 与后台刷新互斥
                 ├─ [pre filter]  process_adaptor_state_pre_filter     :950
                 └─ [post filter] process_adaptor_state_post_filter    :954   ← 默认/无过滤也走这
  └─ adaptor_vid_iter_->get_next_row(row, result_output_)              :583   ← 后续行直接从结果迭代器取
```

**惰性执行**：真正的图搜索发生在第一次 `get_next_row`，`do_table_scan()`（:56）只驱动
pre-filter 模式下的二级索引扫描（`inv_idx_scan_iter_`）。查询结果一次性算出 top-k 存进
`adaptor_vid_iter_`（`ObVectorQueryVidIterator`，定义在
`src/query/api/query/vector/ob_vector_query_result.h:76`），后续行只是迭代它。

**输出行格式**（`ObAdaptorIterRowIdx`，头文件 :45）：

```
cell(0)=distance  cell(1)=vid  cell(2..)=extra_info(主键列/分区键)
```

extra_info 是"把主键冗余进索引"的机制（`extra_column_count_`）：有它时 vid→主键不用回表，
直接从 extra_info 还原 rowkey（`build_rowkey_obj_from_extra_info` :3259）。

---

## 二、五个关键成员（读懂参数结构 = 读懂设计）

### 1. 每张辅表一组扫描资源

迭代器成员是"三件套"模式：`xxx_iter_` + `xxx_scan_param_` + `xxx_iter_first_scan_`。
首次 `do_table_scan`，之后 `rescan` 复用（`do_aux_table_scan_need_reuse` :3641）。

| tablet_id 成员 | 对应表 | 扫描函数 |
|---|---|---|
| `delta_buf_tablet_id_` | delta buffer 表 | `do_delta_buf_table_scan` :3517 |
| `index_id_tablet_id_` | index_id 表 | `do_index_id_table_scan` :3530 |
| `snapshot_tablet_id_` | snapshot_data 表 | `do_snapshot_table_scan` :3543 |
| `com_aux_vec_tablet_id_` | 主表（向量列数据源+过滤回表） | `do_com_aux_vec_table_scan` :3556 |
| `vid_rowkey_tablet_id_` | vid_rowkey 表 | `do_vid_rowkey_table_scan` :3598 |
| `rowkey_vid_tablet_id_` | rowkey_vid 表 | `do_rowkey_vid_table_scan` :3608 |
| `embedded_tablet_id_` | embedded 表（hybrid 索引） | `do_embedded_table_scan` :3584 |

tablet id 由上层 `set_related_tablet_ids()`（头文件 :326）一次注入。

### 2. 两级路径决策：`vec_index_type_` × `vec_idx_try_path_`

```cpp
// src/query/api/query/vector/ob_vector_index_util.h:149
enum ObVecIndexType {              // 计划期决定（优化器）
  VEC_INDEX_POST_WITHOUT_FILTER=1, // 无过滤，最简单路径
  VEC_INDEX_PRE=2,                 // 计划期就确定 pre filter
  VEC_INDEX_POST_ITERATIVE_FILTER=3,// 计划期确定迭代过滤
  VEC_INDEX_ADAPTIVE_SCAN=4        // 自适应：运行期再定，可能中途 retry 换路
};
// :159 计划期"建议"或运行期(含 retry)实际走法
enum ObVecIdxAdaTryPath { VEC_INDEX_PRE_FILTER=1, VEC_INDEX_ITERATIVE_FILTER=2,
                          VEC_INDEX_IN_FILTER=3, VEC_INDEX_POST_FILTER=4 };
```

判定辅助函数（头文件 :458-475）：`is_pre_filter() / is_in_filter() / is_post_filter() /
is_iter_filter()`——ADAPTIVE 类型要看 try_path 才知道实际走哪条。

### 3. `query_cond_`（ObVectorQueryConditions，定义在 ob_vector_index_adaptor.h:339）

传给 adaptor 的查询请求：`query_vector_`（查询向量）、`ef_search_`、`query_limit_`（会动态放大）、
`query_scn_`（读快照版本，取自 snapshot_scan_param :3143）、`distance_threshold_`（距离上限过滤）、
`is_last_search_`（迭代终止预告）、`row_iter_`（snapshot 表的行迭代器，供 adaptor 惰性加载快照图）。

### 4. `adaptive_ctx_`（ObHnswAadaptiveCtx，头文件 :178）

运行期统计：过滤扫描行数 / 结果行数 → 选择性，供 retry 判断和路径学习。

### 5. `ObSimpleMaxHeap`（头文件 :650）

距离最大堆，元素 `(vid, distance, is_snap)`——**is_snap 标记是两套图合并去重的基础**。

关键常量（头文件 :498-508）：

| 常量 | 值 | 用途 |
|---|---|---|
| `MAX_VSAG_QUERY_RES_SIZE` | 16384 | limit+offset 上限（inner_init :208 校验报错） |
| `VSAG_MAX_EF_SEARCH` | 160000 | ef_search 放大上限 |
| `FIXED_MAGNIFICATION_RATIO` | 2.0 | 迭代首轮 limit 放大倍数 |
| `FIXED_MAGNIFICATION_RATIO_EACH_ITERATIVE` | 10.0 | 每轮迭代放大上限 |
| `MAX_HNSW_BRUTE_FORCE_SIZE` | 20000 | pre filter 候选 ≤ 此数转暴力 |
| `CHANGE_PATH_WINDOW_SIZE` | 30 | 路径统计窗口 |
| `DECAY_FACTOR` | 0.5 | 窗口统计衰减 |

---

## 三、默认路径：post filter 的五状态机（必读）

`process_adaptor_state_post_filter_once`（:2820）驱动一个显式状态机
（`ObVidAdaLookupStatus`，头文件 :33）。每个状态做两件事：**SQL 侧扫辅表**（`prepare_state` :2891）
+ **adaptor 侧吃数据**（`call_pva_interface` :2934），转移由 adaptor 返回的
`PluginVectorQueryResStatus`（ob_vector_index_adaptor.h:115）决定（`next_state` :3007）。

```
STATES_INIT ──扫 delta buffer 表──► adaptor 增量位图判断
   │ PVQ_LACK_SCN(缺版本)          │ PVQ_OK(增量已齐)
   ▼                               ▼
QUERY_INDEX_ID_TBL ──扫 index_id 表──► 重建/更新 vbitmap
   │ PVQ_COM_DATA(有数据待进图)
   ▼
QUERY_ROWKEY_VEC ──SQL侧逐vid取向量──► complete_delta_buffer_table_data
   │ (PVQ_COM_DATA 自循环分批, PVQ_LACK_SCN)
   ▼
QUERY_SNAPSHOT_TBL ──扫 snapshot 表──► 快照图加载检查
   │ PVQ_OK
   ▼
STATES_SET_RESULT ── query_result(): 查增量图+快照图,堆合并 ──► adaptor_vid_iter_
   │ PVQ_REFRESH(后台正在刷新,数据陈旧)
   ▼
STATES_REFRESH ── query_need_refresh_memdata() 等刷新 ──► FINISH
```

### 各状态明细

| 状态 | prepare_state（SQL 侧） | call_pva_interface（adaptor 侧） |
|---|---|---|
| `STATES_INIT` :2896 | `do_delta_buf_table_scan` :2901（整表 range :3654） | `check_delta_buffer_table_readnext_status` :2947：把 delta buffer 行喂给 adaptor，与内存增量图已加载数据比对版本 |
| `QUERY_INDEX_ID_TBL` :2910 | `do_index_id_table_scan` :2911 | `check_index_id_table_readnext_status` :2964：async 模式下用 index_id 表行重建 vbitmap（记录哪些 vid 在增量图/被删） |
| `QUERY_ROWKEY_VEC` :2905 | `prepare_complete_vector_data` :2906（见下） | `complete_delta_buffer_table_data` :2955：把向量写进**内存增量图**（不写任何表） |
| `QUERY_SNAPSHOT_TBL` :2915 | `do_snapshot_table_scan` :2916 | `check_snapshot_table_wait_status` :2969：检查快照图是否需要从 snapshot 表行重新加载 |
| `STATES_SET_RESULT` :2973 | （row_iter_ 挂给 query_cond_ :2974） | `query_result` :2977：真正调 vsag 搜两套图、合并、产出迭代器 |
| `STATES_REFRESH` :2981 | — | `query_need_refresh_memdata` :2990 |

两个 PVQ 状态的语义值得记住：

- **PVQ_WAIT**（:3012, :3039）：保持当前状态再循环一轮——等后台线程把数据加载完。
- **PVQ_LACK_SCN**：内存图版本落后于查询快照 → 去扫对应辅表补。
- `QUERY_ROWKEY_VEC` 的 PVQ_COM_DATA **自循环**（:3030-3031）是分批实现：`get_vec_cnt()`
  每批最多 `VI_PARAM_DATA_BATCH_SIZE=1000`（ob_vector_index_adaptor.h:143），`do_next_batch()` 推进游标。

### `prepare_complete_vector_data`（:3302）——增量数据进图的最后一公里

```
对 ada_ctx 里每个待进图的 vid：
  vid ──vid_rowkey 表点查──► 主键 rowkey（get_rowkey_from_vid_rowkey_table :3396）
  rowkey ──主表点查──► 向量字节（get_vector_from_com_aux_vec_table :3446）
                    或 embedded 表（hybrid 索引, get_vector_from_embedded_table :3419,
                      rowkey=主表rowkey+vid 拼接 :3428）
  ada_ctx.set_vector(i, ptr, len)   ← adaptor 随后写进内存增量图
```

查询中查不到（OB_ITER_END）→ `set_vector(i, nullptr, 0)` 跳过（:3350-3352），不报错——**容忍
并发删除**，宁可漏不可错。

### 三个"跳过"优化（查询到底要不要扫辅表）

1. **async 模式**（`skip_delta_buffer_`，:2833）：不扫 delta buffer 表（DML 根本不写它），
   直接从 `QUERY_INDEX_ID_TBL` 开始重建 vbitmap。
2. **快照图已加载且新鲜**（`adaptor->get_can_skip()==SKIP` 且 `!rb_flag_`，:2842-2855）：
   跳过全部表扫描，直接 `STATES_SET_RESULT` 查图。**这是热查询的快路径。**
3. **follower 副本**：`prepare_follower_query_cond`（:3286）——非 leader 上 snapshot 迭代器
   换成本地 tablet 的（`snapshot_scan_param_.tablet_id_ != snapshot_tablet_id_` 判断），
   因为 follower 也有全量辅表数据但没有最新内存图。

---

## 四、结果物化：`adaptor_vid_iter_` → expr datum

`inner_get_next_row`（:565）/`inner_get_next_rows`（:627）把迭代器行写进 `result_output_`
表达式的 datum。一个精彩的优化在 `save_distance_expr_result`（:537/:692）：

> `need_save_distance_result()`（头文件 :358）为真时，把 top-k 里**已经算好的距离**
> 直接写进 `distance_calc_` 表达式（ORDER BY l2_distance 那个表达式）的 datum 并标记
> evaluated+projected——上层算子不再重复计算距离。行批量被限制在
> `MAX_OPTIMIZE_BATCH_COUNT=16`（`adjust_batch_count` :484）也是为这个优化服务的。

例外：hybrid + hnsw_bq（标量量化）索引的 vsag 距离是量化距离，需要精算——
`calc_dis_by_vid`（:497）按 vid 回表取原始向量，用 `ObExprVectorDistance::DisFunc` 手工算精确距离。

---

## 五、迭代过滤（iterative filter）：放大-重试循环

`process_adaptor_state_post_filter`（:2335）。适用：ADAPTIVE 选了 ITERATIVE 或计划期定为
POST_ITERATIVE。思想：**先按 2 倍 limit 多取些候选，逐个回表过滤；不够就放大 ef/limit 再搜一轮**。

```
首轮: limit 放大 2 倍, ef=max(ef, limit)                    :2346-2348
while (!end_search):
  process_adaptor_state_post_filter_once   ← 首轮: 完整状态机 + query_result → tmp iter
    或 adaptor->query_next_result          ← 续轮: vsag 继续搜,增量结果 → tmp iter :2359
  post_query_vid_with_filter               ← 对 tmp iter 的候选逐个过滤 :2370
```

### `post_query_vid_with_filter`（:2693）逐候选过滤

对 `tmp_adaptor_vid_iter_` 的每个 (vid, distance)：

- **simple_cmp 快速路径**（:2744）：过滤条件是"单列 rowkey 简单比较"（`check_is_simple_cmp_filter`
  :2292 识别，注释 :2287 列了三个条件）→ 直接拿 extra_info 里的主键值比较，**不回表**。
- 否则 **`filter_by_index_back`**（:2637）：
  `set_rowkey_by_vid`（:2378）把 vid 翻成 rowkey（有 extra_info 直接还原 :2419；
  没有则点查 vid_rowkey 表 :2402）→ `data_filter_iter_` 按主键点查主表 →
  `get_single_row_from_data_filter_iter`（:2871）在存储层应用 pushdown filter。
- 过滤通过 → `adaptor_vid_iter_->add_result(vid, distance, extra_info)`（:2754）。

### 放大策略（:2778-2800）

```
need_cnt_next = 还差多少个
select_ratio  = added_cnt / unfiltered_vid_cnt   (本轮实测选择性)
added_cnt==0  → new_limit = old_ef * 2           (选择性为 0, 直接放大)
can_be_last_search(old_ef, need_cnt, select_ratio)（:2273）:
    select_ratio >= 0.05 且 expect_get*2 <= old_ef → 最后一轮按原 ef 搜
否则  new_limit = min(ceil(need/select_ratio), limit*10)   (每轮最多放大10倍)
ef_search = max(ef, new_limit), 上限 160000
```

终止条件：候选总数 < query_limit（搜完了，:2760）；或累计扫描向量数超过
`_hnsw_max_scan_vectors`（默认 **20000**，ob_parameter_seed.ipp:1154，:2354 读取，:2761 判断）。
hnsw_bq 需要精算 reorder，所以 limit 会被抬到 `get_reorder_count`（:3108，与 ef/refine_k 相关）。

---

## 六、pre filter：先把"哪些 vid 合法"算成位图，再搜图

`process_adaptor_state_pre_filter`（:1424）。思想：**过滤选择性好的时候，先扫过滤索引拿到
合法 vid 集合，塞进 adaptor 的 roaring bitmap，让 vsag 搜索时跳过非法节点**（ob_hnsw_bitmap_filter）。

两条取 vid 的路：

| 场景 | 函数 | 数据源 |
|---|---|---|
| 主键过滤（`is_primary_pre_with_rowkey_with_filter_`） | `..._with_rowkey` :1560 | 扫 **rowkey_vid 表**（主键范围 → vid，`do_rowkey_vid_table_scan`） |
| 二级索引过滤 | `..._with_idx_filter` :2228 | 扫 `inv_idx_scan_iter_`（普通二级索引）→ 主键 → `get_vid_from_idx_filter` :1897 / `get_pk_increment_from_idx_filter` :2092 |

初始化：`init_prefilter`（:1522）——rowkey 模式带主键范围（:1527）；否则
`adaptor->get_vid_bound()` 拿 [min_vid, max_vid] 建位图（:1552）。

### 暴力短路（go_brute_force_）

扫过滤索引时同步攒 `vids[]`（:1578 分配 MAX_HNSW_BRUTE_FORCE_SIZE 个）：

- 候选总数 ≤ 20000 → `go_brute_force_=true`，**不搜图**，直接对这批 vid 调
  `vsag_query_vids` 算距离（见下节）。
- 超了 → 停止攒，把已攒的塞进 bitmap（:1611-1617），继续走图搜索。
- ipivf（稀疏向量）不支持按 id 算距离，强制 `go_brute_force_=false`（:1438）。

### 与 post filter 的兜底交互（:1441-1451）

```
brute force 中若发现 need_complete_data（内存图里缺某个 vid 的数据，距离返回 -1）:
  query_cond_.only_complete_data_ = true
  → process_adaptor_state_post_filter   ← 先跑一遍完整状态机补数据（含 complete_delta_buffer_table_data）
  → 再 brute force（此时 check_need_complete_data=false, 不再递归）
```

另一分支（:1452）：`check_if_complete_data` 判断增量数据没齐 → 同样先走
`process_adaptor_state_post_filter_once` 补数据。

---

## 七、两套图的暴力合并：不丢不重的机制

`vsag_query_vids(vector, vids, cnt, &distances, is_snap)`（adaptor 侧声明
ob_vector_index_adaptor.h:637）按 `is_snap` 分别查增量图/快照图，返回**等长距离数组**，
`-1` 表示该图里没有这个 vid。

**非 bq 路径**（`..._brute_force_not_bq` :1054，两调用 :1079-1080）：

```cpp
for (i in vids):
  bool is_snap = distances_inc[i] == -1;              // :1096 增量图没有 → 必是快照的
  double distance = distances_inc[i] == -1 ? distances_snap[i] : distances_inc[i];
  if (distance != -1 && distance <= distance_threshold_)   // :1088 双保险
    max_heap.push(vid, distance, is_snap);
```

要点：**同一个 vid 在两套图都存在时（后台刷新瞬间），优先取增量图距离**——增量图是更新
的一方，快照图是旧版本；is_snap 标记让后续 extra_info 查找（`get_extra_info_by_vids` :1005，
分 snap/inc 两批 :1023/:1027）去对应的内存结构取。

**bq 路径**（:1238 `merge_and_sort_brute_force_results_bq`）：量化距离需要精算，两图各建一个堆
（snap_heap/incr_heap），`build_brute_force_result_iterator_bq`（:1280）合并时用
`added_vids` hashmap 去重（:1314-1345），**增量优先**（先放 incr :1317，snap 里已存在的丢弃 :1339）。

去重语义回答了阅读指南 step 3 的一半问题：**增量 vid 在快照图里的旧副本被增量版本覆盖，
删除由 vbitmap 记录、扫辅表重建——查询看到的是"增量优先、快照兜底"的单调版本**。

---

## 八、自适应路径切换：retry + 跨执行学习

### 运行期 retry（`process_adaptor_state` :869）

```cpp
can_retry_ = check_if_can_retry();   // 头文件 :473: ADAPTIVE 且当前路径是 PRE/ITERATIVE 且无 relevance 列
inner_process_adaptor_state(...)     // 先按当前路径跑
若返回 OB_VECTOR_INDEX_ADAPTIVE_NEED_RETRY 且 can_retry_:
  reset_filter_path()                // :792 换路: PRE ↔ ITERATIVE 互换（ipivf: PRE→POST :802-803）
  do_table_scan()/rescan() 重扫过滤索引
  inner_process_adaptor_state(...)   // 再跑一遍
```

retry 触发条件：

- **pre filter 侧**（`check_pre_filter_need_retry` :774）：过滤后候选 ≤ 20000 → 不 retry
  （暴力就好）；否则选择性超阈值 → 图搜索不划算了，切走。阈值（ob_vector_index_util.h:317-322）：
  rowkey 路径 `0.35`，二级索引路径 `0.15`，ipivf `0.1`。
- **iterative 侧**（`check_iter_filter_need_retry` :754）：迭代 ≥3 轮后，若
  `iter_selectivity * row_count` 的预估输出 ≤ 100 万（rowkey）/30 万（idx）且选择性 < 同上阈值
  → 说明过滤其实很准，pre filter 更优，切走。
- **强制切换**（`check_need_force_switch_run_path` :734）：iterative 路径 + 并行 block granule
  （多 worker 分片，无法共享迭代状态）→ 直接 retry 成 pre。

### 跨执行学习（`updata_vec_exec_ctx` :833）

每次执行结束把所选路径计入 `plan_stat->vec_index_exec_ctx_`（计数器原子累加 :843-848）。
窗口满 30 次后：三个计数器按 `log(count)*0.5` 衰减（:854-856），并把当前路径写入 `cur_path_`
（:863）。下次执行 `inner_init`（:234-238）读 `cur_path_` 直接预选——**同一 prepare 的语句
会根据历史执行统计自动收敛到更优的过滤路径**。`check_if_can_retry` 排除 relevance 列
（混合检索）是因为换路需要重算 rel_map。

---

## 九、一次 HNSW 查询访问的表（汇总，回答验收问题）

以上默认 sync 模式、无过滤、快照图已加载的最快路径为例：**一张表都不扫，纯内存查图**
（get_can_skip==SKIP 快路径 :2842）。冷路径/其他模式按下表：

| 表 | 访问函数 | 扫描方式 | 何时访问 |
|---|---|---|---|
| delta buffer 表 | `do_delta_buf_table_scan` :3517 | 整表 range | sync 模式 STATES_INIT：把未进图的增量行喂给 adaptor |
| index_id 表 | `do_index_id_table_scan` :3530 | 整表 range | async 模式 / PVQ_LACK_SCN：重建 vbitmap |
| snapshot_data 表 | `do_snapshot_table_scan` :3543 | 整表 range | 快照图内存缺失或版本落后（rb_flag_ / PVQ_LACK_SCN） |
| vid_rowkey 表 | `get_rowkey_from_vid_rowkey_table` :3396 | **逐 vid 点查** | QUERY_ROWKEY_VEC 补数据、无 extra_info 的回表翻译、bq 精算距离 |
| rowkey_vid 表 | `do_rowkey_vid_table_scan` :3608 | 主键范围 | pre filter（主键过滤）：主键→vid；iterative 简单比较路径取 extra_info |
| 主表（com_aux_vec） | `get_vector_from_com_aux_vec_table` :3446 / `filter_by_index_back` :2637 | 逐 rowkey 点查 | 补数据取向量列；iterative filter 回表过滤 |
| embedded 表 | `do_embedded_table_scan` :3584 | 逐 (rowkey+vid) 点查 | 仅 hybrid 索引：取 embedding 后的向量 |

辅表扫描全部是"首扫 do_table_scan + 复用 rescan"（`do_aux_table_scan_need_reuse` :3641），
range 用 `set_whole_range` 全表（:3654）——**辅表很小且无过滤条件，整表扫是对的**。

---

## 十、一页总结

| 问题 | 答案 |
|---|---|
| 查询入口与惰性点？ | `inner_get_next_row` :565 首行触发 `process_adaptor_state_hnsw` :915；结果物化进 `adaptor_vid_iter_` 后续行直取 |
| post filter 状态机？ | INIT→(INDEX_ID)→ROWKEY_VEC→SNAPSHOT→SET_RESULT→(REFRESH)→FINISH；prepare_state 扫表 + call_pva_interface 喂 adaptor，PVQ_* 驱动转移 |
| 增量图数据哪来？ | sync: delta buffer 表；async: index_id 表 + `prepare_complete_vector_data` :3302 逐 vid 回主表取向量，`complete_delta_buffer_table_data` 写内存图 |
| 两图怎么合并？ | 图搜索在 adaptor.query_result 内；暴力路径显式合并：`vsag_query_vids(-1=不存在)` + 增量优先 + is_snap 标记 + bq hashmap 去重 |
| iterative filter？ | limit 放大 2 倍起步、每轮上限 10 倍、ef≤160000；逐候选回表或 simple_cmp；终止=够数或扫 2 万向量 |
| pre filter？ | 过滤索引→vid 集→roaring bitmap 注入 vsag；候选≤2 万直接暴力 |
| 自适应？ | 运行期 retry（选择性阈值 0.35/0.15/0.1）+ plan_stat 30 次窗口 log 衰减学习，跨执行收敛 |
| 快路径在哪？ | `get_can_skip()==SKIP && !rb_flag_` :2842 跳过全部表扫描 |

### 排查锚点

| 现象 | 查哪里 |
|---|---|
| 召回率差 | session 变量 `ob_hnsw_ef_search`（默认 64，:3094）；`_hnsw_max_scan_vectors`（默认 20000）；iterative 的 limit 放大日志 `iteractive filter log` :2764 |
| 首查慢 | 大概率在补数据：`SYCN_DELTA_query_data` 日志 :3363；QUERY_ROWKEY_VEC 自循环逐 vid 点查 vid_rowkey+主表 |
| limit 报错 not supported | `limit+offset > 16384` :208 |
| 想看路径选择历史 | plan_stat 的 `vec_index_exec_ctx_`（计数器/cur_path），日志 `begin to reset plan stat filter path` :857 |
| 距离重复计算优化没生效 | `need_save_distance_result` :358（bq 非 hybrid 才生效）与 `adjust_batch_count` :484 |

下一篇建议：进 adaptor 侧精读 `query_result / check_delta_buffer_table_readnext_status /
complete_delta_buffer_table_data / update_incr_bitmap`（阅读指南 step 3），把本笔记里
"adaptor 侧吃数据"的黑盒打开——特别是 vbitmap 的位图语义和增量转快照的一致性协议。
