# 文件分支

在 PostgreSQL 中，一个表的所有信息分别存储在几个不同的分支中，每个分支包含特定类型的数据，分支文件的存放路径通常为\$PGDATA/base/\[数据库OID]/xxx，当其大小达到 1GB （默认值，可以编译时修改）时，就会创建该分支的另一个文件 (这些文件有时被称为段)。段的序列号会被添加到文件名的末尾。

```c
typedef enum ForkNumber
{
	InvalidForkNumber = -1,
	MAIN_FORKNUM = 0,
	FSM_FORKNUM,
	VISIBILITYMAP_FORKNUM,
	INIT_FORKNUM
} ForkNumber;
```

*   **主文件（主分支）** 是存储表实际数据的文件，以表的`relfilenode`命名（通常来说和表的oid保持一致，但也有不一致的情形，比如说使用了 `TRUNCATE`、`REINDEX、CLUSTER` 等等）。
*   **空闲空间映射** 用于跟踪页内的可用空间。其容量一直在变化，vacuum 后变大，并在新的行版本出现时变小。空闲空间映射用于快速找到可以容纳被插入的新数据的页面。所有与空闲空间映射相关的文件都带有 \_fsm 后缀。为了加快搜索速度，空闲空间映射以一棵树的形式组织，它至少有三个数据页 (因此即使是几乎空的表，其文件大小也会有所体现)。
*   **可见性映射** 可以快速显示页面是否需要被清理或冻结。 它为每个表页面提供了两个比特位。第一个比特，为仅包含最新行版本的页面设置。vacuum 操作会跳过这样的页面，因为没有东西需要清理。此外，当某个事务尝试从这样的页面读取一行数据时，没有必要检查其可见性，因此便可以使用仅索引扫描。当页面包含的行版本都已被冻结后，便会设置第二个比特。可见性映射文件带有 \_vm 后缀。它们通常是最小的文件
*   **初始分支** 仅适用于UNLOGGED TABLE及其索引。此类对象与常规对象相同，不同之处在于对它们执行的任何操作都不会写入预写式日志。这使得这些操作的速度非常的快，但如果发生故障，将无法恢复一致的数据。因此，在恢复期间，PostgreSQL 会简单地删除此类对象的所有分支，并用初始分支覆盖主分支，从而创建了一个伪文件。

