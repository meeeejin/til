# Build and install the source code (5.6)

## Pre-requisites

```bash
$ sudo apt-get update
$ sudo apt-get -y install g++ cmake libbz2-dev libaio-dev bison \
zlib1g-dev libsnappy-dev libboost-all-dev libgflags-dev libzstd1 \
libzstd1-dev libreadline6-dev libncurses5-dev libssl-dev \
libcap-dev libcap2 liblz4-dev gdb git
```

Also, install Zstandard:

```bash
$ git clone https://github.com/facebook/zstd
$ make -j8
$ make install
```

## Build and install

> Modify each path below (e.g., `/home/mijin/mysql-5.6`) according to your environment:

1. Setup the git repository:

```bash
$ git clone https://github.com/facebook/mysql-5.6
$ cd mysql-5.6
$ git submodule init
$ git submodule update
```

2. Run `cmake`:

```bash
$ cmake . -DCMAKE_BUILD_TYPE=RelWithDebInfo -DWITH_SSL=system -DWITH_ZLIB=bundled -DMY SQL_MAINTAINER_MODE=0 -DENABLED_LOCAL_INFILE=1 -DENABLE_DTRACE=0 -DCMAKE_CXX_FLAGS="-march=native" -DCMAKE_INSTALL_PREFIX=/home/mijin/mysql-5.6
```

3. Compile and install:

```bash
$ make -j8 install
```

4. Create a configuration file for MyRocks (e.g., `myrocks.cnf`). My configuration file is as follows:

```bash
$ vi myrocks.cnf

[mysqld]
rocksdb
default-storage-engine=rocksdb
skip-innodb
default-tmp-storage-engine=MyISAM
collation-server=latin1_bin
log-bin
binlog-format=ROW

socket=/tmp/mysql.sock
port=3306
datadir=/home/mijin/test_data

rocksdb_max_open_files=-1
rocksdb_max_background_jobs=8
rocksdb_max_total_wal_size=4G
rocksdb_block_size=16384
rocksdb_table_cache_numshardbits=6
rocksdb_block_cache_size=10G

# rate limiter
rocksdb_bytes_per_sync=4194304
rocksdb_wal_bytes_per_sync=4194304
rocksdb_rate_limiter_bytes_per_sec=104857600

# triggering compaction if there are many sequential deletes
rocksdb_compaction_sequential_deletes_count_sd=1
rocksdb_compaction_sequential_deletes=199999
rocksdb_compaction_sequential_deletes_window=200000

rocksdb_default_cf_options=write_buffer_size=128m;target_file_size_base=32m;max_bytes_for_level_base=512m;level0_file_num_compaction_trigger=4;level0_slowdown_writes_trigger=10;level0_stop_writes_trigger=15;max_write_buffer_number=4;compression_per_level=kLZ4Compression;bottommost_compression=kZSTD;compression_opts=-14:1:0;block_based_table_factory={cache_index_and_filter_blocks=1;filter_policy=bloomfilter:10:false;whole_key_filtering=1};level_compaction_dynamic_level_bytes=true;optimize_filters_for_hits=true;compaction_pri=kMinOverlappingRatio

rocksdb_wal_dir=/home/mijin/test_log

rocksdb_use_direct_io_for_flush_and_compaction=ON
rocksdb_use_direct_reads=ON
```

For more details, refer [my.cnf settings](https://github.com/facebook/mysql-5.6/wiki/my.cnf-tuning) of the MyRocks Wiki.

5. Initialize the database:

```bash
$ sudo chmod +x scripts/mysql_install_db
$ ./scripts/mysql_install_db --defaults-file=/home/mijin/myrocks.cnf
```

6. Start the server:

```bash
$ ./bin/mysqld_safe --defaults-file=/home/mijin/myrocks.cnf
```

7. Shutdown the server:

```bash
$ ./bin/mysqldadmin -uroot shutdown
```