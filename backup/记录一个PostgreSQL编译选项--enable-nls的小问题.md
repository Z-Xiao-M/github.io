## 问题复现
在编译[pg_onnx](https://github.com/kibae/pg_onnx)项目时，我的环境是在ubuntu~24.04上源码编译安装的PostgreSQL 16，项目的整体的编译命令非常简单
```bash
# 拉取项目代码（含子项目）
git clone --recursive https://github.com/kibae/pg_onnx.git
# 进入代码目录 运行脚本 安装microsoft/onnxruntime
./download-onnxruntime-linux.sh
# 运行编译选项
cmake -B build -S . -DCMAKE_BUILD_TYPE=Release
cmake --build build --target pg_onnx --parallel
sudo cmake --install build/pg_onnx
```
但是编译时却碰到了如下错误
```bash
In file included from /u01/app/halo/product/dbms/16/include/postgresql/server/postgres.h:45,
                 from /home/postgres/code/16/contrib/pg_onnx/pg_onnx/bridge/bgworker_side/../../pg_onnx.hpp:14,
                 from /home/postgres/code/16/contrib/pg_onnx/pg_onnx/bridge/bgworker_side/bgworker_side.hpp:8,
                 from /home/postgres/code/16/contrib/pg_onnx/pg_onnx/bridge/bgworker_side/bgworker.cpp:5:
/usr/include/libintl.h:39:14: error: expected unqualified-id before ‘const’
   39 | extern char *gettext (const char *__msgid)
      |              ^~~~~~~
/usr/include/libintl.h:39:14: error: expected ‘)’ before ‘const’
/u01/app/halo/product/dbms/16/include/postgresql/server/c.h:1190:20: note: to match this ‘(’
 1190 | #define gettext(x) (x)
      |                    ^
/usr/include/libintl.h:44:14: error: expected unqualified-id before ‘const’
   44 | extern char *dgettext (const char *__domainname, const char *__msgid)
      |              ^~~~~~~~
/usr/include/libintl.h:44:14: error: expected ‘)’ before ‘const’
/u01/app/halo/product/dbms/16/include/postgresql/server/c.h:1191:23: note: to match this ‘(’
 1191 | #define dgettext(d,x) (x)
      |                       ^
/usr/include/libintl.h:61:14: error: expected unqualified-id before ‘unsigned’
   61 | extern char *ngettext (const char *__msgid1, const char *__msgid2,
      |              ^~~~~~~~
/usr/include/libintl.h:61:14: error: expected ‘)’ before ‘unsigned’
/u01/app/halo/product/dbms/16/include/postgresql/server/c.h:1192:26: note: to match this ‘(’
 1192 | #define ngettext(s,p,n) ((n) == 1 ? (s) : (p))
      |                          ^
/usr/include/libintl.h:61:14: error: expected ‘)’ before ‘unsigned’
   61 | extern char *ngettext (const char *__msgid1, const char *__msgid2,
      |              ^~~~~~~~
/u01/app/halo/product/dbms/16/include/postgresql/server/c.h:1192:25: note: to match this ‘(’
 1192 | #define ngettext(s,p,n) ((n) == 1 ? (s) : (p))
      |                         ^
/usr/include/libintl.h:67:14: error: expected unqualified-id before ‘unsigned’
   67 | extern char *dngettext (const char *__domainname, const char *__msgid1,
      |              ^~~~~~~~~

```

## 解决问题
思考了很久，觉得非常不应该，明明环境都一样，但是死活就是编译不过。以为还是缺少了某种依赖，甚至我还参考了[.github/workflows/cmake-ubuntu-pgsql16.yml](https://github.com/kibae/pg_onnx/blob/main/.github/workflows/cmake-ubuntu-pgsql16.yml)用于测试的yml文件其他依赖选项，还是没能解决。后面发现测试是直接安装的编译好的包，而我是拉取源码手动编译的，可能是缺少了哪个编译选项，最后发现需要在编译时配置`--enable-nls`才能编译通过。

