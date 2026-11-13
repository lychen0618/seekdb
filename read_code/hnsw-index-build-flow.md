# HNSW 向量索引构建核心流程（从 ObIndexBuilder::create_index 开始）

本文回答两个问题：**核心流程怎么实现**、**每一步是怎么被调用到的**。行号基于 v1.4.0，均经 grep 验证。

先给一个颠覆直觉的结论，后文展开：

> **`ObVecIndexBuildTask`（DDL 任务）本身不构建 HNSW 图。** 它做的是"建辅表 schema + 从主表回填数据到辅表"。
> 真正的 HNSW 图（vsag 内存结构）是在**查询时/后台刷新时惰性构建**的——从辅表加载向量，在内存中建图。

---

## 一、全局流程总览

```
ObIndexBuilder::create_index                    ← ob_index_builder.cpp:52
  └─ do_create_index 分发 (:1497)
       └─ is_vec_index (:1554) → do_create_local_index (:1264)
       ├─ 【同步，DDL 事务内】
       │    ├─ 建主索引 schema（delta_buffer 类型）+ 共享辅表 schema
       │    └─ submit_build_index_task (:607)                    ← 登记任务
       │         └─ ObDDLScheduler::create_ddl_task (:1059)
       │              └─ case DDL_CREATE_VEC_INDEX (:1141)
       │                   └─ create_build_vec_index_task (:1750)
       │                        └─ ObVecIndexBuildTask 构造 (:1765)
       │                             + insert_task_record (:2427) → 写 __all_ddl_task
       ├─ 事务提交 → publish_schema
       └─ schedule_ddl_task (:2557)                             ← 交给调度器
            └─ case DDL_CREATE_VEC_INDEX → schedule_build_vec_index_task (:2707)
                 └─ task_queue_.push_task                       ← 进内存任务队列

【异步，DDLTaskExecutor 线程】
ObDDLScheduler::do_work (:987)          ← 后台线程主循环
  └─ task_queue_.get_next_task → task->process() (:1033)
       └─ ObVecIndexBuildTask::process (:221)                   ← 状态机
            PREPARE → GENERATE/WAIT_ROWKEY_VID → GENERATE/WAIT_VEC_AUX
            → GENERATE/WAIT_VID_ROWKEY → VALIDATE_CHECKSUM → SUCCESS
            每个 GENERATE 状态：
              prepare_aux_table (:573)
                └─ ObDomainIndexBuilderUtil::prepare_aux_table (:48)
                     └─ create_aux_index RPC → 子 DDL 任务（建表+回填数据）

【查询/后台刷新时——HNSW 图真正构建的地方】
ObPluginVectorIndexAdaptor::init_mem_data (:789)
  ├─ VIRT_INC：obvectorutil::create_index(...)   ← 建增量 vsag 图
  └─ VIRT_SNAP (:891)：obvectorutil::create_index(...) ← 建快照 vsag 图
complete_delta_buffer_table_data (:2321)         ← 把 delta buffer 行写进增量图
```

---

## 二、入口：ObIndexBuilder::create_index（同步侧）

位置均在 `src/rootserver/ob_index_builder.cpp`：

| 位置 | 动作 |
|---|---|
| `:52` | `create_index`：校验参数、处理重名 |
| `:1497` | `do_create_index`：按索引类型分发；`:1554` 判断 `is_vec_index` → `do_create_local_index`（向量索引均为 `*_LOCAL` 类型） |
| `:1264` | `do_create_local_index`：开 DDL 事务，真正建索引 schema 的流程 |
| `:1309` | 空表检测：空表直接 `INDEX_STATUS_AVAILABLE`（`:1318`），**不提交任务**（跳过整个异步流程） |
| `:1450` | 非空表：`submit_build_index_task` |
| `:1486` | 事务提交后 `publish_schema` + `schedule_ddl_task` |

`submit_build_index_task`（`:607`）构造 `ObCreateDDLTaskParam`（type=`DDL_CREATE_VEC_INDEX`），
经 `ObSysDDLSchedulerUtil::create_ddl_task` 最终 `insert_task_record`（`ob_ddl_scheduler.cpp:2427`）
把任务写进内部表 `__all_ddl_task` —— **与 schema 变更同一事务，保证原子生效**。

