一、前言
----

前些天看到PostgreSQL分会在微信上的一则翻译分享，标题为Oracle 与 PostgreSQL 中的 IN、OUT 和 INOUT 参数。文章大概内容为，在针对Oracle迁移至PostgreSQL中，对于函数/存储过程的输入输出类型这一细节方面，双方处理的差异点，通过了解这些差异点，来提高迁移的成功率。

恰好当处在实现这一块的时候，我也简单琢磨过一会，所以此处决定结合该文章，顺带拉上羲和数据库（以下简称Halo），结合Oracle、PostgreSQL数据库来做一次简单的分享，通过多则示例，加深一下大家对于这一块的理解。相关文章链接如下：

[Oracle 与 PostgreSQL 中的 IN、OUT 和 INOUT 参数](https://mp.weixin.qq.com/s/SFlvmPbWBoL4d-R9_QXCMQ)

[IN, OUT and INOUT parameters in Oracle vs PostgreSQL](https://hexacluster.ai/postgresql/oracle-vs-postgresql-pass-by-value-and-pass-by-reference-for-in-out-inout-parameters/)

接下来就按照人家文章的节奏一步一步来吧。

* * *

二、按值传递和按引用传递？
-------------

在Oracle中，它的IN参数模式为引用传递，OUT以及IN OUT参数模式在默认的情况下，为值传递。更多详细内容如下图：

> 当 OUT 或 IN OUT 参数在过程中更改时，它仅更改参数值的副本。仅当过程无错误地完成后，结果值才会复制回形式参数。
> 
> 如果将集合作为 OUT 或 IN OUT 参数发送，它将按值传递。这意味着在进入过程时，完整的集合将从形式参数复制到实际参数，并在退出过程时复制回形式参数。如果集合很大，这可能会消耗大量的 CPU 和内存。NOCOPY 参数模式提示通过指示运行时引擎尝试通过引用而不是通过值传递 OUT 或 IN OUT 参数来解决此问题。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240319-9eced66b-fb42-408c-b77f-54094b9b581c.png)

而且在Oracle中，当参数使用了IN模式的话，不允许对该参数进行赋值操作。但是这方面没啥影响。

    CREATE FUNCTION func_test (va IN INTEGER) 
    RETURN INTEGER
    AS
    BEGIN
       va := 1;    -- oracle不允许修改
       RETURN 2;
    END ;
    /
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20240319-48a9b83f-7beb-4b5e-bf21-fe567a9ab7a2.png)

在PostgreSQL和Halo数据库中，这三种参数输入模式均为值传递。而当参数使用IN模式，允许对该参数进行赋值操作，由于参数的生命周期的原因，并不会对原有的输入产生任何动作。

PostgreSQL创建相应函数：

    CREATE  FUNCTION func_test (va IN INTEGER) 
    RETURNS INTEGER
    AS $$
    DECLARE
    BEGIN
       va := 1;
       RETURN 2;
    END $$ LANGUAGE plpgsql;
    

执行以下DO语句，它调用了上述的func\_test，传递了一个名为val\_a的变量，初始值为0，观察执行后的打印结果

    DO $$ 
    DECLARE
        val_a INT := 0;
        result_value INT := 0;
    BEGIN
        result_value = func_test(val_a);
        RAISE NOTICE 'The result is: % val_a %', result_value, val_a;
    END $$;
    

输出结果：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240319-40a29dda-aeb7-4950-8b5a-1d78059191c1.png)

可以看到的是输入变量的值并没有并修改掉。Halo数据库在这方面没有进行变动，此处就不再演示了。

* * *

三、将Oracle存储过程迁移到PostgreSQL，若存储过程中存在OUT/INOUT参数是否需要大幅进行变动？
---------------------------------------------------------

在**Oracle**执行如下语句，为创建一个带IN、OUT参数的存储过程，并通过匿名块调用执行，**INOUT其实和OUT差不太多 后续我就不在描述INOUT了 直接使用OUT进行讲解**

    CREATE OR REPLACE PROCEDURE add_numbers(
        a IN  INT,
        b IN INT,
        result OUT INT
    )
    AS 
    BEGIN
    	result := a + b;
    END;
    /
    
    -- 调用存储过程add_numbers
    DECLARE
        va INT := 100;
        vb INT := 200;
        result_value INT := 0;
    BEGIN
         add_numbers(va, vb, result_value);
         DBMS_OUTPUT.PUT_LINE('va is:  ' || va || ' vb is:  ' || vb || ' result is:  ' ||  result_value);
    END;
    /
    

