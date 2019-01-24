# TPC-E vs. TPC-C

아래 두 자료 참고함.

- S. Chen, A. Ailamaki, M. Athanassoulis, P. B. Gibbons, R. Johnson, I. Pandis, and R. Stoica. [TPC-E vs. TPC-C: Characterizing the New TPC-E Benchmark via an I/O Comparison Study](http://www.cs.cmu.edu/~chensm/papers/TPCE-sigmod-record10.pdf). ACM SIGMOD Record, 39(3):5–10, Feb. 2011.
- [TPC-E](http://www.tpc.org/tpce/)

## TPC-E

TPC-E는 실제 데이터 skewness와 참조 무결성 제약 조건(referential integrity constraint)을 함께 고려하여, TPC-C보다 더 현실적인 워크로드를 목표로 설계된 OLTP 벤치마크의 일종이다. TPC-E는 12개의 동시 트랜잭션의 mix 및 33개의 테이블로 구성된다.

<p align="center">
<img src="https://www.researchgate.net/profile/Manos_Athanassoulis/publication/49459513/figure/fig1/AS:341390627229709@1458405276165/TPC-E-models-a-financial-brokerage-house-Figure-is-adapted-from-8.png" alt="TPC-E" />
</p>

TPC-E는 금융 중개소를 모델로 한 벤치마크로, 고객(customers), 중개소(brokerage house), 증권 거래소(stock exchange)의 3가지 컴포넌트로 구성되어 있다. TPC-E는 중개소를 지원하는 데이터베이스 시스템이며, 고객과 증권 거래소는 중개소에서 거래를 유도하기 위해 시뮬레이션된다.

TPC-E 데이터베이스에서 중개소는 고객, 브로커, 금융 시장에 대한 정보를 가지고 있다. TPC-E에는 2가지의 유형의 주요한 트랜잭션이 있는데, 하나는 고객이 만들어내는 트랜잭션이고 다른 하나는 시장에서 발생하는 트랜잭션이다.

1. Customer-initiated transaction

고객이 중개소에 요청을 보내고, 데이터베이스에 쿼리를 수행하거나 업데이트를 수행한 후, 중개소의 요청을 증권 거래소에 제출한다. 그 다음, 증권 거래소에서 받은 응답을 고객에게 전송한다. TPC-E는 즉각적이고 제한이 있는 있는 거래 주문을 모두 지원한다.

2. Market-triggered transaction

증권 거래 시장은 거래 결과 또는 시세표를 중개소에 보낸다. 중개소는 데이터베이스를 업데이트하고, 기록된 한도 제한 거래 주문을 확인하고, 만약 가격 한도까지 오르면 해당 주문을 보낸다.

## TPC-E와 TPC-C 비교

|                 | TPC-E           | TPC-C           |
| :-------------: | :-------------: | :-------------: |
| Business model  | Brokerage House | Wholesale supplier |
| Tables          | 33              | 9               |
| Columns         | 188             | 92              |
| Columns/Tables  | 2-24, avg 5.7   | 3-21, avg 10.2  |
| Transaction mix | 4 RW (23.1%), 6 RO (76.9%) | 3 RW (92%), 2 RO (8%) |
| Data generation | Pseudo-real, based on census data | Random |
| Check constraint| 22              | 9               |
| Referential integrity| Yes        | No              |

- TPC-E는 금융 중개소를, TPC-C는 도매상을 모델로 함
- TPC-E는 TPC-C보다 3배 이상 많은 테이블을 가짐
  - 고객 계좌 정보 기록용 9개 + 중개인과 거래 정보 기록용 9개 + 시장 관련된 테이블 11개 + 주소/우편 번호 같은 고정된 정보를 위한 차원 테이블 4개
  - 33개의 테이블 중 19개는 고객의 수만큼 scale 되고, 5개의 테이블은 TPC-E 런타임 시에 증가하며, 나머지 9개 테이블은 static 함
- TPC-E는 TPC-C보다 2배 많은 컬럼을 가짐
  - 하지만, TPC-E 테이블은 평균 5.7개의 컬럼을 가지는데 이는 TPC-C의 절반 수준임
  - 즉, TPC-E가 복잡한 entity relationship으로 인해 더 많은 테이블을 가지지만, TPC-E 개별 테이블의 구조는 상대적으로 단순함
- TPC-E는 TPC-C보다 2배 많은 트랜잭션 유형을 가짐
  - TPC-E의 경우 76.9%가 read-only인 것에 반해, TPC-C는 8%가 read-only 트랜잭션임
  > TPC-E는 TPC-C보다 **read intensive** 함
- TPC-E는 전수 조사 및 증권 거래소 데이터에 기반해 데이터를 생성하기 때문에, 실세계의 데이터 skewness를 반영함 (**Pseudo-real data**)
- TPC-E는 실제 OLTP 어플리케이션에 존재하는 check constraint, referential integrity 등의 특징을 잘 반영하였음

### Hard Disk trace를 사용한 비교

> SQL Server 이용

#### 실험 환경

| TPC-C           | TPC-E           |
| --------------- | --------------- |
| 64GB memory     | 128GB memory    |
| 4 dual-core 3.4 GHz Xeons system | 4 quad-core 1.8 GHz Opterons system |
| 392 15Krpm SCSI drives (14 RAID-0, 28 disks each) | 336 15Krpm SCSI drives (12 RAID-0, 28 disks each) |
| 14,000 warehouses | 200,000 customers |
| 300 users       |                 |
| 5 minutes long  | 10 minutes long |

#### Spatio-Temporal Graphs

<p align="center">
<img src="https://i.imgur.com/NPl0xeG.png" alt="spatio-temporal" />
</p>

- 로그 디바이스는 sequential write로 인해 대각선 형태를 보임
- 데이터 디바이스는 넓게 분포된 scattered access를 보임

#### Number of I/Os per Second

<p align="center">
<img src="https://i.imgur.com/CgpaZpQ.png" alt="iops" height=200/>
</p>

- TPC-C는 평균 3360 IOPS, TPC-E는 평균 2740 IOPS를 보임
- I/O rate의 표준 편차는 TPC-C는 2.2%, TPC-E는 3.6%를 보임

#### I/O Request Breakdown

<table>
  <tr>
  <td colspan="1"></td>
  <td colspan="3"><center>TPC-C</td>
  <td colspan="3"><center>TPC-E</td>
  </tr>
  <tr>
    <td>Size</td>
    <td>Read</td>
    <td>Write</td>
    <td>Both Types</td>
    <td>Read</td>
    <td>Write</td>
    <td>Both Types</td>
  </tr>
  <tr>
    <td>8KB</td>
    <td>65.70%</td>
    <td>32.66%</td>
    <td>98.36%</td>
    <td>90.68%</td>
    <td>8.29%</td>
    <td>98.97%</td>
  </tr>
  <tr>
    <td>16KB</td>
    <td>0.00%</td>
    <td>1.49%</td>
    <td>1.49%</td>
    <td>0.00%</td>
    <td>0.79%</td>
    <td>0.79%</td>
  </tr>
  <tr>
    <td>Other Sizes</td>
    <td>0.00%</td>
    <td>0.15%</td>
    <td>0.15%</td>
    <td>0.01%</td>
    <td>0.23%</td>
    <td>0.24%</td>
  </tr>
  <tr>
    <td>All Sizes</td>
    <td>65.71%</td>
    <td>34.29%</td>
    <td>100.00%</td>
    <td>90.69%</td>
    <td>9.31%</td>
    <td>100.00%</td>
  </tr>
</table>

- DBMS의 default I/O 크기가 8KB이기 때문에, TPC-C와 TPC-E 모두 8KB I/O가 대부분임
- 연속된 디스크 블록에 write하여 merge되는 경우가 존재해 다른 크기의 write도 존재함
> Random I/O를 유발하는 TPC-C의 경우 설명 가능하지만, TPC-E의 경우는 직관에 반하는 결과임
- I/O 요청 유형만 보면, TPC-C는 read:write가 1.9:1, TPC-E는 read:write가 9.7:1 비율을 보임
> TPC-E는 read-only 트랜잭션의 비율이 높게 설계되어 TPC-C보다 read intensive 함

#### Temporal Locality

- I/O read R2가 이전 I/O read R1과 같은 블록 주소에서 발생할 때, 이를 *read reuse* 라고 정의함
- R1을 이슈하고 R2를 이슈하기까지 걸린 시간을 R2에 대한 *reuse distance* 라고 정의함 (같은 블록에 대해 가장 최근에 발생한 이전 I/O read를 R1으로 생각함)
- *Write reuse* 와 *write reuse distance* 도 마찬가지로 정의함
- 일반적으로 *reuse distance* 가 짧고 reuse 비율이 높을수록 temporal locality에 대한 I/O 최적화가 더 적합함

<p align="center">
<img src="https://i.imgur.com/kuIQqKD.png" height=200/>
<img src="https://i.imgur.com/1MHdCni.png" height=200/>
</p>

- 위 그래프는 read/write reuse에 대한 누적 분포 그래프로, 100%는 read/write access의 전체 횟수를 의미함
- 실험상으로는 TPC-C, TPC-E 모두 큰 temporal locality를 보여주진 않음
- TPC-E는 TPC-C보다 적은 read reuse를 보이는데, 이는 TPC-E가 더 많은 data skewness를 갖도록 설계되었다는 설명과는 상반됨
- 두 벤치마크 모두 I/O reuse가 약 60초 쯤부터 나타나기 시작했다는 점에서, buffer pool이 reuse 패턴에 영향을 끼친다는 것을 짐작할 수 있음

#### Understanding the Buffer Pool Behavior

> :warning: 논문 리뷰 때 지적받은 내용과 약간 다른 의미로 write-after-read를 정의함

- 이전의 read I/O R과 같은 블록 주소에 Write I/O W가 요청되고, R과 W 사이에 해당 블록에 대한 다른 I/O가 없는 경우를 *write-after-read* 라고 정의함
- 그래프의 *read to write distance* 는 R을 이슈하고 W를 이슈하기까지 걸린 시간을 의미함
- 따라서, *read to write distance* 는 페이지가 버퍼 풀에 유지되는 시간을 의미함

<p align="center">
<img src="https://i.imgur.com/wMx7qJF.png" height=200/>
</p>

- 위 그래프는 write-after-read I/O에 대한 누적 분포 그래프로, 두 그래프 모두 60초 쯤에 그래프가 튀는 걸 볼 수 있음
- 이는 페이지를 다시 쓰기 전까지 60초 정도 페이지를 버퍼 풀에 유지하고 있음을 의미함
> TPC-E의 poor temporal locality의 원인 (TPC-E 전체 데이터 크기의 8% 크기의 메모리로 데이터를 캐싱하여, skew된 데이터 접근을 메모리 상에서 완화시킴)
- DBMS의 configuration에 따라 결정되는 요소임

#### Spatial Locality

<p align="center">
<img src="https://www.researchgate.net/profile/Manos_Athanassoulis/publication/49459513/figure/fig2/AS:341390627229713@1458405276413/Cumulative-distributions-for-number-of-read-I-Os-to-a-unit-while-varying-unit-size-from.png" height=200/>
</p>

- 곡선이 가파르다는 것은 대부분의 unit에 적은 횟수의 read가 발생한다는 의미로, poor spatial locality를 의미함

| Unit Size | 8KB blocks | TPC-E 80-th | TPC-E 99-th | TPC-C 80-th | TPC-C 99-th |
| --------: | ---------: | ----------: | ----------: | ----------: | ----------: |
| 8KB       | 1          | 1           | 2           | 1           | 2           |
| 16KB      | 2          | 1           | 3           | 2           | 3           |
| 64KB      | 8          | 2           | 5           | 2           | 6           |
| 256KB     | 32         | 4           | 10          | 4           | 14          |
| 1MB       | 128        | 8           | 29          | 10          | 28          |

- TPC-E의 경우, 128개의 8KB 블록으로 구성된 1MB 유닛에 대해, 전체 트레이스를 통틀어 최대 29개의 블록만 접근된다는 것을 알 수 있음
- TPC-E의 경우, 전반적으로 poor spatial locality를 보여줌

#### Log Write Sizes

<p align="center">
<img src="https://www.researchgate.net/profile/Manos_Athanassoulis/publication/49459513/figure/fig3/AS:341390627229717@1458405276434/Cumulative-distribution-of-log-writes-Log-Write-Size-For-the-most-part-the-sequential.png" height=200/>
</p>

- TPC-C와 TPC-E 모두 약 80%의 log write가 20KB보다 작음
- 두 벤치마크 모두 가장 큰 log write가 60KB였음
> :bulb: TPC-C의 경우, 7%의 log-write가 60KB였음
- DBMS 로깅 매니저가 최대 60KB log write를 flush하도록 설정되어서 위의 그래프를 보임

### SSD trace를 사용한 비교

> :bulb: 다른 configuration으로 replay 필요

TPC-E도 TPC-C만큼 랜덤한 I/O 패턴을 만들어낼 수 있음을 위에서 보였는데, 이는 TPC-E도 SSD를 사용할 경우 성능 향상의 가능성이 있다는 것을 암시한다.

#### 실험 환경

| SSD-only        |
| :-------------: |
| 2GB memory      |
| Dell Precision 390 workstation |
| 2.66GHz Intel Core 2 CPU |
| 32GB Intel X25-E SSD |

> Random read = 35,000 IOPS, random write = 3,300 IOPS

#### 실험 결과

<p align="center">
<img src="https://www.researchgate.net/profile/Manos_Athanassoulis/publication/49459513/figure/fig4/AS:668545323118613@1536405034868/OLTP-trace-replay-on-an-Intel-X25-E-SSD.png" height=200/>
</p>

- y축은 평균 I/O latency
- TPC-E가 TPC-C만큼 랜덤한 I/O 접근 패턴을 보이기 때문에, SSD를 사용하여 성능 향상 가능


## 요약
- TPC-E는 TPC-C보다 더 복잡하고, 현실적인 OLTP 벤치마크 툴임
- TPC-E는 `read : write = 9.7 : 1`, TPC-C는 `read : write = 1.9 : 1`로, TPC-E가 TPC-C보다 **read intensive** 함
- 본 논문의 실험 환경에서는 두 벤치마크 모두 poor temporal/spatial locality를 보임
- TPC-E가 실제 데이터 skewness를 갖도록 설계되었지만, skew된 접근은 메모리 버퍼 풀에 의해 잘 캐싱될 수 있음. 결과적으로, TPC-E의 I/O 접근 패턴도 TPC-C만큼 랜덤할 수 있음을 보임
- :bulb: TPC-E의 높은 *read to write* 비율은 write를 대상으로 하는 I/O 최적화가 TPC-E에서는 덜 효과적일 수 있음을 의미함
- :bulb: TPC-E의 랜덤 I/O 접근 패턴은 TPC-C에 대한 이전의 많은 I/O 연구의 결론들이 여전히 유효함을 의미함
