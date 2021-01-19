# Monitoring

```bash
mysql> show engine rocksdb status;
Type    Name    Status
STATISTICS      rocksdb rocksdb.block.cache.miss COUNT : 2673222
rocksdb.block.cache.hit COUNT : 85683938
rocksdb.block.cache.add COUNT : 2601798
rocksdb.block.cache.add.failures COUNT : 0
rocksdb.block.cache.index.miss COUNT : 1437
rocksdb.block.cache.index.hit COUNT : 40084389
rocksdb.block.cache.index.add COUNT : 1437
rocksdb.block.cache.index.bytes.insert COUNT : 138105725
rocksdb.block.cache.index.bytes.evict COUNT : 0
...
DBSTATS rocksdb
** DB Stats **
Uptime(secs): 1884.3 total, 83.4 interval
Cumulative writes: 739K writes, 6638K keys, 739K commit groups, 1.0 writes per commit group, ingest: 1.40 GB, 0.76 MB/s
Cumulative WAL: 739K writes, 0 syncs, 739812.00 writes per sync, written: 0.72 GB, 0.39 MB/s
Cumulative stall: 00:00:0.000 H:M:S, 0.0 percent
Interval writes: 40K writes, 363K keys, 40K commit groups, 1.0 writes per commit group, ingest: 78.57 MB, 0.94 MB/s
Interval WAL: 40K writes, 0 syncs, 40561.00 writes per sync, written: 0.04 MB, 0.48 MB/s
Interval stall: 00:00:0.000 H:M:S, 0.0 percent

CF_COMPACTION   __system__
** Compaction Stats [__system__] **
Level    Files   Size     Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) CompMergeCPU(sec) Comp(cnt) Avg(sec) KeyIn KeyDrop
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
  L0      0/0    0.00 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   1.0      0.0      0.1      0.01              0.00         1    0.009       0      0
  L1      1/0    2.52 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   0.4      0.9      0.3      0.01              0.00         1    0.009      37     21
  L6      1/0    2.13 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   0.0      0.0      0.0      0.00              0.00         0    0.000       0      0
 Sum      2/0    4.65 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   3.7      0.5      0.2      0.02              0.00         2    0.009      37     21
 Int      0/0    0.00 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   0.0      0.0      0.0      0.00              0.00         0    0.000       0      0

** Compaction Stats [__system__] **
Priority    Files   Size     Score Read(GB)  Rn(GB) Rnp1(GB) Write(GB) Wnew(GB) Moved(GB) W-Amp Rd(MB/s) Wr(MB/s) Comp(sec) CompMergeCPU(sec) Comp(cnt) Avg(sec) KeyIn KeyDrop
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Low      0/0    0.00 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   0.0      0.9      0.3      0.01              0.00         1    0.009      37     21
User      0/0    0.00 KB   0.0      0.0     0.0      0.0       0.0      0.0       0.0   0.0      0.0      0.1      0.01              0.00         1    0.009       0      0
Uptime(secs): 1884.3 total, 83.4 interval
Flush(GB): cumulative 0.000, interval 0.000
AddFile(GB): cumulative 0.000, interval 0.000
AddFile(Total Files): cumulative 0, interval 0
AddFile(L0 Files): cumulative 0, interval 0
AddFile(Keys): cumulative 0, interval 0
Cumulative compaction: 0.00 GB write, 0.00 MB/s write, 0.00 GB read, 0.00 MB/s read, 0.0 seconds
Interval compaction: 0.00 GB write, 0.00 MB/s write, 0.00 GB read, 0.00 MB/s read, 0.0 seconds
Stalls(count): 0 level0_slowdown, 0 level0_slowdown_with_compaction, 0 level0_numfiles, 0 level0_numfiles_with_compaction, 0 stop for pending_compaction_bytes, 0 slowdown for pending_compaction_bytes, 0 memtable_compaction, 0 memtable_slowdown, interval 0 total count
...
** Level 6 read latency histogram (micros):
Count: 2121051 Average: 143.5077  StdDev: 1489.83
Min: 0  Median: 137.8562  Max: 1264999
Percentiles: P50: 137.86 P75: 201.15 P99: 252.94 P99.9: 4188.94 P99.99: 91173.22
------------------------------------------------------
[       0,       1 ]     1700   0.080%   0.080%
(       1,       2 ]     6894   0.325%   0.405%
(       2,       3 ]    77872   3.671%   4.077% #
(       3,       4 ]   311119  14.668%  18.745% ###
(       4,       6 ]   420244  19.813%  38.558% ####
(       6,      10 ]    62674   2.955%  41.513% #
(      10,      15 ]     1021   0.048%  41.561%
(      15,      22 ]       91   0.004%  41.565%
(      22,      34 ]       25   0.001%  41.566%
(      34,      51 ]       10   0.000%  41.567%
(      51,      76 ]       18   0.001%  41.568%
(      76,     110 ]      414   0.020%  41.587%
(     110,     170 ]   384353  18.121%  59.708% ####
(     170,     250 ]   833008  39.273%  98.981% ########
(     250,     380 ]    17591   0.829%  99.811%
(     380,     580 ]      326   0.015%  99.826%
(     580,     870 ]      239   0.011%  99.837%
(     870,    1300 ]      409   0.019%  99.857%
(    1300,    1900 ]      372   0.018%  99.874%
(    1900,    2900 ]      342   0.016%  99.890%
(    2900,    4400 ]      242   0.011%  99.902%
(    4400,    6600 ]      163   0.008%  99.909%
(    6600,    9900 ]      325   0.015%  99.925%
(    9900,   14000 ]      717   0.034%  99.958%
(   14000,   22000 ]       99   0.005%  99.963%
(   22000,   33000 ]      193   0.009%  99.972%
(   33000,   50000 ]      171   0.008%  99.980%
(   50000,   75000 ]      102   0.005%  99.985%
(   75000,  110000 ]      227   0.011%  99.996%
(  110000,  170000 ]       39   0.002%  99.998%
(  170000,  250000 ]       22   0.001%  99.999%
(  250000,  380000 ]       13   0.001%  99.999%
(  380000,  570000 ]        5   0.000%  99.999%
(  570000,  860000 ]       10   0.000% 100.000%
( 1200000, 1900000 ]        1   0.000% 100.000%
...
```