最终输出结果：

    va is:  100 vb is:  200 result is:  300
    

如果使用**PostgreSQL**替换的话，则需要改写成以下语句，并进行执行。

    CREATE OR REPLACE PROCEDURE add_numbers(
       IN a   INT,
       IN b  INT,
       OUT result INT
    )
    AS $$
    DECLARE
    BEGIN
    	result = a + b;
    END $$ LANGUAGE plpgsql;
    
    -- 调用存储过程add_numbers
    DO $$
    DECLARE
        va INT = 100;
        vb INT = 200;
        result_value INT = 0;
    BEGIN
         CALL add_numbers(va, vb, result_value);
        RAISE NOTICE 'va is: % vb is： %  result is： % ', va,  vb,  result_value;
    END $$  LANGUAGE plpgsql;
    

最终输出结果：

    NOTICE:  va is: 100 vb is： 200  result is： 300 
    DO
    

可以看到的是，**如果是将Oracle数据中的带OUT参数的存储过程迁移至PostgreSQL中，在这种场景下，改写后的PostgreSQL的存储过程和原有的Oracle中的存储过程还是较为相似的，其迁移改写的成本还是能够接受的。**

**那如果是带有OUT参数的函数呢？**

* * *

四、将Oracle函数迁移到PostgreSQL，若函数中存在OUT/INOUT参数是否需要大幅进行变动？
-----------------------------------------------------

### **4.1、Oracle的带OUT参数的函数简单示例**

此处就直接使用以下人家文章中的示例吧

    CREATE OR REPLACE FUNCTION test_ro(x number, y OUT number) 
    RETURN boolean IS 
    BEGIN              
        y := x; 
        RETURN true; 
    END;                        
    /
    
    DECLARE               
        vx number := 100;
        vy number := 0;
        ret boolean := false;
    BEGIN
         ret := test_ro(vx, vy);
         IF ret THEN
            DBMS_OUTPUT.PUT_LINE('ret is true  vy is: ' || vy);
         ELSE
            DBMS_OUTPUT.PUT_LINE('ret is false  vy is: ' || vy);
        END IF;
    END;
    /
    

最终输出结果：

    ret is true  vy is: 100
    

### **4.2、PostgreSQL中函数如何使用OUT/INOUT参数**

我们先来尝试一下直接将上述的函数改成PostgreSQL的写法，看看能否正常进行 改写之后大概长这样

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-846556eb-5533-492d-85eb-932017559986.png)

可以看到的是 这样子简单的进行改写是不行的 在PostgreSQL中在函数入参使用OUT这方面和Oracle相比还是大有不同 接下来我来用一些示例来详细描述一下PostgreSQL这方面的使用

在Oracle中函数的OUT参数和RETUN构成两个输出，而在PostgreSQL中则不然。在PostgreSQL中则分为好几种情形，想要正确输出结果 ，必须要注意这几个点。

#### 4.2.1、PostgreSQL函数单个OUT参数

就像上述的示例一样，如果是单个OUT参数的函数，想要创建成功 并且成功调用的话 那么RETURNS语句后面接的数据类型 必须和该单个OUT参数的数据类型保持一致 否则会报错

**RETURNS语句后面接的数据类型与OUT参数类型不一致 报错如下**

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-846556eb-5533-492d-85eb-932017559986.png)

正确的创建姿势 当**RETURNS后接的数据类型和OUT参数一致时，由于PostgreSQL认为此种写法只存在一个返回值 所以逻辑块中的return语句，只能选择不写或者是return；**

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-482d795a-fc9d-44e2-9d5d-4d902885c35c.png)

若此时存在return语句 且return语句不是return；则会报错

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-046f832f-764f-4dd9-aaa3-1a0b5b3e0e2b.png)

此时取调用该函数的话 仅返回一个值 且不允许在OUT参数的位置使用变量，由于内部函数的识别机制，这样子将会无法找到实际该要执行函数。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-b3e12394-3b75-40b4-a8e0-c62c95c48748.png)

#### 4.2.2、PostgreSQL函数多个OUT参数

在这种情形下，RETURNS后接的数据类型必须是record类型，否则会报错

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-1f286797-6e3d-4c6e-89a1-d5c8f150b7db.png)

