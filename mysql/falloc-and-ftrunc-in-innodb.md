# fallocate() and ftruncate() in MySQL/InnoDB

This document summarizes where `fallocate()` and `ftruncate()` are used. The code below is based on MySQL 8.0.15.

## fallocate()

### File Creation

If the file system supports `fallocate()` and FusionIO atomic writes are enabled, InnoDB uses `fallocate()` to create a tablespace. 

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

If the file system supports sparse files and punch hole, InnoDB uses `fallocate()` to create a tablespace file. The code below is carried out after the [above code](#File-Creation).

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

As walking up the call stack, `os_file_truncate_posix()` eventually is called by `space_truncate()`.