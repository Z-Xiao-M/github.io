本文将向大家展示如何在羲和（Halo）数据库中使用DBMS\_SQL，本文测试案例除部分表结构和部分测试数据之外，其余均来自ORACLE官方测试文档，所有测试案例放置文章末尾以供学习使用。  

  

**一、DBMS\_SQL**

DBMS\_SQL是ORACLE数据库提供的一个系统包，用于执行动态SQL，支持使用DDL和DML等。DBMS\_SQL整体运行流程图如下：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-3db3a9ae-1043-4524-8988-8eaa5625b7db.gif)  

下面简单介绍一下本文中所使用到的DBMS\_SQL部分接口，如果查看更为详细的介绍，建议参考ORACLE官方文档。

*   **OPEN\_CURSOR**：要处理SQL语句，必须有一个打开的游标。通过调用OPEN\_CURSOR ，获取一个数据库内部的维护游标编号。当不再需要使用的时候，需要调用CLOSE\_CURSOR关闭
    

    c := DBMS_SQL.OPEN_CURSOR;

*   **PARSE**：解析待执行的动态SQL语句，检查语句语法是否存在问题，并将其与程序中的游标进行关联。
    

    DBMS_SQL.PARSE(c, 'INSERT INTO tab VALUES (:bnd1, :bnd2) ' ||
                              'RETURNING c1*c2 INTO :bnd3', DBMS_SQL.NATIVE);

*   **BIND\_VARIABLE**：用于将特定的值或变量与待执行的SQL语句中的占位符关联起来。
    

    DBMS_SQL.BIND_VARIABLE(c, 'bnd1', c1);

*   **BIND\_ARRAY**：用于将数组变量与待执行的SQL 语句中的占位符关联起来。
    

    DBMS_SQL.BIND_ARRAY(c, 'bnd3', r);

*   **DEFINE\_COLUMN**：用于定义待执行的SELECT语句的最终返回结果集中的列。
    

    DBMS_SQL.DEFINE_COLUMN(source_cursor, 1, id_var);

*   **EXECUTE**：用于执行已解析SQL。
    

    n := DBMS_SQL.EXECUTE(c);

*   **FETCH\_ROWS**：用于从指定游标中获取一行数据。
    

    IF DBMS_SQL.FETCH_ROWS(source_cursor)>0 THEN 
       -- get column values of the row

*   **EXECUTE\_AND\_FETCH**：用于执行已解析的查询SQL，并获取一行数据。
    

    r := DBMS_SQL.EXECUTE_AND_FETCH(c);

*   **VARIABLE\_VALUE**：用于从带有RETURNING的SQL语句中获取绑定变量的返回数据。
    

    DBMS_SQL.VARIABLE_VALUE(c, 'bnd3', r);

*   **COLUMN\_VALUE**：用于获取查询结果集中指定列的数据。
    

    DBMS_SQL.COLUMN_VALUE(c, 1, some_dnames);

*   **CLOSE\_CURSOR**：终章！关闭游标。
    

    DBMS_SQL.CLOSE_CURSOR(c);

 接下来我将使用羲和（Halo）数据库演示上述DBMS\_SQL的各个接口。  

  

**二、环境准备**

羲和（Halo）数据库设置为ORACLE模式，创建一个演示数据库test\_db，创建内部所需的ORACLE拓展。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-df125093-61e5-474c-a9b9-f16f24efe647.png)  

当需要使用oracle相关的特性，建议直接使用hsql进行登录使用。所以此处使用hsql -dtest\_db，登录test\_db数据库进行接下来的相关操作。  

**三、实际内容演示**

*   **single\_Row\_insert**：单行插入，插入数据"一五一十"。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-99d65713-87b0-45b5-aa93-1b71ea311fa9.png)  

*   **single\_Row\_update**：将数据更新成5，5。
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-336403a9-3cf1-494e-b861-5c9adb77e884.png)  

*   **single\_Row\_Delete**: 删除一行数据
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-4013e4a4-aa78-41e2-9aea-a80bb66a2860.png)  

