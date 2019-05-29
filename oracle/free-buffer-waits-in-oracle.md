# Free Buffer Waits in Oracle

- LRU scan threshold = 40%

## Common scenario

1. Scan the LRU list to the threshold to find a free buffer

2. If no free buffer is found, the foreground process posts the DBWR process to make available clean buffers.

3. While the DBWR process is at work, the Oracle session waits on the `free buffer waits` event.

> No single page flushing (MySQL)

## Related statistics

- `free buffer requested`: the number of free buffer requests
- `free buffer inspected`: how many buffers Oracle processes have to look at to get the requested free buffers

If `free buffer inspected > free buffer requested`, that means proceeses are scanning further up the LRU list to get a usable buffer.

## Causes

- Inefficient SQL statements
- **Not enough DBWR processes**
  - More aggressive DBWR process increases the chances of processes waiting on the `write complete waits` event
- Slow I/O subsytem
- Delayed block cleanouts
- **Small buffer cache**