今天的这个主题就很显而易见了，这个我觉得怪怪的想要进行分享的原因是：

**你甚至可以在Oracle数据库内部定义出带默认值的复合类型！！！主要其实也是我玩PG玩多了，没瞅见过自带默认值的复合类型。当然也有还是太年轻的缘故（笑）。**  

我们先来简单了解一下，在Oracle中如何创建复合类型。

  

一、Oracle如何创建复合类型
================

1.1、**CREATE TYPE type\_name  AS OBJECT  object\_type\_def**
------------------------------------------------------------

第一种就是比较常见的**CREATE TYPE type\_name  AS OBJECT  object\_type\_def;**

如下示例：

    CREATE TYPE demo_typ1 AS OBJECT (a1 NUMBER, a2 NUMBER);

运行结果：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-6b346c3f-a70b-456a-a71a-121ab332b47e.png)  

但是这种方式不允许携带默认值，我们可以尝试附加一下默认值，测试一下

    CREATE TYPE sample_type AS OBJECT 
    (
      field1 varchar2(20) := 'Hello',
      field2 varchar2(20) := 'World' 
    );

运行结果：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-cd122f1e-1cb5-4b56-8bad-dbe9135d9f09.png)  

那么那种方法支持创建带默认值的复合类型呢？

  

1.2、**TYPE record\_name IS RECORD field\_definition**
-----------------------------------------------------

**那就是第二种方式**，**TYPE record\_name IS RECORD field\_definition;** 

**语法图如下所示：**

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-e82289b7-08c6-4909-b770-2ee11dbe062f.png)  

从**field\_definition**中不难看出，它就是支持在定义的时候附加默认值，是可选项。

所以第一种方法能做到它也能，第一种方法做不到的它也能。

比如说定义复合类型并使用

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-4ff2c234-7231-4508-b698-1973ed3fc82e.png)  

比如说自定义带默认值的复合类型并使用

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-11b6de71-7151-49d8-b70b-cc065ce2f139.png)  

可能还有别的方式可以定义Oracle中的复合类型哈，各位感兴趣可以再研究研究。

至此，我们便看到了一个具有默认值的复合类型的存在，接下来我们在简单聊一聊像这种带默认值的复合类型的"生命周期"。

  

二、带默认值的复合类型的"生命周期"
==================

没错！像这种带默认值的复合类型其实是存在"生命周期"的。

而这个"生命周期"主要取决于是在哪种场景下使用。

如果是在函数/存储过程/匿名块中使用，那么此复合类型的生命周期大概就是整个函数体，出了函数体作用域就不再存在，就像一个临时的数据类型。

如果是在Package中使用，那么此复合类型便可以向系统类型一样，被广泛使用，而这个时候的生命周期就取决于Package何时被删除。

2.1、在Package中声明定义并使用
--------------------

CREATE PACKAGE 声明一个带默认值的复合类型、同时声明一个名为protest存储过程

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-a33a6f18-0392-45e6-ab83-1726c9bc5151.png)  

CREATE PACKAGE BODY实现protest存储过程，定义一个复合类型带默认值的变量，然后直接打印变量值

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-f35df356-aa4c-4637-8254-5edea397e076.png)  

调用该存储过程

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-e8e5b298-035d-42b6-8ecd-7978c81eb6fc.png)  

那么此Package定义的类型还能在别处被使用吗？

**是，可以的！**

比如说在匿名块中使用package定义的类型（函数存储过程同理）

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-7090d17b-388f-4f8d-a99f-813a1c46aa69.png)  

当且仅当Package被删除之后，该类型生命周期结束。

  

2.2、在函数/存储过程/匿名块中声明定义并使用
------------------------

这个其实在1.2小结就演示过了，此处用一下1.2小节的测试图片。该测试案例实在匿名块中所使用的，函数/存储过程同理。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-11b6de71-7151-49d8-b70b-cc065ce2f139.png)  

当执行结束之后，该类型的生命周期便结束了，之后就不能再被使用，除非重新定义。

虽然测试案例非常简单，但是指不定有Oracle的"老玩家"，把这个玩出花来，这也是我为什么选择最后输出"Hello World"的原因。

我不信哪位同学的第一版程序不是"Hello World"，如果不是，那就叉出去！

  

三、羲和（Halo）数据库在这方面的表现
====================

讲了这么多，该秀的时候还是要秀一下的，不秀的话，那我岂不是白做了（加个表情）。

3.1、在Package中被声明定义并使用
---------------------

如下图所示：

    ![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-749cc16f-0fe6-4d31-923b-39542dc520a9.png)  

比如说在匿名块中使用此package定义的类型（函数存储过程同理）

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-5ed574a9-a8da-4c76-a03e-d49bd2fc59b5.png)  

3.2、在函数/存储过程/匿名块中声明定义并使用
------------------------

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-4f309c3a-90c8-4a2b-a667-f565410152a8.png)  

**可以看到的是，羲和（Halo）数据库在这方面的表现和Oracle的完全吻合！**

  

五、声明
====

**本文为此系列第二篇，如想看到对Oracle功能的吐槽，敬请期待。**

**若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。**

**文章转载请联系，谢谢合作。**