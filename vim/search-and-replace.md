# Search and replace

The `:substitute` command searches for a text and replace it with a another text.


Find each occurrence of **foo** (in all lines), and replace it with **bar**:

```vimscript
:%s/foo/bar/g
```

Change each **foo** to **bar**, but ask for confirmation first:

```vimscript
:%s/foo/bar/gc
```
