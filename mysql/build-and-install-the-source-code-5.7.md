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

In MySQL 5.7, the Boost library is required to build MySQL. Therefore, download it first.

```bash
$ cmake -DDOWNLOAD_BOOST=ON -DWITH_BOOST=/path/to/download/boost -DCMAKE_INSTALL_PREFIX=/path/to/dir
```

If you already have the Boost library, change the default installation directory.

```bash
$ cmake -DWITH_BOOST=/path/to/boost -DCMAKE_INSTALL_PREFIX=/path/to/dir
```

Then build and install the source code.
(8: # of cores in your machine)

```bash
$ make -j8 install
```

Initialize tasks that must be performed before the MySQL server, mysqld, is ready to use.
- `--datadir` : the path to the MySQL data directory
- `--basedir` : the path to the MySQL installation directory.

```bash
$ ./bin/mysql_install_db --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
```

or

```bash
$ ./bin/mysqld --initialize --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
```

Reset the root password.

```bash
$ ./bin/mysqld_safe --skip-grant-tables

$ ./bin/mysql -uroot

root:(none)> use mysql;

root:mysql> update user set authentication_string=password('abc') where user='root';
root:mysql> flush privileges;
root:mysql> quit;

$ ./bin/mysql -uroot

root:mysql> set password = password('abc');
root:mysql> quit;
```

Run the MySQL server.

```bash
$ ./bin/mysqld_safe
```
