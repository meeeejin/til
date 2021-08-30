# How to install TPC-H for PostgreSQL

## Installation

1. Download [TPC-H](http://tpc.org/tpc_documents_current_versions/current_specifications5.asp) and unzip it:

```bash
$ unzip 444122e6-6d11-4ad5-a8b9-37eb76ead0c6-tpc-h-tool.zip -d tpch
$ cd tpch/TPC-H_Tools_v3.0.0
```

2. Make a copy of the makefile template:

```bash
$ cd dbgen
$ cp makefile.suite makefile
$ vi makefile
```

3. Change the following lines:

```bash
...
################
## CHANGE NAME OF ANSI COMPILER HERE
################
CC      = gcc
# Current values for DATABASE are: INFORMIX, DB2, TDAT (Teradata)
#                                  SQLSERVER, SYBASE, ORACLE, VECTORWISE
# Current values for MACHINE are:  ATT, DOS, HP, IBM, ICL, MVS, 
#                                  SGI, SUN, U2200, VMS, LINUX, WIN32 
# Current values for WORKLOAD are:  TPCH
DATABASE= ORACLE
MACHINE = LINUX
WORKLOAD = TPCH
...
```

4. Compile it to create a `dbgen` executable:

```bash
$ make
```

After the compilation, you can check the `dbgen` executable:

```bash
$ ./dbgen -h
TPC-H Population Generator (Version 2.18.0 build 0)
Copyright Transaction Processing Performance Council 1994 - 2010
USAGE:
dbgen [-{vf}][-T {pcsoPSOL}]
        [-s <scale>][-C <procs>][-S <step>]
dbgen [-v] [-O m] [-s <scale>] [-U <updates>]

Basic Options
===========================
-C <n> -- separate data set into <n> chunks (requires -S, default: 1)
-f     -- force. Overwrite existing files
-h     -- display this message
-q     -- enable QUIET mode
-s <n> -- set Scale Factor (SF) to  <n> (default: 1) 
-S <n> -- build the <n>th step of the data/update set (used with -C or -U)
-U <n> -- generate <n> update sets
-v     -- enable VERBOSE mode

Advanced Options
===========================
-b <s> -- load distributions for <s> (default: dists.dss)
-d <n> -- split deletes between <n> files (requires -U)
-i <n> -- split inserts between <n> files (requires -U)
-T c   -- generate cutomers ONLY
-T l   -- generate nation/region ONLY
-T L   -- generate lineitem ONLY
-T n   -- generate nation ONLY
-T o   -- generate orders/lineitem ONLY
-T O   -- generate orders ONLY
-T p   -- generate parts/partsupp ONLY
-T P   -- generate parts ONLY
-T r   -- generate region ONLY
-T s   -- generate suppliers ONLY
-T S   -- generate partsupp ONLY

To generate the SF=1 (1GB), validation database population, use:
        dbgen -vf -s 1

To generate updates for a SF=1 (1GB), use:
        dbgen -v -U 1 -s 1
```

5. Generate a TPC-H benchmark data. `-s 10` means scale 10 (approximately 10GB of data):

```bash
$ ./dbgen -s 10
```

6. Convert `.tbl` files to a CSV format compatible with PostgreSQL

```bash
$ for i in `ls *.tbl`; do sed 's/|$//' $i > ${i/tbl/csv}; echo $i; done;
```

7. Generate Queries:

```bash
$ cd ~
$ git clone https://github.com/tvondra/pg_tpch

$ cp -r pg_tpch/dss tpch/TPC-H_Tools_v3.0.0/dbgen/
$ cp -r tpch/TPC-H_Tools_v3.0.0/dbgen/queries ~/tpch/TPC-H_Tools_v3.0.0/dbgen/dss/

$ cd tpch/TPC-H_Tools_v3.0.0/dbgen
$ for q in `seq 1 22`
do
        DSS_QUERY=dss/templates ./qgen $q > dss/queries/$q.sql
        sed 's/^select/explain (analyze, buffers)\nselect/' dss/queries/$q.sql > dss/queries/$q.explain.sql
done
```

8. To remove `^M` in each file, run the below script:

```bash
$ cd dss/queries
$ for dir in $(ls *.sql);
do
  echo "$dir";
  sed -i s/^M//g $dir  # ^M은 Control + v + m 으로 입력 가능
done
```

9. Update the `.csv` file path of `dbgen/dss/tpch-load.sql`

```bash
$ cd ..
$ vi tpch-load.sql
:%s/\/tmp\/dss-data/\/home\/vldb\/tpch\/TPC-H_Tools_v3.0.0\/dbgen/g
```

10. Start a PostgreSQL server and load TPC-H data:

```bash
$ cd ~
$ initdb -D /home/vldb/test-data/
$ pg_ctl -D /home/vldb/test-data -l /home/vldb/test-log/logfile start
```

```bash
$ cd tpch/TPC-H_Tools_v3.0.0/dbgen/dss
$ psql postgres
psql (13.3)
Type "help" for help.

postgres=# \i tpch-load.sql
...
postgres=# \i tpch-pkeys.sql
...
postgres=# \i tpch-alter.sql
...
postgres=# \i tpch-index.sql
...
```

11. Run the TPC-H benchmark:

```bash
$ ./tpch.sh ./results postgres vldb
```

12. Stop the PostgreSQL server:

```bash
$ pg_ctl -D /home/vldb/test-data -m smart stop
```

 ## Reference

 - [Tomas Vondra, TPC-H PostgreSQL benchmark](https://github.com/tvondra/pg_tpch)