**调用到这里谁在等？** SQL 执行器 `ObCreateIndexExecutor::execute` 发 RPC 后调
`wait_ddl_finish(task_id)`（`ob_index_executor.cpp:120`）阻塞等任务完成——异步任务对客户端是同步的。

---

## 三、任务怎么被调度执行

任务记录入库后，`schedule_ddl_task`（`ob_ddl_scheduler.cpp:2557`）按 `ddl_type_` 分发：

```cpp
case ObDDLType::DDL_CREATE_VEC_INDEX:          // :2592
  ret = schedule_build_vec_index_task(record); // :2707
```

`schedule_build_vec_index_task` 从任务记录还原出 `ObVecIndexBuildTask`，
`inner_schedule_ddl_task`（`:3208`）把它压进 `task_queue_`。

**驱动者是 `ObDDLScheduler::do_work`（`:987`）后台线程**（线程名 `DDLTaskExecutor`）：

```cpp
while (!has_set_stop()) {
  task_queue_.get_next_task(task);   // :1004
  ...
  task->process();                   // :1033 ← 每轮驱动状态机前进一步
}
```

注意 process 是**分步驱动**的：每次调用只推进一个状态，任务随后回到队尾，
按 `calc_next_schedule_ts` 计算的下次时间再调度（WAIT 状态轮询子任务就是这么实现的）。

---

## 四、状态机核心：ObVecIndexBuildTask

`process()`（`src/rootserver/ddl_task/ob_vec_index_build_task.cpp:221`）按 `task_status_` 分发。
三个 GENERATE 状态结构相同，都经由本类的 `prepare_aux_table`（`:573`）→
`ObDomainIndexBuilderUtil::prepare_aux_table`（`src/rootserver/ob_domain_index_builder_util.cpp:48`）。

### 状态 1：PREPARE —— `prepare()` `:536`

只做状态确认，直接 `switch_status` 进入下一状态。

### 状态 2：GENERATE_ROWKEY_VID_SCHEMA —— `prepare_rowkey_vid_table()` `:599`

提交 **rowkey→vid 映射表** 的"建表+回填"子任务：

```cpp
const ObIndexType index_type = INDEX_TYPE_VEC_ROWKEY_VID_LOCAL;
prepare_aux_table(index_type, rowkey_vid_task_submitted_, rowkey_vid_aux_table_id_, rowkey_vid_task_id_);
```

细节：`use_vid_` 为 false 时（不需要 vid 的方案）直接标记 submitted 跳过。

### 状态 3：WAIT_ROWKEY_VID_TABLE_COMPLEMENT —— `wait_aux_table_complement()` `:926`

遍历 `dependent_task_result_map_`（记录所有子任务状态），全部完成才进入下一状态；
任一子任务失败则整体 FAIL。**这就是轮询子任务回填进度的地方。**

### 状态 4：GENERATE_VEC_AUX_SCHEMA —— `prepare_aux_index_tables()` `:648`

提交 **delta buffer 表**（普通向量）或 **hybrid log 表**（带 embedding 的混合索引）的子任务：

```cpp
const ObIndexType domain_table_type = is_post_create_hybrid_vector_ ?
    INDEX_TYPE_HYBRID_INDEX_LOG_LOCAL : INDEX_TYPE_VEC_DELTA_BUFFER_LOCAL;
prepare_aux_table(domain_table_type, delta_buffer_task_submitted_, ...);
// 混合索引额外提交 embedded_vec 表；
// index_id / snapshot_data 共享表已在入口同步阶段建好
```

### 状态 5/6：GENERATE_VID_ROWKEY_SCHEMA `:735` + WAIT

提交 **vid→rowkey 映射表**（回表用）子任务，逻辑同状态 2/3。

### 状态 7：VALIDATE_CHECKSUM —— `validate_checksum()` `:1760`

校验数据一致性（当前实现是 TODO 占位，直接通过）。

