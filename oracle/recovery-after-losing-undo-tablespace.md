# Recovery after losing UNDO tablespace

- [Reference](https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:5669213349582)

If you get the error `ORA-01012: not logged on`, remove the orphaned shared memory segments using `sysresv`. You can remove the shared memory segment using `ipcrm -m`.

```bash
$ sysresv
...
Shared Memory:
ID              KEY
2621451         0x00000000
2654220         0x00000000
2588682         0x00000000
2686989         0x43713384
Semaphores:
ID              KEY
...
Oracle Instance not alive for sid "SID"

$ ipcrm -m 2621451
$ ipcrm -m 2654220
$ ipcrm -m 2588682
$ ipcrm -m 2686989
```

Then, start the database.

```bash
$ sqlplus / as sysdba
...
SQL> startup
ORACLE instance started.

Total System Global Area 5637144576 bytes
Fixed Size                  8632784 bytes
Variable Size            4462741040 bytes
Database Buffers         1140850688 bytes
Redo Buffers               24920064 bytes
Database mounted.
Database opened.
```