# How to install YCSB for RocksDB

## Prerequisite

- Java:

```bash
$ sudo apt-get install openjdk-8-jdk
$ javac -version
javac 1.8.0_292
$ which javac
/usr/bin/javac
$ readlink -f /usr/bin/javac
/usr/lib/jvm/java-8-openjdk-amd64
$ sudo vi /etc/profile
...
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
$ source /etc/profile
$ echo $JAVA_HOME
/usr/lib/jvm/java-8-openjdk-amd64 
```

- Maven 3:

```bash
$ sudo apt-get install maven
```

## Set up

1. Clone the [YCSB](https://github.com/brianfrankcooper/YCSB) git repository:

```bash
$ git clone https://github.com/brianfrankcooper/YCSB
```

2. Compile:

```bash
$ cd YCSB
$ mvn -pl site.ycsb:rocksdb-binding -am clean package
```

## Run

1. Load the data:

```bash
$ ./bin/ycsb load rocksdb -s -P workloads/workloadb -p rocksdb.dir=/home/mijin/ycsb-rocksdb-data
```

2. Run the workload:

```bash
$ ./bin/ycsb run rocksdb -s -P workloads/workloadb -p rocksdb.dir=/home/mijin/ycsb-rocksdb-data
```

To write the output to a file, you can use the below command:

```bash
$ ./bin/ycsb run rocksdb -s -P workloads/workloadb \
    -p rocksdb.dir=/home/mijin/ycsb-rocksdb-data \
    -threads 8 2>&1 | tee result.dat
```

## RocksDB Configuration Parameters

### Use a Rocksdb option file

Use [`rocksdb.optionfile`](https://github.com/facebook/rocksdb/wiki/RocksDB-Options-File) (e.g., `ycsb-rocksdb-options.ini`)

### Add option parameters to `RocksDBClient.java`

Or add configuration parameters (e.g., `.setUseDirectIoForFlushAndCompaction(true)`) to `RocksDBClient.java` file (`YCSB/rocksdb/src/main/java/site/ycsb/db/rocksdb/RocksDBClient.java`):

```java
private RocksDB initRocksDB() throws IOException, RocksDBException {
    ...
    if(cfDescriptors.isEmpty()) {
      final Options options = new Options()
          .optimizeLevelStyleCompaction()
          .setCreateIfMissing(true)
          .setCreateMissingColumnFamilies(true)
          .setUseDirectReads(true)
          .setUseDirectIoForFlushAndCompaction(true)
          .setIncreaseParallelism(rocksThreads)
          .setMaxBackgroundCompactions(rocksThreads)
          .setInfoLogLevel(InfoLogLevel.INFO_LEVEL);
      dbOptions = options;
      return RocksDB.open(options, rocksDbDir.toAbsolutePath().toString());
    } else {
      final DBOptions options = new DBOptions()
          .setCreateIfMissing(true)
          .setCreateMissingColumnFamilies(true)
          .setUseDirectReads(true)
          .setUseDirectIoForFlushAndCompaction(true)
          .setIncreaseParallelism(rocksThreads)
          .setMaxBackgroundCompactions(rocksThreads)
          .setInfoLogLevel(InfoLogLevel.INFO_LEVEL);
      dbOptions = options;

      final List<ColumnFamilyHandle> cfHandles = new ArrayList<>();
      final RocksDB db = RocksDB.open(options, rocksDbDir.toAbsolutePath().toString(), cfDescriptors, cfHandles);
      for(int i = 0; i < cfNames.size(); i++) {
        COLUMN_FAMILIES.put(cfNames.get(i), new ColumnFamily(cfHandles.get(i), cfOptionss.get(i)));
      }
      return db;
    }
    ...
```

- Refer `java/src/main/java/org/rocksdb/Options.java` and `java/src/main/java/org/rocksdb/DBOptions.java` for details of option functions
- After modifying the code, re-compile YCSB. The result of the parameters in rocksdb will be saved under the `OPTIONS` file in the database path:

```bash
$ mvn -pl site.ycsb:rocksdb-binding -am package
```

## Reference

- [Brian Cooper, "YCSB", GitHub](https://github.com/brianfrankcooper/YCSB)
- [Brian Cooper, "YCSB/RocksDB", GitHub](https://github.com/brianfrankcooper/YCSB/tree/master/rocksdb)
- ["Rocksdb running tutorial in YCSB", ProgrammerSought](https://www.programmersought.com/article/2061668498/)