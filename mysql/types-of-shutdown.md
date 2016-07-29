# Types of shutdown

There are two types of shutdown which is the process of stopping the MySQL server. The shutdown mode is controlled by the `innodb_fast_shutdown` option.

## Fast shutdown

The **default** shutdown procedure for InnoDB, based on the configuration setting `innodb_fast_shutdown=1`. To save time, certain flush operations are skipped. The flush operations are performed during the next startup, using the same mechanism as in **crash recovery**.

## Slow shutdown

It does additional InnoDB flushing operations before completing. Also known as a **clean shutdown**, based on the configuration setting `innodb_fast_shutdown=0` or the command `SET GLOBAL innodb_fast_shutdown=0;`. Although the shutdown itself can take longer, that time will be saved on the subsequent startup.
