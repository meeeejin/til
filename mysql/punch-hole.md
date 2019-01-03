# Punch hole

[포스트](https://mysqlserverteam.com/innodb-transparent-page-compression/)
[메뉴얼](https://dev.mysql.com/doc/refman/5.7/en/innodb-page-compression.html)

## Transparent Page Compression

InnoDB는 [file-per-table](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_file_per_table) 테이블스페이스에 존재하는 테이블에 대해 페이지 레벨 압축을 지원한다. 이는 *Transparent Page Compression* 라 불린다. 페이지 압축은 `CREATE TABLE` 또는 `ALTER TABLE` 문에 `COMPRESSION` attribute를 명시하여 사용할 수 있으며, *Zlib*과 *LZ4* 압축 알고리즘을 지원한다. 예를 들어, 아래와 같이 사용할 수 있다:

```bash
CREATE TABLE t1 (c1 INT) COMPRESSION="zlib";
```

한편, `ALTER TABLE ... COMPRESSION`은 테이블스페이스의 압축 attribute만 업데이트한다(즉, metadata만 변경). 새 압축 알고리즘을 설정한 후에 발생하는 쓰기는 새로운 설정을 사용하지만, 기존 압축 페이지에 새 압축 알고리즘을 적용하려면 `OPTIMIZE TABLE`을 사용하여 테이블을 재빌드해야 한다:

```bash
ALTER TABLE t1 COMPRESSION="zlib";
OPTIMIZE TABLE t1;
```

High level에서 봤을 때, transparent page compression은 간단한 페이지 변환이다.

```
Write : Page -> Transform -> Write transformed page to disk -> Punch hole
Read  : Page from disk -> Transform -> Original Page
```

MySQL 5.7에는 여러 개의 페이지 플러싱 전용 thread가 존재한다. 이는 디스크에 페이지를 쓰기 전에 전용 background thread로 "변환"을 오프로드하여 디스크에 기록한 후, 압축과 "hole punching"을 병렬 처리하는 데 적합하다.

그리고 transparent page compression 기능을 사용하려면 운영 체제와 파일 시스템이 [sparse files](https://en.wikipedia.org/wiki/Sparse_file)과 hole punching을 지원해야 한다.

> Sparse file은 0이 아닌 영역에만 물리적인 디스크 공간을 할당하는 파일 유형으로써, 파일 자체가 부분적으로 비어 있는 경우 파일 시스템 공간을 보다 효율적으로 사용할 수 있음

<p align="center">
<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/9/9f/Sparse_file_%28en%29.svg/1024px-Sparse_file_%28en%29.svg.png" alt="sparse file" width="400" />
</p>

## Linux 상에서의 Hole Punch 크기

Linux 시스템에서 파일 시스템의 블록 크기는 hole punching에 사용되는 단위 크기다. 따라서, 페이지 압축은 `InnoDB 페이지 크기 - 파일 시스템 블록 크기`보다 작거나 같은 크기로 압축될 수 있는 경우에만 작동한다. 예를 들어, [innodb_page_size](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_page_size)가 16KB고 파일 시스템 블록 크기가 4KB인 경우, hole punching을 가능하게 하려면 페이지 데이터를 12KB 이하로 압축해야 한다.

## Monitoring

Hole punching에서 `ls -l`로 보여지는 파일 크기는 블록 디바이스의 실제 할당 크기가 아닌 논리적 파일 크기를 표시한다. 이는 sparse file의 일반적인 이슈며, 논리적 크기와 실제 할당 크기는 INNODB_SYS_TABLESPACES information schema 테이블에 쿼리를 수행하여 얻을 수 있다.

다음과 같은 추가적인 열이 information schema 뷰에 추가되었다: `FS_BLOCK_SIZE`, `FILE_SIZE`, `ALLOCATED_SIZE` 및 `COMPRESSION`

- `FS_BLOCK_SIZE`: 파일 시스템 블록 크기
- `FILE_SIZE`: 파일의 논리적 크기이며, ls -l로 볼 수 있음
- `ALLOCATED_SIZE`: 파일 시스템의 블록 디바이스에 실제로 할당된 크기
- `COMPRESSION`: 현재 압축 알고리즘 설정(있는 경우)

Note: 앞서 언급했듯이, `COMPRESSION` 값은 현재 테이블스페이스의 설정이며 현재 테이블스페이스에 있는 모든 페이지가 해당 형식을 갖는 것을 보장하지는 않는다.

다음은 간단한 예시다:

```bash
mysql> select * from information_schema.INNODB_SYS_TABLESPACES WHERE name like 'linkdb%';
+-------+------------------------+------+-------------+----------------------+-----------+---------------+------------+---------------+-------------+----------------+-------------+
| SPACE | NAME                   | FLAG | FILE_FORMAT | ROW_FORMAT           | PAGE_SIZE | ZIP_PAGE_SIZE | SPACE_TYPE | FS_BLOCK_SIZE | FILE_SIZE   | ALLOCATED_SIZE | COMPRESSION |
+-------+------------------------+------+-------------+----------------------+-----------+---------------+------------+---------------+-------------+----------------+-------------+
|    23 | linkdb/linktable#P#p0  |    0 | Antelope    | Compact or Redundant |     16384 |             0 | Single     |           512 |  4861198336 |     2376154112 | LZ4         |

...
```
