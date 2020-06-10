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

Copy those files to the target data directory (e.g., `/home/mijin/tpch`):

```bash
$ cp *.tbl* /home/mijin/tpch/
```

7. Set up the TPC-H schema:

```sql
CREATE USER tpch IDENTIFIED BY tpch;

GRANT CREATE SESSION,
      CREATE TABLE,
      UNLIMITED TABLESPACE
    TO tpch;

CREATE OR REPLACE DIRECTORY tpch_dir AS '/home/mijin/tpch';

GRANT READ ON DIRECTORY tpch_dir TO tpch;

CREATE TABLE tpch.ext_part
(
    p_partkey       NUMBER(10, 0),
    p_name          VARCHAR2(55),
    p_mfgr          CHAR(25),
    p_brand         CHAR(10),
    p_type          VARCHAR2(25),
    p_size          INTEGER,
    p_container     CHAR(10),
    p_retailprice   NUMBER,
    p_comment       VARCHAR2(23)
)
ORGANIZATION EXTERNAL
    (TYPE oracle_loader
          DEFAULT DIRECTORY tpch_dir
              ACCESS PARAMETERS (
                  FIELDS
                      TERMINATED BY '|'
                  MISSING FIELD VALUES ARE NULL
              )
          LOCATION('part.tbl*'))
    PARALLEL 4;

CREATE TABLE tpch.part
(
    p_partkey       NUMBER(10, 0) NOT NULL,
    p_name          VARCHAR2(55) NOT NULL,
    p_mfgr          CHAR(25) NOT NULL,
    p_brand         CHAR(10) NOT NULL,
    p_type          VARCHAR2(25) NOT NULL,
    p_size          INTEGER NOT NULL,
    p_container     CHAR(10) NOT NULL,
    p_retailprice   NUMBER NOT NULL,
    p_comment       VARCHAR2(23) NOT NULL
);

CREATE TABLE tpch.ext_supplier
(
    s_suppkey     NUMBER(10, 0),
    s_name        CHAR(25),
    s_address     VARCHAR2(40),
    s_nationkey   NUMBER(10, 0),
    s_phone       CHAR(15),
    s_acctbal     NUMBER,
    s_comment     VARCHAR2(101)
)
ORGANIZATION EXTERNAL
    (TYPE oracle_loader
          DEFAULT DIRECTORY tpch_dir
              ACCESS PARAMETERS (
                  FIELDS
                      TERMINATED BY '|'
                  MISSING FIELD VALUES ARE NULL
              )
          LOCATION('supplier.tbl'))
    PARALLEL 4;

CREATE TABLE tpch.supplier
(
    s_suppkey     NUMBER(10, 0) NOT NULL,
    s_name        CHAR(25) NOT NULL,
    s_address     VARCHAR2(40) NOT NULL,
    s_nationkey   NUMBER(10, 0) NOT NULL,
    s_phone       CHAR(15) NOT NULL,
    s_acctbal     NUMBER NOT NULL,
    s_comment     VARCHAR2(101) NOT NULL
);

CREATE TABLE tpch.ext_partsupp
(
    ps_partkey      NUMBER(10, 0),
    ps_suppkey      NUMBER(10, 0),
    ps_availqty     INTEGER,
    ps_supplycost   NUMBER,
    ps_comment      VARCHAR2(199)
)
ORGANIZATION EXTERNAL
    (TYPE oracle_loader
          DEFAULT DIRECTORY tpch_dir
              ACCESS PARAMETERS (
                  FIELDS
                      TERMINATED BY '|'
                  MISSING FIELD VALUES ARE NULL
              )
          LOCATION('partsupp.tbl'))
    PARALLEL 4;

CREATE TABLE tpch.partsupp
(
    ps_partkey      NUMBER(10, 0) NOT NULL,
    ps_suppkey      NUMBER(10, 0) NOT NULL,
    ps_availqty     INTEGER NOT NULL,
    ps_supplycost   NUMBER NOT NULL,
    ps_comment      VARCHAR2(199) NOT NULL
);

CREATE TABLE tpch.ext_customer
(
    c_custkey      NUMBER(10, 0),
    c_name         VARCHAR2(25),
    c_address      VARCHAR2(40),
    c_nationkey    NUMBER(10, 0),
    c_phone        CHAR(15),
    c_acctbal      NUMBER,
    c_mktsegment   CHAR(10),
    c_comment      VARCHAR2(117)
)
ORGANIZATION EXTERNAL
    (TYPE oracle_loader
          DEFAULT DIRECTORY tpch_dir
              ACCESS PARAMETERS (
                  FIELDS
                      TERMINATED BY '|'
                  MISSING FIELD VALUES ARE NULL
              )
          LOCATION('customer.tbl'))
    PARALLEL 4;

CREATE TABLE tpch.customer
(
    c_custkey      NUMBER(10, 0) NOT NULL,
    c_name         VARCHAR2(25) NOT NULL,
    c_address      VARCHAR2(40) NOT NULL,
    c_nationkey    NUMBER(10, 0) NOT NULL,
    c_phone        CHAR(15) NOT NULL,
    c_acctbal      NUMBER NOT NULL,
    c_mktsegment   CHAR(10) NOT NULL,
    c_comment      VARCHAR2(117) NOT NULL
);

CREATE TABLE tpch.ext_orders
(
    o_orderkey        NUMBER(10, 0),
    o_custkey         NUMBER(10, 0),
    o_orderstatus     CHAR(1),
    o_totalprice      NUMBER,
    o_orderdate       CHAR(10),
    o_orderpriority   CHAR(15),
    o_clerk           CHAR(15),
    o_shippriority    INTEGER,
    o_comment         VARCHAR2(79)
)
ORGANIZATION EXTERNAL
    (TYPE oracle_loader
          DEFAULT DIRECTORY tpch_dir
              ACCESS PARAMETERS (
                  FIELDS
                      TERMINATED BY '|'
                  MISSING FIELD VALUES ARE NULL
              )
          LOCATION('orders.tbl'))
    PARALLEL 4;

CREATE TABLE tpch.orders
(
    o_orderkey        NUMBER(10, 0) NOT NULL,
    o_custkey         NUMBER(10, 0) NOT NULL,
    o_orderstatus     CHAR(1) NOT NULL,
    o_totalprice      NUMBER NOT NULL,
    o_orderdate       DATE NOT NULL,
    o_orderpriority   CHAR(15) NOT NULL,
    o_clerk           CHAR(15) NOT NULL,
    o_shippriority    INTEGER NOT NULL,
    o_comment         VARCHAR2(79) NOT NULL
);

CREATE TABLE tpch.ext_lineitem
(
    l_orderkey        NUMBER(10, 0),
    l_partkey         NUMBER(10, 0),
    l_suppkey         NUMBER(10, 0),
    l_linenumber      INTEGER,
    l_quantity        NUMBER,
    l_extendedprice   NUMBER,
    l_discount        NUMBER,
    l_tax             NUMBER,
    l_returnflag      CHAR(1),
    l_linestatus      CHAR(1),
    l_shipdate        CHAR(10),
    l_commitdate      CHAR(10),
    l_receiptdate     CHAR(10),
    l_shipinstruct    CHAR(25),
    l_shipmode        CHAR(10),
    l_comment         VARCHAR2(44)
)
ORGANIZATION EXTERNAL
    (TYPE oracle_loader
          DEFAULT DIRECTORY tpch_dir
              ACCESS PARAMETERS (
                  FIELDS
                      TERMINATED BY '|'
                  MISSING FIELD VALUES ARE NULL
              )
          LOCATION('lineitem.tbl'))
    PARALLEL 4;

CREATE TABLE tpch.lineitem
(
    l_orderkey        NUMBER(10, 0),
    l_partkey         NUMBER(10, 0),
    l_suppkey         NUMBER(10, 0),
    l_linenumber      INTEGER,
    l_quantity        NUMBER,
    l_extendedprice   NUMBER,
    l_discount        NUMBER,
    l_tax             NUMBER,
    l_returnflag      CHAR(1),
    l_linestatus      CHAR(1),
    l_shipdate        DATE,
    l_commitdate      DATE,
    l_receiptdate     DATE,
    l_shipinstruct    CHAR(25),
    l_shipmode        CHAR(10),
    l_comment         VARCHAR2(44)
);

CREATE TABLE tpch.ext_nation
(
    n_nationkey   NUMBER(10, 0),
    n_name        CHAR(25),
    n_regionkey   NUMBER(10, 0),
    n_comment     VARCHAR(152)
)
ORGANIZATION EXTERNAL
    (TYPE oracle_loader
          DEFAULT DIRECTORY tpch_dir
              ACCESS PARAMETERS (
                  FIELDS
                      TERMINATED BY '|'
                  MISSING FIELD VALUES ARE NULL
              )
          LOCATION('nation.tbl'))
    PARALLEL 4;

CREATE TABLE tpch.nation
(
    n_nationkey   NUMBER(10, 0),
    n_name        CHAR(25),
    n_regionkey   NUMBER(10, 0),
    n_comment     VARCHAR(152)
);

CREATE TABLE tpch.ext_region
(
    r_regionkey   NUMBER(10, 0),
    r_name        CHAR(25),
    r_comment     VARCHAR(152)
)
ORGANIZATION EXTERNAL
    (TYPE oracle_loader
          DEFAULT DIRECTORY tpch_dir
              ACCESS PARAMETERS (
                  FIELDS
                      TERMINATED BY '|'
                  MISSING FIELD VALUES ARE NULL
              )
          LOCATION('region.tbl'))
    PARALLEL 4;

CREATE TABLE tpch.region
(
    r_regionkey   NUMBER(10, 0),
    r_name        CHAR(25),
    r_comment     VARCHAR(152)
);
```

