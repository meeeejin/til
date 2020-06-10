# How to relocate files in Oracle

- [Reference: Renaming or Moving Oracle Files](https://oracle-base.com/articles/misc/renaming-or-moving-oracle-files)

## Control files

1. Check the current location of the control files:

```sql
SQL> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/orcl/control01.ctl
/u01/app/oracle/oradata/orcl/control02.ctl

SQL> show parameter control_files

NAME                                 TYPE
------------------------------------ ----------------------
VALUE
------------------------------
control_files                        string
/u01/app/oracle/oradata/orcl/c
ontrol01.ctl, /u01/app/oracle/
oradata/orcl/control02.ctl
```

2. Alter the `control_files` parameter:

```sql
SQL> alter system set control_files='/home/mijin/test_data/control01.ctl','/home/mijin/test_data/control02.ctl' scope=spfile;

System altered.
```

3. Shutdown the database:

```sql
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
```

4. Relocate or move the physical control files:

```bash
$ cd /u01/app/oracle/oradata/orcl
$ mv control* /home/mijin/test_data/
```

5. Start the database:

```sql
SQL> startup
ORACLE instance started.

Total System Global Area 1.0133E+10 bytes
Fixed Size                 12170088 bytes
Variable Size            3858762904 bytes
Database Buffers         6241124352 bytes
Redo Buffers               21381120 bytes
Database mounted.
Database opened.
```

6. Check the current location of the control files:

```sql
SQL> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/home/mijin/test_data/control01.ctl
/home/mijin/test_data/control02.ctl
```

## Log files

1. Check the current location of the log files:

```sql
SQL> SELECT member FROM v$logfile;

MEMBER
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/orcl/redo03.log
/u01/app/oracle/oradata/orcl/redo02.log
/u01/app/oracle/oradata/orcl/redo01.log
```

2. Shutdown the database:

```sql
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
```

3. Relocate or move the physical log files:

```bash
$ cd /u01/app/oracle/oradata/orcl
$ mv redo* /home/mijin/test_data/
```

4. Start the database in mount mode:

```sql
SQL> startup mount
ORACLE instance started.

Total System Global Area 1.0133E+10 bytes
Fixed Size                 12170088 bytes
Variable Size            3858762904 bytes
Database Buffers         6241124352 bytes
Redo Buffers               21381120 bytes
Database mounted.
```

5. Rename the the file within the Oracle dictionary:

```sql
SQL> alter database rename file '/u01/app/oracle/oradata/orcl/redo01.log' to '/home/mijin/test_data/redo01.log';
SQL> alter database rename file '/u01/app/oracle/oradata/orcl/redo02.log' to '/home/mijin/test_data/redo02.log';
SQL> alter database rename file '/u01/app/oracle/oradata/orcl/redo03.log' to '/home/mijin/test_data/redo03.log';
```

6. Open the database:

```sql
SQL> alter database open;

Database altered.
```

7. Check the current location of the log files:

```sql
SQL> SELECT member FROM v$logfile;

MEMBER
--------------------------------------------------------------------------------
/home/mijin/test_data/redo03.log
/home/mijin/test_data/redo02.log
/home/mijin/test_data/redo01.log
```

## Data files

1. Check the current location of the data files:

```sql
SQL> select tablespace_name,file_name from dba_data_files;

TABLESPACE_NAME
------------------------------------------------------------
FILE_NAME
--------------------------------------------------------------------------------
SYSTEM
/u01/app/oracle/oradata/orcl/system01.dbf
...
```

2. Shutdown the database:

```sql
SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
```

3. Relocate or move the physical log files:

```bash
$ cd /u01/app/oracle/oradata/orcl
$ mv system01.dbf /home/mijin/test_data/system01.dbf
```

4. Start the database in mount mode:

```sql
SQL> startup mount
ORACLE instance started.

Total System Global Area 1.0133E+10 bytes
Fixed Size                 12170088 bytes
Variable Size            3858762904 bytes
Database Buffers         6241124352 bytes
Redo Buffers               21381120 bytes
Database mounted.
```

5. Rename the the file within the Oracle dictionary:

```sql
SQL> alter database rename file '/u01/app/oracle/oradata/orcl/system01.dbf' to '/home/mijin/test_data/system01.dbf';
```

6. Open the database:

```sql
SQL> alter database open;

Database altered.
```

7. Check the current location of the log files:

```sql
SQL> select tablespace_name,file_name from dba_data_files;

TABLESPACE_NAME
------------------------------------------------------------
FILE_NAME
--------------------------------------------------------------------------------
SYSTEM
/home/mijin/test_data/system01.dbf
...
```