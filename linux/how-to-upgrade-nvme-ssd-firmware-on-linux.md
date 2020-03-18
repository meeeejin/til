# How to upgrade NVMe SSD firmware on Linux

1. Install [NVMe CLI](https://github.com/linux-nvme/nvme-cli/):

```bash
$ git clone https://github.com/linux-nvme/nvme-cli
$ make
$ make install
```

2. Check your current firmware version:

```bash
$ sudo nvme id-ctrl /dev/nvme0 | grep fr
fr      : xxxx
frmw    : 0x17
```

3. Download the firmware update:

```bash
$ sudo nvme fw-download /dev/nvme0 --fw=/path/to/nvme.fw
```

4. Commit the update:

```bash
$ sudo nvme fw-commit /dev/nvme0 --slot=0 --action=1
```

5. Reboot:

```bash
$ sudo reboot
```

6. Check if the firmware update was successful:

```bash
$ sudo nvme id-ctrl /dev/nvme0 | grep fr
fr      : yyyy
frmw    : 0x17
```