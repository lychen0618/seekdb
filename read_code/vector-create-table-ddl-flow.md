# seekdb 向量建表语句：vector 类型与向量索引关键实现及执行流程

以这条语句为例，梳理「列类型 `vector(3)`」和「向量索引 DDL」两条解析线以及整体执行流程（行号基于 v1.4.0）。

```sql
create table t1 (c1 int, c3 vector(3), primary key(c1),
  vector index idx1(c3) with (distance=l2, type=hnsw));
```

---

## 一、`vector(3)` 列类型的关键实现

**① 词法/语法**（`src/sql/parser/sql_parser_mysql_mode.y:5725`）

```yacc
| VECTOR '(' INTNUM ')'
{
  malloc_non_terminal_node($$, result->malloc_pool_, T_COLLECTION, 1, $3);
  $$->int32_values_[0] = 1; /* vector type */
}
```

向量/数组/Map/稀疏向量共用 `T_COLLECTION` 节点，用 `int32_values_[0]` 区分（0=array, **1=vector**, 2=map, 3=sparse vector）。

**② Resolver**（`src/sql/resolver/ob_resolver_utils.cpp:345` → `resolve_vector_type_info`，`:444`）

`resolve_collection_type_info` 按标记分发；vector 分支做两件事：
- **校验维度**：`dim > 0 && dim <= 16000`，否则报 `OB_NOT_SUPPORTED`
- **生成扩展类型信息**：拼出字符串 `"VECTOR(3)"`

列的最终存储形态 = `ObVectorType` 对象类型 + `extended_type_info = "VECTOR(3)"`（维度不占字段长度，存在 schema 的扩展信息里）。

---

## 二、`vector index ... with (distance=l2, type=hnsw)` 的关键实现

**① 语法**（`sql_parser_mysql_mode.y:4700` 与 `:16772`）

```yacc
| VECTOR key_or_index opt_index_name '(' sort_column_list ')' opt_index_option_list
{ ... $$->value_ = 6; }   // value_=6 即 INDEX_KEYNAME::VEC_KEY

vec_index_param: relation_name COMP_EQ vec_index_param_value  // distance=l2 / type=hnsw
```

`WITH(...)` 里的键值对解析成 `T_VEC_INDEX_PARAMS` 选项节点。

**② Resolver 约束检查**（`src/sql/resolver/ddl/ob_create_table_resolver.cpp:1560` 起）

`INDEX_KEYNAME::VEC_KEY` 分支强制：
- 单列向量索引（多列报错，`:1594`）
- 向量列不能做主键（`:2260`）
- 同一列只能建一个向量索引（`:1723`）
- 向量列上只能建向量索引，不能建普通索引（`:1740`）

**③ 参数校验与索引类型判定**（`ob_ddl_resolver.cpp:1253` → `src/observer/vector_index/ob_vector_index_util.cpp:2688`）

```cpp
ObVectorIndexUtil::check_vec_index_param(...)
  // 从列 schema 的 extended_type_info 里取出 vector_dim=3
  // 解析 "distance=l2,type=hnsw" → 校验组合合法
  // 结果写入 index_params_（序列化串）并确定 vec_index_type_
```

`type=hnsw` 最终落到 `vec_index_type_ = INDEX_TYPE_VEC_DELTA_BUFFER_LOCAL`（HNSW 的主索引类型枚举）。

**④ 生成辅助表参数**（`ob_create_table_resolver.cpp:1952` → `ObVecIndexBuilderUtil::append_vec_args`，`src/sql/resolver/ddl/ob_vec_index_builder_util.cpp:55`）

这是最关键的一步——**一条向量索引 DDL 会展开成 5 张内部辅表**（首建时，`append_vec_hnsw_args`，`:130`）：

| 辅表 | index_type | 职责 |
|---|---|---|
| `idx1_..._delta_buffer_...` | `INDEX_TYPE_VEC_DELTA_BUFFER_LOCAL` | 增量数据（同步模式下 DML 直写） |
| `..._rowkey_vid_...` | `..._ROWKEY_VID_LOCAL` | 主键 → vid 映射 |
| `..._vid_rowkey_...` | `..._VID_ROWKEY_LOCAL` | vid → 主键 映射（回表） |
| `..._index_id_...` | index_id 表 | 异步刷新任务的状态协调 |
| `..._index_snapshot_data_table` | `..._SNAPSHOT_DATA_LOCAL` | 快照索引数据（vsag HNSW 落盘） |

> 源码里有条重要注释（`ob_vec_index_builder_util.cpp:142-145`，`append_vec_hnsw_args` 函数开头）：即使 async 索引也会创建 delta buffer 表——**优化器需要靠它识别向量索引扫描路径**，只是 async 模式下 DML 不写它、查询跳过它（`skip_delta_buffer_`）。

---

## 三、整体执行流程

```
SQL 文本
  │
  ▼ ① parser（语法 .y 文件）
  ParseTree: T_CREATE_TABLE
    ├─ 列定义: T_COLLECTION(int32=1, dim=3)          ← vector(3)
    └─ 索引定义: T_INDEX(value_=6, params节点)         ← vector index ... with(...)
  │
  ▼ ② resolver
  ObCreateTableStmt + ObTableSchema
    ├─ 列 c3: ObVectorType + extended_type_info="VECTOR(3)"
    │    （resolve_vector_type_info 校验 1≤dim≤16000）
    ├─ check_vec_index_param: 校验 distance/type → index_params_、vec_index_type_
    └─ append_vec_args: 展开出 5 个 ObCreateIndexArg（辅表）挂进 index_arg_list
  │
  ▼ ③ executor（src/sql/engine/cmd/ob_table_executor.cpp:451）
  ObCreateTableExecutor::execute
    ├─ set_index_arg_list: 把 5 个辅表 arg 塞进 create_table_arg
    └─ ObDDLExecutorUtil::execute_pcreate_table
        → root_command_service → rootserver 的 create_table RPC
  │
  ▼ ④ rootserver/schema service
  一个事务内建齐：主表 t1 + 5 张辅表的 schema 和 tablet
  （所以 show create table 只看到一条 vector index，底层是 1+5 张表）
  │
  ▼ ⑤ 后续
  - 已有数据：DDL 任务（rootserver/ddl_task/ob_vec_index_build_task）扫数据建 HNSW
  - 新写入：同步模式走 delta buffer；async 模式走 change stream → 增量 vsag 索引
  - 查询：DAS_ITER_HNSW_SCAN 同时读 delta + snapshot（见 vector-retrieval-reading-guide.md）
```

**记忆锚点**：`vector(3)` 是"列类型"问题（parser → extended_type_info）；`vector index ... with(...)` 是"索引 DDL 展开"问题（一条语句 → 1 主索引 + 5 辅表）；两者在 `ObCreateTableResolver` 汇合，由 rootserver 一次建齐。
