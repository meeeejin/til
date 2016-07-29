# Crash Recovery

I learned from this [page](https://www.percona.com/live/mysql-conference-2014/sites/default/files/slides/InnoDB%20-%20A%20journey%20to%20the%20core%20II.pdf).

## When is crash recovery used?

- When you crash and restart
- After restoring from backups
- Restarting after fast shutdown

## Check for unclean shutdown

- Redo logs and system tablespace are opened.
- Checkpoints are read to find highest checkpoint number.
- Redo logs are scanned starting from latest checkpoint.
- Crash recovery is initiated if any redo records are found.

## Create file-per-table space ID mapping

- Open all **.ibd** files in directories within the data directory.
- Read their space ID from the page 0 (FSP_HDR) header.
- Populate the space ID to table name mapping.

* In Face, if the most recent copy of the page 0 is stored in SSD cache, it should be read in from the SSD cache, instead of the storage.

## Torn page recovery

All 128 pages from the doublewrite buffer are examined:

- Each **"target"** page from the tablespace is read.
- If the header and trailer LSN do not match or the page checksum is invalid, the page is restored from the doublewrite buffer.
- If the doublewrite buffer version of the page is also corrupt, the server will assert(crash).

* FaCE does not use doublewrite buffer. So this sequences should be replaced with SSD cache (that is, instead of restoring pages from doublewrite buffer, restores pages from the SSD cache).

## Rollback of uncommitted transactions

- Transaction system (**rollback segments**) is initialized.
- Transactions corresponding to **"active"** undo logs are resurrected.
- Redo log records are applied since the latest checkpoint.
- Active transactions are rolled back (using **undo logs**).
