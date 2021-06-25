# How to monitor the status of LTDB clusters

- `cli cluster info`

```bash
skt@lightningdb:3> cli cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:400
cluster_size:400
cluster_current_epoch:400
cluster_my_epoch:334
cluster_stats_messages_ping_sent:188963
cluster_stats_messages_pong_sent:188819
cluster_stats_messages_meet_sent:252
cluster_stats_messages_sent:378034
cluster_stats_messages_ping_received:188511
cluster_stats_messages_pong_received:188565
cluster_stats_messages_meet_received:308
cluster_stats_messages_fail_received:41
cluster_stats_messages_received:377425
```

- `cli cluster nodes`

```bash
skt@lightningdb:3> cli cluster nodes
0014d91e57b27942bfcf727c6fcacf7b862ff674 xxx.xxx.xxx.207:18384 master - 0 1624614884000 285 connected 13844-13884
bbdcac8ea5af909018a64a2fa686e765e1f72bdb xxx.xxx.xxx.206:18339 master - 0 1624614882000 140 connected 6431-6471
8ebff2954b9e88c36bf19cb9b442d5e55a5ee305 xxx.xxx.xxx.207:18381 master - 0 1624614887000 282 connected 13353-13393
817db1bae928b50dd845dd877275921384bd881f xxx.xxx.xxx.207:18392 master - 0 1624614888000 293 connected 15155-15195
f03fc21a238b43bc9b66b70841304b0d74c37a0b xxx.xxx.xxx.208:18386 master - 0 1624614893000 387 connected 14213-14253
```