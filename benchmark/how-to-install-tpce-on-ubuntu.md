# How to install TPC-E on Ubuntu (for MySQL)

## Prerequisite

```bash
$ sudo apt-get install unixodbc unixodbc-dev
```

## Installation

1. If you don't have `bzr` package in your Ubuntu, download it first.

```bash
$ sudo apt-get install bzr
```

2. Download the TPC-E benchmark tool for MySQL (tpcemysql).

```bash
$ bzr branch lp:~percona-dev/perconatools/tpcemysql
```

3. Go to the tpcemysql directory and run following commands.

```bash
$ cd tpcemysql/prj
$ make
```

After build, you can see the below three files in `bin` directory.

```bash
$ cd ..
$ ls -al bin/
total 14116
drwxrwxr-x  2 mijin mijin    4096 Jul 16 11:00 .
drwxrwxr-x 15 mijin mijin    4096 Jul 16 11:34 ..
-rwxrwxr-x  1 mijin mijin 4743136 Jul 15 17:09 EGenLoader
-rwxrwxr-x  1 mijin mijin 8856739 Jul 15 17:10 EGenSimpleTest
-rwxrwxr-x  1 mijin mijin  839969 Jul 15 17:09 EGenValidate
```

4. Then, generate test data.

```bash
$ ./bin/EGenLoader -w 150
```

You can change various parameters for your purpose. The options are as follows:

```bash
$ ./bin/EGenLoader -h
EGen v1.9.0
Usage:
EGenLoader [options] 

 Where
  Option                       Default     Description
   -b number                   1           Beginning customer ordinal position
   -c number                   5000        Number of customers (for this instance)
   -t number                   5000        Number of customers (total in the database)
   -f number                   500         Scale factor (customers per 1 tpsE)
   -w number                   300          Number of Workdays (8-hour days) of 
                                           initial trades to populate
   -i dir                      flat_in/    Directory for input files
   -l [FLAT|ODBC|CUSTOM|NULL]  FLAT        Type of load
   -m [APPEND|OVERWRITE]       OVERWRITE   Flat File output mode
   -o dir                      flat_out/   Directory for output files

   -x                          -x          Generate all tables
   -xf                                     Generate all fixed-size tables
   -xd                                     Generate all scaling and growing tables
                                           (equivalent to -xs -xg)
   -xs                                     Generate scaling tables
                                           (except BROKER)
   -xg                                     Generate growing tables and BROKER
   -g                                      Disable caching when generating growing tables
```

5. Now, build the schema and load initial database.

```bash
$ cd scripts/mysql
$ mysql -u tpce -p tpce < 1_create_table.sql
$ mysql -u tpce -p tpce < 2_load_data.sql
$ mysql -u tpce -p tpce < 3_create_fk.sql
$ mysql -u tpce -p tpce < 4_create_index.sql
$ mysql -u tpce -p tpce < 5_create_sequence.sql
```

6. Start running.

```bash
$ cd ../../
$ ./bin/EGenSimpleTest -S localhost -r 10 -t 3600 -u 64
```

You can change various parameters for your purpose. The options are as follows:

```bash
$ ./bin/EGenSimpleTest -h
EGen v1.9.0
(for MySQL)
(Prepared Statement)

Usage: EGenSimpleTest {options}

  where
   Option      Default                Description
   =========   ===================    =============================================
   -e string   flat_in                Path to EGen input files
   -S string   localhost              Database server
   -D string   tpce                   Database name
   -U string   tpce                   Database user
   -P string   tpce                   Database password
   -c number   5000                   Configured customer count
   -a number   5000                   Active customer count
   -f number   500                    # of customers for 1 TRTPS
   -d number   300                    # of Days of Initial Trades
   -l number   1000                   # of customers in one load unit
   -t number                          Duration of the test (seconds)
   -r number                          Duration of ramp up period (seconds)
   -u number                          # of Users
```
