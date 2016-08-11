# Reads

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-5-access-patterns-and-system-optimizations/).

## To improve the read performance, write related data together

Read performance is a consequence of the write pattern. When a large chunk of data is written at once, it is spread across separate NAND-flash chips. Thus you should write related data in the same page, block, or clustered block, so it can later be read faster with a single I/O request, by taking advantage of the internal parallelism.

## A single large read is better than many small concurrent reads

Concurrent random reads cannot fully make use of the readahead mechanism. In addition, multiple Logical Block Addresses may end up on the same chip, not taking advantage or of the internal parallelism. Moreover, a large read operation will access sequential addresses and will therefore be able to use the readahead buffer if present. Consequently, it is preferable to issue large read requests.
