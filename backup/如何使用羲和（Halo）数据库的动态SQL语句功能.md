一、前言
====

SQL语句在**程序编译**期间就已经确定，绝大多数的编译情况属于这种类型，我们称之为静态SQL。

而与之对应的动态SQL则代表着是“不确定”的SQL，待执行的SQL语句或匿名块只有在**程序运行**时才真正被确定。

也正是因为如此，所以动态SQL在带来了极高的灵活性的同时也带来了不小的复杂性。今天咱们不探讨关于动态SQL的最佳实践，先来学习动态SQL的简单使用。

PL/SQL中存在两种关于动态SQL的实现：

*   Native dynamic SQL（NDS）：在PL/SQL中一般使用EXECUTE IMMEDIATE或OPEN - FOR相关语句来执行本地动态SQL语句。
*   DBMS\_SQL: 内置的系统Package，是一组用于构建、运行和描述动态 SQL 语句的API（不是今天的重点、感兴趣可阅读[羲和（Halo）数据库——DBMS\_SQL浅尝](https://www.modb.pro/db/1743552160716120064)）

接下来让我们来学习一下，在羲和（Halo）数据库中如何正确的使用Native dynamic SQL。

  

二、环境准备
======

现在演示的版本为羲和（halo）14版本，建议当database\_compat\_mode设置为Oracle时，建议使用hsql对羲和（halo）数据库进行操作

当database\_compat\_mode设置为PostgreSQL时，建议使用psql对羲和（halo）数据库进行操作  

本次演示具体环境准备如下所示：

    [halo@halo-centos-8-release 14]$ hsql
    psql (1.0.14.12 (240604))
    Type "help" for help.
    
    halo0root=# -- 创建数据库halo
    halo0root=# CREATE DATABASE halo;
    CREATE DATABASE
    halo0root=# -- 切换至新创建的halo数据库
    halo0root=# \c halo
    You are now connected to database "halo" as user "halo".
    halo=# -- 确认当前模式为Oracle
    halo=# SHOW database_compat_mode;
     database_compat_mode 
    ----------------------
     oracle
    (1 row)
    
    halo=# -- 创建羲和的aux_oracle组件
    halo=# CREATE EXTENSION AUX_ORACLE CASCADE;
    NOTICE:  installing required extension "plorasql"
    NOTICE:  installing required extension "pgcrypto"
    
    CREATE EXTENSION
    halo=# -- 安装完相关组件后 切换一下
    halo=# \c - -
    You are now connected to database "halo" as user "halo".
    halo=# -- 开启动态SQL功能，将参数oracle._enable_named_placeholder 置为on 
    halo=# SET oracle._enable_named_placeholder = on;
    SET
    halo=# 

值得注意的是，使用SET语句设置的变量是会话级别的，当会话退出之后，将会参数将会恢复默认设置。

也可以选择使用如下语句，将参数设置数据库级别，就能避免参数恢复成默认设置了。

    halo=# ALTER DATABASE halo SET oracle._enable_named_placeholder = on;
    ALTER DATABASE
    halo=# -- 设置数据库级别的参数需要重新切换一下 
    halo=# \c - -
    You are now connected to database "halo" as user "halo".

  

三、EXECUTE IMMEDIATE
===================

EXECUTE IMMEDIATE语法如下：

    EXECUTE IMMEDIATE '<dynamic_sql_stmt>;'
      [ INTO { <variable> [, ...] | <record> } ]
      [ USING {[<bind_type>] <bind_argument>} [, ...]} ];

**可以看到INTO子句和USING子句都是可选项，至于何时使用我们需要关注下****dynamic\_sql\_stmt**

**dynamic\_sql\_stmt**：是一个字符串表达式，其中包含待执行的动态SQL 语句。

当**dynamic\_sql\_stmt** 表达式中未使用占位符时，无需添加任何子句，而如果使用了占位符时，则需要依据实际情形，选择合适的子句配合进行相应的处理。

  

