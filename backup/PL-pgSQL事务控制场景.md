# PL/pgSQL事务控制场景
| 执行载体                | 调用上下文                | 操作类型                  | 是否支持 | 示例代码片段                                                                 | 备注                                                                 |
|-------------------------|---------------------------|---------------------------|----------|------------------------------------------------------------------------------|----------------------------------------------------------------------|
| 存储过程 | 独立调用（无外层事务）    | COMMIT/ROLLBACK            | 是       | `CALL transaction_test1(9, 'foo');`（循环中条件提交/回滚）                   | 可实现分段事务，仅留存COMMIT的操作                                    |
| 存储过程            | 独立调用                  | COMMIT AND CHAIN/ROLLBACK AND CHAIN | 是       | `COMMIT AND CHAIN;`（保留隔离级别开启新事务）                                | 事务链会继承原隔离级别，适用于连续事务场景                            |
| 存储过程          | 事务块内（START TRANSACTION后） | COMMIT/ROLLBACK            | 否       | `START TRANSACTION; CALL transaction_test1(9, 'error');`                     | 报错“invalid transaction termination”，事务块内禁用显式事务控制       |
| SECURITY DEFINER的存储过程 | 独立调用                  | COMMIT/ROLLBACK            | 否       | `CREATE PROCEDURE test5b() SECURITY DEFINER AS $$ BEGIN COMMIT; END $$;`      | 特殊权限配置的存储过程禁用事务控制                                   |
| 带SET配置的存储过程     | 独立调用                  | COMMIT/ROLLBACK            | 否       | `CREATE PROCEDURE test5() SET work_mem=555 AS $$ BEGIN COMMIT; END $$;`        | 含参数配置的存储过程禁用事务控制                                     |
| DO块                    | 独立执行                  | COMMIT/ROLLBACK            | 是       | `DO $$ BEGIN FOR i IN 0..9 LOOP INSERT INTO test1 VALUES(i); COMMIT; END LOOP; END $$;` | 功能等同于普通存储过程，支持循环内事务控制                            |
| DO块                    | 事务块内                  | COMMIT/ROLLBACK            | 否       | `START TRANSACTION; DO $$ BEGIN COMMIT; END $$;`                              | 同存储过程，事务块内执行直接报错                                     |
| DO块                    | 只读游标循环（SELECT）    | COMMIT/ROLLBACK            | 是       | `FOR r IN SELECT * FROM test2 LOOP INSERT INTO test1 VALUES(r.x); COMMIT; END LOOP;` | 游标为只读时支持，数据按事务控制结果留存                             |
| DO块                    | 非只读游标循环（UPDATE RETURNING） | ROLLBACK            | 否       | `FOR r IN UPDATE test2 SET x=x*2 RETURNING x LOOP INSERT INTO test1 VALUES(r.x); ROLLBACK; END LOOP;` | 报错“cannot perform transaction commands inside a non-read-only cursor loop” |
| 函数                | 任何上下文                | COMMIT/ROLLBACK            | 否       | `CREATE FUNCTION test2() RETURNS int AS $$ BEGIN COMMIT; RETURN 1; END $$;`   | 函数天生禁用显式事务控制，报错“invalid transaction termination”       |
| 函数                    | 内部调用存储过程（含事务控制） | COMMIT/ROLLBACK            | 否       | `CREATE FUNCTION test3() RETURNS int AS $$ BEGIN CALL transaction_test1(9, 'error'); RETURN 1; END $$;` | 函数调用链中禁用事务控制，报错穿透                                   |
| 任何载体                | 动态执行（EXECUTE）       | COMMIT/ROLLBACK            | 否       | `EXECUTE 'COMMIT';`                                                          | 报错“EXECUTE of transaction commands is not implemented”               |
| 任何载体                | 任何上下文                | SAVEPOINT                  | 否       | `DO $$ BEGIN SAVEPOINT foo; END $$;`                                          | 直接报错“unsupported transaction command in PL/pgSQL”                 |
| 带INOUT参数的存储过程    | 独立调用                  | COMMIT/ROLLBACK            | 是       | `CALL transaction_test10a(10);`（参数x+1返回，不影响事务控制）                | 事务控制仅影响数据，不改变参数计算结果                                |
| DO块/存储过程           | 含EXCEPTION块的内部       | COMMIT/ROLLBACK            | 否       | `BEGIN BEGIN INSERT INTO test1 VALUES(1); COMMIT; EXCEPTION WHEN division_by_zero THEN RAISE; END; END $$;` | 报错“cannot commit while a subtransaction is active”，子事务与显式事务冲突 |
| DO块/存储过程           | EXCEPTION块内部           | COMMIT/ROLLBACK            | 是       | `EXCEPTION WHEN division_by_zero THEN INSERT INTO test1 VALUES(i, 'exception'); COMMIT;` | 无隐式子事务时可生效，按条件留存数据                                 |

# 存储过程
### 存储过程可进行事务控制的场景（核心条件：独立 + 无特殊限制）
独立调用（无外层事务，未用START TRANSACTION包裹）；
自身无特殊配置（不含SECURITY DEFINER、无SET参数配置）；
支持操作：COMMIT/ROLLBACK、COMMIT AND CHAIN/ROLLBACK AND CHAIN（事务链）；
兼容场景：只读游标循环（如SELECT结果循环）中执行事务控制。

### 存储过程不可进行事务控制的场景（触发报错）
调用上下文限制：
事务块内调用（START TRANSACTION后执行CALL）；
被函数调用（函数本身禁用事务控制，调用链穿透报错）。

自身配置限制：
带SECURITY DEFINER权限；
含SET参数配置（如SET work_mem=555）。

操作 / 场景限制：
动态执行事务语句（EXECUTE 'COMMIT'）；
非只读游标循环（如UPDATE ... RETURNING）中执行事务控制；
含EXCEPTION的子块内执行COMMIT/ROLLBACK（子事务冲突）。

# 匿名块
### DO 块可进行事务控制的场景
独立执行（无外层START TRANSACTION事务块）；
支持COMMIT/ROLLBACK、COMMIT AND CHAIN/ROLLBACK AND CHAIN；
只读游标循环（如SELECT结果循环）中执行事务控制。

### DO 块不可进行事务控制的场景
事务块内执行（START TRANSACTION后调用）；
动态执行事务语句（EXECUTE 'COMMIT'等）；
非只读游标循环（如UPDATE ... RETURNING）中执行事务控制；
含EXCEPTION的子块内执行COMMIT/ROLLBACK（子事务冲突）。