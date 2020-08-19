# How to install tpcc-mysql for MyRocks

## Installation

1. Clone tpcc-mysql from [Percona GitHub repositories](https://github.com/Percona-Lab/tpcc-mysql):

```bash
$ git clone https://github.com/Percona-Lab/tpcc-mysql.git
```

2. Go to the tpcc-mysql directory and build binaries:

```bash
$ cd tpcc-mysql/src
$ make
```

## Load 

1. Change the default engine to `rocksdb` instead of `InnoDB` of `create_table.sql`. Also, MyRocks does not support foreign keys, so modify the SQL file as follows:

```bash
$ cat create_table.sql | sed -e "s/Engine=InnoDB/Engine=rocksdb DEFAULT COLLATE=latin1_bin/g" > create_table_myrocks.sql
$ cat add_fkey_idx.sql | grep -v "FOREIGN KEY" > add_fkey_idx_myrocks.sql
```

2. Create a database for TPC-C test. Go to the MyRocks base directory and run following commands:

```bash
[session 1]
$ ./bin/mysqld_safe --defaults-file=/path/to/myrocks.cnf

[session 2]
$ ./bin/mysql -u root -p -e "CREATE DATABASE tpcc100;"
$ ./bin/mysql -u root -p tpcc100 < /path/to/tpcc-mysql/create_table_myrocks.sql
$ ./bin/mysql -u root -p tpcc100 < /path/to/tpcc-mysql/add_fkey_idx_myrocks.sql
```

3. Then go back to the tpcc-mysql directory and load data. Before running the script, change `LD_LIBRARY_PATH` and enter `yourPassword` in the `load.sh` file:

```bash
$ cd tpcc-mysql
$ vi load.sh

export LD_LIBRARY_PATH=/path/to/basedir/lib
./tpcc_load -h $HOST -d $DBNAME -u root -p "" -w $WH >> 1.out &
```

4. Load data:

```bash
$ ./load.sh tpcc100 100
```

In this case, database size is about 100 GB (= 1,000 warehouses).

## Run

1. After loading, run tpcc-mysql test:

```bash
$ ./tpcc_start -h127.0.0.1 -S/tmp/mysql.sock -dtpcc1000 -uroot -pyourPassword -w1000 -c1 -r10 -l1200
```

It means:

- Host: 127.0.0.1
- MyRocks Socket: /tmp/mysql.sock
- DB: tpcc1000
- User: root
- Password: yourPassword
- Warehouse: 1000
- Connection: 1
- Rampup time: 10 (sec)
- Measure: 1200 (sec)


2. With the defined interval (`-i` option), the tool will produce the following output:

```bash
10, trx: 12920, 95%: 9.483, 99%: 18.738, max_rt: 213.169, 12919|98.778, 1292|101.096, 1293|443.955, 1293|670.842
20, trx: 12666, 95%: 7.074, 99%: 15.578, max_rt: 53.733, 12668|50.420, 1267|35.846, 1266|58.292, 1267|37.421
30, trx: 13269, 95%: 6.806, 99%: 13.126, max_rt: 41.425, 13267|27.968, 1327|32.242, 1327|40.529, 1327|29.580
40, trx: 12721, 95%: 7.265, 99%: 15.223, max_rt: 60.368, 12721|42.837, 1271|34.567, 1272|64.284, 1272|22.947
50, trx: 12573, 95%: 7.185, 99%: 14.624, max_rt: 48.607, 12573|45.345, 1258|41.104, 1258|54.022, 1257|26.626
```

Where:

- `10` - the seconds from the start of the benchmark
- `95%: 9.483:` - The 95% Response time of New Order transactions per given interval. In this case it is 9.483 sec
- `99%: 18.738:` - The 99% Response time of New Order transactions per given interval. In this case it is 18.738 sec
- `max_rt: 213.169:` - The Max Response time of New Order transactions per given interval. In this case it is 213.169 sec
- `12919|98.778, 1292|101.096, 1293|443.955, 1293|670.842` - throughput and max response time for the other kind of transactions and can be ignored

## Reference

- [Percona-Lab/tpcc-mysql](https://github.com/Percona-Lab/tpcc-mysql)
- [tpcc-mysql: Simple usage steps and how to build graphs with gnuplot](https://www.percona.com/blog/2013/07/01/tpcc-mysql-simple-usage-steps-and-how-to-build-graphs-with-gnuplot/)
- [mdcallag/mytools: helper scripts](https://github.com/mdcallag/mytools/blob/master/bench/run_tpcc/run1.sh)