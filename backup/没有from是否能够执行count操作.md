一、场景构建
======

这是一个非常简单的场景，这里存在一张表，表名t，往其中随便插入三行数据，然后执行查询语句：`select count(*) from t;`

    create table t(id integer);
    insert into t values(1),(2),(3);
    select count(*) from t;

执行结果

    postgres=# create table t(id integer);
    CREATE TABLE
    postgres=# insert into t values(1),(2),(3);
    INSERT 0 3
    postgres=# select count(*) from t;
     count 
    -------
         3
    (1 row)

可以看到执行count结果为3，这在正常不过了。

问：如果去掉from，问 `select count(*) t;` 能否执行？

如果再去掉t，问 `select count(*)` 能否执行？

如果要是能执行，预期的结果又是多少呢？

  

二、结果公布
======

这里也就不拖拖拉拉，直接放运行结果

    postgres=#  select count(*) t;
     t 
    ---
     1
    (1 row)
    
    postgres=#  select count(*);
     count 
    -------
         1
    (1 row)
    
    postgres=# 

结果是否符合你的预期呢？所以上面的答案就是两条SQL都能执行，且执行结果都为一。

有经验的老师傅，可能已经反应过来了，没看懂的朋友也不用着急，且听我慢慢道来。

本质上来说，执行count操作其实也就是执行count这个聚合函数，

而第一条简单SQL，去掉from的版本 `select count(*) t;`其实就是给count(\*)取了一个别名为t。

    postgres=# \df count
                             List of functions
       Schema   | Name  | Result data type | Argument data types | Type 
    ------------+-------+------------------+---------------------+------
     pg_catalog | count | bigint           |                     | agg
     pg_catalog | count | bigint           | "any"               | agg
    (2 rows)
    
    postgres=# select pg_catalog.count(*) as t;
     t 
    ---
     1
    (1 row)

而第二条SQL就更没啥好说的了，接下来说说为什么结果是一。

这个其实会Oracle的人可能更容易理解，在早先的时候，Oracle执行函数和PostgreSQL执行函数实现的想法不一样。

在Oracle中调用函数并不允许省略from子句，一般的场景下都是from dual;

    SQL> select count(*);
    select count(*)
                  *
    ERROR at line 1:
    ORA-00923: FROM keyword not found where expected
    
    
    SQL> select count(*) from dual;
    
      COUNT(*)
    ----------
             1
    
    SQL> select count(*) t from dual;
    
             T
    ----------
             1

而对于PostgreSQL而言是可以省略掉from子句的

![](https://oss-emcsprod-public.modb.pro/image/editor/20250804-1952276669360779264_585460.png)

所以上面的select count(\*) 就好似PostgreSQL隐含附加上了一个单行的表，所以count的结果是一。

事实上像PostgreSQL这种方式用起来非常的方便，以至于后面Oracle在23版本之后也支持了这种方式来执行函数或表达式。

  

Oracle23版本之前执行结果如下：

![](https://oss-emcsprod-public.modb.pro/image/editor/20250804-1952280153803993088_585460.png)

Oracle23版本之后

![](https://oss-emcsprod-public.modb.pro/image/editor/20250804-1952280611452891136_585460.png)

  

三、多个from
========

在讲完没有from子句也是可以执行count操作，以及执行的count操作结果始终为一之外。

这里再简单拓展一下场景，事实上from子句并不是作为一条SQL的必要选项，对于一条简单的SQL而言，

可以没有from，可以有一个from（最常见），也可以存在多个from （因为from可以是表达式）。下面给出NULL相等判断的一个例子。

    postgres=# CREATE TABLE disttest (x INTEGER, y INTEGER);
    CREATE TABLE
    postgres=# INSERT INTO disttest VALUES (1, 1), (2, 3), (NULL, NULL);
    INSERT 0 3
    postgres=# SELECT x = y  FROM disttest;
     ?column? 
    ----------
     t
     f
     
    (3 rows)
    
    postgres=# SELECT x IS NOT DISTINCT FROM y FROM disttest;
     ?column? 
    ----------
     t
     f
     t
    (3 rows)
    

  

四、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

感觉这篇文章可以和[https://mp.weixin.qq.com/s/nKUA6uBuMI8VGSSAkwuYpg](https://mp.weixin.qq.com/s/nKUA6uBuMI8VGSSAkwuYpg)一起看。

文章转载请联系，谢谢合作~