2.1.1、无占位符的动态SQL语句
------------------

可直接使用EXECUTE IMMEDIATE执行，如存在如下示例

    DECLARE
        v_sql           VARCHAR2(50);
    BEGIN
        EXECUTE IMMEDIATE 'CREATE TABLE job (jobno NUMBER(3),' ||
            ' jname VARCHAR2(9))';
        v_sql := 'INSERT INTO job VALUES (100, ''ANALYST'')';
        EXECUTE IMMEDIATE v_sql;
        v_sql := 'INSERT INTO job VALUES (200, ''CLERK'')';
        EXECUTE IMMEDIATE v_sql;
    END;
    /

运行结果如图：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240716-dee63ae2-4931-4f6a-922d-4429cb7b1f5f.png)  

  

2.1.2、存在占位符的动态SQL
-----------------

当存在占位符时，语法规则就变得复杂多了，可以依据具体情形来使用不同的子句配合。

在了解子句前，我们下来看看占位符的使用规则，占位符的命名规则为以冒号（:）作为前缀，后续可接字母、数字、标识符等等 例如 :1、:a、:name等等

当存在占位符时 我们可以USING子句输入输出数据 接下来使用USING子句演示一下上述内容：

    DECLARE
        v_sql           VARCHAR2(50); 
        v_jobno         job.jobno%TYPE;
        v_jname         job.jname%TYPE;
    BEGIN
        -- 字母作为占位符
        v_sql   := 'INSERT INTO job VALUES ' || '(:a, :b)';
        v_jobno := 300;
        v_jname := 'MANAGER';
        EXECUTE IMMEDIATE v_sql USING v_jobno, v_jname;
    
        -- 数字作为占位符
        v_sql   := 'INSERT INTO job VALUES ' || '(:1, :2)';
        v_jobno := 400;
        v_jname := 'SALESMAN';
        EXECUTE IMMEDIATE v_sql USING v_jobno, v_jname;
    
        -- 标识符作为占位符
        v_sql   := 'INSERT INTO job VALUES ' || '(:p_jobno, :p_jname)';
        v_jobno := 500;
        v_jname := 'PRESIDENT';
        EXECUTE IMMEDIATE v_sql USING v_jobno, v_jname;
    END;
    /

运行结果如下图所示：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240716-407700d0-61fe-4615-81af-77e971c4743c.png)  

在SQL场景中，占位符就是占位符，与实际占位符的名称没有任何关系，比如说

    v_sql   := 'INSERT INTO job VALUES ' || '(:1, :1)';

虽然上面占位符都叫（:1）, 但是实际使用时，必须传递两个数据才行，否则将会报错（**很奇怪吧，不要问我为什么，主要是Oracle的设计便是如此**）

    DECLARE
        v_sql           VARCHAR2(50); 
        v_jobno         job.jobno%TYPE;
        v_jname         job.jname%TYPE;
    BEGIN
    	v_sql   := 'DELETE FROM job WHERE jobno = :p_jobno';
        v_jobno := 500;
        EXECUTE IMMEDIATE v_sql USING v_jobno;
    	-- 同名占位符
    	v_sql   := 'INSERT INTO job VALUES ' || '(:1, :1)';
        v_jobno := 500;
        v_jname := 'PRESIDENT';
        EXECUTE IMMEDIATE v_sql USING v_jobno, v_jname; -- 必须传递两个值
    END;
    /

运行结果如下图所示：  

