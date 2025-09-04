在事务块中调用存在异常的函数

```sql
CREATE TABLE tmp(id int);

CREATE OR REPLACE FUNCTION demo_plpgsql_subxact()
RETURNS void AS $$
BEGIN                        
    INSERT INTO tmp VALUES (-2);
    INSERT INTO tmp VALUES (1/0);  -- error
EXCEPTION
    WHEN division_by_zero THEN
        RAISE INFO '%', SQLERRM;
        INSERT INTO tmp VALUES (-3);            
END;
$$ LANGUAGE plpgsql;

BEGIN; -- 开启事务
INSERT INTO tmp VALUES (-1);
ELECT demo_plpgsql_subxact(); -- 调用函数
select * from tmp;
COMMIT;

truncate tmp;

-- 等价于
BEGIN; -- 开启事务
INSERT INTO tmp VALUES (-1);
SAVEPOINT exception;  -- 保存点
INSERT INTO tmp VALUES (-2);
INSERT INTO tmp VALUES (1/0);  -- error
ROLLBACK TO SAVEPOINT exception; -- 异常回滚到保存点
INSERT INTO tmp VALUES (-3); 
SELECT * FROM tmp;
COMMIT;
```

运行结果

```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# CREATE TABLE tmp(id int);
CREATE TABLE
postgres=# CREATE OR REPLACE FUNCTION demo_plpgsql_subxact()
postgres-# RETURNS void AS $$
postgres$# BEGIN  
postgres$#     INSERT INTO tmp VALUES (-2);
postgres$#     INSERT INTO tmp VALUES (1/0);  -- error
postgres$# EXCEPTION
postgres$#     WHEN division_by_zero THEN
postgres$#                 RAISE INFO '%', SQLERRM;
postgres$#         INSERT INTO tmp VALUES (-3); 
postgres$# END;
postgres$# $$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=# BEGIN;
BEGIN
postgres=*# INSERT INTO tmp VALUES (-1);
INSERT 0 1
postgres=*# SELECT demo_plpgsql_subxact(); -- 调用函数
INFO:  division by zero
 demo_plpgsql_subxact 
----------------------
 
(1 row)

postgres=*# select * from tmp;
 id 
----
 -1
 -3
(2 rows)

postgres=*# COMMIT;
COMMIT
postgres=# truncate tmp;
TRUNCATE TABLE
postgres=# BEGIN;
BEGIN
postgres=*# INSERT INTO tmp VALUES (-1);
INSERT 0 1
postgres=*# SAVEPOINT exception;
SAVEPOINT
postgres=*# INSERT INTO tmp VALUES (-2);
INSERT 0 1
postgres=*# INSERT INTO tmp VALUES (1/0);  -- error
ERROR:  division by zero
postgres=!# ROLLBACK TO SAVEPOINT exception;
ROLLBACK
postgres=*# INSERT INTO tmp VALUES (-3); 
INSERT 0 1
postgres=*# COMMIT;
COMMIT
postgres=# SELECT * FROM tmp;
 id 
----
 -1
 -3
(2 rows)

postgres=# 
```

在事务块中，调用不存在异常的函数

```sql
TRUNCATE tmp;

CREATE OR REPLACE FUNCTION demo_plpgsql_subxact()
RETURNS void AS $$
BEGIN                        
    INSERT INTO tmp VALUES (-2);
    INSERT INTO tmp VALUES (-3);
EXCEPTION
    WHEN division_by_zero THEN
        RAISE INFO '%', SQLERRM;        
END;
$$ LANGUAGE plpgsql;

BEGIN; -- 开启事务块
INSERT INTO tmp VALUES (-1);
select demo_plpgsql_subxact();
INSERT INTO tmp VALUES (-4);
SELECT * FROM tmp;
COMMIT;


TRUNCATE tmp;
-- 等价于
BEGIN; -- 开启事务块
INSERT INTO tmp VALUES (-1);
SAVEPOINT exception;
INSERT INTO tmp VALUES (-2);
INSERT INTO tmp VALUES (-3);
RELEASE SAVEPOINT exception;
INSERT INTO tmp VALUES (-4);
SELECT * FROM tmp;
COMMIT;
```

运行结果

