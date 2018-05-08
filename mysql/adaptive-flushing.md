# InnoDB adaptive flushing

> 수정 중..

- [dirty 페이지 flushing과 write combining ](https://www.xaprb.com/blog/2010/05/25/dirty-pages-fast-shutdown-and-write-combining/)
- [로그 체크포인트와 dirty 페이지 간의 관계](https://www.percona.com/blog/2012/02/17/the-relationship-between-innodb-log-checkpointing-and-dirty-buffer-pool-pages/)

# InnoDB adaptive flushing in MySQL 5.6: checkpoint age and io capacity

> [원문](https://www.percona.com/blog/2013/10/30/innodb-adaptive-flushing-in-mysql-5-6-checkpoint-age-and-io-capacity/)

MySQL 5.6에서 InnoDB는 플러싱 작업을 수행하는 전용 스레드(`page_cleaner`)를 가진다. `Page_cleaner`는 다음 두 가지 요소를 기반으로 버퍼 풀에서 dirty 페이지를 플러시 한다.

- **Access pattern**: 버퍼 풀에 free 페이지가 더 이상 존재하지 않을 때, LRU 플러셔는 LRU list에서 가장 오래 전에 사용된 페이지를 찾아 플러시 한다.
- **Age**: 수정된 지 가장 오래되었고 아직 플러시 되지 않은 페이지들은 flush list에 존재하며, 몇 가지의 휴리스틱에 기반하여 flush list 플러셔에 의해 플러시 된다.

## Flush list flushing and checkpoint age

Flush list에 저장할 수 있는 오래된(aged) 페이지의 양은 InnoDB의 전체 로그 파일 사이즈에 의해 제한된다. 따라서, flush list 플러싱의 주요 목적은 이 리스트의 페이지를 로그 파일에 충분한 여유 공간을 만들 수 있는 속도로 플러시하는 것이다. 반면, 너무 적극적인 플러싱(aggressive flushing)은 쓰기 결합을 줄이고 I/O 서브시스템에 대한 불필요한 로드를 유발하여, 결국 더 큰 redo 로그를 사용함으로써 얻는 성능 이점을 없애버린다. MySQL 5.6에서 플러시 할 페이지의 양은 현재 체크포인트 경과 시간을 기반으로 InnoDB adaptive 루틴에서 계산된다. 이 때 사용되는 수식은 다음과 같다:

```bash
percentage of the IO capacity that should be used for flushing =
        ((srv_max_io_capacity / srv_io_capacity) * (lsn_age_factor * sqrt(lsn_age_factor))) / 7.5;
```

우리는 R을 이용해 위 수식을 모델링하였고 곡선이 더 평평해지고 결과적으로 플러싱이 덜 aggressive 하도록 개선할 수 있음을 확인하였다. [새로운 공식](https://www.percona.com/doc/percona-server/5.6/performance/page_cleaner_tuning.html#adaptive-flushing-tuning)은 Percona Server 5.6에서 디폴트로 켜져 있다.

![newformula](https://www.percona.com/blog/wp-content/uploads/2013/10/Rplot04.png)

## Flush list flushing and io_capacity

InnoDB는 백그라운드 플러싱 속도를 제어할 수 있는 두 개의 변수(`innodb_io_capacity`와 `innodb_io_capacity_max`)를 제공한다. 이 변수들에 대한 자세한 설명은 [여기](https://dev.mysql.com/doc/refman/5.6/en/innodb-parameters.html#sysvar_innodb_io_capacity)서 확인할 수 있다. 그러나 문서에서 다뤄지지 않은 몇 가지 사항이 있다.

- **innodb_io_capacity_max** 는 플러싱 속도를 실제로 제한하는 유일한 변수로, adaptive 플러싱의 경우 가장 중요한 변수이다. 위 수식 및 그림 참고.
- **innodb_io_capacity** 는 서버가 비활동 또는 종료 상태인 경우, insert 버퍼의 병합과 플러싱 중에 I/O 작업을 제한하는 데에 사용된다.

이 변수를 실질적으로 사용하자면, 위 내용은 다음과 같은 의미를 가진다.

- MySQL 서버가 active한 상태(사용자 요청을 처리 중)인 경우, 플러싱 속도를 높이거나 낮추기 위해 `innodb_io_capacity_max`를 조정해야 한다.
- MySQL 서버가 유휴(idle) 상태이거나 종료 중인 경우, flush list에서 페이지를 비우는 것은
  
– if the MySQL server is in an idle state or performing shutdown flushing of the pages from flush_list will be limited by innodb_io_capacity value only.

– if change_buffering is ON and server is in active state it will allow to use either 5% of innodb_io_capacity or vary rate from 5% to 55%  if more than 50% of insert buffer size was already used.
– if change_buffering is ON and server is idle it will use 100% of innodb_io_capacity for merge operations
