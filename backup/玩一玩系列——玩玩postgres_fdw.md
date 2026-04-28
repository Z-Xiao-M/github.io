一、前言
====

本文仅记录和分享一下关于PostgreSQL的FDW（外部数据包装器）的简单介绍和使用，此次不包含源码分析。

  

二、Foreign Data Wrapper
======================

外部数据包装器（FDW）是PostgreSQL的一个功能，负责从远程数据源获取数据，并将其返回给 PostgreSQL 执行器。

它允许您访问和查询存储在外部数据源中的数据，就像它们是本地 Postgres 表一样。

远程数据源可以是同构数据库，比如说PostgreSQL对应的FDW便是postgres\_fdw，

也可以是异构数据库，当远程数据源对应的是Oracle数据库时，便是oracle\_fdw，

当远程数据源对应的是mysql数据库时，便是mysql\_fdw,

当远程数据源对应的是duckdb时，便是李红艳老师写的duckdb\_fdw。

外部数据源也可以不是数据库，比方说想要更方便的查看PostgreSQL的日志文件内容，便可以使用file\_fdw。

当然还有很多，这里就不一一举例、"报菜名"了。

其实真的不得不感慨PostgreSQL的扩展能力，如果您也想要写一个FDW，可以参考[官方文档](https://www.postgresql.org/docs/16/fdwhandler.html)，PostgreSQL提供了一系列的函数接口，

按照需求去实现对应的接口就能够写出一个能用的fdw了。

  

三、简单使用
======

不同的fdw的使用，可能存在一些特定的设置，比方说Oracle\_fdw需要设置一些环境变量，如ORACLE\_HOME等等之类的。

这就需要按照对应的fdw的要求去做了，这里以postgres\_fdw为例，在我自己本地玩一玩，这里我就不引入别的环境了。

对于安装方面就没啥好说了，如果安装PG的时候，是下载源码安装。那么postgres\_fdw是直接附加在PG的源码里的，“三板斧”编译安装即可。

    cd contrib/postgres_fdw/
    make && make install

之后直接安装postgres\_fdw拓展即可，这里没啥好说的。

对于实际使用而言，也是简单的"三板斧"操作。

我们先来准备一个简单的环境，在我的本地存在一个foreign\_db数据库，数据库内部存在两张表（让大模型生成的）。这其中了一个字段多、另一个字段少，每个表里都存在一条数据。实际内容如下：

    foreign_db=# \dt
                    List of relations
     Schema |         Name         | Type  |  Owner   
    --------+----------------------+-------+----------
     public | twenty_columns_table | table | postgres
     public | two_columns_table    | table | postgres
    (2 rows)
    
    foreign_db=# \d+ two_columns_table
                                                Table "public.two_columns_table"
     Column |          Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
    --------+------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
     id     | integer                |           | not null |         | plain    |             |              | 
     name   | character varying(100) |           |          |         | extended |             |              | 
    Indexes:
        "two_columns_table_pkey" PRIMARY KEY, btree (id)
    Access method: heap
    
    foreign_db=# \d+ twenty_columns_table
                                              Table "public.twenty_columns_table"
     Column |         Type          | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
    --------+-----------------------+-----------+----------+---------+----------+-------------+--------------+-------------
     id     | integer               |           | not null |         | plain    |             |              | 
     col1   | character varying(50) |           |          |         | extended |             |              | 
     col2   | character varying(50) |           |          |         | extended |             |              | 
     col3   | character varying(50) |           |          |         | extended |             |              | 
     col4   | character varying(50) |           |          |         | extended |             |              | 
     col5   | character varying(50) |           |          |         | extended |             |              | 
     col6   | character varying(50) |           |          |         | extended |             |              | 
     col7   | character varying(50) |           |          |         | extended |             |              | 
     col8   | character varying(50) |           |          |         | extended |             |              | 
     col9   | character varying(50) |           |          |         | extended |             |              | 
     col10  | character varying(50) |           |          |         | extended |             |              | 
     col11  | character varying(50) |           |          |         | extended |             |              | 
     col12  | character varying(50) |           |          |         | extended |             |              | 
     col13  | character varying(50) |           |          |         | extended |             |              | 
     col14  | character varying(50) |           |          |         | extended |             |              | 
     col15  | character varying(50) |           |          |         | extended |             |              | 
     col16  | character varying(50) |           |          |         | extended |             |              | 
     col17  | character varying(50) |           |          |         | extended |             |              | 
     col18  | character varying(50) |           |          |         | extended |             |              | 
     col19  | character varying(50) |           |          |         | extended |             |              | 
     col20  | character varying(50) |           |          |         | extended |             |              | 
    Indexes:
        "twenty_columns_table_pkey" PRIMARY KEY, btree (id)
    Access method: heap
    
    foreign_db=# select * from two_columns_table;
     id |   name   
    ----+----------
      1 | John Doe
    (1 row)
    
    foreign_db=# select * from twenty_columns_table;
     id |  col1  |  col2  |  col3  |  col4  |  col5  |  col6  |  col7  |  col8  |  col9  |  col10  |  col11  |  col12  |  col13  |  col14  |  col15  |  col16  |  col17  |  col18  |  col19  
    |  col20  
    ----+--------+--------+--------+--------+--------+--------+--------+--------+--------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------
    +---------
      1 | Value1 | Value2 | Value3 | Value4 | Value5 | Value6 | Value7 | Value8 | Value9 | Value10 | Value11 | Value12 | Value13 | Value14 | Value15 | Value16 | Value17 | Value18 | Value19 
    | Value20
    (1 row)

接下来让我们在Postgres数据库中进行三板斧操作演示

1.  CREATE SERVER 创建外部服务器对象，提供远端信息。
    

    postgres=# \h CREATE SERVER
    Command:     CREATE SERVER
    Description: define a new foreign server
    Syntax:
    CREATE SERVER [ IF NOT EXISTS ] server_name [ TYPE 'server_type' ] [ VERSION 'server_version' ]
        FOREIGN DATA WRAPPER fdw_name
        [ OPTIONS ( option 'value' [, ... ] ) ]
    
    URL: https://www.postgresql.org/docs/16/sql-createserver.html
    
    postgres=# CREATE SERVER foreign_server FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host '127.0.0.1', port '19211', dbname 'foreign_db'); -- 根据实际来填
    CREATE SERVER
    

2. CREATE USER MAPPING 创建用户映射，以标识将在远程服务器上使用的角色。我这里就直接用同一个用户的信息好了。

    postgres=# \h CREATE USER MAPPING 
    Command:     CREATE USER MAPPING
    Description: define a new mapping of a user to a foreign server
    Syntax:
    CREATE USER MAPPING [ IF NOT EXISTS ] FOR { user_name | USER | CURRENT_ROLE | CURRENT_USER | PUBLIC }
        SERVER server_name
        [ OPTIONS ( option 'value' [ , ... ] ) ]
    
    URL: https://www.postgresql.org/docs/16/sql-createusermapping.html
    
    postgres=# CREATE USER MAPPING FOR CURRENT_USER SERVER foreign_server
    postgres-# OPTIONS (user 'postgres', password 'postgres');
    CREATE USER MAPPING
    

3.  CREATE FOREIGN TABLE 创建外部表，我们以字段较少的表为例，值得注意的是，外部表的表名可以不和远端的表名一致，但是表中字段信息要一致，

如果字段名称不能正确匹配上的话，将会爆出列不存在的错误。

    postgres=# \h CREATE FOREIGN TABLE
    Command:     CREATE FOREIGN TABLE
    Description: define a new foreign table
    Syntax:
    CREATE FOREIGN TABLE [ IF NOT EXISTS ] table_name ( [
      { column_name data_type [ OPTIONS ( option 'value' [, ... ] ) ] [ COLLATE collation ] [ column_constraint [ ... ] ]
        | table_constraint }
        [, ... ]
    ] )
    [ INHERITS ( parent_table [, ... ] ) ]
      SERVER server_name
    [ OPTIONS ( option 'value' [, ... ] ) ]
    
    CREATE FOREIGN TABLE [ IF NOT EXISTS ] table_name
      PARTITION OF parent_table [ (
      { column_name [ WITH OPTIONS ] [ column_constraint [ ... ] ]
        | table_constraint }
        [, ... ]
    ) ]
    { FOR VALUES partition_bound_spec | DEFAULT }
      SERVER server_name
    [ OPTIONS ( option 'value' [, ... ] ) ]
    
    where column_constraint is:
    
    [ CONSTRAINT constraint_name ]
    { NOT NULL |
      NULL |
      CHECK ( expression ) [ NO INHERIT ] |
      DEFAULT default_expr |
      GENERATED ALWAYS AS ( generation_expr ) STORED }
    
    and table_constraint is:
    
    [ CONSTRAINT constraint_name ]
    CHECK ( expression ) [ NO INHERIT ]
    
    and partition_bound_spec is:
    
    IN ( partition_bound_expr [, ...] ) |
    FROM ( { partition_bound_expr | MINVALUE | MAXVALUE } [, ...] )
      TO ( { partition_bound_expr | MINVALUE | MAXVALUE } [, ...] ) |
    WITH ( MODULUS numeric_literal, REMAINDER numeric_literal )
    
    URL: https://www.postgresql.org/docs/16/sql-createforeigntable.html
    
    postgres=# CREATE FOREIGN TABLE t_two_columns_table (
    postgres(#         id INTEGER,
    postgres(#         name VARCHAR(100)
    postgres(# )
    postgres-# SERVER foreign_server
    postgres-# OPTIONS (schema_name 'public', table_name 'two_columns_table');
    CREATE FOREIGN TABLE

接下来便可以查看这个外部表的数据是否和远端一致了。

    postgres=# -- 切换至foreign_db
    postgres=# \c foreign_db
    You are now connected to database "foreign_db" as user "postgres".
    foreign_db=# SELECT * FROM two_columns_table;
     id |   name   
    ----+----------
      1 | John Doe
    (1 row)
    

可以看到的是，两个表虽然表名不一致，但是查询出来的数据缺失一摸一样的。而对于字段多的那张表也可以采取这样子的操作去访问这其中的数据。

虽然此次并不涉及源码分析，但是其实看一下三板斧提供的信息也不难看出，其实就是根据SERVER和USER MAPPING的信息创建了一个会话连接，

当我们访问外部表的时候，再根据外部表的信息，将当前运行的SQL变换，通过会话连接下发至远端会话查询，最后再将远端返回的数据进行处理和输出。

简单验证的话，我们可以通过`ps -aux|grep postgres`查看后台会话进程。

    [postgres@iZuf6hwo0wgeev4dvua4csZ ~]$ ps -aux|grep postgres
    postgres  999988  0.0  1.0 235992 18684 ?        Ss   Jan20   1:15 /u01/app/halo/product/dbms/16/bin/postgres
    postgres  999989  0.0  1.4 236264 25556 ?        Ss   Jan20   0:00 postgres: checkpointer 
    postgres  999990  0.0  0.3 235992  6584 ?        Ss   Jan20   0:10 postgres: background writer 
    postgres  999992  0.0  0.4 235992  7948 ?        Ss   Jan20   0:09 postgres: walwriter 
    postgres  999993  0.0  0.4 237560  8524 ?        Ss   Jan20   0:37 postgres: autovacuum launcher 
    postgres  999994  0.0  0.3 237452  5796 ?        Ss   Jan20   0:00 postgres: logical replication launcher 
    postgres 1246437  0.0  1.2 255108 23080 ?        Ss   14:33   0:00 postgres: postgres postgres [local] idle
    postgres 1246439  0.0  0.9 238080 17172 ?        Ss   14:33   0:00 postgres: postgres foreign_db 127.0.0.1(60164) idle

以及使用`EXPLAIN VERBOSE` 查看实际执行的远端SQL是什么。

    postgres=# EXPLAIN VERBOSE SELECT * FROM t_two_columns_table;
                                          QUERY PLAN                                      
    --------------------------------------------------------------------------------------
     Foreign Scan on public.t_two_columns_table  (cost=100.00..119.99 rows=333 width=222)
       Output: id, name
       Remote SQL: SELECT id, name FROM public.two_columns_table
    (3 rows)

  

四、关于创建外部表的小妙招
=============

当远端的表中存在太多的字段，手动创建外部表感觉好麻烦，有没有更简单的方法来偷偷懒呢？

誒，程序员的事能叫偷懒吗！ 有的，兄弟有的，我们可以使用IMPORT FOREIGN SCHEMA帮助我们来导入外部表，

使用这个的一个前提是，正在使用的fdw有完整的按照PostgreSQL定义的这个[FDW Routines for IMPORT FOREIGN SCHEMA](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-IMPORT)接口来实现相关的功能。

    postgres=# \h IMPORT FOREIGN SCHEMA
    Command:     IMPORT FOREIGN SCHEMA
    Description: import table definitions from a foreign server
    Syntax:
    IMPORT FOREIGN SCHEMA remote_schema
        [ { LIMIT TO | EXCEPT } ( table_name [, ...] ) ]
        FROM SERVER server_name
        INTO local_schema
        [ OPTIONS ( option 'value' [, ... ] ) ]
    
    URL: https://www.postgresql.org/docs/16/sql-importforeignschema.html
    
    postgres=# -- 查看当前外部表信息
    postgres=# \dE+
                                           List of relations
     Schema |        Name         |     Type      |  Owner   | Persistence |  Size   | Description 
    --------+---------------------+---------------+----------+-------------+---------+-------------
     public | t_two_columns_table | foreign table | postgres | permanent   | 0 bytes | 
    (1 row)
    
    postgres=# IMPORT FOREIGN SCHEMA public LIMIT TO(twenty_columns_table)
    postgres-# FROM SERVER foreign_server INTO public;
    IMPORT FOREIGN SCHEMA
    postgres=# -- 再次查看当前外部表信息
    postgres=# \dE+
                                           List of relations
     Schema |         Name         |     Type      |  Owner   | Persistence |  Size   | Description 
    --------+----------------------+---------------+----------+-------------+---------+-------------
     public | t_two_columns_table  | foreign table | postgres | permanent   | 0 bytes | 
     public | twenty_columns_table | foreign table | postgres | permanent   | 0 bytes | 
    (2 rows)
    
    postgres=# SELECT * FROM twenty_columns_table;
     id |  col1  |  col2  |  col3  |  col4  |  col5  |  col6  |  col7  |  col8  |  col9  |  col10  |  col11  |  col12  |  col13  |  col14  |  col15  |  col16  |  col17  |  col18  |  col19  
    |  col20  
    ----+--------+--------+--------+--------+--------+--------+--------+--------+--------+---------+---------+---------+---------+---------+---------+---------+---------+---------+---------
    +---------
      1 | Value1 | Value2 | Value3 | Value4 | Value5 | Value6 | Value7 | Value8 | Value9 | Value10 | Value11 | Value12 | Value13 | Value14 | Value15 | Value16 | Value17 | Value18 | Value19 
    postgres=# SELECT * FROM twenty_columns_table \gx
    -[ RECORD 1 ]--
    id    | 1
    col1  | Value1
    col2  | Value2
    col3  | Value3
    col4  | Value4
    col5  | Value5
    col6  | Value6
    col7  | Value7
    col8  | Value8
    col9  | Value9
    col10 | Value10
    col11 | Value11
    col12 | Value12
    col13 | Value13
    col14 | Value14
    col15 | Value15
    col16 | Value16
    col17 | Value17
    col18 | Value18
    col19 | Value19
    col20 | Value20

详细的命令使用，请查看官方文档的[IMPORT FOREIGN SCHEMA](https://www.postgresql.org/docs/16/sql-importforeignschema.html) ，可以指定某些表导入，就像上面的示例一样。

也可以指定哪些表不被导入，还可以将远端的整个schema下的表导入进来，非常的方便~

这里就在演示了，篇幅太长了。

  

五、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文章转载请联系，谢谢合作。