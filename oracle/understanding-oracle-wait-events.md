# Understanding Oracle wait events

- [Reference](https://docs.oracle.com/database/121/REFRN/GUID-2FDDFAA4-24D0-4B80-A157-A907AF5C68E2.htm#REFRN-GUID-2FDDFAA4-24D0-4B80-A157-A907AF5C68E2)

## db file sequential read

- 한 블록씩 읽어들일 때 발생 (single block I/O)
- 여러 경우에 발생:
  - Index scan
  - Rollback or UNDO segment 읽기
  - Control file 재구성
  - 데이터 파일 header dump 또는 읽기
- *Wait Time: 실제 I/O 시간*

## db file parallel read

- 복구 수행시, 복구해야 하는 블록들을 여러 개의 데이터 파일로부터 동시에 읽어들일 때 발생
- 버퍼 prefetching 시에도 발생
- 모든 read 요청이 끝날 때까지 대기하는 이벤트
- 이름과는 달리, parallel query나 parallel DML을 통해서는 발생하지 않음 (이 경우엔 `PX`가 붙은 이벤트가 발생)
- *Wait Time: 모든 I/O 끝날 때까지 대기한 시간*

## db file parallel write

- **DBWR**가 dirty block write I/O가 끝나기를 대기하는 이벤트
- *Wait Time: While there are outstanding I/Os, DBWR waits for some of the writes to complete. DBWR does not wait for all of the outstanding I/Os to complete.*

## CPU time

- 원활하게 일을 수행했던 service time을 의미
- Parsing/executing/fetching 등에 소요된 CPU time
- CPU time 비중이 아래쪽으로 밀려날수록 어딘가 이상이 발생했다는 걸 의미 (I/O 관련 이슈)
- Latch contention은 CPU time에 함께 계산되므로 함께 비교해야 함

```
Response Time = Service Time   +   Wait Time
              = CPU Time       +   Queue Time
```

## free buffer waits

- Free buffer를 찾으려고 시도했지만 못 찾았을 때 (after `free buffer inspected`), Oracle은 1초 간 대기했다가 다시 free buffer를 탐색함. 이 때 증가
- 세션이 dirty buffer를 dirty queue로 옮겼는데, dirty queue가 가득 찼을 때 발생
  - Dirty queue가 먼저 write 되어야 함
  - 세션은 이 이벤트가 끝날 때까지 대기하고 다시 free buffer를 찾으려고 시도함
- 파일이 읽기 전용에서 읽기/쓰기용으로 바뀌어 모든 버퍼가 suspend 상태가 될 때 발생
- *Wait Time: 1 second*

## write complete waits

- DBWR가 이미 write하고 있는 블록을 수정하기 위해, **foreground process**가 대기하는 이벤트
- DBWR가 완전히 다 쓰고 난 후에, 해당 블록을 사용할 수 있으므로 그때까지 대기
- I/O 시스템이 느린 경우 발생 가능
- DBWR의 작업량이 너무 많은 경우 발생 가능:
  - 잦은 체크포인트
  - Redo log file 크기가 작은 경우 등
- *Wait Time: 1 second*

## buffer busy waits

- 버퍼가 사용 가능할 때까지 대기
  - `buffer busy waits`: 다른 세션이 해당 버퍼를 먼저 pin한 상태이기 때문에 대기
  - `read by other session`: 다른 세션이 해당 버퍼를 디스크에서 읽어오고 있는 상태이기 때문에 대기
- *Wait Time: 1 second, 두 번째 시도부터는 3 seconds*

## pman timer

## Data Guard: Gap Manager

## Data Guard: Timer