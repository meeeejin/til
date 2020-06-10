# How to add a datafile to a tablespace

The following example enables automatic extension for a datafile added to the users tablespace:

```sql
ALTER TABLESPACE users
    ADD DATAFILE '/home/mijin/test_data/users02.dbf' SIZE 1000M
      AUTOEXTEND ON
      MAXSIZE UNLIMITED;
```