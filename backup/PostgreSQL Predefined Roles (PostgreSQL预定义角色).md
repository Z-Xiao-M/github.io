> [!CAUTION]
> 以下内容来自AI总结 如有错误 需自行甄别 此处用作记录
> 原文：https://boringsql.com/posts/postgresql-predefined-roles/

### 1. 一段话总结
PostgreSQL的**预定义角色**是15个内置的专项权限集合，旨在解决传统管理中“基础权限不足”与“超级用户权限过度授予”的二元困境，无需授予超级用户权限即可实现精细化运维——按功能分为数据访问（如`pg_read_all_data`）、监控与可观测性（如`pg_monitor`）、系统操作（如`pg_signal_backend`）、文件系统访问（如`pg_read_server_files`）及特殊用途五大类；其演进从PostgreSQL 9.6（2016）的首个角色`pg_signal_backend`持续到18（2025）的`pg_signal_autovacuum_worker`，不断解决实际运维痛点；其中**pg_database_owner**为特殊角色，无默认权限、成员资格随数据库动态变化且不可显式授予，修复了`public` schema的权限问题并支持模板继承，最终实现PostgreSQL运维从临时权限管理到**系统化工夫委派**的转变。


---


### 2. 思维导图（mindmap）
- PostgreSQL预定义角色
  - 1. 定义与核心价值
    - 解决“超级用户权限过度授予”痛点
    - 15个内置行政角色，提供granular（精细化）访问
    - 无需超级用户权限，覆盖常见运维场景
  - 2. 角色分类（按功能）
    - 数据访问角色
      - `pg_database_owner`：数据库级ownership，特殊角色
      - `pg_read_all_data`：所有表、视图、序列的读权限
      - `pg_write_all_data`：所有表、视图、序列的写权限
    - 监控与可观测性角色
      - `pg_monitor`：元角色（包含以下3个角色）
      - `pg_read_all_settings`：配置信息访问
      - `pg_read_all_stats`：统计视图访问
      - `pg_stat_scan_tables`：表扫描统计
    - 系统操作角色
      - `pg_signal_backend`：取消查询、终止会话
      - `pg_checkpoint`：执行CHECKPOINT命令（15+）
      - `pg_maintain`：VACUUM/ANALYZE/REINDEX等（17+）
      - `pg_signal_autovacuum_worker`：信号自动清理进程（18+）
    - 文件系统访问角色
      - `pg_read_server_files`：读取服务器文件
      - `pg_write_server_files`：写入服务器文件
      - `pg_execute_server_program`：执行服务器程序
    - 特殊用途角色
      - `pg_create_subscription`：逻辑复制管理（16+）
      - `pg_use_reserved_connections`：使用预留连接（16+）
  - 3. 演进历程（按版本）
    - 9.6（2016）：`pg_signal_backend`（解决无超级用户取消查询问题）
    - 10（2017）：`pg_monitor`等4个监控角色（提升可观测性）
    - 11（2018）：3个文件系统角色（安全管理文件访问）
    - 14（2021）：`pg_read_all_data`/`pg_write_all_data`/`pg_database_owner`（优化数据访问）
    - 15（2022）：`pg_checkpoint`（授权CHECKPOINT操作）
    - 16（2023）：`pg_use_reserved_connections`/`pg_create_subscription`（高流量连接/逻辑复制）
    - 17（2024）：`pg_maintain`（非超级用户执行维护操作）
    - 18（2025）：`pg_signal_autovacuum_worker`（信号自动清理进程）
  - 4. 特殊角色：pg_database_owner
    - 特性：数据库级动态成员、零默认权限、不可显式授予
    - 作用：修复`public` schema权限问题、支持模板继承（如函数自动应用到新数据库）
  - 5. 核心优势
    - 避免超级用户权限过度授予
    - 覆盖当前+未来创建的对象
    - 集群级作用域（除`pg_database_owner`）
    - 自动适配PostgreSQL新功能


---


