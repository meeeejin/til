# Build and install the source code (5.7)

Building MySQL 5.7 from the source code enables you to customize build parameters, compiler optimizations, and installation location.

## Pre-requisites

- libreadline

```bash
$ sudo apt-get install libreadline6 libreadline6-dev
```

- libaio

```bash
$ sudo apt-get install libaio1 libaio-dev
```

- etc.

```bash
$ sudo apt-get install build-essential cmake libncurses5 libncurses5-dev bison
```

## Build and install

1. In MySQL 5.7, the Boost library is required to build MySQL. Therefore, download it first:

```bash
$ cmake -DDOWNLOAD_BOOST=ON -DWITH_BOOST=/path/to/download/boost -DCMAKE_INSTALL_PREFIX=/path/to/dir
```

If you already have the Boost library, change the default installation directory:

```bash
$ cmake -DWITH_BOOST=/path/to/boost -DCMAKE_INSTALL_PREFIX=/path/to/dir
```

2. Then, build and install the source code (8: # of cores in your machine):

```bash
$ make -j8 install
```

3. `mysqld --initialize` handles initialization tasks that must be performed before the MySQL server, mysqld, is ready to use:

- `--datadir` : the path to the MySQL data directory (e.g., `/home/mijin/test_data`)
- `--basedir` : the path to the MySQL installation directory (e.g., `/home/mijin/mysql-5.7.24`)


```bash
$ ./bin/mysqld --initialize --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
```

If you want to change the page size to 4K (default: 16K), add the `innodb_page_size` parameter. For example:

```bash
$ ./bin/mysqld --initialize --innodb_page_size=4k --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
```

4. Reset the root password:

```bash
$ ./bin/mysqld_safe --skip-grant-tables

$ ./bin/mysql -uroot

root:(none)> use mysql;

root:mysql> update user set authentication_string=password('abc') where user='root';
root:mysql> flush privileges;
root:mysql> quit;

$ ./bin/mysql -uroot -p

root:mysql> set password = password('abc');
root:mysql> quit;
```

5. Run the MySQL server:

```bash
$ ./bin/mysqld_safe
```

To specify `my.cnf` to use, use the command below:

```bash
$ ./bin/mysqld_safe --defaults-file=/path/to/my.cnf
```

6. Shutdown the MySQL server:

```bash
$ ./bin/mysqladmin -uroot -pabc shutdown
```