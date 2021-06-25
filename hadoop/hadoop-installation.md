# How to install Hadoop 3 (with Yarn) on Ubuntu

## Java Installation

1. Download JDK 8 from [Oracle JDK download site](https://www.oracle.com/kr/java/technologies/javase/javase-jdk8-downloads.html) (e.g., `jdk-8u291-linux-x64.tar.gz`)

2. Extract the tar package:

```bash
$ tar -xvzf jdk-8u291-linux-x64.tar.gz
```

3. Add the JDK bin directory to the existing PATH variable:

```bash
$ vi .bash_profile

export JAVA_HOME=/home/mijin/jdk1.8.0_291
export PATH=$JAVA_HOME/bin:$PATH
```

4. Reload `.bash_profile`:

```bash
$ source .bash_profile
```

5. Check the Java version:

```bash
$ java -version
java version "1.8.0_291"
Java(TM) SE Runtime Environment (build 1.8.0_291-b10)
Java HotSpot(TM) 64-Bit Server VM (build 25.291-b10, mixed mode)
```

## Host Configuration

1. Update `/etc/hosts`:

```bash
xxx.xxx.xxx.xxx server01
xxx.xxx.xxx.xxx server02
xxx.xxx.xxx.xxx server03
xxx.xxx.xxx.xxx server04
```

2. Copy the public key of each server to `~/.ssh/authorized_keys`:

```bash
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

3. Synchronize `~/.ssh/authorized_keys` across all servers:

```bash
$ scp ~/.ssh/authorized_keys server02:~/.ssh/authorized_keys
$ scp ~/.ssh/authorized_keys server03:~/.ssh/authorized_keys
$ scp ~/.ssh/authorized_keys server04:~/.ssh/authorized_keys
```

## Hadoop Installation

1. Download [Hadoop](https://hadoop.apache.org/release/3.2.2.html):

```bash
$ wget https://archive.apache.org/dist/hadoop/common/hadoop-3.2.2/hadoop-3.2.2.tar.gz
```

2. Extract the tar package:

```bash
$ tar -xvzf hadoop-3.2.2.tar.gz
```

### Hadoop Configuration

1. `~/.bashrc`:

```bash
export JAVA_HOME=/home/mijin/jdk1.8.0_291
export HADOOP_HOME=/home/mijin/hadoop-3.2.2
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$HADOOP_HOME/lib/native
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

2. `$HADOOP_HOME/etc/hadoop/core-site.xml`:

```bash
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://server01:9000</value>
    </property>
  <property>
      <name>hadoop.tmp.dir</name>
      <value>/nvme/data_01/hadoop/,/nvme/data_02/hadoop/,/nvme/data_03/hadoop/,/nvme/data_04/hadoop/</value>
  </property>
</configuration>
```

3. `$HADOOP_HOME/etc/hadoop/hdfs-site.xml`:

```bash
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/nvme/data_01/hadoop/namenode</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/nvme/data_01/hadoop/data,/nvme/data_02/hadoop/data,/nvme/data_03/hadoop/data,/nvme/data_04/hadoop/data</value>
    </property>
    <property>
        <name>dfs.namenode.http-address</name>
        <value>server01:50070</value>
    </property>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>server01:50090</value>
    </property>
    <property>
        <name>dfs.namenode.secondary.https-address</name>
        <value>server01:50091</value>
    </property>
    <property>
        <name>dfs.permissions</name>
        <value>false</value>
    </property>
</configuration>
```

4. `$HADOOP_HOME/etc/hadoop/yarn-site.xml`:

```bash
<configuration>
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>1013760</value>
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>1024</value>
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>1013760</value>
    </property>
    <property>
        <name>yarn.nodemanager.resource.cpu-vcores</name>
        <value>120</value>
    </property>
    <property>
        <name>yarn.scheduler.maximum-allocation-vcores</name>
        <value>120</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>server01:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>server01:8030</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>server01:8031</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>server01:8088</value>
    </property>
    <property>
        <name>yarn.log-aggregation-enable</name>
        <value>true</value>
    </property>
    <property>
        <name>yarn.nodemanager.local-dirs</name>
        <value>/nvme/data_01/yarn,/nvme/data_02/yarn,/nvme/data_03/yarn,/nvme/data_04/yarn</value>
    </property>
    <property>
        <name>yarn.nodemanager.log-dirs</name>
        <value>/nvme/data_01/yarn/logs</value>
    </property>
    <property>
        <name>yarn.log.server.url</name>
        <value>http://server01:19888/jobhistory/logs</value>
    </property>
    <property>
        <name>yarn.nodemanager.delete.debug-delay-sec</name>
        <value>86400</value>
    </property>
    <property>
        <name>yarn.nodemanager.pmem-check-enabled</name>
        <value>false</value>
    </property>
    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
</configuration>
```

5. `$HADOOP_HOME/etc/hadoop/mapred-site.xml`:

```bash
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>yarn.app.mapreduce.am.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.map.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
    <property>
        <name>mapreduce.reduce.env</name>
        <value>HADOOP_MAPRED_HOME=${HADOOP_HOME}</value>
    </property>
</configuration>
```

6. `$HADOOP_HOME/etc/hadoop/hadoop-env.sh`:

```bash
...
export JAVA_HOME=/home/skt/jdk1.8.0_291
...
```

7. `$HADOOP_HOME/etc/hadoop/workers`:

```bash
server02
server03
server04
```

### Start Hadoop

- Master Node:

```bash
$ hdfs namenode -format -force
$ start-dfs.sh
$ start-yarn.sh

$ jps
762430 ResourceManager
762835 Jps
762001 SecondaryNameNode
761623 NameNode
127006 ZeppelinServer
```

- Worker Node:

```bash
$ jps
614638 DataNode
614928 NodeManager
615216 Jps
```

### Stop Hadoop

- Master Node:

```bash
$ stop-dfs.sh
$ stop-yarn.sh
$ rm -rf $HADOOP_HOME/data/namenode/*
```

- Worker Node:

```bash
$ rm -rf $HADOOP_HOME/data/namenode/*
```

### Web UI

- NameNode (http://server01:50070)
- ResourceManager (http://server01:8088)

## Reference

- [Hadoop 3.2.2](https://hadoop.apache.org/release/3.2.2.html)