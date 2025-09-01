看到今天的主题，可能有不少的朋友会吐槽一个Oracle的CHAR（1）类型有什么让人费解的这不是简简单单吗？而且CHAR（1）在各个数据库中也都支持的呀？我想说的是大伙，别着急咱们接下来慢慢看。

一、ORACLE——PL/SQL中CHAR（1）的表现
===========================

开门见题，还是以一个例子开头，如果是执行如下的匿名块，Oracle的输出结果是什么呢？

    DECLARE
     col1 CHAR(1);
     BEGIN
     col1 := '';
     DBMS_OUTPUT.PUT_LINE('Length :'||TO_CHAR(LENGTH(col1)));
     IF (col1 IS NULL) THEN
     dbms_output.put_line('Treated as NULL');
     END IF;
     IF (col1 = '') THEN
     dbms_output.put_line('Treated as EMPTY STRING');
     END IF;
     IF (col1 = ' ') THEN
     dbms_output.put_line('Treated as space');
     END IF;
     END;
    /
    

整个匿名块其实非常简单，就是定义了一个名为col1的变量，且变量类型为CHAR(1)，接下来被赋值成了 '' ，输出一下当前的长度，接下来就是几个简单的判断，最后输出对应的字符串。建议思考一会…

**如果看到这的话，估计有同学会说，所见即所得嘛，既然变量col1被赋值成了''，那么符合col1 = ''这一条件，最后应该输出’Treated as EMPTY STRING’才对。**

**如果是对Oracle的SQL比较了解的同学，可能会给出col1变量符合 IS NULL的条件，最终应该会输出’Treated as NULL’。**

那么最终到底结果如何呢？接下来给出运行结果

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-09efa2ab-73da-4e89-901d-c3b96a288c3b.png)

可以看到的是最终输出长度为一，最终结果’Treated as space’。

这时候有的同学可能想说：“我先不纠结为什么最后结果是这样的，你刚刚是不是说了‘**对Oracle的SQL比较了解**’，难道说在SQL层面还有新的花样？”。

接下来咱们来看看Oracle的SQL层面对类型为CHAR（1）的表现。

二、ORACLE——SQL中CHAR（1）的表现
========================

那么当存在一个字段，字段类型为CHAR（1），插入字段值为 '' ，这个时候到底实际的数据值到底是什么呢？

以如下SQL进行测试：

    CREATE TABLE ta(col1 CHAR(1));
    INSERT INTO ta VALUES(''); 
    SELECT 1,col1 FROM ta WHERE col1 IS NULL;
    SELECT 1,col1 FROM ta WHERE col1 = '';
    SELECT 1,col1 FROM ta WHERE col1 = ' ';
    

创建一张名为ta的表，表中有一名为col1的字段，字段类型为CHAR(1)，插入一条 '' 数据，接下来使用三条SELECT分别去判断此时col1到底为何值？

建议大家再思考一会…

最终运行结果如下所示：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-7cf6b8d4-247e-4d46-8804-4e2ae89336fe.png)

可以看到的是，最终判定col1不为 '' ,也不为 ' '（空格），而是NULL。

**可以看到的是，对于CHAR（1）这个同样的类型，同样的输入值，但是最终的输出，在Oracle的SQL层面和PL/SQL层面完全不一致。**

**至于这到底算不算是Oracle的bug，就交由诸君定夺。**

**不过我合理怀疑Oracle数据库研发存在多个团队，SQL层面有一个团队，PL/SQL层面也存在一个团队，**

**两个团队同时进行研发，两个团队中对于部分重合的内容设计各有各的想法，比如说（CHAR(1)），就像经典的修桥问题，如果用行业黑话来讲就是“颗粒度没对齐”，**

**这也就是今天的主题 《研发团队"颗粒度"没对齐——让人费解的CHAR（1）类型》。**

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-54854a7f-7413-4426-adb7-cf9e379de2ee.png)

三、PostgreSQL中CHAR（1）的表现
=======================

前面也说到了其实不仅仅只有Oracle支持CHAR类型，那么这块再把PostgreSQL拉过来玩玩。以如下SQL进行测试，

首先是plpgsql

    DO $$
    DECLARE
     col1 CHAR(1);
     BEGIN
     col1 := '';
     raise notice 'Length : %', LENGTH(col1);
     IF (col1 IS NULL) THEN
     raise notice 'Treated as NULL';
     END IF;
     IF (col1 = '') THEN
     raise notice 'Treated as EMPTY STRING';
     END IF;
     IF (col1 = ' ') THEN
     raise notice 'Treated as space';
     END IF;
     END;
    $$ language plpgsql;
    

运行结果

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-8583e13d-bb43-4783-8c48-0acd462b7a45.png)

接下来是相同的SQL测试

    CREATE TABLE ta(col1 CHAR(1));
    INSERT INTO ta VALUES(''); 
    SELECT 1,col1 FROM ta WHERE col1 IS NULL;
    SELECT 1,col1 FROM ta WHERE col1 = '';
    SELECT 1,col1 FROM ta WHERE col1 = ' ';
    

运行结果：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-91fa40ae-4121-4c6e-b2c7-d8f71bddac4b.png)

可以看到的是，在PostgreSQL中如果类型为CHAR（1），且输入时 '' ,那么实际进行匹配的时候，既可以是所见即所得的 ''，也可以是 ' '（空格）。

而对于PostgreSQL而言，不管是PL/SQL层面还是SQL层面，它的输出都是一致的，没有割裂的感觉。这是因为PostgreSQL的PL/SQL引擎其实是在原本SQL引擎的基础上实现的。

四、羲和（Halo）数据库
=============

都讲到这了，拉上我们的数据库玩玩。对于羲和（Halo）数据库而言，则是要分模式来测试了。

比如说**羲和数据库（PG模式）运行结果如下：**

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-6bed9d1f-7a09-455f-b746-2b3addb81a66.png)

可以看到的是，和PostgreSQL输出结果一致。

接下来是**羲和数据库（Oracle模式）运行结果如下：**

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-a40a24b9-b735-42ae-ae54-8ab82d6e775f.png)

可以看到的对于羲和数据库Oracle模式而言，对于CHAR（1）类型，如论在何种场景之下，保证了和SQL场景下的输出结果一致。

而对于其他类型比如说VARCHAR2（1）等没有任何歧义的，我们的最终输出结果与Oracle一致.

如下所示：

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-9b29384b-f7a9-42ab-93a4-5542841ccf81.png) ![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-b022dd54-427a-46a8-b7c0-f790b7e699d2.png)

![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-9bc266bb-d8dd-4f3d-8c40-686432b01b83.png) ![](https://oss-emcsprod-public.modb.pro/image/editor/20240617-2ad28087-2273-4c9c-aeb5-67bf29fcc84f.png)

五、声明
====

本文为此系列第一篇，如想看到对Oracle功能的吐槽，敬请期待期待。

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文章转载请联系，谢谢合作。