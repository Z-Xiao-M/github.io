# plpgsql事务控制场景
## 支持的场景如下
#### 存储过程调用COMMIT/ROLLBACK
```sql
postgres=# CREATE TABLE test1 (a int, b text);
CREATE TABLE
postgres=# CREATE PROCEDURE transaction_test1(x int, y text)
LANGUAGE plpgsql
AS $$
BEGIN
    FOR i IN 0..x LOOP
        INSERT INTO test1 (a, b) VALUES (i, y);
        IF i % 2 = 0 THEN
            COMMIT;
        ELSE
            ROLLBACK;
        END IF;
    END LOOP;
END
$$;
CREATE PROCEDURE
postgres=# CALL transaction_test1(9, 'foo');
CALL
postgres=# SELECT * FROM test1;
 a |  b  
---+-----
 0 | foo
 2 | foo
 4 | foo
 6 | foo
 8 | foo
(5 rows)
```
#### 匿名块（DO）调用COMMIT/ROLLBACK
```sql
postgres=# TRUNCATE test1;
TRUNCATE TABLE
postgres=# DO
LANGUAGE plpgsql
$$
BEGIN
    FOR i IN 0..9 LOOP
        INSERT INTO test1 (a) VALUES (i);
        IF i % 2 = 0 THEN
            COMMIT;
        ELSE
            ROLLBACK;
        END IF;
    END LOOP;
END
$$;
DO
postgres=# SELECT * FROM test1;
 a | b 
---+---
 0 | 
 2 | 
 4 | 
 6 | 
 8 | 
(5 rows)
```
#### 存储过程嵌套调用含有事务控制语句的存储过程或者匿名块
```sql
postgres=# CREATE PROCEDURE transaction_test6(c text)
LANGUAGE plpgsql
AS $$
BEGIN
    CALL transaction_test1(9, c);
END;
$$;
CREATE PROCEDURE
postgres=# CALL transaction_test6('bar');
CALL
postgres=# SELECT * FROM test1;
 a |  b  
---+-----
 0 | bar
 2 | bar
 4 | bar
 6 | bar
 8 | bar
(5 rows)

postgres=# 
postgres=# TRUNCATE test1;
TRUNCATE TABLE
postgres=# CREATE PROCEDURE transaction_test7()
LANGUAGE plpgsql
AS $$
BEGIN
    DO 'BEGIN CALL transaction_test1(9, $x$baz$x$); END;';
END;
$$;
CREATE PROCEDURE
postgres=# CALL transaction_test7();
CALL
postgres=# 
postgres=# SELECT * FROM test1;
 a |  b  
---+-----
 0 | baz
 2 | baz
 4 | baz
 6 | baz
 8 | baz
(5 rows)
```
#### 遍历只读游标时执行COMMIT/ROLLBACK（后面的支持场景如果没有明确标识存储过程，代表现象和匿名块一致）
```sql
postgres=# CREATE TABLE test2 (x int);
CREATE TABLE
postgres=# INSERT INTO test2 VALUES (0), (1), (2), (3), (4);
INSERT 0 5
postgres=# TRUNCATE test1;
TRUNCATE TABLE
postgres=# DO LANGUAGE plpgsql $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN SELECT * FROM test2 ORDER BY x LOOP
        INSERT INTO test1 (a) VALUES (r.x);
        IF r.x % 2 = 0 THEN
            COMMIT;
        ELSE
            ROLLBACK;
        END IF;
    END LOOP;
END;
$$;
DO
postgres=# SELECT * FROM test1;
 a | b 
---+---
 0 | 
 2 | 
 4 | 
(3 rows)
```
#### 存在EXCEPTION块（隐式子事务），且COMMIT/ROLLBACK位于异常处理之中
```sql
postgres=# DO LANGUAGE plpgsql $$
BEGIN
    FOR i IN 1..10 LOOP
      BEGIN
        INSERT INTO test1 VALUES (i, 'good');
        INSERT INTO test1 VALUES (i/0, 'bad');
      EXCEPTION
        WHEN division_by_zero THEN
            INSERT INTO test1 VALUES (i, 'exception');
            IF (i % 3) > 0 THEN COMMIT; ELSE ROLLBACK; END IF;
      END;
    END LOOP;
END;
$$;
DO
postgres=# SELECT * FROM test1;
 a  |     b     
----+-----------
  1 | exception
  2 | exception
  4 | exception
  5 | exception
  7 | exception
  8 | exception
 10 | exception
(7 rows)
```
#### 事务链（COMMIT AND CHAIN/ROLLBACK AND CHAIN）
```sql
postgres=# TRUNCATE test1;
TRUNCATE TABLE
postgres=# DO LANGUAGE plpgsql $$
BEGIN
    ROLLBACK;
    SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    FOR i IN 0..3 LOOP
        RAISE INFO 'transaction_isolation = %', current_setting('transaction_isolation');
        INSERT INTO test1 (a) VALUES (i);
        IF i % 2 = 0 THEN
            COMMIT AND CHAIN;
        ELSE
            ROLLBACK AND CHAIN;
        END IF;
    END LOOP;
END
$$;
INFO:  transaction_isolation = repeatable read
INFO:  transaction_isolation = repeatable read
INFO:  transaction_isolation = repeatable read
INFO:  transaction_isolation = repeatable read
DO
postgres=# SELECT * FROM test1;
 a | b 
---+---
 0 | 
 2 | 
(2 rows)
```

