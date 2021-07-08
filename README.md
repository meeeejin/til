# TIL (Today I Learned)

TIL is a collection of the things that I learned today. The contents can be anything.

---

### Categories

* [Arcus](#arcus)
* [Benchmark](#benchmark)
* [CPU](#cpu)
* [DBMS](#dbms)
* [Docker](#docker)
* [Git](#git)
* [Go](#go)
* [GPU](#gpu)
* [Hadoop](#hadoop)
* [IPL](#ipl)
* [LightningDB](#lightningdb)
* [Linux](#linux)
* [MariaDB](#mariadb)
* [MyRocks](#myrocks)
* [MySQL](#mysql)
* [NVRAM](#nvram)
* [Oracle](#oracle)
* [Percona](#percona)
* [PostgreSQL](#postgresql)
* [PMDK](#pmdk)
* [Predix](#predix)
* [RocksDB](#rocksdb)
* [Rockset](#rockset)
* [Spark](#spark)
* [SSD](#ssd)
* [TensorFlow](#tensorflow)
* [Vim](#vim)
* [Zeppelin](#zeppelin)
* [Zero](#zero)

---

### Arcus

- [How to install Arcus on Ubuntu 16.04](arcus/how-to-install-arcus-on-ubuntu.md)
- [How to install Arcus-memcached on Ubuntu 16.04](arcus/how-to-install-arcus-memcached-on-ubuntu.md)
- [Arcus persistence](arcus/arcus-persistence.md) :kr:
- [Separation of cmdlogbuf module into two modules](arcus/log-buffer-and-log-file.md) :kr:

### Benchmark

- [tpcc-mysql: Quick start guide](benchmark/tpcc-mysql.md)
- [How to install SysBench 0.5 on Ubuntu](benchmark/how-to-install-sysbench-0.5-on-ubuntu.md)
- [TPC-E vs. TPC-C](benchmark/tpc-e-versus-tpc-c.md) :kr:
- [How to install BenchmarkSQL for Oracle](benchmark/how-to-install-benchmarksql-for-oracle.md)
- [How to install BenchmarkSQL for PostgreSQL](benchmark/how-to-install-benchmarksql-for-postgresql.md)
- [How to install TPC-E on Ubuntu (for MySQL)](benchmark/how-to-install-tpce-on-ubuntu.md)
- [How to install LinkBench on Ubuntu](benchmark/how-to-install-linkbench-on-ubuntu.md)
- [How to install TPC-H for Oracle](benchmark/how-to-install-tpch-for-oracle.md)
- [Summary of LinkBench for MySQL](benchmark/summary-of-linkbench-for-mysql.md)
- [How to install YCSB for RocksDB](benchmark/how-to-install-ycsb-for-rocksdb.md)

### CPU

- [AMD uProf](cpu/amd-uprof.md) :kr:

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
- [Clone a single branch in Git](git/clone-a-single-branch.md)

### Go

### GPU

- [How to install CUDA on Ubuntu](gpu/cuda-installation.md)
- [Nvidia Nsight Systems](gpu/nvidia-nsight-systems.md) :kr:

### Hadoop

- [How to install Hadoop 3 (with Yarn) on Ubuntu](hadoop/hadoop-installation.md)
- [Hadoop commands](hadoop/hadoop-command.md)

### IPL

- [Basic concept](ipl/basic-concept.md) :kr:
- [Merge operation](ipl/merge-operation.md)

### LightningDB

- [How to deploy LightningDB](lightningdb/deploy-ltdb.md)
- [How to monitor the status of LTDB clusters](lightningdb/monitor-status.md)
- [How to reset a LTDB cluster](lightningdb/reset-cluster.md)

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
- [Redirect stderr to stdout](linux/redirect-stderr-to-stdout.md)
- [How to use ftrace](linux/ftrace.md)

### MariaDB

- [MariaDB S3 engine](mariadb/mariadb-s3-engine.md) :kr:

### MyRocks

- [Build and install the source code (5.6)](myrocks/build-and-install-the-source-code-5.6.md)
- [How to install SysBench 1.0 for MyRocks](myrocks/how-to-install-sysbench-for-myrocks.md)
- [How to install tpcc-mysql for MyRocks](myrocks/how-to-install-tpcc-for-myrocks.md)
- [How to monitor MyRocks](myrocks/monitoring.md)

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
- [Understanding InnoDB Lock Stats](mysql/innodb-lock-stats.md) :kr:
- [How to get the LBA of DWB](mysql/how-to-get-lba-of-dwb.md)
- [InnoDB Flushing and Checkpoints](https://www.slideshare.net/meeeejin/innodb-flushing-and-checkpoints) :link:
- [Secondary Index Search in InnoDB](https://www.slideshare.net/meeeejin/secondary-index-search-in-innodb) :link:
- [MySQL Space Management](https://www.slideshare.net/meeeejin/mysql-space-management) :link:
- [MySQL Buffer Management](https://www.slideshare.net/meeeejin/mysql-buffer-management) :link:
- [Temporary Tablespaces](mysql/temporary-tablespace.md) :kr:

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
- [RocksDB detail](https://www.slideshare.net/meeeejin/rocksdb-detail) :link:
- [RocksDB compaction](https://www.slideshare.net/meeeejin/rocksdb-compaction) :link:

### Rockset

- [Remote Compactions in RocksDB-Cloud](rockset/remote-compaction.md) :kr:

### Spark

- [How to install Spark 3 on Ubuntu](spark/spark-installation.md)
- [How to install Spark RAPIDS](spark/spark-rapids-installation.md)
- [How to run Spark Executor via command line](spark/run-executor-cli.md)

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
- [How to get SMART information for NVMe](ssd/nvme-waf.md)

### TensorFlow

- [How to use Slim](tensorflow/how-to-use-slim.md)
- [How to install TensorFlow GPU on Ubuntu](tensorflow/how-to-install-tf-gpu.md)

### Vim

- [Search and replace](vim/search-and-replace.md)

### Zeppelin

- [How to install Zeppelin](zeppelin/zeppelin-installation.md)

### Zero

- [Buffer Manager](zero/buffer-manager.md) :kr:

## About

I borrowed this idea from [thoughtbot/til](https://github.com/thoughtbot/til) and [jbranchaud/til](https://github.com/jbranchaud/til).
