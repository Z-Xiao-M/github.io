# 什么是TOAST

PostgreSQL 的 TOAST（The Oversized-Attribute Storage Technique，超大属性存储技术）是针对大尺寸数据（如长文本、二进制数据等）的存储优化机制，当字段数据超过阈值时，会自动将其压缩或拆分后存储到独立的 TOAST 表中，主表仅保留引用指针，既解决了单条记录存储容量受限问题，又通过透明操作、多种存储策略和独立表设计，平衡了存储效率与访问性能，对 TEXT、BYTEA、JSONB 等可能存储大数据的类型尤为有效。

# 基础内容演示

```sql
postgres=# -- 创建一个带有text字段的测试表
postgres=# create table t(a text);
CREATE TABLE
postgres=# -- 查询相关元数据信息 可以看到reltoastrelid不为0
postgres=# select * from pg_class where relname = 't' \gx
-[ RECORD 1 ]-------+-------
oid                 | 190095
relname             | t
relnamespace        | 2200
reltype             | 190097
reloftype           | 0
relowner            | 10
relam               | 2
relfilenode         | 190095
reltablespace       | 0
relpages            | 0
reltuples           | -1
relallvisible       | 0
reltoastrelid       | 190098
relhasindex         | f
relisshared         | f
relpersistence      | p
relkind             | r
relnatts            | 1
relchecks           | 0
relhasrules         | f
relhastriggers      | f
relhassubclass      | f
relrowsecurity      | f
relforcerowsecurity | f
relispopulated      | t
relreplident        | d
relispartition      | f
relrewrite          | 0
relfrozenxid        | 7322
relminmxid          | 1
relacl              | 
reloptions          | 
relpartbound        | 

postgres=# -- 依据reltoastrelid 查询元数据信息
postgres=# -- 大致可以看到toast表是没有同名数据类型的
postgres=# select * from pg_class where oid = 190098 \gx
-[ RECORD 1 ]-------+----------------
oid                 | 190098
relname             | pg_toast_190095
relnamespace        | 99
reltype             | 0
reloftype           | 0
relowner            | 10
relam               | 2
relfilenode         | 190098
reltablespace       | 0
relpages            | 0
reltuples           | -1
relallvisible       | 0
reltoastrelid       | 0
relhasindex         | t
relisshared         | f
relpersistence      | p
relkind             | t
relnatts            | 3
relchecks           | 0
relhasrules         | f
relhastriggers      | f
relhassubclass      | f
relrowsecurity      | f
relforcerowsecurity | f
relispopulated      | t
relreplident        | n
relispartition      | f
relrewrite          | 0
relfrozenxid        | 7322
relminmxid          | 1
relacl              | 
reloptions          | 
relpartbound        | 

postgres=# -- 查看toast表 附加上pg_toast
postgres=# \d+ pg_toast.pg_toast_190095
TOAST table "pg_toast.pg_toast_190095"
   Column   |  Type   | Storage 
------------+---------+---------
 chunk_id   | oid     | plain
 chunk_seq  | integer | plain
 chunk_data | bytea   | plain
Owning table: "public.t"
Indexes:
    "pg_toast_190095_index" PRIMARY KEY, btree (chunk_id, chunk_seq)
Access method: heap

postgres=# -- 查询表对应的toast表名
postgres=# SELECT 
  oid AS main_table_oid,
  reltoastrelid AS toast_table_oid,
  reltoastrelid::regclass::text AS toast_table_name
FROM pg_class WHERE relname='t'; 
 main_table_oid | toast_table_oid |     toast_table_name     
----------------+-----------------+--------------------------
         190095 |          190098 | pg_toast.pg_toast_190095
(1 row)

postgres=# -- 删除表
postgres=# drop table t;
DROP TABLE
```
# 尝试触发toast机制

