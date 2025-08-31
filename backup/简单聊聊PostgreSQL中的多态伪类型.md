一、前言
====

也是好久没有写文章了，主要是前阵子比较忙，还有外加上喜欢骑车出去兜风，以至于一而再，再而三的咕咕。

那么接下来直接步入正题，让我们来瞅瞅PostgreSQL中的多态伪类型。

  

二、构建问题
======

在当前环境中存在一个名为display的函数，函数原型如下

    create or replace function display(val in boolean)
    returns text as $$
    begin
      raise notice '%', 'display(boolean)';
      return NULL;
    end; $$ language plpgsql;

函数体仅打印一下，它接受布尔类型的数据。当我们输入true或者false，显而易见的能正常工作。

但除此之外，我们输入别的，比如说integer数值，则会报错。

    postgres=# select display(true);
    NOTICE:  display(boolean)
     display 
    ---------
     
    (1 row)
    
    postgres=# select display(false);
    NOTICE:  display(boolean)
     display 
    ---------
     
    (1 row)
    
    postgres=# select display(1); -- error
    ERROR:  function display(integer) does not exist
    LINE 1: select display(1);
                   ^
    HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
    postgres=# select display(1::boolean);
    NOTICE:  display(boolean)
     display 
    ---------
     
    (1 row)

这其实也是很正常，符合预期的，这个时候，我们手动再去创建一个名为display，参数类型为integer就好了

    postgres=# create or replace function display(val in integer)
    postgres-# returns text as $$
    postgres$# begin
    postgres$#   raise notice '%', 'display(integer)';
    postgres$#   return NULL;
    postgres$# end; $$ language plpgsql;
    CREATE FUNCTION
    postgres=# \df display
                            List of functions
     Schema |  Name   | Result data type | Argument data types | Type 
    --------+---------+------------------+---------------------+------
     public | display | text             | val boolean         | func
     public | display | text             | val integer         | func
    (2 rows)
    
    postgres=# select display(true);
    NOTICE:  display(boolean)
     display 
    ---------
     
    (1 row)
    
    postgres=# select display(false);
    NOTICE:  display(boolean)
     display 
    ---------
     
    (1 row)
    
    postgres=# select display(1);
    NOTICE:  display(integer)
     display 
    ---------
     
    (1 row)

而要是再输入其他的类型，那么我们还需要继续上面的步骤，去创建对应类型的函数，输入的其他类型越多，创建的同名函数也就越多，在数据库中的元数据也就越多。

那有没有看起来更加简洁的方法呢？这边引出了今天的主角——多态伪类型。

  

三、simple家族多态伪类型
===============

如下图所示，便是在PostgreSQL中的simple家族的多个多态伪类型。

