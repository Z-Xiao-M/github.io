一、场景复现
======

开门见山，一个关于oracle\_fdw的简单场景如下：

    halo0root=# \d+ emp
                                                   Foreign table "public.emp"
     Column |         Type          | Collation | Nullable | Default | FDW options  | Storage  | Stats target | Description 
    --------+-----------------------+-----------+----------+---------+--------------+----------+--------------+-------------
     name   | character varying(20) |           | not null |         | (key 'true') | extended |              | 
     age    | numeric               |           |          |         |              | main     |              | 
    Server: oradb
    FDW options: (schema 'ZM', "table" 'EMP')
    
    halo0root=# \des+
                                                          List of foreign servers
     Name  | Owner | Foreign-data wrapper | Access privileges | Type | Version |               FDW options                | Description 
    -------+-------+----------------------+-------------------+------+---------+------------------------------------------+-------------
     oradb | halo  | oracle_fdw           |                   |      |         | (dbserver '//10.16.6.188:1521/orclpdb1') | 
    (1 row)
    
    halo0root=# EXPLAIN VERBOSE DELETE FROM emp WHERE age = 25;
                                                            QUERY PLAN                                                         
    ---------------------------------------------------------------------------------------------------------------------------
     Delete on public.emp  (cost=10000.00..20000.00 rows=0 width=0)
       Oracle statement: DELETE FROM "ZM"."EMP" WHERE "NAME" = :k1
       ->  Foreign Scan on public.emp  (cost=10000.00..20000.00 rows=1000 width=58)
             Output: name
             Oracle query: SELECT /*9ad36f25074bd711*/ r1."NAME", r1."AGE" FROM "ZM"."EMP" r1 WHERE (r1."AGE" = 25) FOR UPDATE
             Oracle plan: SELECT STATEMENT
             Oracle plan:   FOR UPDATE
             Oracle plan:     BUFFER SORT
             Oracle plan:       TABLE ACCESS FULL EMP  (filter "R1"."AGE"=25)
     Query Identifier: -1174117518881599247
    (10 rows)

这里可以很明显的看到，虽然我们敲得是EXPLAIN VERBOSE DELETE ...，但是实际的Oracle query却是SELECT，同时UPDATE也是如此

    halo0root=# EXPLAIN VERBOSE UPDATE emp SET name = 'zzz' WHERE age = 25;
                                                            QUERY PLAN                                                         
    ---------------------------------------------------------------------------------------------------------------------------
     Update on public.emp  (cost=10000.00..20000.00 rows=0 width=0)
       Oracle statement: UPDATE "ZM"."EMP" SET "NAME" = :p1 WHERE "NAME" = :k1
       ->  Foreign Scan on public.emp  (cost=10000.00..20000.00 rows=1000 width=230)
             Output: 'zzz'::character varying(20), name, emp.*
             Oracle query: SELECT /*9ad36f25074bd711*/ r1."NAME", r1."AGE" FROM "ZM"."EMP" r1 WHERE (r1."AGE" = 25) FOR UPDATE
             Oracle plan: SELECT STATEMENT
             Oracle plan:   FOR UPDATE
             Oracle plan:     BUFFER SORT
             Oracle plan:       TABLE ACCESS FULL EMP  (filter "R1"."AGE"=25)
     Query Identifier: -2711433616002583057
    (10 rows)

那么问题就如文章标题所述了，这是为什么呢？

  

二、原理分析
======

原因其实也很简单，因为对于PostgreSQL而言，外部表是一种很特殊的存在，它仅仅只是一个远端表的映射，实际的数据依旧还是存储在远端，

当我们在本地的PostgreSQL访问外部表的时候，实际上就是将当前的查询语句，下发至远端数据库，再将查询到的数据传回本地的PostgreSQL中参与实际的计算之类的。

但是说到这还仅仅只是说明了一下查询的原理，实际像我们上面那个场景还不足以解释清楚，为什么我想要执行的是UPDATE、DELETE，实际下发的确实SELECT呢？

是的，要想将上面的问题解答清楚，我们还需要和其余的一些内核知识串联一下。

回顾一下PostgreSQL，它是一款发非常强大的关系型数据库，虽然现在我们在很多的场景都是谈论它的拓展能力有多强大，各种各样的插件百花齐放，

但是其实我觉得它能做到这么强大的原因在于它的基础（OLTP）打得足够的好，也正是由于它的基础足够的好，反而能够衍生出多种多样的插件。

称之为OLTP的王者其实也不为过，那么聊到了OLTP，对于PostgreSQL内核的执行器而言，必然就是经典的火山模型，它处理数据只能一行一行的处理，不论是执行的是

DELETE操作还是UPDATE操作，都是如此。

因此对于外部表进行DELETE或UPDATE操作，必须要确保符合条件的数据是否存在，所以必须先通过查询获取到远端符合条件的数据，然后根据符合条件的数据再去最终删除或者更新