### 3. 详细总结
#### 1. 引言：PostgreSQL传统权限管理的痛点
传统PostgreSQL运维中，权限分配存在**二元选择困境**：要么仅授予基础权限（无法满足运维需求），要么直接授予超级用户权限（风险极高，如误删数据库）。这一问题源于PostgreSQL历史上运维权限选项有限，或用户对现有选项认知不足，导致大量非必要场景下过度授予超级用户权限。


#### 2. 什么是预定义角色？
- **定义**：PostgreSQL内置的**15个专项行政角色**，为常见运维任务提供“按需分配”的精细化权限，无需授予超级用户权限即可完成监控、备份、维护等操作。
- **角色分类详情**（按功能划分）：

| 功能类别               | 角色名称                          | 核心权限                                                                 | 备注（适用版本）          |
|------------------------|-----------------------------------|--------------------------------------------------------------------------|---------------------------|
| 数据访问               | `pg_database_owner`               | 数据库级ownership，无默认权限，成员资格随数据库动态变化                   | 特殊角色，14+             |
| 数据访问               | `pg_read_all_data`                | 读取所有表、视图、序列的数据                                             | 14+                       |
| 数据访问               | `pg_write_all_data`               | 写入所有表、视图、序列的数据                                             | 14+，仍受RLS（行级安全）限制 |
| 监控与可观测性         | `pg_monitor`                      | 元角色，继承`pg_read_all_settings`/`pg_read_all_stats`/`pg_stat_scan_tables` | 10+，覆盖全面监控需求     |
| 监控与可观测性         | `pg_read_all_settings`            | 访问数据库配置参数（如`pg_settings`）                                     | 10+                       |
| 监控与可观测性         | `pg_read_all_stats`               | 访问统计视图（如`pg_stat_database`/`pg_stat_activity`）                   | 10+                       |
| 监控与可观测性         | `pg_stat_scan_tables`             | 扫描表以收集详细统计信息（如I/O模式）                                     | 10+                       |
| 系统操作               | `pg_signal_backend`               | 取消查询（`pg_cancel_backend`）、终止会话（`pg_terminate_backend`）        | 9.6+，首个预定义角色      |
| 系统操作               | `pg_checkpoint`                   | 执行`CHECKPOINT`命令                                                     | 15+                       |
| 系统操作               | `pg_maintain`                     | 执行VACUUM、ANALYZE、REINDEX、REFRESH MATERIALIZED VIEW等维护操作          | 17+                       |
| 系统操作               | `pg_signal_autovacuum_worker`     | 向自动清理进程发送信号（如取消特定表的VACUUM）                           | 18+                       |
| 文件系统访问           | `pg_read_server_files`            | 读取服务器文件系统中的文件                                               | 11+                       |
| 文件系统访问           | `pg_write_server_files`           | 向服务器文件系统写入文件                                                 | 11+                       |
| 文件系统访问           | `pg_execute_server_program`       | 在服务器上执行外部程序                                                   | 11+                       |
| 特殊用途               | `pg_create_subscription`          | 管理逻辑复制订阅                                                         | 16+                       |
| 特殊用途               | `pg_use_reserved_connections`     | 使用预留连接（解决高流量数据库运维连接问题）                             | 16+                       |


#### 3. 为什么使用预定义角色？
1. **解决权限过度授予问题**：打破“基础权限→超级用户”的二元限制，将运维能力拆解为专项角色（如监控团队仅需`pg_monitor`，备份服务仅需`pg_read_all_data`）。
2. **抽象化权限管理**：从“手动管理单个对象权限”转向“管理逻辑能力集”，明确“允许做什么”而非“允许访问哪个对象”，降低维护复杂度。
3. **覆盖范围广**：
   - 自动包含**当前及未来创建的所有表、视图、序列、schema**，无需重复授权；
   - 多数角色作用于**集群级**（仅`pg_database_owner`为数据库级），简化多数据库运维。
4. **自动适配新功能**：PostgreSQL新版本的运维功能会自动纳入预定义角色体系，无需手动调整权限。

