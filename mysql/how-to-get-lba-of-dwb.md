# How to get the LBA of DWB

![ibdata1-jeremy-cole](http://d.jcole.us/blog/files/innodb/20130103/72dpi/ibdata1_File_Overview.png)

The below scripts does not contain the wasted pages (`FSP_EXTENT_SIZE / 2`) to cause the fragment array to be filled up.

## Prior to MySQL 8.0.20

1. Download the script:

```bash
$ wget https://raw.githubusercontent.com/meeeejin/scripts/master/mysql/get-dwb-lba.sh
$ sudo chmod +x get-dwb-lba.sh
```

2. Run the script:

```bash
$ ./get-dwb-lba.sh [path-to-ibdata1] [page-size]
```

- `path-to-ibdata1`: Path to the system tablespace file which contains DWB blocks (e.g., ibdata1)
- `page-size`: The page size in KB (e.g., 4, 16)

For example:

```
$ ./get-dwb-lba.sh ibdata1 4

DWB LBA of ibdata1:
  4 KB Pages, Total # of DWB pages = 256 * 2 = 512; assuming 512 byte sectors

Start           Block1          Block2          End
43520000        43520000        43522048        43524095
```

What each means is as follows:

- `path-to-ibdata1`: *ibdata1*
- `page-size`: 4KB
- Total number of DWB pages = 512
- The size of sector = 512
- `Start`: LBA where the DWB area starts (= 43520000)
- `Block1`: LBA where the first block of the DWB area starts (= 43520000)
- `Block2`: LBA where the second block of the DWB area starts (= 43522048)
- `End`: LBA where the DWB area ends (= 43524095)

## After MySQL 8.0.20

> To be updated

## Using `innodb_ruby` to dump file segments

- **4KB** pages with 32 fragment pages and 512 DWB pages:

```bash
$ innodb_space -s ibdata1 space-inodes-detail

...
INODE fseg_id=15, pages=544, frag=32 pages (15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46), full=2 extents (256-511, 512-767), not_full=0 extents () (0/0 pages used), free=0 extents ()
```

- **16KB** pages with 32 fragment pages and 64 DWB pages:

```bash
$ innodb_space -s ibdata1 space-inodes-detail

...
INODE fseg_id=15, pages=160, frag=32 pages (13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44), full=2 extents (64-127, 128-191), not_full=0 extents () (0/0 pages used), free=0 extents ()
```

## Reference

- [hdparm(8) — Linux manual page](https://man7.org/linux/man-pages/man8/hdparm.8.html)
- [InnoDB Tidbit: The doublewrite buffer wastes 32 pages (512 KiB)](https://blog.jcole.us/2013/05/05/innodb-tidbit-the-doublewrite-buffer-wastes-32-pages-512-kib/)
- [The basics of InnoDB space file layout](https://blog.jcole.us/2013/01/03/the-basics-of-innodb-space-file-layout/)
- [A parser for InnoDB file formats, in Ruby](https://github.com/jeremycole/innodb_ruby)
- [meeeejin/til/innodb_ruby](https://github.com/meeeejin/til/blob/master/mysql/innodb-ruby.md)
