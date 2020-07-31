# How to install Arcus on Ubuntu

Refer the [Arcus repository](https://github.com/naver/arcus) for more details.

## Prerequisite

```bash
# Install JDK & Ant
$ sudo apt-get install openjdk-8-jdk openjdk-8-jre ant

# Install dependencies
$ sudo apt-get install build-essential autoconf automake libtool libcppunit-dev python-setuptools python-dev

# Install Git
sudo apt-get install git
```

## Installation

1. Clone the Arcus repository:

```bash
$ git clone https://github.com/naver/arcus
```

2. Build:

```bash
$ cd arcus/scripts
$ ./build.sh
```

3. Setup a sample local cache cloud with conf file:

```bash
$ ./arcus.sh quicksetup conf/local.sample.json
```

4. Test:

```bash
$ echo "stats" | nc localhost 11211 | grep version
STAT version 1.12.1
```

You can list all registered cache clouds using below commands:

```bash
$ ./arcus.sh memcached listall
Server Roles
        {'zookeeper': ['127.0.0.1'], 'memcached': []}

-----------------------------------------------------------------------------------------------------
serviceCode  status  total  online  offline  created                     modified                  
-----------------------------------------------------------------------------------------------------
test         OK          2       2        0  2020-08-01 00:57:37.812233  2020-08-01 01:14:20.835327

Done.
```

You can stop the memcached using the `serviceCode`:

```bash
$ ./arcus.sh memcached stop test
Server Roles
        {'zookeeper': ['127.0.0.1'], 'memcached': []}

[127.0.0.1] Executing task 'mc_stop_server'
[localhost] local: ps -ef | grep -e memcached | grep -e '-p 11211' | grep -v 'ssh' | awk '{print $2}'
[127.0.0.1] Executing task 'mc_stop_server'
[localhost] local: ps -ef | grep -e memcached | grep -e '-p 11212' | grep -v 'ssh' | awk '{print $2}'

Done.
```

You can unregister the memcached using the `serviceCode`:

```bash
$ ./arcus.sh memcached unregister test                                                                                 
Server Roles
        {'zookeeper': ['127.0.0.1'], 'memcached': []}

!!Caution!!
This will delete an Arcus cluster permanently.
Please type in the following texts to confirm: "Delete test"
>> Delete test
[True, True, True, True, True, True]

Done.
```

You can stop the zookeeper using the below command:

```bash
$ ./arcus.sh zookeeper stop                                                                                            
Server Roles
        {'zookeeper': ['127.0.0.1'], 'memcached': []}

[127.0.0.1] Executing task 'zk_stop'
[localhost] local: bin/zkServer.sh stop
JMX enabled by default
Using config: /home/parallels/arcus/zookeeper/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED

Done.
```