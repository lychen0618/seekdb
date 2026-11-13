# HNSW 索引实际创建流程（ObPluginVectorIndexAdaptor 精读）

本文承接《hnsw-index-build-flow.md》，回答一个问题：**HNSW 图到底在哪里、被谁、怎么真正创建出来**。
行号基于 v1.4.0，均经读码验证。涉及的三个核心文件：

| 简称 | 路径 |
|---|---|
| adaptor cpp | `src/observer/vector_index/ob_plugin_vector_index_adaptor.cpp` |
| adaptor h | `src/query/api/query/vector/ob_vector_index_adaptor.h` |
| vsag 封装 | `src/oblib/lib/vector/ob_vector_util.cpp` / `ob_vsag_adaptor.cpp` |

先给结论：

> **DDL 任务不建图**（见前篇）。真正创建 HNSW 图的入口是
> `ObPluginVectorIndexAdaptor::init_mem_data`（adaptor cpp:789），
> 它**不被 DDL 调用，而是被三个场景惰性触发**，每次经
> `try_init_mem_data`（adaptor h:753，原子检查 `is_inited` + 写锁，并发下只建一次）进入，
> 最终调到 vsag 库的 `vsag::Factory::CreateIndex` 造出一个**空图**，再由后续路径灌数据。

---

## 一、全局链路总览

```
ObPluginVectorIndexAdaptor::init_mem_data (:789)
  └─ obvectorutil::create_index            (ob_vector_util.cpp:81，薄封装)
       └─ obvsag::create_index             (ob_vsag_adaptor.cpp:810)
            ├─ construct_vsag_create_param (:593)   ← 拼 vsag JSON 参数
            └─ vsag::Factory::CreateIndex  (:841)   ← 真正构造 HNSW 索引对象
                 └─ 包装成 HnswIndexHandler (:845)  ← OB 侧句柄
```

三个触发场景（都先过 `try_init_mem_data`）：

| 场景 | 调用点 | 建哪个图 |
|---|---|---|
| ① 首次查询（sync/async 模式） | `ob_das_hnsw_scan_iter.cpp:1464 query_result` 驱动 | VIRT_INC 增量图 |
| ② async 模式 Change Stream 写入 | `ob_cs_plugin_async_index.cpp:1118 try_init_mem_data(VIRT_INC)` | VIRT_INC 增量图 |
| ③ 后台定时任务全量构建 | `ob_vector_index_async_task_util.cpp:1478/1529 add_snap_index` → adaptor cpp:1786 | VIRT_SNAP 快照图 |

---

## 二、前置：adaptor 从哪来、参数从哪来

查询迭代器第一次用到向量索引时（`ob_das_hnsw_scan_iter.cpp:934`）：

```
ObDASHNSWScanIter
  └─ ObPluginVectorIndexService::acquire_adapter_guard   (service cpp:549)
       └─ ObPluginVectorIndexMgr::get_and_merge_adapter  (:557)
```

mgr 维护两张 map：**partial adapter**（每个辅表 tablet 一个"半成品"，`:609 create_partial_adapter` 创建）
和 **complete adapter**。查询触发合并：`check_and_merge_partial_inner`（`:635`）按
`data_tablet_id` 把同一索引的 inc / vbitmap / snap / embedded 四个 partial adaptor 归类
（`:680-705`），凑齐后 `:719 replace_with_complete_adapter` 换成完整 adaptor。

**参数来源**：建索引时 `check_vec_index_param` 写进 schema 的 `index_params` 字符串，
在 `set_param`（adaptor cpp:595）里经 `parser_params_from_string`（`:609`）解析成
`ObVectorIndexParam`（`type_ / dist_algorithm_ / m_ / ef_construction_ / ef_search_ / dim_` 等）。
DDL 阶段唯一留给未来建图的东西就是这串参数。

---

## 三、核心创建逻辑：`init_mem_data`（adaptor cpp:789）

一个 adaptor 持有三块内存数据（`init`，`:555/:574` 时分配骨架）：
`incr_data_`（增量图 + insert 位图）、`vbitmap_data_`（两个位图）、`snap_data_`（快照图）。
`init_mem_data` 按 type 分三个分支：

### 分支一：VIRT_INC（:796）——增量 HNSW 图