```sql
postgres=# -- 创建一个带有text字段的测试表
postgres=# CREATE TABLE t (c text);
CREATE TABLE
postgres=# -- 生成指定长度的随机字符串数据
postgres=# CREATE OR REPLACE FUNCTION random_string(length integer)
 RETURNS text
 LANGUAGE sql
 IMMUTABLE
AS $function$
SELECT 
  string_agg(
    (ARRAY['0','1','2','3','4','5','6','7','8','9',
           'A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R','S','T','U','V','W','X','Y','Z',
           'a','b','c','d','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y','z']) [floor(random() * 62 + 1)::INTEGER],
    '') FROM generate_series(1, GREATEST(length, 1));
$function$;
CREATE FUNCTION
postgres=# -- 查看对应的toast表名
postgres=# SELECT 
  oid AS main_table_oid,
  reltoastrelid AS toast_table_oid,
  reltoastrelid::regclass::text AS toast_table_name
FROM pg_class WHERE relname='t'; 
 main_table_oid | toast_table_oid |     toast_table_name     
----------------+-----------------+--------------------------
         190100 |          190103 | pg_toast.pg_toast_190100
(1 row)

postgres=# -- 插入长度为2004的数据看是否触发
postgres=# insert into t select random_string(2004);
INSERT 0 1
postgres=# -- 可以看到对应的toast表大小为0 没有触发
postgres=# select pg_relation_size('t'), pg_relation_size('pg_toast.pg_toast_190100');
 pg_relation_size | pg_relation_size 
------------------+------------------
             8192 |                0
(1 row)

postgres=# -- 插入长度为2005的数据看是否触发
postgres=# insert into t select random_string(2005);
INSERT 0 1
postgres=# -- 成功触发
postgres=# select pg_relation_size('t'), pg_relation_size('pg_toast.pg_toast_190100');
 pg_relation_size | pg_relation_size 
------------------+------------------
             8192 |             8192
(1 row)

postgres=# 
```
# 为什么2005长度才能触发

有的资料写的是2KB触发，有的资料写的是2000字节触发，当然这是可以配置的，这里是2005，至于为什么是2005，那么需要我们来瞅瞅代码，值得关注的宏为`TOAST_TUPLE_THRESHOLD`

```c
/*
 * Find the maximum size of a tuple if there are to be N tuples per page.
 */
#define MaximumBytesPerTuple(tuplesPerPage) \
	MAXALIGN_DOWN((BLCKSZ - \
				   MAXALIGN(SizeOfPageHeaderData + (tuplesPerPage) * sizeof(ItemIdData))) \
				  / (tuplesPerPage))

/*
 * These symbols control toaster activation.  If a tuple is larger than
 * TOAST_TUPLE_THRESHOLD, we will try to toast it down to no more than
 * TOAST_TUPLE_TARGET bytes through compressing compressible fields and
 * moving EXTENDED and EXTERNAL data out-of-line.
 *
 * The numbers need not be the same, though they currently are.  It doesn't
 * make sense for TARGET to exceed THRESHOLD, but it could be useful to make
 * it be smaller.
 *
 * Currently we choose both values to match the largest tuple size for which
 * TOAST_TUPLES_PER_PAGE tuples can fit on a heap page.
 *
 * XXX while these can be modified without initdb, some thought needs to be
 * given to needs_toast_table() in toasting.c before unleashing random
 * changes.  Also see LOBLKSIZE in large_object.h, which can *not* be
 * changed without initdb.
 */
#define TOAST_TUPLES_PER_PAGE	4

#define TOAST_TUPLE_THRESHOLD	MaximumBytesPerTuple(TOAST_TUPLES_PER_PAGE)

#define TOAST_TUPLE_TARGET		TOAST_TUPLE_THRESHOLD
```
AI说的比我写的好，这里就直接贴一下好了
<img width="799" height="589" alt="Image" src="https://github.com/user-attachments/assets/1f1f2d17-4a40-45fc-af1d-32ff113d80e6" />

代码逻辑

