# CURSOR (游标) 简单介绍

游标[（Cursor）](https://www.postgresql.org/docs/current/plpgsql-cursors.html)是一种用于在数据库中遍历和处理大量查询结果集的机制。它允许应用程序逐条或分批地获取数据，而不是一次性加载所有结果到内存中，从而避免内存溢出。游标的主要作用是优化内存使用并提高性能，特别适用于处理大结果集或需要逐步传输数据的场景，例如数据分页、批量数据处理或与应用程序的交互式数据访问。

在PostgreSQL中，在SQL场景允许使用[DECLARE](\(https://www.postgresql.org/docs/current/sql-declare.html\))语句去定义一个游标，然后使用[FETCH](https://www.postgresql.org/docs/current/sql-fetch.html)语句去操作先前定义的游标，同时在plpgsql中又存在另一种方式去[定义](https://www.postgresql.org/docs/current/plpgsql-cursors.html#PLPGSQL-CURSOR-DECLARATIONS)和[使用](https://www.postgresql.org/docs/current/plpgsql-cursors.html#PLPGSQL-CURSOR-USING)游标。

今天不打算在这里赘述游标的定义和使用（比方说游标向前移、向后移，以二进制的形式返回还是什么的），打算写一点个人感觉相对冷门的用法。

# 场景构建

其实还是和上篇文章息息相关，就是将一个游标中的数据插入到另一张表中，并且在这其中使用了事务控制语句，如`commit/rollback`。如果我们使用下面的语句就会出现这个`cursor "<unnamed portal xx>" does not exist`现象。

```sql
postgres=# CREATE TABLE test(a INT);
CREATE TABLE
postgres=# DO $$
DECLARE
 p_curdata refcursor;
 val int;
BEGIN
 open p_curdata for SELECT generate_series(1,10);
 LOOP
   FETCH p_curdata INTO val;
   EXIT WHEN val IS NULL;
   INSERT INTO test VALUES(val);
   IF val % 2 = 0 THEN
      COMMIT;
   ELSE
      ROLLBACK;
   END IF;
 END LOOP;
END; $$;
ERROR:  cursor "<unnamed portal 1>" does not exist
CONTEXT:  PL/pgSQL function inline_code_block line 8 at FETCH
```

原因在于在plpgsql中使用事务控制语句，会将当前打开的cursor进行自动关闭，该怎么解决呢？

# 解决方案

#### 参考上一篇文章——[PL/pgSQL事务控制场景](https://mp.weixin.qq.com/s/Gp5zyMkPV_scOLH2Hk7v-w)

上一篇文章中我们有介绍过在FOR循环中使用只读游标，plpgsql内部会自动帮我们pin住这个只读游标，因此我们可以使用下面的两种写法。

##### 隐式游标变量

```sql
postgres=# TRUNCATE test;
TRUNCATE TABLE
postgres=# DO $$
DECLARE
  val int;
  cursor_info record;
  is_print bool DEFAULT false;
BEGIN
  FOR val IN SELECT generate_series(1,10) LOOP
    -- 查看一下pg_cursors中的信息
    IF is_print is false THEN
      is_print = true;
      SELECT * INTO cursor_info FROM pg_cursors;
      RAISE NOTICE 'cursor_info = %', cursor_info;
    END IF;

    -- 插入数据
    INSERT INTO test VALUES(val);
    IF val % 2 = 0 THEN
       COMMIT;
    ELSE
       ROLLBACK;
    END IF;
  END LOOP;
END; $$;
NOTICE:  cursor_info = ("<unnamed portal 2>","SELECT generate_series(1,10)",f,f,f,"2025-11-10 16:34:27.8992+08")
DO
postgres=# -- 查看表中数据
postgres=# SELECT * FROM test;
 a  
----
  2
  4
  6
  8
 10
(5 rows)
```

##### 显式游标变量

```sql
postgres=# TRUNCATE test;
TRUNCATE TABLE
postgres=# DO $$
DECLARE
  val record;
  cursor_info record;
  is_print bool DEFAULT false;
  p_curdata CURSOR FOR SELECT id FROM generate_series(1,10) t(id);
BEGIN
  FOR val IN p_curdata LOOP
    -- 查看一下pg_cursors中的信息
    IF is_print is false THEN
      is_print = true;
      SELECT * INTO cursor_info FROM pg_cursors;
      RAISE NOTICE 'cursor_info = %', cursor_info;
    END IF;

    -- 插入数据
    INSERT INTO test(a) VALUES(val.id);
    IF val.id % 2 = 0 THEN
       COMMIT;
    ELSE
       ROLLBACK;
    END IF;
  END LOOP;
END; $$;
NOTICE:  cursor_info = ("<unnamed portal 11>","SELECT id FROM generate_series(1,10) t(id)",f,f,t,"2025-11-10 16:50:09.976953+08")
DO
postgres=# SELECT * FROM test;
 a  
----
  2
  4
  6
  8
 10
(5 rows)
```

#### 相对冷门的写法

这个写法需要配合SQL层面的`DECLARE`语句来实现，且需要将cursor定义成HOLD CURSOR.

```sql
postgres=# TRUNCATE test;
TRUNCATE TABLE
postgres=# DO $$              
        DECLARE
        val INT;
        cursor_info record;
        is_print bool DEFAULT false;
        p_curdata refcursor = 'hold_cursor';
BEGIN
        EXECUTE 'DECLARE hold_cursor CURSOR WITH HOLD FOR SELECT generate_series(1,10)';

        LOOP
                FETCH p_curdata INTO val;
                EXIT WHEN val IS NULL;
            -- 查看一下pg_cursors中的信息
            IF is_print is false THEN
              is_print = true;
              SELECT * INTO cursor_info FROM pg_cursors;
              RAISE NOTICE 'cursor_info = %', cursor_info;
            END IF;

            IF val % 2 = 0 THEN
                INSERT INTO test(a) VALUES(val); -- 插入数据
               COMMIT;
            -- ELSE
               -- ROLLBACK; -- 这里不能使用ROLLBACK 如果使用会导致hold_cursor关闭
            END IF;
        END LOOP;
        -- 需要手动关闭
        CLOSE p_curdata;
END; $$;
NOTICE:  cursor_info = (hold_cursor,"DECLARE hold_cursor CURSOR WITH HOLD FOR SELECT generate_series(1,10)",t,f,f,"2025-11-11 00:47:57.064936+08")
DO
postgres=# SELECT * FROM test;
 a  
----
  2
  4
  6
  8
 10
(5 rows)

postgres=# SELECT * FROM pg_cursors;
 name | statement | is_holdable | is_binary | is_scrollable | creation_time 
------+-----------+-------------+-----------+---------------+---------------
(0 rows)
```
这里指的注意的是，和上面的FOR循环中的显式/隐式游标不同，这种写法是不能使用ROLLBACK的，如果使用了，即使定义的这个游标为`HOLD`状态也会直接导致跟随实际的事务处理掉，现象如下：
```sql
postgres=# do $$
declare
  p_CurData refcursor := 'hold_cursor';
  val int;
begin
  execute 'DECLARE hold_cursor CURSOR WITH HOLD FOR SELECT 42';
  loop
    fetch p_CurData into val;
    exit when val is null;
    raise notice 'val = %', val;
    rollback;
  end loop;
  close p_CurData;
end; $$;
NOTICE:  val = 42
ERROR:  cursor "hold_cursor" does not exist
CONTEXT:  PL/pgSQL function inline_code_block line 8 at FETCH
```
这是有意为之。
![](https://fastly.jsdelivr.net/gh/bucketio/img10@main/2025/11/11/1762794090873-d01d3a8d-98a4-4b98-87f9-1df35caf3904.png)


# 额外的一点补充（hold cursor）

如果声明的hold cursor，数据量不多，那么数据将会存放至内存之中；如果数据量过多，则会存放至临时文件中。可通过命令`ls -lh $PGDATA/base/pgsql_tmp/`，查看临时文件来判断，其实也就是物化。

```sql
postgres=# CREATE TABLE test_cursor_mat (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);
CREATE TABLE
postgres=# -- 插入1000000行数据
postgres=# INSERT INTO test_cursor_mat (data)
SELECT 'Test data row ' || i || repeat('x', 500)
FROM generate_series(1, 1000000) i;
INSERT 0 1000000
postgres=# ANALYZE test_cursor_mat;
ANALYZE
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_cursor_mat')) AS total_table_size;
 total_table_size 
------------------
 580 MB
(1 row)

postgres=# DECLARE hold_cursor CURSOR WITH HOLD FOR SELECT * FROM test_cursor_mat; -- hold_cursor
DECLARE CURSOR
postgres=# -- 查看当前的PID 
postgres=# SELECT pg_backend_pid();
 pg_backend_pid 
----------------
          18392
(1 row)

postgres=# -- 查看是否存在对应的临时文件
postgres=# \! ls -lh $PGDATA/base/pgsql_tmp/
total 525M
-rw------- 1 postgres postgres 525M Nov 10 14:24 pgsql_tmp18392.3
postgres=# -- 释放该hold cursor
postgres=# discard all;
DISCARD ALL
postgres=# -- 再次查看是否存在对应的临时文件
postgres=# \! ls -lh $PGDATA/base/pgsql_tmp/
total 0
```

少量数据 (这里的100条数据估计不超过8k)

```sql
postgres=# DECLARE hold_cursor CURSOR WITH HOLD FOR SELECT generate_series(1,100);
DECLARE CURSOR
postgres=# \! ls -lh $PGDATA/base/pgsql_tmp/
total 0
postgres=# select * from pg_cursors;
    name     |                                statement                                | is_holdable | is_binary | is_scrollable |         creation_time         
-------------+-------------------------------------------------------------------------+-------------+-----------+---------------+-------------------------------
 hold_cursor | DECLARE hold_cursor CURSOR WITH HOLD FOR SELECT generate_series(1,100); | t           | f         | f             | 2025-11-10 14:36:38.913502+08
(1 row)

```

