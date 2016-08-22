# Multi-threaded LRU flushing

I learned from this [blog](https://www.percona.com/blog/2016/05/05/percona-server-5-7-multi-threaded-lru-flushing/).

## Background

If for some reason the free page demand exceeds the cleaner thread flushing capability, the server will execute a single-page LRU flush, which is performed in the context of the query thread itself.

The problem with this flushing mode is that it will iterate over the LRU list of a buffer pool instance, while holding the buffer pool mutex in InnoDB (or the finer-grained LRU list mutex in XtraDB). Once the single-page flusher finds a page to flush, it might have trouble in getting a free doublewrite buffer slot. That suggested that single-page LRU flushes are never a good idea.

The easiest way not to avoid a single-page flush is simply not to do it! Wait until a cleaner thread finally provides some free pages for the query thread to use. This is related to the `innodb_empty_free_list_algorithm` server option.

Even with this strategy it’s still a bad situation to be in, as it causes query stalls when page cleaner threads aren’t able to keep up with the free page demand.

### MySQL 5.7 LRU/flush list flushing

![alt tag](https://www.percona.com/blog/wp-content/uploads/2016/03/MySQL-MT-flushing-cropped.png)

LRU batch flushing does not necessarily happen when it’s needed the most. All buffer pool instances have their **LRU lists flushed first (for free pages)**, and **flush lists flushed second (for checkpoint age and buffer pool dirty page percentage targets)**. If the flush list flush is in progress, LRU flushing will have to wait until the next iteration. Further, all flushing is synchronized once per second-long iteration by the coordinator thread waiting for everything to complete. This one second mark may well become a thirty or more second mark if one of the workers is stalled. So if we have a very hot buffer pool instance, everything else will have to wait for it. And it’s long been known that buffer pool instances are not used uniformly (some are hotter and some are colder).

## Multi-threaded LRU flusher

A fix should:

- Decouple the “LRU list flushing” from “flush list flushing” so that the two can happen in parallel if needed.
- Recognize that different buffer pool instances require different amounts of flushing, and remove the synchronization between the instances.

![alt tag](https://www.percona.com/blog/wp-content/uploads/2016/03/Untitled-drawing-12.png)

### Features

- Each buffer pool instance has its own private LRU flusher thread.
- The thread monitors the free list length of its instance.
- The thread flushes.
- The thread sleeps until the next free list length check.
- The sleep time is adjusted depending on the free list length: thus a hot instance LRU flusher may not sleep at all in order to keep up with the demand, while a cold instance flusher might only wake up once per second for a short check.

LRU flusher threads are the only threads that can flush a given buffer pool instance.
