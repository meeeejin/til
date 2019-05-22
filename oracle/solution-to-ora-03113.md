# Solution to ORA-03113

- ORA-03113: end-of-file on communication channel

## Solution

```bash
$ sqlplus / as sysdba
...

SQL> startup nomount
...

SQL> alter database mount;
Database altered.

SQL> recover database until cancel;
Media recovery complete. 

SQL> alter database open resetlogs;
Database altered.

SQL> shutdown immediate
Database closed. 
Database dismounted. 
ORACLE instance shut down. 

SQL> startup 
...
Database mounted.
Database opened.
```

