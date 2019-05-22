# How to clean UNDO tablespaces

1. Create a temporary UNDO tablespace (UNDOTBS2):

```bash
SQL> CREATE UNDO TABLESPACE UNDOTBS2 DATAFILE '/oradata/undotbs2.dbf' size 5M;
```

2. Change the default UNDO tablespace to UNDOTBS2:

```bash
SQL> ALTER SYSTEM UNDO_TABLESPACE=UNDOTBS2;
```

3. Remove existing UNDO tablespace (UNDOTBS1):

```bash
SQL> DROP TABLESPACE UNDOTBS1 INCLUDING CONTENTS AND DATAFILES;
```

4. Create a new UNDOTBS1:

```bash
SQL> CREATE UNDO TABLESPACE UNDOTBS1 DATAFILE '/oradata/undotbs1.dbf' size 100M;
```

5. Change the default UNDO tablespace to UNDOTBS1:

```bash
SQL> ALTER SYSTEM UNDO_TABLESPACE=UNDOTBS1;
```

3. Remove UNDOTBS2 (temporary UNDO tablespace):

```bash
SQL> DROP TABLESPACE UNDOTBS2 INCLUDING CONTENTS AND DATAFILES;
```