8. Add the parallel clause and remove any previous data:

```sql
ALTER TABLE tpch.part     PARALLEL 4;
ALTER TABLE tpch.supplier PARALLEL 4;
ALTER TABLE tpch.partsupp PARALLEL 4;
ALTER TABLE tpch.customer PARALLEL 4;
ALTER TABLE tpch.orders   PARALLEL 4;
ALTER TABLE tpch.lineitem PARALLEL 4;

TRUNCATE TABLE tpch.part;
TRUNCATE TABLE tpch.supplier;
TRUNCATE TABLE tpch.partsupp;
TRUNCATE TABLE tpch.customer;
TRUNCATE TABLE tpch.orders;
TRUNCATE TABLE tpch.lineitem;
TRUNCATE TABLE tpch.nation;
TRUNCATE TABLE tpch.region;
```

9. Load the data:

```sql
ALTER SESSION SET nls_date_format='YYYY-MM-DD';
ALTER SESSION ENABLE PARALLEL DML;

INSERT /*+ APPEND */ INTO  tpch.part     SELECT * FROM tpch.ext_part;
INSERT /*+ APPEND */ INTO  tpch.supplier SELECT * FROM tpch.ext_supplier;
INSERT /*+ APPEND */ INTO  tpch.partsupp SELECT * FROM tpch.ext_partsupp;
INSERT /*+ APPEND */ INTO  tpch.customer SELECT * FROM tpch.ext_customer;
INSERT /*+ APPEND */ INTO  tpch.orders   SELECT * FROM tpch.ext_orders;
INSERT /*+ APPEND */ INTO  tpch.lineitem SELECT * FROM tpch.ext_lineitem;
INSERT /*+ APPEND */ INTO  tpch.nation   SELECT * FROM tpch.ext_nation;
INSERT /*+ APPEND */ INTO  tpch.region   SELECT * FROM tpch.ext_region;
```

