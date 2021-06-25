# How to install Spark RAPIDS

1. Download the [RAPIDS jars](https://github.com/NVIDIA/spark-rapids/blob/branch-21.08/docs/download.md):

```bash
$ wget https://repo1.maven.org/maven2/com/nvidia/rapids-4-spark_2.12/0.5.0/rapids-4-spark_2.12-0.5.0.jar
$ wget https://repo1.maven.org/maven2/ai/rapids/cudf/0.19.2/cudf-0.19.2-cuda11.jar
```

- cudf-0.19.2-cuda11.jar
- rapids-4-spark_2.12-0.5.0.jar

2. Export the location to these jars:

```bash
export SPARK_RAPIDS_DIR=/home/mijin/sparkRapidsPlugin
export SPARK_CUDF_JAR=${SPARK_RAPIDS_DIR}/cudf-0.19.2-cuda11.jar
export SPARK_RAPIDS_PLUGIN_JAR=${SPARK_RAPIDS_DIR}/rapids-4-spark_2.12-0.5.0.jar
```

3. Install the GPU discovery script:

```bash
$ wget https://github.com/apache/spark/blob/master/examples/src/main/scripts/getGpusResources.sh
```

## Reference

- [Getting Started with RAPIDS Accelerator with on premise cluster or local mode](https://github.com/NVIDIA/spark-rapids/blob/branch-21.06/docs/get-started/getting-started-on-prem.md)