# How to use lb-mysql on IOD SSD

> The content below is currently being edited, please do not follow it yet.

## to-do

- 정상 테스트 후, IOD 용 `lb-mysql` (`lb-mysql-for-iod`) push 예정 (push 후 이미지 pull/초기 셋업 가이드 추가)
    - 1개 인스턴스는 정상 동작
    - 간단하게 4개 인스턴스를 동시에 돌리는 테스트를 해봤는데, 4 lb-mysql의 경우 CPU-bound 됨. Dell sever에서 다시 test or 다른 워크로드 섞어서 test
    - 로컬에서 4개 인스턴스가 제대로 돌아갔었는데, 해당 이유 확인 필요

## Run 

1. Pull the image:

```bash
$ sudo docker pull meeeejin/lb-mysql-for-iod:latest
Pulling repository registry
...
```

2. Check the downloaded image:

```bash
$ sudo docker images
(수정 필요)
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

4. Create `lb-mysql-for-iod` containers for each IOD namespace using the image. Before running the commands below, the data and log folders **must be empty**:

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

8. Run LinkBench:

```bash
$ cd root/linkbench
$ ./run.sh
```

9. If you used `run.sh`, you could get the following four result files:

- `iostat.txt`: The `iostat` output file 
- `mpstat.txt`: The `mpstat` output file
- `final-stats.csv`: The final summary output file
- `streaming-stats.csv`: The per-thread statistics output file