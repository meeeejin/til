# innodb_empty_free_list_algorithm

The system variable `innodb_empty_free_list_algorithm` delegates all the LRU flushes to the to the LRU manager thread, and never attempts to evict a page or perform a LRU single page flush by a query thread. Also it introduces a backoff algorithm to reduce buffer pool free list mutex pressure on empty buffer pool free lists.

There are two values for `innodb_empty_free_list_algorithm`:

| Value  | Description |
| ------------- | ------------- |
| legacy  | Server will use the upstream algorithm.   |
| backoff  | Percona implementation will be used, `Default` |

When `legacy` option is set, the buffer pool free list producer (the cleaner thread) always acquires the mutex with high priority. And if there is no free page in the free list, both the page cleaner thread and the user thread prepare free page and add the free page into the free list. It occurs the buffer pool free list mutex contention.

When the `backoff` is selected and if there is no free page in the free list, XtraDB flushes LRU list using the page cleaner thread only.