```c
static HeapTuple
heap_prepare_insert(Relation relation, HeapTuple tup, TransactionId xid,
					CommandId cid, int options)
{
	/*
	 * To allow parallel inserts, we need to ensure that they are safe to be
	 * performed in workers. We have the infrastructure to allow parallel
	 * inserts in general except for the cases where inserts generate a new
	 * CommandId (eg. inserts into a table having a foreign key column).
	 */
	if (IsParallelWorker())
		ereport(ERROR,
				(errcode(ERRCODE_INVALID_TRANSACTION_STATE),
				 errmsg("cannot insert tuples in a parallel worker")));

	tup->t_data->t_infomask &= ~(HEAP_XACT_MASK);
	tup->t_data->t_infomask2 &= ~(HEAP2_XACT_MASK);
	tup->t_data->t_infomask |= HEAP_XMAX_INVALID;
	HeapTupleHeaderSetXmin(tup->t_data, xid);
	if (options & HEAP_INSERT_FROZEN)
		HeapTupleHeaderSetXminFrozen(tup->t_data);

	HeapTupleHeaderSetCmin(tup->t_data, cid);
	HeapTupleHeaderSetXmax(tup->t_data, 0); /* for cleanliness */
	tup->t_tableOid = RelationGetRelid(relation);

	/*
	 * If the new tuple is too big for storage or contains already toasted
	 * out-of-line attributes from some other relation, invoke the toaster.
	 */
	if (relation->rd_rel->relkind != RELKIND_RELATION &&
		relation->rd_rel->relkind != RELKIND_MATVIEW)
	{
		/* toast table entries should never be recursively toasted */
		Assert(!HeapTupleHasExternal(tup));
		return tup;
	}
	else if (HeapTupleHasExternal(tup) || tup->t_len > TOAST_TUPLE_THRESHOLD)  // 关注此处
		return heap_toast_insert_or_update(relation, tup, NULL, options);		// 后续设计插入数据至toast表 可以参考后面的调用堆栈
	else
		return tup;
}
```
在64位系统，默认page为8KB的场景，需要2005字节长度触发，因为2005 + 24（HeapTupleHeaderData堆元组头对齐） + 4（变长数据四字节VARHDRSZ） = 2033，而TOAST\_TUPLE\_THRESHOLD刚好是2032。

对更多细节感兴趣的同学，还可以关注函数`SET_VARSIZE、heap_form_tuple`，应该可以解决你的疑惑。

# 明明仅仅插入了一条数据，为什么显示toast的表中存在两条

```sql
postgres=# CREATE TABLE t (c text);
CREATE TABLE
postgres=# insert into t select random_string(2005);
INSERT 0 1
postgres=# SELECT 
  oid AS main_table_oid,
  reltoastrelid AS toast_table_oid,
  reltoastrelid::regclass::text AS toast_table_name
FROM pg_class WHERE relname='t'; 
 main_table_oid | toast_table_oid |     toast_table_name     
----------------+-----------------+--------------------------
         190106 |          190109 | pg_toast.pg_toast_190106
(1 row)

postgres=# select count(*) from t;
 count 
-------
     1
(1 row)

postgres=# select count(*) from pg_toast.pg_toast_190106;
 count 
-------
     2
(1 row)

postgres=# select length(chunk_data) from pg_toast.pg_toast_190106;
 length 
--------
   1996
      9
(2 rows)
```
因为数据太长被切片了，可以直接通过运行pg\_controldata读取Maximum size of a TOAST chunk获得chunk大小
```shell
postgres@zxm-VMware-Virtual-Platform:~$ pg_controldata | grep 'Maximum size of a TOAST chunk' 
Maximum size of a TOAST chunk:        1996
``` 
也可以通过计算宏`TOAST_MAX_CHUNK_SIZE`，
```c
/*
 * When we store an oversize datum externally, we divide it into chunks
 * containing at most TOAST_MAX_CHUNK_SIZE data bytes.  This number *must*
 * be small enough that the completed toast-table tuple (including the
 * ID and sequence fields and all overhead) will fit on a page.
 * The coding here sets the size on the theory that we want to fit
 * EXTERN_TUPLES_PER_PAGE tuples of maximum size onto a page.
 *
 * NB: Changing TOAST_MAX_CHUNK_SIZE requires an initdb.
 */
#define EXTERN_TUPLES_PER_PAGE	4	/* tweak only this */

#define EXTERN_TUPLE_MAX_SIZE	MaximumBytesPerTuple(EXTERN_TUPLES_PER_PAGE)

#define TOAST_MAX_CHUNK_SIZE	\
	(EXTERN_TUPLE_MAX_SIZE -							\
	 MAXALIGN(SizeofHeapTupleHeader) -					\
	 sizeof(Oid) -										\
	 sizeof(int32) -									\
	 VARHDRSZ)

```
2032 - 24（元组头）- 4（chunk_id）- 4 (chunk_seq) - 4 (变长类型四字节头) = 1996


