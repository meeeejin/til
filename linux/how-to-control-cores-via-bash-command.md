# How to control cores via bash command

I learned from this [page](http://stackoverflow.com/questions/6372972/disable-all-but-one-core-via-bash-command).

## Disable specific cores

Disable core 0:

```bash
$ sudo -i
root# echo 0 >/sys/devices/system/cpu/cpu0/online
```

Disable core7 to core47:

```bash
$ sudo -i
root# for i in $(seq 7 47); do echo 0 >/sys/devices/system/cpu/cpu$i/online; done
```

## Enable specific cores

Enable core 0:

```bash
$ sudo -i
root# echo 1 >/sys/devices/system/cpu/cpu0/online
```

Enable core7 to core47:

```bash
$ sudo -i
root# for i in $(seq 7 47); do echo 1 >/sys/devices/system/cpu/cpu$i/online; done
```
