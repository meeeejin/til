# Synchronous IO

I learned from this [page](http://stackoverflow.com/questions/5055859/how-are-the-o-sync-and-o-direct-flags-in-open2-different-alike).

## O_SYNC

O_SYNC flag guarantees that the call will not return before all data has been transferred to the disk. This still does not guarantee that the data isn't somewhere in the harddisk write cache, but it is as much as the OS can guarantee.

## fsync()

fsync() flushes all **modified data** of the file referred to by the file descriptor fd to the disk device (or other permanent storage device). This includes writing through or flushing a disk cache if present. It also flushes metadata information associated with the file.
