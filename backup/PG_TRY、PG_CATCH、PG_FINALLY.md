> [!CAUTION]
> 以下内容来自AI总结 如有错误 需自行甄别 此处用作记录

# 基本用途与核心宏
这些宏用于捕获代码中可能通过ereport(ERROR)抛出的错误，并执行错误恢复逻辑。核心宏包括：

PG_TRY()：标记 “可能抛出错误的代码块” 的开始。

PG_CATCH()：标记 “错误恢复代码块” 的开始（仅当PG_TRY中的代码抛出ERROR时执行）。

PG_FINALLY()：标记 “无论是否出错都必须执行的清理代码块”（替代PG_CATCH，适用于正常流程和错误流程需相同清理的场景）。

PG_END_TRY()：标记整个错误处理块的结束。

# 使用方式
1. 基础try-catch模式（处理错误并决定是否传播）
```c
PG_TRY();
{
    // 可能抛出ereport(ERROR)的代码
    ...
}
PG_CATCH();
{
    // 错误恢复逻辑（如释放资源、记录日志等）
    ...
    // 处理后可选择：
    // - 用PG_RE_THROW()将错误继续向外传播
    // - 执行事务/子事务回滚，避免系统不一致
}
PG_END_TRY();
```
错误恢复代码必须处理错误（传播或回滚），否则可能导致系统状态不一致。

2. try-finally模式（统一清理逻辑）
当 “正常执行结束” 和 “错误发生时” 需要执行相同的清理操作（如释放临时内存、关闭连接），可使用PG_FINALLY：
```c 
PG_TRY();
{
    // 可能抛出错误的代码
    ...
}
PG_FINALLY();
{
    // 无论是否出错，都会执行的清理代码（如释放资源）
    ...
}
PG_END_TRY();
```
清理代码执行后，若原代码块抛出了错误，错误会自动继续向外传播（无需手动PG_RE_THROW）。

# 限制与注意事项
PG_CATCH与PG_FINALLY互斥，同一个PG_TRY()/PG_END_TRY()块中，不能同时使用PG_CATCH和PG_FINALLY，只能二选一。

ereport(FATAL)无法捕获，PG_TRY仅能捕获ereport(ERROR)级别的错误；ereport(FATAL)会直接通过proc_exit()终止进程，不会进入PG_CATCH或PG_FINALLY块。因此，不要在错误恢复代码中放置 “非进程本地资源” 的清理逻辑（如跨进程共享资源），除非已考虑FATAL错误的处理（可参考storage/ipc.h中的PG_ENSURE_ERROR_CLEANUP宏）。

错误恢复代码应避免新错误，虽然系统会 propagate（传播）恢复代码中抛出的新ereport(ERROR)，但嵌套层数有限制。因此，错误恢复代码应尽量简单，避免在处理错误时再抛出新错误（至少在弹出错误栈前应如此）。

局部变量需用volatile修饰，若函数中的局部变量在PG_TRY块中被修改，且在PG_CATCH块中被使用，必须声明为volatile（符合 POSIX 标准）。
原因：编译器可能会优化掉未标记volatile的变量读写，导致PG_CATCH中读取到错误值。注意：gcc 的-Wclobbered警告对此类问题检测效果较差，需手动确保。

嵌套使用时的变量后缀，宏内部会声明变量，若需要嵌套使用PG_TRY（如在PG_CATCH中再用PG_TRY），可能导致变量名冲突（编译器-Wshadow警告）。解决：可给宏传递可选后缀（变量名允许的字符），确保嵌套时变量名唯一。要求：同一PG_TRY块的所有宏（PG_TRY/PG_CATCH/PG_END_TRY）必须使用相同的后缀。
示例：
```c
PG_TRY(outer);  // 后缀"outer"
{
    PG_TRY(inner);  // 后缀"inner"，避免与外层冲突
    { ... }
    PG_CATCH(inner);
    { ... }
    PG_END_TRY(inner);
}
PG_CATCH(outer);
{ ... }
PG_END_TRY(outer);
```

# 总结
这些宏是 PostgreSQL 内部实现 “错误捕获与恢复” 的基础机制，类似于高级语言中的try-catch-finally，但针对 PostgreSQL 的错误模型（ereport级别）做了特殊设计。使用时需特别注意错误级别（ERROR vs FATAL）、变量修饰和嵌套冲突，以确保代码的正确性和系统稳定性。