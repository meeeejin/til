# How to install BenchmarkSQL for Oracle

- [Reference: Oracle 18c single instance BenchmarkSQL running test](http://www.programmersought.com/article/5175664665)

1. Install [JDK 1.8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html).

2. Decompress the downloaded file.

```bash
$ cd Downloads
$ tar -zxvf jdk-8u212-linux-x64.tar.gz
```

3. Add the two lines below (for setting the environment variables) to `~/.bashrc` file.

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

7. If you don't have `ant` package in your Ubuntu, download it first. Then, compile the source code.

```bash
$ sudo apt-get install ant
$ ant
```

8. Copy 18c drive to benchmark.

```bash
$ cp /u01/app/oracle/product/12.2.0/dbhome_1/jdbc/lib/ojdbc8.jar /home/mijin/Downloads/benchmarksql-5.0/lib/oracle/ojdbc8.jar
```

9. Create a database user.

```bash
$ sqlplus / as sysdba
...

SQL> create user benchmarksql identified by "benchmarksql";
...

SQL> grant dba, connect to benchmarksql;
...

SQL> alter user benchmarksql default tablespace users;
...
```

10. Modify the configuration file for BenchmarkSQL.

```bash
$ cd /home/mijin/Downloads/benchmarksql-5.0/run
$ cp props.ora myprops.ora
$ vi myprops.ora
...
```

Change each parameter for your preference. Please change **host name** or **port number** according to your environment.

11. Create tables.

```bash
$ ./runSQL.sh myprops.ora ./sql.common/tableCreates.sql 
```

> You can skip step 11~13 by just executing the `runDatabaseBuild.sh` script with your configuration file.

12. Generate test data with 200 warehouse.

```bash
$ ./runLoader.sh myprops.ora numWarehouses 200
```

13. Create indices and finish build.

```bash
$ ./runSQL.sh myprops.ora ./sql.common/indexCreates.sql
$ ./runSQL.sh myprops.ora ./sql.common/foreignKeys.sql
$ ./runSQL.sh myprops.ora ./sql.oracle/extraHistID.sql
$ ./runSQL.sh myprops.ora ./sql.common/buildFinish.sql
```

14. Start running.

```bash
$ ./runBenchmark.sh myprops.ora
```