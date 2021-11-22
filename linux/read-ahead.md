# Disk Readahead

## Get the readahead value

```bash
$ blockdev --report
RO    RA   SSZ   BSZ   StartSec            Size   Device
...
rw   256  4096  4096          0     57847840768   /dev/nvme0n1
rw   256  4096  4096       2048     28923400192   /dev/nvme0n1p1
rw   256  4096  4096   56494080     28922871808   /dev/nvme0n1p2
rw   256   512  4096          0    256060514304   /dev/sdc
rw   256   512  4096       2048    256059465728   /dev/sdc1
rw   256   512  4096          0   1000204886016   /dev/sda
rw   256   512  4096       2048   1000203091968   /dev/sda1
rw   256   512  4096          0    256060514304   /dev/sdb
rw   256   512  4096       2048    128028684800   /dev/sdb1
rw   256   512  4096  250058752    128030433280   /dev/sdb2
```

- `RA`: the size of readahead in sectors (512 bytes)
    - e.g., The readahead size: `RA * sector size = 256 * 512 = 131,072 bytes = 128 KB`

## Set the readahead value

```bash
$ sudo blockdev --setra 2048 /dev/nvme0n1
```

- `--setra`: set readahead (in 512-byte sectors)
    - e.g., `--setra 2048`: set the readahead value to 1MB (*1 * 1024 * 1024 / 512 = 2048*)