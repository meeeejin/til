# innodb_ruby

Jeremy Cole의 [블로그 포스트](https://blog.jcole.us/2013/01/03/a-quick-introduction-to-innodb-ruby/) 및 [innodb_ruby wiki 페이지](https://github.com/jeremycole/innodb_ruby/wiki)를 참고해 [innodb_ruby](https://github.com/jeremycole/innodb_ruby) 사용법을 정리한 문서이다.

## Install

RubyGems를 사용해 root 권한으로 innodb_ruby 설치한다.

```bash
$ sudo gem install innodb_ruby
```

`innodb_space` 명령으로 실행 가능하며, `innodb_space -f <file> [-p <page>] [-l <level>] <mode> [<mode>, ...]` 형식으로 사용 가능하다.

```bash
$ innodb_space ...
```

## Basic Tutorial

`innodb_space`는 아래의 두 가지 방식으로 사용 가능하다.

1. 하나의 tablespace 파일(ibdata 또는 .ibd)에 대해:

| Option   |  Parameters | Description |
|:----------:|-------------|-------------|
| -f | \<filename\> | Load the tablespace file (system or table)  |

2. file-per-table tablespace 파일을 자동으로 로드하는 system tablespace에 대해:

| Option   |  Parameters | Description |
|:----------:|-------------|-------------|
| -s | \<filename> | Load the system tablespace file (e.g. ibdata1) |
| -T | \<table name> | Use the given table name. |
| -I | \<index name> | Use the given index name |

## Space File Structure

### system-spaces

기본적인 통계 정보(page 및 index 개수)를 포함하여, 시스템에서 사용 가능한 모든 tablespace 목록을 나열하는 모드다.
System tablespace에 대해 `system-spaces` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -s ibdata1 system-spaces
name                            pages       indexes     
(system)                        65536       7           
mysql/engine_cost               6           1           
mysql/gtid_executed             6           1           
mysql/help_category             7           2           
mysql/help_keyword              15          2           
mysql/help_relation             9           1           
mysql/help_topic                576         2           
...
```

### space-page-type-regions

Tablespace의 모든 페이지를 순회하고, 같은 유형의 페이지를 `영역(regions)`으로 묶어, 전체 페이지의 유형을 요약해 출력하는 모드다.
file-per-table tablespace에 대해 `space-page-type-regions` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-page-type-regions
start       end         count       type                
0           0           1           FSP_HDR             
1           1           1           IBUF_BITMAP         
2           2           1           INODE               
3           35          33          INDEX               
36          63          28          FREE (ALLOCATED)    
64          621         558         INDEX               
622         639         18          FREE (ALLOCATED)
```

### space-page-type-summary
Tablespace의 모든 페이지를 순회하고, 유형별 총 페이지 개수를 요약해 출력하는 모드다.
file-per-table tablespace에 대해 `space-page-type-summary` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-page-type-summary
type                count       percent     description         
INDEX               591         54.32       B+Tree index        
ALLOCATED           494         45.40       Freshly allocated   
FSP_HDR             1           0.09        File space header   
IBUF_BITMAP         1           0.09        Insert buffer bitmap
INODE               1           0.09        File segment inode
```

### space-indexes

System space 또는 file-per-table space에서 사용할 수 있는 모든 index를 나열하는 모드다.
file-per-table tablespace에 대해 `space-indexes` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-indexes
id          name                            root        fseg        used        allocated   fill_factor
78                                          3           internal    1           1           100.00%     
78                                          3           leaf        590         608         97.04%
```

모든 index는 non-leaf 페이지에 사용되는 `internal` 파일 세그먼트(file segment)와 leaf 페이지에 사용되는 `leaf` 파일 세그멘트를 갖는다. 페이지는 하나의 파일 세그먼트에 할당될 수 있지만, 현재 미사용 중일 수 있다(**type FREE (ALLOCATED)**). 따라서 `fill_factor`는 `사용 중인 페이지 / 할당된 전체 페이지`의 비율을 나타낸다(index 페이지가 *얼마나 꽉 찼는지* 와는 관련이 없으며, 이는 다른 문제).

### space-extents-illustrate

Tablespace에 속한 모든 extent의 페이지들을 보여주는 모드다. 블록은 페이지의 데이터 양에 따라 크기가 지정되고, index/목적 별로 색깔이 지정된다.
file-per-table tablespace에 대해 `space-extents-illustrate` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-extents-illustrate
![space-extents-illustrate-result](https://i.imgur.com/R0L4P09.png)
```

### space-lsn-age-illustrate

Tablespace에 속한 모든 extent의 페이지를 색칠된 블록으로 보여주는 모드다. 블록은 페이지의 수정 LSN의 age에 따라 색깔이 지정된다.
file-per-table tablespace에 대해 `space-lsn-age-illustrate` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-lsn-age-illustrate
![space-lsn-age-illustrate-result](https://i.imgur.com/g95WCPw.png)
```

## Page Structure

작성 중..
