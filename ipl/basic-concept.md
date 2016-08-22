# Basic concept

## IPL (In-Page Logging)

- data 변경 내용을 log로 기록
- page 단위의 overwrite → 각 block의 log 영역에 log sector 단위(512 bytes)의 append-only, sequential write
- 1 block = 128 KBytes = 15 data pages + 1 log page(512 bytes * 16)

## Write

1. INSERT/DELETE/UPDATE 발생
2. Buffer 상의 log sector에 변경 사항 기록
3. Data page 변경
4. Flash memory에 저장할 때는 log sector만 log page에 기록

## Read

1. 원본 data page 읽어옴 (block에 존재하는 모든 log sector read)
2. 해당 block의 log page에서 현재 data page에 대한 변경 사항 유무 확인
3. 변경 사항 있으면, 변경 사항 + data page 적용해서 buffer에 올림

### Overhead?

- Page를 buffer로 올릴 때, log page에 대한 검색 수행
- 원본 data page에 log sector의 변경 사항 적용

하지만 IPL 기법으로 인해 write/erase operation이 발생하는 횟수가 크게 줄어들기 때문에 전체적으로 봤을 땐 성능이 크게 향상됨.
