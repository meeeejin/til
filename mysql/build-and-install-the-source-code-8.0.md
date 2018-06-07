# Build and install the source code (8.0)

Building MySQL 8.0 from the source code enables you to customize build parameters, compiler optimizations, and installation location.

## Pre-requisites

- libreadline

```bash
$ sudo apt-get install libreadline6 libreadline6-dev
```

- libaio

```bash
$ sudo apt-get install libaio1 libaio-dev
```

- libssl


```bash
$ sudo apt-get install libssl-dev
```

- etc.

```bash
$ sudo apt-get install build-essential cmake libncurses5 libncurses5-dev bison
```

## Build and install

In MySQL 8.0, the Boost library is required to build MySQL. Therefore, download it first.

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
$ ./bin/mysqld --initialize --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
$ ./bin/mysql_ssl_rsa_setup --datadir=/path/to/datadir
```

Reset the root password. First, create a text file containing the password-assignment statement on a single line. Replace the password with the password that you want to use.

```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass';
```

Then, start the MySQL server with the special `--init-file` option:

```bash
$ ./bin/mysqld --init-file=/home/mijin/mysql-init
```

The server executes the contents of the file named by the `--init-file` option at startup, changing the 'root'@'localhost' account password. After the server has started successfully, shut down the server and delete `/home/mijin/mysql-init`.

```bash
$ ./bin/mysqladmin -uroot -pMyNewPass shutdown
```

Run the MySQL server.

```bash
$ ./bin/mysqld_safe
```
