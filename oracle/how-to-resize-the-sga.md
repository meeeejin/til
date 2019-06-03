# How to resize the SGA

1. Check the size of SGA and the location of spfile:

```bash
$ sqlplus / as sysdba
...

SQL> startup
...

SQL> select * from v$sgainfo;

NAME                                  BYTES RES     CON_ID
-------------------------------- ---------- --- ----------
Fixed SGA Size                      8625368 No           0
Redo Buffers                       24928256 No           0
Buffer Cache Size                 872415232 Yes          0
In-Memory Area Size                       0 No           0
Shared Pool Size                 1962934272 Yes          0
Large Pool Size                           0 Yes          0
Java Pool Size                     16777216 Yes          0
Streams Pool Size                 201326592 Yes          0
Shared IO Pool Size                67108864 Yes          0
Data Transfer Cache Size                  0 Yes          0
Granule Size                       16777216 No           0

NAME                                  BYTES RES     CON_ID
-------------------------------- ---------- --- ----------
Maximum SGA Size                 3087007744 No           0
Startup overhead in Shared Pool  1175760304 No           0
Free SGA Memory Available                 0              0

14 rows selected.

SQL> show parameter spfile;
NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string      /u01/app/oracle/product/12.2.0
                                                 /dbhome_1/dbs/spfileSID.ora
```

Current maximum size of SGA is **3087007744** bytes, and I want to increase the size.

2. Create a pfile from the spfile:

```bash
SQL> create pfile from spfile;

File created.
```

3. Open the pfile and change the target size of SGA using an editor. In my case, the location of pfile is `/u01/app/oracle/product/12.2.0/dbhome_1/dbs/initSID.ora`.:

```bash
$ vi initSID.ora
```

4. Shutdown the database and restart it with the modified pfile:

```bash
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.

SQL> startup pfile=/u01/app/oracle/product/12.2.0/dbhome_1/dbs/initSID.ora
...
```

5. Check the modified maximum size of SGA. Then, create a spfile from the modified pfile:

```bash
SQL> select * from v$sgainfo;

NAME                                  BYTES RES     CON_ID
-------------------------------- ---------- --- ----------
Fixed SGA Size                      8632592 No           0
Redo Buffers                       24920064 No           0
Buffer Cache Size                2382364672 Yes          0
In-Memory Area Size                       0 No           0
Shared Pool Size                 1073741824 Yes          0
Large Pool Size                  2030043136 Yes          0
Java Pool Size                     16777216 Yes          0
Streams Pool Size                  33554432 Yes          0
Shared IO Pool Size                       0 Yes          0
Data Transfer Cache Size                  0 Yes          0
Granule Size                       16777216 No           0

NAME                                  BYTES RES     CON_ID
-------------------------------- ---------- --- ----------
Maximum SGA Size                 5570035712 No           0
Startup overhead in Shared Pool  1189901912 No           0
Free SGA Memory Available                 0              0

14 rows selected.

SQL> create spfile from pfile;

File created.
```

The maximum size of SGA has increased to **5570035712** bytes.

6. Restart the database:

```bash
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.

SQL> startup
...
```