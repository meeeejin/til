# Code related to log files

This document summarizes the codes used when creating and reusing log files.

## Log File Creation

InnoDB creates all log files when starting the server.

> storage/innobase/srv/srv0start.cc: 2164
```cpp
/** Start InnoDB.
@param[in]  create_new_db       Whether to create a new database
@param[in]  scan_directories    Scan directories for .ibd files for
                                        recovery "dir1;dir2; ... dirN"
@return DB_SUCCESS or error code */
dberr_t srv_start(bool create_new_db, const std::string &scan_directories) {
...
    if (create_new_db) {
    ...
    err = create_log_files(logfilename, dirnamelen, flushed_lsn, logfile0,
                           new_checkpoint_lsn);
    ...
```

Create `srv_n_log_files` log files.

> storage/innobase/srv/srv0start.cc: 366
```cpp
/** Creates all log files.
@param[in,out]  logfilename     buffer for log file name
@param[in]      dirnamelen      length of the directory path
@param[in]      lsn             FIL_PAGE_FILE_FLUSH_LSN value
@param[out]     logfile0          name of the first log file
@param[out]     checkpoint_lsn  lsn of the first created checkpoint
@return DB_SUCCESS or error code */
static dberr_t create_log_files(char *logfilename, size_t dirnamelen, lsn_t lsn,
                                char *&logfile0, lsn_t &checkpoint_lsn) {
...
    for (unsigned i = 0; i < srv_n_log_files; i++) {
    sprintf(logfilename + dirnamelen, "ib_logfile%u", i ? i : INIT_LOG_FILE0);

    err = create_log_file(&files[i], logfilename);

    if (err != DB_SUCCESS) {
      return (err);
    }
  }
...
```

> storage/innobase/srv/srv0start.cc: 277
```cpp
/** Creates a log file.
 @return DB_SUCCESS or error code */
static MY_ATTRIBUTE((warn_unused_result)) dberr_t
    create_log_file(pfs_os_file_t *file, /*!< out: file handle */
                    const char *name)    /*!< in: log file name */
{
    bool ret;

    *file = os_file_create(innodb_log_file_key, name,
                         OS_FILE_CREATE | OS_FILE_ON_ERROR_NO_EXIT,
                         OS_FILE_NORMAL, OS_LOG_FILE, srv_read_only_mode, &ret);
    ...
```

`os_file_create()` eventually is called by `os_file_create_func()`. In `os_file_create_func()`, InnoDB creates a new file with `open()` function.

> storage/innobase/os/os0file.cc: 3201
```cpp
/** NOTE! Use the corresponding macro os_file_create(), not directly
this function!
Opens an existing file or creates a new.
@param[in]  name        name of the file or path as a null-terminated
                                string
@param[in]  create_mode create mode
@param[in]  purpose     OS_FILE_AIO, if asynchronous, non-buffered I/O
                                is desired, OS_FILE_NORMAL, if any normal file;
                                NOTE that it also depends on type, os_aio_..
                                and srv_.. variables whether we really use async
                                I/O or unbuffered I/O: look in the function
                                source code for the exact rules
@param[in]  type        OS_DATA_FILE or OS_LOG_FILE
@param[in]  read_only   true, if read only checks should be enforcedm
@param[in]  success     true if succeeded
@return handle to the file, not defined if error, error number
        can be retrieved with os_file_get_last_error */
pfs_os_file_t os_file_create_func(const char *name, ulint create_mode,
                                  ulint purpose, ulint type, bool read_only,
                                  bool *success) {
...
    do {
    file.m_file = ::open(name, create_flag, os_innodb_umask);

    if (file.m_file == -1) {
        const char *operation;

        operation =
            (create_mode == OS_FILE_CREATE && !read_only) ? "create" : "open";

        *success = false;

        if (on_error_no_exit) {
        retry = os_file_handle_error_no_exit(name, operation, on_error_silent);
        } else {
        retry = os_file_handle_error(name, operation);
        }
    } else {
        *success = true;
        retry = false;
    }

    } while (retry);
``` 