> 更多详细内容请参考 [第 1 章：介绍 – PostgreSQL 14 Internals](https://postgres-internals.cn/docs/chapter01/)



# UNLOGGED TABLE

通过上面的表述，我们了解到对于postgresql而言，存在四种文件分支，而UNLOGGED TABLE是唯一一个同时拥有四种分支的表。接下来让我们简单瞅瞅并验证一下上面的内容。

```sql
psql (16.10)
Type "help" for help.

postgres=# -- 创建测试表
postgres=# CREATE UNLOGGED TABLE test_unlogged_table(a int);
CREATE TABLE
postgres=# -- 查看测试表信息 获取relfilenode
postgres=# SELECT * FROM PG_CLASS WHERE relname = 'test_unlogged_table' \gx
-[ RECORD 1 ]-------+--------------------
oid                 | 157241
relname             | test_unlogged_table
relnamespace        | 2200
reltype             | 157243
reloftype           | 0
relowner            | 10
relam               | 2
relfilenode         | 157241
reltablespace       | 0
relpages            | 0
reltuples           | -1
relallvisible       | 0
reltoastrelid       | 0
relhasindex         | f
relisshared         | f
relpersistence      | u
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
relfrozenxid        | 7288
relminmxid          | 1
relacl              | 
reloptions          | 
relpartbound        | 

postgres=# -- 查看当前数据库信息 获取对应的oid
postgres=# SELECT * FROM PG_DATABASE WHERE datname = current_database() \gx
-[ RECORD 1 ]--+-------------------------------------
oid            | 5
datname        | postgres
datdba         | 10
encoding       | 6
datlocprovider | c
datistemplate  | f
datallowconn   | t
datconnlimit   | -1
datfrozenxid   | 722
datminmxid     | 1
dattablespace  | 1663
datcollate     | zh_CN.UTF-8
datctype       | zh_CN.UTF-8
daticulocale   | 
daticurules    | 
datcollversion | 2.39
datacl         | {=Tc/postgres,postgres=CTc/postgres}

postgres=# \! ls $PGDATA/base/5/157241* -al
-rw------- 1 postgres postgres 0  9月  2 15:43 /data/16/base/5/157241
-rw------- 1 postgres postgres 0  9月  2 15:43 /data/16/base/5/157241_init
postgres=# -- 插入一行数据
postgres=# INSERT INTO test_unlogged_table values(1);
INSERT 0 1
postgres=# -- VACUUM
postgres=# VACUUM test_unlogged_table;
VACUUM
postgres=# -- 查看所有分支文件
postgres=# \! ls $PGDATA/base/5/157241* -al
-rw------- 1 postgres postgres  8192  9月  2 15:44 /data/16/base/5/157241
-rw------- 1 postgres postgres 24576  9月  2 15:44 /data/16/base/5/157241_fsm
-rw------- 1 postgres postgres     0  9月  2 15:43 /data/16/base/5/157241_init
-rw------- 1 postgres postgres  8192  9月  2 15:44 /data/16/base/5/157241_vm
postgres=# 
```



# 瞅瞅CREATA TABLE代码逻辑

词法语法略过，只简单介绍部分重要的函数或代码片段

## 获取relfilenode和表的oid

获取relfilenode和表的oid，调用堆栈就不展示了，对应的函数GetNewRelFileNumber具体逻辑如下：

```sql
RelFileNumber
GetNewRelFileNumber(Oid reltablespace, Relation pg_class, char relpersistence)
{
	RelFileLocatorBackend rlocator;
	char	   *rpath;
	bool		collides;
	BackendId	backend;

	/*
	 * If we ever get here during pg_upgrade, there's something wrong; all
	 * relfilenumber assignments during a binary-upgrade run should be
	 * determined by commands in the dump script.
	 */
	Assert(!IsBinaryUpgrade);

	switch (relpersistence)
	{
		case RELPERSISTENCE_TEMP:
			backend = BackendIdForTempRelations();
			break;
		case RELPERSISTENCE_UNLOGGED:
		case RELPERSISTENCE_PERMANENT:
			backend = InvalidBackendId;
			break;
		default:
			elog(ERROR, "invalid relpersistence: %c", relpersistence);
			return InvalidRelFileNumber;	/* placate compiler */
	}

	/* This logic should match RelationInitPhysicalAddr */
	rlocator.locator.spcOid = reltablespace ? reltablespace : MyDatabaseTableSpace;
	rlocator.locator.dbOid =
		(rlocator.locator.spcOid == GLOBALTABLESPACE_OID) ?
		InvalidOid : MyDatabaseId;

	/*
	 * The relpath will vary based on the backend ID, so we must initialize
	 * that properly here to make sure that any collisions based on filename
	 * are properly detected.
	 */
	rlocator.backend = backend;

	do
	{
		CHECK_FOR_INTERRUPTS();

		/* Generate the OID */
		if (pg_class)
			rlocator.locator.relNumber = GetNewOidWithIndex(pg_class, ClassOidIndexId,
															Anum_pg_class_oid);
		else
			rlocator.locator.relNumber = GetNewObjectId();

		/* Check for existing file of same name */
		rpath = relpath(rlocator, MAIN_FORKNUM);

		if (access(rpath, F_OK) == 0)
		{
			/* definite collision */
			collides = true;
		}
		else
		{
			/*
			 * Here we have a little bit of a dilemma: if errno is something
			 * other than ENOENT, should we declare a collision and loop? In
			 * practice it seems best to go ahead regardless of the errno.  If
			 * there is a colliding file we will get an smgr failure when we
			 * attempt to create the new relation file.
			 */
			collides = false;
		}

		pfree(rpath);
	} while (collides);

	return rlocator.locator.relNumber;
}
```

按照生成的oid，生成相对的文件路径并检查对应的文件是否存在

<img width="980" height="197" alt="Image" src="https://github.com/user-attachments/assets/bf01c9dd-ba8c-4464-9c30-4223d4c0a8a2" />


## 分支文件创建

分支文件创建对应的函数heapam\_relation\_set\_new\_filelocator逻辑如下：

```c
static void
heapam_relation_set_new_filelocator(Relation rel,
									const RelFileLocator *newrlocator,
									char persistence,
									TransactionId *freezeXid,
									MultiXactId *minmulti)
{
	SMgrRelation srel;

	/*
	 * Initialize to the minimum XID that could put tuples in the table. We
	 * know that no xacts older than RecentXmin are still running, so that
	 * will do.
	 */
	*freezeXid = RecentXmin;

	/*
	 * Similarly, initialize the minimum Multixact to the first value that
	 * could possibly be stored in tuples in the table.  Running transactions
	 * could reuse values from their local cache, so we are careful to
	 * consider all currently running multis.
	 *
	 * XXX this could be refined further, but is it worth the hassle?
	 */
	*minmulti = GetOldestMultiXactId();
    // 会在此函数内部创建主分支
	srel = RelationCreateStorage(*newrlocator, persistence, true);

	/*
	 * If required, set up an init fork for an unlogged table so that it can
	 * be correctly reinitialized on restart.  An immediate sync is required
	 * even if the page has been logged, because the write did not go through
	 * shared_buffers and therefore a concurrent checkpoint may have moved the
	 * redo pointer past our xlog record.  Recovery may as well remove it
	 * while replaying, for example, XLOG_DBASE_CREATE* or XLOG_TBLSPC_CREATE
	 * record. Therefore, logging is necessary even if wal_level=minimal.
	 */
	if (persistence == RELPERSISTENCE_UNLOGGED)
	{
		Assert(rel->rd_rel->relkind == RELKIND_RELATION ||
			   rel->rd_rel->relkind == RELKIND_MATVIEW ||
			   rel->rd_rel->relkind == RELKIND_TOASTVALUE);
		// 为UNLOGGED TABLE创建初始化分支
		smgrcreate(srel, INIT_FORKNUM, false);
		log_smgrcreate(newrlocator, INIT_FORKNUM);
		smgrimmedsync(srel, INIT_FORKNUM);
	}

	smgrclose(srel);
}
```

RelationCreateStorage的函数逻辑如下：

```c
SMgrRelation
RelationCreateStorage(RelFileLocator rlocator, char relpersistence,
					  bool register_delete)
{
	SMgrRelation srel;
	BackendId	backend;
	bool		needs_wal;

	Assert(!IsInParallelMode());	/* couldn't update pendingSyncHash */

	switch (relpersistence)
	{
		case RELPERSISTENCE_TEMP:
			backend = BackendIdForTempRelations();
			needs_wal = false;
			break;
		case RELPERSISTENCE_UNLOGGED:
			backend = InvalidBackendId;
			needs_wal = false;
			break;
		case RELPERSISTENCE_PERMANENT:
			backend = InvalidBackendId;
			needs_wal = true;
			break;
		default:
			elog(ERROR, "invalid relpersistence: %c", relpersistence);
			return NULL;		/* placate compiler */
	}

	srel = smgropen(rlocator, backend);
    //创建主分支文件
	smgrcreate(srel, MAIN_FORKNUM, false);

	if (needs_wal)
		log_smgrcreate(&srel->smgr_rlocator.locator, MAIN_FORKNUM);

	/*
	 * Add the relation to the list of stuff to delete at abort, if we are
	 * asked to do so.
	 */
	if (register_delete)
	{
		PendingRelDelete *pending;

		pending = (PendingRelDelete *)
			MemoryContextAlloc(TopMemoryContext, sizeof(PendingRelDelete));
		pending->rlocator = rlocator;
		pending->backend = backend;
		pending->atCommit = false;	/* delete if abort */
		pending->nestLevel = GetCurrentTransactionNestLevel();
		pending->next = pendingDeletes;
		pendingDeletes = pending;
	}

	if (relpersistence == RELPERSISTENCE_PERMANENT && !XLogIsNeeded())
	{
		Assert(backend == InvalidBackendId);
		AddPendingSync(&rlocator);
	}

	return srel;
}
```

涉及到外存管理那块太过细节的内容就不在此处展示了。

## 其他工作
在完成了文件创建之后，还有一些工作需要处理，比方说维护元数据（将相关信息插入pg*class，将表中的字段信息插入值pg*\_attribute，相关依赖信息）、创建相关对象（同名数据类型以及对应的数组类型）和TOAST。

heap\_create\_with\_catalog部分代码片段：

```c
/*
	 * Decide whether to create a pg_type entry for the relation's rowtype.
	 * These types are made except where the use of a relation as such is an
	 * implementation detail: toast tables, sequences and indexes.
	 */
	if (!(relkind == RELKIND_SEQUENCE ||
		  relkind == RELKIND_TOASTVALUE ||
		  relkind == RELKIND_INDEX ||
		  relkind == RELKIND_PARTITIONED_INDEX))
	{
		Oid			new_array_oid;
		ObjectAddress new_type_addr;
		char	   *relarrayname;

		/*
		 * We'll make an array over the composite type, too.  For largely
		 * historical reasons, the array type's OID is assigned first.
		 */
		new_array_oid = AssignTypeArrayOid();

		/*
		 * Make the pg_type entry for the composite type.  The OID of the
		 * composite type can be preselected by the caller, but if reltypeid
		 * is InvalidOid, we'll generate a new OID for it.
		 *
		 * NOTE: we could get a unique-index failure here, in case someone
		 * else is creating the same type name in parallel but hadn't
		 * committed yet when we checked for a duplicate name above.
		 */
		new_type_addr = AddNewRelationType(relname,
										   relnamespace,
										   relid,
										   relkind,
										   ownerid,
										   reltypeid,
										   new_array_oid);
		new_type_oid = new_type_addr.objectId;
		if (typaddress)
			*typaddress = new_type_addr;

		/* Now create the array type. */
		relarrayname = makeArrayTypeName(relname, relnamespace);

		TypeCreate(new_array_oid,	/* force the type's OID to this */
				   relarrayname,	/* Array type name */
				   relnamespace,	/* Same namespace as parent */
				   InvalidOid,	/* Not composite, no relationOid */
				   0,			/* relkind, also N/A here */
				   ownerid,		/* owner's ID */
				   -1,			/* Internal size (varlena) */
				   TYPTYPE_BASE,	/* Not composite - typelem is */
				   TYPCATEGORY_ARRAY,	/* type-category (array) */
				   false,		/* array types are never preferred */
				   DEFAULT_TYPDELIM,	/* default array delimiter */
				   F_ARRAY_IN,	/* array input proc */
				   F_ARRAY_OUT, /* array output proc */
				   F_ARRAY_RECV,	/* array recv (bin) proc */
				   F_ARRAY_SEND,	/* array send (bin) proc */
				   InvalidOid,	/* typmodin procedure - none */
				   InvalidOid,	/* typmodout procedure - none */
				   F_ARRAY_TYPANALYZE,	/* array analyze procedure */
				   F_ARRAY_SUBSCRIPT_HANDLER,	/* array subscript procedure */
				   new_type_oid,	/* array element type - the rowtype */
				   true,		/* yes, this is an array type */
				   InvalidOid,	/* this has no array type */
				   InvalidOid,	/* domain base type - irrelevant */
				   NULL,		/* default value - none */
				   NULL,		/* default binary representation */
				   false,		/* passed by reference */
				   TYPALIGN_DOUBLE, /* alignment - must be the largest! */
				   TYPSTORAGE_EXTENDED, /* fully TOASTable */
				   -1,			/* typmod */
				   0,			/* array dimensions for typBaseType */
				   false,		/* Type NOT NULL */
				   InvalidOid); /* rowtypes never have a collation */

		pfree(relarrayname);
	}
	else
	{
		/* Caller should not be expecting a type to be created. */
		Assert(reltypeid == InvalidOid);
		Assert(typaddress == NULL);

		new_type_oid = InvalidOid;
	}

	/*
	 * now create an entry in pg_class for the relation.
	 *
	 * NOTE: we could get a unique-index failure here, in case someone else is
	 * creating the same relation name in parallel but hadn't committed yet
	 * when we checked for a duplicate name above.
	 */
	AddNewRelationTuple(pg_class_desc,
						new_rel_desc,
						relid,
						new_type_oid,
						reloftypeid,
						ownerid,
						relkind,
						relfrozenxid,
						relminmxid,
						PointerGetDatum(relacl),
						reloptions);

	/*
	 * now add tuples to pg_attribute for the attributes in our new relation.
	 */
	AddNewAttributeTuples(relid, new_rel_desc->rd_att, relkind);

	/*
	 * Make a dependency link to force the relation to be deleted if its
	 * namespace is.  Also make a dependency link to its owner, as well as
	 * dependencies for any roles mentioned in the default ACL.
	 *
	 * For composite types, these dependencies are tracked for the pg_type
	 * entry, so we needn't record them here.  Likewise, TOAST tables don't
	 * need a namespace dependency (they live in a pinned namespace) nor an
	 * owner dependency (they depend indirectly through the parent table), nor
	 * should they have any ACL entries.  The same applies for extension
	 * dependencies.
	 *
	 * Also, skip this in bootstrap mode, since we don't make dependencies
	 * while bootstrapping.
	 */
	if (relkind != RELKIND_COMPOSITE_TYPE &&
		relkind != RELKIND_TOASTVALUE &&
		!IsBootstrapProcessingMode())
	{
		ObjectAddress myself,
					referenced;
		ObjectAddresses *addrs;

		ObjectAddressSet(myself, RelationRelationId, relid);

		recordDependencyOnOwner(RelationRelationId, relid, ownerid);

		recordDependencyOnNewAcl(RelationRelationId, relid, 0, ownerid, relacl);

		recordDependencyOnCurrentExtension(&myself, false);

		addrs = new_object_addresses();

		ObjectAddressSet(referenced, NamespaceRelationId, relnamespace);
		add_exact_object_address(&referenced, addrs);

		if (reloftypeid)
		{
			ObjectAddressSet(referenced, TypeRelationId, reloftypeid);
			add_exact_object_address(&referenced, addrs);
		}

		/*
		 * Make a dependency link to force the relation to be deleted if its
		 * access method is.
		 *
		 * No need to add an explicit dependency for the toast table, as the
		 * main table depends on it.
		 */
		if (RELKIND_HAS_TABLE_AM(relkind) && relkind != RELKIND_TOASTVALUE)
		{
			ObjectAddressSet(referenced, AccessMethodRelationId, accessmtd);
			add_exact_object_address(&referenced, addrs);
		}

		record_object_address_dependencies(&myself, addrs, DEPENDENCY_NORMAL);
		free_object_addresses(addrs);
	}

	/* Post creation hook for new relation */
	InvokeObjectPostCreateHookArg(RelationRelationId, relid, 0, is_internal);
```

ProcessUtilitySlow部分代码片段

```c
if (IsA(stmt, CreateStmt))
{
  CreateStmt *cstmt = (CreateStmt *) stmt;
  Datum   toast_options;
  static char *validnsps[] = HEAP_RELOPT_NAMESPACES;

  /* Remember transformed RangeVar for LIKE */
  table_rv = cstmt->relation;

  /* Create the table itself */
  address = DefineRelation(cstmt,
               RELKIND_RELATION,
               InvalidOid, NULL,
               queryString);
  EventTriggerCollectSimpleCommand(address,
                   secondaryObject,
                   stmt);

  /*
   * Let NewRelationCreateToastTable decide if this
   * one needs a secondary relation too.
   */
  CommandCounterIncrement();

  /*
   * parse and validate reloptions for the toast
   * table
   */
  toast_options = transformRelOptions((Datum) 0,
                    cstmt->options,
                    "toast",
                    validnsps,
                    true,
                    false);
  (void) heap_reloptions(RELKIND_TOASTVALUE,
               toast_options,
               true);

  NewRelationCreateToastTable(address.objectId,
                toast_options);
}
```

还有很多细节的内容，感兴趣的同学可以自己动手调试着玩玩。

## 验证一下最终结果

```sql
postgres=# SELECT pg_relation_filepath('test_unlogged_table');
 pg_relation_filepath 
----------------------
 base/5/157247
(1 row)

postgres=# \! ls $PGDATA/base/5/157247* -al
-rw------- 1 postgres postgres 0  9月  2 17:41 /data/16/base/5/157247
-rw------- 1 postgres postgres 0  9月  2 17:41 /data/16/base/5/157247_init
postgres=# SELECT * FROM PG_CLASS WHERE relname = 'test_unlogged_table' \gx
-[ RECORD 1 ]-------+--------------------
oid                 | 157247
relname             | test_unlogged_table
relnamespace        | 2200
reltype             | 157249
reloftype           | 0
relowner            | 10
relam               | 2
relfilenode         | 157247
reltablespace       | 0
relpages            | 0
reltuples           | -1
relallvisible       | 0
reltoastrelid       | 0
relhasindex         | f
relisshared         | f
relpersistence      | u
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
relfrozenxid        | 7293
relminmxid          | 1
relacl              | 
reloptions          | 
relpartbound        | 

postgres=# 
```
# 玩一玩初始分支

让我们来尝试着删除UNLOGGED TABLE的主分支文件，然后重启，看看是否能够进行查询

```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# -- 创建测试表
postgres=# CREATE UNLOGGED TABLE test_unlogged_table(a int);
CREATE TABLE
postgres=# -- 获取主分支文件路径  
postgres=# SELECT pg_relation_filepath('test_unlogged_table');
 pg_relation_filepath 
----------------------
 base/5/173637
(1 row)

postgres=# \q
postgres@zxm-VMware-Virtual-Platform:~$ rm $PGDATA/base/5/173637
postgres@zxm-VMware-Virtual-Platform:~$ ls $PGDATA/base/5/173637* -al 
-rw------- 1 postgres postgres 0  9月  2 19:17 /data/16/base/5/173637_init
postgres@zxm-VMware-Virtual-Platform:~$ pg_ctl restart
waiting for server to shut down.... done
server stopped
waiting for server to start....2025-09-02 19:18:15.616 CST [12493] LOG:  redirecting log output to logging collector process
2025-09-02 19:18:15.616 CST [12493] HINT:  Future log output will appear in directory "log".
 done
server started
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# select * from test_unlogged_table;
 a 
---
(0 rows)

postgres=# SELECT pg_relation_filepath('test_unlogged_table');
 pg_relation_filepath 
----------------------
 base/5/173637
(1 row)

postgres=# 
```

可以看到是可以的，如果我们在退出前，在运行一下checkpoint呢？

```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# -- 创建测试表
postgres=# CREATE UNLOGGED TABLE test_unlogged_table(a int);
CREATE TABLE
postgres=# -- 获取主分支文件路径  
postgres=# SELECT pg_relation_filepath('test_unlogged_table');
 pg_relation_filepath 
----------------------
 base/5/181829
(1 row)

postgres=# checkpoint;
CHECKPOINT
postgres=# \q
postgres@zxm-VMware-Virtual-Platform:~$ rm $PGDATA/base/5/181829
postgres@zxm-VMware-Virtual-Platform:~$ ls $PGDATA/base/5/181829* -al
-rw------- 1 postgres postgres 0  9月  2 19:19 /data/16/base/5/181829_init
postgres@zxm-VMware-Virtual-Platform:~$ pg_ctl restart
waiting for server to shut down.... done
server stopped
waiting for server to start....2025-09-02 19:19:59.113 CST [12510] LOG:  redirecting log output to logging collector process
2025-09-02 19:19:59.113 CST [12510] HINT:  Future log output will appear in directory "log".
 done
server started
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# select * from test_unlogged_table;
ERROR:  could not open file "base/5/181829": 没有那个文件或目录
postgres=# 
```

可以看到就不行了，这是为什么呢？
关键在于**Recovery**，这里就不细说了，可以考虑看看代码或者将日志级别调低查看一下，就能明白了。
同时那现在怎么办呢？查询不了了
这里有两个解决方法，一个就是kill一下会话，触发异常，另一个就是手动创建一下文件即可

```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# select * from test_unlogged_table;
ERROR:  could not open file "base/5/181829": 没有那个文件或目录
postgres=# \! touch $PGDATA/base/5/181829
postgres=# select * from test_unlogged_table;
 a 
---
(0 rows)

postgres=# 
```

相关"恢复"代码如下，其实就是unlink，然后create，感兴趣的朋友可以看看

```sql
static void
ResetUnloggedRelationsInDbspaceDir(const char *dbspacedirname, int op)
{
	DIR		   *dbspace_dir;
	struct dirent *de;
	char		rm_path[MAXPGPATH * 2];

	/* Caller must specify at least one operation. */
	Assert((op & (UNLOGGED_RELATION_CLEANUP | UNLOGGED_RELATION_INIT)) != 0);

	/*
	 * Cleanup is a two-pass operation.  First, we go through and identify all
	 * the files with init forks.  Then, we go through again and nuke
	 * everything with the same OID except the init fork.
	 */
	if ((op & UNLOGGED_RELATION_CLEANUP) != 0)
	{
		HTAB	   *hash;
		HASHCTL		ctl;

		/*
		 * It's possible that someone could create a ton of unlogged relations
		 * in the same database & tablespace, so we'd better use a hash table
		 * rather than an array or linked list to keep track of which files
		 * need to be reset.  Otherwise, this cleanup operation would be
		 * O(n^2).
		 */
		ctl.keysize = sizeof(Oid);
		ctl.entrysize = sizeof(unlogged_relation_entry);
		ctl.hcxt = CurrentMemoryContext;
		hash = hash_create("unlogged relation OIDs", 32, &ctl,
						   HASH_ELEM | HASH_BLOBS | HASH_CONTEXT);

		/* Scan the directory. */
		dbspace_dir = AllocateDir(dbspacedirname);
		while ((de = ReadDir(dbspace_dir, dbspacedirname)) != NULL)
		{
			ForkNumber	forkNum;
			int			relnumchars;
			unlogged_relation_entry ent;

			/* Skip anything that doesn't look like a relation data file. */
			if (!parse_filename_for_nontemp_relation(de->d_name, &relnumchars,
													 &forkNum))
				continue;

			/* Also skip it unless this is the init fork. */
			if (forkNum != INIT_FORKNUM)
				continue;

			/*
			 * Put the OID portion of the name into the hash table, if it
			 * isn't already.
			 */
			ent.reloid = atooid(de->d_name);
			(void) hash_search(hash, &ent, HASH_ENTER, NULL);
		}

		/* Done with the first pass. */
		FreeDir(dbspace_dir);

		/*
		 * If we didn't find any init forks, there's no point in continuing;
		 * we can bail out now.
		 */
		if (hash_get_num_entries(hash) == 0)
		{
			hash_destroy(hash);
			return;
		}

		/*
		 * Now, make a second pass and remove anything that matches.
		 */
		dbspace_dir = AllocateDir(dbspacedirname);
		while ((de = ReadDir(dbspace_dir, dbspacedirname)) != NULL)
		{
			ForkNumber	forkNum;
			int			relnumchars;
			unlogged_relation_entry ent;

			/* Skip anything that doesn't look like a relation data file. */
			if (!parse_filename_for_nontemp_relation(de->d_name, &relnumchars,
													 &forkNum))
				continue;

			/* We never remove the init fork. */
			if (forkNum == INIT_FORKNUM)
				continue;

			/*
			 * See whether the OID portion of the name shows up in the hash
			 * table.  If so, nuke it!
			 */
			ent.reloid = atooid(de->d_name);
			if (hash_search(hash, &ent, HASH_FIND, NULL))
			{
				snprintf(rm_path, sizeof(rm_path), "%s/%s",
						 dbspacedirname, de->d_name);
				if (unlink(rm_path) < 0)
					ereport(ERROR,
							(errcode_for_file_access(),
							 errmsg("could not remove file \"%s\": %m",
									rm_path)));
				else
					elog(DEBUG2, "unlinked file \"%s\"", rm_path);
			}
		}

		/* Cleanup is complete. */
		FreeDir(dbspace_dir);
		hash_destroy(hash);
	}

	/*
	 * Initialization happens after cleanup is complete: we copy each init
	 * fork file to the corresponding main fork file.  Note that if we are
	 * asked to do both cleanup and init, we may never get here: if the
	 * cleanup code determines that there are no init forks in this dbspace,
	 * it will return before we get to this point.
	 */
	if ((op & UNLOGGED_RELATION_INIT) != 0)
	{
		/* Scan the directory. */
		dbspace_dir = AllocateDir(dbspacedirname);
		while ((de = ReadDir(dbspace_dir, dbspacedirname)) != NULL)
		{
			ForkNumber	forkNum;
			int			relnumchars;
			char		relnumbuf[OIDCHARS + 1];
			char		srcpath[MAXPGPATH * 2];
			char		dstpath[MAXPGPATH];

			/* Skip anything that doesn't look like a relation data file. */
			if (!parse_filename_for_nontemp_relation(de->d_name, &relnumchars,
													 &forkNum))
				continue;

			/* Also skip it unless this is the init fork. */
			if (forkNum != INIT_FORKNUM)
				continue;

			/* Construct source pathname. */
			snprintf(srcpath, sizeof(srcpath), "%s/%s",
					 dbspacedirname, de->d_name);

			/* Construct destination pathname. */
			memcpy(relnumbuf, de->d_name, relnumchars);
			relnumbuf[relnumchars] = '\0';
			snprintf(dstpath, sizeof(dstpath), "%s/%s%s",
					 dbspacedirname, relnumbuf, de->d_name + relnumchars + 1 +
					 strlen(forkNames[INIT_FORKNUM]));

			/* OK, we're ready to perform the actual copy. */
			elog(DEBUG2, "copying %s to %s", srcpath, dstpath);
			copy_file(srcpath, dstpath);
		}

		FreeDir(dbspace_dir);

		/*
		 * copy_file() above has already called pg_flush_data() on the files
		 * it created. Now we need to fsync those files, because a checkpoint
		 * won't do it for us while we're in recovery. We do this in a
		 * separate pass to allow the kernel to perform all the flushes
		 * (especially the metadata ones) at once.
		 */
		dbspace_dir = AllocateDir(dbspacedirname);
		while ((de = ReadDir(dbspace_dir, dbspacedirname)) != NULL)
		{
			ForkNumber	forkNum;
			int			relnumchars;
			char		relnumbuf[OIDCHARS + 1];
			char		mainpath[MAXPGPATH];

			/* Skip anything that doesn't look like a relation data file. */
			if (!parse_filename_for_nontemp_relation(de->d_name, &relnumchars,
													 &forkNum))
				continue;

			/* Also skip it unless this is the init fork. */
			if (forkNum != INIT_FORKNUM)
				continue;

			/* Construct main fork pathname. */
			memcpy(relnumbuf, de->d_name, relnumchars);
			relnumbuf[relnumchars] = '\0';
			snprintf(mainpath, sizeof(mainpath), "%s/%s%s",
					 dbspacedirname, relnumbuf, de->d_name + relnumchars + 1 +
					 strlen(forkNames[INIT_FORKNUM]));

			fsync_fname(mainpath, false);
		}

		FreeDir(dbspace_dir);

		/*
		 * Lastly, fsync the database directory itself, ensuring the
		 * filesystem remembers the file creations and deletions we've done.
		 * We don't bother with this during a call that does only
		 * UNLOGGED_RELATION_CLEANUP, because if recovery crashes before we
		 * get to doing UNLOGGED_RELATION_INIT, we'll redo the cleanup step
		 * too at the next startup attempt.
		 */
		fsync_fname(dbspacedirname, true);
	}
}
```
