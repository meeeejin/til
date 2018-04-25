# Monitoring InnoDB mutex and lock waits

[Detail](https://dev.mysql.com/doc/refman/5.6/en/monitor-innodb-mutex-waits-performance-schema.html)

1. To ensure that all InnoDB mutex and rw-lock instances are instrumented and enabled, add the following `performance-schema-instrument` rule to your MySQL configuration file:

```bash
performance-schema-instrument='wait/synch/mutex/innodb/%=ON'
performance-schema-instrument='wait/synch/sxlock/innodb/%=ON'
```

2. The following query returns the instrument name (`EVENT_NAME`), the number of wait events (`COUNT_STAR`), and the total wait time for the events for that instrument (`SUM_TIMER_WAIT`). Data is presented in descending order, by the total wait time (`SUM_TIMER_WAIT`).

```bash
mysql> SELECT EVENT_NAME, COUNT_STAR, SUM_TIMER_WAIT/1000000000 SUM_TIMER_WAIT_MS
       FROM performance_schema.events_waits_summary_global_by_event_name
       WHERE SUM_TIMER_WAIT > 0
       AND EVENT_NAME LIKE 'wait/synch/mutex/innodb/%'
       OR EVENT_NAME LIKE 'wait/synch/sxlock/innodb/%'
       ORDER BY SUM_TIMER_WAIT_MS;
```
