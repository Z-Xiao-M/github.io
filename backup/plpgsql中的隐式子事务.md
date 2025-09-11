# 子事务
在 PostgreSQL 中，子事务（Subtransaction）是指在一个主事务内部创建的嵌套事务，用于实现更细粒度的事务控制。它允许在主事务的范围内，将一部分操作划分为独立的子单元，以便单独进行回滚或提交。
```sql
postgres=# create table users(name text);
CREATE TABLE
postgres=# -- 开始主事务
postgres=# BEGIN;
BEGIN
postgres=*# -- 执行主事务中的操作
postgres=*# INSERT INTO users VALUES ('Alice');
INSERT 0 1
postgres=*# -- 创建保存点（子事务开始）
postgres=*# SAVEPOINT sp1;
SAVEPOINT
postgres=*# -- 子事务中的操作
postgres=*# INSERT INTO users VALUES ('Bob');
INSERT 0 1
postgres=*# -- 回滚到保存点
postgres=*# ROLLBACK TO sp1;
ROLLBACK
postgres=*# RELEASE SAVEPOINT sp1;
RELEASE
postgres=*# COMMIT;
COMMIT
postgres=# select * from users;
 name  
-------
 Alice
(1 row)
```

# plpgsql中的子事务
使用plpgsql创建函数虽然可以写SAVEPOINT，但是并不支持调用。
```sql
postgres=# CREATE OR REPLACE FUNCTION test_func()
postgres-# RETURNS void AS $$
postgres$# BEGIN                        
postgres$#     INSERT INTO users VALUES ('Alice');
postgres$# SAVEPOINT sp1;         
postgres$# END;
postgres$# $$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=# select test_func();
ERROR:  unsupported transaction command in PL/pgSQL
CONTEXT:  PL/pgSQL function test_func() line 4 at SQL statement
```
那是不是说明plpgsql就不支持子事务呢？
那倒是未必~ plpgsql支持隐式子事务，隐式子事务的开启取决于函数中是否存在异常块。

# 简单演示
在事务块中调用存在异常的函数
```sql
CREATE TABLE tmp(id int);

CREATE OR REPLACE FUNCTION demo_plpgsql_subxact()
RETURNS void AS $$
BEGIN                        
    INSERT INTO tmp VALUES (-2);
    INSERT INTO tmp VALUES (1/0);  -- error
EXCEPTION
    WHEN division_by_zero THEN
        RAISE INFO '%', SQLERRM;
        INSERT INTO tmp VALUES (-3);            
END;
$$ LANGUAGE plpgsql;

BEGIN; -- 开启事务
INSERT INTO tmp VALUES (-1);
ELECT demo_plpgsql_subxact(); -- 调用函数
select * from tmp;
COMMIT;

truncate tmp;

-- 等价于
BEGIN; -- 开启事务
INSERT INTO tmp VALUES (-1);
SAVEPOINT exception;  -- 保存点
INSERT INTO tmp VALUES (-2);
INSERT INTO tmp VALUES (1/0);  -- error
ROLLBACK TO SAVEPOINT exception; -- 异常回滚到保存点
INSERT INTO tmp VALUES (-3); 
SELECT * FROM tmp;
COMMIT;
```

运行结果

```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# CREATE TABLE tmp(id int);
CREATE TABLE
postgres=# CREATE OR REPLACE FUNCTION demo_plpgsql_subxact()
postgres-# RETURNS void AS $$
postgres$# BEGIN  
postgres$#     INSERT INTO tmp VALUES (-2);
postgres$#     INSERT INTO tmp VALUES (1/0);  -- error
postgres$# EXCEPTION
postgres$#     WHEN division_by_zero THEN
postgres$#                 RAISE INFO '%', SQLERRM;
postgres$#         INSERT INTO tmp VALUES (-3); 
postgres$# END;
postgres$# $$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=# BEGIN;
BEGIN
postgres=*# INSERT INTO tmp VALUES (-1);
INSERT 0 1
postgres=*# SELECT demo_plpgsql_subxact(); -- 调用函数
INFO:  division by zero
 demo_plpgsql_subxact 
----------------------
 
(1 row)

postgres=*# select * from tmp;
 id 
----
 -1
 -3
(2 rows)

postgres=*# COMMIT;
COMMIT
postgres=# truncate tmp;
TRUNCATE TABLE
postgres=# BEGIN;
BEGIN
postgres=*# INSERT INTO tmp VALUES (-1);
INSERT 0 1
postgres=*# SAVEPOINT exception;
SAVEPOINT
postgres=*# INSERT INTO tmp VALUES (-2);
INSERT 0 1
postgres=*# INSERT INTO tmp VALUES (1/0);  -- error
ERROR:  division by zero
postgres=!# ROLLBACK TO SAVEPOINT exception;
ROLLBACK
postgres=*# INSERT INTO tmp VALUES (-3); 
INSERT 0 1
postgres=*# COMMIT;
COMMIT
postgres=# SELECT * FROM tmp;
 id 
----
 -1
 -3
(2 rows)

postgres=# 
```

