# Doublewrite buffer (DWB)

According to the [MySQL manual](https://dev.mysql.com/doc/refman/8.0/en/innodb-disk-io.html), InnoDB uses a novel file flush technique involving a structure called the doublewrite buffer (DWB), which is enabled by default in most cases (`innodb_doublewrite=ON`). It adds safety to recovery following a crash or power outage, and improves performance on most varieties of Unix by reducing the need for `fsync()` operations.

Before writing pages to a data file, InnoDB first writes them to a storage area called the DWB. Only after the write and the flush to the DWB has completed does InnoDB write the pages to their proper positions in the data file. If there is an operating system, storage subsystem, or mysqld process crash in the middle of a page write (causing a torn page condition), InnoDB can later find a good copy of the page from the DWB during recovery.

## DWB Struct

> storage/innobase/include/buf0dblwr.h
```cpp
/** Doublewrite control struct */
struct buf_dblwr_t{
    ib_mutex_t  mutex;  /*!< mutex protecting the first_free
                field and write_buf */
    ulint       block1; /*!< the page number of the first
                doublewrite block (64 pages) */
    ulint       block2; /*!< page number of the second block */
    ulint       first_free;/*!< first free position in write_buf
                measured in units of UNIV_PAGE_SIZE */
    ulint       b_reserved;/*!< number of slots currently reserved
                for batch flush. */
    os_event_t  b_event;/*!< event where threads wait for a
                batch flush to end. */
    ulint       s_reserved;/*!< number of slots currently
                reserved for single page flushes. */
    os_event_t  s_event;/*!< event where threads wait for a
                single page flush slot. */
    bool*       in_use; /*!< flag used to indicate if a slot is
                in use. Only used for single page
                flushes. */
    bool        batch_running;/*!< set to TRUE if currently a batch
                is being written from the doublewrite
                buffer. */
    byte*       write_buf;/*!< write buffer used in writing to the
                doublewrite buffer, aligned to an
                address divisible by UNIV_PAGE_SIZE
                (which is required by Windows aio) */
    byte*       write_buf_unaligned;/*!< pointer to write_buf,
                but unaligned */
    buf_page_t**    buf_block_arr;/*!< array to store pointers to
                the buffer blocks which have been
                cached to write_buf */
};
```

## Create a DWB

> storage/innobase/buf/buf0dblwr.cc: `buf_dblwr_create()`
```cpp
/****************************************************************//**
Creates the doublewrite buffer to a new InnoDB installation. The header of the
doublewrite buffer is placed on the trx system header page.
@return true if successful, false if not. */
MY_ATTRIBUTE((warn_unused_result))
bool
buf_dblwr_create(void)
/*==================*/
{
    ...
    if (buf_dblwr) {
        /* Already inited */

        return(true);
    }

start_again:
    mtr_start(&mtr);
    buf_dblwr_being_created = TRUE;

    doublewrite = buf_dblwr_get(&mtr);

    if (mach_read_from_4(doublewrite + TRX_SYS_DOUBLEWRITE_MAGIC)
        == TRX_SYS_DOUBLEWRITE_MAGIC_N) {
        /* The doublewrite buffer has already been created:
        just read in some numbers */

        buf_dblwr_init(doublewrite);

        mtr_commit(&mtr);
        buf_dblwr_being_created = FALSE;
        return(true);
    }

    ib::info() << "Doublewrite buffer not found: creating new";
    
    ulint min_doublewrite_size =
        ( ( 2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE
          + FSP_EXTENT_SIZE / 2
          + 100)
        * UNIV_PAGE_SIZE);
    ...
    block2 = fseg_create(TRX_SYS_SPACE, TRX_SYS_PAGE_NO,
                 TRX_SYS_DOUBLEWRITE
                 + TRX_SYS_DOUBLEWRITE_FSEG, &mtr);
    ...
    fseg_header = doublewrite + TRX_SYS_DOUBLEWRITE_FSEG;
    prev_page_no = 0;

    for (i = 0; i < 2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE
             + FSP_EXTENT_SIZE / 2; i++) {
        // Allocate blocks
        new_block = fseg_alloc_free_page(
            fseg_header, prev_page_no + 1, FSP_UP, &mtr);
        ...
        if (i == FSP_EXTENT_SIZE / 2) {
            ut_a(page_no == FSP_EXTENT_SIZE);
            mlog_write_ulint(doublewrite
                     + TRX_SYS_DOUBLEWRITE_BLOCK1,
                     page_no, MLOG_4BYTES, &mtr);
            mlog_write_ulint(doublewrite
                     + TRX_SYS_DOUBLEWRITE_REPEAT
                     + TRX_SYS_DOUBLEWRITE_BLOCK1,
                     page_no, MLOG_4BYTES, &mtr);

        } else if (i == FSP_EXTENT_SIZE / 2
               + TRX_SYS_DOUBLEWRITE_BLOCK_SIZE) {
            ut_a(page_no == 2 * FSP_EXTENT_SIZE);
            mlog_write_ulint(doublewrite
                     + TRX_SYS_DOUBLEWRITE_BLOCK2,
                     page_no, MLOG_4BYTES, &mtr);
            mlog_write_ulint(doublewrite
                     + TRX_SYS_DOUBLEWRITE_REPEAT
                     + TRX_SYS_DOUBLEWRITE_BLOCK2,
                     page_no, MLOG_4BYTES, &mtr);

        } else if (i > FSP_EXTENT_SIZE / 2) {
            ut_a(page_no == prev_page_no + 1);
        }
        ...
    }
}
```

The following is quoted from [Jeremy's post](https://blog.jcole.us/2013/05/05/innodb-tidbit-the-doublewrite-buffer-wastes-32-pages-512-kib/).

- The DWB is used as a “scratch area” to write (by default) 128 pages contiguously before flushing them out to their final destinations. But actually, it allocates a total of **160** pages:
    - `FSP_EXTENT_SIZE / 2 = 64 / 2 = 32 pages`
    - `2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE = 2 * 64 = 128 pages`
- The initial 32 pages allocated are there purely just to cause the fragment array to be filled up thus forcing the `fseg_alloc_free_page` calls that follow to start allocating complete extents for the remaining 128 pages (that the DWB actually needs). The code then checks which extents were allocated and adds the initial page numbers for those extents to the `TRX_SYS` header as the DWB allocation. In a typical system, InnoDB would allocate the following pages:
    - Fragment pages 13-44: Perpetually unused fragment pages, but left allocated to the file segment for the DWB
    - Extent starting at page 64-ending at page 127: Block 1 of the DWB in practice.
    - Extent starting at page 128-ending at page 191: Block 2 of the DWB in practice

## Initialize the DWB

> storage/innobase/buf/buf0dblwr.cc: `buf_dblwr_init()`
```cpp
/****************************************************************//**
Creates or initialializes the doublewrite buffer at a database start. */
static
void
buf_dblwr_init(
/*===========*/
    byte*   doublewrite)    /*!< in: pointer to the doublewrite buf
                header on trx sys page */
{
    ulint   buf_size;

    buf_dblwr = static_cast<buf_dblwr_t*>(
        ut_zalloc_nokey(sizeof(buf_dblwr_t)));

    /* There are two blocks of same size in the doublewrite
    buffer. */
    buf_size = 2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE;

    /* There must be atleast one buffer for single page writes
    and one buffer for batch writes. */
    ut_a(srv_doublewrite_batch_size > 0
         && srv_doublewrite_batch_size < buf_size);
    ...
```

- Initialize **two blocks** of same size for the DWB
    - One for single page writes and the other for batch writes
- DWB is **2MB** in total (= 2 * 64 * 16K pages)
    - `buf_size = 2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE;`
    ```cpp
    /* include/trx0sys.h */
    /** Size of the doublewrite block in pages */
    #define TRX_SYS_DOUBLEWRITE_BLOCK_SIZE  FSP_EXTENT_SIZE

    /* include/fsp0types.h */
    /** File space extent size in pages
    page size | file space extent size
    ----------+-----------------------
    4 KiB     | 256 pages = 1 MiB
    8 KiB     | 128 pages = 1 MiB
    16 KiB    |  64 pages = 1 MiB
    32 KiB    |  64 pages = 2 MiB
    64 KiB    |  64 pages = 4 MiB
    */
    #define FSP_EXTENT_SIZE         ((UNIV_PAGE_SIZE <= (16384) ?   \
                    (1048576 / UNIV_PAGE_SIZE) :    \
                    ((UNIV_PAGE_SIZE <= (32768)) ?  \
                    (2097152 / UNIV_PAGE_SIZE) :    \
                    (4194304 / UNIV_PAGE_SIZE))))
    ```
- `ulong srv_doublewrite_batch_size = 120`
    - The size of the buffer that is used for *batch flushing* (i.e., LRU flushing and flush_list flushing)
    - The rest of the pages are used for *single page flushing*
    - In summary, **120 slots for batch flushing** and **8 slots for single page flushing** when using 16K pages

> storage/innobase/buf/buf0dblwr.cc: `buf_dblwr_init()`
```cpp
    ...
    mutex_create(LATCH_ID_BUF_DBLWR, &buf_dblwr->mutex);

    buf_dblwr->b_event = os_event_create("dblwr_batch_event");
    buf_dblwr->s_event = os_event_create("dblwr_single_event");
    buf_dblwr->first_free = 0;
    buf_dblwr->s_reserved = 0;
    buf_dblwr->b_reserved = 0;

    buf_dblwr->block1 = mach_read_from_4(
        doublewrite + TRX_SYS_DOUBLEWRITE_BLOCK1);
    buf_dblwr->block2 = mach_read_from_4(
        doublewrite + TRX_SYS_DOUBLEWRITE_BLOCK2);

    buf_dblwr->in_use = static_cast<bool*>(
        ut_zalloc_nokey(buf_size * sizeof(bool)));

    buf_dblwr->write_buf_unaligned = static_cast<byte*>(
        ut_malloc_nokey((1 + buf_size) * UNIV_PAGE_SIZE));

    buf_dblwr->write_buf = static_cast<byte*>(
        ut_align(buf_dblwr->write_buf_unaligned,
             UNIV_PAGE_SIZE));

    buf_dblwr->buf_block_arr = static_cast<buf_page_t**>(
        ut_zalloc_nokey(buf_size * sizeof(void*)));
}
```

The below code is about the DWB header:

> storage/innobase/include/trx0sys.h
```cpp
/** Doublewrite buffer */
/* @{ */
/** The offset of the doublewrite buffer header on the trx system header page */
#define TRX_SYS_DOUBLEWRITE     (UNIV_PAGE_SIZE - 200)
/*-------------------------------------------------------------*/
#define TRX_SYS_DOUBLEWRITE_FSEG    0   /*!< fseg header of the fseg
                        containing the doublewrite
                        buffer */
#define TRX_SYS_DOUBLEWRITE_MAGIC   FSEG_HEADER_SIZE
                        /*!< 4-byte magic number which
                        shows if we already have
                        created the doublewrite
                        buffer */
#define TRX_SYS_DOUBLEWRITE_BLOCK1  (4 + FSEG_HEADER_SIZE)
                        /*!< page number of the
                        first page in the first
                        sequence of 64
                        (= FSP_EXTENT_SIZE) consecutive
                        pages in the doublewrite
                        buffer */
#define TRX_SYS_DOUBLEWRITE_BLOCK2  (8 + FSEG_HEADER_SIZE)
                        /*!< page number of the
                        first page in the second
                        sequence of 64 consecutive
                        pages in the doublewrite
                        buffer */
...
```

## Write a page to the DWB

### Batch flushing

> storage/innobase/buf/buf0dblwr.cc: `buf_dblwr_add_to_batch()`
```cpp
/********************************************************************//**
Posts a buffer page for writing. If the doublewrite memory buffer is
full, calls buf_dblwr_flush_buffered_writes and waits for for free
space to appear. */
void
buf_dblwr_add_to_batch(
/*====================*/
    buf_page_t* bpage)  /*!< in: buffer block to write */
{
    ...
try_again:
    mutex_enter(&buf_dblwr->mutex);
    ...
    if (buf_dblwr->batch_running) {

        /* This not nearly as bad as it looks. There is only
        page_cleaner thread which does background flushing
        in batches therefore it is unlikely to be a contention
        point. The only exception is when a user thread is
        forced to do a flush batch because of a sync
        checkpoint. */
        int64_t sig_count = os_event_reset(buf_dblwr->b_event);
        mutex_exit(&buf_dblwr->mutex);

        os_event_wait_low(buf_dblwr->b_event, sig_count);
        goto try_again;
    }

    if (buf_dblwr->first_free == srv_doublewrite_batch_size) {
        mutex_exit(&(buf_dblwr->mutex));

        buf_dblwr_flush_buffered_writes();

        goto try_again;
    }

    byte*   p = buf_dblwr->write_buf
        + univ_page_size.physical() * buf_dblwr->first_free;

    if (bpage->size.is_compressed()) {
        ...
    } else {
        ut_a(buf_page_get_state(bpage) == BUF_BLOCK_FILE_PAGE);

        UNIV_MEM_ASSERT_RW(((buf_block_t*) bpage)->frame,
                   bpage->size.logical());

        memcpy(p, ((buf_block_t*) bpage)->frame, bpage->size.logical());
    }

    buf_dblwr->buf_block_arr[buf_dblwr->first_free] = bpage;

    buf_dblwr->first_free++;
    buf_dblwr->b_reserved++;

    ...

    if (buf_dblwr->first_free == srv_doublewrite_batch_size) {
        mutex_exit(&(buf_dblwr->mutex));

        buf_dblwr_flush_buffered_writes();

        return;
    }

    mutex_exit(&(buf_dblwr->mutex));
}
```

- Write a page to the DWB buffer in memory first before writing it to disk using `memcpy()`
- If the DWB is full, flush the buffered writes from the DWB in memory to DWB in the disk. Then, write each page to the datafile in the disk:

> storage/innobase/buf/buf0dblwr.cc: `buf_dblwr_flush_buffered_writes()`
```cpp
/********************************************************************//**
Flushes possible buffered writes from the doublewrite memory buffer to disk,
and also wakes up the aio thread if simulated aio is used. It is very
important to call this function after a batch of writes has been posted,
and also when we may have to wait for a page latch! Otherwise a deadlock
of threads can occur. */
void
buf_dblwr_flush_buffered_writes(void)
/*=================================*/
{
    ...
    /* Write out the first block of the doublewrite buffer */
    len = ut_min(TRX_SYS_DOUBLEWRITE_BLOCK_SIZE,
             buf_dblwr->first_free) * UNIV_PAGE_SIZE;

    fil_io(IORequestWrite, true,
           page_id_t(TRX_SYS_SPACE, buf_dblwr->block1), univ_page_size,
           0, len, (void*) write_buf, NULL);

    if (buf_dblwr->first_free <= TRX_SYS_DOUBLEWRITE_BLOCK_SIZE) {
        /* No unwritten pages in the second block. */
        goto flush;
    }

    /* Write out the second block of the doublewrite buffer. */
    len = (buf_dblwr->first_free - TRX_SYS_DOUBLEWRITE_BLOCK_SIZE)
           * UNIV_PAGE_SIZE;

    write_buf = buf_dblwr->write_buf
            + TRX_SYS_DOUBLEWRITE_BLOCK_SIZE * UNIV_PAGE_SIZE;

    fil_io(IORequestWrite, true,
           page_id_t(TRX_SYS_SPACE, buf_dblwr->block2), univ_page_size,
           0, len, (void*) write_buf, NULL);
    ...
flush:
    ...
    /* Now flush the doublewrite buffer data to disk */
    fil_flush(TRX_SYS_SPACE);
    ...
    for (ulint i = 0; i < first_free; i++) {
        buf_dblwr_write_block_to_datafile(
            buf_dblwr->buf_block_arr[i], false);
    }
    ...
}
```

> storage/innobase/buf/buf0dblwr.cc: `buf_dblwr_write_block_to_datafile()`
```cpp
/********************************************************************//**
Writes a page that has already been written to the doublewrite buffer
to the datafile. It is the job of the caller to sync the datafile. */
static
void
buf_dblwr_write_block_to_datafile(
/*==============================*/
    const buf_page_t*   bpage,  /*!< in: page to write */
    bool            sync)   /*!< in: true if sync IO
                    is requested */
{
    ...
    ulint   type = IORequest::WRITE;

    if (sync) {
        type |= IORequest::DO_NOT_WAKE;
    }

    IORequest   request(type);

    if (bpage->zip.data != NULL) {
        ut_ad(bpage->size.is_compressed());

        fil_io(request, sync, bpage->id, bpage->size, 0,
               bpage->size.physical(),
               (void*) bpage->zip.data,
               (void*) bpage);
    } else {
        ...
    }
}
```

### Single page flushing

> storage/innobase/buf/buf0dblwr.cc: `buf_dblwr_write_single_page()`
```cpp
/********************************************************************//**
Writes a page to the doublewrite buffer on disk, sync it, then write
the page to the datafile and sync the datafile. This function is used
for single page flushes. If all the buffers allocated for single page
flushes in the doublewrite buffer are in use we wait here for one to
become free. We are guaranteed that a slot will become free because any
thread that is using a slot must also release the slot before leaving
this function. */
void
buf_dblwr_write_single_page(
/*========================*/
    buf_page_t* bpage,  /*!< in: buffer block to write */
    bool        sync)   /*!< in: true if sync IO requested */
{
    ...
    /* total number of slots available for single page flushes
    starts from srv_doublewrite_batch_size to the end of the
    buffer. */
    size = 2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE;
    ut_a(size > srv_doublewrite_batch_size);
    n_slots = size - srv_doublewrite_batch_size;
    ...
retry:
    mutex_enter(&buf_dblwr->mutex);
    if (buf_dblwr->s_reserved == n_slots) {
        /* All slots are reserved. */
        int64_t sig_count = os_event_reset(buf_dblwr->s_event);
        mutex_exit(&buf_dblwr->mutex);
        os_event_wait_low(buf_dblwr->s_event, sig_count);

        goto retry;
    }

    for (i = srv_doublewrite_batch_size; i < size; ++i) {
        if (!buf_dblwr->in_use[i]) {
            break;
        }
    }

    /* We are guaranteed to find a slot. */
    ut_a(i < size);
    buf_dblwr->in_use[i] = true;
    buf_dblwr->s_reserved++;
    buf_dblwr->buf_block_arr[i] = bpage;
    ...
    mutex_exit(&buf_dblwr->mutex);

    /* Lets see if we are going to write in the first or second
    block of the doublewrite buffer. */
    if (i < TRX_SYS_DOUBLEWRITE_BLOCK_SIZE) {
        offset = buf_dblwr->block1 + i;
    } else {
        offset = buf_dblwr->block2 + i
             - TRX_SYS_DOUBLEWRITE_BLOCK_SIZE;
    }
    ...
    if (bpage->size.is_compressed()) {
        ...
    } else {
        /* It is a regular page. Write it directly to the
        doublewrite buffer */
        fil_io(IORequestWrite, true,
               page_id_t(TRX_SYS_SPACE, offset), univ_page_size, 0,
               univ_page_size.physical(),
               (void*) ((buf_block_t*) bpage)->frame,
               NULL);
    }

    /* Now flush the doublewrite buffer data to disk */
    fil_flush(TRX_SYS_SPACE);

    /* We know that the write has been flushed to disk now
    and during recovery we will find it in the doublewrite buffer
    blocks. Next do the write to the intended position. */
    buf_dblwr_write_block_to_datafile(bpage, sync);
}
```
- Find a empty slot between `srv_doublewrite_batch_size` and `2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE` (i.e., `120 <= i < 128`)
- Then, writes a page to the DWB on disk, sync it, then write the page to the datafile and sync the datafile

## Update the DWB

> storage/innobase/buf/buf0dblwr.cc: `buf_dblwr_update()`
```cpp
/********************************************************************//**
Updates the doublewrite buffer when an IO request is completed. */
void
buf_dblwr_update(
/*=============*/
    const buf_page_t*   bpage,  /*!< in: buffer block descriptor */
    buf_flush_t     flush_type)/*!< in: flush type */
{
    ...
    switch (flush_type) {
    case BUF_FLUSH_LIST:
    case BUF_FLUSH_LRU:
        mutex_enter(&buf_dblwr->mutex);

        ut_ad(buf_dblwr->batch_running);
        ut_ad(buf_dblwr->b_reserved > 0);
        ut_ad(buf_dblwr->b_reserved <= buf_dblwr->first_free);

        buf_dblwr->b_reserved--;

        if (buf_dblwr->b_reserved == 0) {
            mutex_exit(&buf_dblwr->mutex);
            /* This will finish the batch. Sync data files
            to the disk. */
            fil_flush_file_spaces(FIL_TYPE_TABLESPACE);
            mutex_enter(&buf_dblwr->mutex);

            /* We can now reuse the doublewrite memory buffer: */
            buf_dblwr->first_free = 0;
            buf_dblwr->batch_running = false;
            os_event_set(buf_dblwr->b_event);
        }

        mutex_exit(&buf_dblwr->mutex);
        break;
    case BUF_FLUSH_SINGLE_PAGE:
        {
            const ulint size = 2 * TRX_SYS_DOUBLEWRITE_BLOCK_SIZE;
            ulint i;
            mutex_enter(&buf_dblwr->mutex);
            for (i = srv_doublewrite_batch_size; i < size; ++i) {
                if (buf_dblwr->buf_block_arr[i] == bpage) {
                    buf_dblwr->s_reserved--;
                    buf_dblwr->buf_block_arr[i] = NULL;
                    buf_dblwr->in_use[i] = false;
                    break;
                }
            }

            /* The block we are looking for must exist as a
               reserved block. */
            ut_a(i < size);
        }
        os_event_set(buf_dblwr->s_event);
        mutex_exit(&buf_dblwr->mutex);
        break;
    case BUF_FLUSH_N_TYPES:
        ut_error;
    }
}
```

- Updates the DWB when an I/O request is completed
- Batch flushing
    - If an I/O is completed, decrease the value of `buf_dblwr->b_reserved` by 1
    - If `buf_dblwr->b_reserved == 0`, it means the batch flushing is finished. So, sync the datafiles to the disk and prepare to reuse the DWB by initializing the related values
- Single page flushing
    - If an I/O is completed, find the used slot and initialize the related values for reusing it

## >= MySQL 8.0.20

Prior to MySQL 8.0.20, the DWB storage area is located in the InnoDB system tablespace. As of MySQL 8.0.20, DWB storage area is located in doublewrite files.

The following new variables are provided for DWB configuration:

### `innodb_doublewrite_batch_size`

- Defines the number of doublewrite pages to write in a batch
- Default = 0
- Minimin = 0
- Maximum = 256

### `innodb_doublewrite_dir`

- Defines the directory for doublewrite files. If no directory is specified, doublewrite files are created in the `innodb_data_home_dir` directory
- Ideally, the doublewrite directory should be placed on the fastest storage media available

### `innodb_doublewrite_files`

- Defines the number of doublewrite files
- Default = `innodb_buffer_pool_instances * 2`
    - Two doublewrite files are created for each buffer pool instance: A flush list doublewrite file and an LRU list doublewrite file
    - The flush list doublewrite file
        - For pages flushed from the buffer pool flush list
        - The default size = `InnoDB page size * doublewrite page bytes`
    - The LRU list doublewrite file
        - For pages flushed from the buffer pool LRU list + single page flushes
        - The default size = `InnoDB page size * (doublewrite pages + (512 / the number of buffer pool instances))` where 512 is the total number of slots reserved for single page flushes
- Minimum = 2
- Maximum = 256
- Doublewrite file names have the following format: `#ib_page_size_file_number.dblwr`. For example, 

```bash
#ib_16384_0.dblwr
#ib_16384_1.dblwr
```

### `innodb_doublewrite_pages`

- Defines the maximum number of doublewrite pages per thread for a batch write
- Default = `innodb_write_io_threads` (Default = 4)
- Minimum = `innodb_write_io_threads`
- Maximum = 512

## Reference

- [MySQL 5.7 reference manual: InnoDB Disk I/O](https://dev.mysql.com/doc/refman/8.0/en/innodb-disk-io.html)
- [InnoDB Tidbit: The doublewrite buffer wastes 32 pages (512 KiB)](https://blog.jcole.us/2013/05/05/innodb-tidbit-the-doublewrite-buffer-wastes-32-pages-512-kib/)
- [MySQL 8.0 reference manual: InnoDB Startup Options and System Variables](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_doublewrite)