```cpp
TCWLockGuard lock_guard(incr_data_->mem_data_rwlock_);      // :797 写锁
if (!incr_data_->is_inited()) {                              // :798 double-check
  incr_data_->mem_ctx_->init(parent_mem_ctx_, all_vsag_use_mem_);  // :799 内存上下文
  // 稀疏向量（IPIVF）走单独分支 :800；普通分支：
  build_type = enforce_type == VIAT_MAX ? param->type_ : enforce_type;   // :814
  build_type = build_type==VIAT_HNSW_SQ || VIAT_HNSW_BQ
                 ? VIAT_HGRAPH : build_type;                             // :816
  obvectorutil::create_index(incr_data_->index_, build_type, "float32",
      VEC_INDEX_ALGTH[param->dist_algorithm_],               // :822 距离→"l2"/"cosine"...
      param->dim_, param->m_, param->ef_construction_, param->ef_search_,
      incr_data_->mem_ctx_, param->extra_info_actual_size_, ...)  // :819-831
  // 分配 roaring insert_bitmap :835-841；置 is_inited :848
}
```

两个关键设计：

- **`:816` SQ/BQ 量化索引的增量图必须用 HGRAPH 建**（注释原话 "sq/bq must use hgraph to
  build incr index"）。量化结构适合批量构建、不适合逐条插入——实时性由 fp32 的 hgraph 顶住，
  量化版快照图留给后台批量任务（`try_init_snap_data`，`:4241`，按序列化数据里的实际类型
  决定建 HNSW 还是 HGRAPH/SQ/BQ）。
- 传给 vsag 的 allocator 是 `ObVsagMemContext`——vsag 所有分配都记账到 `all_vsag_use_mem_`
  （即 `vector_memory_limit` 租户上限），这是图内存受控的关键。

失败路径：`:853-858` `free_memdata_resource` + 析构 mem_ctx，不留半初始化状态。

### 分支二：VIRT_BITMAP（:860）——不建图

只创建两个 roaring64 位图（insert/delete），给 vbitmap 语义用，与 vsag 无关。

### 分支三：VIRT_SNAP（:890）——快照 HNSW 图

结构与分支一相同（`:910-922` 同样调 `create_index`），多一个细节：`:896` HNSW_SQ 时
`build_metric` 会被 `get_hnswsq_type_metric` 换算。无锁版本 `init_snap_data_without_lock`
（`:941`）供后台任务在外圈持锁时使用；SQ/BQ 还有专门的数组式内存结构
`init_hnswsq_mem_data`（`:997`，vid_array + vec_array，先攒数组再批量建图 `build_hnswsq_index` :1895）。

---

## 四、最底层：vsag 索引对象怎么造出来

`ob_vsag_adaptor.cpp:810 obvsag::create_index` 三步：

1. **`adjust_create_index_max_degree`**（`:831`，实现在 `:84-87`）：hgraph 的 `max_degree`
   要 **×2 才能对齐 hnsw 的召回率**——注释原话。
2. **`construct_vsag_create_param`**（`:593`）把参数拼成 JSON（模板见 `:565-592` 注释），
   普通 HNSW 长这样：

   ```json
   {"dim":3,"dtype":"float32","metric_type":"l2","use_old_serial_format":true,
    "hnsw":{"ef_construction":200,"ef_search":64,"max_degree":16}}
   ```

   `use_old_serial_format:true`（`:638`）是兼容性妥协——`:621-622` 注释说明 OB 的
   `ObIStreamBuf` 回调式流不支持 seek 到 footer，而 vsag 新序列化格式需要 footer seek，
   所以锁死旧格式。hgraph 类型则带 `base_quantization_type`（fp32/sq8/rabitq，`:604-620`）
   和 `build_thread_count:0`（`:659-662`）。
3. **`vsag::Factory::CreateIndex(index_type_str, json, vsag_allocator)`**（`:841`）返回
   `shared_ptr<vsag::Index>`，再包一层 `HnswIndexHandler`（`:845`）作为 OB 侧句柄。

到这里"创建"完成——但得到的是**空图**，HNSW 的邻接表、分层结构里还没有任何节点。

---

## 五、空图之后：数据怎么进图（三条路径）