After creating a log file, InnoDB calls `os_file_set_size` function, and **it writes the specified number (`srv_log_file_size`) of zeros to the file specific offset (`0`) aligning page sizes.**

> storage/innobase/srv/srv0start.cc: 299
```cpp
/** Creates a log file.
 @return DB_SUCCESS or error code */
static MY_ATTRIBUTE((warn_unused_result)) dberr_t
    create_log_file(pfs_os_file_t *file, /*!< out: file handle */
                    const char *name)    /*!< in: log file name */
{
    bool ret;

    *file = os_file_create(innodb_log_file_key, name,
                            OS_FILE_CREATE | OS_FILE_ON_ERROR_NO_EXIT,
                            OS_FILE_NORMAL, OS_LOG_FILE, srv_read_only_mode, &ret);
    ...
    ret = os_file_set_size(name, *file, 0, (os_offset_t)srv_log_file_size,
                            srv_read_only_mode, true);
    ...
    ret = os_file_close(*file);
    ut_a(ret);

    return (DB_SUCCESS);
```

> storage/innobase/os/os0file.cc: 5346
```cpp
/**  Write the specified number of zeros to a file from specific offset.
@param[in]  name        name of the file or path as a null-terminated
                                string
@param[in]  file        handle to a file
@param[in]  offset      file offset
@param[in]  size        file size
@param[in]  read_only   Enable read-only checks if true
@param[in]  flush       Flush file content to disk
@return true if success */
bool os_file_set_size(const char *name, pfs_os_file_t file, os_offset_t offset,
                      os_offset_t size, bool read_only, bool flush) {
    /* Write up to FSP_EXTENT_SIZE bytes at a time. */
    ulint buf_size = 0;

    if (size <= UNIV_PAGE_SIZE) {
        buf_size = 1;
    } else {
        buf_size = ut_min(static_cast<ulint>(64),
                        static_cast<ulint>(size / UNIV_PAGE_SIZE));
    }

    ut_ad(buf_size != 0);

    buf_size *= UNIV_PAGE_SIZE;

    /* Align the buffer for possible raw i/o */
    byte *buf2;

    buf2 = static_cast<byte *>(ut_malloc_nokey(buf_size + UNIV_PAGE_SIZE));

    byte *buf = static_cast<byte *>(ut_align(buf2, UNIV_PAGE_SIZE));

    /* Write buffer full of zeros */
    memset(buf, 0, buf_size);
    ...
    err = os_file_write(request, name, file, buf, current_size, n_bytes);
    ...
```

## Log File Reuse

When writing data from the log buffer to the log file, if there is insufficient free space in the log file, `start_next_file()` function is called.

> storage/innobase/log/log0write.cc :1602
```cpp
static void log_files_write_buffer(log_t &log, byte *buffer, size_t buffer_size,
                                   lsn_t start_lsn) {
    ut_ad(log_writer_mutex_own(log));

    using namespace Log_files_write_impl;

    validate_buffer(log, buffer, buffer_size);

    validate_start_lsn(log, start_lsn, buffer_size);

    checkpoint_no_t checkpoint_no = log.next_checkpoint_no.load();

    const auto real_offset = compute_real_offset(log, start_lsn);

    bool write_from_log_buffer;

    auto write_size = compute_how_much_to_write(log, real_offset, buffer_size,
                                                write_from_log_buffer);

    if (write_size == 0) {
        start_next_file(log, start_lsn);
        return;
    ...
```

In this function, the new log file number `n` is calculated through `uint32_t nth_file = static_cast<uint32_t>(real_offset / log.file_size);`, and the `n`th log file header is flushed.

