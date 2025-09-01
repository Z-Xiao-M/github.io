一、前言
====

文章标题为PostgreSQL——关于autocommit功能的实现，其实更为准确来说应该是psql——关于autocommit功能的实现，因为这个参数其实属于psql。

对于autocommit，相信绝大多数的DBA同学都不会感到陌生。因为不管是Oracle数据库还是PostgreSQL数据库（估计关系型数据库）都存在这麽一个参数。

今天我们来看一看在PostgreSQL中，这个功能是如何实现的。在了解这个功能之前，我们先铺垫一下autocommit参数的查看和使用。

  

二、autocommit参数的查看和使用
====================

2.1、查看autocommit参数
------------------

在PostgreSQL中查看该参数我们可以使用\\set，前提是我们使用psql正确登录成功连接会话进程

    [postgres@halo-centos-8-release ~]$ psql
    psql (17beta1)
    Type "help" for help.
    
    postgres=# \set
    AUTOCOMMIT = 'on'
    COMP_KEYWORD_CASE = 'preserve-upper'
    DBNAME = 'postgres'
    ECHO = 'none'
    ECHO_HIDDEN = 'off'
    ......
    postgres=#  

而在Oracle数据库中我们可通过show autocommit; 比方说使用sqlplus / as sysdba登录连接Oracle数据库

    SQL> show autocommit;
    autocommit OFF

可以看到的是，在PostgreSQL中，autocommit功能是默认开启的，在这种情形之下，每条SQL语句都自成一个事务，当SQL语句一旦执行完便提交此事务。

而当我们想将多个SQL语句，组成一个大的事务操作时，可以将这多个SQL语句置于BEGIN；... END/COMMIT;

    BEGIN;
    -- 相关SQL操作
    END; -- COMMIT；

而在Oracle中autocommit则是默认关闭的，当进行了一个或多个修改数据的操作时，需要在最后显式使用COMMIT命令，不然当连接断开时，之前对数据库进行的相关操作将会被回滚。（注：DDL除外，在Oracle数据库中DDL隐含了COMMIT操作）

    -- 一个或多个操作
    COMMIT；

如果PostgreSQL想和Oracle在这方面保持一致的话，简单修改一下autocommit参数，将其置为off即可，接下来介绍一下如何修改该参数。

  

2.2、修改autocommit参数设置
--------------------

在PostgreSQL中修改该参数我们可以使用\\set AUTOCOMMIT off/on （区分大小写），如下所示

    [postgres@halo-centos-8-release ~]$ psql
    psql (17beta1)
    Type "help" for help.
    
    postgres=# \set AUTOCOMMIT off
    postgres=# \set
    AUTOCOMMIT = 'off'
    COMP_KEYWORD_CASE = 'preserve-upper'
    DBNAME = 'postgres'
    ......

在Oracle数据库中修改该参数我们可以使用set autocommit off/on; （分号可写可不写），如下所示  

    SQL> set autocommit on;
    SQL> show autocommit;
    autocommit IMMEDIATE
    SQL> set autocommit off;
    SQL> show autocommit;
    autocommit OFF

如上文所述，当我们将PostgreSQL的autocommit置为off时，表现和Oracle一致，当存在一条或多条和修改数据相关的SQL语句，我们需要显示使用COMMIT/END命令，对当前的相关操作进行事务提交操作。

通常来说一个比较好的写法就是将相关操作都写如形如begin transaction 和end transaction 语句之间，当执行到end语句（或其他如commit语句）时，事务自动提交。

接下来让我们思考一下PostgreSQL的autocommit的实现，为什么当PostgreSQL的autocommit置为off时，使用end或commit能将整个事务进行提交。

  

三、如何实现？
=======

这个问题其实挺简单的，估计有不少同学都直接想明白了。首先既然能够直接使用END或者COMMIT进行事务提交，那么当前必然处于一个事务中，那么必然存在开启事务的操作，而我们又并没有进行这个操作。（心机之蛙......，请忽略此处😁）

那么这个操作肯定是有人代劳了，而autocommit是psql的一个参数，那么肯定是psql帮忙发起了一个begin transaction的操作。

那如何验证我们的猜想呢？

接下来我打算从两个角度来验证一下我们上面的猜想

3.1、日志分析
--------

