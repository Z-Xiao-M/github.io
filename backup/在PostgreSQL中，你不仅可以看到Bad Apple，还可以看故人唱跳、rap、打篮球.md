一、前言
====

也是在前不久，我翻到了一个“有趣”的项目，如标题所言，这个PostgreSQL插件项目就是可能做到在psql客户端播放Bad Apple，和鸽鸽的唱跳、rap、打篮球的名场面。

项目名称：**[pg\_badapple](https://github.com/higuoxing/pg_badapple)**

项目地址：[https://github.com/higuoxing/pg\_badapple](https://github.com/higuoxing/pg_badapple)

为什么我会对这个感兴趣呢？

其实不在于鸽鸽，在于Bad Apple（真的，你信我！）

虽然本人并非二次元，但是好多知识都是在b站上学习到的。

印象中大概可能是四五年前吧，可能是我大二、大三的时候？（真没想到一晃这么多年过去了），那个时候基本上就是Bad Apple非常火的时候，

甚至可以号称是只要有屏幕，你就可以看到各路大神，在那个上面播放Bad Apple。

我印象中最深的一次就是，刷到过一个用示波器屏幕播放这个的，这还真不是我tree new bee，应该还能在b站上找到类似的视频，这里我就不去找了，感兴趣的朋友可以自己去瞅瞅。

  

二、pg\_badapple
==============

简单的安装流程如下所示：

    [postgres@halo-centos8 contrib]$ pwd
    /home/postgres/code/postgres/contrib
    [postgres@halo-centos8 contrib]$ git clone https://github.com/higuoxing/pg_badapple.git
    Cloning into 'pg_badapple'...
    remote: Enumerating objects: 33, done.
    remote: Counting objects: 100% (33/33), done.
    remote: Compressing objects: 100% (23/23), done.
    remote: Total 33 (delta 13), reused 25 (delta 8), pack-reused 0 (from 0)
    Receiving objects: 100% (33/33), 1.05 MiB | 113.00 KiB/s, done.
    Resolving deltas: 100% (13/13), done.
    [postgres@halo-centos8 contrib]$ cd pg_badapple
    [postgres@halo-centos8 pg_badapple]$ ls
    badapple--1.0.0.sql  badapple.c  badapple.control  badapple.txt  basketball.txt  Makefile  README.md
    [postgres@halo-centos8 pg_badapple]$ make 
    gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wshadow=compatible-local -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -O0 -fPIC -fvisibility=hidden -I. -I./ -I/u01/app/halo/product/dbms/16/include/postgresql/server -I/u01/app/halo/product/dbms/16/include/postgresql/internal  -D_GNU_SOURCE -I/usr/include/libxml2   -c -o badapple.o badapple.c -MMD -MP -MF .deps/badapple.Po
    gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wshadow=compatible-local -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -O0 -fPIC -fvisibility=hidden badapple.o -L/u01/app/halo/product/dbms/16/lib   -Wl,--as-needed -Wl,-rpath,'/u01/app/halo/product/dbms/16/lib',--enable-new-dtags  -fvisibility=hidden -shared -o badapple.so
    [postgres@halo-centos8 pg_badapple]$ make install
    /usr/bin/mkdir -p '/u01/app/halo/product/dbms/16/share/postgresql/extension'
    /usr/bin/mkdir -p '/u01/app/halo/product/dbms/16/share/postgresql/extension'
    /usr/bin/mkdir -p '/u01/app/halo/product/dbms/16/lib/postgresql'
    /usr/bin/install -c -m 644 .//badapple.control '/u01/app/halo/product/dbms/16/share/postgresql/extension/'
    /usr/bin/install -c -m 644 .//badapple.txt .//basketball.txt .//badapple--1.0.0.sql  '/u01/app/halo/product/dbms/16/share/postgresql/extension/'
    /usr/bin/install -c -m 755  badapple.so '/u01/app/halo/product/dbms/16/lib/postgresql/'
    [postgres@halo-centos8 pg_badapple]$ psql
    psql (16.8)
    Type "help" for help.
    
    postgres=# create extension badapple;
    CREATE EXTENSION
    postgres=# 

整个项目也很简单，就两个函数，想要播放Bad Apple就执行`SELECT play_badapple();`，想要看鸽鸽名场面就执行`SELECT play_basketball();`

2.1、Bad Apple
-------------

墨天轮似乎不能放视频，我没仔细研究就放张执行的结果图片好了

![](https://oss-emcsprod-public.modb.pro/image/editor/20250529-1928110380144930816_585460.png)

2.2、疑是故人来（铁山靠）
--------------

![](https://oss-emcsprod-public.modb.pro/image/editor/20250529-1928110823050850304_585460.png)

  

三、如何实现？
=======

整个项目都很简单，两个函数对应两个txt文件，代码实际逻辑在badapple.c之中。

![](https://oss-emcsprod-public.modb.pro/image/editor/20250529-1928112675763007488_585460.png)

查看其中一个txt文件

![](https://oss-emcsprod-public.modb.pro/image/editor/20250529-1928113126344503296_585460.png)

其实不放代码，估计你也明白了是如何实现的了。不过有始有终好了，让我们看一下实际的代码。

    /*
     * get the lines from a text file
     */
    static char **readfile(const char *path) {
    

阅读起来应该没啥难度，就是计算一下对应的txt文件路径，然后读取文件中相关字符内容，然后按照30行一帧的形式，按照固定的频率，打印出来，这样子就形成了连贯的字符动画

  

四、如何播放其他有趣的字符动画？
================

由于原作者只提供了两个函数和对应的字符文件来播放着对应的动画，我们要是想播放别的动画的话就显得无能为力了。

不过作者已经给出了实现思路我们简单改一改也是ok的。

首先我们需要确定的是，我们想播放其他任意的字符动画。那我们随便取个函数名叫play\_anything，

按照作者的思路，我们需要提供对应的字符文件，那么我们将字符文件的路径干脆就作为函数参数就好了，

而对于字符文件可能多少行算一帧也是有些不确定的，我们我们把多少行算一帧也作为一个参数好了。

也就是我们只需要实现一个名为play\_anything函数，函数存在两个参数，一个参数为具体的字符文件路径，另一个参数就是多少行字符构成一帧。

代码如下：

    Datum play_anything(PG_FUNCTION_ARGS) {
      char **play_lines;
      int numframes = 0;
      int numlines = 0;
      int nline = PG_GETARG_INT32(1);  /* per frame. */
      char play_frame[4000];
      struct timespec start_tm;
    
      play_lines = readfile(text_to_cstring(PG_GETARG_TEXT_PP(0)));
      for (int i = 0; play_lines[i]; i++)
        numlines++;
      numframes = numlines / nline;
    
      if (clock_gettime(CLOCK_MONOTONIC, &start_tm) != 0)
        ereport(ERROR, (errmsg("clock_gettime() failed due to: %m")));
    
      for (;;) {
        int frame;
        int len = 0;
        float elapsed;
        struct timespec current_tm;
        MemSet(play_frame, 0, sizeof(play_frame));
    
        if (clock_gettime(CLOCK_MONOTONIC, &current_tm) != 0)
          ereport(ERROR, (errmsg("clock_gettime() failed due to: %m")));
    
        elapsed = (current_tm.tv_sec - start_tm.tv_sec) +
                  ((current_tm.tv_nsec - start_tm.tv_nsec) / 1000000000.0);
        frame = (int)(elapsed * 10);
    
        if (frame >= numframes)
          break;
    
        for (int row = 0; row < nline; ++row) {
          memcpy(&play_frame[len], play_lines[frame * nline + row],
                 strlen(play_lines[frame * nline + row]));
          len += strlen(play_lines[frame * nline + row]);
        }
    
        ereport(NOTICE, (errmsg("\033c" /* Reset device. */
                                "%s",
                                play_frame)));
        pg_usleep(50000);
        CHECK_FOR_INTERRUPTS();
      }
    
      PG_RETURN_VOID();
    }

当然对应的sql文件也还是要改一下我这里就不展示了，就是创建个函数。

也可以选择不手动改，直接使用我改完的项目，项目地址：[https://github.com/Z-Xiao-M/pg\_display](https://github.com/Z-Xiao-M/pg_display)

函数也有了，接下来就是字符文件该如何生成的问题了，这里我用的另一个开源项目叫[video2ascii](https://github.com/jankozik/video2ascii)，项目地址[https://github.com/jankozik/video2ascii](https://github.com/jankozik/video2ascii)

这个展开讲也挺多内容的，这块就不过多描述了，需要大伙自己学一下，这里给个示例

    # video-to-ascii会生成对应的字符shell脚本

好像一般这个video-to-ascii生成的字符文件大概是25行一帧，没仔细研究。

理论上来说这样子就通过执行`select play_anything(xxx.txt文件路径，25);`

达成我们的目标了，要是觉得不是select play\_anything不是很好看，可以考虑在这个基础之上，再去创建一个函数封装一下，就像下面的图一样。

![play_anything](https://oss-emcsprod-public.modb.pro/image/editor/20250530-e478007f-9f07-4f48-8c46-70e2347afb35.png)

  

五、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。

本文没啥技术含量，权且博君一笑，图个乐呵~

无任何侮辱和鄙视他人的主观意愿，谢谢合作~