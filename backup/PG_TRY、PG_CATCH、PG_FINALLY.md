> [!CAUTION]
> 以下内容来自AI总结 如有错误 需自行甄别 此处用作记录

# PG_TRY系列宏的核心功能与用法
这些宏是 PostgreSQL 内核中用于捕获ereport(ERROR)级错误的核心机制，类似高级语言中的try-catch-finally，但专为 PostgreSQL 的错误处理模型设计。
1. 基础try-catch模式（处理错误并决定是否传播）
```c
PG_TRY();
{
    // 可能抛出ereport(ERROR)的代码（如数据库操作、内存分配等）
    risky_operation();
}
PG_CATCH();
{
    // 错误恢复逻辑：释放资源、记录日志等
    cleanup_resources();
    // 选择1：继续传播错误（让上层处理）
    PG_RE_THROW();
    // 选择2：回滚子事务，终止错误传播（需确保系统状态一致）
    // SubTransactionAbort();
}
PG_END_TRY();
```
作用：捕获PG_TRY块中抛出的ERROR级错误，在PG_CATCH中执行恢复逻辑。
注意：必须通过PG_RE_THROW或事务回滚处理错误，否则可能导致系统状态不一致。

2. try-finally模式（统一清理逻辑）
当正常流程和错误流程需要相同的清理操作（如释放临时内存、关闭连接）时使用：
```c
PG_TRY();
{
    // 可能出错的代码
    start_operation();
}
PG_FINALLY();
{
    // 无论是否出错，必执行的清理（如释放资源）
    cleanup_operation();
}
PG_END_TRY();
```
特点：清理代码执行后，若原代码块有错误，会自动向外传播（无需手动PG_RE_THROW）。

3. 关键限制与注意事项
PG_CATCH与PG_FINALLY互斥：同一PG_TRY块中只能二选一，不能同时使用。
无法捕获FATAL级错误：ereport(FATAL)会直接终止进程（通过proc_exit()），不会进入PG_CATCH或PG_FINALLY。因此，非进程本地资源（如跨进程共享内存）的清理不应依赖这些宏。
错误恢复代码应避免新错误：虽然系统支持传播恢复代码中抛出的新ERROR，但嵌套层数有限，建议恢复逻辑保持简单。
局部变量需volatile修饰：若PG_TRY块中修改的变量在PG_CATCH中使用，必须声明为volatile（防止编译器优化导致数据不一致）。
嵌套使用时的变量后缀：嵌套PG_TRY时，可通过添加后缀（如PG_TRY(outer)、PG_TRY(inner)）避免变量名冲突（适配-Wshadow编译警告）。