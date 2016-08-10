# TRIM

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-4-advanced-functionalities-and-internal-parallelism/).

## TRIM

Let’s imagine that a program write files to all the logical block addresses of an SSD: this SSD would be considered full. Now let’s assume that all those files gets deleted. The filesystem would report 100% free space, although the drive would still be full, because an SSD controller has no way to know when logical data is deleted by the host. The SSD controller will see the free space only when the logical block addresses that used to be holding the files get overwritten. At that moment, the garbage collection process will erase the blocks associated with the deleted files, providing free pages for incoming writes.

As a consequence, instead of erasing the blocks as soon as they are known to be holding stale data, the erasing is being delayed, which hurts performance badly.

Another concern is that, since the pages holding deleted files are unknown to the SSD controller, the garbage collection mechanism will continue move them around to ensure wear leveling. This increases write amplification, and interferes with the foreground workload of the host for no good reason.

A solution to the problem of delayed erasing is the TRIM command, which can be sent by the operating system to notify the SSD controller that pages are no longer in use in the logical space. With that information, the garbage collection process knows that it doesn’t need to move those pages around, and also that it can erase them whenever needed. The TRIM command will only work if the SSD controller, the operating system, and the filesystem are supporting it.
