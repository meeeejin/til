# Merge operation

1. There is *no free log sector* in an erase unit.
2. **The IPL storage manager** allocates a free erase unit.
3. Also, it computes the new version of the data pages by applying the log records to the previous version.
4. And it writes the new version into the free erase unit.
5. Then, it erases and frees the old erase unit.

The algorithmic description of the merge operation is as follows.

```bash
	Input: A: an old erase unit to merge
	Output:	B: a new erase unit with merged content

	procedure Merge(A, B)
1:	allocate a free erase unit B
2:	for each data page p in A do
3:		if any log record for p exists then
4:	    	p'' ← apply the log record(s) to p
5:	        write p'' to B
		else
6:	    	write p to B
	    endif
	endfor
7:	erase and free A
```

When a merge operation is completed for the data pages stored in a particular erase unit, the content of the erase unit (= the merged data pages) is physically relocated to another erase unit in flash memory. Therefore, **the logical-to-physical mapping of the data pages should be updated** as well.

*The mapping information needs to be updated only when a merge operation is performed.*