```sql
postgres@zxm-VMware-Virtual-Platform:~$ psql
psql (16.10)
Type "help" for help.

postgres=# TRUNCATE tmp;
TRUNCATE TABLE
postgres=# CREATE OR REPLACE FUNCTION demo_plpgsql_subxact()
RETURNS void AS $$
BEGIN                        
    INSERT INTO tmp VALUES (-2);
    INSERT INTO tmp VALUES (-3);
EXCEPTION
    WHEN division_by_zero THEN
        RAISE INFO '%', SQLERRM;        
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=# BEGIN; -- 开启事务块
BEGIN
postgres=*# INSERT INTO tmp VALUES (-1);
INSERT 0 1
postgres=*# select demo_plpgsql_subxact();
 demo_plpgsql_subxact 
----------------------
 
(1 row)

postgres=*# INSERT INTO tmp VALUES (-4);
INSERT 0 1
postgres=*# SELECT * FROM tmp;
 id 
----
 -1
 -2
 -3
 -4
(4 rows)

postgres=*# COMMIT;
COMMIT
postgres=# TRUNCATE tmp;
TRUNCATE TABLE
postgres=# -- 等价于
postgres=# BEGIN; -- 开启事务块
BEGIN
postgres=*# INSERT INTO tmp VALUES (-1);
INSERT 0 1
postgres=*# SAVEPOINT exception; -- 开启子事务
SAVEPOINT
postgres=*# INSERT INTO tmp VALUES (-2);
INSERT 0 1
postgres=*# INSERT INTO tmp VALUES (-3);
INSERT 0 1
postgres=*# RELEASE SAVEPOINT exception;
RELEASE
postgres=*# INSERT INTO tmp VALUES (-4);
INSERT 0 1
postgres=*# SELECT * FROM tmp;
 id 
----
 -1
 -2
 -3
 -4
(4 rows)

postgres=*# COMMIT;
COMMIT
postgres=# 
```

