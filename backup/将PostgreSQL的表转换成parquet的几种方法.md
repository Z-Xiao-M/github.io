# 茴香豆的茴字，怎样写的?
读过书，……我便要考你一考。茴香豆的茴字，怎样写的?
> 孔乙己喝过半碗酒，涨红的脸色渐渐复了原，旁人便又问道，“孔乙己，你当真认识字么?”孔乙己看着问他的人，显出不屑置辩的神气。他们便接着说道，“你怎的连半个秀才也捞不到呢?”孔乙己立刻显出颓唐不安模样，脸上笼上了一层灰色，嘴里说些话;这回可是全是之乎者也之类，一些不懂了。在这时候，众人也都哄笑起来：店内外充满了快活的空气。

> 在这些时候，我可以附和着笑，掌柜是决不责备的。而且掌柜见了孔乙己，也每每这样问他，引人发笑。孔乙己自己知道不能和他们谈天，便只好向孩子说话。有一回对我说道，“你读过书么?”我略略点一点头。他说，“读过书，……我便要考你一考。茴香豆的茴字，怎样写的?”我想，讨饭一样的人，也配考我么? 便回过脸去，不再理会。孔乙己等了许久，很恳切的说道，“不能写罢?……我教给你，记着!这些字应该记着。将来做掌柜的时候，写帐要用。”我暗想我和掌柜的等级还很远呢，而且我们掌柜也从不将茴香豆上帐;又好笑，又不耐烦，懒懒的答他道，“谁要你教，不是草头底下一个来回的回字么?”孔乙己显出极高兴的样子，将两个指头的长指甲敲着柜台，点头说，“对呀对呀!……回字有四样写法，你知道么?”我愈不耐烦了，努着嘴走远。孔乙己刚用指甲蘸了酒，想在柜上写字，见我毫不热心，便又叹一口气，显出极惋惜的样子。

> 有几回，邻舍孩子听得笑声，也赶热闹，围住了孔乙己。他便给他们茴香豆吃，一人一颗。孩子吃完豆，仍然不散，眼睛都望着碟子。孔乙己着了慌，伸开五指将碟子罩住，弯腰下去说道，“不多了，我已经不多了。”直起身又看一看豆，自己摇头说，“不多不多!多乎哉?不多也。”于是这一群孩子都在笑声里走散了。

# 怎么将PostgreSQL的表转换成parquet
学过PostregreSQL，……我便要考你一考。将PostgreSQL的表转换成parquet，该怎样做呢?😀

