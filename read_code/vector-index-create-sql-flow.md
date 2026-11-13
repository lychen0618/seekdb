# CREATE VECTOR INDEX 语句执行流与关键函数

本文梳理独立建向量索引语句的完整执行链路（行号基于 v1.4.0，均经 grep 验证）：

```sql
-- 两种等价的写法（本文主要讲独立语句）：
create vector index idx1 on t1(c3) with (distance=l2, type=hnsw);

create table t1 (c1 int, c3 vector(3), primary key(c1),
  vector index idx1(c3) with (distance=l2, type=hnsw));  -- 见另一份笔记
```

---

## 一、执行流总览

```
SQL: create vector index idx1 on t1(c3) with (distance=l2, type=hnsw)
  │
  ▼ ① parser
  create_index_stmt → T_CREATE_INDEX(value_=6 即 VEC_KEY)
    └─ WITH(...) → T_VEC_INDEX_PARAMS
  │
  ▼ ② resolver（SQL 节点）
  ObCreateIndexResolver
    ├─ 单列/约束检查
    ├─ check_vec_index_param：解析 distance/type → index_params_、vec_index_type_
    └─ 产出 ObCreateIndexArg
  │
  ▼ ③ executor（Observer 本地）
  ObCreateIndexExecutor::execute
    └─ 向量索引强制走串行路径：root_command_service().create_index() RPC → rootserver
  │
  ▼ ④ rootserver
  ObIndexBuilder::create_index → do_create_index 分发
    └─ is_vec_index → do_create_local_index        ← 向量索引是 LOCAL 索引
    ├─ 空表优化检查（空表则直接置 AVAILABLE，结束）
    ├─ 事务内：submit_build_index_task
    │    └─ ObDDLScheduler::create_ddl_task(DDL_CREATE_VEC_INDEX)
    │         └─ create_build_vec_index_task
    │              └─ 构造 ObVecIndexBuildTask + insert_task_record → __all_ddl_task
    └─ 事务提交后 publish_schema + schedule_ddl_task
  │
  ▼ ⑤ 后台异步执行
  DDLScheduler scan_task_ 线程驱动 ObVecIndexBuildTask::process()
    └─ 状态机：建辅表 schema → 补数据（complement）→ 校验 → SUCCESS
```

注意 ③ 的一个关键点：**向量索引不走并行 DDL**。`ob_index_executor.cpp:90` 显式判断
`is_vec_index(...)` 后强制走串行 `create_index` 路径（普通索引才可能走 `parallel_create_index`）。

---

## 二、各阶段关键函数

### ① Parser（`src/sql/parser/sql_parser_mysql_mode.y`）

| 位置 | 内容 |
|---|---|
| `:7520` | `create_index_stmt:` 产生式：`opt_index_keyname INDEX ... ON table '(' cols ')' opt_index_option_list` |
| `:7572-7573` | `opt_index_keyname:` → `VECTOR { $$[0] = 6 }`，6 即 `INDEX_KEYNAME::VEC_KEY` |
| `:16772` | `vec_index_params:` → `distance=l2`、`type=hnsw` 等键值对，最终成 `T_VEC_INDEX_PARAMS` |

归约结果：`T_CREATE_INDEX` 节点，keyname 标记 6，WITH 参数挂在 index option 列表里。

### ② Resolver（`src/sql/resolver/ddl/ob_create_index_resolver.cpp`）

| 位置 | 内容 |
|---|---|
| `:134-135` | `is_vec_index` 单列约束：多列向量索引报错 |
| `:493-498` | `is_vec_index(index_arg.index_type_)` 分支：设置 `vec_index_type_`、`set_index_params(index_params_)` |

参数合法性校验复用同一个函数（定义在 `src/observer/vector_index/ob_vector_index_util.cpp:2688`）：

```cpp
ObVectorIndexUtil::check_vec_index_param(...)
  // 取列 extended_type_info 里的 dim；解析 "distance=l2,type=hnsw"
  // 写入 index_params_ 串，确定 vec_index_type_
```

### ③ Executor（`src/sql/engine/cmd/ob_index_executor.cpp`）

| 位置 | 内容 |
|---|---|
| `:44` | `ObCreateIndexExecutor::execute` 入口 |
| `:90` | `is_vec_index(create_index_arg.index_type_)` → 不走并行 DDL |
| `:93` | `root_command_service().create_index(create_index_arg, res)` RPC 到 rootserver |

### ④ Rootserver（`src/rootserver/ob_index_builder.cpp`）

