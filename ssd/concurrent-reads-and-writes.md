# Concurrent reads and writes

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-5-access-patterns-and-system-optimizations/).

## Separate read and write requests

A workload made of a mix of small interleaved reads and writes will prevent the internal caching and readahead mechanism to work properly, and will cause the throughput to drop. It is best to *avoid simultaneous reads and writes*, and **perform them one after the other in large chunks**, preferably of the size of the clustered block.

For example, if 1000 files have to be updated, you could iterate over the files, doing a read and write on a file and then moving to the next file, but that would be slow. It would be better to reads all 1000 files at once and then write back to those 1000 files at once.