实际处理逻辑位于 `src/backend/access/common/toast_internals.c` `toast_save_datum`

```c
	/*
	 * Split up the item into chunks
	 */
	while (data_todo > 0)
	{
		int			i;

		CHECK_FOR_INTERRUPTS();

		/*
		 * Calculate the size of this chunk
		 */
		// 超过1996的大小 按照1996分割 多次处理
		chunk_size = Min(TOAST_MAX_CHUNK_SIZE, data_todo);

		/*
		 * Build a tuple and store it
		 */
		t_values[1] = Int32GetDatum(chunk_seq++);
		SET_VARSIZE(&chunk_data, chunk_size + VARHDRSZ);
		memcpy(VARDATA(&chunk_data), data_p, chunk_size);
		toasttup = heap_form_tuple(toasttupDesc, t_values, t_isnull);
		// 插入toast表
		heap_insert(toastrel, toasttup, mycid, options, NULL);

		/*
		 * Create the index entry.  We cheat a little here by not using
		 * FormIndexDatum: this relies on the knowledge that the index columns
		 * are the same as the initial columns of the table for all the
		 * indexes.  We also cheat by not providing an IndexInfo: this is okay
		 * for now because btree doesn't need one, but we might have to be
		 * more honest someday.
		 *
		 * Note also that there had better not be any user-created index on
		 * the TOAST table, since we don't bother to update anything else.
		 */
		// 处理toast表对应的索引
		for (i = 0; i < num_indexes; i++)
		{
			/* Only index relations marked as ready can be updated */
			if (toastidxs[i]->rd_index->indisready)
				index_insert(toastidxs[i], t_values, t_isnull,
							 &(toasttup->t_self),
							 toastrel,
							 toastidxs[i]->rd_index->indisunique ?
							 UNIQUE_CHECK_YES : UNIQUE_CHECK_NO,
							 false, NULL);
		}

		/*
		 * Free memory
		 */
		heap_freetuple(toasttup);

		/*
		 * Move on to next chunk
		 */
		data_todo -= chunk_size;
		data_p += chunk_size;
	}
```

# 数据都在toast表中那么主表里面存了个啥

主表插入了一条18字节长度的"TOAST pointer"