没有exception则不会触发子事务的动作，部分`exec_stmt_block`代码片段如下
```c
static int
exec_stmt_block(PLpgSQL_execstate *estate, PLpgSQL_stmt_block *block)
{
	// initialize 

	if (block->exceptions)
	{
		/*
		 * Execute the statements in the block's body inside a sub-transaction
		 */
		MemoryContext oldcontext = CurrentMemoryContext;
		ResourceOwner oldowner = CurrentResourceOwner;
		ExprContext *old_eval_econtext = estate->eval_econtext;
		ErrorData  *save_cur_error = estate->cur_error;
		MemoryContext stmt_mcontext;

		estate->err_text = gettext_noop("during statement block entry");

		/*
		 * We will need a stmt_mcontext to hold the error data if an error
		 * occurs.  It seems best to force it to exist before entering the
		 * subtransaction, so that we reduce the risk of out-of-memory during
		 * error recovery, and because this greatly simplifies restoring the
		 * stmt_mcontext stack to the correct state after an error.  We can
		 * ameliorate the cost of this by allowing the called statements to
		 * use this mcontext too; so we don't push it down here.
		 */
		stmt_mcontext = get_stmt_mcontext(estate);

		BeginInternalSubTransaction(NULL);  // 开启子事务
		/* Want to run statements inside function's memory context */
		MemoryContextSwitchTo(oldcontext);

		PG_TRY();
		{
			/*
			 * We need to run the block's statements with a new eval_econtext
			 * that belongs to the current subtransaction; if we try to use
			 * the outer econtext then ExprContext shutdown callbacks will be
			 * called at the wrong times.
			 */
			plpgsql_create_econtext(estate);

			estate->err_text = NULL;

			/* Run the block's statements */
			rc = exec_stmts(estate, block->body);

			estate->err_text = gettext_noop("during statement block exit");

			/*
			 * If the block ended with RETURN, we may need to copy the return
			 * value out of the subtransaction eval_context.  We can avoid a
			 * physical copy if the value happens to be a R/W expanded object.
			 */
			if (rc == PLPGSQL_RC_RETURN &&
				!estate->retisset &&
				!estate->retisnull)
			{
				int16		resTypLen;
				bool		resTypByVal;

				get_typlenbyval(estate->rettype, &resTypLen, &resTypByVal);
				estate->retval = datumTransfer(estate->retval,
											   resTypByVal, resTypLen);
			}

			/* Commit the inner transaction, return to outer xact context */
			ReleaseCurrentSubTransaction(); // 释放子事务
			MemoryContextSwitchTo(oldcontext);
			CurrentResourceOwner = oldowner;

			/* Assert that the stmt_mcontext stack is unchanged */
			Assert(stmt_mcontext == estate->stmt_mcontext);

			/*
			 * Revert to outer eval_econtext.  (The inner one was
			 * automatically cleaned up during subxact exit.)
			 */
			estate->eval_econtext = old_eval_econtext;
		}
		PG_CATCH();
		{
			ErrorData  *edata;
			ListCell   *e;

			estate->err_text = gettext_noop("during exception cleanup");

			/* Save error info in our stmt_mcontext */
			MemoryContextSwitchTo(stmt_mcontext);
			edata = CopyErrorData();
			FlushErrorState();

			/* Abort the inner transaction */
			RollbackAndReleaseCurrentSubTransaction(); // 发生了异常回滚子事务
			MemoryContextSwitchTo(oldcontext);
			CurrentResourceOwner = oldowner;

			/*
			 * Set up the stmt_mcontext stack as though we had restored our
			 * previous state and then done push_stmt_mcontext().  The push is
			 * needed so that statements in the exception handler won't
			 * clobber the error data that's in our stmt_mcontext.
			 */
			estate->stmt_mcontext_parent = stmt_mcontext;
			estate->stmt_mcontext = NULL;

			/*
			 * Now we can delete any nested stmt_mcontexts that might have
			 * been created as children of ours.  (Note: we do not immediately
			 * release any statement-lifespan data that might have been left
			 * behind in stmt_mcontext itself.  We could attempt that by doing
			 * a MemoryContextReset on it before collecting the error data
			 * above, but it seems too risky to do any significant amount of
			 * work before collecting the error.)
			 */
			MemoryContextDeleteChildren(stmt_mcontext);

			/* Revert to outer eval_econtext */
			estate->eval_econtext = old_eval_econtext;

			/*
			 * Must clean up the econtext too.  However, any tuple table made
			 * in the subxact will have been thrown away by SPI during subxact
			 * abort, so we don't need to (and mustn't try to) free the
			 * eval_tuptable.
			 */
			estate->eval_tuptable = NULL;
			exec_eval_cleanup(estate);

			/* Look for a matching exception handler */
			foreach(e, block->exceptions->exc_list)
			{
				PLpgSQL_exception *exception = (PLpgSQL_exception *) lfirst(e);

				if (exception_matches_conditions(edata, exception->conditions))
				{
					/*
					 * Initialize the magic SQLSTATE and SQLERRM variables for
					 * the exception block; this also frees values from any
					 * prior use of the same exception. We needn't do this
					 * until we have found a matching exception.
					 */
					PLpgSQL_var *state_var;
					PLpgSQL_var *errm_var;

					state_var = (PLpgSQL_var *)
						estate->datums[block->exceptions->sqlstate_varno];
					errm_var = (PLpgSQL_var *)
						estate->datums[block->exceptions->sqlerrm_varno];

					assign_text_var(estate, state_var,
									unpack_sql_state(edata->sqlerrcode));
					assign_text_var(estate, errm_var, edata->message);

					/*
					 * Also set up cur_error so the error data is accessible
					 * inside the handler.
					 */
					estate->cur_error = edata;

					estate->err_text = NULL;

					rc = exec_stmts(estate, exception->action);

					break;
				}
			}

			/*
			 * Restore previous state of cur_error, whether or not we executed
			 * a handler.  This is needed in case an error got thrown from
			 * some inner block's exception handler.
			 */
			estate->cur_error = save_cur_error;

			/* If no match found, re-throw the error */
			if (e == NULL)
				ReThrowError(edata);

			/* Restore stmt_mcontext stack and release the error data */
			pop_stmt_mcontext(estate);
			MemoryContextReset(stmt_mcontext);
		}
		PG_END_TRY();

		Assert(save_cur_error == estate->cur_error);
	}
	else
	{
		/*
		 * Just execute the statements in the block's body
		 */
		estate->err_text = NULL;

		rc = exec_stmts(estate, block->body);
	}
    // ....
``` 