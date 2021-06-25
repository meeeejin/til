# How to run Spark Executor via command line

```bash
$ /home/mijin/jdk1.8.0_291/bin/java -cp /home/mijin/spark/conf/:/home/mijin/spark/jars/*:/home/mijin/hadoop-3.2.2/etc/hadoop/ -Xmx30720M -Dspark.driver.port=46745 -Dspark.sql.catalog.ltdb.port=28200 -Duser.timezone=GMT org.apache.spark.executor.CoarseGrainedExecutorBackend --driver-url spark://CoarseGrainedScheduler@server01:46745 --executor-id 100 --hostname xxx.xxx.xxx.xxx --cores 40 --app-id app-20210624151615-0000 --worker-url spark://Worker@xxx.xxx.xxx.xxx:34111 --resourcesFile resource-executor.json
```

You need to modify below parameters:

- `-Dspark.driver.port=46745`
- `-Dspark.sql.catalog.ltdb.port=28200`
- `spark://CoarseGrainedScheduler@server01:46745`
- `--app-id app-20210624151615-0000`
- `spark://Worker@xxx.xxx.xxx.xxx:34111`