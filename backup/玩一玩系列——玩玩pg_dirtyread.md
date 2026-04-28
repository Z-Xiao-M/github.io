一、前言
====

很早其实就知道有这么一个插件存在，叫做pg\_dirtyread，就是一直没去上手玩玩，趁着有空，写点东西。

pg\_dirtyread项目地址[https://github.com/df7cb/pg\_dirtyread](https://github.com/df7cb/pg_dirtyread) ，项目介绍是

> Read dead but unvacuumed tuples from a PostgreSQL relation

其实更加准确的说法是读取表中所有的元组、包括未清理的死元组。

因此如果要是不小心删除一部分表中的数据，及时停止PG的自动回收，使用这个插件有可能还有机会把删除的数据恢复回来。

简单介绍一下安装和使用。

  

二、编译安装
======

我个人一般喜欢源码安装，pg\_dirtyread的编译安装也很简单，运行如下命令即可

    cd contrib/
    git clone https://github.com/df7cb/pg_dirtyread.git
    cd pg_dirtyread/
    make && make install

  

三、上手体验
======

编译安装之后、创建拓展，开始上手体验

    [postgres@iZuf6hwo0wgeev4dvua4csZ pg_dirtyread]$ psql
    psql (16.6)
    Type "help" for help.
    
    postgres=# create extension pg_dirtyread;
    CREATE EXTENSION
    postgres=# \dx
                              List of installed extensions
         Name     | Version |   Schema   |               Description                
    --------------+---------+------------+------------------------------------------
     pg_dirtyread | 2       | public     | Read dead but unvacuumed rows from table
     plpgsql      | 1.0     | pg_catalog | PL/pgSQL procedural language
    (2 rows)

这个数字2多多少少显得有点突兀，但是无伤大雅。进入正题，通过源码我们可以看到，pg\_dirtyread提供了一个返回多行的函数

    CREATE FUNCTION pg_dirtyread(regclass)
    RETURNS SETOF record
    AS 'MODULE_PATHNAME'
    LANGUAGE C;

因此接下来就围绕着pg\_dirtyread这个函数展开。项目首页提供了一个非常简要的示例，让我们来复现一下。

    postgres=# CREATE TABLE foo (bar bigint, baz text);
    CREATE TABLE
    postgres=# -- 关闭自动回收
    postgres=# ALTER TABLE foo SET (autovacuum_enabled = false, toast.autovacuum_enabled = false);
    ALTER TABLE
    postgres=# -- 插入两行数据
    postgres=# INSERT INTO foo VALUES (1, 'Test'), (2, 'New Test');
    INSERT 0 2
    postgres=# -- 删除一行数据
    postgres=# DELETE FROM foo WHERE bar = 1;
    DELETE 1
    postgres=# -- 通过pg_dirtyread查看表中元组情况
    postgres=# SELECT * FROM pg_dirtyread('foo') as t(bar bigint, baz text);
     bar |   baz    
    -----+----------
       1 | Test
       2 | New Test
    (2 rows)
    
    postgres=# -- 直接查看表中数据
    postgres=# SELECT * FROM foo;
     bar |   baz    
    -----+----------
       2 | New Test
    (1 row)

可以看到即使数据被删除，也就是**已经被标记成了死元组，再没被回收的前提之下**，我们还是能够通过插件提供的功能，看到相应的数据。因此也可以用这个来尝试恢复被删除的数据。

如果要是回收了，或者TRUNCATE TABLE那就莫得办法了。

    postgres=# INSERT INTO foo SELECT * FROM pg_dirtyread('foo') as t(bar bigint, baz text) where bar = 1;
    INSERT 0 1
    postgres=# SELECT * FROM foo;
     bar |   baz    
    -----+----------
       2 | New Test
       1 | Test
    (2 rows)

值得注意到的是，当我们使用pg\_dirtyread函数时，需要提供带查询的表中相应的字段或系统字段如（tableoid、ctid、xmin、xmax、cmin、cmax等等），这在内部是通过字段名匹配的，乱填的字段名会报错。

    postgres=# SELECT * FROM pg_dirtyread('foo') as t(barr bigint, baz text);   -- barr 
    ERROR:  Error converting tuple descriptors!
    DETAIL:  Attribute "barr" does not exist in type foo.
    postgres=# SELECT * FROM pg_dirtyread('foo')
    postgres-#       AS t(tableoid oid, ctid tid, xmin xid, xmax xid, cmin cid, cmax cid, dead boolean,
    postgres(#            bar bigint, baz text);
     tableoid | ctid  | xmin | xmax | cmin | cmax | dead | bar |   baz    
    ----------+-------+------+------+------+------+------+-----+----------
        20374 | (0,1) | 2581 | 2582 |    0 |    0 | t    |   1 | Test
        20374 | (0,2) | 2581 |    0 |    0 |    0 | f    |   2 | New Test
        20374 | (0,3) | 2583 |    0 |    0 |    0 | f    |   1 | Test
    (3 rows)
    
    postgres=# SELECT * FROM foo;
     bar |   baz    
    -----+----------
       2 | New Test
       1 | Test
    (2 rows)

但是，我是说但是，你要是说：”我就是要整点花活，写点不一样的字段，可不可以？“

那还有一种比较冷门的用法，就是当表中的字段被删除之后，使用pg\_dirtyread访问元组数据，此时被删除的字段可以用dropped\_xxx来标识。

    postgres=# CREATE TABLE bar (
    postgres(#   id int,
    postgres(#   a int,
    postgres(#   b bigint,
    postgres(#   c text,
    postgres(#   d varchar(10),
    postgres(#   e boolean,
    postgres(#   f bigint[]
    postgres(# );
    CREATE TABLE
    postgres=# ALTER TABLE bar SET (autovacuum_enabled = false, toast.autovacuum_enabled = false);
    ALTER TABLE
    postgres=# INSERT INTO bar VALUES (1, 2, 3, '4', '5', true, '{7}');
    INSERT 0 1
    postgres=# -- 删除表中的字段
    postgres=# ALTER TABLE bar DROP COLUMN a, DROP COLUMN b, DROP COLUMN c, DROP COLUMN d, DROP COLUMN e, DROP COLUMN f;
    ALTER TABLE
    postgres=# -- 使用dropped_ 来表示字段名
    postgres=# SELECT * FROM pg_dirtyread('bar')
    postgres-#   bar(id int, dropped_2 int, dropped_3 bigint, dropped_4 text,
    postgres(#       dropped_5 varchar(10), dropped_6 boolean, dropped_7 bigint[]);
     id | dropped_2 | dropped_3 | dropped_4 | dropped_5 | dropped_6 | dropped_7 
    ----+-----------+-----------+-----------+-----------+-----------+-----------
      1 |         2 |         3 | 4         | 5         | t         | {7}
    (1 row)

  

四、实现原理分析
========

来简单的进行分析一下，pg\_dirtyread是如何实现的。这个插件很简单就一个函数，我们再将这个函数的实现再简化一下，保留一些主干，就得到了如下的伪代码。

    Datum
    pg_dirtyread(PG_FUNCTION_ARGS)
    {
        if (SRF_IS_FIRSTCALL())
        {
            // 初次调用函数 做一些初始化的动作
            // 获取一些信息 比如获取输入的表的OID 打开表
            relid = PG_GETARG_OID(0);
            heap_open(relid, AccessShareLock);
            
            // 根据输入的字段信息 生成相应的映射关系
            usr_ctx->map = dirtyread_convert_tuples_by_name(usr_ctx->reltupdesc, ...
            // 设定扫描表的策略 
            usr_ctx->scan = heap_beginscan(usr_ctx->rel, SnapshotAny, 0, NULL ...
        }
    
        // 每次需要执行的动作 尝试扫描表，获取一个表中的元组 
        if ((tuplein = heap_getnext(usr_ctx->scan, ForwardScanDirection)) != NULL)
        {
            // 将获取的到元组数据 根据map的映射关系 填充生成待返回的元组 然后return 
            tuplein = dirtyread_do_convert_tuple(tuplein, usr_ctx->map, usr_ctx->oldest_xmin);
            ...
        }
        else
        {
            // 扫描到表的最后了 没有下一个元组数据量 做一些资源释放和清理工作
            heap_endscan(usr_ctx->scan);
            heap_close ...
        }
    }

关于字段名匹配的函数dirtyread\_convert\_tuples\_by\_name实际匹配调用的是dirtyread\_convert\_tuples\_by\_name\_map，简化一下就是

    AttrNumber *
    dirtyread_convert_tuples_by_name_map(TupleDesc indesc, TupleDesc outdesc, const char *msg)
    {
      for (i = 0; i < n; i++)
      {
        // ...
        for (j = 0; j < indesc->natts; j++)
        {
          Form_pg_attribute inatt = TupleDescAttr(indesc, j);
          // ...
          if (strcmp(attname, NameStr(inatt->attname)) == 0)
          {
            /* Found it, check type */
            // ...
          }
        }
    
        /* Check dropped columns */
        if (attrMap[i] == 0)
          if (strncmp(attname, "dropped_", sizeof("dropped_") - 1) == 0)
          {
            // ...
          }
    
        /* Check system columns */
        if (attrMap[i] == 0)
          for (j = 0; system_columns[j].attname; j++)
            if (strcmp(attname, system_columns[j].attname) == 0)
            {
             // ...
            }
      }
      // ...
    }

system\_columns的声明则是如下

    static const struct system_columns_t {
    	char	   *attname;
    	Oid			atttypid;
    	int32		atttypmod;
    	int			attnum;
    } system_columns[] = {
    	{ "ctid",     TIDOID,  -1, SelfItemPointerAttributeNumber },
    #if PG_VERSION_NUM < 120000
    	{ "oid",      OIDOID,  -1, ObjectIdAttributeNumber },
    #endif
    	{ "xmin",     XIDOID,  -1, MinTransactionIdAttributeNumber },
    	{ "cmin",     CIDOID,  -1, MinCommandIdAttributeNumber },
    	{ "xmax",     XIDOID,  -1, MaxTransactionIdAttributeNumber },
    	{ "cmax",     CIDOID,  -1, MaxCommandIdAttributeNumber },
    	{ "tableoid", OIDOID,  -1, TableOidAttributeNumber },
    	{ "dead",     BOOLOID, -1, DeadFakeAttributeNumber }, /* fake column to return HeapTupleIsSurelyDead */
    	{ 0 },
    };

简单无需多言~~~

  

五、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

玩一玩系列，原则上可能更多的针对PG的插件简单的使用，而不是插件实际的具体实现，用做个人记录，如果后续没有源码分析环节，敬请理解。

在文章发布之前，墨天轮的2024年度的原创评选结果已经新鲜出炉了。

混了个墨天轮2024年度作者评选参与奖，在此感谢一下小墨和墨天轮。

文章转载请联系，谢谢合作。