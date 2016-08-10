# Logical block mapping

I learned from this [page](http://codecapsule.com/2014/02/12/coding-for-ssds-part-3-pages-blocks-and-the-flash-translation-layer/).

## Logical block mapping

The logical block mapping translates logical block addresses (LBAs) from the host space into physical block address (PBAs) in the physical NAND-flash memory space. This mapping table is stored in the RAM of the SSD for speed of access, and is persisted in flash memory in case of power failure. When the SSD powers up, the table is read from the persisted version and reconstructed into the RAM of the SSD

## Page-level mapping

- The naive approach.
- The page-level mapping maps any logical page from the host to a physical page.
- The major **drawback** is that the mapping table requires a lot of RAM.

## Block-level mapping

- In case of workloads with a lot of small updates, full blocks of flash memory will be written whereas pages would have been enough.
- This increases the **write amplification** and makes block-level mapping widely inefficient.

The tradeoff between page-level mapping and block-level mapping is the one of *performance* versus *space*.

## Log-block mapping

- Log-block mapping uses an approach similar to log-structured file systems.
- Incoming write operations are written sequentially to log blocks. When a log block is full, it is merged with the data block associated to the same logical block number (LBN) into a "free" block.
- Only a few log blocks need to be maintained.

### Switch-merge (Swap-merge)

Let’s imagine that all the addresses in a logical block were written at once. This would mean that all the new data for those addresses would be written to the *same log block*. Since this log block contains the data for a whole logical block, it would be **useless to merge this log block with a data block into a free block**, because the resulting free block would contain exactly the data as the log block. It would be faster to only update the metadata in the data block mapping table, and switch the the data block in the data block mapping table for the log block, this is a switch-merge.
