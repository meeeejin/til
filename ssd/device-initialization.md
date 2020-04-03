# Device initialization

- Reference: [SSD precondition](https://github.com/intel/fiovisualizer/blob/master/Workloads/Precondition/SSD%20precondition.txt)

## Erasing the drive

Fill SSD with sequential data twice of it's capacity. This will gurantee all available memory is filled with a data including factory provisioned area:

```bash
$ sudo dd if=/dev/zero of=/dev/**deviceID** bs=1024k
$ sudo dd if=/dev/zero of=/dev/**deviceID** bs=1024k
```

**dd** will finish with the message `No space left on device` or an `out of space error`. This shows that dd has successfully completed writing to the entire disk and there is no more space left for dd to continue.
