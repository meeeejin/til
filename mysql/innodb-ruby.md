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

Tablespace의 모든 페이지를 순회하고, 같은 유형의 페이지를 `영역(regions)`으로 묶어, 전체 페이지의 유형을 요약해 출력하는 모드다. 즉, tablespace의 시작부터 끝까지 순회하며, 주어진 페이지 유형의 연속된(contiguous) 블록을 한 줄씩 출력하는 모드다.
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

### space-index-pages-summary

모든 index 페이지에 대해 space-consumption 관련 데이터를 보여주는 모드다. 이 모드를 사용해 데이터와 여유(free) 공간의 크기 및 테이블에 대한 레코드 개수를 쉽게 확인할 수 있다.
file-per-table tablespace에 대해 `space-index-pages-summary` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-index-pages-summary
page        index   level   data    free    records
3           78      1       8260    7700    590     
4           78      0       7517    8693    85      
5           78      0       15127   1043    171     
6           78      0       15099   1071    170     
...
```

### space-index-pages-free-plot

`gnuplot`과 `Ruby gnuplot gem`이 설치되어 있으면, index의 space-consumption 정보를 scatter plot으로 그릴 수 있는 모드다.
2개의 file-per-table tablespace에 대해 각각 `space-index-pages-free-plot` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-index-pages-free-plot
Wrote tpcc57_100_item_free.png
```
![tpcc57_100_item_free](https://i.imgur.com/o3dFqrr.png)

```bash
$ innodb_space -f tpcc57_100/orders.ibd space-index-pages-free-plot
Wrote tpcc57_100_orders_free.png
```
![tpcc57_100_orders_free](https://i.imgur.com/HKj4YPw.png)

y축은 각 페이지의 여유 공간을 나타내며, x축은 페이지 번호이자 파일 오프셋이다.

### space-extents-illustrate

Tablespace에 속한 모든 extent의 페이지들을 보여주는 모드다. 블록은 페이지의 데이터 양에 따라 크기가 지정되고, index/목적 별로 색깔이 지정된다.
file-per-table tablespace에 대해 `space-extents-illustrate` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-extents-illustrate
```
![space-extents-illustrate-result](https://i.imgur.com/R0L4P09.png)

### space-lsn-age-illustrate

Tablespace에 속한 모든 extent의 페이지를 색칠된 블록으로 보여주는 모드다. 블록은 페이지의 수정 LSN의 age에 따라 색깔이 지정된다.
file-per-table tablespace에 대해 `space-lsn-age-illustrate` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd space-lsn-age-illustrate
```
![space-lsn-age-illustrate-result](https://i.imgur.com/g95WCPw.png)

256 color를 사용해(`tmux -2`) 같은 tablespace 다시 그린 결과는 다음과 같다:
![space-lsn-age-illustrate-result-256](https://i.imgur.com/iq1NxBW.png)

## Page Structure

### page-account

페이지 번호가 주어졌을 때, 해당 페이지에 대해 설명하는 모드다.
System tablespace의 3번 페이지에 대해 `page-account` 모드를 사용한 결과는 다음과 같다:

```
$ innodb_space -s ibdata1 -p 3 page-account
Accounting for page 3:
  Page type is SYS (System internal, used for various purposes in the system tablespace).
  Extent descriptor for pages 0-63 is at page 0, offset 158.
  Extent is not fully allocated to an fseg; may be a fragment extent.
  Page is marked as used in extent descriptor.
  Extent is in full_frag list of space.
  Page is in fragment array of fseg 1.
```

file-per-table tablespace의 100번 페이지에 대해 `page-account` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd -p 100 page-account
Accounting for page 100:
  Page type is INDEX (B+Tree index, table and index data stored in B+Tree structure).
  Extent descriptor for pages 64-127 is at page 0, offset 198.
  Extent is fully allocated to fseg 2.
  Page is marked as used in extent descriptor.
  Extent is in full list of fseg 2.
  Fseg is in leaf fseg of index 78.
  Index root is page 3.
```

### page-dump

페이지 번호가 주어졌을 때, 해당 페이지에 대해 `innodb_ruby`가 이해하는 모든 정보를 intelligently dump하는 모드다.
file-per-table tablespace의 100번 페이지에 대해 `page-dump` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd -p 100 page-dump
#<Innodb::Page::Index:0x000000024a0fb8>:

fil header:
{:checksum=>451784286,
 :offset=>100,
 :prev=>99,
 :next=>101,
 :lsn=>247858850703,
 :type=>:INDEX,
 :flush_lsn=>0,
 :space_id=>50}

fil trailer:
{:checksum=>451784286, :lsn_low32=>3045714831}

page header:
{:n_dir_slots=>43,
 :heap_top=>15187,
 :garbage_offset=>0,
 :garbage_size=>0,
...

fseg header:
{:leaf=>nil, :internal=>nil}

sizes:
  header           120
  trailer            8
  directory         86
...
```

첫 번째 행은 해당 페이지를 핸들링하는 클래스를 나타낸다. 그리고 `fil header`와 `fil trailer`는 모든 페이지가 공통적으로 갖는 필드이며, 페이지 자체에 대한 정보를 주로 포함한다. FIL header 다음에 오는 정보들은 페이지 유형에 따라 결정된다. 위와 같은 index 페이지를 dump 했을 때는 아래와 같은 정보가 출력된다.

- page header: index 페이지에 대한 정보
- fseg header: index에서 사용되는 파일 세그먼트(extent의 그룹)의 space management와 관련된 정보
- size: 페이지의 다양한 구성 요소들(free space, data space, record size 등)의 크기를 byte 단위로 요약
- system records, infimum and supremum
- record 검색을 보다 효율적으로 수행하는 데 사용되는 page directory의 내용
- 사용자가 저장한 실제 데이터인 user record (record describer가 로드되지 않은 경우, 해당 필드는 파싱되지 않음)

### page-records

페이지 번호가 주어졌을 때, 해당 페이지 내의 모든 레코드를 요약하는 모드다.
file-per-table tablespace의 100번 페이지에 대해 `page-records` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd -p 100 page-records
Record 128: () → ()

Record 213: () → ()

Record 303: () → ()

Record 382: () → ()

Record 468: () → ()
...
```

### page-directory-summary

페이지 번호가 주어졌을 때, 해당 페이지의 page directory 내용을 dump하는 모드다.
Record describer를 사용 가능한 경우, 각 레코드의 key가 출력된다.
file-per-table tablespace의 100번 페이지에 대해 `page-directory-summary` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -f tpcc57_100/item.ibd -p 100 page-directory-summary
slot    offset  type          owned   key
0       99      infimum       1       
1       382     conventional  4       ()
2       737     conventional  4       ()
3       1098    conventional  4       ()
4       1459    conventional  4       ()
5       1811    conventional  4       ()
6       2149    conventional  4       ()
7       2476    conventional  4       ()
8       2807    conventional  4       ()
...
```

### page-illustrate

페이지 번호가 주어졌을 때, 해당 페이지의 내용을 `region type`을 기준으로 그리는 모드다.

> record data가 free로 나타남. 코드 확인 필요

```bash
$ innodb_space -f tpcc57_100/item.ibd -p 100 page-illustrate
```

## Record Structure

- `-R` 또는 `--record` 이용
- Record describer가 있어야 레코드의 내용을 파싱할 수 있음. Ruby class 예시:
```bash
class SimpleTBTreeDescriber < Innodb::RecordDescriber
  type :clustered
  key "i", :INT, :NOT_NULL
  row "s", "CHAR(10)", :NOT_NULL
end
```
- 레코드 내용에 맞게 Ruby class를 작성한 후, `innodb_space`에 argument로 넘겨 주어야 로드 가능
```bash
innodb_space -r ./simple_t_btree_describer.rb -d SimpleTBTreeDescriber ...
```

### record-dump

레코드 offset이 주어졌을 때, 해당 레코드 및 레코드의 데이터에 대한 내용을 dump하는 모드다.
file-per-table tablespace의 100번 페이지의 100번 레코드에 대해 `record-dump` 모드를 사용한 결과는 다음과 같다:

```bash
$ innodb_space -r ./simple_t_btree_describer.rb -d SimpleTBTreeDescriber -f tpcc57_100/item.ibd -p 100 -R 100 record-dump
Record at offset 100

Header:
  Next record offset  : 7629
  Heap number         : 64
  Type                : conventional
  Deleted             : false
  Length              : 5

System fields:
  Transaction ID: 129111012278283
  Roll Pointer:
    Undo Log: page 7566704, offset 29285
    Rollback Segment ID: 0
    Insert: false

Key fields:
  i: 123456789

Non-key fields:
  s: "mum%\x10\x00\x00\x00\x10\x00"
```

Record describer의 형식에 따라 파싱 결과가 달라진다.

### record-history

레코드의 history를 요약하는 모드다.

```bash
$ innodb_space -f tpcc57_100/item.ibd -p 100 -R 100 record-history
```

## Index Structure

### index-recurse

### index-record-offsets

### index-level-summary