在事务块中，调用不存在异常的函数

```sql
TRUNCATE tmp;

CREATE OR REPLACE FUNCTION demo_plpgsql_subxact()
RETURNS void AS $$
BEGIN                        
    INSERT INTO tmp VALUES (-2);
    INSERT INTO tmp VALUES (-3);
EXCEPTION
    WHEN division_by_zero THEN
        RAISE INFO '%', SQLERRM;        
END;
$$ LANGUAGE plpgsql;

BEGIN; -- 开启事务块
INSERT INTO tmp VALUES (-1);
select demo_plpgsql_subxact();
INSERT INTO tmp VALUES (-4);
SELECT * FROM tmp;
COMMIT;


TRUNCATE tmp;
-- 等价于
BEGIN; -- 开启事务块
INSERT INTO tmp VALUES (-1);
SAVEPOINT exception;
INSERT INTO tmp VALUES (-2);
INSERT INTO tmp VALUES (-3);
RELEASE SAVEPOINT exception;
INSERT INTO tmp VALUES (-4);
SELECT * FROM tmp;
COMMIT;
```

运行结果

```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# TRUNCATE tmp;
TRUNCATE TABLE
postgres=# CREATE OR REPLACE FUNCTION demo_plpgsql_subxact()
RETURNS void AS $$
BEGIN                        
    INSERT INTO tmp VALUES (-2);
    INSERT INTO tmp VALUES (-3);
EXCEPTION
    WHEN division_by_zero THEN
        RAISE INFO '%', SQLERRM;        
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=# BEGIN; -- 开启事务块
BEGIN
postgres=*# INSERT INTO tmp VALUES (-1);
INSERT 0 1
postgres=*# select demo_plpgsql_subxact();
 demo_plpgsql_subxact 
----------------------
 
(1 row)

postgres=*# INSERT INTO tmp VALUES (-4);
INSERT 0 1
postgres=*# SELECT * FROM tmp;
 id 
----
 -1
 -2
 -3
 -4
(4 rows)

postgres=*# COMMIT;
COMMIT
postgres=# TRUNCATE tmp;
TRUNCATE TABLE
postgres=# -- 等价于
postgres=# BEGIN; -- 开启事务块
BEGIN
postgres=*# INSERT INTO tmp VALUES (-1);
INSERT 0 1
postgres=*# SAVEPOINT exception; -- 开启子事务
SAVEPOINT
postgres=*# INSERT INTO tmp VALUES (-2);
INSERT 0 1
postgres=*# INSERT INTO tmp VALUES (-3);
INSERT 0 1
postgres=*# RELEASE SAVEPOINT exception;
RELEASE
postgres=*# INSERT INTO tmp VALUES (-4);
INSERT 0 1
postgres=*# SELECT * FROM tmp;
 id 
----
 -1
 -2
 -3
 -4
(4 rows)

postgres=*# COMMIT;
COMMIT
postgres=# 
```
# 源码展示
没有exception则不会触发子事务的动作，部分`exec_stmt_block`代码片段如下
```c
static int
exec_stmt_block(PLpgSQL_execstate *estate, PLpgSQL_stmt_block *block)
{
  // initialize 
  if (block->exceptions)
  {
    BeginInternalSubTransaction(NULL);  // 开启子事务

    PG_TRY();
    {
      /* Run the block's statements */
      rc = exec_stmts(estate, block->body); // 执行相关操作

      /* Commit the inner transaction, return to outer xact context */
      ReleaseCurrentSubTransaction(); // 释放子事务
    }
    PG_CATCH();
    {

      /* Abort the inner transaction */
      RollbackAndReleaseCurrentSubTransaction(); // 发生了异常回滚子事务

      // 异常匹配和处理
      foreach(e, block->exceptions->exc_list)
      {
        PLpgSQL_exception *exception = (PLpgSQL_exception *) lfirst(e);
        if (exception_matches_conditions(edata, exception->conditions))
        {
          rc = exec_stmts(estate, exception->action); // exception块中的其余操作
          break;
        }
      }
    }
    PG_END_TRY();
  }
  else
  {
    /*
     * Just execute the statements in the block's body
     */
    estate->err_text = NULL;
    // 没有exception块 执行此处 不会开启子事务
    rc = exec_stmts(estate, block->body);
  }
  // ...
}
``` 