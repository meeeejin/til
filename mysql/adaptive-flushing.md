# InnoDB adaptive flushing in MySQL 5.6: checkpoint age and io capacity

> [원문(2013/10/30)](https://www.percona.com/blog/2013/10/30/innodb-adaptive-flushing-in-mysql-5-6-checkpoint-age-and-io-capacity/)

MySQL 5.6에서 InnoDB는 플러싱 작업을 수행하는 전용 스레드(`page_cleaner`)를 가진다. `Page_cleaner`는 다음 두 가지 요소를 기반으로 버퍼 풀에서 dirty 페이지를 플러시 한다.

- **Access pattern**: 버퍼 풀에 free 페이지가 더 이상 존재하지 않을 때, LRU 플러셔는 LRU list에서 가장 오래 전에 사용된 페이지를 찾아 플러시 한다.
- **Age**: 수정된 지 가장 오래되었고 아직 플러시 되지 않은 페이지들은 flush list에 존재하며, 몇 가지 휴리스틱에 기반하여 flush list 플러셔에 의해 플러시 된다.

## Flush list flushing and checkpoint age

Flush list에 저장할 수 있는 오래된(aged) 페이지의 양은 InnoDB의 전체 로그 파일 사이즈에 의해 제한된다. 따라서, flush list 플러싱의 주요 목적은 이 리스트의 페이지를 로그 파일에 충분한 여유 공간을 만들 수 있는 속도로 플러시하는 것이다. 반면, 너무 적극적인 플러싱(aggressive flushing)은 쓰기 결합을 줄이고 I/O 서브시스템에 대한 불필요한 로드를 유발하여, 결국 더 큰 redo 로그를 사용함으로써 얻는 성능 이점을 없애버린다. MySQL 5.6에서 플러시 할 페이지의 양은 현재 체크포인트 경과 시간을 기반으로 InnoDB adaptive 루틴에서 계산된다. 이 때 사용되는 수식은 다음과 같다:

```bash
percentage of the IO capacity that should be used for flushing =
        ((srv_max_io_capacity / srv_io_capacity) * (lsn_age_factor * sqrt(lsn_age_factor))) / 7.5;
```

우리는 R을 이용해 위 수식을 모델링하였는데, 곡선을 더 평평하게 만들어 결과적으로 플러싱이 덜 aggressive 하도록 개선할 수 있음을 확인하였다. [새로운 공식](https://www.percona.com/doc/percona-server/5.6/performance/page_cleaner_tuning.html#adaptive-flushing-tuning)은 Percona Server 5.6에서 디폴트로 켜져 있다.

![newformula](https://www.percona.com/blog/wp-content/uploads/2013/10/Rplot04.png)

## Flush list flushing and io_capacity

InnoDB는 백그라운드 플러싱 속도를 제어할 수 있는 두 개의 변수(`innodb_io_capacity`와 `innodb_io_capacity_max`)를 제공한다. 이 변수들에 대한 자세한 설명은 [여기](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_io_capacity)서 확인할 수 있다. 그러나 문서에서 다뤄지지 않은 몇 가지 사항이 있다.

- **innodb_io_capacity_max** 는 플러싱 속도를 실제로 제한하는 유일한 변수로, adaptive 플러싱의 경우 가장 중요한 변수이다. 위 수식 및 그림 참고.
- **innodb_io_capacity** 는 서버가 비활동 또는 종료 상태인 경우, insert 버퍼의 병합과 플러싱 중에 I/O 작업을 제한하는 데에 사용된다.

이 변수를 실질적으로 사용하자면, 위 내용은 다음과 같은 의미를 가진다.

- MySQL 서버가 active한 상태(사용자 요청을 처리 중)인 경우, 플러싱 속도를 높이거나 낮추기 위해 `innodb_io_capacity_max`를 조정해야 한다.
- MySQL 서버가 idle(유휴) 상태이거나 종료 중인 경우, flush list에서 페이지를 비우는 것은 `innodb_io_capacity` 값에 의해서만 제한된다.

- `change_buffering`이 켜져 있고 서버가 active 상태라면, `innodb_io_capacity`의 5%를 사용하는 것 또는 insert 버퍼 크기의 50% 이상이 이미 사용되었다면 그 비율이 5%에서 55%까지 변화하는 것을 허용한다.
- `change_buffering`이 켜져 있고 서버가 idle 상태라면, 병합(merge) 작업에 `innodb_io_capacity`를 100% 사용한다.

# Dirty pages, fast shutdown, and write combining

> [원문(2010/05/25)](https://www.xaprb.com/blog/2010/05/25/dirty-pages-fast-shutdown-and-write-combining/)

전통적인 트랜잭션 데이터베이스를 고가용성으로 만드는 것을 어렵게 하는 것 중 하나는 비교적 느린 종료(shutdown) 및 시작(start-up) 시간이다. 어플리케이션은 일반적으로 대부분의 또는 모든 쓰기 작업을 데이터베이스에 위임한다. 이 작업은 (대게 큰) 메모리에 많은 "dirty" 데이터를 가지고 수행되는 경향이 있다. 종료 시에, dirty 메모리는 디스크로 쓰여야 하기 때문에, 시작 시 복구 과정을 실행할 필요가 없다. 그리고 clean start-up 시에도 데이터베이스가 warm up 해야 할 가능성이 있는데, 이는 또한 매우 오랜 시간이 걸릴 수 있다.

일부 데이터베이스는 운영 체제가 대부분의 메모리 관리 요구를 처리하도록 한다. 그러나 운영 체제의 디자인이 데이터베이스의 목표와 정확히 일치하지 않는 경우에는 문제가 있다.

다른 데이터베이스는 문제를 스스로 해결한다. InnoDB(사실상 트랜잭션형 MySQL 스토리지 엔진)는 이 범주에 속한다. InnoDB가 최신 하드웨어를 적절히 사용하도록 설정하면, 기본적으로 거대한 버퍼 풀의 서버 메모리를 전부 사용하는데, 이 때 버퍼는 I/O 작업을 위해 운영 체제를 우회(bypass)하는 `O_DIRECT` 모드로 열린 파일을 갖는다.

디자인 선택과 그에 따른 결과는 생각할 만한 가치가 있다. 종료 및 재시작을 드물게 한다고 가정할 때, dirty 메모리를 많이 보유하려는 선택은 큰 성능 이점을 가지는데, 이는 빠른 종료와 복구에 대한 요구와 균형을 이루어야 한다. InnoDB에는 시작 및 종료 동작을 변경할 수 있는 몇 가지 설정이 있지만, 이 설정들이 정상 동작 중에 성능에 미치는 영향을 이해해야 한다.

먼저, 많은 dirty 데이터를 메모리에서 실행하는 것이 좋은 이유를 살펴 보자.

## Write combining

대부분의 데이터베이스에는 페이지, 버퍼 또는 블록이라는 개념이 있다. 일반적으로 이것은 많은 논리 단위(행,row)를 저장할 수 있는 데이터의 물리적 단위이다. InnoDB의 디폴트 값은 16KB 페이지이다. 이 때, 일반적인 행이 80 바이트 길이라고 상상해보자. 대부분의 경우에 많은 행이 16KB에 들어갈 수 있다.

행을 삽입, 갱신 또는 제거한다고 가정하자. InnoDB가 그 결과를 디스크게 즉시 써야 할까? 그렇다면, 전체 16KB 페이지와 다른 인덱스 페이지 등을 써야 하며, 많은 페이지를 추가로 써야 할 수도 있다. 이러한 많은 작업은 80 바이트를 위한 것이다! 이를 피하기 위해, InnoDB는 수정된 페이지를 dirty한 상태로 메모리에 남겨둔다. 대신 트랜잭션을 커밋할 때, WAL 로그가 crash가 발생해도 변경 사항이 계속 유지되도록 보장한다. (로그는 매우 컴팩트한 요소이며 page-oriented 하지 않다.)

이제 당신이 조금 더 변화를 주었다고 가정하자. 대부분의 경우, 두 가지 변경 사항이 동일한 페이지를 건드릴 확률이 높다. 실제로 이를 증명할 통계를 뽑아보면, 대부분의 변경 사항이 전체 페이지의 일부 또는 행의 일부에 집중된다는 것을 알 수 있다. 대부분의 워크로드는 `매우 큰 머리와 매우 긴 꼬리(a very tall head and a very long tail, 파레토 법칙)`를 가진다. 파레토 법칙을 그래프로 나타내면 아래와 같고, 초록색 부분이 큰 머리이고 노란색 부분이 긴 꼬리이다. 덜 active한 페이지 및 행과 비교했을 때, 수십, 수백, 심지어 수천 배의 변경 사항이 동일한 페이지 및 행에 집중된다.

![pareto-principle](https://upload.wikimedia.org/wikipedia/commons/thumb/8/8a/Long_tail.svg/500px-Long_tail.svg.png)

궁극적으로, 우리가 선호하는 "hot 페이지"가 쓰여 체크포인트를 완료할 수 있다. 즉, 수많은 변경 사항이 단일 쓰기 작업으로 디스크에 쓰여 진다. 이것이 **쓰기 결합**(**write combining**)이며, 매우 효율적이다. 서버는 쓰기 결합으로 인해 초당 수만 건의 쓰기를 허용할 수 있으며, ACID 속성을 보장할 수 있다. 만약 쓰기를 결합하지 않으면, 초당 더 많은 I/O 작업을 수행해야 한다.

## Dirty pages and the long tail

단점은 메모리에 있는 dirty 페이지의 양으로, 이들은 종료하는 동안 디스크에 쓰여야 한다. 종료는 강제 체크포인트와 같다. 서버는 쓰기 작업을 결합할 수 있다는 것을 알고 있기 때문에 많은 작업을 지연하고 있다. 이 때 갑자기 종료 작업으로 인해 모든 청구서가 한 번에 날아온다. 즉, 갑자기 한번에 많은 양의 데이터를 디스크에 써야하는 상황이 온 것이다. 여기서 문제는 서버 메모리의 대부분이 dirty 데이터일 수 있다는 것이다. 디폴트로, 버퍼 상태가 걱정스럽기 시작하여 페이지를 열심히 플러시 해야하는 상황이 오기 전까지, InnoDB는 버퍼 풀의 최대 90%까지 dirty한 데이터를 가질 수 있다.

대부분의 쓰기가 가장 hot한 페이지에 집중된다면, 왜 그렇게 많은 dirty 페이지가 있어야만 할까? 그에 대한 대답은 긴 꼬리(long tail)이다. 키 큰 머리(tall head)로 가지 않는 몇몇 쓰기 작업은 매우 흩어져 있는 긴 꼬리 부분으로 간다. 다시 한번 말하지만, 많은 일회성 쓰기가 자신의 쓰기 작업을 위해 전체 페이지(16KB)를 dirty하게 만들고 있으며, 그 16KB 페이지는 다른 쓰기 작업에 의해 수정(dirty)되지는 않을 것이다. 따라서 긴 꼬리는 오직 80 바이트만 수정되어 쓰여진 16KB 페이지로 가득 차 있다. 이는 결국 많은 페이지의 데이터가 된다.

## Fast shutdown on demand

필요한 경우 데이터베이스를 빨리 종료할 수 있게 하려면, 이를 위해 무엇을 할 수 있을까? 이것은 답하기 어려운 질문이다. 여기 당신이 취할 수 있는 몇 가지 전략이 있다.

- InnoDB가 dirty 페이지를 최소한으로 유지하도록 설정할 수 있다. 문제는, 쓰기 결합이 훨씬 적어지기 시작한다는 것이다. 일반 웹 어플리케이션의 데이터베이스를 가져와서 dirty 페이지 비율을 낮추고 디스크 활동을 관찰해보아라. 디스크 활동이 천정부지로 오를 것이다. InnoDB는 격렬하게 페이지를 플러시하기 시작하며, 뒤돌아서 즉시 같은 페이지를 다시 플러시한다. 하지만 InnoDB는 특별히 그런 목적으로 설계되거나 최적화되지 않았다. 그러나, 이 방법은 실제로 계획된 빠른 종료를 위한 유용한 기술이다.
- 페이지 크기를 줄일 수 있다. 페이지를 더 작게 만들면, 이론상 긴 꼬리 페이지를 플러시하는 일이 줄어든다. 이걸 조심해라! InnoDB의 디폴트 페이지 크기가 이미 너무 작고, 디폴트 페이지 크기가 아닌 경우 실사용 사례가 많지 않음을 보여주는 연구가 있다.
- 종료하기 전에 dirty 페이지를 플러시하지 않도록 InnoDB를 설정할 수 있다. 이는 본질적으로 체크포인트 없이 종료하는 것과 동일하며, 충돌(crashing)과 동일하다. 복구 과정은 데이터베이스가 사용 가능하게 되기 전에, 시작 시에 실행해야 한다. 이는 crash 복구의 메커니즘으로 인해 clean shutdown보다 훨씬 느릴 수 있다.
- 이 목적을 위해 별도의 스레드를 가진 버전으로 업그레이드 하거나 네이티브 비동기 I/O를 사용하여, InnoDB가 훨씬 더 많이 플러시 할 수 있도록 만들 수 있다. 이것은 실제로는 종료 과정에 도움이 되지 않을 수도 있다. 사실을 말하면, 나(*블로그 작성자*)는 이 방법을 확인해보지 않았다.

# The relationship between Innodb Log checkpointing and dirty Buffer pool pages

> [원문](https://www.percona.com/blog/2012/02/17/the-relationship-between-innodb-log-checkpointing-and-dirty-buffer-pool-pages/)

## Some basic facts

- 우리 대부분은 InnoDB에 쓰기를 하는 것은 메모리의 버퍼 풀 페이지들을 업데이트하고, 페이지 작업을 트랜잭션 로그(REDO)에 기록한다는 것을 알고 있다.
- 뒷편에서는, 업데이트 된(dirty) 버퍼 풀 페이지들이 테이블스페이스로 플러시 된다.
- InnoDB가 dirty 버퍼 풀 페이지들을 가진 상태로 멈춘다면(crash), InnoDB 복구는 데이터베이스의 마지막 일관된(consistent) 상태를 재구성해야 한다.
- 복구는 트랜잭션 로그를 사용하는데, 이는 아직 테이블스페이스로 플러시 되지 않은 로그에서 페이지 작업을 재실행함으로써(따라서, 이름이 `redo log`임) 가능하다.

궁극적으로 이 메커니즘은 느린 드라이브에 대한 최적화였다. 모든 변경 사항을 순차적으로 로그에 기록할 수 있다면, 테이블스페이스 전체에 걸친 변경 내용을 무작위로 쓰는 것보다 트랜잭션이 들어올 때 바로 쓰는 것이 더 빠르다. 왜냐하면 순차 I/O가 랜덤 I/O보다 더 빠르기 때문이다.

그러나, 오늘날의 플래시 스토리지 세계에서 랜덤 I/O가 현저히 저렴할지라도(가격이 아니라 레이턴시 관점에서 볼 때), 테이블스페이스 업데이트를 지연할수록, 잠재적으로 절약하고, 압축하고, 병합할 수 있는 IOPs가 더 많아지기 때문에 이것은 여전히 최적화 방안이다. 그 이유는 다음과 같다:

- 페이지가 플러시 되기 전에 동일한 행이 여러번 기록될 수 있다.
- 페이지가 플러시 되기 전에 같은 페이지 내의 여러 행을 쓸 수 있다.

## Innodb Log Checkpointing

먼저, InnoDB 로그 체크포인트에 대해 우리는 무엇을 발견할 수 있을까?

```bash
mysql> SHOW ENGINE INNODB STATUS\G
---
LOG
---
Log sequence number 9682004056
Log flushed up to   9682004056
Last checkpoint at  9682002296
```

위 내용은 로그의 가상 헤드(`Log sequence number`), 로그가 디스크로 플러시 된 마지막 위치(`Log flushed up to`) 및 마지막 체크포인트(`Last checkpoint at`)를 보여준다. LSN은 영원히 커지지만, 트랜잭션 로그 내의 실제 위치는 순환 방식으로 재사용된다. 이 숫자를 기반으로, 가장 오래된 체크포인트 되지 않은 트랜잭션이 트랜잭션 로그에서 몇 바이트 뒤에 있는지 `Last checkpoint at` 값에서 `Log sequence number`를 빼서 확인할 수 있다.

```bash
---
LOG
---
Log sequence number 9682004056
Log flushed up to   9682004056
Last checkpoint at  9682002296
Max checkpoint age    108005254
Checkpoint age target 104630090
Modified age          1760
Checkpoint age        1760
```

여기에서 가장 흥미로운 부분은 체크포인트 age(`Checkpoint age`)인데, 이 값은 위에서 설명한 뺄셈의 결과이다. `Max checkpoint age`는 InnoDB가 트랜잭션 로그에 쓰는 걸 허용하는 가장 멀리 떨어진 체크포인트(원문: as roughly the furthest back Innodb will allow us to go in the transaction logs)를 나타낸다. *(간단하게 설명하면, 체크포인트 age가 커질 수 있는 최대값을 말함)* Dirty 버퍼를 플러시하기 위해 InnoDB에서 클라이언트의 작업을 막지 않고서는, `Checkpoint age`는 이 값을 초과할 수 없다. `Max checkpoint age`는 모든 트랜잭션 로그의 총 바이트 수의 약 80%인 것으로 나타나지만, 항상 이렇다고 확신할 수는 없다.

트랜잭션 로그는 원형(circular)이고, 체크포인트 age는 가장 오랫동안 플러시되지 않은 트랜잭션이 로그에서 얼마나 멀리 떨어져 있는지를 나타낸다는 것을 기억해라. 잠재적으로 충돌시 데이터를 손실하지 않고는 트랜잭션 로그를 덮어쓸 수 없기 때문에, InnoDB는 그러한 작업을 허용하지 않고, 계속해서 (안전하게) 로그를 쓸 수 있을 때까지 들어오는 쓰기를 차단한다.

## Dirty Buffer Pool Pages

한편, dirty 버퍼와 관련한 이슈도 있는데, 아래 두 숫자는 `SHOW ENGINE INNODB STATUS`의 **BUFFER POOL AND MEMORY** 섹션과 관련이 있다.

```bash
Database pages          65530
...
Modified db pages       3
```

위의 경우 데이터가 수정된 3개의 페이지가 dirty 버퍼 안에 있는데, 이것은 전체 버퍼 풀에 비하면 아주 작은 비율을 차지한다. InnoDB의 페이지는 행, 인덱스 등을 포함하는 반면, 하나의 트랜잭션은 한 개 또는 수백만 개의 행을 수정할 수 있다. 또한 버퍼 풀의 수정된 단일 페이지에는 트랜잭션 로그의 여러 트랜잭션에서 수정된 데이터가 포함될 수 있다.

전에 말했듯이, dirty 페이지는 백그라운드에서 디스크로 플러시 된다. 플러시 되는 순서는 연관된 트랜잭션이나 트랜잭션 로그에서 그들의 수정된 위치와 관련이 거의 없다. 이것의 효과는, dirty 페이지 플러싱을 관리하는 쓰레드가 자신의 일로서, 체크포인트 age를 최적화하기 위해 반드시 플러시를 하는 것이 아니라, I/O를 최적화하고 버퍼 풀의 LRU를 따르려고 플러시 하는 것이다.

버퍼가 비순차적으로 플러시 될 수 있기 때문에, 트랜잭션 로그에 디스크로 완전히 플러시 된 많은 트랜잭션이 존재하는 반면(즉, 해당 트랜잭션과 관련된 모든 페이지가 clean 페이지), 여전히 플러시 되지 않은 더 오래된 트랜잭션들이 존재할 수도 있다. 본질적으로, 이것이 fuzzy 체크포인트라는 것이다.

체크포인트 과정은 실제로 논리적인 작업이다. 때때로 (dirty 페이지의 chunk가 플러시 되기 때문에) 버퍼 풀의 dirty 페이지를 살펴보면서 가장 오래된 LSN을 갖는 페이지를 찾는데, 이것이 체크포인트다. 그 페이지보다 오래된 모든 페이지들은 완전히 플러시 되어야만 한다.

**이것이 중요한 이유는, 체크포인트 age가 dirty 버퍼 플러싱에 영향을 미치지 않는다면, dirty 버퍼가 너무 커져서 클라이언트 작업이 지연(stall)될 수 있기 때문이다. 플러시 할 dirty 페이지를 결정하는 알고리즘은 이 과정에 [잘] 최적화 되어 있지 않고, 가끔은 그 자체로 불충분하다.**

그럼 여기서 어떻게 최적화할 수 있을까? 목적은 InnoDB가 더 많은 dirty 페이지를 플러시 하도록 만드는 것이다. 이를 튜닝할 수 있는 많은 방법이 있는데, 이는 아래의 설정값들을 변경시킴으로써 가능하다. 아래는 가장 중요한 설정값들의 목록이고 대충 정렬되어있다. 가장 오래된 순, 동시에 가장 효과가 적은 순으로 정렬되어 있다:

- [innodb_max_dirty_pages_pct](http://dev.mysql.com/doc/refman/5.5/en/innodb-parameters.html#sysvar_innodb_max_dirty_pages_pct): This attempts to keep the percentage of dirty pages under control, and before the Innodb plugin this was really the only way to tune dirty buffer flushing. However, I have seen servers with 3% dirty buffers and they are hitting their max checkpoint age. The way this increases dirty buffer flushing also doesn’t scale well on high io subsystems, it effectively just doubles the dirty buffer flushing per second when the % dirty pages exceeds this amount.
- [innodb_io_capacity](http://dev.mysql.com/doc/refman/5.5/en/innodb-parameters.html#sysvar_innodb_io_capacity): This setting, in spite of all our grand hopes that it would allow Innodb to make better use of our IO in all operations, simply controls the amount of dirty page flushing per second (and other background tasks like read-ahead). Make this bigger, you flush more per second. This does not adapt, it simply does that many iops every second if there are dirty buffers to flush. It will effectively eliminate any optimization of I/O consolidation if you have a low enough write workload (that is, dirty pages get flushed almost immediately, we might be better off without a transaction log in this case). It also can quickly starve data reads and writes to the transaction log if you set this too high.
- [innodb_write_io_threads](http://dev.mysql.com/doc/refman/5.5/en/innodb-parameters.html#sysvar_innodb_write_io_threads): Controls how many threads will have writes in progress to the disk. I’m not sure why this is still useful if you can use Linux native AIO. These can also be rendered useless by filesystems that don’t allow parallel writing to the same file by more than one thread (particularly if you have relatively few tables and/or use the global tablespaces) *cough ext3 cough*.
- [innodb_adaptive_flushing](http://dev.mysql.com/doc/refman/5.5/en/innodb-parameters.html#sysvar_innodb_adaptive_flushing): An Innodb plugin/5.5 setting that tries to be smarter about flushing more aggressively based on the number of dirty pages and the rate of transaction log growth.
- [innodb_adaptive_flushing_method](https://www.percona.com/doc/percona-server/5.5/scalability/innodb_io_55.html#innodb_adaptive_flushing_method): (Percona Server only) This adds a few new different algorithms, but the more effective ones adjust the amount of dirty page flushing using a formula that considers the Checkpoint age and the [Checkpoint age target](https://www.percona.com/doc/percona-server/5.5/scalability/innodb_io_55.html#innodb_checkpoint_age_target) (something you can set in Percona Server, otherwise it is effectively just a hair under the Max Checkpoint age). The two main methods here would be ‘estimate’ (good default) and ‘keep_average’ designed for SSD type storage.
