# How to install Zeppelin

## Installation

1. Download [Zeppelin](https://zeppelin.apache.org/download.html):

```bash
$ wget https://mirror.navercorp.com/apache/zeppelin/zeppelin-0.9.0/zeppelin-0.9.0-bin-all.tgz
```

2. Extract the tar package:

```bash
$ tar -xvzf zeppelin-0.9.0-bin-all.tgz
$ mv zeppelin-0.9.0-bin-all zeppelin
```

## Configuration

1. Copy the template files:

```bash
$ cd zeppelin/conf
$ cp zeppelin-env.sh.template zeppelin-env.sh
$ cp zeppelin-site.xml.template zeppelin-site.xml
```

2. Modify the configuration files:

    - `zeppelin-env.sh`:
    
    ```bash
    export JAVA_HOME=/home/mijin/jdk1.8.0_291
    export SPARK_MASTER=spark://xxx.xxx.xxx.xxx:7077
    export ZEPPELIN_PORT=8888
    export SPARK_HOME=/home/mijin/spark
    export HADOOP_CONF_DIR=/home/mijin/hadoop-3.2.2/etc/hadoop
    ```

    - `zeppelin-site.xml`:

    ```bash
    <property>
        <name>zeppelin.server.addr</name>
        <value>0.0.0.0</value>
        <description>Server binding address</description>
    </property>
    <property>
        <name>zeppelin.server.port</name>
        <value>8888</value>
        <description>Server port.</description>
    </property>
    ```

## Start Zeppelin

```bash
$ ./bin/zeppelin-daemon.sh start
```

## Stop Zeppelin

```bash
$ ./bin/zeppelin-daemon.sh stop
```

## Web UI

- http://server01:8888