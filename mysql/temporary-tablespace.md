# Temporary Tablespaces

InnoDB에는 session temporary tablespaces 및 global temporary tablespace의 크게 2가지 temporary tablespace가 존재함.

## Session Temporary Tablespaces

### 특징

User 및 optimizer가 생성한 temporary tables 저장

### 동작 방식

- *서버 시작 시*, 10개의 temporary tablespace로 구성된 *Pool* 생성
    - `.ibt` 파일 **10개**로 관리
    - 각 파일은 생성 시 **페이지 5개 크기**로 생성

```bash
shell> cd BASEDIR/data/#innodb_temp
shell> ls
temp_10.ibt  temp_2.ibt  temp_4.ibt  temp_6.ibt  temp_8.ibt
temp_1.ibt   temp_3.ibt  temp_5.ibt  temp_7.ibt  temp_9.ibt
```

- Pool의 크기는 절대 줄어들지 않으며, 필요한 만큼 tablespace 크기가 추가되는 방식
- 디스크에 temporary table을 생성하라는 첫 요청을 받으면, temporary tablespace pool에서 session temporary tablespace가 세션에 할당
- *세션 시작 시*, 최대 두 개의 temporary tablespace가 할당
    1. User가 생성하는 temporary table 용
    2. Optimizer가 생성하는 internal temporary table 용
- *세션 종료 시*, 해당 temporary tablespace은 truncate 되며 pool로 다시 release
- *서버 종료 시*, temporary tablespace pool 삭제

### 관련 서버 파라미터

- [`internal_tmp_disk_storage_engine`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_internal_tmp_disk_storage_engine): Internal temporary table에 사용되는 storage engine (default: `INNODB`)
- [`innodb_temp_tablespaces_dir`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_temp_tablespaces_dir): Session temporary tablespace pool을 생성할 위치

## Global temporary tablespaces

### 특징

User가 생성한 temporary table에 대한 rollback segment 저장

### 동작 방식

- *서버 시작 시*, global temporary tablespace를 위한 파일 생성
    - 약 **12MB** 크기의 `ibtmp1` 파일 **1개** 생성
- 필요한 크기만큼 자동으로 증가
- 아래 쿼리를 사용해 현재 크기 (`TotalSizeBytes`) 확인 가능

```sql
mysql> SELECT FILE_NAME, TABLESPACE_NAME, ENGINE, INITIAL_SIZE, TOTAL_EXTENTS*EXTENT_SIZE
       AS TotalSizeBytes, DATA_FREE, MAXIMUM_SIZE FROM INFORMATION_SCHEMA.FILES
       WHERE TABLESPACE_NAME = 'innodb_temporary'\G
*************************** 1. row ***************************
      FILE_NAME: ./ibtmp1
TABLESPACE_NAME: innodb_temporary
         ENGINE: InnoDB
   INITIAL_SIZE: 12582912
 TotalSizeBytes: 12582912
      DATA_FREE: 6291456
   MAXIMUM_SIZE: NULL
```
- *서버 종료 시*, 삭제

### 관련 서버 파라미터

- [innodb_temp_data_file_path](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_temp_data_file_path): Global temporary tablespace 데이터 파일의 상대 경로, 이름, 크기 및 속성을 정의 (default: `ibtmp1:12M:autoextend`)

## Hash Join

### 동작 방식

기존 Hash join과 크게 다르지 않음 (정리 예정)

### 주의 사항

- 8.0.18 버전까지는 inner join에서만 hash join 사용 가능
- 8.0.20 버전부터 inner/outer join에서 hash join 사용 가능
- [`join_buffer_size`](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_join_buffer_size) 크기 변경 (default: 256KB); Maximum = 4GB - 1

## Reference

- [15.6.3.5 Temporary Tablespaces, MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/innodb-temporary-tablespace.html)
- [Hash join in MySQL 8, MySQL Server Blog](https://mysqlserverteam.com/hash-join-in-mysql-8/)