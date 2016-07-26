# Basic operations

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/).

## Read & Write & Erase

- Reads are aligned on page size
- Writes are aligned on page size
- Pages cannot be overwritten
- Erases are aligned on block size

## Write amplification

Because writes are aligned on the page size, any write operation that is not both aligned on the page size and a multiple of the page size will require more data to be written than necessary, a concept called **write amplification**. Writing one byte will end up writing a page, which can amount up to 16 KB for some models of SSD and be extremely inefficient.

## Wear leveling

- distributes P/E cycles as evenly as possible among the blocks

Because NAND-flash cells are wearing off, one of the main goals of the FTL is to distribute the work among cells as evenly as possible so that blocks will reach their P/E cycle limit and wear off at the same time.
