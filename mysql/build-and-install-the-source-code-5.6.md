# Build and install the source code (5.6)

Building MySQL 5.6 from the source code enables you to customize build parameters, compiler optimizations, and installation location.

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

1. Download the source code of MySQL 5.6 Community Server using `wget`:

```bash
$ wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.6.26.tar.gz
```

2. Extract the `mysql-5.6.26.tar.gz` file:

```bash
$ tar -xvzf mysql-5.6.26.tar.gz
$ cd mysql-5.6.26
```

3. Then build and install the source code:
(8: # of cores in your machine)

```bash
$ cmake -DCMAKE_INSTALL_PREFIX=/path/to/basedir
$ make -j8 install
```

For example:

```bash
$ cmake -DCMAKE_INSTALL_PREFIX=/home/mijin/mysql-5.6.26
$ make -j8 install
```

4. Initialize the data directory, including the *mysql* database:
- `--datadir` : the path to the MySQL data directory
- `--basedir` : the path to the MySQL installation directory

```bash
$ cd scripts
$ sudo chmod +x mysql_install_db
$ ./mysql_install_db --user=mysql --datadir=/path/to/datadir --basedir=/path/to/basedir
```

For example:

```bash
$ ./mysql_install_db --user=mysql --datadir=/home/mijin/test-data --basedir=/home/mijin/mysql-5.6.26
```

5. Open `.bashrc` and add MySQL installation path to your path:

```bash
$ vi ~/.bashrc

export PATH=/path/to/basedir/bin:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/basedir/lib/

$ source ~/.bashrc
```

For example:

```bash
export PATH=/home/mijin/mysql-5.6.26/bin:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/mijin/mysql-5.6.26/lib/
```

6. Create a configuration file (`my.cnf`) for the MySQL server. For example, create a `my.cnf` in your local directory and copy the below texts to your `my.cnf`. You should modify the path `/path/to/datadir` to your local path. And if you separate the log device from the data device, uncomment `innodb_log_group_home_dir=/path/to/logdir/` and correct the path of the log directory:

```bash
$ vi my.cnf

#
# The MySQL database server configuration file
#
[client]
user    = root
port    = 3306
socket  = /tmp/mysql.sock

[mysql]
prompt  = \u:\d>\_

[mysqld_safe]
socket  = /tmp/mysql.sock

[mysqld]
# Basic settings
default-storage-engine = innodb
pid-file        = /path/to/datadir/mysql.pid
socket          = /tmp/mysql.sock
port            = 3306
datadir         = /path/to/datadir/
log-error       = /path/to/datadir/mysql_error.log

#
# Innodb settings
#
# Page size
innodb_page_size=16KB

# Buffer pool settings
innodb_buffer_pool_size=1G
innodb_buffer_pool_instances=8

# Transaction log settings
innodb_log_file_size=100M
innodb_log_files_in_group=2
innodb_log_buffer_size=32M

# Log group path (iblog0, iblog1)
# If you separate the log device, uncomment and correct the path
#innodb_log_group_home_dir=/path/to/logdir/

# Flush settings (SSD-optimized)
# 0: every 1 seconds, 1: fsync on commits, 2: writes on commits
innodb_flush_log_at_trx_commit=0
innodb_flush_neighbors=0
innodb_flush_method=O_DIRECT
```

For details on each variable, refer to [MySQL 5.6 Server System Variable](https://dev.mysql.com/doc/refman/5.6/en/server-system-variable-reference.html)

7. You can start the MySQL server using the below command:

```bash
$ ./bin/mysqld_safe --defaults-file=/path/to/my.cnf
```

8. You can shut down the server using the below command:

```bash
$ ./bin/mysqladmin -uroot shutdown
```
