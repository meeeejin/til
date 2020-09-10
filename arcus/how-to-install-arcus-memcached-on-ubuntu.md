# How to install Arcus-memcached on Ubuntu 16.04

Refer the [Arcus-memcached repository](https://github.com/naver/arcus-memcached) for more details.

## Prerequisite

```bash
# Install JDK & Ant
$ sudo apt-get install openjdk-8-jdk openjdk-8-jre ant

# Install dependencies
$ sudo apt-get install build-essential autoconf automake libtool libcppunit-dev python-setuptools python-dev libevent-dev

# Install Git
sudo apt-get install git
```

## Installation

1. Clone the Arcus repository:

```bash
$ git clone https://github.com/naver/arcus-memcached
```

2. Build:

```bash
$ cd arcus-memcached
$ ./config/autorun.sh
$ ./configure --with-libevent=<libevent_install_path>
$ make
$ make install
```

3. Run:

```bash
$ ./memcached -E .libs/default_engine.so
```

You can specify the logger module using `-X` and the configuration file using `-e`:

```bash
$ ./memcached -E .libs/default_engine.so -X .libs/syslog_logger.so -e config_file=engines/default/default_engine.conf
```

Also, you can enable the **persistence** option by building the [persistence](https://github.com/naver/arcus-memcached/tree/persistence) branch and specifying the conf file (`-e config_file=engines/default/default_engine.conf`). Change the value of `use_persistence` to `true` and update `data_path` and `logs_path` for your environment:

- `data_path`: the path to the ARCUS data directory (e.g., `/home/mijin/arcus-data`)
- `logs_path`: the path to the ARCUS log directory (e.g., `/home/mijin/arcus-log`)

> `engines/default/default_engine.conf`
```bash
# This is a default engine config file

# collection max size (default : 50000)
# The collection max size limits the maximum number of elements that can be stored
# in each collection item. Its hard limit is 1000000.
#max_list_size=50000
#max_set_size=50000
#max_map_size=50000
#max_btree_size=50000

#
# Persistence configuration
#
# use persistence (true or false, default: false)
use_persistence=true
#
# The path of the snapshot file (default: ARCUS-DB)
data_path=/path/to/arcus-data
#
# The path of the command log file (default: ARCUS-DB)
logs_path=/path/to/arcus-log
#
# asynchronous logging
#async_logging=true
#
# checkpoint interval (unit: percentage, default: 100)
# The ratio of the command log file size to the snapshot file size.
# 100 means checkpoint if snapshot file size is 10GB, command log file size is 20GB or more
chkpt_interval_pct_snapshot=100
#
# checkpoint interval minimum file size (unit: MB, default: 256)
chkpt_interval_min_logsize=256
```

To see details on arcus-memcached start options, run memcached with `-h` option like below:

```bash
$ ./memcached -h
arcus-memcached 1.12.1-451
-p <num>      TCP port number to listen on (default: 11211)
-U <num>      UDP port number to listen on (default: 11211, 0 is off)
-s <file>     UNIX socket path to listen on (disables network support)
-a <mask>     access mask for UNIX socket, in octal (default: 0700)
-l <ip_addr>  interface to listen on (default: INADDR_ANY, all addresses)
-d            run as a daemon
-r            maximize core file limit
-u <username> assume identity of <username> (only when run as root)
-m <num>      max memory to use for items in megabytes (default: 64 MB)
-M            return error on memory exhausted (rather than removing items)
-g            sticky(gummed) memory limit in megabytes (default: 0 MB)
-c <num>      max simultaneous connections (default: 1024)
-k            lock down all paged memory.  Note that there is a
              limit on how much memory you may lock.  Trying to
              allocate more than that would fail, so be sure you
              set the limit correctly for the user you started
              the daemon with (not for -u <username> user;
              under sh this is done with 'ulimit -S -l NUM_KB').
-v            verbose (print errors/warnings while in event loop)
-vv           very verbose (also print client commands/reponses)
-vvv          extremely verbose (also print internal state transitions)
-h            print this help and exit
-i            print memcached and libevent license
-P <file>     save PID in <file>, only used with -d option
-f <factor>   chunk size growth factor (default: 1.25)
-n <bytes>    minimum space allocated for key+value+flags (default: 48)
-L            Try to use large memory pages (if available). Increasing
              the memory page size could reduce the number of TLB misses
              and improve the performance. In order to get large pages
              from the OS, memcached will allocate the total item-cache
              in one large chunk.
-D <char>     Use <char> as the delimiter between key prefixes and IDs.
              This is used for per-prefix stats reporting. The default is
              ":" (colon). If this option is specified, stats collection
              is turned on automatically; if not, then it may be turned on
              by sending the "stats detail on" command to the server.
-t <num>      number of threads to use (default: 4)
-R            Maximum number of requests per event, limits the number of
              requests process for a given connection to prevent 
              starvation (default: 20)
-C            Disable use of CAS
-b            Set the backlog queue limit (default: 1024)
-B            Binding protocol - one of ascii, binary, or auto (default)
-I            Override the size of each slab page. Adjusts max item size
              (default: 1mb, min: 1k, max: 128m)
-E <engine>   Engine to load, must be given (for example, -E .libs/default_engine.so)
-q            Disable detailed stats commands
-X module,cfg Load the module and initialize it with the config
-O ip:port    Tap ip:port

Environment variables:
MEMCACHED_PORT_FILENAME   File to write port information to
MEMCACHED_TOP_KEYS        Number of top keys to keep track of
```

4. Test:

```bash
$ make test
./sizes
Thread stats    1000
Global stats    64
Settings        192
Libevent thread 296
Connection      24272
----------------------------------------
libevent thread cumulative      296
Thread stats cumulative         1000
./testapp
1..49
ok 1 - cache_create
ok 2 - cache_constructor
ok 3 - cache_constructor_fail
ok 4 - cache_destructor
ok 5 - cache_reuse
...
./t/unixsocket.t .................. ok                                  
===(  142154;281  1867/2846  1/8 )======================================memcached shutdown by signal(SIGINT)
Main thread is now terminating from clock handler.
Initiating arcus memcached shutdown...
Listen sockets closed.
Worker thread[0] is now terminating from libevent process.
Worker thread[1] is now terminating from libevent process.
Worker thread[2] is now terminating from libevent process.
Worker thread[3] is now terminating from libevent process.
Worker threads terminated.
SNAPSHOT module destroyed.
ITEM change log module destroyed.
ITEM module destroyed.
SLABS module destroyed.
ASSOC module destroyed.
PREFIX module destroyed.
Memcached engine destroyed.
Arcus memcached terminated.
./t/verbosity.t ................... ok                                  
./t/whitespace.t .................. ok                                  
./t/longkey.t ..................... ok         
All tests successful.
Files=91, Tests=143407, 303 wallclock secs (11.67 usr  1.15 sys + 32.73 cusr  4.77 csys = 50.32 CPU)
Result: PASS
```
