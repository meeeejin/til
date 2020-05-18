# A comparison of buffer management algorithms between three DBMSs

이 문서에서는 MySQL, Oracle 그리고 PostgreSQL의 버퍼 관리 방식 (페이지를 읽어오는 과정에 초점을 맞춤)에 대해 간단히 정리하고, 세 DBMS의 버퍼 관리 알고리즘을 비교합니다.

## MySQL

## Oracle

- [참고 자료](https://docs.oracle.com/cd/E25178_01/server.1111/e25789/memory.htm)
- [참고 포스트](https://otsteam.tistory.com/164)

### Page read

클라이언트 프로세스가 버퍼를 요청하면, 서버 프로세스는 버퍼 캐시에서 해당 버퍼를 검색합니다:

1. 서버 프로세스가 버퍼 캐시의 전체 버퍼를 탐색합니다. 원하는 버퍼를 찾으면, 이 버퍼에 대해 **logical read**를 수행합니다 (**cache hit**).

2. 원하는 버퍼를 찾지 못하면 (**cache miss**), 서버 프로세스는 다음 단계를 수행합니다:
    1. 디스크의 데이터 파일에서 메모리로 해당 블록을 읽어옵니다 (**physical read**).
    2. 메모리 (버퍼 캐시)로 읽어들인 버퍼에 대해 **logical read**를 수행합니다.

### 버퍼 매니저 동작 방식

#### Free 버퍼를 찾는 과정

1. Oracle 프로세스는 LRU 리스트에 lock을 걸고, LRU의 tail부터 free 버퍼를 찾습니다. 이를 찾는 중에 dirty 버퍼를 만나면, 이 dirty 버퍼를 LRUW 리스트로 옮깁니다.

2. Scan depth만큼 LRU 리스트를 탐색했는데도 free 버퍼를 찾지 못하면, 서버 프로세스는 스캔을 중단하고 DBWR에게 dirty 버퍼를 모으라는 신호를 보내고 lock을 해제합니다 ([free buffer wait](https://github.com/meeeejin/til/blob/master/oracle/understanding-oracle-wait-events.md#free-buffer-waits)).

3. 신호를 받은 DBWR는 LRU 리스트에 lock을 걸고, LRU의 tail부터 `DBWR lru scans` (default: 8) 만큼의 디스크에 쓰여야 할 dirty 버퍼를 모읍니다.

4. LRUW에 모인 dirty 버퍼는 DBWR에 의해 디스크로 쓰여지고, 이렇게 쓰여진 버퍼는 free 버퍼가 되어 다시 사용될 수 있도록 LRU의 tail에 위치하게 됩니다.

### 페이지 교체 알고리즘: LRU 기반의 블록 레벨 교체 알고리즘

Oracle은 dirty 버퍼와 non-dirty 버퍼에 대한 포인터를 포함하는 LRU  (Least Recently Used) 리스트를 사용합니다. LRU 리스트에는 hot end와 cold end가 있습니다. Cold 버퍼는 최근에 사용되지 않은 버퍼이며, hot 버퍼는 자주 접근되고 최근에 사용된 버퍼입니다. LRU는 크게 두 개의 리스트로 구성되며, 버퍼 캐시 상의 버퍼 블록은 반드시 이 둘 중 하나의 리스트에만 속합니다:

![lru](https://www.relationaldbdesign.com/database-creation-architecture/module5/images/buffercache_mo.gif)

#### LRU list

- Free buffer: 사용되지 않은 버퍼
- Pinned buffer: 현재 다른 유저에 의해 사용 중이어서 재사용될 수 없는 상태의 버퍼
- Dirty buffer: 유저가 사용하여 내용이 변경된 버퍼로 LRUW 리스트로 옮겨질 수 있고, 결국엔 디스크로 flush 될 버퍼

#### LRUW (LRU Write) list

Dirty list, dirty queue라고도 불리며, DBWR는 이 리스트의 버퍼들을 디스크에 flush 하여 free 버퍼로 만듭니다.

#### Touch count

![touch-count-based](http://wiki.gurubee.net/download/attachments/6259339/Cache_Buffer-009.png)

오라클은 개별 버퍼마다 touch count를 관리하며, 프로세스에 의해서 스캔이 이루어질 때마다 touch count를 1씩 증가시킵니다.

- Cold end의 tail에 있으면서 touch count가 1 이하인 버퍼가 free 버퍼로 사용됩니다.

- Cold end의 tail에 있으면서 touch count가 2 이상인 버퍼를 만나면, hot end의 head 부분으로 옮기고 touch count를 0으로 초기화합니다.

- Hot end로 옮기는 기준은 `_DB_AGING_HOT_CRITERIA` 파라미터에 의해 결정되며, default로 2입니다.

- Single block I/O를 통해 읽어온 블록은 mid-point에 삽입되며, 이 블록의 touch count는 1입니다.

- Full table scan이나 index full scan으로 읽혀진 데이터들은 mid-point에 삽입되었다가 바로 cold end의 tail로 옮겨져서 버퍼 캐시에 머무를 확률이 낮아집니다.

## PostgreSQL

- [참고 자료](http://www.interdb.jp/pg/pgsql08.html)

### Page read

![page-read](http://www.interdb.jp/pg/img/fig-8-02.png)

1. 테이블 또는 index 페이지를 읽을 때, 백엔드 프로세스는 페이지의 `buffer_tag`를 포함하는 읽기 요청을 버퍼 관리자에게 보냅니다.

2. 버퍼 관리자는 요청된 페이지를 저장하는 슬롯의 `buffer_ID`를 리턴합니다. 요청된 페이지가 버퍼 풀에 없는 경우, 버퍼 관리자는 페이지를 디스크에서 버퍼 풀 슬롯 중 하나로 로드한 다음 해당 `buffer_ID`를 리턴합니다.

3. 백엔드 프로세스는 `buffer_ID` 슬롯에 접근하여 원하는 페이지를 읽습니다.

### 버퍼 매니저 동작 방식

백엔드 프로세스가 원하는 페이지에 접근하려고 할 때, `ReadBufferExtended()`를 호출합니다. 이 때, `ReadBufferExtended()`의 동작은 크게 3가지 경우로 나뉩니다.

#### 1. 요청된 페이지가 버퍼 풀에 있는 경우

![first-case](http://www.interdb.jp/pg/img/fig-8-08.png)

가장 간단한 경우입니다. 말 그대로, 원하는 페이지가 이미 버퍼 풀에 저장되어 있는 경우입니다:

1. 원하는 페이지의 `buffer_tag`를 생성하고 (이 예제에서 `buffer_tag`는 `Tag_C`), hash 함수를 사용하여 생성한 `buffer_tag`와 연관된 항목들을 포함하는 *hash bucket 슬롯*을 계산합니다.

2. 해당 hash bucket 슬롯을 커버하는 `BufMappingLock` 파티션을 shared 모드로 획득합니다 (이 lock은 5번째 단계에서 해제됨).

3. 태그가 `Tag_C`인 항목을 찾고, 해당 항목에서 `buffer_id`를 얻습니다. 이 예제에서 `buffer_id`는 2입니다.

4. `buffer_id` 2에 대한 buffer descriptor를 pin 합니다. 즉, descriptor의 `refcount` 및 `usage_count`가 1씩 증가합니다.

5. `BufMappingLock`을 해제합니다.

6. `buffer_id` 2를 사용하여 버퍼 풀 슬롯에 접근합니다.

#### 2. 요청된 페이지를 디스크에서 읽어서 빈 슬롯에 저장하는 경우

![second-case](http://www.interdb.jp/pg/img/fig-8-09.png)

두 번째는 원하는 페이지가 버퍼 풀에 없고, `freelist`가 비어 있지 않은 경우입니다.

1. 버퍼 테이블을 검색합니다:
    1. 원하는 페이지의 `buffer_tag`를 생성하고 (이 예제에서 `buffer_tag`는 `Tag_E`), hash bucket 슬롯을 계산합니다.
    2. `BufMappingLock` 파티션을 shared 모드로 획득합니다.
    3. 버퍼 테이블을 검색합니다. 하지만 원하는 페이지를 찾지 못했습니다.
    4. `BufMappingLock`을 해제합니다.

2. `freelist`에서 *비어 있는 buffer descriptor*를 확보하여 pin 합니다. 이 예제에서 획득한 descriptor의 `buffer_id`는 4입니다.

3. *Exclusive* 모드에서 `BufMappingLock` 파티션을 획득합니다 (이 lock은 6번째 단계에서 해제됨). 

4. `buffer_tag`가 `Tag_E`이고 `buffer_id`가 4인 새 데이터 항목을 생성합니다. 생성한 항목을 버퍼 테이블에 삽입합니다.

5. 다음과 같이 `buffer_id` 4를 사용하여 스토리지에서 버퍼 풀 슬롯으로 원하는 페이지 데이터를 읽어옵니다:
    1. `io_in_progress_lock`을 exclusive 모드로 획득합니다.
    2. 다른 프로세스의 접근을 방지하기 위해 해당 descriptor의 `io_in_progress` 비트를 1로 설정합니다.
    3. 스토리지에서 버퍼 풀 슬롯으로 원하는 페이지 데이터를 읽어옵니다.
    4. 해당 descriptor의 상태를 변경합니다. `io_in_progress` 비트는 **0**으로 설정되고, 유효한 비트는 **1**로 설정됩니다.
    5. `io_in_progress_lock`을 해제합니다.

6. `BufMappingLock`을 해제합니다.

7. `buffer_id` 4를 사용하여 버퍼 풀 슬롯에 접근합니다.

#### 3. 요청된 페이지를 디스크에서 읽어서 victim 슬롯에 저장하는 경우

![third-case](http://www.interdb.jp/pg/img/fig-8-10.png)

세 번째는 모든 버퍼 풀 슬롯이 사용 중이지만 원하는 페이지는 버퍼 풀에 없는 경우입니다. 

1. 원하는 페이지의 `buffer_tag`를 생성하고 (이 예제에서 `buffer_tag`는 `Tag_M`), 버퍼 테이블을 검색합니다. 하지만 원하는 페이지를 찾지 못했습니다.

2. **Clock-sweep** 알고리즘을 사용하여 victim 버퍼 풀 슬롯을 선택하고, 버퍼 테이블에서 victim 슬롯의 `buffer_id`를 포함하는 이전 항목을 가져 와서 buffer descriptor 레이어에 victim 슬롯을 pin 합니다. 이 예제에서 victim 슬롯의 `buffer_id`는 5이고, 이전 항목은 `Tag_F, id=5` 입니다. Clock-sweep은 다음 섹션에서 설명합니다.
```bash
StrategyGetBuffer() // victim 버퍼 선택
    LWLockAcquire(BufFreelistLock, LW_EXCLUSIVE);
    if (bgwriterLatch)
        LWLockRelease(BufFreelistLock);
        bgwriterLatch를 기다리고 있는 프로세스를 깨움
        LWLockAcquire(BufFreelistLock, LW_EXCLUSIVE);
    freelist에서 buffer 가져옴
        있으면, return buf
    없으면, for문 돌면서 Clock-sweep 알고리즘 수행
        unpin && usable count == 0 → usable buffer
        usable buffer가 있으면, return buf
PinBuffer_Locked() // victim 버퍼 pinning
LWLockRelease(BufFreelistLock);
```

3. Victim 페이지 데이터가 dirty면, flush (write and fsync) 합니다. 그렇지 않으면 4단계로 넘어갑니다.
    1. `buffer_id` 5를 사용하여, descriptor의 shared `content_lock` 및 exclusive `io_in_progress` lock을 획득합니다.
    2. 해당 descriptor의 상태를 변경합니다. `io_in_progress` 비트는 **1**로 설정하고, `just_dirtied` 비트는 **0**으로 설정합니다.
    3. 상황에 따라, WAL 버퍼의 WAL 데이터를 현재 WAL 세그먼트 파일에 기록하기 위해 `XLogFlush()` 함수가 호출됩니다.
    4. Victim 페이지 데이터를 스토리지로 flush 합니다.
    5. 해당 descriptor의 상태를 변경합니다. `io_in_progress` 비트는 **0**으로 설정되고, 유효한 비트는 **1**로 설정됩니다.
    6. `io_in_progress` 및 `content_lock` lock을 해제합니다.

4. 이전 항목을 포함하는 슬롯을 커버하는 old `BufMappingLock` 파티션을 exclusive 모드로 획득합니다.

5. New `BufMappingLock` 파티션을 획득하고, 새 항목을 버퍼 테이블에 삽입합니다.
    1. 새로운 `buffer_tag`인 `Tag_M`과 victim의 `buffer_id`로 구성된 새 항목을 만듭니다.
    2. 새 항목을 포함하는 슬롯을 커버하는 new `BufMappingLock` 파티션을 exclusive 모드로 획득합니다.
    3. 새 항목을 버퍼 테이블에 삽입합니다.

![third-case-2](http://www.interdb.jp/pg/img/fig-8-11.png)

6. 버퍼 테이블에서 이전 항목을 삭제하고, old `BufMappingLock` 파티션을 해제합니다.

7. 스토리지에서 victim 버퍼 슬롯으로 원하는 페이지 데이터를 읽어옵니다. 그리고 나서, `buffer_id` 5를 갖는 descriptor의 플래그를 업데이트합니다. Dirty 비트는 **0**으로 설정하고, 다른 비트들을 초기화합니다.

8. New `BufMappingLock` 파티션을 해제합니다.

9. `buffer_id` 5를 사용하여 버퍼 풀 슬롯에 접근합니다.

### 페이지 교체 알고리즘: Clock Sweep

이 알고리즘은 오버헤드가 적은 NFU (Not Frequently Used)를 변형한 알고리즘입니다. 즉, 덜 자주 사용하는 페이지를 효율적으로 선택합니다.

![clock-sweep](http://www.interdb.jp/pg/img/fig-8-12.png)

위 그림에서 buffer descriptor는 blue (pin) 또는 cyan (unpin) 색 상자로 표시되며, 상자의 숫자는 각 descriptor의 `usage_count`를 나타냅니다.

1. `nextVictimBuffer`는 첫 번째 descriptor (`buffer_id` 1)를 가리킵니다. 하지만, 이 descriptor는 pin 되어 있으므로 건너 뜁니다.

2. `nextVictimBuffer`는 두 번째 descriptor (`buffer_id` 2)를 가리킵니다. 이 descriptor는 pin 되어 있진 않지만, `usage_count`는 2입니다. 따라서 `usage_count`를 1만큼 감소시키고, `nextVictimBuffer`는 세 번째 victim으로 넘어갑니다.

3. `nextVictimBuffer`는 세 번째 descriptor (`buffer_id` 3)를 가리킵니다. 이 descriptor는 unpin 상태이며 `usage_count`는 0입니다. 따라서 이 descriptor가 이번 라운드의 victim이 됩니다.

`nextVictimBuffer`가 unpin 상태의 descriptor를 sweep 할 때마다 `usage_count`가 1씩 감소합니다. 따라서 unpin 상태의 descriptor가 버퍼 풀에 존재하면 이 알고리즘은 `nextVictimBuffer`을 회전시키면서 `usage_count`가 0인 victim을 항상 찾을 수 있습니다.