正确的创建 **和单个OUT参数类似 此处只能不写return或者是写上return;**

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-b674cd98-097d-4afc-b5c5-71dbea9911f8.png)

否则将会不允许创建

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-b46dcac7-fea9-4ee5-a0f4-67620c812a54.png)

此时的话该如何进行调用呢？可以考虑以下两种方式，一种是定义一个record变量接收返回数据 第二种是使用select into语句进行操作

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-73ad4559-fb7b-4f2a-bb5c-2c4d330c7aa7.png)

#### 4.2.3、迁移带out参数的函数解决方案

回到开头的Oracle的函数，那么该如何改写才能原本Oracle的功能保持一致呢？其实解决方案在上面已经给出来了。

改写创建语句

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-92c3c673-ded7-4b95-ae13-8d70a48f8fd8.png)

改写调用语句

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-73094d1d-8c61-4f27-87aa-fbabd37cbe57.png)

**可以很明显的看出 如果是将Oracle中带OUT参数的函数迁移到PostgreSQL中，改写的成本还是非常非常大的。此处也是给出了迁移改写的解决的方案，供大家参考。**

那麽是否是所有的函数/存储过程均能通过上述的改写来达成目的呢？接下来我们看看Oracle包中的关于此处内容的测试案例。

五、将Oracle的Package中的函数/存储过程迁移至PostgreSQL中
----------------------------------------

关于上面的那个问题，此处给出一个示例 当然此处的示例还进行了简化 实际上来说其实一般的包会更为复杂 你甚至能看到各种奇奇怪怪的写法

这个Package内部存在两个名为pro\_test的存储过程 一个不存在任何参数 另一个带有一个OUT参数 ，同时还存在两个名为func\_test，这两函数 除OUT修饰的参数类型 一个为int 另一个为varchar2外 其余均一模一样

    CREATE OR REPLACE PACKAGE pkg_test AS 
        -- 同名存储过程
        PROCEDURE pro_test;
        PROCEDURE pro_test(va out varchar2);
    
        -- 同名同参函数 仅输出类型不一致
        FUNCTION func_test(va int, vb int, vc out int) RETURN int;
        FUNCTION func_test(va int, vb int, vc out varchar2) RETURN int;
    END;
    /
    
    CREATE OR REPLACE PACKAGE BODY pkg_test AS 
      -- 无参存储过程
      PROCEDURE pro_test
        AS 
      BEGIN  
        DBMS_OUTPUT.PUT_LINE('pro_test'); 
      END;
      
      -- 带out参数存储过程
      PROCEDURE pro_test(va out varchar2)
        AS 
      BEGIN  
        DBMS_OUTPUT.PUT_LINE('pro_test(out varchar2)'); 
      END;
    
      -- out参数类型为int
      FUNCTION func_test(va int, vb int, vc out int) RETURN int
        AS 
      BEGIN  
        DBMS_OUTPUT.PUT_LINE('func_test(out int)');
        vc := va + vb;
        RETURN 25;
      END; 
    
     -- out参数类型为varchar2
     FUNCTION func_test(va int, vb int, vc out varchar2) RETURN int
      AS 
      BEGIN  
        DBMS_OUTPUT.PUT_LINE('func_test(out varchar2)');
        vc := 'func_test';
        RETURN va + vb;
      END; 
    END;
    /
    

执行以下匿名块

    DECLARE 
      p_va varchar2(20);
      f_va int;
      f_vb varchar2(20);
      f_ret int;
    BEGIN 
      -- procedure
      pkg_test.pro_test;
      pkg_test.pro_test(p_va);
    
      -- function
      f_ret := pkg_test.func_test(25, 25, f_va);
      f_ret := pkg_test.func_test(25, 25, f_vb);
    END;
    / 
    

Oracle正常创建外加正常调用 输出最终结果

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-2a666eaf-b1de-42a7-98c3-908d84d3afc1.png)

接下来我们尝试按照上面的来进行改写 尝试将该Package迁移至PostgreSQL 看看能否成功