| 位置 | 内容 |
|---|---|
| `:52` | `ObIndexBuilder::create_index` 入口（参数校验、重复名处理） |
| `:1497` | `do_create_index`：按索引类型分发（LOCAL/GLOBAL） |
| `:1551-1555` | `is_vec_index(arg.index_type_)` → **`do_create_local_index`**（向量索引类型均为 `*_LOCAL`） |
| `:1264` | `do_create_local_index`：真正建索引 schema 的事务流程（向量走这条） |
| `:1309` | `ObCreateIndexOnEmptyTableHelper::check_create_index_on_empty_table_opt` —— **空表检测** |
| `:1318` | 空表：直接 `INDEX_STATUS_AVAILABLE`，**不提交构建任务，语句到此结束** |
| `:1450` | 非空表：`submit_build_index_task`（同事务内登记 DDL 任务） |
| `:1486` | 事务提交后：`publish_schema()` + `ObSysDDLSchedulerUtil::schedule_ddl_task(task_record)` |
| `:607` | `submit_build_index_task` 定义：构造 `ObCreateDDLTaskParam`（向量索引 → `DDL_CREATE_VEC_INDEX`） |

### ⑤ 任务登记与调度（`src/rootserver/ddl_task/ob_ddl_scheduler.cpp`）

| 位置 | 内容 |
|---|---|
| `:1059` | `ObDDLScheduler::create_ddl_task`：按 DDL 类型分发 |
| `:1141` | `case DDL_CREATE_VEC_INDEX:` |
| `:1750` | `create_build_vec_index_task` |
| `:1765` | `SMART_VAR(ObVecIndexBuildTask, index_task)` —— **任务对象在此构造** |
| `:1790` | `insert_task_record`（调用点；函数定义在 `:2427`）→ `ObDDLTaskRecordOperator::insert_record` 写入内部表 `__all_ddl_task` |
| `:874` 起 | `scan_task_` 定时扫描线程：捞任务记录、驱动 `process()` |

任务记录与 DDL schema 在**同一事务**内落库，保证"索引 schema 可见"与"构建任务登记"原子生效。

---

## 三、ObVecIndexBuildTask 状态机（`src/rootserver/ddl_task/ob_vec_index_build_task.cpp`）

入口 `process()` 在 `:221`，按 `task_status_` 分发：

```
PREPARE                                  ← :536  prepare()
  ▼
GENERATE_ROWKEY_VID_SCHEMA               ← :599  prepare_rowkey_vid_table()
  ▼
WAIT_ROWKEY_VID_TABLE_COMPLEMENT         ← wait_aux_table_complement()
  ▼
GENERATE_VEC_AUX_SCHEMA                  ← :648  prepare_aux_index_tables()
  ▼
WAIT_VEC_AUX_TABLE_COMPLEMENT            ← wait_aux_table_complement()
  ▼
GENERATE_VID_ROWKEY_SCHEMA               ← :735  prepare_vid_rowkey_table()
  ▼
WAIT_VID_ROWKEY_TABLE_COMPLEMENT         ← wait_aux_table_complement()
  ▼
VALIDATE_CHECKSUM                        ← validate_checksum()
  ▼
SUCCESS                                  ← :1755 succ()

任意状态失败 → FAIL → clean_on_failed()（清理已建辅表）
```

每个 "GENERATE_xxx_SCHEMA" 状态做的事：在 DDL 事务里创建对应辅表 schema（delta buffer /
index_id / snapshot_data / rowkey_vid / vid_rowkey），并提交数据补全（complement）任务——
**从主表存量数据回填辅表**。"WAIT_xxx_COMPLEMENT" 轮询补全进度，全部完成才进入下一状态。

这就是 5 张辅表在独立 CREATE INDEX 场景下的创建方式：**由任务状态机渐进式创建**。

---

## 四、与 CREATE TABLE 内联写法的关键差异

| | `create table ... vector index ...` | `create vector index ... on t` |
|---|---|---|
| 解析入口 | `ob_create_table_resolver.cpp`（索引作为表定义一部分） | `ob_create_index_resolver.cpp` |
| 辅表 schema 何时建 | resolver 阶段就展开成 5 个 `ObCreateIndexArg`（`ob_vec_index_builder_util.cpp:55 append_vec_args`），随主表一次建齐 | **DDL 任务状态机渐进创建**（GENERATE_xxx_SCHEMA 状态） |
| 存量数据 | 表刚建，为空，无需回填 | 非空表需 complement 任务从主表回填辅表 |
| 是否提交 ObVecIndexBuildTask | 空表优化，**不提交** | 非空表**提交**；空表同样走优化不提交 |
| 建表结果可见时间 | 主表建好即完成 | schema 先可见（UNAVAILABLE），构建完成转 AVAILABLE |

一句话：**内联写法 = resolver 一次展开、随表建齐；独立写法 = rootserver 建主索引 schema，
ObVecIndexBuildTask 状态机分步建辅表并回填存量数据**。

---

## 五、常见问题排查锚点

| 现象 | 查哪里 |
|---|---|
| 语句报错单列/参数不支持 | `ob_create_index_resolver.cpp:134`、`check_vec_index_param` |
| 索引一直 UNAVAILABLE | `__all_ddl_task` 任务状态；`process()` 卡在哪个 WAIT_xxx_COMPLEMENT |
| 构建失败想看清理逻辑 | `clean_on_failed()` |
| 空表为何没有任务 | `ob_index_builder.cpp:1309` 空表优化分支（local 路径） |
