# Direct I/O

I learned from this [page](http://stackoverflow.com/questions/5055859/how-are-the-o-sync-and-o-direct-flags-in-open2-different-alike).

## Direct I/O
Direct I/O is a feature of the file system whereby file reads and writes go directly from the applications to the storage device, **bypassing** the operating system read and write caches. Direct I/O is used only by applications (such as databases) that manage their own caches.

## O_DIRECT

O_DIRECT flag only promises that the kernel will avoid copying data from user space to kernel space, and will instead write it directly via DMA (Direct memory access; if possible). Data does not go into caches. There is no strict guarantee that the function will return only after all data has been transferred.
