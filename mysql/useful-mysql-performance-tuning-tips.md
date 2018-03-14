# Useful MySQL performance tuning tips

[Detail](https://www.percona.com/live/mysql-conference-2014/sites/default/files/slides/PLMCE2014-Innodb-Architecture-And-Performance-Optimization.pdf).

## Separate Undo Tablespace

Store undo tablespace in separate set of files

- `innodb_undo_directory`
- `innodb_undo_tablespaces`
- `innodb_undo_logs`

## Page Checksum

Page checksum is checked when page is read to buffer pool, and updated when page is flushed to disk.
It can be significant overhead when using very fast storage. It can be disabled by:

`innodb_checksums=0`

Or we can use faster checksums for fast storage:

`innodb_checksum_algorithm=crc32`

> The remains are still being arranged..
