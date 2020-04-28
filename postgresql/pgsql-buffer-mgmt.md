# Buffer Management of PostgreSQL

PostgreSQL 12.2 기준으로 작성됨.

## Reference

- [The Internals of PostgreSQL: Chapter 8 Buffer Manager](http://www.interdb.jp/pg/pgsql08.html)

## Page Read

![page-read](http://www.interdb.jp/pg/img/fig-8-02.png)

1. 테이블 또는 index 페이지를 읽을 때, 백엔드 프로세스는 페이지의 `buffer_tag`를 포함하는 읽기 요청을 버퍼 관리자에게 보냅니다.

2. 버퍼 관리자는 요청된 페이지를 저장하는 슬롯의 `buffer_ID`를 리턴합니다. 요청된 페이지가 버퍼 풀에 없는 경우, 버퍼 관리자는 페이지를 디스크에서 버퍼 풀 슬롯 중 하나로 로드한 다음 해당 `buffer_ID`를 리턴합니다.

3. 백엔드 프로세스는 `buffer_ID` 슬롯에 접근하여 원하는 페이지를 읽습니다.

## 버퍼 매니저 동작 방식

백엔드 프로세스가 원하는 페이지에 접근하려고 할 때, `ReadBuffer_common()`를 호출합니다. 이때, `ReadBuffer_common()`의 동작은 크게 3가지로 나뉩니다.

> src/backend/storage/buffer/bufmgr.c: ReadBuffer_common() - 읽기 요청 처리
```cpp
/* 
 * ReadBuffer_common -- common logic for all ReadBuffer variants
 *
 * *hit is set to true if the request was satisfied from shared buffer cache. */

static Buffer                                                                                         
ReadBuffer_common(SMgrRelation smgr, char relpersistence, ForkNumber forkNum,
                  BlockNumber blockNum, ReadBufferMode mode,
                  BufferAccessStrategy strategy, bool *hit)                                                              
{
    ...
    else
    {
        /*
         * lookup the buffer. IO_IN_PROGRESS is set if the requested block is
         * not currently in memory.
         */                                                                                                
        bufHdr = BufferAlloc(smgr, relpersistence, forkNum, blockNum,
                             strategy, &found);
        if (found)
            pgBufferUsage.shared_blks_hit++;
        else if (isExtend)
            pgBufferUsage.shared_blks_written++;
        else if (mode == RBM_NORMAL || mode == RBM_NORMAL_NO_LOG
                 || mode == RBM_ZERO_ON_ERROR)
            pgBufferUsage.shared_blks_read++;
    }
    ...
```

> src/backend/storage/buffer/bufmgr.c: BufferAlloc() - 요청 페이지를 버퍼에서 탐색; 버퍼에 없는 경우 victim 설정해 비우고 원하는 페이지 읽어 옴
```cpp
/*
 * BufferAlloc -- subroutine for ReadBuffer.  Handles lookup of a shared
 *		buffer.  If no buffer exists already, selects a replacement
 *		victim and evicts the old page, but does NOT read in new page.
 *
 * "strategy" can be a buffer replacement strategy object, or NULL for
 * the default strategy.  The selected buffer's usage_count is advanced when
 * using the default strategy, but otherwise possibly not (see PinBuffer).
 *
 * The returned buffer is pinned and is already marked as holding the
 * desired page.  If it already did have the desired page, *foundPtr is
 * set true.  Otherwise, *foundPtr is set false and the buffer is marked
 * as IO_IN_PROGRESS; ReadBuffer will now need to do I/O to fill it.
 *
 * *foundPtr is actually redundant with the buffer's BM_VALID flag, but
 * we keep it for simplicity in ReadBuffer.
 *
 * No locks are held either at entry or exit.
 */
static BufferDesc *
BufferAlloc(SMgrRelation smgr, char relpersistence, ForkNumber forkNum,
			BlockNumber blockNum,
			BufferAccessStrategy strategy,
			bool *foundPtr)
{
    ...
```

### 1. 요청된 페이지가 버퍼 풀에 있는 경우

![first-case](http://www.interdb.jp/pg/img/fig-8-08.png)

가장 간단한 경우입니다. 말 그대로, 원하는 페이지가 이미 버퍼 풀에 저장되어 있는 경우입니다 (따로 명시하지 않았으면 `BufferAlloc()` 내 코드):

1. 원하는 페이지의 `buffer_tag`를 생성하고 (이 예제에서 `buffer_tag`는 `Tag_C`), hash 함수를 사용하여 생성한 `buffer_tag`와 연관된 항목들을 포함하는 *hash bucket 슬롯*을 계산합니다.

```cpp
	/* create a tag so we can lookup the buffer */
	INIT_BUFFERTAG(newTag, smgr->smgr_rnode.node, forkNum, blockNum);

    /* determine its hash code and partition lock ID */
	newHash = BufTableHashCode(&newTag);
```

2. 해당 hash bucket 슬롯을 커버하는 `BufMappingLock` 파티션을 shared 모드로 획득합니다 (이 lock은 5번째 단계에서 해제됨).

```cpp
	newPartitionLock = BufMappingPartitionLock(newHash);

	/* see if the block is in the buffer pool already */
	LWLockAcquire(newPartitionLock, LW_SHARED);
```

3. 태그가 `Tag_C`인 항목을 찾고, 해당 항목에서 `buffer_id`를 얻습니다. 이 예제에서 `buffer_id`는 2입니다.

```cpp
	buf_id = BufTableLookup(&newTag, newHash);
```

4. `buffer_id` 2에 대한 buffer descriptor를 pin 합니다. 즉, descriptor의 `refcount` 및 `usage_count`가 1씩 증가합니다.

```cpp
	if (buf_id >= 0)
	{
		/*
		 * Found it.  Now, pin the buffer so no one can steal it from the
		 * buffer pool, and check to see if the correct data has been loaded
		 * into the buffer.
		 */
		buf = GetBufferDescriptor(buf_id);

		valid = PinBuffer(buf, strategy);
```

> src/backend/storage/buffer/bufmgr.c: PinBuffer() - 요청 페이지를 pinning
```cpp
/*
 * PinBuffer -- make buffer unavailable for replacement.
 *
 * For the default access strategy, the buffer's usage_count is incremented
 * when we first pin it; for other strategies we just make sure the usage_count
 * isn't zero.  (The idea of the latter is that we don't want synchronized
 * heap scans to inflate the count, but we need it to not be zero to discourage
 * other backends from stealing buffers from our ring.  As long as we cycle
 * through the ring faster than the global clock-sweep cycles, buffers in
 * our ring won't be chosen as victims for replacement by other backends.)
 *
 * This should be applied only to shared buffers, never local ones.
 *
 * Since buffers are pinned/unpinned very frequently, pin buffers without
 * taking the buffer header lock; instead update the state variable in loop of
 * CAS operations. Hopefully it's just a single CAS.
 *
 * Note that ResourceOwnerEnlargeBuffers must have been done already.
 *
 * Returns true if buffer is BM_VALID, else false.  This provision allows
 * some callers to avoid an extra spinlock cycle.
 */
static bool
PinBuffer(BufferDesc *buf, BufferAccessStrategy strategy)
{
	Buffer		b = BufferDescriptorGetBuffer(buf);
	bool		result;
	PrivateRefCountEntry *ref;

	ref = GetPrivateRefCountEntry(b, true);

	if (ref == NULL)
	{
		...
		for (;;)
		{
			...
			if (strategy == NULL)
			{
				/* Default case: increase usagecount unless already max. */
				if (BUF_STATE_GET_USAGECOUNT(buf_state) < BM_MAX_USAGE_COUNT)
					buf_state += BUF_USAGECOUNT_ONE;
			}
			...
		}
	}
	...

	ref->refcount++;
	Assert(ref->refcount > 0);
	ResourceOwnerRememberBuffer(CurrentResourceOwner, b);
	return result;
}
```

5. `BufMappingLock`을 해제합니다.

```cpp
		/* Can release the mapping lock as soon as we've pinned it */
		LWLockRelease(newPartitionLock);
        ...
		return buf;
```

6. `buffer_id` 2를 사용하여 버퍼 풀 슬롯에 접근합니다.

> src/backend/storage/buffer/bufmgr.c: ReadBuffer_common()
```cpp
    /* if it was already in the buffer pool, we're done */
	if (found)
	{
		if (!isExtend)
		{
            ...
			return BufferDescriptorGetBuffer(bufHdr);
		}
    ...
```

### 2. 요청된 페이지를 디스크에서 읽어서 빈 슬롯에 저장하는 경우

![second-case](http://www.interdb.jp/pg/img/fig-8-09.png)

두 번째는 원하는 페이지가 버퍼 풀에 없고, `freelist`가 비어 있지 않은 경우입니다 (따로 명시하지 않았으면 `BufferAlloc()` 내 코드):

1. 버퍼 테이블을 검색합니다:
    1. 원하는 페이지의 `buffer_tag`를 생성하고 (이 예제에서 `buffer_tag`는 `Tag_E`), hash bucket 슬롯을 계산합니다.
    2. `BufMappingLock` 파티션을 shared 모드로 획득합니다.
    3. 버퍼 테이블을 검색합니다. 하지만 원하는 페이지를 찾지 못했습니다.
    4. `BufMappingLock`을 해제합니다.

```cpp
	/* create a tag so we can lookup the buffer */
	INIT_BUFFERTAG(newTag, smgr->smgr_rnode.node, forkNum, blockNum);

	/* determine its hash code and partition lock ID */
	newHash = BufTableHashCode(&newTag);
	newPartitionLock = BufMappingPartitionLock(newHash);

	/* see if the block is in the buffer pool already */
	LWLockAcquire(newPartitionLock, LW_SHARED);
	buf_id = BufTableLookup(&newTag, newHash);
    ...
    /*
	 * Didn't find it in the buffer pool.  We'll have to initialize a new
	 * buffer.  Remember to unlock the mapping lock while doing the work.
	 */
	LWLockRelease(newPartitionLock);
```

2. `freelist`에서 *비어 있는 buffer descriptor*를 확보하여 pin 합니다. 이 예제에서 획득한 descriptor의 `buffer_id`는 4입니다.

```cpp
	/* Loop here in case we have to try another victim buffer */
	for (;;)
	{
		...
		/*
		 * Select a victim buffer.  The buffer is returned with its header
		 * spinlock still held!
		 */
		buf = StrategyGetBuffer(strategy, &buf_state);
        ...
        /* Pin the buffer and then release the buffer spinlock */
		PinBuffer_Locked(buf);
```

3. *Exclusive* 모드에서 `BufMappingLock` 파티션을 획득합니다 (이 lock은 6번째 단계에서 해제됨).

```cpp
		if (oldFlags & BM_TAG_VALID)
		{
            ...
                LWLockAcquire(newPartitionLock, LW_EXCLUSIVE);
            ...
        }
		else
		{
			/* if it wasn't valid, we need only the new partition */
			LWLockAcquire(newPartitionLock, LW_EXCLUSIVE);
			/* remember we have no old-partition lock or tag */
			oldPartitionLock = NULL;
			/* this just keeps the compiler quiet about uninit variables */
			oldHash = 0;
		}
```

4. `buffer_tag`가 `Tag_E`이고 `buffer_id`가 4인 새 데이터 항목을 생성합니다. 생성한 항목을 버퍼 테이블에 삽입합니다.

```cpp
		/*
		 * Try to make a hashtable entry for the buffer under its new tag.
		 * This could fail because while we were writing someone else
		 * allocated another buffer for the same block we want to read in.
		 * Note that we have not yet removed the hashtable entry for the old
		 * tag.
		 */
		buf_id = BufTableInsert(&newTag, newHash, buf->buf_id);
```

5. `BufMappingLock`을 해제합니다.

```cpp
	LWLockRelease(newPartitionLock);
```

6. 다음과 같이 `buffer_id` 4를 사용하여 스토리지에서 버퍼 풀 슬롯으로 원하는 페이지 데이터를 읽어옵니다:
    1. `io_in_progress_lock`을 exclusive 모드로 획득합니다.
    2. 다른 프로세스의 접근을 방지하기 위해 해당 descriptor의 `io_in_progress` 비트를 1로 설정합니다.
    3. 스토리지에서 버퍼 풀 슬롯으로 원하는 페이지 데이터를 읽어옵니다.
    4. 해당 descriptor의 상태를 변경합니다. `io_in_progress` 비트는 **0**으로 설정되고, 유효한 비트는 **1**로 설정됩니다.
    5. `io_in_progress_lock`을 해제합니다.

```cpp
	/*
	 * Buffer contents are currently invalid.  Try to get the io_in_progress
	 * lock.  If StartBufferIO returns false, then someone else managed to
	 * read it before we did, so there's nothing left for BufferAlloc() to do.
	 */
	if (StartBufferIO(buf, true))
		*foundPtr = false;
	else
		*foundPtr = true;

	return buf;
}
```

> src/backend/storage/buffer/bufmgr.c: StartBufferIO()
```cpp
/*
 * StartBufferIO: begin I/O on this buffer
 *	(Assumptions)
 *	My process is executing no IO
 *	The buffer is Pinned
 *
 * In some scenarios there are race conditions in which multiple backends
 * could attempt the same I/O operation concurrently.  If someone else
 * has already started I/O on this buffer then we will block on the
 * io_in_progress lock until he's done.
 *
 * Input operations are only attempted on buffers that are not BM_VALID,
 * and output operations only on buffers that are BM_VALID and BM_DIRTY,
 * so we can always tell if the work is already done.
 *
 * Returns true if we successfully marked the buffer as I/O busy,
 * false if someone else already did the work.
 */
static bool
StartBufferIO(BufferDesc *buf, bool forInput)
{
	uint32		buf_state;

	Assert(!InProgressBuf);

	for (;;)
	{
		/*
		 * Grab the io_in_progress lock so that other processes can wait for
		 * me to finish the I/O.
		 */
		LWLockAcquire(BufferDescriptorGetIOLock(buf), LW_EXCLUSIVE);

		buf_state = LockBufHdr(buf);

		if (!(buf_state & BM_IO_IN_PROGRESS))
			break;

		/*
		 * The only way BM_IO_IN_PROGRESS could be set when the io_in_progress
		 * lock isn't held is if the process doing the I/O is recovering from
		 * an error (see AbortBufferIO).  If that's the case, we must wait for
		 * him to get unwedged.
		 */
		UnlockBufHdr(buf, buf_state);
		LWLockRelease(BufferDescriptorGetIOLock(buf));
		WaitIO(buf);
	}

	/* Once we get here, there is definitely no I/O active on this buffer */

	if (forInput ? (buf_state & BM_VALID) : !(buf_state & BM_DIRTY))
	{
		/* someone else already did the I/O */
		UnlockBufHdr(buf, buf_state);
		LWLockRelease(BufferDescriptorGetIOLock(buf));
		return false;
	}

	buf_state |= BM_IO_IN_PROGRESS;
	UnlockBufHdr(buf, buf_state);

	InProgressBuf = buf;
	IsForInput = forInput;

	return true;
}
```

> src/backend/storage/buffer/bufmgr.c: ReadBuffer_common()
```cpp
	if (isExtend)
	{
        ...
    }
    else
    {
        ...
        smgrread(smgr, forkNum, blockNum, (char *) bufBlock);
        ...
    }
```

7. `buffer_id` 4를 사용하여 버퍼 풀 슬롯에 접근합니다.

> src/backend/storage/buffer/bufmgr.c: ReadBuffer_common()
```cpp
    return BufferDescriptorGetBuffer(bufHdr);
```

### 3. 요청된 페이지를 디스크에서 읽어서 victim 슬롯에 저장하는 경우

![third-case](http://www.interdb.jp/pg/img/fig-8-10.png)

세 번째는 모든 버퍼 풀 슬롯이 사용 중이지만 원하는 페이지는 버퍼 풀에 없는 경우입니다 (따로 명시하지 않았으면 `BufferAlloc()` 내 코드):

1. 원하는 페이지의 `buffer_tag`를 생성하고 (이 예제에서 `buffer_tag`는 `Tag_M`), 버퍼 테이블을 검색합니다. 하지만 원하는 페이지를 찾지 못했습니다.

```cpp
	/* create a tag so we can lookup the buffer */
	INIT_BUFFERTAG(newTag, smgr->smgr_rnode.node, forkNum, blockNum);

	/* determine its hash code and partition lock ID */
	newHash = BufTableHashCode(&newTag);
	newPartitionLock = BufMappingPartitionLock(newHash);

	/* see if the block is in the buffer pool already */
	LWLockAcquire(newPartitionLock, LW_SHARED);
	buf_id = BufTableLookup(&newTag, newHash);
    ...
    /*
	 * Didn't find it in the buffer pool.  We'll have to initialize a new
	 * buffer.  Remember to unlock the mapping lock while doing the work.
	 */
	LWLockRelease(newPartitionLock);
```

2. **Clock-sweep** 알고리즘을 사용하여 victim 버퍼 풀 슬롯을 선택하고, 버퍼 테이블에서 victim 슬롯의 `buffer_id`를 포함하는 이전 항목을 가져 와서 buffer descriptor 레이어에 victim 슬롯을 pin 합니다. 이 예제에서 victim 슬롯의 `buffer_id`는 5이고, 이전 항목은 `Tag_F, id=5` 입니다. Clock-sweep은 다음 섹션에서 설명합니다.

```cpp
	/* Loop here in case we have to try another victim buffer */
	for (;;)
	{
		...
		/*
		 * Select a victim buffer.  The buffer is returned with its header
		 * spinlock still held!
		 */
		buf = StrategyGetBuffer(strategy, &buf_state);
        ...
        /* Pin the buffer and then release the buffer spinlock */
		PinBuffer_Locked(buf);
```

> src/backend/storage/buffer/freelist.c: StrategyGetBuffer() - 요약: Clock-sweep 사용해 victim 버퍼 선택
```cpp
buf = StrategyGetBuffer(strategy, &buf_state); // victim 버퍼 선택
    freelist에서 buffer 가져옴
        있으면, return buf
    없으면, for문 돌면서 Clock-sweep 알고리즘 수행
        unpinned && usage count == 0 → usable buffer
        usable buffer가 있으면, return buf
```

3. Victim 페이지 데이터가 dirty면, flush (write and fsync) 합니다. 그렇지 않으면 4단계로 넘어갑니다.
    1. `buffer_id` 5를 사용하여, descriptor의 shared `content_lock` 및 exclusive `io_in_progress` lock을 획득합니다.
    2. 해당 descriptor의 상태를 변경합니다. `io_in_progress` 비트는 **1**로 설정하고, `just_dirtied` 비트는 **0**으로 설정합니다.
    3. 상황에 따라, WAL 버퍼의 WAL 데이터를 현재 WAL 세그먼트 파일에 기록하기 위해 `XLogFlush()` 함수가 호출됩니다.
    4. Victim 페이지 데이터를 스토리지로 flush 합니다.
    5. 해당 descriptor의 상태를 변경합니다. `io_in_progress` 비트는 **0**으로 설정되고, 유효한 비트는 **1**로 설정됩니다.
    6. `io_in_progress` 및 `content_lock` lock을 해제합니다.

```cpp
		/*
		 * If the buffer was dirty, try to write it out.  There is a race
		 * condition here, in that someone might dirty it after we released it
		 * above, or even while we are writing it out (since our share-lock
		 * won't prevent hint-bit updates).  We will recheck the dirty bit
		 * after re-locking the buffer header.
		 */
		if (oldFlags & BM_DIRTY)
		{
			/*
			 * We need a share-lock on the buffer contents to write it out
			 * (else we might write invalid data, eg because someone else is
			 * compacting the page contents while we write).  We must use a
			 * conditional lock acquisition here to avoid deadlock.  Even
			 * though the buffer was not pinned (and therefore surely not
			 * locked) when StrategyGetBuffer returned it, someone else could
			 * have pinned and exclusive-locked it by the time we get here. If
			 * we try to get the lock unconditionally, we'd block waiting for
			 * them; if they later block waiting for us, deadlock ensues.
			 * (This has been observed to happen when two backends are both
			 * trying to split btree index pages, and the second one just
			 * happens to be trying to split the page the first one got from
			 * StrategyGetBuffer.)
			 */
			if (LWLockConditionalAcquire(BufferDescriptorGetContentLock(buf),
										 LW_SHARED))
			{
				/*
				 * If using a nondefault strategy, and writing the buffer
				 * would require a WAL flush, let the strategy decide whether
				 * to go ahead and write/reuse the buffer or to choose another
				 * victim.  We need lock to inspect the page LSN, so this
				 * can't be done inside StrategyGetBuffer.
				 */
				if (strategy != NULL)
				{
					XLogRecPtr	lsn;

					/* Read the LSN while holding buffer header lock */
					buf_state = LockBufHdr(buf);
					lsn = BufferGetLSN(buf);
					UnlockBufHdr(buf, buf_state);

					if (XLogNeedsFlush(lsn) &&
						StrategyRejectBuffer(strategy, buf))
					{
						/* Drop lock/pin and loop around for another buffer */
						LWLockRelease(BufferDescriptorGetContentLock(buf));
						UnpinBuffer(buf, true);
						continue;
					}
				}

				/* OK, do the I/O */
				TRACE_POSTGRESQL_BUFFER_WRITE_DIRTY_START(forkNum, blockNum,
														  smgr->smgr_rnode.node.spcNode,
														  smgr->smgr_rnode.node.dbNode,
														  smgr->smgr_rnode.node.relNode);

				FlushBuffer(buf, NULL);
				LWLockRelease(BufferDescriptorGetContentLock(buf));

				ScheduleBufferTagForWriteback(&BackendWritebackContext,
											  &buf->tag);

				TRACE_POSTGRESQL_BUFFER_WRITE_DIRTY_DONE(forkNum, blockNum,
														 smgr->smgr_rnode.node.spcNode,
														 smgr->smgr_rnode.node.dbNode,
														 smgr->smgr_rnode.node.relNode);
			}
			else
			{
				/*
				 * Someone else has locked the buffer, so give it up and loop
				 * back to get another one.
				 */
				UnpinBuffer(buf, true);
				continue;
			}
		}
```

4. 이전 항목을 포함하는 슬롯을 커버하는 old 및 new `BufMappingLock` 파티션을 exclusive 모드로 획득합니다.

```cpp
		/*
		 * To change the association of a valid buffer, we'll need to have
		 * exclusive lock on both the old and new mapping partitions.
		 */
		if (oldFlags & BM_TAG_VALID)
		{
			/*
			 * Need to compute the old tag's hashcode and partition lock ID.
			 * XXX is it worth storing the hashcode in BufferDesc so we need
			 * not recompute it here?  Probably not.
			 */
			oldTag = buf->tag;
			oldHash = BufTableHashCode(&oldTag);
			oldPartitionLock = BufMappingPartitionLock(oldHash);

			/*
			 * Must lock the lower-numbered partition first to avoid
			 * deadlocks.
			 */
			if (oldPartitionLock < newPartitionLock)
			{
				LWLockAcquire(oldPartitionLock, LW_EXCLUSIVE);
				LWLockAcquire(newPartitionLock, LW_EXCLUSIVE);
			}
			else if (oldPartitionLock > newPartitionLock)
			{
				LWLockAcquire(newPartitionLock, LW_EXCLUSIVE);
				LWLockAcquire(oldPartitionLock, LW_EXCLUSIVE);
			}
			else
			{
				/* only one partition, only one lock */
				LWLockAcquire(newPartitionLock, LW_EXCLUSIVE);
			}
		}
```

5. 새 항목을 버퍼 테이블에 삽입합니다.
    1. 새로운 `buffer_tag`인 `Tag_M`과 victim의 `buffer_id`로 구성된 새 항목을 만듭니다.
    2. 새 항목을 포함하는 슬롯을 커버하는 new `BufMappingLock` 파티션을 exclusive 모드로 획득합니다.
    3. 새 항목을 버퍼 테이블에 삽입합니다.

```cpp
		/*
		 * Try to make a hashtable entry for the buffer under its new tag.
		 * This could fail because while we were writing someone else
		 * allocated another buffer for the same block we want to read in.
		 * Note that we have not yet removed the hashtable entry for the old
		 * tag.
		 */
		buf_id = BufTableInsert(&newTag, newHash, buf->buf_id);
```

![third-case-2](http://www.interdb.jp/pg/img/fig-8-11.png)

6. 버퍼 테이블에서 이전 항목을 삭제하고, old `BufMappingLock` 파티션을 해제합니다.

```cpp
	if (oldPartitionLock != NULL)
	{
		BufTableDelete(&oldTag, oldHash);
		if (oldPartitionLock != newPartitionLock)
			LWLockRelease(oldPartitionLock);
	}
```

7. New `BufMappingLock` 파티션을 해제합니다.

```cpp
	LWLockRelease(newPartitionLock);
```

8. 스토리지에서 victim 버퍼 슬롯으로 원하는 페이지 데이터를 읽어옵니다. 그리고 나서, `buffer_id` 5를 갖는 descriptor의 플래그를 업데이트합니다. Dirty 비트는 **0**으로 설정하고, 다른 비트들을 초기화합니다.

```cpp
	/*
	 * Buffer contents are currently invalid.  Try to get the io_in_progress
	 * lock.  If StartBufferIO returns false, then someone else managed to
	 * read it before we did, so there's nothing left for BufferAlloc() to do.
	 */
	if (StartBufferIO(buf, true))
		*foundPtr = false;
	else
		*foundPtr = true;

	return buf;
}
```

> src/backend/storage/buffer/bufmgr.c: StartBufferIO()
```cpp
/*
 * StartBufferIO: begin I/O on this buffer
 *	(Assumptions)
 *	My process is executing no IO
 *	The buffer is Pinned
 *
 * In some scenarios there are race conditions in which multiple backends
 * could attempt the same I/O operation concurrently.  If someone else
 * has already started I/O on this buffer then we will block on the
 * io_in_progress lock until he's done.
 *
 * Input operations are only attempted on buffers that are not BM_VALID,
 * and output operations only on buffers that are BM_VALID and BM_DIRTY,
 * so we can always tell if the work is already done.
 *
 * Returns true if we successfully marked the buffer as I/O busy,
 * false if someone else already did the work.
 */
static bool
StartBufferIO(BufferDesc *buf, bool forInput)
{
	uint32		buf_state;

	Assert(!InProgressBuf);

	for (;;)
	{
		/*
		 * Grab the io_in_progress lock so that other processes can wait for
		 * me to finish the I/O.
		 */
		LWLockAcquire(BufferDescriptorGetIOLock(buf), LW_EXCLUSIVE);

		buf_state = LockBufHdr(buf);

		if (!(buf_state & BM_IO_IN_PROGRESS))
			break;

		/*
		 * The only way BM_IO_IN_PROGRESS could be set when the io_in_progress
		 * lock isn't held is if the process doing the I/O is recovering from
		 * an error (see AbortBufferIO).  If that's the case, we must wait for
		 * him to get unwedged.
		 */
		UnlockBufHdr(buf, buf_state);
		LWLockRelease(BufferDescriptorGetIOLock(buf));
		WaitIO(buf);
	}

	/* Once we get here, there is definitely no I/O active on this buffer */

	if (forInput ? (buf_state & BM_VALID) : !(buf_state & BM_DIRTY))
	{
		/* someone else already did the I/O */
		UnlockBufHdr(buf, buf_state);
		LWLockRelease(BufferDescriptorGetIOLock(buf));
		return false;
	}

	buf_state |= BM_IO_IN_PROGRESS;
	UnlockBufHdr(buf, buf_state);

	InProgressBuf = buf;
	IsForInput = forInput;

	return true;
}
```

> src/backend/storage/buffer/bufmgr.c: ReadBuffer_common()
```cpp
	if (isExtend)
	{
        ...
    }
    else
    {
        ...
        smgrread(smgr, forkNum, blockNum, (char *) bufBlock);
        ...
    }
```

9. `buffer_id` 5를 사용하여 버퍼 풀 슬롯에 접근합니다.

> src/backend/storage/buffer/bufmgr.c: ReadBuffer_common()
```cpp
    return BufferDescriptorGetBuffer(bufHdr);
```

## 페이지 교체 알고리즘: Clock Sweep

이 알고리즘은 오버헤드가 적은 NFU (Not Frequently Used)를 변형한 알고리즘입니다. 즉, 덜 자주 사용하는 페이지를 효율적으로 선택합니다:

> src/backend/storage/buffer/freelist.c: StrategyGetBuffer()
```cpp
/*
 * StrategyGetBuffer
 *
 *	Called by the bufmgr to get the next candidate buffer to use in
 *	BufferAlloc(). The only hard requirement BufferAlloc() has is that
 *	the selected buffer must not currently be pinned by anyone.
 *
 *	strategy is a BufferAccessStrategy object, or NULL for default strategy.
 *
 *	To ensure that no one else can pin the buffer before we do, we must
 *	return the buffer with the buffer header spinlock still held.
 */
BufferDesc *
StrategyGetBuffer(BufferAccessStrategy strategy, uint32 *buf_state)
{
    ...
    /* Nothing on the freelist, so run the "clock sweep" algorithm */
	trycounter = NBuffers;
	for (;;)
	{
		buf = GetBufferDescriptor(ClockSweepTick());

		/*
		 * If the buffer is pinned or has a nonzero usage_count, we cannot use
		 * it; decrement the usage_count (unless pinned) and keep scanning.
		 */
		local_buf_state = LockBufHdr(buf);

		if (BUF_STATE_GET_REFCOUNT(local_buf_state) == 0)
		{
			if (BUF_STATE_GET_USAGECOUNT(local_buf_state) != 0)
			{
				local_buf_state -= BUF_USAGECOUNT_ONE;

				trycounter = NBuffers;
			}
			else
			{
				/* Found a usable buffer */
				if (strategy != NULL)
					AddBufferToRing(strategy, buf);
				*buf_state = local_buf_state;
				return buf;
			}
		}
		else if (--trycounter == 0)
		{
			/*
			 * We've scanned all the buffers without making any state changes,
			 * so all the buffers are pinned (or were when we looked at them).
			 * We could hope that someone will free one eventually, but it's
			 * probably better to fail than to risk getting stuck in an
			 * infinite loop.
			 */
			UnlockBufHdr(buf, local_buf_state);
			elog(ERROR, "no unpinned buffers available");
		}
		UnlockBufHdr(buf, local_buf_state);
	}
}
```

![clock-sweep](http://www.interdb.jp/pg/img/fig-8-12.png)

위 그림에서 buffer descriptor는 blue (pin) 또는 cyan (unpin) 색 상자로 표시되며, 상자의 숫자는 각 descriptor의 `usage_count`를 나타냅니다.

1. `nextVictimBuffer`는 첫 번째 descriptor (`buffer_id` 1)를 가리킵니다. 하지만, 이 descriptor는 pin 되어 있으므로 건너 뜁니다.

2. `nextVictimBuffer`는 두 번째 descriptor (`buffer_id` 2)를 가리킵니다. 이 descriptor는 pin 되어 있진 않지만, `usage_count`는 2입니다. 따라서 `usage_count`를 1만큼 감소시키고, `nextVictimBuffer`는 세 번째 victim으로 넘어갑니다.

3. `nextVictimBuffer`는 세 번째 descriptor (`buffer_id` 3)를 가리킵니다. 이 descriptor는 unpin 상태이며 `usage_count`는 0입니다. 따라서 이 descriptor가 이번 라운드의 victim이 됩니다.

`nextVictimBuffer`가 unpin 상태의 descriptor를 sweep 할 때마다 `usage_count`가 1씩 감소합니다. 따라서 unpin 상태의 descriptor가 버퍼 풀에 존재하면 이 알고리즘은 `nextVictimBuffer`을 회전시키면서 `usage_count`가 0인 victim을 항상 찾을 수 있습니다.

