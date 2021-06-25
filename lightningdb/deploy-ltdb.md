# How to deploy LightningDB

1. Make LightningDB cluster 4 with the following command:

```bash
$ ltcli
Cluster '3' selected.
skt@lightningdb:3> deploy 4
Select installer

    [INSTALLER LIST]
    (1) [DOWNLOAD] lightningdb.release.release.flashbase_v1.3.1.44c438.bin
    (2) [LOCAL] bash
    (3) [LOCAL] lightningdb.release.support-binary-arrow-scan.71c63b-bugfix.bin
    (4) [LOCAL] lightningdb.release.support-binary-arrow-scan.71c63b-dirty.bin
    (5) [LOCAL] lightningdb.release.release.flashbase_v1.3.1.44c438.bin
    (6) [LOCAL] lightningdb.release.support-binary-arrow-scan.cb86e1-dirty.bin

Please enter the number, file path or url of the installer you want to use.
You can also add file in list by copy to '$FBPATH/releases/'
4
OK, lightningdb.release.support-binary-arrow-scan.71c63b-dirty.bin
```

In the above case, I chose to use the 4th binary,`lightningdb.release.support-binary-arrow-scan.71c63b-dirty.bin`.

2. Type hosts (IP address or hostname):

```bash
Please type hosts separated by comma(,) [xxx.xxx.xxx.205, xxx.xxx.xxx.206, xxx.xxx.xxx.207, xxx.xxx.xxx.208]

OK, ['xxx.xxx.xxx.205', 'xxx.xxx.xxx.206', 'xxx.xxx.xxx.207', 'xxx.xxx.xxx.208']
```

3. Define how many master processes will be created in the cluster per server:

```bash
How many masters would you like to create on each host? [128]

OK, 128
Please type ports separate with comma(,) and use hyphen(-) for range. [18400-18527]
20400-20527
OK, ['20400-20527']
```

4. Define how many slave processes will be created for a master process:

```bash
How many replicas would you like to create on each master? [0]

OK, 0
```

5. Type the number of SSDs (disks) and the path of DB files:

```bash
How many ssd would you like to use? [4]

OK, 4
Type prefix of db path [/nvme/data_]

OK, /nvme/data_
```

6. Check all settings and type *y*:

```bash
+--------------+----------------------------------------------------------------+
| NAME         | VALUE                                                          |
+--------------+----------------------------------------------------------------+
| installer    | lightningdb.release.support-binary-arrow-scan.71c63b-dirty.bin |
| hosts        | xxx.xxx.xxx.205                                                |
|              | xxx.xxx.xxx.206                                                |
|              | xxx.xxx.xxx.207                                                |
|              | xxx.xxx.xxx.208                                                |
| master ports | 20400-20527                                                    |
| ssd count    | 4                                                              |
| db path      | /nvme/data_                                                    |
+--------------+----------------------------------------------------------------+
Do you want to proceed with the deploy according to the above information? (y/n)
y
```

7. Deploy cluster:

```bash
Check status of hosts...
+-----------------+--------+
| HOST            | STATUS |
+-----------------+--------+
| xxx.xxx.xxx.205 | OK     |
| xxx.xxx.xxx.206 | OK     |
| xxx.xxx.xxx.207 | OK     |
| xxx.xxx.xxx.208 | OK     |
+-----------------+--------+
OK
Check port...
OK
Checking cluster exist...
+-----------------+--------+
| HOST            | STATUS |
+-----------------+--------+
| xxx.xxx.xxx.208 | CLEAN  |
| xxx.xxx.xxx.207 | CLEAN  |
| xxx.xxx.xxx.206 | CLEAN  |
| xxx.xxx.xxx.205 | CLEAN  |
+-----------------+--------+
OK
Tranfer intaller and execute...
 - xxx.xxx.xxx.205
 - xxx.xxx.xxx.205
 - xxx.xxx.xxx.205
 - xxx.xxx.xxx.208
Sync conf...
Complete to deploy cluster 4.
Cluster '4' selected.
We suggest that you begin by typing: cluster create
```

8. Start LightningDB:

```bash
skt@lightningdb:4> cluster create
...
```

9. Stop LightningDB:

```bash
skt@lightningdb:4> cluster stop
...
```

## Reference

- [Lightning DB Docs/Installation](https://docs.lightningdb.io/install-ltcli/)