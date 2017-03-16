# innodb_flush_method

`innodb_flush_method` variable defines the method used to flush data to InnoDB data files and log files. [Detail](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_flush_method)

## fsync(Default)
- `fsync()` to flush both the *data* and *log* files

## O_DSYNC
- `O_SYNC` to open and flush the *log* files
- `fsync()` to flush the *data* files

## O_DIRECT
- `O_DIRECT` to *open the data* files
- `fsync()` to flush both the *data* and *log* files