```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# create extension pageinspect;
CREATE EXTENSION
postgres=# CREATE TABLE t (c text);
CREATE TABLE
postgres=# insert into t select random_string(2005);
INSERT 0 1
postgres=# SELECT 
  oid AS main_table_oid,
  reltoastrelid AS toast_table_oid,
  reltoastrelid::regclass::text AS toast_table_name
FROM pg_class WHERE relname='t'; 
 main_table_oid | toast_table_oid |     toast_table_name     
----------------+-----------------+--------------------------
         190157 |          190160 | pg_toast.pg_toast_190157
(1 row)

postgres=# select chunk_id from pg_toast.pg_toast_190157;
 chunk_id 
----------
   190162
   190162
(2 rows)

postgres=# SELECT encode(t_data,'hex')
FROM   heap_page_items(get_raw_page('t',0));
                encode                
--------------------------------------
 0112d9070000d5070000d2e60200d0e60200
(1 row)

postgres=# select x'01'::int as va_header, x'12'::int as va_tag, x'07d9'::int as va_rawsize,
x'07d5'::int as va_extinfo, x'02e6d2'::int as va_valueid, x'02e6d0'::int as va_toastrelid;
 va_header | va_tag | va_rawsize | va_extinfo | va_valueid | va_toastrelid 
-----------+--------+------------+------------+------------+---------------
         1 |     18 |       2009 |       2005 |     190162 |        190160
(1 row)

postgres=# 
```
- va_header 是标识大端还是小端，决定了我们需要怎么去解析数据，这里是小端0x01，对于大端应该是0x80
- va_tag 标识"TOAST pointer"的状态
- va_rawsize 带有变长数据头(4 字节) + 原始数据长度
- va_extinfo 原始数据长度
- va_valueid 对应chunk_id 
- va_toastrelid 就显而易见了，是toast表的oid

相关数据结构和接口

```c
/*
 * Type tag for the various sorts of "TOAST pointer" datums.  The peculiar
 * value for VARTAG_ONDISK comes from a requirement for on-disk compatibility
 * with a previous notion that the tag field was the pointer datum's length.
 */
typedef enum vartag_external
{
	VARTAG_INDIRECT = 1,
	VARTAG_EXPANDED_RO = 2,
	VARTAG_EXPANDED_RW = 3,
	VARTAG_ONDISK = 18
} vartag_external;

/*
 * struct varatt_external is a traditional "TOAST pointer", that is, the
 * information needed to fetch a Datum stored out-of-line in a TOAST table.
 * The data is compressed if and only if the external size stored in
 * va_extinfo is less than va_rawsize - VARHDRSZ.
 *
 * This struct must not contain any padding, because we sometimes compare
 * these pointers using memcmp.
 *
 * Note that this information is stored unaligned within actual tuples, so
 * you need to memcpy from the tuple into a local struct variable before
 * you can look at these fields!  (The reason we use memcmp is to avoid
 * having to do that just to detect equality of two TOAST pointers...)
 */
typedef struct varatt_external
{
	int32		va_rawsize;		/* Original data size (includes header) */
	uint32		va_extinfo;		/* External saved size (without header) and
								 * compression method */
	Oid			va_valueid;		/* Unique ID of value within TOAST table */
	Oid			va_toastrelid;	/* RelID of TOAST table containing it */
}			varatt_external;

SET_VARTAG_EXTERNAL(result, VARTAG_ONDISK);

static inline void
SET_VARTAG_EXTERNAL(void *PTR, vartag_external tag)
{
  SET_VARTAG_1B_E(PTR, tag);
}

#define SET_VARTAG_1B_E(PTR,tag) \
  (((varattrib_1b_e *) (PTR))->va_header = 0x01, \
   ((varattrib_1b_e *) (PTR))->va_tag = (tag))
```

省略部分代码`src/backend/access/common/toast_internals.c` `toast_save_datum`