## psql配合duckdb
postgresql生成csv，再通过duckdb转换成parquet
> 来自老乡的blog https://github.com/digoal/blog/blob/master/202311/20231130_01.md
```sql
psql -c "copy (select id, md5(random()::text) as info, clock_timestamp() ts from generate_series(1,10000) id) to stdout with (format csv, header on)" | duckdb -c "COPY (SELECT * FROM read_csv('/dev/stdin', delim=',', header=true, columns={'id': 'INTEGER', 'info': 'VARCHAR', 'ts': 'timestamp'})) TO '/tmp/test.parquet' (FORMAT 'parquet', COMPRESSION 'ZSTD', ROW_GROUP_SIZE 100000);"
```
主要分成两个部分
```sql
-- postgresql执行
copy (select id, md5(random()::text) as info, clock_timestamp() ts from generate_series(1,10000) id) to stdout with (format csv, header on)
-- duckdb执行
COPY (SELECT * FROM read_csv('/dev/stdin', delim=',', header=true, columns={'id': 'INTEGER', 'info': 'VARCHAR', 'ts': 'timestamp'})) TO '/tmp/test.parquet' (FORMAT 'parquet', COMPRESSION 'ZSTD', ROW_GROUP_SIZE 100000);
```
简单演示
```sql
postgres@zxm-VMware-Virtual-Platform:~/test$ psql -c "copy (select id, md5(random()::text) as info, clock_timestamp() ts from generate_series(1,10000) id) to stdout with (format csv, header on)" | duckdb -c "COPY (SELECT * FROM read_csv('/dev/stdin', delim=',', header=true, columns={'id': 'INTEGER', 'info': 'VARCHAR', 'ts': 'timestamp'})) TO '/tmp/test.parquet' (FORMAT 'parquet', COMPRESSION 'ZSTD', ROW_GROUP_SIZE 100000);"  
postgres@zxm-VMware-Virtual-Platform:~/test$ file /tmp/test.parquet
/tmp/test.parquet: Apache Parquet
postgres@zxm-VMware-Virtual-Platform:~/test$ duckdb
DuckDB v1.3.2 (Ossivalis) 0b83e5d2f6
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
D select count(*) from '/tmp/test.parquet';
┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│    10000     │
└──────────────┘
D select * from '/tmp/test.parquet' limit 5;
┌───────┬──────────────────────────────────┬────────────────────────────┐
│  id   │               info               │             ts             │
│ int32 │             varchar              │         timestamp          │
├───────┼──────────────────────────────────┼────────────────────────────┤
│     1 │ ebacc378cc0186bc3dc9eb55aae01e66 │ 2025-09-23 09:43:43.807629 │
│     2 │ 31c4f9c72131e3c345b67ca3cb0be76f │ 2025-09-23 09:43:43.807645 │
│     3 │ b280ef0e548239ddadcf3f20922b6203 │ 2025-09-23 09:43:43.807648 │
│     4 │ fd34351a122b6410380c486f295c849a │ 2025-09-23 09:43:43.80765  │
│     5 │ 84ebc3f89f473d213fbb5bb933428a9b │ 2025-09-23 09:43:43.807653 │
└───────┴──────────────────────────────────┴────────────────────────────┘
D 
```

