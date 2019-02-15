# How to install SysBench 0.5 on Ubuntu

# Prerequisite

```bash
sudo apt-get install automake
sudo apt-get install libtool
sudo apt-get install libmysqlclient-dev
sudo apt-get install libssl1.0.0 libssl-dev
```

# Installation

1. Downloads SysBench 0.5(i.e. sysbench_0.5.orig.tar.gz) from [Percona repositories](http://repo.percona.com/apt/pool/main/s/sysbench/).

2. Untar the sysbench tar file.

```bash
tar -xvf sysbench_0.5.orig.tar.gz
```

3. Go to the SysBench directory and run following commands.

```bash
./autogen.sh
./configure
make
make install
```

If you have MySQL headers and libraries in non-standard locations (and no mysql_config can be found in the PATH), you can specify them explicitly with `--with-mysql-includes` and `--with-mysql-libs` options to `./configure` like below example.

```bash
./configure --with-mysql-includes=/home/mijin/mysql-5.6.26/include --with-mysql-libs=/home/mijin/mysql-5.6.26/lib
```

4. Now you can run SysBench 0.5 using `sysbench` command.

# Usage

1. Before running the benchmark, you should create a database for SysBench test. For example:

```bash
mysql -uroot

root:(none)> create database sbtest;
root:(none)> quit
```

2. Then, `prepare` the test. At the `prepare` stage, SysBench performs preparative actions for those tests which need them. (e.g. creating the necessary files on disk for the fileio test, or filling the test database for the oltp test)

```bash
sysbench --test=/home/mijin/sysbench-0.5/sysbench/tests/db/oltp.lua \
        --mysql-host=localhost  --mysql-db=sbtest --mysql-user=root \
        --max-requests=0  --oltp-table-size=1000000 \
        --max-time=600  --oltp-tables-count=200 --report-interval=10 \
        --db-ps-mode=disable  --random-points=10   --mysql-table-engine=InnoDB \
        --mysql-port=3307   --num-threads=128  prepare
```

3. Now, we can run the actual test.

```bash
sysbench --test=/home/mijin/sysbench-0.5/sysbench/tests/db/oltp.lua \
        --mysql-host=localhost  --mysql-db=sbtest --mysql-user=root \
        --max-requests=0  --oltp-table-size=1000000 \
        --max-time=600  --oltp-tables-count=200 --report-interval=10 \
        --db-ps-mode=disable  --random-points=10   --mysql-table-engine=InnoDB \
        --mysql-port=3307   --num-threads=128  run
```

You can get more information from [SysBench manual](http://imysql.com/wp-content/uploads/2014/10/sysbench-manual.pdf) and [SysBench Github repository](https://github.com/akopytov/sysbench).

# Error

If you get an error message as follows:

```bash
$ sysbench
sysbench: error while loading shared libraries: libmysqlclient.so.18: cannot open shared object file: No such file or directory
```

Export the LD_LIBRARY_PATH variable.

```bash
export LD_LIBRARY_PATH=/path/to/mysql/lib/
```