## 不支持的场景如下
#### 函数调用COMMIT/ROLLBACK
```sql
postgres=# CREATE FUNCTION transaction_test2() RETURNS int
LANGUAGE plpgsql
AS $$
BEGIN
    FOR i IN 0..9 LOOP
        INSERT INTO test1 (a) VALUES (i);
        IF i % 2 = 0 THEN
            COMMIT;
        ELSE
            ROLLBACK;
        END IF;
    END LOOP;
    RETURN 1;
END
$$;
CREATE FUNCTION
postgres=# SELECT transaction_test2();
2025-11-07 11:11:56.311 CST [6399] ERROR:  invalid transaction termination
2025-11-07 11:11:56.311 CST [6399] CONTEXT:  PL/pgSQL function transaction_test2() line 6 at COMMIT
2025-11-07 11:11:56.311 CST [6399] STATEMENT:  SELECT transaction_test2();
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function transaction_test2() line 6 at COMMIT
```
#### 在函数内调用含有事务控制语句的存储过程或动态匿名块
```sql
postgres=# CREATE FUNCTION transaction_test3() RETURNS int
LANGUAGE plpgsql
AS $$
BEGIN
    CALL transaction_test1(9, 'error');
    RETURN 1;
END;
$$;
CREATE FUNCTION
postgres=# SELECT transaction_test3();
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function transaction_test1(integer,text) line 6 at COMMIT
SQL statement "CALL transaction_test1(9, 'error')"
PL/pgSQL function transaction_test3() line 3 at CALL
postgres=# 
postgres=# CREATE FUNCTION transaction_test4() RETURNS int
LANGUAGE plpgsql
AS $$
BEGIN
    EXECUTE 'DO LANGUAGE plpgsql $x$ BEGIN COMMIT; END $x$';
    RETURN 1;
END;
$$;
CREATE FUNCTION
postgres=# SELECT transaction_test4();
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function inline_code_block line 1 at COMMIT
SQL statement "DO LANGUAGE plpgsql $x$ BEGIN COMMIT; END $x$"
PL/pgSQL function transaction_test4() line 3 at EXECUTE
```