### 状态 8：SUCCESS —— `succ()` `:1755` → `cleanup_impl()` `:1800`

上报结果、释放表锁、索引置 AVAILABLE、删任务记录。
失败路径：FAIL → `clean_on_failed()` 删除已建辅表。

---

## 五、子任务链：建表 + 数据回填怎么提交

`ObDomainIndexBuilderUtil::prepare_aux_table`（`ob_domain_index_builder_util.cpp:48`）核心三步：

```cpp
// 1. 构造 ObCreateAuxIndexArg（目标辅表的 create_index_arg + snapshot_version）
// 2. 调 LocalManagementService::create_aux_index（:91，进程内调用）
//    → ObDDLService::create_aux_index (ob_ddl_service.cpp:5350)
//       ├─ 建辅表 schema（事务内）
//       └─ create_aux_index_task_ (:5303)
//            └─ create_ddl_task(DDL_CREATE_INDEX)   ← 子任务
// 3. res.ddl_task_id_ 记进 dependent_task_result_map_，供 WAIT 状态轮询
```

子任务类型是普通 `DDL_CREATE_INDEX`（`ob_ddl_service.cpp:5326`），即**通用建索引任务**：
扫主表存量数据 → 写入目标辅表。对向量索引而言，"回填"就是把每行的向量值、主键
写进 delta buffer / rowkey_vid / vid_rowkey 表。

---

## 六、关键：HNSW 图真正在哪里构建

DDL 任务只负责**把数据搬进普通行存辅表**；vsag HNSW 图是**内存结构，惰性构建**。

### 构图入口：`ObPluginVectorIndexAdaptor::init_mem_data`（`ob_plugin_vector_index_adaptor.cpp:789`）

```cpp
type == VIRT_INC (:796)：
  obvectorutil::create_index(incr_data_->index_, build_type, FLOAT32,
                             距离算法, dim, m, ef_construction, ef_search, ...)
  + 分配 insert_bitmap（记录增量 vid）

type == VIRT_SNAP (:891)：
  obvectorutil::create_index(snap_data_->index_, ...)   ← 快照图
```

`obvectorutil::create_index` 封装 vsag 库（第三方 HNSW 实现），参数全部来自建索引时
存进 schema 的 `index_params_`（distance/type/m/ef_construction 等，DDL 时由
`check_vec_index_param` 写入）。

### 数据进图的时机

| 时机 | 代码 | 说明 |
|---|---|---|
| 查询时（同步模式） | `complete_delta_buffer_table_data`（`:2321`） | 扫描迭代器读 delta buffer 表 → `write_into_delta_mem` 写进增量图，再搜增量图+快照图 |
| 后台刷新 | `observer/vector_index/` 刷新任务 | 定期把增量图数据构建/合并进快照，序列化落盘到 snapshot data 表 |
| 写入时（async 模式） | `ObCSPluginAsyncIndex::write_to_vsag_`（`ob_cs_plugin_async_index.cpp:1044`） | Change Stream 消费 redo，直接把新向量写进内存增量图 |

所以"新建向量索引后的首次查询"实际发生的事：辅表里已有回填好的向量 →
首次查询触发 `init_mem_data` 建空图 → 从 delta buffer 加载数据进图 → 搜索。
图常驻内存（受 `vector_memory_limit` 控制），后续查询复用。

> 构图细节（`init_mem_data` 三分支、vsag 参数构造、数据进图三路径、快照反序列化）
> 见另一篇笔记《hnsw-index-adaptor-create-flow.md》。

---

## 七、一页总结

| 问题 | 答案 |
|---|---|
| 从 create_index 到任务执行的链路？ | `create_index → submit_build_index_task → create_ddl_task → insert_task_record(__all_ddl_task) → schedule_ddl_task → task_queue_ → do_work 线程 → process()` |
| 每个 GENERATE 状态做什么？ | 经 `prepare_aux_table → create_aux_index` 提交子任务：建一张辅表 schema + 通用建索引任务回填数据 |
| WAIT 状态做什么？ | 轮询 `dependent_task_result_map_` 里子任务进度，全完成才推进 |
| HNSW 图在哪构建？ | **DDL 任务不建图**。`init_mem_data` 调 vsag `create_index` 建内存图；查询/刷新/Change Stream 从辅表向图灌数据 |
| 为什么这么设计？ | 图是内存结构、不事务化；辅表是行存表、可事务回填。DDL 只搬数据，图按需构建，崩溃后可从辅表恢复 |
| 索引一直 UNAVAILABLE 查哪里？ | `__all_ddl_task` 看任务状态 → 定位卡在哪个 WAIT → 查对应子任务（同表里的记录） |

