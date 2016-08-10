# Over-provisioning

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-4-advanced-functionalities-and-internal-parallelism/).

## Over-provisioning

Over-provisioning is simply *having more physical blocks than logical blocks*, by keeping a ratio of the physical blocks reserved for the controller and not visible to the user (OS or file system).

### Why manufacturers offer over-provisioning?

To cope with the inherent limited lifespan of NAND-flash cells. The invisible over-provisioned blocks are here to seamlessly replace the blocks wearing off in the visible space.

### Ratio of over-provisioning area

As a rule of thumb, somewhere around 25% of over-provisioning is recommended for sustained workload of random writes. If the workload is not so heavy, somewhere around 10-15% may be largely enough.

### Over-provisioning is useful for wear leveling and performance

The over-provisioning will act as a buffer of NAND-flash blocks, helping the garbage collection process to absorb peaks of writes.