*   **Multiple-row insert**: 往c1中插入10、20、30，c2中插入11、21、31
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-99d5da77-15a0-46ea-93d3-86ed3f8bc802.png)  

为multi\_Row\_update做准备，再次运行multi\_Row\_insert插入重复数据。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-bf7ef716-4e95-43c7-9019-b1f81af5e162.png)  

*   **multi\_Row\_update**：将c2为11，对应的c1更新成100
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-6d01eb1e-65d6-49af-b3d7-3a58a17dacd6.png)  

*   **Multiple-row delete**：多行删除
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-ece6a99e-47c7-4a47-9dae-9c5ab0296900.png)  

*   **exec**：支持运行DDL和没有绑定参数的DML语句。
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-91fcb596-c4d1-42d3-8b63-bf40d87fe87c.png)  

*   **copy**：将源表中的数据复制到目标表，这个存储过程有助于帮助大家理解和使用DBMS\_SQL。
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-6d1cadb1-e747-4898-afcb-9444e9d1b136.png)  

有点长...  

![](https://oss-emcsprod-public.modb.pro/image/editor/20240106-b17c2808-eb91-4e83-9175-adc361652aac.png)  

至此，对于DBMS\_SQL的相关简单演示就到此结束了。若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。本文所有测试案例在文末供大家在ORACLE数据库上学习使用。  

  

**四、相关网站链接**

> [https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS\_SQL.html#GUID-E9BAA1FD-DBAC-453F-8674-162B10133505](https://docs.oracle.com/en/database/oracle/oracle-database/19/arpls/DBMS_SQL.html#GUID-E9BAA1FD-DBAC-453F-8674-162B10133505)  

**五、测试SQL语句**

本文所用SQL语句如下：

    -- 单行插入
    CREATE TABLE tab(c1 NUMBER, c2 NUMBER);  -- 依据ORACLE测试案例 创建tab表
    
    CREATE OR REPLACE PROCEDURE single_Row_insert
         (c1 NUMBER, c2 NUMBER, r OUT NUMBER) is
    c NUMBER;
    n NUMBER;
    begin
      c := DBMS_SQL.OPEN_CURSOR;
      DBMS_SQL.PARSE(c, 'INSERT INTO tab VALUES (:bnd1, :bnd2) ' ||
                        'RETURNING c1*c2 INTO :bnd3', DBMS_SQL.NATIVE);
    DBMS_SQL.BIND_VARIABLE(c, 'bnd1', c1);
      DBMS_SQL.BIND_VARIABLE(c, 'bnd2', c2);
      DBMS_SQL.BIND_VARIABLE(c, 'bnd3', r);
      n := DBMS_SQL.EXECUTE(c); 
      DBMS_SQL.VARIABLE_VALUE(c, 'bnd3', r); -- get value of outbind variable
      DBMS_SQL.CLOSE_CURSOR(c);
    END;
    /
    
    declare
    c3 number(7,2);
    begin
    single_Row_insert(5, 10, c3);
    dbms_output.put_line('c3 = '|| c3);
    end;
    /
    
    select * from tab;
    
    -- 单行更新
    CREATE OR REPLACE PROCEDURE single_Row_update
           (c1 NUMBER, c2 NUMBER, r out NUMBER) IS
      c NUMBER;
      n NUMBER;
      BEGIN
        c := DBMS_SQL.OPEN_CURSOR;
        DBMS_SQL.PARSE(c, 'UPDATE tab SET c1 = :bnd1, c2 = :bnd2 ' ||
                          'WHERE rownum < 2' || 
                          'RETURNING c1*c2 INTO :bnd3', DBMS_SQL.NATIVE);
        DBMS_SQL.BIND_VARIABLE(c, 'bnd1', c1);
        DBMS_SQL.BIND_VARIABLE(c, 'bnd2', c2);
        DBMS_SQL.BIND_VARIABLE(c, 'bnd3', r);
        n := DBMS_SQL.EXECUTE(c); 
        DBMS_SQL.VARIABLE_VALUE(c, 'bnd3', r);-- get value of outbind variable
        DBMS_SQL.CLOSE_CURSOR(c);
      END;
    /
    
    -- 调用single_Row_update
    declare
    c3 number(7,2);
    begin
    single_Row_update(5, 5, c3);  
    dbms_output.put_line('c3 = '|| c3);
    end;
    /
    -- 查看数据是否更新成功
    select * from tab;
    
    
    
    -- 单行删除
    CREATE OR REPLACE PROCEDURE single_Row_Delete
         (c1 NUMBER, r OUT NUMBER) is
    c NUMBER;
    n number;
    BEGIN
      c := DBMS_SQL.OPEN_CURSOR;
      DBMS_SQL.PARSE(c, 'DELETE FROM tab WHERE ROWNUM = :bnd1 ' ||
                    'RETURNING c1*c2 INTO :bnd2', DBMS_SQL.NATIVE);
      DBMS_SQL.BIND_VARIABLE(c, 'bnd1', c1);
      DBMS_SQL.BIND_VARIABLE(c, 'bnd2', r);
      n := DBMS_SQL.EXECUTE(c); 
      DBMS_SQL.VARIABLE_VALUE(c, 'bnd2', r);-- get value of outbind variable
      DBMS_SQL.CLOSE_CURSOR(c);
    END;
    /
     
    -- 调用single_Row_Delete
    declare
    c3 number(7,2);
    begin
    single_Row_Delete(1, c3);  -- ROWNUM = 1 删除一行数据
    dbms_output.put_line('c3 = '|| c3);
    end;
    /
    -- 查看删除结果
    select * from tab;

    -- 插入多行
    CREATE OR REPLACE PROCEDURE multi_Row_insert
         (c1 DBMS_SQL.NUMBER_TABLE, c2 DBMS_SQL.NUMBER_TABLE, 
          r OUT DBMS_SQL.NUMBER_TABLE) is
    c NUMBER;
    n NUMBER;
    BEGIN
      c := DBMS_SQL.OPEN_CURSOR;
      DBMS_SQL.PARSE(c, 'insert into tab VALUES (:bnd1, :bnd2) ' ||
                        'RETURNING c1*c2 INTO :bnd3', DBMS_SQL.NATIVE);
      DBMS_SQL.BIND_ARRAY(c, 'bnd1', c1);
      DBMS_SQL.BIND_ARRAY(c, 'bnd2', c2);
      DBMS_SQL.BIND_ARRAY(c, 'bnd3', r);
      n := DBMS_SQL.EXECUTE(c); 
      DBMS_SQL.VARIABLE_VALUE(c, 'bnd3', r);-- get value of outbind variable
      DBMS_SQL.CLOSE_CURSOR(c);
    END;
    /
    
    
    -- 调用multi_Row_insert
    DECLARE  
      my_number_table DBMS_SQL.NUMBER_TABLE;  
      my_number_table2 DBMS_SQL.NUMBER_TABLE;  
      my_number_table3 DBMS_SQL.NUMBER_TABLE;  
    BEGIN  
      my_number_table(1) := 10;  
      my_number_table(2) := 20;  
      my_number_table(3) := 30;  
    
      my_number_table2(1) := 11;  
      my_number_table2(2) := 21;  
      my_number_table2(3) := 31;  
      multi_Row_insert(my_number_table,my_number_table2,my_number_table3);
    
      -- 查看my_number_table3数据
      for i in 1 .. my_number_table3.count loop 
      dbms_output.put_line('c1*c2 = ' || my_number_table3(i));
      end loop;
    END;
    /
    
    -- 查看数据是否插入成功
    select * from tab;
    
    -- 再次调用multi_Row_insert 为multi_Row_update做准备
    DECLARE  
      my_number_table DBMS_SQL.NUMBER_TABLE;  
      my_number_table2 DBMS_SQL.NUMBER_TABLE;  
      my_number_table3 DBMS_SQL.NUMBER_TABLE;  
    BEGIN  
      my_number_table(1) := 10;  
      my_number_table(2) := 20;  
      my_number_table(3) := 30;  
    
      my_number_table2(1) := 11;  
      my_number_table2(2) := 21;  
      my_number_table2(3) := 31;  
      multi_Row_insert(my_number_table,my_number_table2,my_number_table3);
    
      -- 查看my_number_table3数据
      for i in 1 .. my_number_table3.count loop 
      dbms_output.put_line('c1*c2 = ' || my_number_table3(i));
      end loop;
    END;
    /
    
    -- 查看数据 
    select * from tab;
    
    
    
    -- 多行更新
    CREATE OR REPLACE PROCEDURE multi_Row_update
         (c1 NUMBER, c2 NUMBER, r OUT DBMS_SQL.NUMBER_TABLE) IS
    c NUMBER;
    n NUMBER;
    BEGIN
      c := DBMS_SQL.OPEN_CURSOR;
      DBMS_SQL.PARSE(c, 'UPDATE tab SET c1 = :bnd1 WHERE c2 = :bnd2 ' ||
                        'RETURNING c1*c2 INTO :bnd3', DBMS_SQL.NATIVE);
      DBMS_SQL.BIND_VARIABLE(c, 'bnd1', c1);
      DBMS_SQL.BIND_VARIABLE(c, 'bnd2', c2);
      DBMS_SQL.BIND_ARRAY(c, 'bnd3', r);
      n := DBMS_SQL.EXECUTE(c); 
      DBMS_SQL.VARIABLE_VALUE(c, 'bnd3', r);-- get value of outbind variable
      DBMS_SQL.CLOSE_CURSOR(c);
    END;
    /
    
    -- 调用multi_Row_update
    DECLARE   
        my_number_table3 DBMS_SQL.NUMBER_TABLE;  
    BEGIN  
      multi_Row_update(100, 11, my_number_table3);
    
      for i in 1 .. my_number_table3.count loop 
      dbms_output.put_line(' cur value := ' || my_number_table3(i));
      end loop;
    END;
    /
    
    -- 查看是否更新成功
    select * from tab;
    
    -- 多行删除
    CREATE OR REPLACE PROCEDURE multi_row_delete
         (c1 DBMS_SQL.NUMBER_TABLE,
          r OUT DBMS_SQL.NUMBER_TABLE) is
    c NUMBER;
    n NUMBER;
    BEGIN
      c := DBMS_SQL.OPEN_CURSOR;
      DBMS_SQL.PARSE(c, 'DELETE FROM tab WHERE c1 = :bnd1 ' ||
                        'RETURNING c1*c2 INTO :bnd2', DBMS_SQL.NATIVE);
      DBMS_SQL.BIND_ARRAY(c, 'bnd1', c1);
      DBMS_SQL.BIND_ARRAY(c, 'bnd2', r);
      n := DBMS_SQL.EXECUTE(c); 
      DBMS_SQL.VARIABLE_VALUE(c, 'bnd2', r);-- get value of outbind variable
      DBMS_SQL.CLOSE_CURSOR(c);
    END;
    /
    
    -- 调用multi_Row_delete
    DECLARE                                  
      my_number_table DBMS_SQL.NUMBER_TABLE;   
        my_number_table3 DBMS_SQL.NUMBER_TABLE;  
    BEGIN                               
      my_number_table(1) := 100;  
      my_number_table(2) := 20;  
      my_number_table(3) := 30;  
                                     
      multi_Row_delete(my_number_table, my_number_table3);
       
      for i in 1 .. my_number_table3.count loop 
      dbms_output.put_line('c1*c2 = ' || my_number_table3(i));
      end loop;
    END;
    /
    
    -- 查看是否删除成功
    select * from tab;

    -- exec 
     CREATE OR REPLACE PROCEDURE exec(STRING IN varchar2) AS
        cursor_name NUMBER;
        ret INTEGER;
    BEGIN
       cursor_name := DBMS_SQL.OPEN_CURSOR;
       DBMS_SQL.PARSE(cursor_name, string, DBMS_SQL.NATIVE);
       ret := DBMS_SQL.EXECUTE(cursor_name);
       DBMS_SQL.CLOSE_CURSOR(cursor_name);
    END;
    /
    
    call exec('create table acct(id NUMBER, name VARCHAR2(30), birthdate DATE)');
    -- 2024年了...
    insert into acct values(1, 'acct1', TO_DATE('2024-01-01', 'YYYY-MM-DD'));
    insert into acct values(2, 'acct2', TO_DATE('2024-01-02', 'YYYY-MM-DD'));
    insert into acct values(3, 'acct3', TO_DATE('2024-01-03', 'YYYY-MM-DD'));
    insert into acct values(4, 'acct4', TO_DATE('2024-01-04', 'YYYY-MM-DD'));
    
    -- 设置date输出显示格式
    alter session set nls_date_format='DD-MON-YY';
    select * from acct;
    
    -- copy 
    -- 准备目标表
    create table acct_test(id NUMBER, name VARCHAR2(30), birthdate DATE);
    
    CREATE OR REPLACE PROCEDURE copy ( 
         source      IN VARCHAR2, 
         destination IN VARCHAR2) IS 
         id_var             NUMBER; 
         name_var           VARCHAR2(30); 
         birthdate_var      DATE; 
         source_cursor      INTEGER; 
         destination_cursor INTEGER; 
         ignore             INTEGER; 
      BEGIN 
     
      -- Prepare a cursor to select from the source table: 
         source_cursor := dbms_sql.open_cursor; 
         DBMS_SQL.PARSE(source_cursor, 
             'SELECT id, name, birthdate FROM ' || source, 
              DBMS_SQL.NATIVE); 
         DBMS_SQL.DEFINE_COLUMN(source_cursor, 1, id_var); 
         DBMS_SQL.DEFINE_COLUMN(source_cursor, 2, name_var, 30); 
         DBMS_SQL.DEFINE_COLUMN(source_cursor, 3, birthdate_var); 
         ignore := DBMS_SQL.EXECUTE(source_cursor); 
     
      -- Prepare a cursor to insert into the destination table: 
         destination_cursor := DBMS_SQL.OPEN_CURSOR; 
         DBMS_SQL.PARSE(destination_cursor, 
                      'INSERT INTO ' || destination || 
                      ' VALUES (:id_bind, :name_bind, :birthdate_bind)', 
                       DBMS_SQL.NATIVE); 
     
      -- Fetch a row from the source table and insert it into the destination table: 
         LOOP 
           IF DBMS_SQL.FETCH_ROWS(source_cursor)>0 THEN 
             -- get column values of the row 
             DBMS_SQL.COLUMN_VALUE(source_cursor, 1, id_var); 
             DBMS_SQL.COLUMN_VALUE(source_cursor, 2, name_var); 
             DBMS_SQL.COLUMN_VALUE(source_cursor, 3, birthdate_var); 
     
      -- Bind the row into the cursor that inserts into the destination table. You 
      -- could alter this example to require the use of dynamic SQL by inserting an 
      -- if condition before the bind. 
            DBMS_SQL.BIND_VARIABLE(destination_cursor, ':id_bind', id_var); 
            DBMS_SQL.BIND_VARIABLE(destination_cursor, ':name_bind', name_var); 
            DBMS_SQL.BIND_VARIABLE(destination_cursor, ':birthdate_bind', 
                                                                       birthdate_var); 
            ignore := DBMS_SQL.EXECUTE(destination_cursor); 
          ELSE 
     
      -- No more rows to copy: 
            EXIT; 
          END IF; 
        END LOOP; 
     
      -- Commit and close all cursors: 
         COMMIT; 
         DBMS_SQL.CLOSE_CURSOR(source_cursor); 
         DBMS_SQL.CLOSE_CURSOR(destination_cursor); 
       EXCEPTION 
         WHEN OTHERS THEN 
           IF DBMS_SQL.IS_OPEN(source_cursor) THEN 
             DBMS_SQL.CLOSE_CURSOR(source_cursor); 
           END IF; 
           IF DBMS_SQL.IS_OPEN(destination_cursor) THEN 
             DBMS_SQL.CLOSE_CURSOR(destination_cursor); 
           END IF; 
           RAISE; 
      END; 
    /  
    
    -- 运行copy
    exec copy('acct','acct_test');
    select * from acct_test;