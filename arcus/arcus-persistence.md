# Arcus persistence

## Reference

- [ARCUS User Meetup 2019 Autumn: ARCUS PERSISTENCE](https://www.slideshare.net/JaM2in/arcus-persistence)
- [ARCUS-memcached Github Repo: persistence branch](https://github.com/naver/arcus-memcached/tree/persistence)

## 스냅샷

- ARCUS 캐시 노드의 메모리 상에 적재된 모든 캐시 아이템을 디스크에 기록하여 특정 시점의 백업 복사본 생성
- 하나의 cache lock을 이용해 데이터 무결성 보장
- 별도의 스냅샷 thread 존재
- 스냅샷 과정
    1. 캐시 락 획득
    2. 해시 테이블을 버켓 순서대로, 일정 개수(16개)의 캐시 아이템만 스캔해 그 아이템의 포인터만 복사함
    3. 캐시 락 해제
    4. 캐시 아이템의 포인터를 이용해 그 아이템의 실제 데이터 복사해 디스크에 기록
        1. 디스크에 쓸 캐시 아이템의 참조 카운트 ++;
        2. 데이터 디스크에 기록
        3. 참조 카운트 --;
        4. 해당 캐시 아이템이 해시 테이블에서 이미 제거된 상태라면, 캐시 아이템이 차지하고 있는 메모리 공간 반환 (free)
    5. 위 동작 반복해 모든 캐시 아이템의 백업 생성

### 질문
- [x] 캐시 락 포인터 복사한 후 해당 캐시 아이템 데이터가 바뀌는 경우 어떻게 처리?
    - 참조 카운트를 1 증가시켜 참조 중임을 나타냄
    - 조회될 수는 있지만 변경은 안됨
    - 변경 위해서는, 새로운 캐시 아이템 생성해서 해시 테이블에 추가하고 기존 캐시 아이템은 해시 테이블에서 제거; 기존 캐시 아이템은 메모리에서 그대로 유지
- [x] 스냅샷을 데이터 기록하는 게 아니라 로그 기록? 아니면 데이터 기록 전에 스냅샷 로그를 기록?
    - 일반 캐시 데이터 flush할 때처럼 데이터 기록 전에 스냅샷 로그 기록

## 명령 로깅

- 캐시 데이터 변경 시 변경 전후의 데이터가 아닌 변경 요청 자체를 기록
- Worker thread가 로그 버퍼에 로그를 기록하면, log flush thread가 로그 버퍼의 로그들을 로그 파일에 flush
- LSN 사용: `<로그 파일의 번호, 그 파일 내에서의 오프셋>`

### 두 가지 모드 지원

1. 동기 (synchronous) 모드

- Default 모드
- 변경 연산에 대한 명령 로그가 디스크에 확실히 반영된 후에 (`fsync()`) 그 변경 연산의 수행을 완료
- 데이터 일관성 보장
- 매번 `fsync()`를 호출해야하므로 비동기 모드보다는 낮은 성능
- 하나씩 write/fsync 했을 때의 오버헤드를 줄이기 위해, group commit 제공 

2. 비동기 (asynchronous) 모드

- 변경 연산에 대한 명령 로그를 로그 버퍼에 기록하지만 디스크에 반영하는 것은 보장하지 않은 상태로 그 변경 연산의 수행을 완료
- 캐시 노드가 비정상적으로 종료되는 경우, 일부 데이터 손실 가능
- 성능을 우선시하는 경우에 유용

### 명령 로깅 메타데이터

- `log_global` 구조체를 사용해 로깅에 관한 메타정보 관리

> `cmdlogbuf.c`
```cpp
/* log global structure */
struct log_global {
    log_FILE        log_file;       /* active한 로그 파일 정보 */
    log_BUFFER      log_buffer;     /* 로그 버퍼 정보 */
    log_FLUSHER     log_flusher;    /* 로그 플러셔 정보 */
    LogSN           nxt_write_lsn;  /* 로그 버퍼에 기록할 다음 로그 레코드의 LSN */
    LogSN           nxt_flush_lsn;  /* 로그 파일에 기록할 다음 로그 레코드의 LSN (로그 버퍼에 있는 첫 번째 로그 레코드의 LSN) */
    LogSN           nxt_fsync_lsn;  /* 로그 파일에 fsync 요청해야할 로그 레코드의 LSN (마지막 fsync() 호출 후, 그 시점의 nxt_flush_lsn) */
    pthread_mutex_t log_write_lock;
    pthread_mutex_t log_flush_lock;
    pthread_mutex_t log_fsync_lock;
    pthread_mutex_t flush_lsn_lock;
    pthread_mutex_t fsync_lsn_lock;
    pthread_cond_t  log_flush_cond;
    bool            async_mode;     /* sync or async 모드; Default는 sync */
    volatile bool   initialized;
};
```

### 명령 로그 레코드

- 메타 정보를 가지는 header와 변경 연산에 대한 데이터를 가지는 body로 구성

> `cmdlog.h`
```cpp
typedef struct _loghdr {
    uint8_t     logtype;        /* 로그 레코드 유형 */
    uint8_t     updtype;        /* 업데이트 유형 */
    uint8_t     reserved_8[2];
    uint32_t    body_length;    /* 로그 레코드 body 길이 */
} LogHdr;

typedef struct _logrec {
    LogHdr      header;         /* 로그 레코드 헤더 */
    char        *body;          /* 로그 레코드 데이터 */
} LogRec;
```

### 명령 로그 버퍼

> `cmdlogbuf.c`
```cpp
/* log buffer structure */
typedef struct _log_buffer {
    /* log buffer */
    char       *data;   /* log buffer pointer */
    uint32_t    size;   /* log buffer size */
    uint32_t    head;   /* the head position in log buffer */
    uint32_t    tail;   /* the tail position in log buffer */
    int32_t     last;   /* the last position in log buffer */

    /* flush request queue */
    log_FREQ   *fque;   /* flush request queue pointer */
    uint32_t    fqsz;   /* flush request queue size */
    uint32_t    fbgn;   /* the queue index to begin flush */
    uint32_t    fend;   /* the queue index to end flush */
    int32_t     dw_end; /* the queue index to end dual write */
} log_BUFFER;
```

- 명령 로그 버퍼는 circular queue 구조이며, 이 버퍼에서는 크게 아래의 두 작업이 발생한다:
    - Worker thread에 의한 로그 레코드 write
    - Log flush thread에 의한 로그 레코드 flush

#### Worker thread에 의한 로그 레코드 write

> `cmdlogbuf.c`: `do_log_buff_write()`
```cpp
static LogSN do_log_buff_write(LogRec *logrec, bool dual_write)
{
    log_BUFFER *logbuff = &log_gl.log_buffer;
    ...
    uint32_t total_length = sizeof(LogHdr) + logrec->header.body_length;
    ...

    /* 1. log_write_lock 획득 */
    pthread_mutex_lock(&log_gl.log_write_lock);

    /* 2. log write 할 위치 탐색 */
    /* 초기화 시, head/tail = 0; last = -1; */
    while (1) {
        if (logbuff->head <= logbuff->tail) {
            assert(logbuff->last == -1);
            /* logbuff->head == logbuff->tail: empty state (NO full state) */
            if (total_length < (logbuff->size - logbuff->tail)) {
                /* 로그 레코드 쓸 공간이 남아있을 경우, break; */
                break; /* enough buffer space */
            }
            if (logbuff->head > 0) {
                logbuff->last = logbuff->tail;
                logbuff->tail = 0;
                /* increase log flush end pointer
                 * to make to-be-flushed log data contiguous in memory.
                 */
                if (logbuff->fque[logbuff->fend].nflush > 0) {
                    if ((++logbuff->fend) == logbuff->fqsz) logbuff->fend = 0;
                }
                if (total_length < logbuff->head) {
                    /* 로그 레코드 쓸 공간이 남아있을 경우, break; */
                    break; /* enough buffer space */
                }
            }
        } else { /* logbuff->head > logbuff->tail */
            assert(logbuff->last != -1);
            if (total_length < (logbuff->head - logbuff->tail)) {
                /* 로그 레코드 쓸 공간이 남아있을 경우, break; */
                break; /* enough buffer space */
            }
        }
        /* 로그 레코드 쓸 여유 공간이 없을 경우, 로그 버퍼의 데이터를 직접 flush */
        /* Lack of log buffer space: force flushing data on log buffer */
        pthread_mutex_unlock(&log_gl.log_write_lock);
        pthread_mutex_lock(&log_gl.log_flush_lock);
        (void)do_log_buff_flush(false);
        pthread_mutex_unlock(&log_gl.log_flush_lock);
        pthread_mutex_lock(&log_gl.log_write_lock);
    }

    /* 3. logbuff->tail에 로그 레코드 write */
    /* write log record at the found location of log buffer */
    lrec_write_to_buffer(logrec, &logbuff->data[logbuff->tail]);

    /* 4. 로그 레코드 크기만큼 logbuff->tail 위치 조정 */
    logbuff->tail += total_length;

    /* 5. 로그 레코드 write할 다음 위치 업데이트 */
    /* update nxt_write_lsn */
    current_lsn = log_gl.nxt_write_lsn;
    log_gl.nxt_write_lsn.roffset += total_length;

    /* 6. 로그 flush 요청 업데이트 */
    /* update log flush request */
    if (logbuff->fque[logbuff->fend].nflush > 0 &&
        logbuff->fque[logbuff->fend].dual_write != dual_write) {
        if ((++logbuff->fend) == logbuff->fqsz) logbuff->fend = 0;
    }
    while (total_length > 0) {
        /* check remain length */
        spare_length = CMDLOG_FLUSH_AUTO_SIZE - logbuff->fque[logbuff->fend].nflush;
        if (spare_length >= total_length) spare_length = total_length;

        logbuff->fque[logbuff->fend].nflush += spare_length;
        logbuff->fque[logbuff->fend].dual_write = dual_write;
        if (logbuff->fque[logbuff->fend].nflush == CMDLOG_FLUSH_AUTO_SIZE) {
            if ((++logbuff->fend) == logbuff->fqsz) logbuff->fend = 0;
        }
        total_length -= spare_length;
    }

    /* 7. log_write_lock 해제 */
    pthread_mutex_unlock(&log_gl.log_write_lock);

    /* 8. Flush 해야하는데 flusher가 자고 있는 경우 flusher thread wakeup */
    /* wake up log flush thread if flush requests exist */
    if (logbuff->fbgn != logbuff->fend) {
        if (log_gl.log_flusher.sleep == true) {
            do_log_flusher_wakeup(&log_gl.log_flusher);
        }
    }
    return current_lsn;
```

1. `log_write_lock` 획득
2. log write 할 위치 탐색
    1. 로그 레코드 쓸 공간이 남아있을 경우, while 문 break; 필요시 `logbuff->tail` 위치 조정
    2. 로그 레코드 쓸 여유 공간이 없을 경우, 로그 버퍼의 데이터를 직접 flush
3. `logbuff->tail`에 로그 레코드 write
4. 로그 레코드 크기만큼 `logbuff->tail` 위치 조정
5. 로그 flush 요청 업데이트
6. `log_write_lock` 해제
7. Flush 해야하는데 flusher가 자고 있는 경우 flusher thread wakeup

#### Log flush thread에 의한 로그 레코드 flush

- Log flush thread는 flush할 로그 레코드가 있으면 계속해서 flush를 수행

> `cmdlogbuf.c`: `log_flush_thread_main()`
```cpp
/* Log Flush Thread */
static void *log_flush_thread_main(void *arg)
{
    log_FLUSHER *flusher = &log_gl.log_flusher;
    struct timeval  tv;
    struct timespec to;
    uint32_t nflush;

    flusher->running = RUNNING_STARTED;
    while (1)
    {
        if (flusher->reqstop) {
            logger->log(EXTENSION_LOG_INFO, NULL, "Command log flush thread recognized stop request.\n");
            break;
        }

        /* 1. log_flush_lock 획득 */
        pthread_mutex_lock(&log_gl.log_flush_lock);
        /* 2. 로그 버퍼의 로그 레코드를 로그 파일로 flush */
        nflush = do_log_buff_flush(false);
        /* 3. log_flush_lock 해제 */
        pthread_mutex_unlock(&log_gl.log_flush_lock);

        /* 4. Flush할 데이터가 없으면 10ms간 sleep; 있으면 1-3 과정 반복 */
        if (nflush == 0) {
            /* nothing to flush: do 10 ms sleep */
            gettimeofday(&tv, NULL);
            if ((tv.tv_usec + 10000) < 1000000) {
                tv.tv_usec += 10000;
                to.tv_sec  = tv.tv_sec;
                to.tv_nsec = tv.tv_usec * 1000;
            } else {
                to.tv_sec  = tv.tv_sec + 1;
                to.tv_nsec = 0;
            }
            pthread_mutex_lock(&flusher->lock);
            flusher->sleep = true;
            pthread_cond_timedwait(&flusher->cond, &flusher->lock, &to);
            flusher->sleep = false;
            pthread_mutex_unlock(&flusher->lock);
        }
    }
    flusher->running = RUNNING_STOPPED;
    return NULL;
}
```

1. log_flush_lock 획득 
2. 로그 버퍼의 로그 레코드를 로그 파일로 flush 
3. log_flush_lock 해제
4. Flush할 데이터가 없으면 10ms간 sleep; 있으면 1-3 과정 반복

> `cmdlogbuf.c`: `do_log_buff_flush()`
```cpp
static uint32_t do_log_buff_flush(bool flush_all)
{
    log_BUFFER *logbuff = &log_gl.log_buffer;
    uint32_t    nflush = 0;
    bool        dual_write_flag = false;
    bool        dual_write_complete_flag = false;

    /* 1. log_write_lock 획득 */
    /* computate flush size */
    pthread_mutex_lock(&log_gl.log_write_lock);

    /* 2. Flush 할 로그 데이터 양 계산 */
    if (logbuff->fbgn == logbuff->dw_end) {
        logbuff->dw_end = -1;
        dual_write_complete_flag = true;
    }
    if (logbuff->fbgn != logbuff->fend) {
        nflush = logbuff->fque[logbuff->fbgn].nflush;
        dual_write_flag = logbuff->fque[logbuff->fbgn].dual_write;
        assert(nflush > 0);
    } else {
        if (flush_all && logbuff->fque[logbuff->fend].nflush > 0) {
            nflush = logbuff->fque[logbuff->fend].nflush;
            dual_write_flag = logbuff->fque[logbuff->fend].dual_write;
            if ((++logbuff->fend) == logbuff->fqsz) logbuff->fend = 0;
        }
    }
    if (nflush > 0) {
        if (logbuff->head == logbuff->last) {
            logbuff->last = -1;
            logbuff->head = 0;
        }
    }

    /* 3. log_write_lock 해제 */
    pthread_mutex_unlock(&log_gl.log_write_lock);

    ...

    if (nflush > 0) {
        /* 4. 로그 버퍼의 데이터를 로그 파일에 flush */
        do_log_file_write(&logbuff->data[logbuff->head], nflush, dual_write_flag);

        /* 5. 다음 로그 레코드가 적힐 오프셋 업데이트 */
        /* update nxt_flush_lsn */
        pthread_mutex_lock(&log_gl.flush_lsn_lock);
        log_gl.nxt_flush_lsn.roffset += nflush;
        pthread_mutex_unlock(&log_gl.flush_lsn_lock);

        /* update next flush position */
        /* 6. log_write_lock 획득 */
        pthread_mutex_lock(&log_gl.log_write_lock);
        /* 7. 로그 버퍼의 head 위치 조정 */
        logbuff->head += nflush;
        if (logbuff->head == logbuff->last) {
            logbuff->last = -1;
            logbuff->head = 0;
        }
        /* 8. Flush 요청 reset */
        /* clear the flush request itself */
        logbuff->fque[logbuff->fbgn].nflush = 0;
        logbuff->fque[logbuff->fbgn].dual_write = false;
        if ((++logbuff->fbgn) == logbuff->fqsz) logbuff->fbgn = 0;
        /* 9. log_write_lock 해제 */
        pthread_mutex_unlock(&log_gl.log_write_lock);
    }
    return nflush;
```

1. `log_write_lock` 획득 
2. Flush 할 로그 데이터 양 계산 (`CMDLOG_FLUSH_AUTO_SIZE = 32KB`)
3. `log_write_lock` 해제
4. 로그 버퍼의 데이터를 로그 파일에 flush
5. 다음 로그 레코드가 적힐 오프셋 업데이트
6. `log_write_lock` 획득
7. 로그 버퍼의 head 위치 조정 
8. Flush 요청 reset
9. `log_write_lock` 해제 

> `cmdlogbuf.c`: `do_log_file_write()`
```cpp
static void do_log_file_write(char *log_ptr, uint32_t log_size, bool dual_write)
{
    log_FILE *logfile = &log_gl.log_file;
    assert(logfile->fd != -1);

    /* The log data is appended */
    /* 1. write() 호출해 로그 파일에 write */
    ssize_t nwrite = disk_write(logfile->fd, log_ptr, log_size);
    if (nwrite != log_size) {
        logger->log(EXTENSION_LOG_WARNING, NULL,
                    "log file(%d) write - write(%ld!=%ld) error=(%d:%s)\n",
                    logfile->fd, nwrite, (ssize_t)log_size,
                    errno, strerror(errno));
    }
    /* FIXME::need error handling */
    assert(nwrite == log_size);
    /* 2. 로그 파일 크기 업데이트 */
    logfile->size += log_size;

    if (dual_write && logfile->next_fd != -1) {
        /* next_fd is guaranteed concurrency by log_flush_lock */

        /* The log data is appended */
        nwrite = disk_write(logfile->next_fd, log_ptr, log_size);
        if (nwrite != log_size) {
            logger->log(EXTENSION_LOG_WARNING, NULL,
                        "log file(%d) write - write(%ld!=%ld) error=(%d:%s)\n",
                        logfile->next_fd, nwrite, (ssize_t)log_size,
                        errno, strerror(errno));
        }
        /* FIXME::need error handling */
        assert(nwrite == log_size);
        logfile->next_size += log_size;
    }
}
```

1. write() 호출해 로그 파일에 write
2. 로그 파일 크기 업데이트 

### 동기 (Synchronous) 모드의 Group Commit

- 별도의 group commit용 thread 존재
- 주기적으로 `fsync()` 호출해 여러 요청 처리에서 생성된 로그 레코드를 한꺼번에 로그 디스크에 확실히 반영

> `cmdlogmgr.c`: `do_cmdlog_gcommit_thread_main()`
```cpp
typedef struct _group_commit {
    pthread_mutex_t   lock;       /* group commit mutex */
    pthread_cond_t    cond;       /* group commit conditional variable */
    log_waiter_t *    wait_head;
    log_waiter_t *    wait_tail;
    uint32_t          wait_cnt;   /* group commit wait count */
    bool              sleep;      /* group commit thread sleep */
    volatile uint8_t  running;    /* Is it running, now ? */
    volatile bool     reqstop;    /* request to stop group commit thread */
} group_commit_t;

static void *do_cmdlog_gcommit_thread_main(void *arg)
{
    group_commit_t *gcommit = &logmgr_gl.group_commit;
    log_waiter_t *waiters = NULL;
    LogSN now_fsync_lsn;
    struct timeval  tv;
    struct timespec to;

    gcommit->running = RUNNING_STARTED;
    while (1) {
        if (gcommit->reqstop) {
            logger->log(EXTENSION_LOG_INFO, NULL,
                        "Group commit thread recognized stop request.\n");
            break;
        }

        pthread_mutex_lock(&gcommit->lock);
        if (gcommit->wait_cnt == 0) {
            /* 1 second sleep */
            gettimeofday(&tv, NULL);
            to.tv_sec = tv.tv_sec + 1;
            to.tv_nsec = tv.tv_usec * 1000;
            gcommit->sleep = true;
            pthread_cond_timedwait(&gcommit->cond, &gcommit->lock, &to);
            gcommit->sleep = false;
        } else {
            pthread_mutex_unlock(&gcommit->lock);

            /* synchronize log file after 2ms */
            /* 2ms 후 로그 파일 fsync() */
            usleep(2000);
            cmdlog_file_sync();
            cmdlog_get_fsync_lsn(&now_fsync_lsn);

            pthread_mutex_lock(&gcommit->lock);
            waiters = do_cmdlog_get_commit_waiter(gcommit, &now_fsync_lsn);
        }
        pthread_mutex_unlock(&gcommit->lock);

        if (waiters) {
            do_cmdlog_callback_and_free_waiters(waiters);
            waiters = NULL;
        }
    }
    gcommit->running = RUNNING_STOPPED;
    return NULL;
}
```

- [ ] log waiter에 대한 이해 필요

> `cmdlogbuf.c`: `cmdlog_file_sync()`
```cpp
void cmdlog_file_sync(void)
{
    LogSN now_flush_lsn;
    log_FILE *logfile = &log_gl.log_file;
    int fd;
    int prev_fd = -1;
    int next_fd = -1;
    int ret;

    /* 1. log_fsync_lock 획득 */
    pthread_mutex_lock(&log_gl.log_fsync_lock);

    /* get current fd info */
    /* 2. log_flush_lock 획득 */
    pthread_mutex_lock(&log_gl.log_flush_lock);
    /* 3. 현재 fd에 대한 정보 획득 */
    now_flush_lsn = log_gl.nxt_flush_lsn;
    if (logfile->prev_fd != -1) {
        prev_fd = logfile->prev_fd;
    }
    fd = logfile->fd;
    if (logfile->next_fd != -1 && logfile->next_size > 0) {
        next_fd = logfile->next_fd;
    }
    /* 4. log_flush_lock 해제 */
    pthread_mutex_unlock(&log_gl.log_flush_lock);

    /* 5. prev_fd가 존재한다면, 해당 파일에 fsync() 후 close() */
    if (prev_fd != -1) {
        /* If prev_fd is set, the prev file access only the log sync thread,
         * so log_fsync_lock is not needed to close the prev file.
         */
        ret = disk_fsync(prev_fd);
        (void)disk_close(prev_fd);

        pthread_mutex_lock(&log_gl.log_flush_lock);
        logfile->prev_fd = -1;
        do_log_file_close_waiter_wakeup(&log_gl.log_file);
        pthread_mutex_unlock(&log_gl.log_flush_lock);
    }

    /* 6. 현재 열려있는 로그 파일에 대해 fsync() 수행 */
    if (LOGSN_IS_GT(&now_flush_lsn, &log_gl.nxt_fsync_lsn)) {
        do {
            /* fsync curr fd */
            ret = disk_fsync(fd);
            if (ret < 0) {
                logger->log(EXTENSION_LOG_WARNING, NULL,
                            "log file fsync error (%d:%s)\n",
                            errno, strerror(errno));
                if (log_gl.async_mode == false) {
                    /* [FATAL] untreatable error => abnormal shutdown by assertion */
                    assert(ret == 0);
                }
                break;
            }

            if (next_fd != -1 && logfile->next_fd != -1) {
                ret = disk_fsync(next_fd);
                if (ret < 0) {
                    logger->log(EXTENSION_LOG_WARNING, NULL,
                                "log file fsync error (%d:%s)\n",
                                errno, strerror(errno));
                    break;
                }
            }

            /* update nxt_fsync_lsn */
            /* 7. 다음에 fsync 되어야 하는 LSN 업데이트 */
            pthread_mutex_lock(&log_gl.fsync_lsn_lock);
            log_gl.nxt_fsync_lsn = now_flush_lsn;
            pthread_mutex_unlock(&log_gl.fsync_lsn_lock);
        } while(0);
    }
    /* 8. log_fsync_lock 해제 */
    pthread_mutex_unlock(&log_gl.log_fsync_lock);
}
```

1. `log_fsync_lock` 획득
2. `log_flush_lock` 획득 
3. 현재 fd에 대한 정보 획득
4. `log_flush_lock` 해제
5. `prev_fd가` 존재한다면, 해당 파일에 `fsync()` 후 `close()`
6. 현재 열려있는 로그 파일에 대해 `fsync()` 수행
7. 다음에 `fsync()` 되어야 하는 LSN 업데이트
8. `log_fsync_lock` 해제