#### 在事务块中调用含有事务控制语句的存储过程或匿名块
```sql
postgres=# START TRANSACTION;
START TRANSACTION
postgres=*# CALL transaction_test1(9, 'error');
2025-11-07 11:08:49.437 CST [5483] ERROR:  invalid transaction termination
2025-11-07 11:08:49.437 CST [5483] CONTEXT:  PL/pgSQL function transaction_test1(integer,text) line 6 at COMMIT
2025-11-07 11:08:49.437 CST [5483] STATEMENT:  CALL transaction_test1(9, 'error');
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function transaction_test1(integer,text) line 6 at COMMIT
postgres=!# COMMIT;
ROLLBACK
postgres=#
postgres=# START TRANSACTION;
START TRANSACTION
postgres=*# DO LANGUAGE plpgsql $$ BEGIN COMMIT; END $$;
2025-11-07 11:09:44.316 CST [5483] ERROR:  invalid transaction termination
2025-11-07 11:09:44.316 CST [5483] CONTEXT:  PL/pgSQL function inline_code_block line 1 at COMMIT
2025-11-07 11:09:44.316 CST [5483] STATEMENT:  DO LANGUAGE plpgsql $$ BEGIN COMMIT; END $$;
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function inline_code_block line 1 at COMMIT
postgres=!# COMMIT;
ROLLBACK

```
#### 存储过程设置了proconfig或者标识为SECURITY DEFINER
```sql
postgres=# CREATE PROCEDURE transaction_test5()
LANGUAGE plpgsql
SET work_mem = 555
AS $$
BEGIN
    COMMIT;
END;
$$;
CREATE PROCEDURE
postgres=# CALL transaction_test5();
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function transaction_test5() line 3 at COMMIT
postgres=# 
postgres=# CREATE PROCEDURE transaction_test5b()
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
    COMMIT;
END;
$$;
CREATE PROCEDURE
postgres=# CALL transaction_test5b();
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function transaction_test5b() line 3 at COMMIT
```
#### 存储过程动态调用存储过程
```sql
postgres=# CREATE PROCEDURE transaction_test8()
LANGUAGE plpgsql
AS $$
BEGIN
    EXECUTE 'CALL transaction_test1(10, $x$baz$x$)';
END;
$$;
CREATE PROCEDURE
postgres=# CALL transaction_test8();
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function transaction_test1(integer,text) line 6 at COMMIT
SQL statement "CALL transaction_test1(10, $x$baz$x$)"
PL/pgSQL function transaction_test8() line 3 at EXECUTE
```
#### 匿名块动态调用事务控制语句
```sql
postgres=# DO LANGUAGE plpgsql $$
BEGIN
    EXECUTE 'COMMIT';
END;
$$;
ERROR:  EXECUTE of transaction commands is not implemented
CONTEXT:  PL/pgSQL function inline_code_block line 3 at EXECUTE
postgres=# DO LANGUAGE plpgsql $$
BEGIN
    EXECUTE 'ROLLBACK';
END;
$$;
ERROR:  EXECUTE of transaction commands is not implemented
CONTEXT:  PL/pgSQL function inline_code_block line 3 at EXECUTE
```
#### 遍历非只读游标（RETURNING子句）
```sql
postgres=# TRUNCATE test1;
TRUNCATE TABLE
postgres=# DO LANGUAGE plpgsql $$
DECLARE
    r RECORD;
BEGIN
    FOR r IN UPDATE test2 SET x = x * 2 RETURNING x LOOP
        INSERT INTO test1 (a) VALUES (r.x);
        ROLLBACK;
    END LOOP;
END;
$$;
ERROR:  cannot perform transaction commands inside a cursor loop that is not read-only
CONTEXT:  PL/pgSQL function inline_code_block line 7 at ROLLBACK
```
#### SAVEPOINT
```sql
postgres=# DO LANGUAGE plpgsql $$
BEGIN
    SAVEPOINT foo;
END;
$$;
ERROR:  unsupported transaction command in PL/pgSQL
CONTEXT:  PL/pgSQL function inline_code_block line 3 at SQL statement
```
#### 修改事务隔离级别前，未COMMIT/ROLLBACK
```sql
postgres=# DO LANGUAGE plpgsql $$
BEGIN
    RAISE INFO '%', current_setting('transaction_isolation');
    SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
END;
$$;
INFO:  read committed
ERROR:  SET TRANSACTION ISOLATION LEVEL must be called before any query
CONTEXT:  SQL statement "SET TRANSACTION ISOLATION LEVEL REPEATABLE READ"
PL/pgSQL function inline_code_block line 4 at SQL statement
```

#### 存在EXCEPTION块（隐式子事务），且COMMIT/ROLLBACK不在异常处理之中
```sql
postgres=# TRUNCATE test1;
TRUNCATE TABLE
postgres=# DO LANGUAGE plpgsql $$
BEGIN
    BEGIN
        INSERT INTO test1 (a) VALUES (1);
        COMMIT;
        INSERT INTO test1 (a) VALUES (1/0);
        COMMIT;
    EXCEPTION
        WHEN division_by_zero THEN
            RAISE NOTICE 'caught division_by_zero';
    END;
END;
$$;
ERROR:  cannot commit while a subtransaction is active
CONTEXT:  PL/pgSQL function inline_code_block line 5 at COMMIT
postgres=# DO LANGUAGE plpgsql $$
BEGIN
    BEGIN
        INSERT INTO test1 (a) VALUES (1);
        ROLLBACK;
        INSERT INTO test1 (a) VALUES (1/0);
        ROLLBACK;
    EXCEPTION
        WHEN division_by_zero THEN
            RAISE NOTICE 'caught division_by_zero';
    END;
END;
$$;
ERROR:  cannot roll back while a subtransaction is active
CONTEXT:  PL/pgSQL function inline_code_block line 5 at ROLLBACK
postgres=# 
```

# 声明
上述的示例均是在`AUTOCOMMIT = 'on'`的前提之下完成的，因为在很早之前我们就聊过[`AUTOCOMMIT`](https://mp.weixin.qq.com/s/dTECp4GKJjMQyKHfOV3kCg)的一个实现机制，当`AUTOCOMMIT = 'off'`等价于在运行真正的语句之前为你开一个新的事务。
```sql
postgres=# \set AUTOCOMMIT off
postgres=# CREATE TABLE test1 (a int, b text);
CREATE TABLE
postgres=*# DO       
LANGUAGE plpgsql
$$
BEGIN
    FOR i IN 0..9 LOOP
        INSERT INTO test1 (a) VALUES (i);
        IF i % 2 = 0 THEN
            COMMIT;
        ELSE
            ROLLBACK;
        END IF;
    END LOOP;
END
$$;
DO
ERROR:  invalid transaction termination
CONTEXT:  PL/pgSQL function inline_code_block line 6 at COMMIT
```
以及本文的示例来自[plpgsql_transaction.sql](https://github.com/postgres/postgres/blob/master/src/pl/plpgsql/src/sql/plpgsql_transaction.sql)