- **示例对比**：传统授权 vs 预定义角色授权
  - 传统授权（需逐个schema授权）：
    ```sql
    GRANT USAGE ON SCHEMA finance, hr, app, audit TO analytics_team;
    GRANT SELECT ON ALL TABLES IN SCHEMA finance TO analytics_team;
    GRANT SELECT ON ALL SEQUENCES IN SCHEMA finance TO analytics_team;
    ```
  - 预定义角色授权（一次完成）：
    ```sql
    GRANT pg_read_all_data TO analytics_team;
    ```


#### 4. 预定义角色的演进历程
PostgreSQL各版本通过新增预定义角色，持续解决实际运维痛点，演进趋势从“单一功能”到“覆盖全运维场景”：

| PostgreSQL版本 | 发布年份 | 新增角色                          | 解决的核心问题                                                         |
|----------------|----------|-----------------------------------|--------------------------------------------------------------------------|
| 9.6            | 2016     | `pg_signal_backend`               | 无超级用户权限时无法取消查询、终止会话的问题                             |
| 10             | 2017     | `pg_monitor`/`pg_read_all_settings`/`pg_read_all_stats`/`pg_stat_scan_tables` | 缺乏精细化监控权限，无法安全授予监控团队访问配置和统计数据的能力           |
| 11             | 2018     | `pg_read_server_files`/`pg_write_server_files`/`pg_execute_server_program` | 服务器文件操作（如COPY FROM）需超级用户权限，存在安全风险                 |
| 14             | 2021     | `pg_read_all_data`/`pg_write_all_data`/`pg_database_owner` | 备份、ETL等场景需批量访问数据，且`public` schema权限混乱                 |
| 15             | 2022     | `pg_checkpoint`                   | 执行CHECKPOINT维护操作需超级用户权限，过度授权                           |
| 16             | 2023     | `pg_use_reserved_connections`/`pg_create_subscription` | 高流量数据库无法让非超级用户使用预留连接；逻辑复制管理需超级用户权限       |
| 17             | 2024     | `pg_maintain`                     | VACUUM、REINDEX等核心维护操作需超级用户权限，自动化脚本风险高             |
| 18             | 2025     | `pg_signal_autovacuum_worker`     | 无法在非超级用户权限下调整自动清理进程，导致性能问题（如峰值期VACUUM阻塞） |


#### 5. 特殊角色：`pg_database_owner`的“魔法特性”
`pg_database_owner`是唯一与其他预定义角色差异显著的角色，核心特性如下：
1. **动态成员资格**：成员资格随当前数据库变化，数据库所有者自动成为该角色成员（如`app_prod`数据库所有者`first_app`，自动拥有`pg_database_owner`权限）。
2. **零默认权限**：自身无任何预设权限，权限仅通过“手动授予”或“继承”获得，支持灵活的模板继承（如在`template1`中定义函数并授予该角色，新数据库自动继承）。
3. **不可显式授予**：无法通过`GRANT pg_database_owner TO 角色名`显式分配，避免权限滥用（执行该命令会报错：`ERROR: role "pg_database_owner" cannot have explicit members`）。
4. **修复`public` schema权限**：PostgreSQL 15前`public` schema由`postgres`用户所有，默认允许所有用户创建对象；15+后`public` schema由`pg_database_owner`所有，解决默认权限宽松的问题。

- **模板继承示例**：
  ```sql
  -- 1. 在template1中定义函数并授予pg_database_owner
  \c template1 postgres
  CREATE OR REPLACE FUNCTION db_owner_stats() RETURN ... SECURITY DEFINER;
  GRANT EXECUTE ON FUNCTION db_owner_stats() TO pg_database_owner;

  -- 2. 新创建的数据库自动继承该函数权限
  CREATE DATABASE app_prod OWNER first_app; -- first_app自动拥有db_owner_stats执行权限
  CREATE DATABASE demo_prod OWNER second_app; -- second_app自动拥有db_owner_stats执行权限
  ```


#### 6. 结论
PostgreSQL预定义角色将运维从“临时权限管理”升级为“系统化工夫委派”，核心价值在于：无需信任用户即可授予必要的运维能力，避免超级用户权限滥用；且其演进持续贴合实际运维需求，每个版本新增角色均针对性解决痛点。建议在需要超级用户权限完成运维任务时，优先检查是否存在对应的预定义角色。


---


