# How to install LinkBench on Ubuntu

- Reference: [LinkBench GitHub](https://github.com/facebookarchive/linkbench)

## Prerequisites

- Java

1. Install the latest [JDK](https://www.oracle.com/java/technologies/javase-jdk13-downloads.html).

2. Move and decompress the downloaded file:

```bash
$ cd /usr/local
$ sudo mkdir java
$ cd java
$ sudo mv jdk-13.0.2_linux-x64_bin.tar.gz .
$ sudo tar -zxvf jdk-13.0.2_linux-x64_bin.tar.gz
```

3. (Optional) Register the java executables:

```bash
$ sudo update-alternatives --install /usr/bin/java java /usr/local/java/jdk-13.0.2/bin/java 1
$ sudo update-alternatives --install /usr/bin/javac javac /usr/local/java/jdk-13.0.2/bin/javac 1
$ sudo update-alternatives --install /usr/bin/javaws javaws /usr/local/java/jdk-13.0.2/bin/javaws 1
```

If another version of Java is already installed, select the version of Java to use using below commands:

```bash
$ sudo update-alternatives --config java
$ sudo update-alternatives --config javac
$ sudo update-alternatives --config javaws
```

4. Add the three lines below (for setting the environment variables) to `/etc/profile` file:

```bash
$ sudo vi /etc/profile
...
export JAVA_HOME=/usr/local/java/jdk-13.0.2
export JRE_HOME=$JAVA_HOME/jre
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin

$ . /etc/profile
```

5. Check the version of Java:

```bash
$ java -version
java version "13.0.2" 2020-01-14
Java(TM) SE Runtime Environment (build 13.0.2+8)
Java HotSpot(TM) 64-Bit Server VM (build 13.0.2+8, mixed mode, sharing)
```

- Maven

```bash
$ sudo apt-get install maven
```

- MySQL Connector / Server

## Installation

1. Clone the source code:

```bash
$ git clone git@github.com:facebook/linkbench.git
```

2. Build LinkBench (skip all tests):

```bash
$ cd linkbench
$ mvn clean package -DskipTests
```

3. Start the MySQL server.

```bash
$ ./bin/mysqld_safe --defaults-file=my.cnf
```

4. Create a new database called `linkdb` and the needed tables:

```bash
$ ./bin/mysql -uroot -pxxxx

mysql> create database linkdb;

mysql> use linkdb;

mysql> CREATE TABLE `linktable` (
  `id1` bigint(20) unsigned NOT NULL DEFAULT '0',
  `id2` bigint(20) unsigned NOT NULL DEFAULT '0',
  `link_type` bigint(20) unsigned NOT NULL DEFAULT '0',
  `visibility` tinyint(3) NOT NULL DEFAULT '0',
  `data` varchar(255) NOT NULL DEFAULT '',
  `time` bigint(20) unsigned NOT NULL DEFAULT '0',
  `version` int(11) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (link_type, `id1`,`id2`),
  KEY `id1_type` (`id1`,`link_type`,`visibility`,`time`,`id2`,`version`,`data`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1 PARTITION BY key(id1) PARTITIONS 16;

CREATE TABLE `counttable` (
  `id` bigint(20) unsigned NOT NULL DEFAULT '0',
  `link_type` bigint(20) unsigned NOT NULL DEFAULT '0',
  `count` int(10) unsigned NOT NULL DEFAULT '0',
  `time` bigint(20) unsigned NOT NULL DEFAULT '0',
  `version` bigint(20) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`,`link_type`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

CREATE TABLE `nodetable` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `type` int(10) unsigned NOT NULL,
  `version` bigint(20) unsigned NOT NULL,
  `time` int(10) unsigned NOT NULL,
  `data` mediumtext NOT NULL,
  PRIMARY KEY(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

5. Make a copy of the example config file:

```bash
$ cp config/LinkConfigMysql.properties config/MyConfig.properties
```

Open `MyConfig.properties` and change the MySQL connection information. For example:

```bash
# MySQL connection information
host = localhost
user = root
password = xxxx
port = 3306
dbid = linkdb
```

If you want to change the scale of the benchmark, open `FBWorkload.properties` and increase the value of `maxid1` to get a larger database:

```bash
  # start node id (inclusive)
  startid1 = 1

  # end node id for initial load (exclusive)
  # With default config and MySQL/InnoDB, 1M ids ~= 1GB
  maxid1 = 10000001
```

6. Load the data:

```bash
$ ./bin/linkbench -c config/MyConfig.properties -l
```

7. Run the request phase:

```bash
$ ./bin/linkbench -c config/MyConfig.properties -r
```

LinkBench supports output of statistics in csv format for easier analysis. There are two categories of statistic: the final summary and per-thread statistics output periodically through the benchmark. -csvstats controls the former and -csvstream the latter:

```bash
$ ./bin/linkbench -c config/MyConfig.properties -csvstats final-stats.csv -csvstream streaming-stats.csv -r
```