远端的表。一个简答简单的验证方法便是将日志等级调低，查看一下日志信息即可，如下所示

    halo0root=# INSERT INTO emp VALUES('xm', 18);
    INSERT 0 1
    halo0root=# SELECT * FROM emp;
     name | age 
    ------+-----
     xm   |  18
    (1 row)
    
    halo0root=# set client_min_messages='debug2';
    LOG:  duration: 0.466 ms  statement: set client_min_messages='debug2';
    SET
    halo0root=#  UPDATE emp SET name = 'zm' WHERE age = 18; -- UPDATE
    DEBUG:  oracle_fdw: add target columns for update on 32191
    DEBUG:  oracle_fdw: plan foreign table scan
    DEBUG:  oracle_fdw: set NLS_LANG=AMERICAN_AMERICA.AL32UTF8
    DEBUG:  oracle_fdw: begin remote transaction
    DEBUG:  oracle_fdw: remote query is: SELECT /*d339c4b693f43f0c*/ r1."NAME", r1."AGE" FROM "ZM"."EMP" r1 WHERE (r1."AGE" = 18) FOR UPDATE
    DEBUG:  oracle_fdw: remote statement is: UPDATE "ZM"."EMP" SET "NAME" = :p1 WHERE "NAME" = :k1
    DEBUG:  oracle_fdw: begin foreign table scan on 32191
    DEBUG:  oracle_fdw: begin foreign table modify on 32191
    DEBUG:  oracle_fdw: execute query in foreign table scan 
    DEBUG:  oracle_fdw: end foreign table modify on 32191
    DEBUG:  oracle_fdw: end foreign table scan
    DEBUG:  oracle_fdw: commit remote transaction
    LOG:  duration: 26.098 ms  statement: UPDATE emp SET name = 'zm' WHERE age = 18;
    UPDATE 1
    halo0root=# DELETE FROM emp WHERE name = 'zm';
    DEBUG:  oracle_fdw: add target columns for update on 32191
    DEBUG:  oracle_fdw: plan foreign table scan
    DEBUG:  oracle_fdw: set NLS_LANG=AMERICAN_AMERICA.AL32UTF8
    DEBUG:  oracle_fdw: begin remote transaction
    DEBUG:  oracle_fdw: remote query is: SELECT /*cd06aea77148cf7d*/ r1."NAME" FROM "ZM"."EMP" r1 WHERE (r1."NAME" = 'zm') FOR UPDATE
    DEBUG:  oracle_fdw: remote statement is: DELETE FROM "ZM"."EMP" WHERE "NAME" = :k1
    DEBUG:  oracle_fdw: begin foreign table scan on 32191
    DEBUG:  oracle_fdw: begin foreign table modify on 32191
    DEBUG:  oracle_fdw: execute query in foreign table scan 
    DEBUG:  oracle_fdw: end foreign table modify on 32191
    DEBUG:  oracle_fdw: end foreign table scan
    DEBUG:  oracle_fdw: commit remote transaction
    LOG:  duration: 16.580 ms  statement: DELETE FROM emp WHERE name = 'zm';
    DELETE 1

事实上，postgres\_fdw的实现逻辑也是如此，不过在查询时直接返回的是远端对应那行数据的ctid，然后当执行更新或者删除操作的时候，根据ctid去处理即可，感兴趣的朋友可以自己动手实践看看。

同时可能有的朋友会想到：那oracle\_fdw也可以这样子做呀，根据条件查询的时候，将对应的rowid返回，然后真正去执行更新或者删除操作时，根据rowid去做即可。但是看起来oracle\_fdw并没有这么做，就像上面的示例一般，最终更新或者删除确实依据NAME这个字段，这是为啥？

我只想说这是个好问题，如果注意力足够集中的话，可以发现的是name字段对应的FDW options的内容为(key 'true')，这是oracle\_fdw需要更新或者删除操作的必要条件。

    halo0root=# \d+ emp
                                                   Foreign table "public.emp"
     Column |         Type          | Collation | Nullable | Default | FDW options  | Storage  | Stats target | Description 
    --------+-----------------------+-----------+----------+---------+--------------+----------+--------------+-------------
     name   | character varying(20) |           | not null |         | (key 'true') | extended |              | 
     age    | numeric               |           |          |         |              | main     |              | 
    Server: oradb
    FDW options: (schema 'ZM', "table" 'EMP')

其实name字段是这张表的主键。

试想一下，当我们返回的是rowid，然后根据rowid去删除，在PostgreSQL中依据ctid去操作能很快的定位到对应的数据，但是对于Oracle而言，可能就需要进行全表扫描了（我猜的）。

而Oracle\_fdw的作者，然后搭建在Oracle中给表创建好合适的主键或者联合组建，然后在PostgreSQL中创建外部表的时候标识对应的主键字段。

当我们执行删除或者更新操作的时候，我们第一次查询便可以返回对应的字段数据，后面依据返回字段数据中的主键字段数据，便可以根据主键去快速定位到具体的行，然后去执行相应的操作（完结撒花）。

其实还有很多细节，大家可以自行消化一下，比方说在整个操作都是在一整个的事务中，然后还有就是使用的是SELECT ... FOR UPDATE等等之类的，这些我就不说了，dddd。

  

三、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文章转载请联系，谢谢合作~