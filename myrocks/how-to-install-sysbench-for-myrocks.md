# How to install SysBench 1.0 for MyRocks

## Prerequisite

```bash
$ sudo apt-get install make automake libtool pkg-config libaio-dev libmysqlclient-dev libssl-dev
```

## Build and install

1. First, download the desired Sysbench version from the [release list](https://github.com/akopytov/sysbench/releases).

2. Extract the tar.gz file.

3. Then, Build and install:

```bash
$ ./autogen.sh
$ ./configure --with-mysql-includes=/path/to/mysql-5.6/include --with-mysql-libs=/path/to/mysql-5.6/lib
$ make -j8
$ make install
```

## Usage example

For more details, please refer the [Sysbench Repo](https://github.com/akopytov/sysbench#usage).

### OLTP Insert

1. Before running the benchmark, you should create a database for SysBench test. For example:

```bash
$ mysql -uroot

mysql> create database sbtest;
mysql> quit
```

2. Then, `prepare` the test. At the `prepare` stage, SysBench performs preparative actions for those tests which need them. (e.g. creating the necessary files on disk for the fileio test, or filling the test database for the oltp test):

> Note that `--threads=1`. Values > 1 resulted in an error.

```bash
$ sysbench oltp_insert \
    --mysql-host=localhost --mysql-db=sbtest --mysql-user=root \
    --mysql-socket=/tmp/mysql.sock --table-size=10000 \
    --time=600 --tables=10 --report-interval=10 \
    --mysql-port=3306 --threads=1 --mysql-storage-engine=rocksdb prepare

sysbench 1.0.16 (using bundled LuaJIT 2.1.0-beta2)

Creating table 'sbtest1'...
Inserting 10000 records into 'sbtest1'
Creating a secondary index on 'sbtest1'...
Creating table 'sbtest2'...
Inserting 10000 records into 'sbtest2'
Creating a secondary index on 'sbtest2'...
...
Creating table 'sbtest10'...Inserting 10000 records into 'sbtest10'
Creating a secondary index on 'sbtest10'...
```

3. Then, Run:

```bash
$ sysbench oltp_insert \
    --mysql-host=localhost --mysql-db=sbtest --mysql-user=root \
    --mysql-socket=/tmp/mysql.sock --table-size=10000 \
    --time=600 --tables=10 --report-interval=10 \
    --mysql-port=3306 --threads=16 --mysql-storage-engine=rocksdb prepare

sysbench 1.0.16 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 16
Report intermediate results every 10 second(s)
Initializing random number generator from current time

Initializing worker threads...

Threads started!

[ 10s ] thds: 16 tps: 3112.24 qps: 3112.24 (r/w/o: 0.00/3112.24/0.00) lat (ms,95%): 9.22 er
r/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 16 tps: 3144.45 qps: 3144.45 (r/w/o: 0.00/3144.45/0.00) lat (ms,95%): 9.06 er
r/s: 0.00 reconn/s: 0.00
...
```