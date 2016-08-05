# Change buffer (Insert buffer)

The change buffer is a special data structure that caches changes to secondary index pages when affected pages are not in the buffer pool. The buffered changes, which may result from DML opearations, are merged later when the pages are loaded into the buffer pool by other read opertaions.

## The high-level idea

Modifications to secondary indexes usually happen in relatively random order, potentially causing a lot of random disk I/O operations. Instead of performing these random I/O operaions necessary to read secondary index pages, modifications are cached in the **change buffer**.

## Features

- A special table/index stored in the system tablespace.
- The number of the root page of the change buffer index is 4.
- Whenever a secondary page index page is read, the change buffer bitmap is checked for pending merges. Otherwise, the change buffer index is randomly traversed for merges.
- The change buffer occupies part of the InnoDB buffer pool.

## Merge

- When affected pages are read into the buffer pool by other operations
- Periodic purge operations that run when the system is mostly idle
- During a slow shutdown

Change buffer merging may take several hours when there are numerous secondary indexes to update and many affected rows. During this time, disk I/O is increased, which can cause a significant slowdown for disk-bound queries. Change buffer merging may also continue to occur after a transaction is committed. In fact, change buffer merging may continue to occur after a server shutdown and restart
