# TIL (Today I Learned)

TIL is a collection of the things that I learned today. The contents can be anything.

---

### Categories

* [Arcus](#arcus)
* [Benchmark](#benchmark)
* [DBMS](#dbms)
* [Docker](#docker)
* [Git](#git)
* [Go](#go)
* [IPL](#ipl)
* [Linux](#linux)
* [MariaDB](#mariadb)
* [MySQL](#mysql)
* [NVRAM](#nvram)
* [Oracle](#oracle)
* [Percona](#percona)
* [PostgreSQL](#postgresql)
* [PMDK](#pmdk)
* [Predix](#predix)
* [RocksDB](#rocksdb)
* [SSD](#ssd)
* [Vim](#vim)
* [Zero](#zero)

---

### Arcus

- [How to install Arcus on Ubuntu 16.04](arcus/how-to-install-arcus-on-ubuntu.md)

### Benchmark

- [tpcc-mysql: Quick start guide](benchmark/tpcc-mysql.md)
- [How to install SysBench 0.5 on Ubuntu](benchmark/how-to-install-sysbench-0.5-on-ubuntu.md)
- [TPC-E vs. TPC-C](benchmark/tpc-e-versus-tpc-c.md) :kr:
- [How to install BenchmarkSQL for Oracle](benchmark/how-to-install-benchmarksql-for-oracle.md)
- [How to install BenchmarkSQL for PostgreSQL](benchmark/how-to-install-benchmarksql-for-postgresql.md)
- [How to install TPC-E on Ubuntu (for MySQL)](benchmark/how-to-install-tpce-on-ubuntu.md)
- [How to install LinkBench on Ubuntu](benchmark/how-to-install-linkbench-on-ubuntu.md)
- [How to install TPC-H for Oracle](benchmark/how-to-install-tpch-for-oracle.md)

### DBMS

- [A comparison of buffer management algorithms between three DBMSs](dbms/a-comparison-of-buffer-management-algorithms.md) :kr:
- [Database Meets AI: A Survey](dbms/db-meets-ai.md) :kr:

### Docker

- [How to install MySQL and use a Docker volume on Docker](docker/mysql-with-docker-volume.md)
- [How to use lb-mysql on IOD SSD](docker/lb-mysql-on-iod.md)
- [Container networking](docker/container-networking.md) :kr:
- [How to use Docker Hub](docker/how-to-use-docker-hub.md) :kr:

### Git

- [Rename folders with git](git/rename-folders-with-git.md)
- [Modify previous commit messages](git/modify-previous-commit-messages.md)
- [Remove a file added in an older commit](git/Remove-a-file-added-in-an-older-commit.md)

### Go


### IPL

- [Basic concept](ipl/basic-concept.md) :kr:
- [Merge operation](ipl/merge-operation.md)

### Linux

- [Direct I/O](linux/direct-io.md)
- [Synchronous I/O](linux/synchronous-io.md)
- [Extracts certain rows from a file](linux/extracts-certain-rows-from-a-file.md)
- [AWK command](linux/awk-command.md)
- [How to control cores via bash command](linux/how-to-control-cores-via-bash-command.md)
- [Copy and paste in tmux](linux/copy-and-paste-in-tmux.md)
- [Extract multiple lines from a file](linux/extract-multiple-lines-from-a-file.md)
- [How to add a kernel boot parameter](linux/how-to-add-a-kernel-boot-parameter.md)
- [perf and FlameGraph](linux/perf-and-flamegraph.md) :kr:
- [How to create software RAID 0 with mdadm](linux/how-to-create-software-raid0-with-mdadm.md)
- [How to upgrade NVMe SSD firmware on Linux](linux/how-to-upgrade-nvme-ssd-firmware-on-linux.md)
- [How to replace the string `\n` with an actual newline](linux/replace-the-string-n-with-an-actual-newline.md)

### MariaDB

- [MariaDB S3 engine](mariadb/mariadb-s3-engine.md)

### MySQL

- [Build and install the source code (5.6)](mysql/build-and-install-the-source-code-5.6.md)
- [Run MySQL](mysql/run-mysql.md)
- [The InnoDB recovery process](mysql/the-innodb-recovery-process.md)
- [Types of shutdown](mysql/types-of-shutdown.md)
- [Crash recovery](mysql/crash-recovery.md)
- [Redo](mysql/redo.md)
- [Undo](mysql/undo.md)
- [What happens when you UPDATE](mysql/what-happens-when-you-update.md)
- [Change buffer](mysql/change-buffer.md)
- [MySQL connection error](mysql/mysql-connection-error.md)
- [Log flush at commit](mysql/log-flush-at-commit.md)
- [innodb_flush_method](mysql/innodb-flush-method.md)
- [Update root password in MySQL 5.7](mysql/update-root-password-in-mysql-5.7.md)
- [Build and install the source code (5.7)](mysql/build-and-install-the-source-code-5.7.md)
- [Build and install the source code (8.0)](mysql/build-and-install-the-source-code-8.0.md)
- [Useful MySQL performance tuning tips](mysql/useful-mysql-performance-tuning-tips.md)
- [Curated contents about flushing mechanisms](mysql/curated-contents-about-flushing-mechanisms.md)
- [Monitoring InnoDB mutex and lock waits](mysql/monitoring-innodb-mutex-and-lock-waits.md)
- [How InnoDB performs a checkpoint](mysql/how-innodb-performs-a-checkpoint.md) :kr:
- [InnoDB adaptive flushing](mysql/adaptive-flushing.md) :kr:
- [innodb_ruby](mysql/innodb-ruby.md) :kr:
- [Punch hole](mysql/punch-hole.md) :kr:
- [Mid-point insertion strategy in MySQL/InnoDB](mysql/midpoint-insertion.md)
- [All about InnoDB flushing](mysql/all-abount-innodb-flushing.md) :kr:
- [fallocate() and ftruncate() in MySQL/InnoDB](mysql/falloc-and-ftrunc-in-innodb.md)
- [Code related to log files](mysql/code-related-to-log-files.md)
- [InnoDB page splits](mysql/page-split.md)
- [MySQL/InnoDB page checksum](mysql/innodb-page-checksum.md)
- [Doublewrite buffer](mysql/doublewrite-buffer.md)

### NVRAM

- [Types of NVDIMM](nvram/types-of-nvdimm.md) :kr:

### Oracle

- [Install Oracle 12c on Ubuntu](oracle/install-oracle-12c-on-ubuntu.md)
- [How to clean UNDO tablespaces](oracle/how-to-clean-undo-tablespaces.md)
- [Solution to ORA-03113](oracle/solution-to-ora-03113.md)
- [How to use Statspack](oracle/how-to-use-statspack.md)
- [How to resize the online redo log files](oracle/how-to-resize-the-online-redo-logfiles.md)
- [Free buffer waits in Oracle](oracle/free-buffer-waits-in-oracle.md)
- [Recovery after losing UNDO tablespace](oracle/recovery-after-losing-undo-tablespace.md)
- [How to resize the SGA](oracle/how-to-resize-the-sga.md)
- [Understanding Oracle wait events](oracle/understanding-oracle-wait-events.md) :kr:
- [Oracle encoding error](oracle/oracle-encoding-error.md) :kr:
- [How to add a datafile to a tablespace](oracle/how-to-add-a-datafile-to-a-tablespace.md)
- [How to relocate files in Oracle](oracle/how-to-relocate-files-in-oracle.md)

### Percona

- [Multi-threaded LRU flushing](percona/multi-threaded-LRU-flushing.md)
- [innodb_empty_free_list_algorithm](percona/innodb_empty_free_list_algorithm.md)
- [Build and install the source code (5.6)](percona/build-and-install-the-source-code-5.6.md)
- [Installing Percona Monitoring and Management (PMM) on Ubuntu](percona/install-pmm-on-ubuntu.md)

### PMDK

- [Building PMDK on Ubuntu](pmdk/building-pmdk-on-ubuntu.md)

### PostgreSQL

- [PostgreSQL installation from source code](postgresql/installation-from-source-code.md)
- [Buffer management of PostgreSQL](postgresql/pgsql-buffer-mgmt.md) :kr:
- [PostgreSQL hit ratio](postgresql/pgsql-hit-ratio.md)

### Predix

- [Predix overview](predix/predix-overview.md) :kr:

### RocksDB

- [How to use db_bench](rocksdb/how-to-use-db_bench.md)
- [Write stalls](rocksdb/write-stalls.md) :kr:

### SSD

- [Basic operations](ssd/basic-operations.md)
- [FTL (Flash Translation Layer)](ssd/ftl.md)
- [Device initialization](ssd/device-initialization.md)
- [Logical block mapping](ssd/logical-block-mapping.md)
- [Garbage collection](ssd/garbage-collection.md)
- [TRIM](ssd/trim.md)
- [Over-provisioning](ssd/over-provisioning.md)
- [Writes](ssd/writes.md)
- [Reads](ssd/reads.md)
- [Concurrent reads and writes](ssd/concurrent-reads-and-writes.md)

### Vim

- [Search and replace](vim/search-and-replace.md)

### Zero

- [Buffer Manager](zero/buffer-manager.md) :kr:

## About

I borrowed this idea from [thoughtbot/til](https://github.com/thoughtbot/til) and [jbranchaud/til](https://github.com/jbranchaud/til).
