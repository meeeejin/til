# Writes

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-5-access-patterns-and-system-optimizations/).

## Random writes are not always slower than sequential writes

If the writes are small, then random writes are slower than sequential writes.
If writes are both multiple of and aligned to the size of a clustered block, the random writes will use all the available levels of *internal parallelism*, and will perform just as well as sequential writes.

## A single large write is better than many small concurrent writes

A single large write request offers the **same throughput** as many small concurrent writes, however in terms of latency, a large single write has a **better response time** than concurrent writes. Therefore, whenever possible, it is best to perform large writes.

## When the writes are small and cannot be grouped or buffered, multi-threading is beneficial

Many concurrent small write requests will offer a better throughput than a single small write request. So if the I/O is small and cannot be batched, it is better to use *multiple* threads.
