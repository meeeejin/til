# Understanding InnoDB Lock Stats

InnoDB는 리소스에 대한 exclusive access를 위해서 *mutex*를 사용하고, shared access를 위해서 *rw-lock*을 사용한다.

## rw-lock

rw-lock은 버퍼 풀 페이지, 테이블스페이스, data-dictionary 등 공통적으로 공유된 자원에 접근하는 것을 제어하기 위해 사용된다. InnoDB는 lock 상태를 모니터링 할 수 있도록 `SHOW ENGINE INNODB STATUS`를 통해 아래와 같은 정보를 제공한다:

```bash
RW-shared spins 38667, rounds 54868, OS waits 16539
RW-excl spins 6353, rounds 126218, OS waits 3936
RW-sx spins 1896, rounds 43888, OS waits 966
Spin rounds per wait: 1.42 RW-shared, 19.87 RW-excl, 23.15 RW-sx
```

### 3 types of rw-locks

1. Shared (`RW-shared`): 리소스에 대한 shared access 제공. 여러 개의 shared lock 허용
2. Exclusive (`RW-excl`): 리소스에 대한 exclusive access 제공. Shared lock은 exclusive lock을 대기
3. Shared-Exclusive (`RW-sx`): inconsistent read로 리소스에 대한 write access 제공 (relaxed exclusive)

### Locking step

1. 필요한 lock을 얻으려고 시도한다:
    - SUCCESS: return immediately `(spins=0, rounds=0, OS waits=0)`
    - FAILURE: enter spin-loop
2. Spin count를 증가시키고 **spin wait**을 수행한다
3. N 라운드 동안 spin wait을 수행한다. 이때, N은 default로 30이며, `innodb_sync_spin_loops` 값으로 조정할 수 있다:
    1. 각 라운드는 CPU가 X 사이클 동안 PAUSE 상태가 되도록 하는 [PAUSE 로직](https://dev.mysql.com/doc/refman/5.7/en/innodb-performance-spin_lock_polling.html)을 수행한다
        - X = {random value from between (0 - `innodb_spin_wait_delay`) * `innodb_spin_wait_pause_multiplier`}
        - `innodb_spin_wait_delay` = 6 (Default, tunable)
        - `innodb_spin_wait_pause_multiplier` = 50
        - C 컴파일러 및 프로세서 종류에 따라 duration 및 PAUSE 지원 여부 다름
    2. busy-wait 하면서 lock을 사용할 수 있는 지 반복해서 확인한다:
        - AVAILABLE: spin-cycle exit
    3. 필요한 lock을 얻기 위해 재시도한다:
        - SUCCESS: return `(spins=1, rounds=M (M <= N), OS waits=0)`
        - FAILURE and round가 남아있는 경우: spin-cycle 재개
    4. N 라운드 동안에 lock을 얻지 못했다면, 더이상 CPU 사이클을 낭비할 필요가 없다. 따라서, 사용 중인 CPU 사이클을 OS에게 반환하고, 필요할 때 깨어날 수 있도록 sleep 한다
    5. InnoDB의 sync-array infrastructure를 사용해 signal을 받을 수 있도록 한다:
        1. Lock을 얻지 못한 thread는 위 array에 slot을 예약해 자신을 등록한다
        2. 대기를 시작하기 전에, lock을 사용할 수 있는지 다시 한번 확인한다 (Slot을 예약하는 작업은 시간이 많이 걸리고, 그러는 동안 lock이 사용 가능한 상태가 될 수 있기 때문)
        3. 여전히 lock을 획득하지 못했다면, sync-array infrastructure가 signal을 줄 때까지 대기한다
        4. 이렇게 대기하는 작업을 **OS wait**이라 하며, 이 루프에 들어가면 OS waits count가 증가한다
    6. 해당 thread가 signal을 받으면, 필요한 lock을 얻기 위해 재시도한다:
        - SUCCESS: return `(spins=1, rounds=N, OS waits=1)`
        - FAILURE: rounds count를 0으로 리셋하고 3-i 단계부터 재시작 (spins count는 유지)

정리하자면 각 count의 의미는 아래와 같다:

- `spins`: thread가 rw-lock을 얻으려고 시도했지만, 실패해서 spin-loop에 빠진 횟수
- `rounds`: PAUSE 로직이 실행된 round의 횟수
- `OS waits`: spin-loop 내에서 lock을 얻지 못해 포기하고 sleep 한 횟수

## Mutex

### Locking step

Mutex에 대한 lock을 얻는 작업은 rw-lock과 동일하다. 위의 복잡한 내용을 간단하게 정리하면 크게 두 단계로 나눌 수 있다:

1. 먼저 mutex에 대한 lock을 시도한다. 만약 다른 thread가 이미 해당 mutex에 대한 lock을 가지고 있다면, **spin wait**을 수행한다. 이는 루프를 돌면서 "Are you free?" 하면서 free 됐는지 반복해서 확인하는 것을 의미한다.
2. Spin wait이 어느 정도 지속되면, 포기하고 mutex가 free 될 때까지 sleep 한다.

```bash
Mutex spin waits 5870888, rounds 19812448, OS waits 375285
```

- `Mutex spin waits`: thread가 mutex를 얻으려고 시도했지만, 실패해서 spin-loop에 빠진 횟수
- `rounds`: mutex 상태를 확인하며 spin wait cycle에서 thread가 루프를 돈 횟수
- `OS waits`: thread가 spin wait을 포기하고 sleep 상태로 빠진 횟수

## Reference

- [Understand InnoDB spin waits, win a Percona Live ticket](https://www.percona.com/blog/2011/09/02/understand-innodb-spin-waits-win-a-percona-live-ticket/)
- [Understanding InnoDB rw-lock stats](https://mysqlonarm.github.io/Understanding-InnoDB-rwlock-stats/)