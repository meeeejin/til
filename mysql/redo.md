# Redo

I learned from this [page](https://www.percona.com/live/mysql-conference-2014/sites/default/files/slides/InnoDB%20-%20A%20journey%20to%20the%20core%20II.pdf).

## Redo log

The redo log is a InnoDB log and a Write-Ahead Log (WAL). Two or more redo logs (*ib_logfile0, ib_logfile1, ...*) are pre-allocated and used in a circular fashion. And they are used to reconstruct changes if necessary ([crash recovery](crash-recovery.md)).

## Redo logging in practice

- Every data change is written to the redo log buffer as the table data is modified.
- The redo log buffer is written to disk (*but not synced*) when each page modification is complete.
- The redo log is synced to disk up to a the page's modification LSN before a page can be flushed to disk.
- Depending on `innodb_flush_log_at_trx_commit`, the log may be synced to disk at transaction commit. 
