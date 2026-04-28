# 一、前言

说起临时表，多数的DBA同学们第一时间都会想到Oracle的全局临时表，而对于PG而言，虽然存在对应的全局临时表语法，但是并没有实质的全局临时表功能。

而如果我们在PG系数据库上使用全局临时表的功能，则要借助插件pgtt的功能，而为了更加了解pgtt实现全局临时表的实现思想，我们需要对PG原生的临时表有一定的理解。

# 二、PostgreSQL临时表

PostgreSQL存在两种临时表，一种是会话级临时表，另一种是事务级临时表。二者当会话结束时都会消失，区别主要是表中数据的生命周期不一致。

## 2.1、会话级临时表

会话级临时表中数据的生命周期为整个会话。语法如下

    CREATE TEMPORARY | TEMP TABLE ...

TEMP与TEMPORARY等价

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-684984a9-0541-427f-a88c-8e42e6cca448.png)

多个会话允许创建同名临时表，如下会话一创建完临时表的同时打开新的会话，会话二创建同名临时表，运行结果如下

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-4cdf80d0-9693-4e41-9d6d-fd50434a6332.png)

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-acb2d9b1-65a2-4357-b25b-90023c6feac4.png)

**两者最主要的区别在于，temp\_ta的schema不一致！**

**会话一为pg\_temp\_3.temp\_ta**

**会话二为pg\_temp\_4.temp\_ta**

## 2.2、事务级临时表

语法如下

    CREATE TEMPORARY | TEMP TABLE ... ON COMMIT ...

在ON COMMIT后面有三个选项分别是

**DELETE ROWS：数据仅存在事务周期中，事务提交后，临时表中的数据就清除掉了。**

\*\*DROP：\*\***数据仅存在事务周期中，事务提交后，临时表就删除了，所以需要把建表语句放在一个事务中**。

PERSERVE ROWS：（默认）创建会话级临时表的选项，可以不写。

### 2.2.1、ON COMMIT DELETE ROWS

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-1eb124e0-0a2b-4c08-9e64-5c96103061ed.png)

### 2.2.2、ON COMMIT DROP

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-ed15c035-9b71-4b37-9bef-d5b15d237db8.png)

# 三、pg\_temp和pg\_temp\_xx

在上面我们查看执行计划的时候，能看到pg\_temp\_xx（当然还是要区分版本，好像是15版本之后，仅会显示pg\_temp），

而这个pg\_temp\_xx和pg\_temp有什么关系了，我们接着用PG14来研究研究。

我们先创建一张会话级的临时表，然后插入一点数据。

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-d0ae4df9-d67d-4fe0-8c03-6526a0ba9929.png)

接着我们查看一下执行计划，看看是pg\_temp\_xx多少，

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-6832b5a4-7a96-4cdf-b35d-30661ff5986f.png)

可以看到的是此时为pg\_temp\_3，这个时候我们使用查询一下pg\_temp\_3.temp\_ta

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-ecdf9241-3419-4319-8513-d8b2c515f8cc.png)

可以看到就是我们刚刚插入的数据，这个时候我们再使用pg\_temp，查询一下表temp\_ta

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-a5ec294e-3354-407c-91e3-925d48f25d69.png)

可以看到二者的结果一模一样，再查看一下此时的执行计划

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-aa681005-8237-45f2-b889-1d47b28dbe91.png)

可以看到，实际上还是访问的pg\_temp\_3下的temp\_ta表。

所以pg\_temp是外在表现，而pg\_temp\_xx是内在本质，PG允许你使用pg\_temp访问当前会话下的临时表，

在内部处理时，会根据实际情形，将pg\_temp变换成pg\_temp\_xx。而xx的含义是什么呢？

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-6503a972-8409-436a-b2e2-8acc7e059fb3.png)

xx其实就是MyBackendId。

# 四、MyBackendId

都看到这了，肯定有同学好奇，想问“那有什么办法知道当前的会话的MyBackendId吗？”我想说“木有”

更为准确的说法是“没有直接的方法告知你当前会话的MyBackendId，PG没有提供函数给大家直接获取这个值，但是通过弯弯绕绕的话，我们还是有办法获取到MyBackendId的”

那就是**virtualxid（虚拟事务ID）**

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-fc565956-1211-46c5-b2ed-7b799728ab8a.png)

上图为虚拟事务ID的数据结构，而实际在使用虚拟事务ID时如下

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-e1f82d6a-921b-415a-a4df-71c396e69923.png)

可以看到上虚拟事务ID的组成之一便为MyBackendId。因此我们可以通过查询pg\_lock\_status函数或者pg\_locks视图获取virtualxid，最后获取MyBackendId

    postgres=# \sv pg_locks
    CREATE OR REPLACE VIEW pg_catalog.pg_locks AS
     SELECT l.locktype,
        l.database,
        l.relation,
        l.page,
        l.tuple,
        l.virtualxid,
        l.transactionid,
        l.classid,
        l.objid,
        l.objsubid,
        l.virtualtransaction,
        l.pid,
        l.mode,
        l.granted,
        l.fastpath,
        l.waitstart
       FROM pg_lock_status() l(locktype, database, relation, page, tuple, virtualxid, transactionid, classid, objid, objsubid, virtualtransaction, pid, mode, granted, fastpath, waitstart)

    postgres=# \sf pg_lock_status
    CREATE OR REPLACE FUNCTION pg_catalog.pg_lock_status(OUT locktype text, OUT database oid, OUT relation oid, OUT page integer, OUT tuple smallint, OUT virtualxid text, OUT transactionid xid, OUT classid oid, OUT objid oid, OUT objsubid smallint, OUT virtualtransaction text, OUT pid integer, OUT mode text, OUT granted boolean, OUT fastpath boolean, OUT waitstart timestamp with time zone)
     RETURNS SETOF record
     LANGUAGE internal
     PARALLEL SAFE STRICT
    AS $function$pg_lock_status$function$

获取virtualxid

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-a8d7c16d-1a7a-477d-bd61-658a6046b522.png)

而这个3即为MyBackendId，我们可以创建一张临时表，然后再通过查询执行计划验证一下

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-1c71d63c-5d6e-4fb5-a2d5-a9759cd5899c.png)

可以看到当前会话的MyBackendId，此时也正是使用的pg\_temp3。

其实到这里就差不多该结束了，不过可能你对虚拟事务ID还有别的问题？

![](https://oss-emcsprod-public.modb.pro/image/editor/20240624-ff97b121-ce7d-495d-a0a6-6faf8c1eb634.png)

就像下面的图一样，每执行一次，数值增加一个。这是为什么呢？

或者说是PG不是已经有事务ID的功能了吗，怎么还有一个虚拟事务ID，它们之间有什么关系吗?

那如果你想更加深入的理解虚拟事务ID的话，或许你可以想想如果每一条SELECT语句都能实实在在获取到一个真实的事务ID的话，

对于大多数应用系统都是读多写少，那该是一个怎样的画面，当然这仅仅只是一方面。

碍于篇幅的原因，就聊的这，如果后续有时间的话，到时候可以分享分享虚拟事务ID的意义。

# 五、声明

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文章转载请联系，谢谢合作。
