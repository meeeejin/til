# Device initialization

## Erasing the drive

```bash
sudo dd if=/dev/zero of=/dev/**deviceID** bs=512
```

**dd** will finish with the message `No space left on device` or an `out of space error`. This shows that dd has successfully completed writing to the entire disk and there is no more space left for dd to continue.
