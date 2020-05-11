# InnoDB page splits

## Tracking page splits

1. Use `module_index` to enable all counters associated with the `index` subsystem:

```bash
$ mysql -uroot -p
...
mysql> SET GLOBAL innodb_monitor_enable = module_index;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT name, subsystem, status FROM INFORMATION_SCHEMA.INNODB_METRICS
       WHERE subsystem ='index';
+-----------------------------+-----------+---------+
| name                        | subsystem | status  |
+-----------------------------+-----------+---------+
| index_page_splits           | index     | enabled | 
| index_page_merge_attempts   | index     | enabled │
| index_page_merge_successful | index     | enabled |
| index_page_reorg_attempts   | index     | enabled |
| index_page_reorg_successful | index     | enabled |
| index_page_discards         | index     | enabled |
+-----------------------------+-----------+---------+
6 rows in set (0.00 sec)
```

Or enter the following line in the [mysqld] section of the MySQL server configuration file:

```bash
[mysqld]
innodb_monitor_enable = module_index
```

2. InnoDB tracks the number of page splits in `INFORMATION_SCHEMA.INNODB_METRICS`. Look for `index_page_splits`:

```bash
$ mysql -uroot -p
...
mysql> select name, count from INFORMATION_SCHEMA.INNODB_METRICS where name like '%split%';
+-------------------+-------+
| name              | count |
+-------------------+-------+
| index_page_splits |     0 |
+-------------------+-------+
1 row in set (0.01 sec)
```