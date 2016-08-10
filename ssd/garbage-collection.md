# Garbage collection

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/).

## Garbage collection

The garbage collection process in the SSD controller ensures that “stale” pages are erased and restored into a “free” state so that the incoming write commands can be processed.

### Data type

- *Cold* or *static* data: data that changes rarely
- *Hot* or *dynamic* data: data that is updated frequently

### Split cold and hot data

If a page stores partly cold and partly hot data, then the cold data will be copied along with the hot data during garbage collection for wear leveling, increasing write amplification due to the presence of cold data.

This can be avoided by splitting cold data from hot data, simply by storing them in separate pages.

The **drawback** is then that the pages containing the cold data are less frequently erased, and therefore the blocks storing cold and hot data have to be swapped regularly to ensure *wear leveling*.

### Buffer hot data

Extremely hot data should be buffered as much as possible and written to the drive as infrequently as possible.

### Invalidate obsolete data in large batches

When some data is no longer needed or need to be deleted, it is better to wait and invalidate it in a large batches in a **single operation**. This will allow the garbage collector process to handle larger areas at once and will help minimizing internal fragmentation.