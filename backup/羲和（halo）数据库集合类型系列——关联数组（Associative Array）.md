一、前言
====

前不久，咱们带来了关于羲和数据库集合类型中的嵌套表的使用，顺带和Oracle19c对比了一下，基本功能上大差不差，前文链接如下

[羲和（halo）数据库集合类型系列——学会嵌套表(Nested Table)的正确使用](https://www.modb.pro/db/1844644828275109888)  

上次将两款数据库的测试内容和运行结果一起截图，结果贴到文章中反而显得不是很清楚，像我这种眼神不太好使的同学，还需要多点一下图片。

这回咱们简单调整一下，先放出羲和（halo）数据库16版本的运行结果，再给出Oracle19c的运行截图，方便大家对比。

接下来给大家带来的是**羲和（halo）数据库集合类型系列——关联数组（Associative Array）相关内容。**

二、关联数组（Associative Array）
=============================

Associative Arrays（也称为 index-by 表）类似键值对的数组，其中每个键都是唯一的，用于在数组中查找相应的值。

键可以是整型或字符串类型，对于整数类型的键而言，键不必是连续的。

语法定义如下：

    TYPE type_name IS TABLE OF element_type [NOT NULL]
       INDEX BY [PLS_INTEGER | BINARY_INTEGER | VARCHAR2(size_limit)];

而访问的语法和嵌套表一致，只是没有嵌套表的限制，因为关联数组的键值比较随意

    collection_name(n)  

关联数组不需要进行初始化的操作，和绝大多数的类型不一样的是，当声明定义出一个关联数组类型变量，它的默认值不为NULL。

Halo:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845648581729026048_585460.png)  

Oracle:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845649096177188864_585460.png)  

  

三、集合方法
======

关联数组和嵌套表相比的话，嵌套表中元素的添加是通过EXTEND方法实现的，而关联数组可以直接赋值。

所以相较于嵌套表而言，关联数组的集合方法便没有EXTEND方法和TRIM方法。

调用语法一致，调用语法如下：

    collection_name.collection_method

关联数组支持以下集合方法：

EXISTS

COUNT

LIMIT

FIRST and LAST

PRIOR and NEXT

DELETE

通过相关的集合方法，可以帮助我们更容易的处理关联数组的相关内容

3.1、EXISTS
----------

EXISTS用于检查元素是否存在，一般的使用语法为EXISTS(n)，当关联数组中n号位置元素存在，则返回ture，如果不存在，将返回false。

和嵌套表不一样的是，此处的n的类型，可以是非整数类型，也就是字符串类型。由关联数组类型的声明决定。

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845654696185397248_585460.png)  

Oracle:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845654910199758848_585460.png)  

  

3.2、COUNT
---------

同嵌套表一致，COUNT方法用于返回关联数组中元素总体数量

Halo:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845655570398932992_585460.png)  

Oracle:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845655669909848064_585460.png)  

  

3.3、LIMIT
---------

同嵌套表一致，LIMIT方法仅对可变数组（VARRAY）有意义，对于已正常初始化的关联数组而言，返回NULL。

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845689131970691072_585460.png)  

Oracle：

预期行为不一致，不过国外有人和我有相似经历链接如下

[https://asktom.oracle.com/ords/f?p=100:11:0::NO::P11\_QUESTION\_ID:9549043600346802813](https://asktom.oracle.com/ords/f?p=100:11:0::NO::P11_QUESTION_ID:9549043600346802813)  

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845689251030204416_585460.png)  

但如果是INDEX BY BINARY\_INTEGER又是正确的，感兴趣的同学可以自己研究一下

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845708458866343936_585460.png)  

  

3.4、FIRST and LAST
------------------

如果关联数字类型定义键值为整型类型，则FIRST代表为最小值，可以为负数，LAST则为最大值。

若定义键值为字符串类型，则依据设定的排序规则，区分大小，FIRST、LAST分别反应此时的最大最小。

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845709300758573056_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845709387640438784_585460.png)  

  

3.5、PRIOR and NEXT
------------------

由于关联数组键值比较随意，所以如果需要遍历关联数组，就需要使用PRIOR和NEXT这两个方法了。

方法功能和方法名一致，不过有趣的是，当你输入不存在的键值，会依据内部的排序，来获取上一个值和下一个值（此处不演示）。

Halo:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845710941616435200_585460.png)  

Oracle:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845711079927803904_585460.png)  

  

3.6、DELETE
----------

同嵌套表一致，使用DELETE方法删除关联数组中的元素，不会回收存储空间。

Halo:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845712328815374336_585460.png)  

Oracle:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845712423610839040_585460.png)  

  

四、全局集合类型？
=========

在Oracle数据库中，关联数组是不允许创建全局类型的，不过这点在羲和数据库上放开了，我们支持使用如下语法去创建全局关联数组类型

    CREATE TYPE type_name IS TABLE OF element_type [NOT NULL]
       INDEX BY [PLS_INTEGER | BINARY_INTEGER | VARCHAR2(size_limit)];

不过Oracle其实也可以通过别的方法来创建类似的全局类型，比方说和Package结合。

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845718549890953216_585460.png)  

Oracle:

![](https://oss-emcsprod-public.modb.pro/image/editor/20241014-1845718678803939328_585460.png)  

  

五、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文中相关测试内容基础环境为羲和（halo）数据库16版本Oracle模式（已创建拓展）和 Oracle19c。

如需获取同样的体验，敬请期待后续16版本公测。

文章转载请联系，谢谢合作。