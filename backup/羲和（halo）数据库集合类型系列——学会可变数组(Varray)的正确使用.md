一、前言  

前不久，咱们带来了关于羲和数据库集合类型中的嵌套表、关联数组的使用，因为中途临时有事情要忙，所以小咕了一会，相关链接如下

[羲和（halo）数据库集合类型系列——学会嵌套表(Nested Table)的正确使用](https://www.modb.pro/db/1844644828275109888)  

[羲和（halo）数据库集合类型系列——关联数组（Associative Array）](https://www.modb.pro/db/1845638860305432576)  

今天要给大家带来的是**羲和（halo）数据库集合类型基础用法最终章——****可变数组(Varray)****相关内容。**  

可变数组（Varray）其实和嵌套表没什么太大的差异，最大的差异点就在于可变数组大小是声明类型时便已经确定了，而嵌套表是可以一直动态增长的。

  

二、可变数组（Varray）
==============

访问可变数组（Varray）的元素也和访问嵌套表元素一致，支持通过下标访问，最小下标值为1，通过小括号访问。  

    -- 访问n号元素 n为int类型 n >= 1
    collection_name(n)  

可变数组始终是密集的，嵌套表是可密集也可以稀疏的集合。如下图：  

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848181786733535232_585460.png)  

但其并不意味着可变数组（Varray）不支持DELETE集合方法，它允许通过DELETE删除集合中所有元素。

可变数组（Varray）语法定义如下：

    -- 在Package或匿名块中
    TYPE type_name IS {VARRAY | VARYING ARRAY} (size_limit) 
       OF element_type [NOT NULL];
    -- 在SQL场景中，创建全局的可变数组类型
    CREATE TYPE type_name IS {VARRAY | VARYING ARRAY} (size_limit) 
       OF element_type [NOT NULL];

  

三、可变数组（Varray）变量初始化
===================

如果想正常使用可变数组（Varray），则必须对可变数组（Varray）进行初始化的动作，而对可变数组（Varray）进行初始化，需要使用对应构造函数。对于此处的构造函数而言，便是和可变数组（Varray）类型同名的函数。

初始化的动作可以发生在声明区域，也可以发生在执行区域。初始化的元素个数不能超过定义的大小。一个简单的示例如下  

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848184527259275264_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848184626512224256_585460.png)  

  

四、集合方法
======

**集合方法是**对集合进行操作的内置函数或过程，一般点上相关方法，便代表着调用该方法。  

    collection_name.collection_method

可变数组（varray）也支持以下集合方法：

EXISTS

COUNT

LIMIT

FIRST and LAST

PRIOR and NEXT

EXTEND

TRIM

DELETE

通过相关的集合方法，可以帮助我们更容易的处理可变数组（Varray）的相关内容。

4.1、EXISTS
----------

EXISTS用于检查元素是否存在，一般的使用语法为EXISTS(n)，当可变数组（Varray）中n号位置元素存在，则返回ture，如果不存在，将返回false。

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848185664979951616_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848185740037021696_585460.png)  

  

4.2、COUNT
---------

COUNT方法用于返回可变数组中元素总体数量

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848186046620733440_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848186150018715648_585460.png)  

4.3、LIMIT
---------

LIMIT方法返回可变数组在声明时定义的大小。

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848187079027691520_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848187139488583680_585460.png)  

  

4.4、FIRST and LAST
------------------

FIRST方法用于返回可变数组中第一个元素下标，LAST方法用于返回可变数组中最后一个元素下标。

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848187457474486272_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848187550999076864_585460.png)  

  

4.5、PRIOR and NEXT
------------------

PRIOR方法用于返回上一个存在元素的下标，如果不存在上一个元素，则返回NULL。一般语法为PRIOR(n)

NEXT方法用于返回下一个存在元素的下标，如果不存在下一个元素，则返回NULL。一般语法为NEXT(n)

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848187895548567552_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848187997211168768_585460.png)  

4.6、EXTEND
----------

EXTEND方法用于扩充可变数组的大小，不得超过声明的最大大小。  

*   `EXTEND`将一个 null 元素追加到可变数组中。
    
*   `EXTEND(n)`将n个null 元素追加到可变数组中。
    

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848188590527963136_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848188668140425216_585460.png)  

  

4.7、TRIM
--------

有扩充也就会有回收，TRIM方法便是用于删除可变数组中尾部元素，并回收存储空间。

*   `TRIM`从集合的末尾删除一个元素。
    
*   `TRIM(n)`从集合的末尾删除n个元素。
    

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848189093601173504_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848189184534745088_585460.png)  

4.8、DELETE
----------

DELETE方法在可变数组中用于删除全部的元素。等价于n.TRIM(n.COUNT);

*   `DELETE`从集合中删除所有元素。
    

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848190068743299072_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848190124674342912_585460.png)  

  

五、全局可变数组类型
==========

全局可变数组类型可用作函数/存储过程参数或表字段类型。  

Halo：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848199773402259456_585460.png)  

Oracle：

![](https://oss-emcsprod-public.modb.pro/image/editor/20241021-1848199854222303232_585460.png)  

  

六、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文中相关测试内容基础环境为羲和（halo）数据库16版本Oracle模式（已创建拓展）和 Oracle19c。如需获取同样的体验，敬请期待后续16版本公测。

文章转载请联系，谢谢合作。