一、集合类型
======

在Oracle数据库中集合类型分为以下三种，分别为关联数组(Associative Array)、嵌套表(Nested Table)和可变数组(VARRAY)。更加详细的介绍如下：

关联数组(Associative Array)：又称索引表，通过数字或字符串作为下标来查找集合中的元素，元素数量不限。

嵌套表(Nested Table)：与关联数组类似，使用有序数字作为元素的下标，元素的最大数量不限。

可变数组(VARRAY)：使用有序数字作为元素的下标，元素的最大数量在定义时确定。

**我们注意到了部分客户对集合类型存在着使用需求，而羲和（halo）数据库14版本的集合类型尚不足以满足客户的要求，因此我们在羲和（halo）数据库16版本专门针对集合类型做了增强处理，**

**接下来将会给大家带来羲和（halo）数据库16版本集合类型系列文章，今天的主题是如何正确使用嵌套表(Nested Table)。**

![](https://oss-emcsprod-public.modb.pro/image/editor/20241011-1844648667641380864_585460.png)  

  

二、嵌套表(Nested Table)
===================

如果换一种角度来理解嵌套表(Nested Table)的话，可以将其类似看做是没有上限的一维数组。

并且访问嵌套表的元素也和访问数组元素类似，支持通过下标访问，最小下标值为1，通过小括号访问。

    -- 访问n号元素 n为int类型 n >= 1
    collection_name(n)  

当然也仅仅是**类似**看作一维数组，除了上面说到的和数组的区别在于：数组是有固定上限的，而嵌套表是无界的，大小可以动态增加。

除此之外，数组总是_密集_的（有连续的下标），嵌套表则不一定，嵌套表初始状态是密集集合，但是后续如果使用了DELETE集合方法（后续展开介绍），则会变成稀疏集合，从而不连续。

一张经典的图如下：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241011-1844672437994221568_585460.png)  

嵌套表语法定义如下：

    -- 在Package或匿名块中
    TYPE type_name IS TABLE OF element_type [NOT NULL];
    -- 在SQL场景中，创建全局的嵌套表类型
    CREATE TYPE type_name IS TABLE OF element_type [NOT NULL];

  

三、嵌套表变量初始化
==========

如果想正常使用嵌套表，则必须对嵌套表进行初始化的动作，而对嵌套表进行初始化，需要使用对应构造函数。对于此处的构造函数而言，便是和嵌套表类型同名的函数。

初始化的动作可以发生在声明区域，也可以发生在执行区域。  

![](https://oss-emcsprod-public.modb.pro/image/editor/20241011-1844680882106499072_585460.png)  

没有初始化的嵌套表被自动赋值为NULL，当引用此嵌套表时，将会抛出预定义的异常。  

![](https://oss-emcsprod-public.modb.pro/image/editor/20241011-1844726204576923648_585460.png)  

  

四、集合方法
======

**集合方法是**对集合进行操作的内置函数或过程，一般点上相关方法，便代表着调用该方法。  

    collection_name.collection_method

嵌套表支持以下集合方法：

EXISTS

COUNT

LIMIT

FIRST and LAST

PRIOR and NEXT

EXTEND

TRIM

DELETE

通过相关的集合方法，可以帮助我们更容易的处理嵌套表的相关内容。

4.1、EXISTS
----------

EXISTS用于检查元素是否存在，一般的使用语法为EXISTS(n)，当嵌套表中n号位置元素存在，则返回ture，如果不存在，将返回false

当嵌套表变得稀疏的时候，这个方法可以帮助我们避免引用不存在的元素。（引用不存在的元素将会抛出预定义的异常）

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844922694947667968_585460.png)  

4.2、COUNT
---------

COUNT方法用于返回嵌套表中元素总体数量

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844923462081675264_585460.png)  

4.3、LIMIT
---------

LIMIT方法仅对可变数组（VARRAY）有意义，对于已正常初始化的嵌套表而言，返回NULL

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844925043442282496_585460.png)  

4.4、FIRST and LAST
------------------

FIRST方法用于返回嵌套表中第一个元素下标，LAST方法用于返回嵌套表中最后一个元素下标。

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844926308214734848_585460.png)  

4.5、PRIOR and NEXT
------------------

PRIOR方法用于返回上一个存在元素的下标，如果不存在上一个元素，则返回NULL。一般语法为PRIOR(n)

NEXT方法用于返回下一个存在元素的下标，如果不存在下一个元素，则返回NULL。一般语法为NEXT(n)  

这对方法会跳过已删除的元素，所以当嵌套表变得稀疏时，也就是下标不连续，这个时候使用这对方法，可以更加方便的遍历整个嵌套表。

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844928942543831040_585460.png)  

4.6、EXTEND
----------

EXTEND方法用于扩充嵌套表的大小  

*   `EXTEND`将一个 null 元素追加到嵌套表中。
    
*   `EXTEND(n)`将n个null 元素追加到嵌套表中。
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844931504868257792_585460.png)  

4.7、TRIM
--------

有扩充也就会有回收，TRIM方法便是用于删除嵌套表中尾部元素，并回收存储空间。

*   `TRIM`从集合的末尾删除一个元素。
    
*   `TRIM(n)`从集合的末尾删除n个元素。
    

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844935062082646016_585460.png)  

4.8、DELETE
----------

DELETE方法用于删除嵌套表中的元素，咋一看和TRIM方法好像是一样的，但其实并不一样。  

二者的区别在于：DELETE方法删除元素后，元素的存储空间并没有被回收，因此，还可以通过赋值重新引用该元素；使用TRIM方法删除元素后，空间被回收，不可以再引用该元素，除非再次使用EXTEND方法。

因此使用DELETE方法，便可以将嵌套表变得稀疏。

*   `DELETE`从集合中删除所有元素。
    
*   `DELETE(n)`删除嵌套表第_`n`_个元素。
    

因此我们可以简单复现一下这张图

![](https://oss-emcsprod-public.modb.pro/image/editor/20241011-1844672437994221568_585460.png)  

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844939774001508352_585460.png)  

对删除的位置，重新赋值，可以重新访问

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844941662563434496_585460.png)  

  

五、全局嵌套表类型
=========

全局嵌套表类型可用作函数/存储过程参数或表字段类型。  

![](https://oss-emcsprod-public.modb.pro/image/editor/20241012-1844949243973435392_585460.png)  

  

六、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文中相关测试内容基础环境为羲和（halo）数据库16版本Oracle模式（已创建拓展）和 Oracle19c。

如需获取同样的体验，敬请期待后续16版本公测。

文章转载请联系，谢谢合作。