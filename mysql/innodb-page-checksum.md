# MySQL/InnoDB page checksum

The below code is based on [MySQL 5.7](https://github.com/mysql/mysql-server/tree/5.7) (Specifically, 5.7.24). 

## Checksum calculation before writing pages to the storage

1. Before writing the page to the storage, InnoDB calculates the checksum value of the page and writes it in the page frame. The function below writes a page to the storage:

> storage/innobase/buf/buf0flu.cc: `buf_flush_write_block_low()`
```cpp
/********************************************************************//**
Does an asynchronous write of a buffer page. NOTE: in simulated aio and
also when the doublewrite buffer is used, we must call
buf_dblwr_flush_buffered_writes after we have posted a batch of
writes! */
static
void
buf_flush_write_block_low(
/*======================*/
    buf_page_t* bpage,      /*!< in: buffer block to write */
    buf_flush_t flush_type, /*!< in: type of flush */
    bool        sync)       /*!< in: true if sync IO request */
{
    ...
    switch (buf_page_get_state(bpage)) {
    ...
    case BUF_BLOCK_FILE_PAGE:
        frame = bpage->zip.data;
        if (!frame) {
            frame = ((buf_block_t*) bpage)->frame;
        }

        buf_flush_init_for_writing(
            reinterpret_cast<const buf_block_t*>(bpage),
            reinterpret_cast<const buf_block_t*>(bpage)->frame,
            bpage->zip.data ? &bpage->zip : NULL,
            bpage->newest_modification,
            fsp_is_checksum_disabled(bpage->id.space()));
        break;
    }
    ...
}
```

2. `buf_flush_write_block_low()` calls `buf_flush_init_for_writing()` to initialize the page for writing. Here, a function that calculates the checksum value is called:

> storage/innobase/buf/buf0flu.cc: `buf_flush_init_for_writing()`
```cpp
/** Initialize a page for writing to the tablespace.
@param[in]  block       buffer block; NULL if bypassing the buffer pool
@param[in,out]  page        page frame
@param[in,out]  page_zip_   compressed page, or NULL if uncompressed
@param[in]  newest_lsn  newest modification LSN to the page
@param[in]  skip_checksum   whether to disable the page checksum */
void
buf_flush_init_for_writing(
    const buf_block_t*  block,
    byte*           page,
    void*           page_zip_,
    lsn_t           newest_lsn,
    bool            skip_checksum)
{
    ...
    switch ((srv_checksum_algorithm_t) srv_checksum_algorithm) {
    case SRV_CHECKSUM_ALGORITHM_CRC32:
    case SRV_CHECKSUM_ALGORITHM_STRICT_CRC32:
        checksum = buf_calc_page_crc32(page);
        mach_write_to_4(page + FIL_PAGE_SPACE_OR_CHKSUM,
                checksum);
        break;
    case SRV_CHECKSUM_ALGORITHM_INNODB:
    case SRV_CHECKSUM_ALGORITHM_STRICT_INNODB:
        checksum = (ib_uint32_t) buf_calc_page_new_checksum(
            page);
        mach_write_to_4(page + FIL_PAGE_SPACE_OR_CHKSUM,
                checksum);
        checksum = (ib_uint32_t) buf_calc_page_old_checksum(
            page);
        break;
    case SRV_CHECKSUM_ALGORITHM_NONE:
    case SRV_CHECKSUM_ALGORITHM_STRICT_NONE:
        mach_write_to_4(page + FIL_PAGE_SPACE_OR_CHKSUM,
                checksum);
        break;
        /* no default so the compiler will emit a warning if
        new enum is added and not handled here */
    }
    ...
}
```

3. For example, the `CRC32` checksum of the page is calculated in `buf_calc_page_crc32()`. See [`storage/innobase/ut/ut0crc32.cc`](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/ut/ut0crc32.cc) for more implementation details:

> storage/innobase/buf/buf0checksum.cc: `buf_calc_page_crc32()`
```cpp
/** Calculates the CRC32 checksum of a page. The value is stored to the page
when it is written to a file and also checked for a match when reading from
the file. When reading we allow both normal CRC32 and CRC-legacy-big-endian
variants. Note that we must be careful to calculate the same value on 32-bit
and 64-bit architectures.
@param[in]  page            buffer page (UNIV_PAGE_SIZE bytes)
@param[in]  use_legacy_big_endian   if true then use big endian
byteorder when converting byte strings to integers
@return checksum */
uint32_t
buf_calc_page_crc32(
    const byte* page,
    bool        use_legacy_big_endian /* = false */)
{
    /* Since the field FIL_PAGE_FILE_FLUSH_LSN, and in versions <= 4.1.x
    FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID, are written outside the buffer pool
    to the first pages of data files, we have to skip them in the page
    checksum calculation.
    We must also skip the field FIL_PAGE_SPACE_OR_CHKSUM where the
    checksum is stored, and also the last 8 bytes of page because
    there we store the old formula checksum. */

    ut_crc32_func_t crc32_func = use_legacy_big_endian
        ? ut_crc32_legacy_big_endian
        : ut_crc32;

    const uint32_t  c1 = crc32_func(
        page + FIL_PAGE_OFFSET,
        FIL_PAGE_FILE_FLUSH_LSN - FIL_PAGE_OFFSET);

    const uint32_t  c2 = crc32_func(
        page + FIL_PAGE_DATA,
        UNIV_PAGE_SIZE - FIL_PAGE_DATA - FIL_PAGE_END_LSN_OLD_CHKSUM);

    return(c1 ^ c2);
}
```

> storage/innobase/ut/ut0crc32.cc: `ut_crc32_sw()`
```cpp
/** Calculates CRC32 in software, without using CPU instructions.
@param[in]  buf data over which to calculate CRC32
@param[in]  len data length
@return CRC-32C (polynomial 0x11EDC6F41) */
uint32_t
ut_crc32_sw(
    const byte* buf,
    ulint       len)
{
    uint32_t    crc = 0xFFFFFFFFU;

    ut_a(ut_crc32_slice8_table_initialized);

    /* Calculate byte-by-byte up to an 8-byte aligned address. After
    this consume the input 8-bytes at a time. */
    while (len > 0 && (reinterpret_cast<uintptr_t>(buf) & 7) != 0) {
        ut_crc32_8_sw(&crc, &buf, &len);
    }

    while (len >= 128) {
        /* This call is repeated 16 times. 16 * 8 = 128. */
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
        ut_crc32_64_sw(&crc, &buf, &len);
    }

    while (len >= 8) {
        ut_crc32_64_sw(&crc, &buf, &len);
    }

    while (len > 0) {
        ut_crc32_8_sw(&crc, &buf, &len);
    }

    return(~crc);
}
```

In the case of the `INNODB` checksum of the page, the functions below are called. See [`storage/innobase/include/ut0rnd.ic`](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/include/ut0rnd.ic) for more implementation details:

> storage/innobase/buf/buf0checksum.cc: `buf_calc_page_new_checksum()`
```cpp
/********************************************************************//**
Calculates a page checksum which is stored to the page when it is written
to a file. Note that we must be careful to calculate the same value on
32-bit and 64-bit architectures.
@return checksum */
ulint
buf_calc_page_new_checksum(
/*=======================*/
    const byte* page)   /*!< in: buffer page */
{
    ulint checksum;

    /* Since the field FIL_PAGE_FILE_FLUSH_LSN, and in versions <= 4.1.x
    FIL_PAGE_ARCH_LOG_NO_OR_SPACE_ID, are written outside the buffer pool
    to the first pages of data files, we have to skip them in the page
    checksum calculation.
    We must also skip the field FIL_PAGE_SPACE_OR_CHKSUM where the
    checksum is stored, and also the last 8 bytes of page because
    there we store the old formula checksum. */

    checksum = ut_fold_binary(page + FIL_PAGE_OFFSET,
                  FIL_PAGE_FILE_FLUSH_LSN - FIL_PAGE_OFFSET)
        + ut_fold_binary(page + FIL_PAGE_DATA,
                 UNIV_PAGE_SIZE - FIL_PAGE_DATA
                 - FIL_PAGE_END_LSN_OLD_CHKSUM);
    checksum = checksum & 0xFFFFFFFFUL;

    return(checksum);
}
```

> storage/innobase/include/ut0rnd.ic: `ut_fold_binary()`
```cpp
/*************************************************************//**
Folds a binary string.
@return folded value */
UNIV_INLINE
ulint
ut_fold_binary(
/*===========*/
    const byte* str,    /*!< in: string of bytes */
    ulint       len)    /*!< in: length */
{
    ulint       fold = 0;
    const byte* str_end = str + (len & 0xFFFFFFF8);
                  
    ut_ad(str || !len);
                 
    while (str < str_end) {
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
    }

    switch (len & 0x7) {
    case 7:
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        // Fall through.
    case 6:
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        // Fall through.
    case 5:
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        // Fall through.
    case 4:
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        // Fall through.
    case 3:
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        // Fall through.
    case 2:
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
        // Fall through.
    case 1:
        fold = ut_fold_ulint_pair(fold, (ulint)(*str++));
    }

    return(fold);
}
```

> storage/innobase/include/ut0rnd.ic: `ut_fold_ulint_pair()`
```cpp
#define UT_HASH_RANDOM_MASK 1463735687
#define UT_HASH_RANDOM_MASK2    1653893711

/*************************************************************//**
Folds a pair of ulints.
@return folded value */
UNIV_INLINE
ulint
ut_fold_ulint_pair(
/*===============*/
    ulint   n1, /*!< in: ulint */
    ulint   n2) /*!< in: ulint */
{   
    return(((((n1 ^ n2 ^ UT_HASH_RANDOM_MASK2) << 8) + n1)
        ^ UT_HASH_RANDOM_MASK) + n2);
}
```

4. Finally, the calculated checksum value is written in the page frame:

> storage/innobase/buf/buf0flu.cc: `buf_flush_init_for_writing()`
```cpp
void
buf_flush_init_for_writing(
    ...
    /* With the InnoDB checksum, we overwrite the first 4 bytes of
    the end lsn field to store the old formula checksum. Since it
    depends also on the field FIL_PAGE_SPACE_OR_CHKSUM, it has to
    be calculated after storing the new formula checksum.

    In other cases we write the same value to both fields.
    If CRC32 is used then it is faster to use that checksum
    (calculated above) instead of calculating another one.
    We can afford to store something other than
    buf_calc_page_old_checksum() or BUF_NO_CHECKSUM_MAGIC in
    this field because the file will not be readable by old
    versions of MySQL/InnoDB anyway (older than MySQL 5.6.3) */

    mach_write_to_4(page + UNIV_PAGE_SIZE - FIL_PAGE_END_LSN_OLD_CHKSUM,
            checksum);
}
```

## Checksum validation after reading pages from the storage

1. After reading a page frame from storage, InnoDB makes sure that the read page is not corrupted before the I/O finishes:

> storage/innobase/buf/buf0buf.cc: `buf_page_io_complete()`
```cpp
/********************************************************************//**
Completes an asynchronous read or write request of a file page to or from
the buffer pool.
@return true if successful */
bool
buf_page_io_complete(
/*=================*/
    buf_page_t* bpage,  /*!< in: pointer to the block in question */
    bool        evict)  /*!< in: whether or not to evict the page
                from LRU list. */

{
    ...
    if (io_type == BUF_IO_READ) {
        ...
        /* From version 3.23.38 up we store the page checksum
        to the 4 first bytes of the page end lsn field */
        if (compressed_page
            || buf_page_is_corrupted(
                true, frame, bpage->size,
                fsp_is_checksum_disabled(bpage->id.space()))) {
                    ...
        }
        ...
    }
    ...
}
```

2. Checks if the page is corrupt:

> storage/innobase/buf/buf0buf.cc: `buf_page_is_corrupted()`
```cpp
/** Checks if a page is corrupt.
@param[in]  check_lsn   true if we need to check and complain about
the LSN
@param[in]  read_buf    database page
@param[in]  page_size   page size
@param[in]  skip_checksum   if true, skip checksum
@param[in]  page_no     page number of given read_buf
@param[in]  strict_check    true if strict-check option is enabled
@param[in]  is_log_enabled  true if log option is enabled
@param[in]  log_file    file pointer to log_file
@return TRUE if corrupted */
ibool
buf_page_is_corrupted(
    bool            check_lsn,
    const byte*     read_buf,
    const page_size_t&  page_size,
    bool            skip_checksum
#ifdef UNIV_INNOCHECKSUM
    ,uintmax_t      page_no,
    bool            strict_check,
    bool            is_log_enabled,
    FILE*           log_file
#endif /* UNIV_INNOCHECKSUM */
)
{
    ...
    checksum_field1 = mach_read_from_4(
        read_buf + FIL_PAGE_SPACE_OR_CHKSUM);

    checksum_field2 = mach_read_from_4(
        read_buf + page_size.logical() - FIL_PAGE_END_LSN_OLD_CHKSUM);
    ...
    const srv_checksum_algorithm_t  curr_algo =
        static_cast<srv_checksum_algorithm_t>(srv_checksum_algorithm);

    bool    legacy_checksum_checked = false;

    switch (curr_algo) {
    case SRV_CHECKSUM_ALGORITHM_CRC32:
    case SRV_CHECKSUM_ALGORITHM_STRICT_CRC32:

        if (buf_page_is_checksum_valid_crc32(read_buf,
            checksum_field1, checksum_field2,
#ifdef UNIV_INNOCHECKSUM
            page_no, is_log_enabled, log_file, curr_algo,
#endif /* UNIV_INNOCHECKSUM */
            false)) {
            return(FALSE);
        }
    ...
    case SRV_CHECKSUM_ALGORITHM_INNODB:
    case SRV_CHECKSUM_ALGORITHM_STRICT_INNODB:

        if (buf_page_is_checksum_valid_innodb(read_buf,
            checksum_field1, checksum_field2
#ifdef UNIV_INNOCHECKSUM
            , page_no, is_log_enabled, log_file, curr_algo
#endif /* UNIV_INNOCHECKSUM */
        )) {
            return(FALSE);
        }
    ...
    }
}
```

3. For example, the verification of the `CRC32` checksum value is done as follows:

> storage/innobase/buf/buf0buf.cc: `buf_page_is_checksum_valid_crc32()`
```cpp
/** Checks if the page is in crc32 checksum format.
@param[in]  read_buf        database page
@param[in]  checksum_field1     new checksum field
@param[in]  checksum_field2     old checksum field
@param[in]  page_no         page number of given read_buf
@param[in]  is_log_enabled      true if log option is enabled
@param[in]  log_file        file pointer to log_file
@param[in]  curr_algo       current checksum algorithm
@param[in]  use_legacy_big_endian   use legacy big endian algorithm
@return true if the page is in crc32 checksum format. */
UNIV_INLINE
bool
buf_page_is_checksum_valid_crc32(
    const byte*         read_buf,
    ulint               checksum_field1,
    ulint               checksum_field2,
#ifdef UNIV_INNOCHECKSUM
    uintmax_t           page_no,
    bool                is_log_enabled,
    FILE*               log_file,
    const srv_checksum_algorithm_t  curr_algo,
#endif /* UNIV_INNOCHECKSUM */
    bool                use_legacy_big_endian)
{
    const uint32_t  crc32 = buf_calc_page_crc32(read_buf,
                            use_legacy_big_endian);

#ifdef UNIV_INNOCHECKSUM
    if (is_log_enabled
        && curr_algo == SRV_CHECKSUM_ALGORITHM_STRICT_CRC32) {
        fprintf(log_file, "page::%" PRIuMAX ";"
            " crc32 calculated = %u;"
            " recorded checksum field1 = " ULINTPF
            " recorded checksum field2 = " ULINTPF
            "\n", page_no, crc32,
            checksum_field1, checksum_field2);
    }
#endif /* UNIV_INNOCHECKSUM */

    if (checksum_field1 != checksum_field2) {
        return(false);
    }

    if (checksum_field1 == crc32) {
        return(true);
    }

    return(false);
}
```

The `INNODB` checksum value is verified in the function below:

> storage/innobase/buf/buf0buf.cc: `buf_page_is_checksum_valid_innodb()`
```cpp
/** Checks if the page is in innodb checksum format.
@param[in]  read_buf    database page
@param[in]  checksum_field1 new checksum field
@param[in]  checksum_field2 old checksum field
@param[in]  page_no     page number of given read_buf
@param[in]  is_log_enabled  true if log option is enabled
@param[in]  log_file    file pointer to log_file
@param[in]  curr_algo   current checksum algorithm
@return true if the page is in innodb checksum format. */
UNIV_INLINE
bool
buf_page_is_checksum_valid_innodb(
    const byte*         read_buf,
    ulint               checksum_field1,
    ulint               checksum_field2
#ifdef UNIV_INNOCHECKSUM
    ,uintmax_t          page_no,
    bool                is_log_enabled,
    FILE*               log_file,
    const srv_checksum_algorithm_t  curr_algo
#endif /* UNIV_INNOCHECKSUM */
    )
{
    /* There are 2 valid formulas for
    checksum_field2 (old checksum field) which algo=innodb could have
    written to the page:

    1. Very old versions of InnoDB only stored 8 byte lsn to the
    start and the end of the page.

    2. Newer InnoDB versions store the old formula checksum
    (buf_calc_page_old_checksum()). */

    ulint   old_checksum = buf_calc_page_old_checksum(read_buf);
    ulint   new_checksum = buf_calc_page_new_checksum(read_buf);
    ...
    if (checksum_field2 != mach_read_from_4(read_buf + FIL_PAGE_LSN)
        && checksum_field2 != old_checksum) {
        return(false);
    }

    /* old field is fine, check the new field */

    /* InnoDB versions < 4.0.14 and < 4.1.1 stored the space id
    (always equal to 0), to FIL_PAGE_SPACE_OR_CHKSUM */

    if (checksum_field1 != 0 && checksum_field1 != new_checksum) {
        return(false);
    }

    return(true);
}
```

Check out `buf_page_is_corrupted()` in [`storage/innobase/buf/buf0buf.cc`](https://github.com/mysql/mysql-server/blob/5.7/storage/innobase/buf/buf0buf.cc) for more information.