# Ninja
Ninja 是一款专注于快速构建的轻量级构建系统，核心目标是替代传统 Make 工具，凭借高效的依赖解析与任务调度能力，在 Chrome、LLVM 等大型项目中能显著提升编译效率，但其不擅长手动编写构建规则，需配合 CMake、Meson 等工具自动生成核心构建文件 build.ninja；使用时，先按系统差异安装（Linux 用包管理器、macOS 用 Homebrew、Windows 用 Chocolatey 或官网下载），再通过 CMake 等生成 build.ninja（如创建 build 目录并执行 cmake -G Ninja ..），最后执行 ninja 命令启动编译，可按需指定目标（ninja <目标名>）、清理产物（ninja clean）、设置并行线程数（ninja -j N）或查看详细命令（ninja -v），它适合大型项目及频繁迭代场景，能自动检测文件变更仅重新编译受影响部分，且不建议手动维护 build.ninja。

# 问题随笔
这是我的虚拟机配置8核4G，编译pg_duckdb出现的问题
系统资源过少，夯死，没有任何报错信息。
调整成8核12G之后就正常了（苦逼的笔记本~）
```
OVERRIDE_GIT_DESCRIBE=v1.3.2 \
GEN=ninja \
CMAKE_VARS="-DCXX_EXTRA=-fvisibility=default -DBUILD_SHELL=0 -DBUILD_PYTHON=0 -DBUILD_UNITTESTS=0" \
DISABLE_SANITIZER=1 \
DISABLE_ASSERTIONS=0 \
EXTENSION_CONFIGS="../pg_duckdb_extensions.cmake" \
make -C third_party/duckdb \
release
make[1]: 进入目录“/home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb”
mkdir -p ./build/release && \
cd build/release && \
cmake -G "Ninja" -DFORCE_COLORED_OUTPUT=1     -DENABLE_SANITIZER=FALSE -DENABLE_UBSAN=0  -DCXX_EXTRA=-fvisibility=default -DBUILD_SHELL=0 -DBUILD_PYTHON=0 -DBUILD_UNITTESTS=0 -DENABLE_EXTENSION_AUTOLOADING= -DENABLE_EXTENSION_AUTOINSTALL= -DDUCKDB_EXTENSION_CONFIGS="../pg_duckdb_extensions.cmake" -DLOCAL_EXTENSION_REPO=""  -DOVERRIDE_GIT_DESCRIBE="v1.3.2" -DDUCKDB_EXPLICIT_VERSION=""  -DCMAKE_BUILD_TYPE=Release ../.. && \
cmake --build . --config Release
-- git hash 71c5c07cdd, version v1.3.2, extension folder v1.3.2
-- Extensions will be deployed to: /home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb/build/release/repository
-- Load extension 'json' from '/home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb/extensions' @ v1.3.2
-- Load extension 'icu' from '/home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb/extensions' @ v1.3.2
-- Load extension 'httpfs' from https://github.com/duckdb/duckdb-httpfs @ 7ce5308
-- Load extension 'core_functions' from '/home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb/extensions' @ v1.3.2
-- Load extension 'parquet' from '/home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb/extensions' @ v1.3.2
-- Load extension 'jemalloc' from '/home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb/extensions' @ v1.3.2
CMake Warning at CMakeLists.txt:1345 (message):
  Extension 'httpfs' has a vcpkg.json, but build was not run with VCPKG.  If
  build fails, check out VCPKG build instructions in
  'duckdb/extension/README.md' or try manually installing the dependencies in
  /home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb/build/release/_deps/httpfs_extension_fc-src/vcpkg.json


-- Extensions linked into DuckDB: [json, icu, httpfs, core_functions, parquet, jemalloc]
-- Configuring done (2.7s)
-- Generating done (0.3s)
-- Build files have been written to: /home/postgres/code/16/contrib/pg_duckdb/third_party/duckdb/build/release
[3/504] Building CXX object src/execution/CMakeFiles/duckdb_execution.dir/ub_duckdb_execution.cpp.o
```