```c
Datum
toast_save_datum(Relation rel, Datum value,
				 struct varlena *oldexternal, int options)
{
	struct varlena *result;
	struct varatt_external toast_pointer;
    
	// ......

	/*
	 * Get the data pointer and length, and compute va_rawsize and va_extinfo.
	 *
	 * va_rawsize is the size of the equivalent fully uncompressed datum, so
	 * we have to adjust for short headers.
	 *
	 * va_extinfo stored the actual size of the data payload in the toast
	 * records and the compression method in first 2 bits if data is
	 * compressed.
	 */
	if (VARATT_IS_SHORT(dval))
	{
		toast_pointer.va_rawsize = data_todo + VARHDRSZ;	/* as if not short */
		toast_pointer.va_extinfo = data_todo;
	}
	else if (VARATT_IS_COMPRESSED(dval))
	{
		/* rawsize in a compressed datum is just the size of the payload */
		toast_pointer.va_rawsize = VARDATA_COMPRESSED_GET_EXTSIZE(dval) + VARHDRSZ;

		/* set external size and compression method */
		VARATT_EXTERNAL_SET_SIZE_AND_COMPRESS_METHOD(toast_pointer, data_todo,
													 VARDATA_COMPRESSED_GET_COMPRESS_METHOD(dval));
		/* Assert that the numbers look like it's compressed */
		Assert(VARATT_EXTERNAL_IS_COMPRESSED(toast_pointer));
	}
	else
	{
		toast_pointer.va_rawsize = VARSIZE(dval);
		toast_pointer.va_extinfo = data_todo;
	}

	/*
	 * Insert the correct table OID into the result TOAST pointer.
	 *
	 * Normally this is the actual OID of the target toast table, but during
	 * table-rewriting operations such as CLUSTER, we have to insert the OID
	 * of the table's real permanent toast table instead.  rd_toastoid is set
	 * if we have to substitute such an OID.
	 */
	if (OidIsValid(rel->rd_toastoid))
		toast_pointer.va_toastrelid = rel->rd_toastoid;
	else
		toast_pointer.va_toastrelid = RelationGetRelid(toastrel);

      // ......

	/*
	 * Create the TOAST pointer value that we'll return
	 */
	result = (struct varlena *) palloc(TOAST_POINTER_SIZE);
	SET_VARTAG_EXTERNAL(result, VARTAG_ONDISK);  // 这里设置VARTAG_ONDISK
	memcpy(VARDATA_EXTERNAL(result), &toast_pointer, sizeof(toast_pointer));

	return PointerGetDatum(result);
}
```

# 调用堆栈

```c
toast_save_datum(Relation rel, Datum value, struct varlena * oldexternal, int options) (\home\postgres\code\18\src\backend\access\common\toast_internals.c:198)
toast_tuple_externalize(ToastTupleContext * ttc, int attribute, int options) (\home\postgres\code\18\src\backend\access\table\toast_helper.c:263)
heap_toast_insert_or_update(Relation rel, HeapTuple newtup, HeapTuple oldtup, int options) (\home\postgres\code\18\src\backend\access\heap\heaptoast.c:217)
heap_prepare_insert(Relation relation, HeapTuple tup, TransactionId xid, CommandId cid, int options) (\home\postgres\code\18\src\backend\access\heap\heapam.c:2305)
heap_insert(Relation relation, HeapTuple tup, CommandId cid, int options, BulkInsertState bistate) (\home\postgres\code\18\src\backend\access\heap\heapam.c:2098)
heapam_tuple_insert(Relation relation, TupleTableSlot * slot, CommandId cid, int options, BulkInsertState bistate) (\home\postgres\code\18\src\backend\access\heap\heapam_handler.c:255)
table_tuple_insert(Relation rel, TupleTableSlot * slot, CommandId cid, int options, struct BulkInsertStateData * bistate) (\home\postgres\code\18\src\include\access\tableam.h:1365)
ExecInsert(ModifyTableContext * context, ResultRelInfo * resultRelInfo, TupleTableSlot * slot, _Bool canSetTag, TupleTableSlot ** inserted_tuple, ResultRelInfo ** insert_destrel) (\home\postgres\code\18\src\backend\executor\nodeModifyTable.c:1234)
ExecModifyTable(PlanState * pstate) (\home\postgres\code\18\src\backend\executor\nodeModifyTable.c:4468)
ExecProcNodeFirst(PlanState * node) (\home\postgres\code\18\src\backend\executor\execProcnode.c:469)
ExecProcNode(PlanState * node) (\home\postgres\code\18\src\include\executor\executor.h:315)
ExecutePlan(QueryDesc * queryDesc, CmdType operation, _Bool sendTuples, uint64 numberTuples, ScanDirection direction, DestReceiver * dest) (\home\postgres\code\18\src\backend\executor\execMain.c:1678)
standard_ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count) (\home\postgres\code\18\src\backend\executor\execMain.c:366)
ExecutorRun(QueryDesc * queryDesc, ScanDirection direction, uint64 count) (\home\postgres\code\18\src\backend\executor\execMain.c:303)
ProcessQuery(PlannedStmt * plan, const char * sourceText, ParamListInfo params, QueryEnvironment * queryEnv, DestReceiver * dest, QueryCompletion * qc) (\home\postgres\code\18\src\backend\tcop\pquery.c:161)
PortalRunMulti(Portal portal, _Bool isTopLevel, _Bool setHoldSnapshot, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (\home\postgres\code\18\src\backend\tcop\pquery.c:1272)
PortalRun(Portal portal, long count, _Bool isTopLevel, DestReceiver * dest, DestReceiver * altdest, QueryCompletion * qc) (\home\postgres\code\18\src\backend\tcop\pquery.c:788)
exec_simple_query(const char * query_string) (\home\postgres\code\18\src\backend\tcop\postgres.c:1274)
PostgresMain(const char * dbname, const char * username) (\home\postgres\code\18\src\backend\tcop\postgres.c:4767)
BackendMain(const void * startup_data, size_t startup_data_len) (\home\postgres\code\18\src\backend\tcop\backend_startup.c:124)
```

