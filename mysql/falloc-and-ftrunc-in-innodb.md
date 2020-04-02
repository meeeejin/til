# fallocate() and ftruncate() in MySQL/InnoDB

This document summarizes where `fallocate()` and `ftruncate()` are used. The code below is based on MySQL 8.0.15.

## fallocate()

1. [File Creation](#File-Creation)
2. [Punch Hole](#Punch-Hole)
3. [File Extension](#File-Extension)

### File Creation

If the file system supports `fallocate()` and **FusionIO atomic writes** are enabled, InnoDB uses `fallocate()` to create a tablespace. 

> storage/innobase/fil/fil0fil.cc: 5013
```cpp
/** Create a tablespace (an IBD or IBT) file
@param[in]  space_id    Tablespace ID
@param[in]  name        Tablespace name in dbname/tablename format.
                                For general tablespaces, the 'dbname/' part
                                may be missing.
@param[in]  path        Path and filename of the datafile to create.
@param[in]  flags       Tablespace flags
@param[in]  size        Initial size of the tablespace file in pages,
                                must be >= FIL_IBD_FILE_INITIAL_SIZE
@param[in]  type        FIL_TYPE_TABLESPACE or FIL_TYPE_TEMPORARY
@return DB_SUCCESS or error code */
static dberr_t fil_create_tablespace(space_id_t space_id, const char *name,
                                     const char *path, ulint flags,
                                     page_no_t size, fil_type_t type) {
...
#if !defined(NO_FALLOCATE) && defined(UNIV_LINUX)
  if (fil_fusionio_enable_atomic_write(file)) {
    int ret = posix_fallocate(file.m_file, 0, size * page_size.physical());

    if (ret != 0) {
      ib::error(ER_IB_MSG_303, path, ulonglong{size * page_size.physical()},
                ret, REFMAN);
      success = false;
    } else {
      success = true;
    }

    atomic_write = true;
  } else {
    atomic_write = false;

    success = os_file_set_size(path, file, 0, size * page_size.physical(),
                               srv_read_only_mode, true);
  }
#else
  atomic_write = false;

  success = os_file_set_size(path, file, 0, size * page_size.physical(),
                             srv_read_only_mode, true);

#endif /* !NO_FALLOCATE && UNIV_LINUX */
...
```

### Punch Hole

If the file system supports **sparse files** and **punch hole**, InnoDB uses `fallocate()` to create a tablespace file. The code below is carried out *after* the [above code](#File-Creation).

> storage/innobase/fil/fil0fil.cc: 5053
```cpp
...
  bool punch_hole = os_is_sparse_file_supported(path, file);

  if (punch_hole) {
    dberr_t punch_err;

    punch_err = os_file_punch_hole(file.m_file, 0, size * page_size.physical());

    if (punch_err != DB_SUCCESS) {
      punch_hole = false;
    }
  }
...
```

> storage/innobase/os/os0file.cc: 2058
```cpp
/** Free storage space associated with a section of the file.
@param[in]  fh      Open file handle
@param[in]  off     Starting offset (SEEK_SET)
@param[in]  len     Size of the hole
@return DB_SUCCESS or error code */
static dberr_t os_file_punch_hole_posix(os_file_t fh, os_offset_t off,
                                        os_offset_t len) {
#ifdef HAVE_FALLOC_PUNCH_HOLE_AND_KEEP_SIZE
  const int mode = FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE;

  int ret = fallocate(fh, mode, off, len);
...
```

If the compression is enabled and the I/O type is write, punch a hole in the file using `fallocate()`.

> storage/innobase/os/os0file.cc: 1713
```cpp
/** Decompress after a read and punch a hole in the file if it was a write
@param[in]  type        IO context
@param[in]  fh      Open file handle
@param[in,out]  buf     Buffer to transform
@param[in,out]  scratch     Scratch area for read decompression
@param[in]  src_len     Length of the buffer before compression
@param[in]  offset      file offset from the start where to read
@param[in]  len     Used buffer length for write and output
                                buf len for read
@return DB_SUCCESS or error code */
static dberr_t os_file_io_complete(const IORequest &type, os_file_t fh,
                                   byte *buf, byte *scratch, ulint src_len,
                                   os_offset_t offset, ulint len) {
...
    if (!type.is_compression_enabled()) {
        ...
    } else if (type.is_read()) {
        ...
    } else if (type.punch_hole()) {
        ...
        /* Nothing to do. */
        if (len == src_len) {
        return (DB_SUCCESS);
        }
        ...
        offset += len;

        return (os_file_punch_hole(fh, offset, src_len - len));
    }
...
```

### File Extension

If the file system supports `fallocate()`, InnoDB uses `fallocate()` to extend a tablespace. 
> storage/innobase/fil/fil0fil.cc: 6153

```cpp
/** Try to extend a tablespace if it is smaller than the specified size.
@param[in,out]  space       tablespace
@param[in]  size        desired size in pages
@return whether the tablespace is at least as big as requested */
bool Fil_shard::space_extend(fil_space_t *space, page_no_t size) {
...
    os_offset_t len;
    dberr_t err = DB_SUCCESS;

    len = ((file->size + n_node_extend) * phy_page_size) - node_start;

    ut_ad(len > 0);

#if !defined(NO_FALLOCATE) && defined(UNIV_LINUX)
    /* This is required by FusionIO HW/Firmware */

    int ret = posix_fallocate(file->handle.m_file, node_start, len);
...
```

## ftruncate()

1. [Session Temporary Tablespaces](#Session-Temporary-Tablespaces)
2. [Temporary Files](#Temporary-Files)
3. [Rebuilt Table](#Rebuilt-Table)

### Session Temporary Tablespaces

The `ftruncate()` is used to truncate a file to a specified size in bytes. The system call is mainly called indirectly using the below function: `os_file_truncate_posix()`. 

> storage/innobase/os/os0file.cc: 3533
```cpp
/** Truncates a file to a specified size in bytes.
Do nothing if the size to preserve is greater or equal to the current
size of the file.
@param[in]  pathname    file path
@param[in]  file        file to be truncated
@param[in]  size        size to preserve in bytes
@return true if success */
static bool os_file_truncate_posix(const char *pathname, pfs_os_file_t file,
                                   os_offset_t size) {
  int res = ftruncate(file.m_file, size);
  if (res == -1) {
    bool retry;

    retry = os_file_handle_error_no_exit(pathname, "truncate", false);

    if (retry) {
      ib::warn(ER_IB_MSG_783) << "Truncate failed for '" << pathname << "'";
    }
  }

  return (res == 0);
}
```

As walking up the call stack, `os_file_truncate_posix()` eventually is called by `fil_truncate_tablespace()`. `fil_truncate_tablespace()` is called by `truncate()` function for session temporary tablespaces.

> storage/innobase/srv/srv0tmp.cc: 142
```cpp
bool Tablespace::truncate() {
  if (!m_inited) {
    return (false);
  }

  bool success = fil_truncate_tablespace(m_space_id, FIL_IBT_FILE_INITIAL_SIZE);
  if (!success) {
    return (success);
  }
  mtr_t mtr;
  mtr_start(&mtr);
  mtr_set_log_mode(&mtr, MTR_LOG_NO_REDO);
  fsp_header_init(m_space_id, FIL_IBT_FILE_INITIAL_SIZE, &mtr, false);
  mtr_commit(&mtr);
  return (true);
}
```

And it is used to truncate and release the session temporary tablespace back to the pool.

> storage/innobase/srv/srv0tmp.cc: 229
```cpp
void Tablespace_pool::free_ts(Tablespace *ts) {
  space_id_t space_id = ts->space_id();
  fil_space_t *space = fil_space_get(space_id);
  ut_ad(space != nullptr);

  if (space->size != FIL_IBT_FILE_INITIAL_SIZE) {
    ts->truncate();
  }

  acquire();

  Pool::iterator it = std::find(m_active->begin(), m_active->end(), ts);
  if (it != m_active->end()) {
    m_active->erase(it);
  } else {
    ut_ad(0);
  }
  
  m_free->push_back(ts);

  release();
}
```

### Temporary Files

`ftruncate()` is used to truncates temporary files at its current position using the below function.

> storage/innobase/os/os0file.cc: 3551
```cpp
/** Truncates a file at its current position.
@return true if success */
bool os_file_set_eof(FILE *file) /*!< in: file to be truncated */
{
  return (!ftruncate(fileno(file), ftell(file)));
}
```

Using `os_file_set_eof()`, InnoDB truncates a temporary file for *InnoDB monitor output*.

> storage/innobase/handler/ha_innodb.cc: 17576
```cpp
  /* We let the InnoDB Monitor to output at most MAX_STATUS_SIZE
  bytes of text. */

  char *str;
  ssize_t flen;

  mutex_enter(&srv_monitor_file_mutex);
  rewind(srv_monitor_file);

  srv_printf_innodb_monitor(srv_monitor_file, FALSE, &trx_list_start,
                            &trx_list_end);
    
  os_file_set_eof(srv_monitor_file);
    
  if ((flen = ftell(srv_monitor_file)) < 0) {
    flen = 0;
  }

  ssize_t usable_len;
  
  if (flen > MAX_STATUS_SIZE) {
    usable_len = MAX_STATUS_SIZE;
    srv_truncated_status_writes++;
  } else {
    usable_len = flen;
  }
```

> storage/innobase/srv/srv0srv.cc: 1651
```cpp
...
    if (!srv_read_only_mode && srv_innodb_status) {
      mutex_enter(&srv_monitor_file_mutex);
      rewind(srv_monitor_file);
      if (!srv_printf_innodb_monitor(srv_monitor_file,
                                     MUTEX_NOWAIT(mutex_skipped), NULL, NULL)) {
        mutex_skipped++;
      } else {
        mutex_skipped = 0;
      }

      os_file_set_eof(srv_monitor_file);
      mutex_exit(&srv_monitor_file_mutex);
    }
...
```

Also, InnoDB truncates a temporary file for *miscellanous diagnostic output*.

> storage/innobase/row/row0ins.cc: 678
```cpp
/** Set detailed error message associated with foreign key errors for
 the given transaction. */
static void row_ins_set_detailed(
    trx_t *trx,              /*!< in: transaction */
    dict_foreign_t *foreign) /*!< in: foreign key constraint */
{
  ut_ad(!srv_read_only_mode);

  mutex_enter(&srv_misc_tmpfile_mutex);
  rewind(srv_misc_tmpfile);

  if (os_file_set_eof(srv_misc_tmpfile)) {
    ut_print_name(srv_misc_tmpfile, trx, foreign->foreign_table_name);
    dict_print_info_on_foreign_key_in_create_format(srv_misc_tmpfile, trx,
                                                    foreign, FALSE);
    trx_set_detailed_error_from_file(trx, srv_misc_tmpfile);
  } else {
    trx_set_detailed_error(trx, "temp file operation failed");
  }

  mutex_exit(&srv_misc_tmpfile_mutex);
}
```

### Rebuilt Table

`ftruncate()` is used when executing `inplace_alter_table()`. Check the [detailed explanation](https://dev.mysql.com/worklog/task/?id=9559).

> storage/innobase/row/row0log.cc: 2560
```cpp
/** Applies operations to a table was rebuilt.
@param[in]  thr query graph
@param[in,out]  dup for reporting duplicate key errors
@param[in,out]  stage   performance schema accounting object, used by
ALTER TABLE. If not NULL, then stage->inc() will be called for each block
of log that is applied.
@return DB_SUCCESS, or error code on failure */
static MY_ATTRIBUTE((warn_unused_result)) dberr_t
    row_log_table_apply_ops(que_thr_t *thr, row_merge_dup_t *dup,
                            ut_stage_alter_t *stage) {
...
  if (index->online_log->head.blocks == index->online_log->tail.blocks) {
    if (index->online_log->head.blocks) {
#ifdef HAVE_FTRUNCATE
      /* Truncate the file in order to save space. */
      if (index->online_log->fd > 0 &&
          ftruncate(index->online_log->fd, 0) == -1) {
        perror("ftruncate");
      }
#endif /* HAVE_FTRUNCATE */
      index->online_log->head.blocks = index->online_log->tail.blocks = 0;
    }
...
```

> storage/innobase/row/row0log.cc: 3330
```cpp
/** Applies operations to a secondary index that was being created.
@param[in]  trx transaction (for checking if the operation was
interrupted)
@param[in,out]  index   index
@param[in,out]  dup for reporting duplicate key errors
@param[in,out]  stage   performance schema accounting object, used by
ALTER TABLE. If not NULL, then stage->inc() will be called for each block
of log that is applied.
@return DB_SUCCESS, or error code on failure */
static dberr_t row_log_apply_ops(const trx_t *trx, dict_index_t *index,
                                 row_merge_dup_t *dup,
                                 ut_stage_alter_t *stage) {
...
  if (index->online_log->head.blocks == index->online_log->tail.blocks) {
    if (index->online_log->head.blocks) {
#ifdef HAVE_FTRUNCATE
      /* Truncate the file in order to save space. */
      if (index->online_log->fd > 0 &&
          ftruncate(index->online_log->fd, 0) == -1) {
        perror("ftruncate");
      }
#endif /* HAVE_FTRUNCATE */
      index->online_log->head.blocks = index->online_log->tail.blocks = 0;
    }
...
```

The above two functions are eventually called in `inplace_alter_table_impl()` function.

> storage/innobase/handler0alter.cc: 5907, 5919
```cpp
/** Implementation of inplace_alter_table()
@tparam     Table       dd::Table or dd::Partition
@param[in]  altered_table   TABLE object for new version of table.
@param[in,out]  ha_alter_info   Structure describing changes to be done
                                by ALTER TABLE and holding data used
                                during in-place alter.
@param[in]  old_dd_tab  dd::Table object describing old version
                                of the table.
@param[in,out]  new_dd_tab  dd::Table object for the new version of the
                                table. Can be adjusted by this call.
                                Changes to the table definition will be
                                persisted in the data-dictionary at statement
                                commit time.
@retval true Failure
@retval false Success
*/
template <typename Table>
bool ha_innobase::inplace_alter_table_impl(TABLE *altered_table,
                                           Alter_inplace_info *ha_alter_info,
                                           const Table *old_dd_tab,
                                           Table *new_dd_tab) {
...
  error = row_merge_build_indexes(
      m_prebuilt->trx, m_prebuilt->table, ctx->new_table, ctx->online,
      ctx->add_index, ctx->add_key_numbers, ctx->num_to_add_index,
      altered_table, ctx->add_cols, ctx->col_map, ctx->add_autoinc,
      ctx->sequence, ctx->skip_pk_sort, ctx->m_stage, add_v, eval_table);
...
  if (error == DB_SUCCESS && ctx->online && ctx->need_rebuild()) {
    DEBUG_SYNC_C("row_log_table_apply1_before");
    error = row_log_table_apply(ctx->thr, m_prebuilt->table, altered_table,
                                ctx->m_stage);
  }
...
```