![](https://oss-emcsprod-public.modb.pro/image/editor/20240716-80ece440-ae01-4b02-ab53-bbf95bcc659c.png)  

同时使用USING子句和INTO子句配合使用，查询job表中数据并输出显示

    DECLARE
        v_sql           VARCHAR2(60);
        v_jobno         job.jobno%TYPE;
        v_jname         job.jname%TYPE;
        r_job           job%ROWTYPE;
    BEGIN
        DBMS_OUTPUT.PUT_LINE('JOBNO    JNAME');
        DBMS_OUTPUT.PUT_LINE('-----    -------');
        v_sql := 'SELECT jobno, jname FROM job WHERE jobno = :p_jobno';
        EXECUTE IMMEDIATE v_sql INTO v_jobno, v_jname USING 100;
        DBMS_OUTPUT.PUT_LINE(v_jobno || '      ' || v_jname);
        EXECUTE IMMEDIATE v_sql INTO v_jobno, v_jname USING 200;
        DBMS_OUTPUT.PUT_LINE(v_jobno || '      ' || v_jname);
        EXECUTE IMMEDIATE v_sql INTO v_jobno, v_jname USING 300;
        DBMS_OUTPUT.PUT_LINE(v_jobno || '      ' || v_jname);
        EXECUTE IMMEDIATE v_sql INTO v_jobno, v_jname USING 400;
        DBMS_OUTPUT.PUT_LINE(v_jobno || '      ' || v_jname);
        EXECUTE IMMEDIATE v_sql INTO r_job USING 500;
        DBMS_OUTPUT.PUT_LINE(r_job.jobno || '      ' || r_job.jname);
    END;
    /

结果如下图所示：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240716-0068e955-3b1b-4844-84ce-c4746e118fa0.png)  

可能有人好奇USING子句中的bind\_type是啥，其实就是IN、OUT、INOUT这些，看一下例子就好了

    -- 更改参数deptid值
    CREATE OR REPLACE PROCEDURE create_dept(
      deptid IN OUT NUMBER,
      dname  IN     VARCHAR2
    ) AUTHID DEFINER AS
    BEGIN
      DBMS_OUTPUT.put_line('deptid : '||deptid);
      DBMS_OUTPUT.put_line('dname  : '||dname);
      deptid := 255;
    END;
    /
    
    -- 传入数据 并打印输出数据
    DECLARE
      plsql_block VARCHAR2(500);
      new_deptid  NUMBER(4)    := 99;
      new_dname   VARCHAR2(30) := 'Advertising';
    BEGIN
      plsql_block := 'call create_dept(:a, :b)';
      EXECUTE IMMEDIATE plsql_block
        USING IN OUT new_deptid, new_dname;
      DBMS_OUTPUT.put_line('new_deptid '||new_deptid);
    END;
    /

结果如下图所示：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240716-45541159-2597-4a0a-b6d9-9e99039c81ea.png)  

  

四、OPEN - FOR
============

其实就是使用游标来处理动态语句。

语法图如下：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240716-61efc481-dfa1-42aa-ad8c-3b08ce05bb18.png)  

看一下在动态SQL场景，关于OPEN - FOR的使用，请看如下示例

    DECLARE
        v_sql           VARCHAR2(60);
        v_jobno         job.jobno%TYPE;
        v_jname         job.jname%TYPE;
        r_job           job%ROWTYPE;
        TYPE JobCurTyp  IS REF CURSOR;
        v_job_cursor    JobCurTyp  ;  
    BEGIN
        DBMS_OUTPUT.PUT_LINE('JOBNO    JNAME');
        DBMS_OUTPUT.PUT_LINE('-----    -------');
    
        v_sql := 'SELECT jobno, jname FROM job WHERE jobno = :p_jobno';
        OPEN v_job_cursor FOR v_sql USING 100;
        LOOP
          FETCH v_job_cursor INTO v_jobno, v_jname;
          EXIT WHEN v_job_cursor%NOTFOUND;
          DBMS_OUTPUT.PUT_LINE(v_jobno || '      ' || v_jname);
        END LOOP; 
    
        CLOSE v_job_cursor;
    END;
    /

使用OPEN - FOR查询jobno为100的记录，并打印输出。结果如下图所示

![](https://oss-emcsprod-public.modb.pro/image/editor/20240716-16d97b56-10f2-4576-97fe-c618daeaaa28.png)  

  

五、声明
====

**若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。**

**文章转载请联系，谢谢合作。**