# Mid-Point Insertion Strategy in MySQL/InnoDB

## Buffer Pool LRU Algorithm

The buffer pool in MySQL/InnoDB is managed as a list using a variation of the least recently used (LRU) algorithm. When room is needed to add a new page to the buffer pool, the least recently used page is evicted and a new page is added to the **middle** of the list (not the head of the list). It has two sublists:

- New sublist: At the head, a sublist of new pages that were accessed recently
- Old sublist: At the tail, a sublist of old pages that were accessed less recently

<p align="center">
<img src="https://dev.mysql.com/doc/refman/8.0/en/images/innodb-buffer-pool-list.png" alt="lru" width="500" />
</p>

By default, the algorithm operates as follows:

1. When InnoDB reads a page into the buffer pool, it initially inserts it at the *midpoint* (the head of the old sublist).

2. Page hits in the:
    - `Old sublist` make pages *young*, moving it to the head of the new sublist (except read-ahead operations)
        - If the delay defined by `innodb_old_blocks_time` not being met, page accesses do not result in making pages young 
    - `New sublist` move pages to the head of the new sublist only if they are a certain distance from the head

3. As the database operates, pages in the buffer pool that are not accessed *age* by moving toward the tail of the list. Eventually, a page that remains unused reaches the tail of the old sublist and is evicted.

## Related System Variable

### `innodb_old_blocks_time`

Specifies how long in milliseconds a block inserted into the old sublist must stay there after its first access before it can be moved to the new sublist:

- The default value: 1000
- The value 0: A block inserted into the old sublist moves immediately to the new sublist the first time it is accessed
- The value > 0: Blocks remain in the old sublist until an access occurs at least that many milliseconds after the first access
- Non-zero values protect against the buffer pool being filled by data that is referenced only for a brief period, such as during a *full table scan*

### `innodb_old_blocks_pct`

Specifies the approximate percentage of the InnoDB buffer pool used for the old block sublist:

- The range of values: 5 to 95
- The default value: 37 (that is, 3/8 of the pool)

## Reference
- [MySQL 8.0 Reference Manual: Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)
- [MySQL 8.0 Reference Manual: InnoDB Startup Options and System Variables](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_old_blocks_time)