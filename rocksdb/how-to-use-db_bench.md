# How to use db_bench

## Pre-requisites

**Linux - Ubuntu**
- Upgrade gcc version at least 4.8
- gflags: `sudo apt-get install libgflags-dev`
  If this doesn't work, here's a nice tutorial:
  (http://askubuntu.com/questions/312173/installing-gflags-12-04)
- snappy: `sudo apt-get install libsnappy-dev`
- zlib: `sudo apt-get install zlib1g-dev`
- bzip2: `sudo apt-get install libbz2-dev`

**[Other platform](https://github.com/facebook/rocksdb/blob/master/INSTALL.md#supported-platforms)**

## Build and install

First, compile RocksDB. [Details](https://github.com/facebook/rocksdb/blob/master/INSTALL.md)
```bash
$ git clone https://github.com/facebook/rocksdb
$ cd rocksdb
$ make
$ make check
```

I installed `gflags` using its github repository. [Details](https://github.com/gflags/gflags/blob/master/INSTALL.md)
```bash
$ git clone https://github.com/gflags/gflags
$ cd gflags
$ mkdir build && cd build
$ ccmake ..
$ make
```

Then, go back to the rocksdb folder and run `db_bench`.
```bash
$ ./db_bench
```

You can see options of `db_bench` using below command.
```bash
$ ./db_bench --help
```
