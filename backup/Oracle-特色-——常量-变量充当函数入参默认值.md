诚如今天的主题所言："同学，你能接受常量/变量充当函数入参默认值吗？"

在PL/SQL中，函数入参默认值不就是输入参数类型后面的DEFAULT，就好比示例中的256，通常来说都是一个写定了的数据值。

在已提供默认值的基础上，当去调用该函数或存储过程时，可以不在输出对应参数。

    CREATE PROCEDURE test_proc(va INTEGER DEFAULT 256)
    AS 
    BEGIN
      DBMS_OUTPUT.PUT_LINE('va = '|| va);
    END;
    /

那常量/变量又是啥？如果对Oracle的Package比较了解的话，这块指的常量/变量，其实就是Package中的常量/变量。

难不成？

没错，正如你所想，在Oracle数据库中，它还真就支持你这样子去定义一个新的函数/存储过程。

所以我也是想吐槽:"还得是Oracle你这个浓眉大眼的家伙，这俩玩意也能给整到一起去！真正意义上的给我整活，咱还得老老实实的给它兼容了"。  

接下来让我们回到今天的主题，看看在Oracle数据中，常量/变量充当函数入参默认值到底是个什么样子的情况？

  

一、Oracle数据库之于常量/变量充当函数入参默认值
===========================

接下来让我们看看这二者在Oracle数据库中是如何结合在一起的。

**创建Package**

Package中存在一个名为v\_var的变量无默认数据、定义一个名为v\_var2的常量，默认值为9527，

以及一个名为test\_proc的声明，它的入参的默认值分别为Package中的常量和变量

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-8fa9a24f-bca7-4b72-a096-3a80ce55c848.png)  

**创建Package Body**  

将Package声明的存储过程进行实现，该存储过程将打印输入进来的参数数据值。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-a9b6a374-b532-4281-a211-ed838e6517ef.png)  

在调用该存储过程前，我们有必要了解了解Package的常量/变量。

在Oracle数据中，对于Package中的常量/变量均为会话级别，常量不允许被修改，变量允许被修改。

如果当前对变量进行了修改，当此会话退出之后，之前的修改不被保存。

比如说，此处尝试修改Package中的常量v\_var2，将其修改成9528。将会触发下方报错

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-474765b4-ac4b-4606-af3d-c3dcbec8b929.png)  

比如说，此处修改Package中的变量v\_var，打印之后，退出重新登录，再次打印。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-0d975cf5-d61d-4c09-955b-7c7bb77c4889.png)  

好好好，接下来让我们回到调用刚刚实现的那个存储过程上来。

这个时候，我们去调用test\_pkg.test\_proc的话，由于变量没有赋值过，所以结果如图所示：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-e9639c1a-c6ec-4d58-8e0a-bc93710ebf5f.png)  

叫上我们的华安老兄，再次调用

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-52392c7d-ef46-4b42-b682-0e33a8fd00b3.png)  

就可以看到华安了

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-2e40b565-3687-4a43-a354-a62aa69e7683.jpeg)  

当然还可以此处也仅仅是演示了简单类型，其实也可以结合复合类型玩玩，比如说：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240619-e78ad617-dd0f-46bb-b744-4db8739f927b.png)  

  

二、羲和（Halo）数据库在这方面的表现
====================

1、创建Package/Package body操作

2、修改常量/变量数据操作

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-57b71514-11b1-4d57-9a62-2ac4247a0bf4.png)  

3、呼叫迷途小书僮华安操作

![](https://oss-emcsprod-public.modb.pro/image/editor/20240618-0699aef0-33fb-4bb3-a86a-9dd97cff77d8.png)  

龙吟虎啸小书僮

![](https://oss-emcsprod-public.modb.pro/image/editor/20240619-7100aa9f-583f-40dc-ad1e-455048e4fa92.jpg)  

4、测试复合类型

![](https://oss-emcsprod-public.modb.pro/image/editor/20240619-bd02cf44-15eb-4f13-88bb-9d6db449f77e.png)  

  

三、声明
====

**本文为此系列第三篇，如想看到对Oracle功能的吐槽，敬请期待。**

**若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。**

**文章转载请联系，谢谢合作。**