> storage/innobase/log/log0write.cc: 1231
```cpp
static void start_next_file(log_t &log, lsn_t start_lsn) {
    const auto before_update = log.current_file_end_offset;

    auto real_offset = before_update;

    ut_a(log.file_size % OS_FILE_LOG_BLOCK_SIZE == 0);
    ut_a(real_offset / log.file_size <= ULINT_MAX);

    ut_a(real_offset <= log.files_real_capacity);

    if (real_offset == log.files_real_capacity) {
        /* Wrapped log files, start at file 0,
        just after its initial headers. */
        real_offset = LOG_FILE_HDR_SIZE;
    }

    ut_a(real_offset + OS_FILE_LOG_BLOCK_SIZE <= log.files_real_capacity);

    /* Flush header of the new log file. */
    uint32_t nth_file = static_cast<uint32_t>(real_offset / log.file_size);
    log_files_header_flush(log, nth_file, start_lsn);

    /* Update following members of log:
    - current_file_lsn,
    - current_file_real_offset,
    - current_file_end_offset.
    The only reason is to optimize future calculations
    of offsets within the new log file. */
    log_files_update_offsets(log, start_lsn);

    ut_a(log.current_file_real_offset == before_update + LOG_FILE_HDR_SIZE ||
        (before_update == log.files_real_capacity &&
            log.current_file_real_offset == LOG_FILE_HDR_SIZE));

    ut_a(log.current_file_real_offset - LOG_FILE_HDR_SIZE ==
        log.current_file_end_offset - log.file_size);

    log.write_ahead_end_offset = 0;
    }
```

`log_files_header_flush()` calls `log_files_header_fill()`. `buf` equal to the size of the log block (`OS_FILE_LOG_BLOCK_SIZE = 512`) is initialized to 0 through the `memset()` function, and then header information is written to `buf` through the `mach_write_to_x()` function.

> storage/innobase/log/log0chkp.cc: 261
```cpp
void log_files_header_fill(byte *buf, lsn_t start_lsn, const char *creator) {
  memset(buf, 0, OS_FILE_LOG_BLOCK_SIZE);
  
  mach_write_to_4(buf + LOG_HEADER_FORMAT, LOG_HEADER_FORMAT_CURRENT);
  mach_write_to_8(buf + LOG_HEADER_START_LSN, start_lsn);
    
  strncpy(reinterpret_cast<char *>(buf) + LOG_HEADER_CREATOR, creator,
          LOG_HEADER_CREATOR_END - LOG_HEADER_CREATOR);
  
  ut_ad(LOG_HEADER_CREATOR_END - LOG_HEADER_CREATOR >= strlen(creator));
  
  log_block_set_checksum(buf, log_block_calc_checksum_crc32(buf));
} 

void log_files_header_flush(log_t &log, uint32_t nth_file, lsn_t start_lsn) {
  ut_ad(log_writer_mutex_own(log));
  
  MONITOR_INC(MONITOR_LOG_NEXT_FILE);

  ut_a(nth_file < log.n_files);

  byte *buf = log.file_header_bufs[nth_file];
  
  log_files_header_fill(buf, start_lsn, LOG_HEADER_CREATOR_CURRENT);
  
  DBUG_PRINT("ib_log", ("write " LSN_PF " file " ULINTPF " header", start_lsn,
                        ulint(nth_file)));

  const auto dest_offset = nth_file * uint64_t{log.file_size};
       
  const auto page_no =
      static_cast<page_no_t>(dest_offset / univ_page_size.physical());

  auto err = fil_redo_io(
      IORequestLogWrite, page_id_t{log.files_space_id, page_no}, univ_page_size,
      static_cast<ulint>(dest_offset % univ_page_size.physical()),
      OS_FILE_LOG_BLOCK_SIZE, buf); 

  ut_a(err == DB_SUCCESS);
}
```

In Summary, after initializing a log block to use with zero (using `memset()`), the necessary information was written to the log block. There was no function to initialize the entire log file for reuse.