---

## 附录：五张辅表对 HNSW 索引的作用

### 总览：五张表在 HNSW 生命周期中的分工

```
                    写入方向
DML(同步) ──────────────► delta buffer 表 ──(查询时加载)──► 内存增量图(VIRT_INC)
Change Stream(异步) ───► index_id 表 ──────(同时)─────────►  内存增量图(VIRT_INC)
                              │                               │ 后台刷新
                              ▼                               ▼
                        (位图重建依据)              snapshot_data 表 ◄── 序列化落盘
                                                          │ 查询时加载
                          rowkey_vid 表 ◄──(vid分配)──     ▼
                              ▲                    内存快照图(VIRT_SNAP)
                              │ 反查
HNSW 搜索输出 vid ──► vid_rowkey 表 ──► 主键 ──► 回主表取行
```

### ① rowkey_vid 表（`INDEX_TYPE_VEC_ROWKEY_VID_LOCAL`）

**作用：主键 → vid 的正向映射表（vid 分配器）。**

- vid（vector id）是 HNSW 图中节点的数字身份证。主表的向量列本身没有这种数字 ID，
  建向量索引时系统为每行向量分配一个 vid，这张表记录"哪个主键对应哪个 vid"
- 列结构：主键列 + vid 列（`VEC_VID()` 生成列，`ob_vec_index_builder_util.cpp:3144`），两列联合主键
- 写入时机：DML 插入新向量时，同步分配 vid 并写入映射
- 查询用途：某些执行路径需要把主键翻译成 vid（如过滤条件含主键时先转 vid 再在图里定位）

### ② vid_rowkey 表（`INDEX_TYPE_VEC_VID_ROWKEY_LOCAL`）

**作用：vid → 主键的反向映射表（回表翻译器）。**

- 与 rowkey_vid 方向相反：vid 在前、主键在后（主键列顺序互换，让两个方向的点查都走前缀扫描）
- 这是查询路径的刚需：HNSW 图搜索的输出天然是一组 **vid + 距离**（top-k），但客户端要的是
  完整行——必须用 vid 反查出主键，再回主表取数据。对应 `ObDASHNSWScanIterParam::vid_rowkey_iter_`
- `inner_get_next_row` 里行的结构 `cell(0)=距离, cell(1)=vid, 之后=主键列` 就是翻译后的结果

> 为什么两张映射表都要？OB 是索引组织表（数据按主键排序存储），主键前缀匹配的扫描最快。
> 正向表（rowkey 在前）支持"主键→vid"，反向表（vid 在前）支持"vid→主键"。
> 单张表只有一种前缀高效，所以同一份映射存两遍、列序互换。

### ③ delta buffer 表（`INDEX_TYPE_VEC_DELTA_BUFFER_LOCAL`）

**作用：同步模式下的新向量暂存表。**

- 列结构：vid + 向量数据列，两列联合主键
- 同步模式（sync_mode=immediate）：DML 事务内直写；查询时
  `complete_delta_buffer_table_data`（`ob_plugin_vector_index_adaptor.cpp:2321`）
  把它加载进内存增量 vsag 图（VIRT_INC）参与搜索
- async 模式：**建而不用**——DML 不写、查询跳过（`skip_delta_buffer_`），
  仅作为优化器识别向量扫描路径的 schema 占位
- 后台刷新后其内容归档进快照，可清理

### ④ index_id 表（`INDEX_TYPE_VEC_INDEX_ID_LOCAL`）

**作用：async 模式的写入日志 + vbitmap（增量位图）的物理载体。异步链路的核心表。**