10. Add the constraints and indexes:

```sql
ALTER TABLE tpch.part
    ADD CONSTRAINT pk_part PRIMARY KEY(p_partkey);

ALTER TABLE tpch.supplier
    ADD CONSTRAINT pk_supplier PRIMARY KEY(s_suppkey);

ALTER TABLE tpch.partsupp
    ADD CONSTRAINT pk_partsupp PRIMARY KEY(ps_partkey, ps_suppkey);

ALTER TABLE tpch.customer
    ADD CONSTRAINT pk_customer PRIMARY KEY(c_custkey);

ALTER TABLE tpch.orders
    ADD CONSTRAINT pk_orders PRIMARY KEY(o_orderkey);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT pk_lineitem PRIMARY KEY(l_linenumber, l_orderkey);

ALTER TABLE tpch.nation
    ADD CONSTRAINT pk_nation PRIMARY KEY(n_nationkey);

ALTER TABLE tpch.region
    ADD CONSTRAINT pk_region PRIMARY KEY(r_regionkey);

ALTER TABLE tpch.partsupp
    ADD CONSTRAINT fk_partsupp_part FOREIGN KEY(ps_partkey) REFERENCES tpch.part(p_partkey);

ALTER TABLE tpch.partsupp
    ADD CONSTRAINT fk_partsupp_supplier FOREIGN KEY(ps_suppkey) REFERENCES tpch.supplier(s_suppkey);

ALTER TABLE tpch.customer
    ADD CONSTRAINT fk_customer_nation FOREIGN KEY(c_nationkey) REFERENCES tpch.nation(n_nationkey);

ALTER TABLE tpch.orders
    ADD CONSTRAINT fk_orders_customer FOREIGN KEY(o_custkey) REFERENCES tpch.customer(c_custkey);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT fk_lineitem_order FOREIGN KEY(l_orderkey) REFERENCES tpch.orders(o_orderkey);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT fk_lineitem_part FOREIGN KEY(l_partkey) REFERENCES tpch.part(p_partkey);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT fk_lineitem_supplier FOREIGN KEY(l_suppkey) REFERENCES tpch.supplier(s_suppkey);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT fk_lineitem_partsupp FOREIGN KEY(l_partkey, l_suppkey)
        REFERENCES tpch.partsupp(ps_partkey, ps_suppkey);

ALTER TABLE tpch.part
    ADD CONSTRAINT chk_part_partkey CHECK(p_partkey >= 0);

ALTER TABLE tpch.supplier
    ADD CONSTRAINT chk_supplier_suppkey CHECK(s_suppkey >= 0);

ALTER TABLE tpch.customer
    ADD CONSTRAINT chk_customer_custkey CHECK(c_custkey >= 0);

ALTER TABLE tpch.partsupp
    ADD CONSTRAINT chk_partsupp_partkey CHECK(ps_partkey >= 0);

ALTER TABLE tpch.region
    ADD CONSTRAINT chk_region_regionkey CHECK(r_regionkey >= 0);

ALTER TABLE tpch.nation
    ADD CONSTRAINT chk_nation_nationkey CHECK(n_nationkey >= 0);

ALTER TABLE tpch.part
    ADD CONSTRAINT chk_part_size CHECK(p_size >= 0);

ALTER TABLE tpch.part
    ADD CONSTRAINT chk_part_retailprice CHECK(p_retailprice >= 0);

ALTER TABLE tpch.partsupp
    ADD CONSTRAINT chk_partsupp_availqty CHECK(ps_availqty >= 0);

ALTER TABLE tpch.partsupp
    ADD CONSTRAINT chk_partsupp_supplycost CHECK(ps_supplycost >= 0);

ALTER TABLE tpch.orders
    ADD CONSTRAINT chk_orders_totalprice CHECK(o_totalprice >= 0);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT chk_lineitem_quantity CHECK(l_quantity >= 0);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT chk_lineitem_extendedprice CHECK(l_extendedprice >= 0);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT chk_lineitem_tax CHECK(l_tax >= 0);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT chk_lineitem_discount CHECK(l_discount >= 0.00 AND l_discount <= 1.00);

ALTER TABLE tpch.lineitem
    ADD CONSTRAINT chk_lineitem_ship_rcpt CHECK(l_shipdate <= l_receiptdate);
```

11. Create queries:

```bash
$ cd queries
$ for i in {1..22}; do ./qgen $i > query-$i.sql; done
```

12. Execute queries:

>> to be updated..