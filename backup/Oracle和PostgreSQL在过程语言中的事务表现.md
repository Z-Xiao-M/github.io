## 异常处理执行插入操作
### oracle 略过异常语句，默认提交
```sql
SQL> CREATE TABLE tmp(id int);

Table created.

CREATE OR REPLACE PROCEDURE test_proc AS 
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN
          INSERT INTO tmp VALUES (1); -- 插入数据-1、1
END;
  9  /

Procedure created.

SQL> CALL test_proc();

Call completed.

SQL> SELECT * FROM tmp;

        ID
----------
        -1
         1

SQL> 
```
### PostgreSQL 异常前的语句全部回滚
```sql
postgres=# CREATE TABLE tmp(id int);
CREATE TABLE
postgres=# CREATE OR REPLACE PROCEDURE test_proc() AS $$ 
DECLARE
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN
          INSERT INTO tmp VALUES (1); -- 插入数据1
END; $$ LANGUAGE PLPGSQL;
CREATE PROCEDURE
postgres=# CALL test_proc();
CALL
postgres=# SELECT * FROM tmp;
 id 
----
  1
(1 row)

postgres=# 
```

## 异常处理执行回滚操作
### oracle 所有数据回滚
```sql
SQL> truncate table tmp;

Table truncated.

CREATE OR REPLACE PROCEDURE test_proc
AS 
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (0);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN
          rollback;              -- 全部回滚 无数据
END;
 11  /

Procedure created.

SQL> CALL test_proc();

Call completed.

SQL> SELECT * FROM tmp;

no rows selected

SQL> 
```
### PostgreSQL 同oracle一致 全部回滚
```sql
postgres=# truncate table tmp;
TRUNCATE TABLE
postgres=# CREATE OR REPLACE PROCEDURE test_proc() AS $$ 
DECLARE
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (0);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN
          rollback;              -- 全部回滚 无数据
END; $$ LANGUAGE PLPGSQL;
CREATE PROCEDURE
postgres=# CALL test_proc();
CALL
postgres=# SELECT * FROM tmp;
 id 
----
(0 rows)

postgres=# 
```

## 异常处理执行提交操作
### oracle 略过异常语句 其余部分正常提交
```sql
SQL> truncate table tmp;

Table truncated.

CREATE OR REPLACE PROCEDURE test_proc
AS 
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (0);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN           -- 提交 插入数据-1、0
          COMMIT;
END;
 11  /

Procedure created.

SQL> CALL test_proc();

Call completed.

SQL> SELECT * FROM tmp;

        ID
----------
        -1
         0

SQL> 
```
### PostgreSQL 全部回滚无数据
```sql
postgres=# truncate table tmp;
TRUNCATE TABLE
postgres=# CREATE OR REPLACE PROCEDURE test_proc() AS $$ 
DECLARE
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (0);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN           -- 提交 无数据
          COMMIT;
END; $$ LANGUAGE PLPGSQL;
CREATE PROCEDURE
postgres=# CALL test_proc();
CALL
postgres=# SELECT * FROM tmp;
 id 
----
(0 rows)

postgres=# 
```

## 测试语句
### oracle
```sql
CREATE TABLE tmp(id int);

CREATE OR REPLACE PROCEDURE test_proc AS 
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN
          INSERT INTO tmp VALUES (1); -- 插入数据-1、1
END;
/
CALL test_proc();
SELECT * FROM tmp;

truncate table tmp;
CREATE OR REPLACE PROCEDURE test_proc
AS 
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (0);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN
          rollback;              -- 全部回滚 无数据
END;
/
CALL test_proc();
SELECT * FROM tmp;

truncate table tmp;
CREATE OR REPLACE PROCEDURE test_proc
AS 
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (0);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN           -- 提交 插入数据-1、0
          COMMIT;
END;
/
CALL test_proc();
SELECT * FROM tmp;
```
### PostgreSQL
```sql
CREATE TABLE tmp(id int);

CREATE OR REPLACE PROCEDURE test_proc() AS $$ 
DECLARE
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN
          INSERT INTO tmp VALUES (1); -- 插入数据1
END; $$ LANGUAGE PLPGSQL;

CALL test_proc();
SELECT * FROM tmp;

truncate table tmp;
CREATE OR REPLACE PROCEDURE test_proc() AS $$ 
DECLARE
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (0);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN
          rollback;              -- 全部回滚 无数据
END; $$ LANGUAGE PLPGSQL;
CALL test_proc();
SELECT * FROM tmp;

truncate table tmp;
CREATE OR REPLACE PROCEDURE test_proc() AS $$ 
DECLARE
BEGIN
  INSERT INTO tmp VALUES (-1);
  INSERT INTO tmp VALUES (0);
  INSERT INTO tmp VALUES (1/0);  -- 制造异常
  EXCEPTION
      WHEN others THEN           -- 提交 无数据
          COMMIT;
END; $$ LANGUAGE PLPGSQL;
CALL test_proc();
SELECT * FROM tmp;
```