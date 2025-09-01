一、RAW类型回顾
=========

在Oracle数据库中，RAW类型是一种可变长数据类型，一般用于存储二进制数据。在使用RAW类型时，必须指定具体长度，语法如下：

    RAW(size)
    

在SQL场景用作表中的字段时，最大可存储2000字节数据（当且仅当Oracle参数`MAX_STRING_SIZE` `=` `STANDARD时`），而用作PL/SQL场景的变量时，最大可存储32767字节数据。

    SQL> show parameter MAX_STRING_SIZE
    
    NAME                                 TYPE                              VALUE
    ------------------------------------ --------------------------------- ------------------------------
    max_string_size                      string                            STANDARD
    SQL> -- 未指定raw数据类型长度 报错
    SQL> CREATE TABLE test_raw(a raw);
    CREATE TABLE test_raw(a raw)
                               *
    ERROR at line 1:
    ORA-00906: missing left parenthesis
    
    
    SQL> -- SQL场景2000字节大小 成功创建
    SQL> CREATE TABLE test_raw(a raw(2000));
    
    Table created.
    
    SQL> -- SQL场景超过2000字节 报错
    SQL> CREATE TABLE test_raw2(a raw(2001));
    CREATE TABLE test_raw2(a raw(2001))
                                 *
    ERROR at line 1:
    ORA-00910: specified length too long for its datatype
    
    
    SQL> -- PL/SQL场景 定义变量32767字节大小
    SQL> DECLARE
      2  va raw(2001);
      3  BEGIN
      4  va := 'ff';
      5  END;
      6  /
    
    PL/SQL procedure successfully completed.
    
    SQL> -- PL/SQL场景 定义变量超过32767字节大小 报错
    SQL> DECLARE
      2  va raw(32768);
      3  BEGIN
      4  va := 'ff';
      5  END;
      6  /
    va raw(32768);
           *
    ERROR at line 2:
    ORA-06550: line 2, column 8:
    PLS-00215: String length constraints must be in range (1 .. 32767)
    

raw类型接收十六进制数据（0~9 和 A~F 或 a~f），如直接输入的数据不符合规范将会报错，比如说’gg’

    SQL> -- 接收十六进制数据
    SQL> DECLARE
      2  va raw(20);
      3  BEGIN
      4  va := '09afAF';
      5  END;
      6  /
    
    PL/SQL procedure successfully completed.
    
    SQL> -- 直接输入非十六进制数据将会报错
    SQL> DECLARE
      2  va raw(20);
      3  BEGIN
      4  va := 'gg';
      5  END;
      6  /
    DECLARE
    *
    ERROR at line 1:
    ORA-06502: PL/SQL: numeric or value error: hex to raw conversion error
    ORA-06512: at line 4
    

也可以考虑使用函数的方式传入传出数据，先以HEXTORAW、RAWTOHEX为例

HEXTORAW将十六进制字符串数据转换成数据，RAWTOHEX则相反，将raw类型数据转换成对应的十六进制字符串

    SQL> set serveroutput on
    SQL> DECLARE
      2  va raw(20) := HEXTORAW('ff');
      3  BEGIN
      4  DBMS_OUTPUT.PUT_LINE(('va: '||va));
      5  DBMS_OUTPUT.PUT_LINE(('RAWTOHEX(va): '||RAWTOHEX(va)));
      6  DBMS_OUTPUT.PUT_LINE('ff ~= HEXTORAW(ff) is '|| (case when 'ff' ~= va then 'true' else 'false' end));
      7  END;
      8  /
    va: FF
    RAWTOHEX(va): FF
    ff ~= HEXTORAW(ff) is true
    
    PL/SQL procedure successfully completed.
    

或者是使用UTL\_RAW这个系统包，比如说UTL\_RAW.CAST\_TO\_RAW、UTL\_RAW.CAST\_TO\_VARCHAR2

UTL\_RAW.CAST\_TO\_RAW，将输入的varchar2类型数据，比如说中文转换成对应的raw类型数据

UTL\_RAW.CAST\_TO\_VARCHAR2 将raw类型数据转换成对应的varchar2类型数据

    SQL> SELECT UTL_RAW.CAST_TO_RAW('ORACLE') FROM dual;
    
    UTL_RAW.CAST_TO_RAW('ORACLE')
    --------------------------------------------------------------------------------
    4F5241434C45
    
    SQL> SELECT UTL_RAW.CAST_TO_VARCHAR2('4F5241434C45') FROM dual;
    
    UTL_RAW.CAST_TO_VARCHAR2('4F5241434C45')
    --------------------------------------------------------------------------------
    ORACLE
    
    SQL> SELECT UTL_RAW.CAST_TO_RAW('羲和(Halo)数据库') FROM dual;
    
    UTL_RAW.CAST_TO_RAW('羲和(HALO)数据库')
    --------------------------------------------------------------------------------
    E7BEB2E5928C2848616C6F29E695B0E68DAEE5BA93
    
    SQL> SELECT UTL_RAW.CAST_TO_VARCHAR2('E7BEB2E5928C2848616C6F29E695B0E68DAEE5BA93') FROM dual;
    
    UTL_RAW.CAST_TO_VARCHAR2('E7BEB2E5928C2848616C6F29E695B0E68DAEE5BA93')
    --------------------------------------------------------------------------------
    羲和(Halo)数据库
    

二、羲和（Halo）数据库表现
===============

![](https://oss-emcsprod-public.modb.pro/image/editor/20240628-a2fac238-8075-4690-a95f-36472bf289fb.png)

raw类型处理十六进制数据

![](https://oss-emcsprod-public.modb.pro/image/editor/20240628-764b5231-8a3e-4c7a-859b-4430a4ad739b.png)

HEXTORAW、RAWTOHEX函数

![](https://oss-emcsprod-public.modb.pro/image/editor/20240628-b5558950-8d22-40b0-b3fe-e13314e644f9.png)

UTL\_RAW系统包

![](https://oss-emcsprod-public.modb.pro/image/editor/20240628-dde97c1a-36a4-4864-aba0-46baa3c73962.png)


三、声明
====

**若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。**

**文章转载请联系，谢谢合作。**