![](https://oss-emcsprod-public.modb.pro/image/editor/20250527-1927263036230873088_585460.png)

这些类型名字也很清楚，很容易见名知意，外加旁边的描述，可以较为自然的了解到底是在什么常见下使用的，这里就不过多解释了。

对于上面的问题而言，我们使用simple中的anyelement即可，要是输入的类型没有数组数据也可以直接使用anynoarray。

    postgres=# create or replace function display(val in anyelement)
    postgres-# returns text as $$
    postgres$# begin
    postgres$#   if pg_typeof(val) = 'integer'::regtype then
    postgres$#     raise notice '%', 'display(integer)';
    postgres$#   elsif pg_typeof(val) = 'boolean'::regtype then
    postgres$#     raise notice '%', 'display(boolean)';
    postgres$#   end if;
    postgres$#   return NULL;
    postgres$# end; $$ language plpgsql;
    CREATE FUNCTION
    postgres=# select display(1);
    NOTICE:  display(integer)
     display 
    ---------
     
    (1 row)
    
    postgres=# select display(true);
    NOTICE:  display(boolean)
     display 
    ---------
     
    (1 row)
    
    postgres=# \df display
                            List of functions
     Schema |  Name   | Result data type | Argument data types | Type 
    --------+---------+------------------+---------------------+------
     public | display | text             | val anyelement      | func
    (1 row)
    
    postgres=# 

我看到YugabyteDB有一个不错的示例，看起来更好理解，如下所示

    postgres=# -- Such a user-defined type is known as a "row" type.
    postgres=# create type rt as (b boolean, i int);
    CREATE TYPE
    postgres=# 
    postgres=# create function display(val in anyelement)
    postgres-#   returns text
    postgres-#   immutable
    postgres-#   language plpgsql
    postgres-# as $body$
    postgres$# declare
    postgres$#   val_type constant regtype := pg_typeof(val);
    postgres$# begin
    postgres$#   return
    postgres$#     case val_type
    postgres$#       when pg_typeof(null::boolean) then '"boolean" input'
    postgres$#       when pg_typeof(null::int)     then '"int" input'
    postgres$#       when pg_typeof(null::bigint)  then '"bigint" input'
    postgres$#       when pg_typeof(null::rt)      then '"rt" input'
    postgres$#       else                               'unsupported data type'
    postgres$#     end::text;
    postgres$# end;
    postgres$# $body$;
    CREATE FUNCTION
    postgres=# select display(true)
    postgres-# union all
    postgres-# select display(43)
    postgres-# union all
    postgres-# select display(567890123456789)
    postgres-# union all
    postgres-# select display((true, 42)::rt)
    postgres-# union all
    postgres-# select display(now()::timestamp);
            display        
    -----------------------
     "boolean" input
     "int" input
     "bigint" input
     "rt" input
     unsupported data type
    (5 rows)
    
    postgres=# \df display
                            List of functions
     Schema |  Name   | Result data type | Argument data types | Type 
    --------+---------+------------------+---------------------+------
     public | display | text             | val anyelement      | func
    (1 row)
    
    postgres=# 

对这个有了大概的了解之后，便可以将业务代码融入其中了。

  

四、simple家族多态伪类型的特点
==================

在上一章节我们简单的看到了关于simple家族多态伪类型的使用，它很好的解决了我们之前预设的问题。

在上面的示例中我们使用了anyelement，它的表述是能接受任意数据类型，当函数中仅有一个anyelement是没有问题的，

但是当函数中有着多个anyelement的时候，则会根据第一个实际的输入，推导出实际的数据类型，并且所有的anyelement只接受该类型的输入。

也就是所有声明成anyelement，都仅接受同一种数据类型数据。一个简单的示例如下：

    postgres=# create or replace function equal2(anyelement, anyelement)
    postgres-# returns text as $$
    postgres$# begin
    postgres$#   return NULL;
    postgres$# end; $$ language plpgsql;
    CREATE FUNCTION
    postgres=# select equal2(1,1);
     equal2 
    --------
     
    (1 row)
    
    postgres=# select equal2(1,1.0);
    ERROR:  function equal2(integer, numeric) does not exist
    LINE 1: select equal2(1,1.0);
                   ^
    HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
    postgres=# select equal2(1,1::bigint);
    ERROR:  function equal2(integer, bigint) does not exist
    LINE 1: select equal2(1,1::bigint);
                   ^
    HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
    postgres=# 

还有就是prepare需要指定具体类型

    [postgres@iZuf6hwo0wgeev4dvua4csZ ~]$ psql
    psql (18beta1)
    Type "help" for help.
    
    postgres=# \sf display
    CREATE OR REPLACE FUNCTION public.display(val anyelement)
     RETURNS text
     LANGUAGE plpgsql
     IMMUTABLE
    AS $function$
    declare
      val_type constant regtype := pg_typeof(val);
    begin
      return
        case val_type
          when pg_typeof(null::boolean) then '"boolean" input'
          when pg_typeof(null::int)     then '"int" input'
          when pg_typeof(null::bigint)  then '"bigint" input'
          when pg_typeof(null::rt)      then '"rt" input'
          else                               'unsupported data type'
        end::text;
    end;
    $function$
    postgres=# prepare p1 as select display($1); -- 不指定类型 报错
    ERROR:  could not determine polymorphic type because input has type unknown
    postgres=# prepare p1(int) as select display($1);
    PREPARE
    postgres=# execute p1(1);
       display   
    -------------
     "int" input
    (1 row)
    
    postgres=# 

  

五、common家族的多态伪类型
================

如下图所示，便是在PostgreSQL中的common家族的多个多态伪类型。

![](https://oss-emcsprod-public.modb.pro/image/editor/20250527-1927277582316220416_585460.png)

在simple家族发布几年之后，PostgreSQL推出了common家族的多态伪类型。与simple家族的最大区别在于，如果输入存在多个不同的类型，

common家族的多态伪类型会尝试着推导出这多个类型的可以转换的公共类型。比方说上面的示例我们简单坐下调整，结果如下：

    postgres=# create or replace function equal2(anycompatible, anycompatible)
    postgres-# returns text as $$
    postgres$# begin
    postgres$#   return NULL;
    postgres$# end; $$ language plpgsql;
    CREATE FUNCTION
    postgres=# select equal2(1,1);
     equal2 
    --------
     
    (1 row)
    
    postgres=# select equal2(1,1::bigint);
     equal2 
    --------
     
    (1 row)
    
    postgres=# select equal2(1,1.0);
     equal2 
    --------
     
    (1 row)
    
    postgres=# \df equal2
                                List of functions
     Schema |  Name  | Result data type |     Argument data types      | Type 
    --------+--------+------------------+------------------------------+------
     public | equal2 | text             | anycompatible, anycompatible | func
    (1 row)
    
    postgres=# 

可以看到的是能正常调用。对于prepare，不指定类型，照样能正常运行。

    [postgres@iZuf6hwo0wgeev4dvua4csZ ~]$ psql
    psql (18beta1)
    Type "help" for help.
    
    postgres=# prepare p1 as select equal2($1,$2);
    PREPARE
    postgres=# execute p1(1,1);
     equal2 
    --------
     
    (1 row)
    
    postgres=# execute p1(1,1.0);
     equal2 
    --------
     
    (1 row)
    
    postgres=# 

相较simple家族更加灵活和强大了。

  

六、"any"
=======

还有一个类型就是"any"，印象中是需要写c语言函数才能声明此类型，所以这里就不再展开了。

一个简单的示例便是pg\_typeof

    postgres=# \sf pg_typeof
    CREATE OR REPLACE FUNCTION pg_catalog.pg_typeof("any")
     RETURNS regtype
     LANGUAGE internal
     STABLE PARALLEL SAFE
    AS $function$pg_typeof$function$

  

七、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文章转载请联系，谢谢合作~

更多详细内容请阅读PostgreSQL官方文档~