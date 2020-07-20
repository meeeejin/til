# MariaDB S3 Engine: Implementation and Benchmarking

## Reference

- [MariaDB S3 Engine: Implementation and Benchmarking - Percona Database Performance Blog](https://www.percona.com/blog/2020/07/17/mariadb-s3-engine-implementation-and-benchmarking/?utm_campaign=Blog%202016%3A%20Subscribers%20To%20Blog%20Weekly%20Recap%20Email%20--%202.15.16&utm_medium=email&_hsmi=91587027&_hsenc=p2ANqtz--lvNYTuI99Xf9uhQNOnF4x1lypdUW2NWySeNQUIZ-3TYSY7siLHTNxc7wqxkpDYNF42s11K4J0xqddaLn6I_lDdD7TUw&utm_content=91587027&utm_source=hs_email)
- [The S3 storage engine in MariaDB 10.5](https://mariadb.org/wp-content/uploads/2020/02/S3.pdf)

## Overview

- MariaDB 10.5부터 S3 스토리지 엔진 플러그인 제공
- S3 엔진은 Aria 스토리지 엔진 기반으로 동작
  - **READ_ONLY**로, write operation은 수행 불가. 테이블 구조 변경은 가능
  - Perfect for **inexpensive archiving** of old data
  - MariaDB 테이블을 AWS S3에 보관하거나 S3 API를 이용해서 타사 공용/사설 클라우드의 데이터를 MariaDB에서 읽을 수 있음
  - `ALTER`로 로컬 디바이스에 있는 테이블을 AWS로 바로 옮길 수 있음
  - S3 엔진은 전용 페이지 캐시를 따로 가짐

## S3 Engine Implementation

S3 엔진은 alpha-level maturity이기 때문에 default로는 로드되지 않음. 따라서 conf 파일에 아래와 같이 명시해 S3 엔진을 enable 시킴:

```bash
[mysqld]
plugin-maturity = alpha
```

S3 버킷과 연결하기 위해, S3 credential 또한 conf 파일에 명시해야 함: 

```bash
#s3
s3=ON
s3-host-name=s3.amazonaws.com
s3-bucket=mariadb
s3-access-key=xxxx
s3-secret-key=xxx
s3-region=eu-north-1
#s3-slave-ignore-updates=1
#s3-pagecache-buffer-size=256M
```

파라미터 설정 후 재시작해 변경 사항 반영하면 아래처럼 플러그인이 설치됨:

```sql
MariaDB [(none)]> install soname 'ha_s3';
Query OK, 0 rows affected (0.000 sec)
 
MariaDB [(none)]> select * from information_schema.engines where engine = 's3'\G
*************************** 1. row ***************************
      ENGINE: S3
     SUPPORT: YES
     COMMENT: Read only table stored in S3. Created by running ALTER TABLE table_name ENGINE=s3
TRANSACTIONS: NO
          XA: NO
  SAVEPOINTS: NO
1 row in set (0.000 sec)
```

## How Do I Move The Table to The S3 Engine?

테스트를 위해, `percona_s3` 테이블 생성:

```sql
MariaDB [s3_test]> show create table percona_s3\G
*************************** 1. row ***************************
       Table: percona_s3
Create Table: CREATE TABLE `percona_s3` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(16) DEFAULT NULL,
  `c_date` datetime DEFAULT current_timestamp(),
  `date_y` datetime DEFAULT current_timestamp(),
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.000 sec)
 
[root@ip-172-31-19-172 ~]# ls -lrth /var/lib/mysql/s3_test/* | grep -i percona_s3
-rw-rw----  1 mysql mysql 1019 Jul 14 01:50 /var/lib/mysql/s3_test/percona_s3.frm
-rw-rw----  1 mysql mysql  96K Jul 14 01:50 /var/lib/mysql/s3_test/percona_s3.ibd
```

Default로 InnoDB를 사용하기 때문에, *.frm* 및 *.ibd* 파일이 자동으로 생성됨. 이제 `percona_s3` 테이블을 S3 엔진으로 convert:

```sql
MariaDB [s3_test]> alter table percona_s3 engine=s3;
Query OK, 0 rows affected (1.934 sec)              
Records: 0  Duplicates: 0  Warnings: 0

MariaDB [s3_test]> show create table percona_s3\G
*************************** 1. row ***************************
       Table: percona_s3
Create Table: CREATE TABLE `percona_s3` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(16) DEFAULT NULL,
  `c_date` datetime DEFAULT current_timestamp(),
  `date_y` datetime DEFAULT current_timestamp(),
  PRIMARY KEY (`id`)
) ENGINE=S3 DEFAULT CHARSET=latin1 PAGE_CHECKSUM=1
1 row in set (1.016 sec)

[root@ip-172-31-19-172 ~]# ls -lrth /var/lib/mysql/s3_test/* | grep -i percona_s3
-rw-rw----  1 mysql mysql 1015 Jul 14 01:54 /var/lib/mysql/s3_test/percona_s3.frm
```

S3 엔진으로 변환이 끝나면, *.frm* 파일만 남은 것을 확인할 수 있음. 즉, InnoDB 스토리지 포맷에서 S3 엔진 스토리지 포맷으로 데이터가 변환된 것을 확인 가능. S3는 데이터와 인덱스 페이지를 따로 저장하므로 아래와 같이 다른 폴더에 저장됨:

```sql
[root@ip-172-31-19-172 ~]# aws s3 ls s3://mariabs3plugin/s3_test/percona_s3/
                           PRE data/
                           PRE index/
2020-07-14 01:59:28       8192 aria
2020-07-14 01:59:28       1015 frm

[root@ip-172-31-19-172 ~]# aws s3 ls s3://mariabs3plugin/s3_test/percona_s3/data/
2020-07-14 01:59:29      16384 000001
[root@ip-172-31-19-172 ~]# aws s3 ls s3://mariabs3plugin/s3_test/percona_s3/index/
2020-07-14 01:59:28       8192 000001
```

- frm 파일: *s3_bucket/database/table/frm*
  - e.g., *s3://mariabs3plugin/s3_test/percona_s3/frm*
- 첫번째 index block: *s3_bucket/database/table/aria*
  - e.g., *s3://mariabs3plugin/s3_test/percona_s3/aria*
- 나머지 index block 파일: *s3_bucket/database/table/index/block_number*
  - e.g., *s3://mariabs3plugin/s3_test/percona_s3/index/000001*
- Data 파일: *s3_bucket/database/table/data/block_number*
  - e.g., *s3://mariabs3plugin/s3_test/percona_s3/data/000001*

## S3 Engine Operation

테스트를 위해, 아래처럼 `percona_s3` 테이블 생성:

```sql
MariaDB [s3_test]> select * from percona_s3;
+----+-----------------+---------------------+---------------------+
| id | name            | c_date              | date_y              |
+----+-----------------+---------------------+---------------------+
|  1 | hercules7sakthi | 2020-06-28 21:47:27 | 2020-07-01 14:37:13 |
+----+-----------------+---------------------+---------------------+
1 row in set (1.223 sec)

MariaDB [s3_test]> pager grep -i engine ; show create table percona_s3;
PAGER set to 'grep -i engine'
) ENGINE=S3 AUTO_INCREMENT=2 DEFAULT CHARSET=latin1 PAGE_CHECKSUM=1 |
1 row in set (0.798 sec)
```

### S3 Engine with INSERT/UPDATE/DELETE:

S3 엔진은 read-only이므로, 이 세개의 statement에 대해 모두 error return:

```sql
MariaDB [s3_test]> insert into percona_s3 (name) values ('anti-hercules7sakthi');
ERROR 1036 (HY000): Table 'percona_s3' is read only
```

### S3 Engine with SELECT:

```sql
MariaDB [s3_test]> select * from percona_s3;
+----+-----------------+---------------------+---------------------+
| id | name            | c_date              | date_y              |
+----+-----------------+---------------------+---------------------+
|  1 | hercules7sakthi | 2020-06-28 21:47:27 | 2020-07-01 14:37:13 |
+----+-----------------+---------------------+---------------------+
1 row in set (1.012 sec)
```

### Adding Index to S3 Engine Table:

```sql
MariaDB [s3_test]> alter table percona_s3 add index idx_name (name);
Query OK, 1 row affected (8.351 sec)               
Records: 1  Duplicates: 0  Warnings: 0
```

### Modifying the Column on S3 Engine Table:

```sql
MariaDB [s3_test]> alter table percona_s3 modify column date_y timestamp DEFAULT current_timestamp();
Query OK, 1 row affected (8.888 sec)               
Records: 1  Duplicates: 0  Warnings: 0
```

### S3 Engine with DROP:

```sql
MariaDB [s3_test]> drop table percona_s3;
Query OK, 0 rows affected (2.084 sec)
```

요약하자면, S3는 read 및 구조 변경만 허용하고 데이터를 추가하거나 변경하는 것은 허용하지 않음. S3 테이블의 데이터를 수정하기 위해선 아래처럼 수행:

1. S3 테이블을 로컬 테이블로 convert (Engine = InnoDB)
2. 데이터 수정
3. 로컬 테이블을 S3 테이블로 convert (Engine = S3)

## Comparing the Query Results on Both S3 and Local

이 섹션에서는 S3 엔진 vs. 로컬에서의 query 결과를 비교함. 아래 4개 조건 하에서 실험 수행:

- `innodb_buffer_pool_dump_at_shutdown` 및 `innodb_buffer_pool_load_at_startup` 파라미터를 disable 함
- 매 쿼리를 실행하기 전/후로 MariaDB 서버를 재시작 함
- MariaDB 서버와 S3는 같은 zone에 속함
- MariaDB와 S3 간 ping time은 1.18 ms

### S3 vs. Local (Count(*))

S3: 0.16 s 소요

```sql
MariaDB [s3_test]> select count(*) from percona_perf_compare;
+----------+
| count(*) |
+----------+
| 14392799 |
+----------+
1 row in set (0.16 sec)
```

Local: 18.718 s 소요

```sql
MariaDB [s3_test]> select count(*) from percona_perf_compare;
+----------+
| count(*) |
+----------+
| 14392799 |
+----------+
1 row in set (18.718 sec)
```

S3 테이블은 read-only이고 MyISAM처럼 테이블에 row count를 가지고 있기 때문에 `Count(*)`은 S3가 더 빠름.

### S3 vs. Local (Entire Table Data)

S3: 16.10 s 소요

```sql
MariaDB [s3_test]> pager md5sum; select * from percona_perf_compare;
PAGER set to 'md5sum'
1210998fc454d36ff55957bb70c9ffaf  -
14392799 rows in set (16.10 sec)
```

Local: 11.16 s 소요

```sql
MariaDB [s3_test]> pager md5sum; select * from percona_perf_compare;
PAGER set to 'md5sum'
1210998fc454d36ff55957bb70c9ffaf  -
14392799 rows in set (11.16 sec)
```

Delay가 존재해 로컬보다 S3가 조금 더 느림

### S3 vs Local (PRIMARY KEY based lookup)

S3: 0.22 s 소요

```sql
MariaDB [s3_test]> pager md5sum; select * from percona_perf_compare where id in (7196399);
PAGER set to 'md5sum'
13b359d17336bb7dcae344d998bbcbe0  -
1 row in set (0.22 sec)
```

Local: 0.00 s 소요

```sql
MariaDB [s3_test]> pager md5sum; select * from percona_perf_compare where id in (7196399);
PAGER set to 'md5sum'
13b359d17336bb7dcae344d998bbcbe0  -
1 row in set (0.00 sec)
```

Delay가 존재해 로컬보다 S3가 조금 더 느림

위 실험들은 default S3 셋팅으로 진행되었지만, S3의 성능을 높이기 위해 아래와 같은 방법을 사용할 수도 있음:

- `s3_block_size` 감소 (Default = 4M)
- 테이블 생성 시 `COMPRESSION_ALGORITHM=zlib` 사용. 이를 통해 S3에서 로컬 캐시로 가는 데이터 양을 줄일 수 있음 (Default = none)
- S3 페이지 캐시 사이즈인 `s3_pagecache_buffer_size` 증가

또한 위 성능은 disk access speed와 서버와 S3 간 network health에 따라 달라질 수 있음:

- Low performance disk + Good network ⇒ favor S3
- Good performance disk + Poor network ⇒ favor Local

## Conclusion

- 복원할 필요 없이 historical data에 쿼리를 수행할 수 있으므로, data archival 시 good solution
- S3 테이블은 completely read-only
- MyISAM처럼 `COUNT(*)` 쿼리는 매우 빠름
- S3도 확인해야하기 때문에 `CREATE TABLE, DROP TABLE, INFORMATION_SCHEMA tables` 쿼리는 느림