| 路径 | 代码 | vsag 调用 |
|---|---|---|
| 查询时增量加载（sync 模式） | `complete_delta_buffer_table_data`（:2321）扫 delta buffer 表 → `write_into_delta_mem`（:2254） | `add_index` 逐批插入增量图 |
| async 模式实时写入 | Change Stream `write_to_vsag_`（ob_cs_plugin_async_index.cpp:1044） | `add_index` 进增量图 |
| 后台全量构建快照 | 定时器（scheduler cpp:59 注册，周期 `VEC_INDEX_SCHEDULAR_BASIC_PERIOD`）→ `run_task`（:1061）→ `check_and_execute_tasks`（:1093）→ 异步任务 `OB_VECTOR_ASYNC_INDEX_BUILT`（async_task_util.h:75）→ `add_snap_index`（util cpp :1478/:1529，adaptor :1766） | 扫主表攒批向量+vid → `obvectorutil::add_index`（:1809）分批插入快照图 → `serialize`（:3272，util :1732 调用）序列化写回 **snapshot_data 表** |

`add_snap_index`（:1766）内部：HNSW/HGRAPH/IPIVF 走 `try_init_mem_data(VIRT_SNAP)`（:1786）
+ 逐批 `add_index`（:1809）；SQ/BQ 走 `init_hnswsq_mem_data` 攒数组（:1818），已有图时直接
`add_index`（:1831-1835），攒够了再 `build_hnswsq_index` 一次性建。

---

## 六、快照的加载：查询侧怎么用建好的图

后台构建完成后，后续查询不再重建图：

```
query_result (:4107)
  ├─ :4140 need_load_data_from_table 判定（非 leader / PVQP_SECOND 需要从表加载）
  ├─ :4164 snapshot_key_prefix 前缀匹配：命中且 !rb_flag_ → 已加载过，跳过
  └─ deserialize_snap_data (:4205)
       ├─ :4229 并发去重（prefix 匹配 + rb_flag 检查）
       ├─ :4231 流式反序列化进 snap_data_->index_
       ├─ :4232 immutable_optimize   ← 只读优化（不可变图可做布局/剪枝优化）
       └─ set_snapshot_key_prefix 记录本次加载的前缀
```

这是"**增量小图 + 批量不可变大图**"双图架构的收益点：快照图建完转 immutable，
查询时反序列化 + 优化一次，长期复用。

---

## 七、设计动机小结

| 设计 | 动机 |
|---|---|
| DDL 不建图、图惰性构建 | HNSW 图是非事务化内存结构；辅表是行存表可事务回填。图永远能从辅表/snapshot 表重建 → 崩溃安全，DDL 提交即返回 |
| 双图（incr + snap）+ 位图合并 | 增量图小、接受逐条插入；快照图批量建完转 immutable（`immutable_optimize`）换查询性能 |
| SQ/BQ 增量图强制 HGRAPH（:816） | 量化结构不适合增量插入，实时性由 fp32 hgraph 兜底 |
| 全部分配走 `ObVsagMemContext` | vsag 内存纳入 `vector_memory_limit` 记账，可观测可淘汰 |
| `use_old_serial_format` 锁死（:638） | OB 流式 buf 不支持 footer seek（:621 注释） |
| hgraph `max_degree` ×2（:84） | 对齐 hnsw 召回率 |
| `try_init_mem_data` 原子检查 + 写锁 | 并发首查只建一次图 |

---

## 八、一页总结

| 问题 | 答案 |
|---|---|
| HNSW 图真正的创建入口？ | `ObPluginVectorIndexAdaptor::init_mem_data`（:789）→ `obvectorutil::create_index` → `obvsag::create_index` → `vsag::Factory::CreateIndex` |
| 谁触发它？ | 首次查询 / Change Stream 异步写 / 后台定时任务（`OB_VECTOR_ASYNC_INDEX_BUILT`），全部经 `try_init_mem_data` 惰性进入 |
| 建图参数从哪来？ | DDL 时写进 schema 的 `index_params` 字符串，`set_param`（:595）解析成 `ObVectorIndexParam` |
| 建完图数据怎么进？ | sync：查询时扫 delta buffer 灌增量图；async：Change Stream 直写增量图；后台任务扫主表批量灌快照图并序列化落 snapshot_data 表 |
| 重启后怎么恢复？ | 不重建——从 snapshot_data 表 `deserialize_snap_data`（:4205）流式加载 + `immutable_optimize` |
| 一句话 | CREATE INDEX 落下的只是 schema + 参数串 + 五张辅表；第一次有人需要这个索引时才真正建 vsag 图，后台任务把图序列化回 snapshot_data 表完成闭环 |
