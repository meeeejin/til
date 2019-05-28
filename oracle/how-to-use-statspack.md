# How to use Statspack

- [Reference: Using Statspack](https://docs.oracle.com/cd/A97385_01/server.920/a96533/statspac.htm)

## Installing interactive Statspack

To run the installation script, you must use SQL*Plus and connect as a user with SYSDBA privilege:

```bash
SQL> conn / as sysdba
SQL> @?/rdbms/admin/spcreate.sql
```

For better performance analysis, set the initialization parameter TIMED_STATISTICS to true:

```bash
SQL> alter system set TIMED_STATISTICS=true;
```

## Taking a Statspack snapshot for data gathering

The simplest interactive way to take a snapshot is to login to SQL*Plus as the PERFSTAT user and run the procedure STATSPACK.SNAP:

```bash
SQL> conn perfstat/perfstat
SQL> exec statspack.snap
```

Check the list of snapshots:

```bash
SQL> select snap_id, snap_time from stats$snapshot;
```

## Running the Statspack report

To examine the change in instancewide statistics between two time periods, the `SPREPORT.SQL` file is run while connected to the PERFSTAT user:

```bash
SQL> conn perfstat/perfstat
SQL> @?/rdbms/admin/spreport.sql
```

## Deleting unnecessary snapshots

```bash
SQL> conn perfstat/perfstat
SQL> @?/rdbms/admin/sppurge.sql
...
Enter value for losnapid: 1
Using 1 for lower bound.
...
Enter value for hisnapid: 3
Using 3 for upper bound.
...
Deleting snapshots 1 - 3.
...
```

## Truncating all snapshots

```bash
SQL> conn PERFSTAT/PERFSTAT
SQL> @?/rdbms/admin/sptrunc.sql 
```