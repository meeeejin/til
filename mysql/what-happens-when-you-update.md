# What happens when you UPDATE

I learned from this [page](https://www.percona.com/live/mysql-conference-2014/sites/default/files/slides/InnoDB%20-%20A%20journey%20to%20the%20core%20II.pdf).

## Transaction start

- A transaction ID (TRX_ID) is assigned and may be written to the highest transaction ID field in the TRX_SYS page.
- A read view is created based on the assigned TRX_ID.

## Record modification

- Undo log space is allocated.
- Previous values from record are copied to undo log.
- Record of undo log modifications are written to redo log.
- Page is modified in buffer pool; rollback pointer is pointed to previous version written in undo log.
- Record of page modifications are written to redo log.
- Page is marked as "dirty" (needs to be flushed to disk).

## Transaction commit

- Undo log page state is set to "purge" (meaning it can be cleaned up when it's no longer needed).
- Record of undo log modifications are written to redo log.
- Redo log buffer is flushed to disk (depending on `innodb_flush_log_at_trx_commit`).

## Background flush

The background flush thread runs continuously to:

- Find the oldest "dirty" pages and add them to a flush batch.
- Ensure the redo log is written and flushed up to the newest LSN in the batch.
- Write a block of up to 128 pages to the double write buffer.
- Write each page to its final destination in a tablespace file.

## Background purge

The background purge thread runs continuously to:

- Find the oldest undo logs that are no longer needed in each rollback segments.
- Actually remove any delete-marked records from indexes.
- Free the undo log pages.
- Prune the history lists.