**相关改写内容如下**

    -- 对于Oracle中的Package 我们可以选择创建一个同名的schema来现应付一下
    CREATE SCHEMA pkg_test;
    
    -- 尝试创建无参存储过程
    CREATE OR REPLACE PROCEDURE pkg_test.pro_test()
      AS $$
    BEGIN  
      RAISE NOTICE 'pro_test';
    END $$ LANGUAGE plpgsql;
    
    -- 尝试创建带OUT参数的同名存储过程  
    -- 此处由于PostgreSQL内核对于同名存储过程的判定
    -- 认为带out参数的存储过程 和 不带任何参数的存储过程是同一个
    -- 想要执行的是replace操作 而不是create操作 
    -- 最终报错
    CREATE OR REPLACE PROCEDURE pkg_test.pro_test(va out varchar)
      AS $$
    BEGIN  
      RAISE NOTICE 'pro_test(out varchar2)';
    END $$ LANGUAGE plpgsql;
    
    -- 接下来尝试迁移函数
    -- out参数类型为int
    CREATE OR REPLACE FUNCTION pkg_test.func_test(va int, vb int, vc out int, ret out int) 
    RETURNS record
      AS $$
    DECLARE
    BEGIN  
      RAISE NOTICE 'func_test(out int)';
      vc := va + vb;
      ret := 25;
    END $$ LANGUAGE plpgsql;
    
    -- out参数类型为varchar
    -- 此处和上述创建同名存储存储一样 最终会创建失败
    CREATE OR REPLACE FUNCTION pkg_test.func_test(va int, vb int, vc out varchar, ret out int) 
    RETURNS record
      AS $$
    DECLARE
    BEGIN  
      RAISE NOTICE 'func_test(out int)';
      vc := 'func_test';
      ret := va + vb;
    END $$ LANGUAGE plpgsql;
    

执行后输出：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-179ebe29-9ac3-4cd7-ae5d-8b67f2fb49a2.png)

**由于PostgreSQL内部对于同名函数和同名存储过程的判定，所以像这种通过改写就无法完成迁移。只能是重新修改逻辑来避免了。**

> **因此如果想要将Oracle成功迁移至原生PostgreSQL中，就单纯在IN、OUT 和 INOUT参数模式这一小细节上，**
> 
> **首先你需要先整理整理是否在Oracle的Package中使用了文中所说的类似的写法，**
> 
> **如果有使用的话，在不变更原有逻辑的情形之下，改写相关SQL是无法帮助你迁移成功的。**
> 
> **如果没有使用的话，就能按照文中给出的改写的解决方案，通过消耗时间、精力、投入相关资源来将这一块改写成PostgreSQL能够支持处理的逻辑。**

**六、羲和（Halo）数据库在**IN、OUT 和 INOUT的支持力度
-------------------------------------

> **接下来的话，就和由Oracle迁移至PostgreSQL这个话题没有关系了，相关的改写方案和技术上面也说的差不多了，**
> 
> **如果可以的话，或许可以考虑考虑由Oracle迁移至Halo数据库，也许能减轻不少关于迁移Oracle的压力。**

### **6.1、将Oracle存储过程迁移到Halo，若存储过程中存在OUT/INOUT参数是否需要大幅进行变动？**

以第三节的示例 在Halo数据库Oracle模式进行测试 结果如图：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-4b60b5ae-3161-4720-ab9c-5fb8155cad6a.png)

### **6.2、将Oracle函数迁移到Halo，若函数中存在OUT/INOUT参数是否需要大幅进行变动？**

以第四节的示例 在Halo数据库Oracle模式进行测试 结果如图：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-8ca4b540-e948-4fe1-aa37-82324e3114ed.png)

### **6.3、将Oracle的Package中的函数/存储过程迁移至Halo中，是否需要大幅进行变动？**

以第五节的示例 在Halo数据库Oracle模式进行测试 结果如图：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240321-e6673fa1-7f42-449f-99bd-d6f1cfa9eaf9.png)

可以看到的是在没有进行修改PL/SQL代码的情况下，原有的Oracle的PL/SQL代码在Halo数据库上依旧能够得到支持。

使用Halo数据库，可以做到不需要花费大量的时间、精力和资源用于改写、改造原有的PL/SQL代码，一定程度上的应用，在不修改原有应用代码的前提上，甚至能做到无感知的迁移替换。

七、声明
----

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

如果您的Halo数据库未出现与文中的截图一致的现象，可能是由于版本问题，麻烦您联系我升级一下数据库版本。

亦或者是在测试中发现了别的问题，欢迎大家向我提供问题信息，我将尽快在新的版本中处理和解决，完善相关功能，争取给大家带来更好的使用体验。

文章转载请联系，谢谢合作。