```bash
mysql> show global status like 'rocksdb%';
+----------------------------------------------------+------------------------------------------+
| Variable_name                                      | Value                                    |
+----------------------------------------------------+------------------------------------------+
| rocksdb_rows_deleted                               | 129720                                   |
| rocksdb_rows_inserted                              | 1685764                                  |
| rocksdb_rows_read                                  | 14286838                                 |
| rocksdb_rows_updated                               | 3371427                                  |
| rocksdb_rows_deleted_blind                         | 0                                        |
| rocksdb_rows_expired                               | 0                                        |
| rocksdb_rows_filtered                              | 0                                        |
| rocksdb_system_rows_deleted                        | 0                                        |
| rocksdb_system_rows_inserted                       | 0                                        |
| rocksdb_system_rows_read                           | 0                                        |
| rocksdb_system_rows_updated                        | 0                                        |
| rocksdb_memtable_total                             | 176868392                                |
| rocksdb_memtable_unflushed                         | 46836824                                 |
| rocksdb_queries_point                              | 4653670                                  |
| rocksdb_queries_range                              | 598903                                   |
| rocksdb_table_index_stats_success                  | 0                                        |
| rocksdb_table_index_stats_failure                  | 0                                        |
| rocksdb_table_index_stats_req_queue_length         | 0                                        |
| rocksdb_covered_secondary_key_lookups              | 424614                                   |
| rocksdb_additional_compaction_triggers             | 0                                        |
| rocksdb_block_cache_add                            | 2693436                                  |
| rocksdb_block_cache_add_failures                   | 0                                        |
| rocksdb_block_cache_bytes_read                     | 6186277511303                            |
| rocksdb_block_cache_bytes_write                    | 43925914567                              |
| rocksdb_block_cache_data_add                       | 2691998                                  |
| rocksdb_block_cache_data_bytes_insert              | 43787713262                              |
| rocksdb_block_cache_data_hit                       | 47953275                                 |
| rocksdb_block_cache_data_miss                      | 2763422                                  |
| rocksdb_block_cache_filter_add                     | 0                                        |
| rocksdb_block_cache_filter_bytes_evict             | 0                                        |
| rocksdb_block_cache_filter_bytes_insert            | 0                                        |
| rocksdb_block_cache_filter_hit                     | 0                                        |
| rocksdb_block_cache_filter_miss                    | 0                                        |
| rocksdb_block_cache_hit                            | 90086503                                 |
| rocksdb_block_cache_index_add                      | 1438                                     |
| rocksdb_block_cache_index_bytes_evict              | 0                                        |
| rocksdb_block_cache_index_bytes_insert             | 138201305                                |
| rocksdb_block_cache_index_hit                      | 42133228                                 |
| rocksdb_block_cache_index_miss                     | 1438                                     |
| rocksdb_block_cache_miss                           | 2764860                                  |
| rocksdb_block_cachecompressed_hit                  | 0                                        |
| rocksdb_block_cachecompressed_miss                 | 0                                        |
| rocksdb_bloom_filter_full_positive                 | 0                                        |
| rocksdb_bloom_filter_full_true_positive            | 0                                        |
| rocksdb_bloom_filter_prefix_checked                | 0                                        |
| rocksdb_bloom_filter_prefix_useful                 | 0                                        |
| rocksdb_bloom_filter_useful                        | 0                                        |
| rocksdb_bytes_read                                 | 2027039128                               |
| rocksdb_bytes_written                              | 1578812205                               |
| rocksdb_compact_read_bytes                         | 1166561200                               |
| rocksdb_compact_write_bytes                        | 2279901099                               |
| rocksdb_compaction_key_drop_new                    | 191043                                   |
| rocksdb_compaction_key_drop_obsolete               | 668                                      |
| rocksdb_compaction_key_drop_user                   | 0                                        |
| rocksdb_flush_write_bytes                          | 712569474                                |
...
| rocksdb_number_sst_entry_put                       | 34860406                                 |
| rocksdb_number_sst_entry_singledelete              | 488841                                   |
| rocksdb_number_superversion_acquires               | 48                                       |
| rocksdb_number_superversion_cleanups               | 19                                       |
| rocksdb_number_superversion_releases               | 21                                       |
| rocksdb_row_lock_deadlocks                         | 0                                        |
| rocksdb_row_lock_wait_timeouts                     | 0                                        |
| rocksdb_select_bypass_executed                     | 0                                        |
| rocksdb_select_bypass_failed                       | 0                                        |
| rocksdb_select_bypass_rejected                     | 0                                        |
| rocksdb_snapshot_conflict_errors                   | 0                                        |
| rocksdb_stall_l0_file_count_limit_slowdowns        | 0                                        |
| rocksdb_stall_locked_l0_file_count_limit_slowdowns | 0                                        |
| rocksdb_stall_l0_file_count_limit_stops            | 0                                        |
| rocksdb_stall_locked_l0_file_count_limit_stops     | 0                                        |
| rocksdb_stall_pending_compaction_limit_stops       | 0                                        |
| rocksdb_stall_pending_compaction_limit_slowdowns   | 0                                        |
| rocksdb_stall_memtable_limit_stops                 | 0                                        |
| rocksdb_stall_memtable_limit_slowdowns             | 0                                        |
| rocksdb_stall_total_stops                          | 0                                        |
| rocksdb_stall_total_slowdowns                      | 0                                        |
| rocksdb_stall_micros                               | 0                                        |
| rocksdb_wal_bytes                                  | 808618570                                |
| rocksdb_wal_group_syncs                            | 1                                        |
| rocksdb_wal_synced                                 | 2535                                     |
| rocksdb_write_other                                | 0                                        |
| rocksdb_write_self                                 | 775839                                   |
| rocksdb_write_timedout                             | 0                                        |
| rocksdb_write_wal                                  | 1551678                                  |
+----------------------------------------------------+------------------------------------------+
```