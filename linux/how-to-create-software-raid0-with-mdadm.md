# How to create software RAID 0 with mdadm

1. Install the `mdadm` package:

```bash
$ sudo apt-get install mdadm
```

2. Create a RAID 0 array (`/dev/md0`) using the `mdadm --create` command:

```bash
$ sudo mdadm --create --verbose /dev/md0 --level=0 --raid-devices=4 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1
```

3. Check the created RAID 0 array:

```bash
$ cat /proc/mdstat 
Personalities : [raid0] 
md0 : active raid0 nvme3n1[3] nvme2n1[2] nvme1n1[1] nvme0n1[0]
      2000429056 blocks super 1.2 512k chunks
      
unused devices: <none>
```

4. Create a partition on the array:

```bash
$ sudo fdisk /dev/md0
```

5. Create a filesystem on the array:

```bash
$ sudo mkfs.ext4 /dev/md0p1
```

6. Mount the array:

```bash
$ sudo mount /dev/md0p1 test_data
```

7. Save the array layout to make sure that the array is reassembled automatically at boot:

```bash
$ sudo mdadm --detail --scan | sudo tee -a /etc/mdamd/mdadm.conf
$ sudo update-initramfs -u
```

## Reference

- [How To Create RAID Arrays with mdadm on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-create-raid-arrays-with-mdadm-on-ubuntu-16-04)