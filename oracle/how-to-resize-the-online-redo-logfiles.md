# How To Resize the Online Redo Logfiles

1. First, check the size and location of the current logs:

```bash
SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC
---------- ---------- ---------- ---------- ---------- ---------- ---
STATUS           FIRST_CHANGE# FIRST_TIM NEXT_CHANGE# NEXT_TIME     CON_ID
---------------- ------------- --------- ------------ --------- ----------
         1          1       1438  209715200        512          1 NO
INACTIVE              41514532 28-MAY-19     41552202 28-MAY-19          0
         2          1       1439  209715200        512          1 NO
CURRENT               41552202 28-MAY-19   1.8447E+19                    0
         3          1       1437  209715200        512          1 NO
INACTIVE              41476368 28-MAY-19     41514532 28-MAY-19          0

SQL> select * from v$logfile;

    GROUP# STATUS  TYPE                              MEMBER  IS_     CON_ID
---------- ------- ------- --------------------------------- --- ---------- 
    3      ONLINE           /home/mijin/test_log/redo03.log  NO           0
    2      ONLINE           /home/mijin/test_log/redo02.log  NO           0
    1      ONLINE           /home/mijin/test_log/redo01.log  NO           0
```

2. Drop **INACTIVE** redo log groups 1 and 3:

```bash
SQL> alter database drop logfile group 1;

Database altered.

SQL> !rm /home/mijin/test_log/redo01.log

SQL> alter database drop logfile group 3;

Database altered.

SQL> !rm /home/mijin/test_log/redo03.log

```

3. Create new redo log groups with 1GB size:

```bash
SQL> alter database add logfile group 1 '/home/mijin/test_log/redo01.log' size 1g;

Database altered.

SQL> alter database add logfile group 3 '/home/mijin/test_log/redo03.log' size 1g;

Database altered.
```

Then, check the changed size:

```bash
SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC
---------- ---------- ---------- ---------- ---------- ---------- ---
STATUS           FIRST_CHANGE# FIRST_TIM NEXT_CHANGE# NEXT_TIME     CON_ID
---------------- ------------- --------- ------------ --------- ----------
         1          1          0 1073741824        512          1 YES
UNUSED                       0                      0                    0
         2          1       1439  209715200        512          1 NO
CURRENT               41552202 28-MAY-19   1.8447E+19                    0
         3          1          0 1073741824        512          1 YES
UNUSED                       0                      0                    0
```

4. Switch until we are into log group 1 or 3, so we can drop **CURRENT** log groups 2:

```bash
SQL> alter system switch logfile; 

System altered.

SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC
---------- ---------- ---------- ---------- ---------- ---------- ---
STATUS           FIRST_CHANGE# FIRST_TIM NEXT_CHANGE# NEXT_TIME     CON_ID
---------------- ------------- --------- ------------ --------- ----------
         1          1       1440 1073741824        512          1 NO
CURRENT               41556266 28-MAY-19   1.8447E+19                    0
         2          1       1439  209715200        512          1 NO
ACTIVE                41552202 28-MAY-19     41556266 28-MAY-19          0
         3          1          0 1073741824        512          1 YES
UNUSED                       0                      0                    0
```

Redo log group 2 becomes **ACTIVE** after `alter system switch log file` which means could not be dropped. In this case, just wait a moment or use `alter system checkpoint` to make redo log group 2 **INACTIVE**.

5. If the redo log group becomes **INACTIVE**, drop it:

```bash
SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC
---------- ---------- ---------- ---------- ---------- ---------- ---
STATUS           FIRST_CHANGE# FIRST_TIM NEXT_CHANGE# NEXT_TIME     CON_ID
---------------- ------------- --------- ------------ --------- ----------
         1          1       1440 1073741824        512          1 NO
CURRENT               41556266 28-MAY-19   1.8447E+19                    0
         2          1       1439  209715200        512          1 NO
INACTIVE              41552202 28-MAY-19     41556266 28-MAY-19          0
         3          1          0 1073741824        512          1 YES
UNUSED                       0                      0                    0

SQL> alter database drop logfile group 2;

Database altered.

SQL> !rm /home/mijin/test_log/redo02.log

SQL> alter database add logfile group 2 '/home/mijin/test_log/redo02.log' size 1g;

Database altered.
```

Check the size of redo log groups:

```bash
SQL> select * from v$log;

    GROUP#    THREAD#  SEQUENCE#      BYTES  BLOCKSIZE    MEMBERS ARC
---------- ---------- ---------- ---------- ---------- ---------- ---
STATUS           FIRST_CHANGE# FIRST_TIM NEXT_CHANGE# NEXT_TIME     CON_ID
---------------- ------------- --------- ------------ --------- ----------
         1          1       1440 1073741824        512          1 NO
CURRENT               41556266 28-MAY-19   1.8447E+19                    0
         2          1          0 1073741824        512          1 YES
UNUSED                       0                      0                    0
         3          1          0 1073741824        512          1 YES
UNUSED                       0                      0                    0
```