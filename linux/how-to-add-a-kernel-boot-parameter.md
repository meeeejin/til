# How to add a kernel boot parameter

1. Open the GRUB file:

```bash
$ sudo vi /etc/default/grub
```

2. Find `GRUB_CMDLINE_LINUX_DEFAULT` and append a new paramter to its end:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash mem=2048m"
```

3. Update the GRUB configuration:

```bash
$ sudo update-grub
```

4. Reboot:

```bash
$ sudo reboot
```