第一种是从日志进行分析，来窥探一下是否存在BEGIN的操作，这种也是相对简便的方法

    -- 首先我们先确定数据库中存在一张可进行操作的表
    CREATE TABLE ta(a INTEGER);

    -- 然后将几个和日志相关的参数打开
    log_statement = 'all'
    logging_collector = on
    log_directory = 'log' 
    log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' 
    log_file_mode = 0600 

    -- 之后进行重启或者加载配置文件的操作
    pg_ctl restart

开始验证

    [postgres@halo-centos-8-release 17]$ psql
    psql (17beta1)
    Type "help" for help.
    
    postgres=# \set AUTOCOMMIT off
    postgres=# -- 插入一行数据
    postgres=# INSERT INTO ta VALUES(1);
    INSERT 0 1
    postgres=*# -- COMMIT
    postgres=*# COMMIT;
    COMMIT

接下来查看一下日志文件

![](https://oss-emcsprod-public.modb.pro/image/editor/20240909-1833077784383676416_585460.png)  

可以看到的是，虽然我们并没有显式的写上BEGIN；但是日志中在INSERT语句之前确存在一个BEGIN。和我们的猜想一致。

  

3.2、查看源码
--------

这种比较直接，但是对于没接触过内核代码的同学，可能还是不太能完全理解。不过没关系，我们来简单分析一下。

开头我们便有说到，其实这个autocommit参数是属于psql的，所以我们应该查看psql的实现，而这个大概的代码也很明晰，代码位于src/bin/psql/common.c的SendQuery函数中

![](https://oss-emcsprod-public.modb.pro/image/editor/20240909-1833080908262416384_585460.png)  

可以看到当autocommit为off，结合相关条件，psql会通过libpq的接口，发送一个BEGIN的命令

![](https://oss-emcsprod-public.modb.pro/image/editor/20240909-1833082149798502400_585460.png)  

而BEGIN整位于TransactionStmtLegacy的语法规则中，而你又恰巧对词法语法有所了解的话，可以看到这个BEGIN便被移进规约处理完毕了

因此这个BEGIN其实就等价于在psql中输入BEGIN；

如此看来，PostgreSQL的autocommit的实现，和我们的猜想一致，当我们将autocommit设置成off时，当我们第一次执行相关SQL操作时，

psql会先发送一个BEGIN命令给数据库进行处理，开启一个事务，直到我们输入其他和事务相关的命令（END/COMMIT/ROLLBACK等等）时，才会处理整个事务。

  

四、为什么在psql手动输入（BEGIN）并没有进入输入（BEGIN;）事务状态呢？
==========================================

我想或许可能有部分同学对这个会存在一点小小的疑问，可能在想 "那按照这么说的话（BEGIN）和（BEGIN；）一致，那为什么在psql手动输入（BEGIN）并没有进入输入（BEGIN;）事务状态呢？"

其实如果对这个存在疑问的话，说明其实对psql还是存在一点没有理解到位的地方。

对于psql而言，一个最基本也是一个最重要的功能便是能够识别和正确处理需要发送给数据库服务端的相关内容信息，其实说白了也就是说期待一个合法的结束。

PostgreSQL的通信协议为Frontend/Backend protocol也简称FE/BE协议，psql通过libpq便是使用这个协议和真正的数据库服务端（会话进程）进行通信。

而这个通信的前提便在于，psql能正确识别它所认可的结束，详见psqlscan.l。可以理解为一堆正则表达式的堆积成的简易词法分析处理，但并不是真正意义上的词法分析，因为此时并不涉及真正的数据库内核。

    [postgres@halo-centos-8-release log]$ psql
    psql (17beta1)
    Type "help" for help.
    
    postgres=# begin --非合法的结束
    postgres-# ^C
    postgres=# 
    postgres=# begin; -- 合法的结束
    BEGIN
    postgres=*# end; -- 合法的结束
    COMMIT
    postgres=# 

通过通信协议发送到数据库服务端（会话进程），才开始正在意义的词法语法语义、查询重写、优化等等一系列的我们所熟知的相关SQL声明周期。

所以如果单纯在psql中输入BEGIN，此时psql都还没正常结束，更加无法将其通过通信协议发送给会话进程处理。如果想看到一样的效果，需要通过libpq的接口发送BEGIN才真正意义上等价于在psql中输入BEGIN; （可参考src/test/examples/testlibpq.c） 

![](https://oss-emcsprod-public.modb.pro/image/editor/20240909-1833097828727611392_585460.png)  

  

五、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。  
  
文章转载请联系，谢谢合作。