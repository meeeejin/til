# How to install Spark 3 on Ubuntu

## Spark Installation

1. Download [Spark 3](https://spark.apache.org/downloads.html):

```bash
$ wget https://www.apache.org/dyn/closer.lua/spark/spark-3.1.2/spark-3.1.2-bin-hadoop3.2.tgz
```

2. Extract the tar package:

```bash
$ tar -xvzf spark-3.1.2-bin-hadoop3.2.tgz
$ mv spark-3.1.2-bin-hadoop3.2 spark
```

## Spark Configuration

1. `~/.bash_profile`:

```bash
...
export SPARK_HOME=/home/mijin/spark
export PATH=$PATH:$SPARK_HOME/bin
```

2. `$SPARK_HOME/conf/workers`:

```bash
server02
server03
server04
```

3. `$SPARK_HOME/conf/spark-env.sh`:

```bash
export JAVA_HOME=/home/mijin/jdk1.8.0_291
export SPARK_MASTER_HOST=server01
export HADOOP_HOME=/home/mijin/hadoop-3.2.2
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export SPARK_WORKER_CORES=1000
```

4. `$SPARK_HOME/conf/spark-defaults.conf` (Running Spark on YARN):

```bash
spark.master yarn

spark.scheduler.mode                    FAIR
spark.driver.memory                     10g
spark.driver.maxResultSize              1g
spark.local.dir                         /nvme/data_01
hive.server2.thrift.port       13100

spark.memory.offHeap.size 1g
spark.memory.offHeap.enabled true

spark.io.compression.codec lz4
spark.rdd.compress true

spark.executor.cores 40
spark.executor.instances 16

spark.ui.enabled true
spark.ui.retainedJobs 20
spark.ui.retainedStages 20
spark.ui.retainedTasks 1000000

spark.worker.ui.retainedExecutors 100
spark.worker.ui.retainedDrivers 100
spark.sql.ui.retainedExecutions 100

spark.eventLog.enabled true
spark.eventLog.dir file:///home/mijin/spark/eventLog

spark.history.fs.logDirectory file:///nvme/data_01/spark/event-logs

spark.sql.shuffle.partitions 250
spark.sql.hive.thriftServer.singleSession true
spark.local.dir /nvme/data_01
```

## Start Spark

- Master Node:

```bash
$ $SPARK_HOME/sbin/start-all.sh

$ jps
763226 Master
```

(Optional)

```bash
$ $SPARK_HOME/sbin/start-history-server.sh
```

- Worker node:

```bash
$ jps
615419 Worker
```

## Stop Spark

- Master Node:

```bash
$ $SPARK_HOME/sbin/stop-all.sh
```

(Optional)

```bash
$ $SPARK_HOME/sbin/stop-history-server.sh
$ rm -rf $SPARK_HOME/eventLog/*
```

## Web UI

- Spark Context: http://server01:4040
- Spark Master: http://server01:8080
- Spark Worker: e.g., http://server02:8081
- Spark History Server: http://server01:18080

## Reference

- [Spark Configuration](https://spark.apache.org/docs/latest/configuration.html)
- [Running Spark on YARN](https://spark.apache.org/docs/latest/running-on-yarn.html)
- [ltdb-for-AWS: Spark Configuration Files](https://github.com/mnms/ltdb-for-AWS/tree/master/conf/spark)