# How to install TPC-H for Oracle

1. Download [TPC-H](http://www.tpc.org/tpc_documents_current_versions/current_specifications5.asp) and unzip it.

2. Make a copy of the makefile template and edit it:

```bash
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

4. Run `make` to create a `dbgen` executable:

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

5. Generate a TPC-H benchmark data. `-s 150` means scale 150 (approximately 150GB of data). Also, splits the data into 4 parts for parallelization. Thus, we need to call `dbgen` 4 times:

```bash
$ ./dbgen -s 150 -S 1 -C 4
$ ./dbgen -s 150 -S 2 -C 4
$ ./dbgen -s 150 -S 3 -C 4
$ ./dbgen -s 150 -S 4 -C 4
```

6. When complete, you’ll have the following files:

```bash
$ ls *.tbl*                                                           
customer.tbl.1  lineitem.tbl.2  orders.tbl.2  part.tbl.3      partsupp.tbl.4  supplier.tbl.4
customer.tbl.2  lineitem.tbl.3  orders.tbl.3  part.tbl.4      region.tbl
customer.tbl.3  lineitem.tbl.4  orders.tbl.4  partsupp.tbl.1  supplier.tbl.1
customer.tbl.4  nation.tbl      part.tbl.1    partsupp.tbl.2  supplier.tbl.2
lineitem.tbl.1  orders.tbl.1    part.tbl.2    partsupp.tbl.3  supplier.tbl.3
```

>> to be updated..