一、PostgreSQL登录事件触发器
===================

在PostgreSQL17版本引入了登录事件触发器，可以用于登录之后记录一些信息或者完成一些初始化动作。一个简单的示例来着PostgreSQL官方文档[https://www.postgresql.org/docs/17/event-trigger-database-login-example.html](https://www.postgresql.org/docs/17/event-trigger-database-login-example.html)

在这里我简单粘贴一下示例：

    -- create test tables and roles
    CREATE TABLE user_login_log (
      "user" text,
      "session_start" timestamp with time zone
    );
    CREATE ROLE day_worker;
    CREATE ROLE night_worker;
    
    -- the example trigger function
    CREATE OR REPLACE FUNCTION init_session()
      RETURNS event_trigger SECURITY DEFINER
      LANGUAGE plpgsql AS
    $$
    DECLARE
      hour integer = EXTRACT('hour' FROM current_time at time zone 'utc');
      rec boolean;
    BEGIN
    -- 1. Forbid logging in between 2AM and 4AM.
    IF hour BETWEEN 2 AND 4 THEN
      RAISE EXCEPTION 'Login forbidden';
    END IF;
    
    -- The checks below cannot be performed on standby servers so
    -- ensure the database is not in recovery before we perform any
    -- operations.
    SELECT pg_is_in_recovery() INTO rec;
    IF rec THEN
      RETURN;
    END IF;
    
    -- 2. Assign some roles. At daytime, grant the day_worker role, else the
    -- night_worker role.
    IF hour BETWEEN 8 AND 20 THEN
      EXECUTE 'REVOKE night_worker FROM ' || quote_ident(session_user);
      EXECUTE 'GRANT day_worker TO ' || quote_ident(session_user);
    ELSE
      EXECUTE 'REVOKE day_worker FROM ' || quote_ident(session_user);
      EXECUTE 'GRANT night_worker TO ' || quote_ident(session_user);
    END IF;
    
    -- 3. Initialize user session data
    CREATE TEMP TABLE session_storage (x float, y integer);
    ALTER TABLE session_storage OWNER TO session_user;
    
    -- 4. Log the connection time
    INSERT INTO public.user_login_log VALUES (session_user, current_timestamp);
    
    END;
    $$;
    
    -- trigger definition
    CREATE EVENT TRIGGER init_session
      ON login
      EXECUTE FUNCTION init_session();
    ALTER EVENT TRIGGER init_session ENABLE ALWAYS;

而在17版本之前，我们可以尝试使用login\_hook这款插件来完成类似的动作。

  

二、login\_hook
=============

login\_hook项目地址[https://github.com/splendiddata/login\_hook](https://github.com/splendiddata/login_hook)

正如开头所说在17版本引入了登录事件触发器，所以对于后续的版本而言，这个插件的意义就没啥太大的意义了（当然在17版本之前还是有意义的），

所以项目的维护者在项目的README写道将于PostgreSQL18停止维护此项目。

2.1、源码编译
--------

源码编译（经典三板斧）演示如下

    [postgres@halo-centos8 postgres]$ cd contrib/
    [postgres@halo-centos8 contrib]$ git clone https://github.com/splendiddata/login_hook.git
    Cloning into 'login_hook'...
    remote: Enumerating objects: 246, done.
    remote: Counting objects: 100% (153/153), done.
    remote: Compressing objects: 100% (112/112), done.
    remote: Total 246 (delta 101), reused 78 (delta 31), pack-reused 93 (from 1)
    Receiving objects: 100% (246/246), 60.97 KiB | 790.00 KiB/s, done.
    Resolving deltas: 100% (156/156), done.
    [postgres@halo-centos8 contrib]$ cd login_hook/
    [postgres@halo-centos8 login_hook]$ make && make install
    gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wshadow=compatible-local -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -O0 -fPIC -fvisibility=hidden -I. -I./ -I/u01/app/halo/product/dbms/16/include/postgresql/server -I/u01/app/halo/product/dbms/16/include/postgresql/internal  -D_GNU_SOURCE -I/usr/include/libxml2   -c -o login_hook.o login_hook.c -MMD -MP -MF .deps/login_hook.Po
    gcc -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Werror=vla -Wendif-labels -Wmissing-format-attribute -Wimplicit-fallthrough=3 -Wcast-function-type -Wshadow=compatible-local -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -Wno-format-truncation -Wno-stringop-truncation -g -O0 -fPIC -fvisibility=hidden -shared -o login_hook.so login_hook.o -L/u01/app/halo/product/dbms/16/lib    -Wl,--as-needed -Wl,-rpath,'/u01/app/halo/product/dbms/16/lib',--enable-new-dtags  -fvisibility=hidden 
    /usr/bin/mkdir -p '/u01/app/halo/product/dbms/16/lib/postgresql'
    /usr/bin/mkdir -p '/u01/app/halo/product/dbms/16/share/postgresql/extension'
    /usr/bin/mkdir -p '/u01/app/halo/product/dbms/16/share/postgresql/extension'
    /usr/bin/mkdir -p '/u01/app/halo/product/dbms/16/share/doc/postgresql/extension'
    /usr/bin/install -c -m 755  login_hook.so '/u01/app/halo/product/dbms/16/lib/postgresql/login_hook.so'
    /usr/bin/install -c -m 644 .//login_hook.control '/u01/app/halo/product/dbms/16/share/postgresql/extension/'
    /usr/bin/install -c -m 644 .//login_hook--1.0.sql .//login_hook--1.0--1.1.sql .//login_hook--1.1.sql .//login_hook--1.1--1.2.sql .//login_hook--1.2.sql .//login_hook--1.2--1.3.sql .//login_hook--1.3.sql .//login_hook--1.3--1.4.sql .//login_hook--1.4.sql .//login_hook--1.4--1.5.sql .//login_hook--1.5.sql .//login_hook--1.5--1.6.sql .//login_hook--1.6.sql  '/u01/app/halo/product/dbms/16/share/postgresql/extension/'
    /usr/bin/install -c -m 644 .//login_hook.html .//login_hook.css '/u01/app/halo/product/dbms/16/share/doc/postgresql/extension/'

2.2、源码分析
--------

在介绍使用之前，我们简单学习一下它的实现思路，因为这个项目真的非常非常的简单，所以分析之后介绍使用其实更加"丝滑"。

只有一个项目中只有一个c文件名为login\_hook.c，里面实际内容也不多，就三个函数，其中最主要的就是\_PG\_init

另外两个不是特别重要，这里简单口述一下，一个用来显示login\_hook的版本，另一个用于显示是否正在执行login\_hook.login函数

简化的\_PG\_init如下

    void _PG_init(void)
    {
    　　　　// 某些状态下不应该进行操作 此处保留了相关注释内容
    	/*
    	 * If no database is selected, then it makes no sense trying to execute
    	 * login code.
    	 * This may occur for example in a replication target database.
    	 */

所以我们想要登录之后，让这个插件帮助我们提前做好各种事情的前提便是——我们需要在login\_hook下创建一个名为login的函数，然后相关具体的动作需要落到这个函数中。

这就比较考验你函数的具体设计与实现了，铁铁~

2.3、简单使用介绍
----------

首先修改postgresql.conf

    session_preload_libraries = 'login_hook'

然后创建拓展 、创建login函数、还有调整权限

    [postgres@halo-centos8 16]$ psql
    psql (16.8)
    Type "help" for help.
    
    postgres=# create extension login_hook;
    CREATE EXTENSION
    postgres=# CREATE OR REPLACE FUNCTION login_hook.login() RETURNS VOID LANGUAGE PLPGSQL AS $$
    postgres$# DECLARE
    postgres$#     ex_state   TEXT;
    postgres$#     ex_message TEXT;
    postgres$#     ex_detail  TEXT;
    postgres$#     ex_hint    TEXT;
    postgres$#     ex_context TEXT;
    postgres$# BEGIN
    postgres$# IF NOT login_hook.is_executing_login_hook()
    postgres$# THEN
    postgres$#     RAISE EXCEPTION 'The login_hook.login() function should only be invoked by the login_hook code';
    postgres$# END IF;
    postgres$# 
    postgres$# BEGIN
    postgres$#    --
    postgres$#    -- Do whatever you need to do at login here.
    postgres$#    -- For example:
    postgres$#    RAISE NOTICE 'Hello %', current_user;
    postgres$# EXCEPTION
    postgres$#    WHEN OTHERS THEN
    postgres$#        GET STACKED DIAGNOSTICS ex_state   = RETURNED_SQLSTATE
    postgres$#                              , ex_message = MESSAGE_TEXT
    postgres$#                              , ex_detail  = PG_EXCEPTION_DETAIL
    postgres$#                              , ex_hint    = PG_EXCEPTION_HINT
    postgres$#                              , ex_context = PG_EXCEPTION_CONTEXT;
    postgres$#        RAISE LOG e'Error in login_hook.login()\nsqlstate: %\nmessage : %\ndetail  : %\nhint    : %\ncontext : %'
    postgres$#                , ex_state
    postgres$#                , ex_message
    postgres$#                , ex_detail
    postgres$#                , ex_hint
    postgres$#                , ex_context;
    postgres$#     END;
    postgres$# END
    postgres$# $$;
    CREATE FUNCTION
    postgres=# GRANT EXECUTE ON FUNCTION login_hook.login() TO PUBLIC;
    GRANT

当我们退出、然后再次登录时就可以看到以下内容

    [postgres@halo-centos8 16]$ psql
    NOTICE:  Hello postgres
    psql (16.8)
    Type "help" for help.
    
    postgres=# CREATE USER halo SUPERUSER;
    CREATE ROLE
    postgres=# \c - halo
    NOTICE:  Hello halo
    You are now connected to database "postgres" as user "halo".

  

三、声明
====

若文中存在错误或不当之处，敬请指出，以便我进行修正和完善。希望这篇文章能够帮助到各位。

文章转载请联系，谢谢合作~