# 为什么不推荐使用SELECT *
这是一个老生常谈的问题，就是在应用程序开发中，推荐需要获取什么数据就去查询对应的字段，而不是直接SELECT * 一把梭。可是还是有人觉得无所谓，觉得我直接查询更多的数据给前端，前端想用哪个字段就用哪个字段，emmmm~
这里可以用toast构建个简单的场景，演示一下为什么不推荐使用SELECT *。
```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# create table t(a int, b text);
CREATE TABLE
postgres=# DO $$
begin
for i in 1 .. 1000 loop
insert into t values(i, random_string(2005));
end loop;
end; $$ language plpgsql;
DO
postgres=# select count(*) from t;
 count 
-------
  1000
(1 row)

postgres=# \q
postgres@zxm-VMware-Virtual-Platform:~$ psql -o /dev/null
psql (16.10)
Type "help" for help.

postgres=# \timing
Timing is on.
postgres=# select a from t;
Time: 2.137 ms
postgres=# select a from t;
Time: 1.180 ms
postgres=# select a from t;
Time: 1.095 ms
postgres=# select a from t;
Time: 0.867 ms
postgres=# select a from t;
Time: 0.863 ms
postgres=# select a from t;
Time: 1.111 ms
postgres=# select a from t;
Time: 1.045 ms
postgres=# select a from t;
Time: 0.892 ms
postgres=# select a from t;
Time: 1.259 ms
postgres=# select a from t;
Time: 0.776 ms
postgres=# \q
postgres@zxm-VMware-Virtual-Platform:~$ psql -o /dev/null
psql (16.10)
Type "help" for help.

postgres=# \timing
Timing is on.
postgres=# select * from t;
Time: 27.057 ms
postgres=# select * from t;
Time: 26.083 ms
postgres=# select * from t;
Time: 19.612 ms
postgres=# select * from t;
Time: 19.230 ms
postgres=# select * from t;
Time: 19.653 ms
postgres=# select * from t;
Time: 14.698 ms
postgres=# select * from t;
Time: 13.891 ms
postgres=# select * from t;
Time: 14.165 ms
postgres=# select * from t;
Time: 14.738 ms
postgres=# select * from t;
Time: 18.510 ms
postgres=# 
``` 
感觉这都不需要再解释了。

> 更多内容参考灿灿老师翻译的 https://postgres-internals.cn/docs/chapter01/#118-toast