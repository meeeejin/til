# Log Flush at Commit

`innodb_flush_log_at_trx_commit` variable controls the balance between strict **ACID** compliance for commit operations and higher **performance**. [Detail](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_flush_log_at_trx_commit)

## With a value of 0:
- The contents of the *InnoDB log buffer* are written to the log file approximately **once per second** and the *log file* is flushed to disk.
- Every 1 second

## With a value of 1(Default):
- The contents of the *InnoDB log buffer* are written out to the log file **at each transaction commit** and the *log file* is flushed to disk
- fsync on commit

## With a value of 2:
- The contents of the *InnoDB log buffer* are written to the log file **after each transaction commit** and the *log file* is flushed to disk approximately **once per second**
- Writes on commit