## duckdb-postgres
使用duckdb的插件[duckdb-postgres](https://github.com/duckdb/duckdb-postgres)，以前也叫做postgres_scanner来着
```sql
postgres@zxm-VMware-Virtual-Platform:~/test$ psql
psql (16.10)
Type "help" for help.

postgres=# create table test as select id, md5(random()::text) as info, clock_timestamp() ts from generate_series(1,10000) id;
SELECT 10000
postgres=# \q
postgres@zxm-VMware-Virtual-Platform:~/test$ duckdb
DuckDB v1.3.2 (Ossivalis) 0b83e5d2f6
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
D INSTALL postgres;
D 
D LOAD postgres;
D ATTACH 'dbname=postgres user=postgres host=127.0.0.1 port=5432' AS db (TYPE postgres, SCHEMA 'public');
D select count(*) from db.test;
┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│    10000     │
└──────────────┘
D COPY db.test TO '/tmp/test2.parquet';
D  select count(*) from '/tmp/test2.parquet';
┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│    10000     │
└──────────────┘
D select * from '/tmp/test2.parquet' limit 5;
┌───────┬──────────────────────────────────┬───────────────────────────────┐
│  id   │               info               │              ts               │
│ int32 │             varchar              │   timestamp with time zone    │
├───────┼──────────────────────────────────┼───────────────────────────────┤
│     1 │ 03035730b2e25daa5c0846ac4240c084 │ 2025-09-23 18:00:01.095988+08 │
│     2 │ 76d27fc563a6bb33e58a3671494129f6 │ 2025-09-23 18:00:01.096066+08 │
│     3 │ 8f508b8def12eab455165dd6eb8a4cd4 │ 2025-09-23 18:00:01.096071+08 │
│     4 │ 0e2af5b95ae49489e4c3d7bdae7bf393 │ 2025-09-23 18:00:01.096074+08 │
│     5 │ d084e3c4d1106debae2d9efdaa2d6449 │ 2025-09-23 18:00:01.096077+08 │
└───────┴──────────────────────────────────┴───────────────────────────────┘
D 
```
## pg_duckdb
这是PostgreSQL的插件，[pg_duckdb](https://github.com/duckdb/pg_duckdb)增强了PostgreSQL的COPY，原生的PostgreSQL的COPY是不支持指定格式为PARQUET的 https://www.postgresql.org/docs/16/sql-copy.html
```
postgres@zxm-VMware-Virtual-Platform:~/test$ psql
psql (16.10)
Type "help" for help.

postgres=# \dx
                  List of installed extensions
   Name    | Version |   Schema   |         Description          
-----------+---------+------------+------------------------------
 pg_duckdb | 1.1.0   | public     | DuckDB Embedded in Postgres
 plpgsql   | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)

postgres=#  COPY test TO '/tmp/test3.parquet';
COPY 10000
postgres=# \! ls /tmp/test3.parquet -al
-rw------- 1 postgres postgres 406699  9月 23 18:09 /tmp/test3.parquet
postgres=# file /tmp/test3.parquet
postgres-# ^C
postgres=# \q
postgres@zxm-VMware-Virtual-Platform:~/test$ psql
psql (16.10)
Type "help" for help.

postgres=# \dx
                  List of installed extensions
   Name    | Version |   Schema   |         Description          
-----------+---------+------------+------------------------------
 pg_duckdb | 1.1.0   | public     | DuckDB Embedded in Postgres
 plpgsql   | 1.0     | pg_catalog | PL/pgSQL procedural language
(2 rows)

postgres=# COPY test TO '/tmp/test3.parquet';
COPY 10000
postgres=# \! ls /tmp/test3.parquet -al
-rw------- 1 postgres postgres 406699  9月 23 18:11 /tmp/test3.parquet
postgres=# \! file /tmp/test3.parquet
/tmp/test3.parquet: Apache Parquet
postgres=# \q
postgres@zxm-VMware-Virtual-Platform:~/test$ duckdb 
DuckDB v1.3.2 (Ossivalis) 0b83e5d2f6
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
D select count(*) from '/tmp/test3.parquet';
┌──────────────┐
│ count_star() │
│    int64     │
├──────────────┤
│    10000     │
└──────────────┘
D select * from '/tmp/test3.parquet' limit 5;
┌───────┬──────────────────────────────────┬───────────────────────────────┐
│  id   │               info               │              ts               │
│ int32 │             varchar              │   timestamp with time zone    │
├───────┼──────────────────────────────────┼───────────────────────────────┤
│     1 │ 03035730b2e25daa5c0846ac4240c084 │ 2025-09-23 18:00:01.095988+08 │
│     2 │ 76d27fc563a6bb33e58a3671494129f6 │ 2025-09-23 18:00:01.096066+08 │
│     3 │ 8f508b8def12eab455165dd6eb8a4cd4 │ 2025-09-23 18:00:01.096071+08 │
│     4 │ 0e2af5b95ae49489e4c3d7bdae7bf393 │ 2025-09-23 18:00:01.096074+08 │
│     5 │ d084e3c4d1106debae2d9efdaa2d6449 │ 2025-09-23 18:00:01.096077+08 │
└───────┴──────────────────────────────────┴───────────────────────────────┘
D 
```

## pg_parquet
[pg_parquet](https://github.com/CrunchyData/pg_parquet)这也是PostgreSQL的插件，同pg_duckdb一致的用法，增强了PostgreSQL的COPY，不过它是一个rust写的插件。这里就贴一下官方的测试案例好了。
```sql
-- create composite types
CREATE TYPE product_item AS (id INT, name TEXT, price float4);
CREATE TYPE product AS (id INT, name TEXT, items product_item[]);

-- create a table with complex types
CREATE TABLE product_example (
    id int,
    product product,
    products product[],
    created_at TIMESTAMP,
    updated_at TIMESTAMPTZ
);

-- insert some rows into the table
insert into product_example values (
    1,
    ROW(1, 'product 1', ARRAY[ROW(1, 'item 1', 1.0), ROW(2, 'item 2', 2.0), NULL]::product_item[])::product,
    ARRAY[ROW(1, NULL, NULL)::product, NULL],
    now(),
    '2022-05-01 12:00:00-04'
);

-- copy the table to a parquet file
COPY product_example TO '/tmp/product_example.parquet' (format 'parquet', compression 'gzip');

-- show table
SELECT * FROM product_example;

-- copy the parquet file to the table
COPY product_example FROM '/tmp/product_example.parquet';

-- show table
SELECT * FROM product_example;
```
