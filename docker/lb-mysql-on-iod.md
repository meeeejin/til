# How to use lb-mysql on IOD SSD

1. Pull the [lb-mysql](https://hub.docker.com/r/meeeejin/lb-mysql) image:

```bash
$ sudo docker pull meeeejin/lb-mysql:latest
Pulling repository registry
...
```

2. Check the downloaded image:

```bash
$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
meeeejin/lb-mysql   latest              931bff0ad011        25 hours ago        1.96GB
...
```

3. Create data/log folders for each IOD namespace and mount IOD partitions. Then, change the permission:

```bash
$ mkdir test_data1 test_data2 test_data3 test_data4
$ mkdir test_log/log1 test_log/log2 test_log/log3 test_log/log4
...
(mount)
...
$ sudo chown -R 999:docker test_data* test_log*
```

4. Create `lb-mysql` containers for each IOD namespace using the image. Before running the commands below, the data and log folders **must be empty**:

```bash
[Session 1]
$ sudo docker run -it \
  --name test1 \
  -v /home/osj/test_data1:/var/lib/mysql \
  -v /home/osj/test_log/log1:/var/log/mysql \
  -v /home/osj/cnf:/etc/mysql/conf.d \
  meeeejin/lb-mysql:latest

[Session 2]
$ sudo docker run -it \
  --name test2 \
  -v /home/osj/test_data2:/var/lib/mysql \
  -v /home/osj/test_log/log2:/var/log/mysql \
  -v /home/osj/cnf:/etc/mysql/conf.d \
  meeeejin/lb-mysql:latest

[Session 3]
$ sudo docker run -it \
  --name test3 \
  -v /home/osj/test_data3:/var/lib/mysql \
  -v /home/osj/test_log/log3:/var/log/mysql \
  -v /home/osj/cnf:/etc/mysql/conf.d \
  meeeejin/lb-mysql:latest

[Session 4]
$ sudo docker run -it \
  --name test4 \
  -v /home/osj/test_data4:/var/lib/mysql \
  -v /home/osj/test_log/log4:/var/log/mysql \
  -v /home/osj/cnf:/etc/mysql/conf.d \
  meeeejin/lb-mysql:latest
```

5. Stop all containers for a while to use the backed up data:

```bash
$ sudo docker stop test1 test2 test3 test4
```

6. Assume that the loaded LinkBench data already exists and all database files (data/log) are stored in the `data_backup` directory. Copy the data/log files in `data_backup` and paste them into the data/log folders corresponding to each IOD namespace.

```bash
$ sudo cp -r data_backup/data/* test_data1/ &
$ sudo cp -r data_backup/data/* test_data2/ &
$ sudo cp -r data_backup/data/* test_data3/ &
$ sudo cp -r data_backup/data/* test_data4/ &
$ sudo cp -r data_backup/log/* test_log/log1/ &
$ sudo cp -r data_backup/log/* test_log/log2/ &
$ sudo cp -r data_backup/log/* test_log/log3/ &
$ sudo cp -r data_backup/log/* test_log/log4/ &
```

It would be better to make a script file and put all these commands and run the script file.

7. Start all the stopped containers:

```bash
$ sudo docker start test1 test2 test3 test4
```

8. Run LinkBench in each container:

```bash
$ cd root/linkbench
$ ./run.sh
```

9. If you use `run.sh`, you could get the following four result files:

- `iostat.txt`: The `iostat` output file 
- `mpstat.txt`: The `mpstat` output file
- `final-stats.csv`: The final summary of statistics
- `streaming-stats.csv`: The per-thread statistics