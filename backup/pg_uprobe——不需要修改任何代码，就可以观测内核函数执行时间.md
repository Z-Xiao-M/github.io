# 前言
之前在看ebpf的时候，了解到了kprobe （内核态探针）和 uprobe （用户态探针）。

> kprobe 和 uprobe 是 Linux 内核提供的两种动态跟踪机制，用于在不修改目标代码（内核或用户程序）的情况下，对特定指令或函数进行监控、调试或性能分析。两者核心功能相似，但适用场景不同：前者针对内核空间，后者针对用户空间。

当时就想着PostgreSQL有没有类似的插件，找了很多用ebpf写的观测插件，最后意外发现了postgrespro的这款新的插件，但是当时用起来还有不少问题，简单改了下之后便没怎么碰了，正好最近他们出了v0.3，恰好我手头上也有一个合适场景可以演示，就提溜着出来玩玩。

# 简单的项目介绍
pg_uprobe 是postgrespro开发的一款PostgreSQL扩展，主要用于跟踪和分析数据库会话中执行的查询，以及剖析 PostgreSQL 核心的 C 函数，帮助用户深入了解数据库内部运行机制、排查性能问题等，支持会话级跟踪和函数级性能分析。底层基于 [Frida](https://frida.re/) 及 [Frida Gum](https://github.com/frida/frida-gum) 库，通过动态代码注入实现功能，无需修改 PostgreSQL 源码。更多细节内容请查看项目主页。

> 项目地址:  https://github.com/postgrespro/pg_uprobe

# 编译安装
```bash
git clone https://github.com/postgrespro/pg_uprobe.git  # 拉取项目代码至contrib目录
cd pg_uprobe  
make && make install
shared_preload_libraries = 'pg_uprobe' #添加至配置文件
CREATE EXTENSION pg_uprobe;          #创建拓展
```

# 简单演示
在[查询耗时同临时表的数量呈线性增长](https://z-xiao-m.github.io/github.io/post/cha-xun-hao-shi-tong-lin-shi-biao-de-shu-liang-cheng-xian-xing-zeng-chang.html)中，我们通过修改代码，添加了一些计时的方法，才获得了`PreCommit_on_commit_actions`和`heap_truncate`两个内核函数的执行时间，而使用pg_uprobe我们可以做到不修改任何代码，就可以观测到这两个内核函数具体的执行时间。
```sql
postgres=# CREATE TEMP TABLE a_gtt (n numeric) ON COMMIT DELETE ROWS;
CREATE TABLE
postgres=# \timing
Timing is on.
postgres=# DO $$
DECLARE
  v_sql VARCHAR(100);
BEGIN
  FOR i IN 1..3000 LOOP
    v_sql := 'CREATE TEMP TABLE a_gtt'||i||'(n numeric) ON COMMIT DELETE ROWS';
    EXECUTE v_sql;
  END LOOP;
END;
$$ LANGUAGE plpgsql;
DO
Time: 3882.167 ms (00:03.882)
postgres=# select set_uprobe('PreCommit_on_commit_actions', 'HIST', false);
select set_uprobe('heap_truncate', 'HIST', false);
         set_uprobe          
-----------------------------
 PreCommit_on_commit_actions
(1 row)

Time: 2.837 ms
  set_uprobe   
---------------
 heap_truncate
(1 row)

Time: 0.708 ms
postgres=# select * from a_gtt;
 n 
---
(0 rows)

Time: 1226.956 ms (00:01.227)
postgres=# select stat_hist_uprobe('PreCommit_on_commit_actions');
                                      stat_hist_uprobe                                       
---------------------------------------------------------------------------------------------
 ("(..., 43.4 us)","                                                  ",0.000)
 ("(43.4 us, 406359.2 us)","@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@             ",75.000)
 ("(406359.2 us, 812674.9 us)","                                                  ",0.000)
 ("(812674.9 us, 1218990.7 us)","@@@@@@@@@@@@                                      ",25.000)
 ("(1218990.7 us, ...)","                                                  ",0.000)
(5 rows)

Time: 2.840 ms
postgres=# select stat_hist_uprobe('heap_truncate');
                                      stat_hist_uprobe                                       
---------------------------------------------------------------------------------------------
 ("(..., 1218778.2 us)","                                                  ",0.000)
 ("(1218778.2 us, 1218779.2 us)","                                                  ",0.000)
 ("(1218779.2 us, 1218780.2 us)",@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@,100.000)
 ("(1218780.2 us, ...)","                                                  ",0.000)
(4 rows)

Time: 0.772 ms
postgres=# select * from a_gtt;
 n 
---
(0 rows)

Time: 1239.887 ms (00:01.240)
postgres=# select stat_hist_uprobe('PreCommit_on_commit_actions');
                                      stat_hist_uprobe                                       
---------------------------------------------------------------------------------------------
 ("(..., 43.4 us)","                                                  ",0.000)
 ("(43.4 us, 410360.8 us)","@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@             ",75.000)
 ("(410360.8 us, 820678.2 us)","                                                  ",0.000)
 ("(820678.2 us, 1230995.6 us)","@@@@@@@@@@@@                                      ",25.000)
 ("(1230995.6 us, ...)","                                                  ",0.000)
(5 rows)

Time: 1.861 ms
postgres=# select stat_hist_uprobe('heap_truncate');
                                       stat_hist_uprobe                                       
----------------------------------------------------------------------------------------------
 ("(..., 1218778.2 us)","                                                  ",0.000)
 ("(1218778.2 us, 1224725.6 us)","@@@@@@@@@@@@@@@@@@@@@@@@@                         ",50.000)
 ("(1224725.6 us, 1230673.1 us)","@@@@@@@@@@@@@@@@@@@@@@@@@                         ",50.000)
 ("(1230673.1 us, ...)","                                                  ",0.000)
(4 rows)

Time: 0.789 ms
```
将us转换成ms，可以看到统计出来的时间大差不差，也不需要重新编译项目代码算得上是比较方便的。

# 函数探针示例
```sql
-- TIME
select set_uprobe('PreCommit_on_commit_actions', 'TIME', false);
select set_uprobe('heap_truncate', 'TIME', false);
select stat_time_uprobe('PreCommit_on_commit_actions');
select stat_time_uprobe('heap_truncate');
select delete_uprobe('PreCommit_on_commit_actions', false);
select delete_uprobe('heap_truncate', false);

-- HIST
select set_uprobe('PreCommit_on_commit_actions', 'HIST', false);
select set_uprobe('heap_truncate', 'HIST', false);
select stat_hist_uprobe('PreCommit_on_commit_actions');
select stat_hist_uprobe('heap_truncate');
select delete_uprobe('PreCommit_on_commit_actions', false);
select delete_uprobe('heap_truncate', false);
```
还有个会话级跟踪，但是这里没有涉及到我就不展开讲了。

# 注意事项
跟踪会带来约 5% 性能损耗，不建议长期开启。

# 评价
是个极好的项目，就是要玩出花来要对PostgreSQL内核比较熟悉，有一定的上手门槛。