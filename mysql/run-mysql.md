# Run MySQL

how to start and shut down the MySQL server 

## Run the database server

```bash
$ mysqld_safe --defaults-file=/path/to/my.cnf
```

## Connect to the database server

```bash
$ mysql -uroot
```

or

```bash
$ mysqladmin -uroot
```

## Shut down the database server

```bash
$ mysqladmin -uroot shutdown
```