三列联合主键（rowkey + vid + 向量数据列），承担两个职责：

**A. 异步写入的持久化日志。** Change Stream 插件处理每批变更时，先写这张表再写内存图
（`ob_cs_plugin_async_index.cpp:212-217`）：

```cpp
// Step 2 — for each tablet group: batch insert into index_id_table, then write into vsag
insert_vector_index_log_batch_(g.events_, g.vec_info_);   // ① 先落表
write_to_vsag_(g.events_, g.vec_info_);                   // ② 再写内存增量图
```

**B. vbitmap 的数据来源。** HNSW 扫描迭代器把它直接当作 vbitmap 的 tablet
（`ob_das_hnsw_scan_iter.cpp:925`：`index_ctx.vbitmap_tablet_id_ = index_id_tablet_id_`）。
async 模式查询时跳过 delta buffer、直接扫这张表重建位图（`:2834` 注释、
`do_index_id_table_scan` `:3530`）。vbitmap（roaring 64 位位图）记录"哪些 vid 还在增量图、
哪些被删了"，查询用它做增量/快照结果的合并去重；进程重启后内存位图丢失，靠扫这张表恢复。
`write_into_index_mem`（`ob_plugin_vector_index_adaptor.cpp:2738`）是从位图更新增量内存数据的地方。

**一句话：index_id 表 = async 模式下 delta buffer 的替代者，且多了"可重建位图/可恢复"的日志职责。**

### ⑤ snapshot_data 表（`INDEX_TYPE_VEC_INDEX_SNAPSHOT_DATA_LOCAL`）

**作用：快照 vsag 索引的持久化存储（图的落盘形态）。**

- **单列主键**（`OB_VEC_INDEX_SNAPSHOT_DATA_TABLE_INDEX_PK_COL_CNT = 1`，
  `ob_vec_index_builder_util.h:38`）——其他辅表主键 2~3 列（行式明细），这张只有 1 列，
  说明它存的不是"一行一个向量"的明细，而是**序列化后的索引块**
- 写入：后台刷新任务定期把内存增量图（VIRT_INC）合并/构建进快照图（VIRT_SNAP），
  然后把快照图序列化写进这张表
- 读取：查询时 `do_snapshot_table_scan`（`ob_das_hnsw_scan_iter.cpp:3543`）经
  `snapshot_iter_` 扫描它，加载进内存快照图后用于 vsag 搜索
- 有了它，重启后不用全量重建 HNSW，从这张表加载即可

### 场景串联：async 模式的一条 INSERT + 一条查询

```
INSERT t1 ...
  └─ 主表提交（不碰任何辅表）

Change Stream 异步处理：
  ├─ rowkey_vid：分配 vid，记录 主键→vid          （映射维护）
  ├─ index_id：写 (rowkey, vid, 向量)             （日志）
  └─ write_to_vsag_：向量进内存增量图 VIRT_INC     （可搜索了）

后台刷新：
  增量图 ─► 快照图 VIRT_SNAP ─序列化─► snapshot_data 表

SELECT ... ORDER BY l2_distance APPROX LIMIT 5：
  ├─ 扫 index_id 表 → 重建 vbitmap（增量 vid 集合）
  ├─ 加载/搜索 快照图（数据源 snapshot_data 表）
  ├─ 搜索 增量图（数据源 index_id 表 + 内存）
  ├─ 位图合并去重 → top-k (vid, 距离)
  ├─ vid_rowkey：vid → 主键
  └─ 回主表取行返回
```

### 五表速查

| 表 | 一句话定位 | 同步模式 | async 模式 |
|---|---|---|---|
| rowkey_vid | 主键→vid 正映射（vid 分配） | ✅ 写 | ✅ 写 |
| vid_rowkey | vid→主键 反映射（回表） | ✅ 用 | ✅ 用 |
| delta buffer | 增量暂存 | ✅ 写+读 | ❌ 占位不用 |
| index_id | 异步日志+位图载体 | — | ✅ 写+读（核心） |
| snapshot_data | HNSW 图落盘 | ✅ | ✅ |
