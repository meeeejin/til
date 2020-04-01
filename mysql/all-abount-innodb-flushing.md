# All about InnoDB Flushing

이 문서는 아래 두 글의 일부를 한글 번역한 문서로, 개인적인 학습을 목적으로 작성한 비공식 문서입니다.

- [Give Love to Your SSDs – Reduce innodb_io_capacity_max!](https://www.percona.com/blog/2019/12/18/give-love-to-your-ssds-reduce-innodb_io_capacity_max/)
- [InnoDB Flushing in Action for Percona Server for MySQL](https://www.percona.com/blog/2020/01/22/innodb-flushing-in-action-for-percona-server-for-mysql/)

## innodb_io_capacity 101

`innodb_io_capacity`를 메뉴얼에서 찾아보면 아래와 같이 설명되어 있습니다:

> The innodb_io_capacity variable defines the number of I/O operations per second (IOPS) available to InnoDB background tasks, such as flushing pages from the buffer pool and merging data from the change buffer.

대부분의 데이터베이스 엔진들과 마찬가지로 InnoDB는 데이터 업데이트가 메모리에서 이루어지며 수정 사항이 redo log 파일에 기록됩니다. 업데이트 된 페이지는 dirty로 표시되는데, 더 많은 데이터를 업데이트 할수록 dirty 페이지의 수가 증가하고 어느 시점엔 이 페이지들을 디스크에 기록해야합니다. 이 과정은 백그라운드로 이루어지며, *flushing* 이라고 부릅니다. `innodb_io_capacity`는 InnoDB가 페이지를 flush 하는 속도를 정의합니다. 더 자세한 설명을 위해, 다음 그래프를 살펴보겠습니다:

![inno-io-capa-idle](https://www.percona.com/blog/wp-content/uploads/2019/12/innodb_io_capacity_with_labels.png)
> Impacts of innodb_io_capacity on idle flushing

위 그래프는 *sysbench* 툴을 사용해 몇 초 간 약 45,000개의 dirty 페이지를 버퍼 풀에 생성한 다음, 세 가지 `innodb_io_capacity` 값 (300, 200, 100)에 대해 flushing 프로세스를 실행한 결과를 나타냅니다. 보다시피, 초당 쓰여지는 페이지 개수는 `innodb_io_capacity` 값과 일치합니다. 이러한 유형의 flushing을 *idle flushing* 이라고 합니다. *Idle flushing*은 InnoDB가 write를 처리하지 않을 때만 발생합니다

`innodb_io_capacity`는 adaptive flushing과 secondary index 업데이트의 백그라운드 merge를 위한 change buffer thread에도 사용됩니다. Adaptive flushing이 활성화된 경우, busy 서버에서는 `innodb_io_capacity_max` 변수가 훨씬 중요합니다. 

## Are Dirty Pages Evil?

데이터베이스가 종료되기 전에 모든 dirty 페이지를 flush 해야하므로 dirty 페이지가 많으면 MySQL shutdown 시간이 늘어납니다. 또한 dirty 페이지의 양은 crash 후 복구 시간에도 영향을 끼치는데, 이는 약간 예외적입니다.

버퍼 풀에서 페이지가 일정 시간 동안 dirty 상태로 유지되면, 디스크로 flush 되기 전에 추가적인 write를 receive 할 수 있습니다. 그 결과, write load가 감소됩니다. 이러한 write load reduction에 취약한 스키마 및 쿼리 패턴이 있습니다. 예를 들어, 다음 스키마를 갖는 테이블에 수집된 metric을 insert하는 경우:

```bash
CREATE TABLE `Metrics` (
  `deviceId` int(10) unsigned NOT NULL,
  `metricId` smallint(5) unsigned NOT NULL,
  `Value` float NOT NULL,
  `TS` int(10) unsigned NOT NULL,
  PRIMARY KEY (`deviceId`,`metricId`,`TS`),
  KEY `idx_ts` (`TS`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
```

각각 8개의 metric을 갖는 20k개의 디바이스가 있다고 가정했을 때, 160k의 hot 페이지가 있는 것입니다. 이상적으로, 이 페이지들은 꽉 차기 전엔 디스크에 쓰이지 않아야합니다. 하지만, 이 페이지들은 mid-inserted b-tree의 일부이기 때문에 실제로는 절반만 꽉 찹니다.

또 다른 예는 마지막 활동 시간이 기록되는 *users* 테이블입니다. 일반적인 스키마는 다음과 같습니다:

```bash
CREATE TABLE `users` (
  `user_id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `last_login` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `last_activity` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `user_pref` json NOT NULL,
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=6521901 DEFAULT CHARSET=latin1
```

특정 시간에 사용자의 일부 하위 집합만 active하므로, 사용자들이 어플리케이션을 사용함에 따라 동일한 페이지가 여러 번 업데이트 됩니다. 이를 더 자세히 설명하기 위해, 위의 스키마를 사용하고 약 6.5M row 중 30k의 랜덤한 하위 집합만 active하게 업데이트하는 실험을 수행해봤습니다. 실험에서 사용된 설정은 아래와 같습니다:

```bash
innodb_adaptive_flushing_lwm = 0  
innodb_io_capacity = 100
Innodb_flush_neighbors = off
```

각 run마다 `innodb_io_capacity_max` 값을 다르게 설정했고 30분 동안 flush 된 페이지에 대한 업데이트 비율을 계산했습니다.

<p align="center">
<img src="https://www.percona.com/blog/wp-content/uploads/2019/12/UpdatePerPageFlushedIOCapMax.png" alt="max-graph"/>
</p>

보시다시피, `innodb_io_capacity_max`를 100으로 제한하면 flush 된 페이지당 약 62개의 업데이트가 반영되는 반면, 5,000으로 설정된 경우 페이지당 약 20개의 업데이트만 존재합니다. 즉, `innodb_io_capacity_max`를 조정하였더니 write load가 3배 차이까지 났습니다.

## Impacts of Excessive Flushing on Performance

InnoDB 페이지가 디스크로 flush 되는 과정에서는 일부 페이지에 대한 access가 제한되고 해당 내용을 필요로 하는 쿼리는 I/O 작업이 완료될 때까지 기다려야 합니다. 또한 write load가 너무 크면 스토리지 및 CPU 리소스에 부담이 됩니다. `innodb_io_capacity_max`를 변경한 위의 실험에서 업데이트 속도는 6,000 trx/s (`innodb_io_capacity_max = 100`) 이상에서 5,400 trx/s (`innodb_io_capacity_max = 4000`) 미만까지 떨어졌습니다. 이는 단순히 `innodb_io_capacity`와 `innodb_io_capacity_max` 값을 크게 설정한다고 성능이 optimal 해지진 않는다는 걸 의미합니다.

## InnoDB Flushing in Action for MySQL

### Idle Flushing

이미 위에서 [idle flushing](#innodb_io_capacity-101)에 대해 논의했습니다. 정리하자면, InnoDB는 write operation이 없으면 (즉, LSN 값이 변하지 않으면) `innodb_io_capacity` 속도로 dirty 페이지를 flush 합니다.

### Dirty Pages Percentage Flushing

이 flushing은 몇년 전에 사용된 기존 InnoDB flushing 알고리즘의 약간 수정된 버전입니다. MySQL을 사용해왔다면, 아마도 이 알고리즘이 마음에 들지 않을 겁니다. 알고리즘은 아래의 변수에 의해 컨트롤 됩니다:

```bash
innodb_io_capacity (default value of 200) 
innodb_max_dirty_pages_pct (default value of 75) 
innodb_max_dirty_pages_pct_lwm (default value of 0 in 5.7 and 10 above 8.0.3)
```

버퍼 풀의 총 페이지 수에 대한 dirty 페이지의 비율이 low watermark (lwm)보다 높은 경우, InnoDB는 dirty 페이지의 실제 percentage를 `innodb_max_dirty_pages_pct` 값으로 나눈 후 `innodb_io_capacity`를 곱한 값의 속도로 페이지를 flush 합니다. 실제 dirty 페이지 percentage가 `innodb_max_dirty_pages_pct`보다 높으면, flush 속도가 `innodb_io_capacity`로 제한됩니다.

<p align="center">
<img src="https://www.percona.com/blog/wp-content/uploads/2020/01/rateDirtyPct.png" alt="lwm-equation" width="600"/>
</p>

이 알고리즘의 주요한 문제점은 right thing에 초점을 맞추고 있지 않다는 것입니다. 위의 식을 사용할 경우, max checkpoint age에 도달해 트랜잭션 처리가 정지될 수 있습니다. 이와 관련해, 아래는 2011년 Vadmin Tkachenko가 작성한 게시물의 그래프입니다: [InnoDB Flushing: a lot of memory and slow disk](https://www.percona.com/blog/2011/03/31/innodb-flushing-a-lot-of-memory-and-slow-disk/)

<p align="center">
<img src="https://www.percona.com/blog/wp-content/uploads/2020/01/vadim_post_notp.png" alt="adf-vadim"/>
</p>

time = 265일 때, NOTP (New Order Transaction per second)가 급격히 감소한 걸 볼 수 있습니다. 이는 InnoDB가 max checkpoint age에 도달하여 페이지를 과도하게 flush 해야하기 때문입니다. 이를 *flush storm* 이라고 합니다. 이러한 storm은 쓰기 작업을 중지시키고 데이터베이스 operation에 악영향을 끼칩니다. Flush storm에 대한 자세한 내용은 [InnoDB Flushing: Theory and solutions](https://www.percona.com/blog/2011/04/04/innodb-flushing-theory-and-solutions/)를 참고하시기 바랍니다.

### Free List Flushing

읽기 및 페이지 생성 속도를 높이기 위해 InnoDB는 각 버퍼 풀 인스턴스에 항상 일정 수의 free 페이지를 보유하려고 합니다. Free 페이지가 없으면, InnoDB는 새 페이지를 로드하기 전에 다른 하나의 페이지를 free 해야 합니다.

이 flushing은 `innodb_lru_scan_depth`에 의해 컨트롤 됩니다. 일정 주기마다 각 버퍼 풀 인스턴스의 LRU list에서 가장 오래된 페이지들이 스캔되는데, 이 변수 값까지 스캔됩니다. 이 때 스캔된 페이지가 clean 페이지면 바로 free 되고, dirty 페이지면 free 되기 전에 flush 됩니다.

### Adaptive Flushing

Adaptive flushing 알고리즘은 InnoDB의 major improvement로, MySQL이 상당햔 양의 write load를 적절한 방식으로 처리할 수 있도록 도와줍니다. 기존 알고리즘은 dirty 페이지 개수를 확인하는 반면, adaptive flushing은 checkpoint age에 초점을 맞춥니다. 

아래 설명 중 일부는 Percona Server for MySQL 8.0.18-에 유효합니다.

#### Some Background

InnoDB는 보통 16KB 페이지에 row를 저장합니다. 이 페이지는 디스크, 데이터 파일 또는 InnoDB 버퍼 풀 (즉, 메모리)에 존재합니다. InnoDB는 버퍼 풀에서만 페이지를 수정합니다.

버퍼 풀의 페이지가 쿼리에 의해 업데이트되면 dirty 페이지가 됩니다. Commit 시 페이지의 수정 사항이 redo log에 기록됩니다. Write 후, LSN (Last Sequence Number)이 증가합니다. Dirty 페이지는 디스크로 즉시 flush 되지 않고 한동안 dirty 상태로 유지됩니다. 이렇게 지연된 페이지 flushing은 일반적인 performance hack입니다.

이제 InnoDB Redo Log의 구조를 살펴보겠습니다. 

<p align="center">
<img src="https://www.percona.com/blog/wp-content/uploads/2020/01/ring_buffer_v2.png" alt="redo" width="600"/>
</p>

> The InnoDB redo log files form a ring buffer

InnoDB 로그 파일은 flush 되지 않은 수정 사항을 포함하는 링 버퍼를 형성합니다. *Head* 는 InnoDB가 현재 트랜잭션 데이터를 쓰는 위치를 가리킵니다. *Tail* 은 가장 오래된 flush 되지 않은 데이터 수정 사항을 가리킵니다. *Head* 와 *Tail* 사이의 간격은 checkpoint age를 의미합니다. Checkpoint age는 byte로 표현됩니다. 로그 파일의 크기와 개수에 따라 max checkpoint age가 결정되며, max checkpoint age는 로그 파일 크기의 약 80% 입니다.

페이지 flushing이 *Tail* 을 이동시키는 동안 쓰기 트랜잭션이 *Head* 를 앞으로 이동시킵니다. *Head* 가 너무 빨리 움직이고, *Tail* 전에 사용 가능한 공간이 12.5% 미만이면, 로그 파일에서 일부 공간이 비워질 때까지 트랜잭션이 더 이상 commit 될 수 없습니다. 즉, 이때 InnoDB는 *flush storm* 을 통해 페이지를 빠른 속도로 flush 합니다. 이러한 *flush storm* 상황은 최대한 피해야 합니다.

#### How the Adaptive Flushing Algorithm Works

Adaptive flushing은 아래 변수에 의해 컨트롤 됩니다:

```bash
innodb_adaptive_flushing  (default ON)
innodb_adaptive_flushing_lwm  (default 10)
innodb_flush_sync (default ON)
innodb_flushing_avg_loops (default 30)
innodb_io_capacity (default value of 200)
innodb_io_capacity_max (default value of at least 2000)
innodb_cleaner_lsn_age_factor (Percona server only, default high_checkpoint)
```

알고리즘의 목표는 flushing 속도 (*Tail* 속도)를 checkpoint age (*Head* 속도)가 변화하는 속도에 맞게 조정하는 것입니다. Checkpoint age가 adaptive flushing low watermark (default: max checkpoint age의 10%)보다 높으면 flushing이 시작됩니다. Percona Server for MySQL은 *Legacy* 및 *High Checkpoint*의 두 가지 알고리즘을 제공합니다.

*Legacy* 알고리즘은 다음과 같습니다. Age factor의 3/2의 제곱과 분모 7.5를 기억하세요:

<p align="center">
<img src="https://www.percona.com/blog/wp-content/uploads/2020/01/legacy_equation.png" alt="leagcy-algo" width="600"/>
</p>

> Legacy age factor

*High Checkpoint* 알고리즘은 다음과 같습니다:

<p align="center">
<img src="https://www.percona.com/blog/wp-content/uploads/2020/01/high_checkpoint_equation.png" alt="hc-algo" width="600"/>
</p>

> Percona High-checkpoint age factor

이번에는 age factor의 제곱 값이 5/2이고 분모는 700.5입니다. 두 방정식에서 `innodb_io_capacity` (*ioCap*)는 `innodb_io_capacity_max` (*ioCapMax*)와의 비율 값에서 분모에 해당합니다. 두 방정식을 함께 그려보면 다음과 같습니다:

<p align="center">
<img src="https://www.percona.com/blog/wp-content/uploads/2020/01/FlushingPressure.png" alt="both-graph"/>
</p>

> Flushing pressure for the legacy and high-checkpoint algorithm

그래프는 default 값인 `ioCapMax / ioCap = 10`를 기준으로 그렸습니다. Percona의 High Checkpoint는 느리게 시작되지만 빠르게 증가합니다. 이는 더 많은 dirty 페이지를 허용하고 일반적으로 좋은 성능을 보입니다.

#### Average Over Time

지금까지는 checkpoint age만 방정식에 사용되었습니다. 이 방법도 좋지만, 알고리즘의 목표는 redo 로그 링 버퍼의 *Tail* 이 *Head* 와 거의 같은 속도로 움직이도록 페이지를 flush 하는 것입니다. 매 `innodb_flushing_avg_loops` 초마다 flush 된 페이지 비율과 redo 로그 *Head* 의 진행률이 측정되고 새 값은 이전 값으로 평균화됩니다. 여기서 목표는 알고리즘에 약간의 관성을 부여하여 변화 정도를 완화하는 것입니다. `innodb_flushing_avg_loops`의 값이 높을수록 알고리즘의 반응 속도가 느려지고 값이 작을수록 반응성이 높아집니다. 이 수치를 *avgPagesFlushed* 및 *avgLsnRate* 라고 합니다.

#### Pages to Flush for the avgLsnRate

*avgLsnRate* 값을 기반으로 InnoDB는 버퍼 풀에서 가장 오래된 dirty 페이지를 스캔하고, *Tail*의 *avgLsnRate* 보다 작은 페이지 수를 계산합니다. 이 값을 *pagesForLsnRate* 라고 하겠습니다.

#### Finally…

이 모든 걸 종합하면, flush 될 실제 페이지 수는 다음과 같습니다:

<p align="center">
<img src="https://www.percona.com/blog/wp-content/uploads/2020/01/finalPagesToFlush.png" alt="equation-final" width="600"/>
</p>

이 값은 *ioCapMax*로 제한됩니다. 보시다시피 *pctOfIoCapToUse* 에는 *ioCap* 이 곱해집니다. *pctOfIoCapToUse* 를 계산하는 식을 되돌아보면 분모에 *ioCap* 이 있습니다. 따라서 *ioCap* 을 다시 곱했고, 이러한 adaptive flushing 알고리즘은 `innodb_io_capacity_max`만 중요하므로 `innodb_io_capacity`와 **독립적**입니다.