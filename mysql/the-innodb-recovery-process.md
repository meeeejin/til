# The InnoDB recovery process

InnoDB crash recovery consists of several steps:

## Applying the redo log

Applying the redo log is the first step and is performed during initialization. If all changes were flushed from the buffer pool to the tablespaces at shutdown or crash, it can be skipped.

## Rolling back incomplete transactions

Rolling back any transactions that were active at the time of crash or [fast shutdown](mysql/types-of-shutdown.md). It is special to crash recovery. And the rollback is performed by a background thread, executed in parallel with transactions from new connections. 

## Change buffer merge

Applying changes from the change buffer to leaf pages of secondary indexes, as the index pages are read to the buffer pool.

## Purge

Deleting delete-marked records that are no longer visible for any active transaction.