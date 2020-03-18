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

1. First, extract the tar file and make a directory for building the source code:

```bash
$ tar zxvf mysql-VERSION.tar.gz
$ cd mysql-VERSION
$ mkdir bld
$ cd bld
```

2. In MySQL 8.0, the Boost library is required to build MySQL. Therefore, download it first:

```bash
$ cmake .. -DDOWNLOAD_BOOST=ON -DWITH_BOOST=/path/to/download/boost -DCMAKE_INSTALL_PREFIX=/path/to/dir/bld
```

If you already have the Boost library, change the default installation directory.

```bash
$ cmake .. -DWITH_BOOST=/path/to/boost -DCMAKE_INSTALL_PREFIX=/path/to/dir/bld
```

3. Then build and install the source code (8: # of cores in your machine):

```bash
$ make -j8 install
```

4. Initialize tasks that must be performed before the MySQL server, mysqld, is ready to use:

- `--datadir` : the path to the MySQL data directory (e.g., `/home/mijin/test_data`)
- `--basedir` : the path to the MySQL installation directory (e.g., `/home/mijin/mysql-8.0.15`)

```bash
$ ./bin/mysqld --initialize --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
$ ./bin/mysql_ssl_rsa_setup --datadir=/path/to/datadir
```

If you want to change the page size to 4K (default: 16K), add the `innodb_page_size` parameter. For example:

```bash
$ ./bin/mysqld --initialize --innodb_page_size=4k --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
$ ./bin/mysql_ssl_rsa_setup --datadir=/path/to/datadir
```

5. Reset the root password. First, create a text file containing the password-assignment statement on a single line. Replace the password with the password that you want to use:

```bash
ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass';
```

Then, start the MySQL server with the special `--init-file` option:

```bash
$ ./bin/mysqld --datadir=/path/to/datadir --init-file=/home/mijin/mysql-init
```

The server executes the contents of the file named by the `--init-file` option at startup, changing the 'root'@'localhost' account password.

6. After the server has started successfully, shut down the server and delete `/home/mijin/mysql-init`:

```bash
$ ./bin/mysqladmin -uroot -pMyNewPass shutdown
```

7. Run the MySQL server.

```bash
$ ./bin/mysqld
```

8. Shutdown the MySQL server:

```bash
$ ./bin/mysqladmin -uroot -pMyNewPass shutdown
```
