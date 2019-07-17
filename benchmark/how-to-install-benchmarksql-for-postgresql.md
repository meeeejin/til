# How to install BenchmarkSQL for PostgreSQL

1. Install [JDK 1.8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html).

2. Decompression the downloaded file.

```bash
$ cd Downloads
$ tar -zxvf jdk-8u212-linux-x64.tar.gz
```

3. Added the two lines below (for setting the environment variables) to `~/.bashrc` file.

```bash
export JAVA_HOME=/home/mijin/Downloads/jdk1.8.0_211
export PATH=$JAVA_HOME/bin:$PATH
```

4. Check the version of Java.

```bash
$ java -version
java version "1.8.0_211"
Java(TM) SE Runtime Environment (build 1.8.0_211-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.211-b12, mixed mode)
```

5. Download [BenchmarkSQL](https://sourceforge.net/projects/benchmarksql/).

6. Unzip the downloaded file.

```bash
$ unzip benchmarksql-5.0.zip
$ cd benchmarksql-5.0
```

7. If you don't have `ant` package in your Ubuntu, download it first (`sudo apt-get install ant`). Then, compile the source code.

```bash
$ sudo apt-get install ant
$ ant
```

8. Create a database user and a database.

```bash
$ psql postgres                                                                                                       ...

postgres=# CREATE USER benchmarksql WITH ENCRYPTED PASSWORD 'changeme';
postgres=# CREATE DATABASE benchmarksql OWNER benchmarksql;
postgres=# \q
```

9. Modify the configuration file for BenchmarkSQL.

```bash
$ cd /home/mijin/Downloads/benchmarksql-5.0/run
$ cp props.ora my_postgres.properties
$ vi my_postgres.properties
...
```

Change each parameter for your preference. Please change **host name** or **port number** according to your environment.

10. Build the schema and load initial database.

```bash
$ ./runDatabaseBuild.sh my_postgres.propertie
```

11. Start running.

```bash
$ ./runBenchmark.sh my_postgres.properties
```