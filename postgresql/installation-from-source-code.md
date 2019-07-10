# PostgreSQL installation from source code

Building PostgreSQL from the source code enables you to customize build parameters, compiler optimizations, and installation location.

- [Reference](https://www.postgresql.org/docs/11/installation.html)

## Get the source

You can download the sources from the [website](https://www.postgresql.org/download). Then, unpack it:

```bash
$ tar xf postgresql-11.4.tar
```

## Build and install

First, make a directory for build and configure the source tree for your system.

```bash
$ mkdir build_dir
$ ./configure --prefix=/path/to/build_dir
```

Then build and install the source code.
(8: # of cores in your machine)

```bash
$ make -j8 install
```

You need to tell the system how to find the newly installed shared libraries. Set the shared library search path in `~/.bashrc`:

```bash
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/path/to/build_dir/lib
export PATH=/path/to/build_dir/bin:$PATH
```

Then, apply the modified path:

```bash
$ source ~/.bashrc
```

## Start the database server

Before you can do anything, you must initialize a database storage area on disk. To initialize a database cluster, use the command `initdb`, which is installed with PostgreSQL. The desired file system location of your database cluster is indicated by the `-D` option:

```bash
$ initdb -D /path/to/data_dir
```

Then, start the server with a log file:

```bash
$ pg_ctl -D /path/to/data_dir -l logfile start
```

You can stop the server using below command:

```bash
$ pg_ctl -D /path/to/data_dir -m smart stop
```