### 4. 关键问题
#### 问题1：PostgreSQL预定义角色如何解决传统权限管理中“超级用户权限过度授予”的痛点？
答案：传统权限管理存在“基础权限不足”与“超级用户权限过度”的二元困境，预定义角色通过以下方式解决：  
1. **精细化权限拆分**：将超级用户的运维能力拆解为15个专项角色（如监控用`pg_monitor`、数据读取用`pg_read_all_data`、维护用`pg_maintain`），仅授予用户完成任务所需的最小权限；  
2. **抽象化权限逻辑**：从“管理单个表/ schema权限”转向“管理功能能力集”，例如授予`pg_read_all_data`即可覆盖所有当前及未来表的读权限，无需逐个授权；  
3. **集群级覆盖**：多数角色作用于集群级（非数据库级），简化多数据库运维，同时避免为每个数据库重复授予权限；  
4. **安全默认策略**：即使是`pg_write_all_data`这类广权限角色，仍受Row Level Security（RLS）限制，进一步降低风险。


#### 问题2：`pg_database_owner`作为特殊预定义角色，其“特殊性”体现在哪些方面？与其他角色有何核心差异？
答案：`pg_database_owner`的特殊性体现在4个核心维度，与其他角色差异显著：  
1. **动态成员资格**：其他角色（如`pg_monitor`）的成员固定，而`pg_database_owner`的成员随数据库变化——数据库所有者自动成为该角色成员（如`app_prod`所有者`first_app`，仅在`app_prod`中拥有该角色权限）；  
2. **零默认权限**：其他角色（如`pg_read_all_data`）有明确预设权限，而`pg_database_owner`自身无任何权限，仅通过“手动授予”或“模板继承”获得能力；  
3. **不可显式授予**：其他角色可通过`GRANT`命令分配（如`GRANT pg_monitor TO exporter`），而`pg_database_owner`无法显式授予，执行`GRANT pg_database_owner TO 角色名`会报错，避免权限滥用；  
4. **数据库级作用域**：其他角色（如`pg_signal_backend`）作用于集群级，而`pg_database_owner`仅作用于单个数据库，解决了`public` schema默认权限混乱的历史问题（15+后`public` schema由其所有，而非`postgres`用户）。


#### 问题3：PostgreSQL预定义角色的演进呈现出怎样的趋势？各关键版本的角色分别针对哪些运维痛点？
答案：预定义角色的演进趋势为“从单一功能到全场景覆盖”，每个关键版本均针对当时的核心运维痛点新增角色，具体如下：  
1. **初始阶段（9.6，2016）**：首个角色`pg_signal_backend`，解决“无超级用户权限无法取消异常查询、终止会话”的痛点，是预定义角色体系的起点；  
2. **监控强化（10，2017）**：新增`pg_monitor`及3个下属监控角色，解决“监控团队需访问配置/统计数据，但无法安全授予超级用户权限”的痛点，提升生产环境可观测性；  
3. **文件安全（11，2018）**：新增3个文件系统角色（如`pg_read_server_files`），解决“ETL/数据导入需读取服务器文件，但需超级用户权限”的安全风险；  
4. **数据访问优化（14，2021）**：新增`pg_read_all_data`/`pg_write_all_data`，解决“备份、ETL需批量访问数据，需逐个schema授权”的繁琐问题；同时新增`pg_database_owner`，修复`public` schema权限混乱；  
5. **维护权限下放（15-17，2022-2024）**：15年`pg_checkpoint`授权CHECKPOINT操作，17年`pg_maintain`开放VACUUM/REINDEX等核心维护操作，解决“自动化维护脚本需超级用户权限，风险高”的痛点；  
6. **高可用与性能（16-18，2023-2025）**：16年`pg_use_reserved_connections`解决高流量数据库运维连接问题、`pg_create_subscription`支持逻辑复制；18年`pg_signal_autovacuum_worker`允许调整自动清理进程，解决峰值期VACUUM阻塞性能的问题。  
整体演进始终围绕“降低超级用户依赖、贴合实际运维需求”展开，逐步覆盖监控、文件、数据、维护、高可用全场景。
