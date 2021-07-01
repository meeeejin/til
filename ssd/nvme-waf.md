# How to get SMART information for NVMe

1. Install `nvme-cli`:

```bash
$ sudo apt-get update
$ sudo apt-get install nvme-cli
```

2. Get SMART information for *Intel NVMe*:

```bash
$ sudo nvme intel smart-log-add /dev/nvme1
Additional Smart Log for NVME device:nvme1 namespace-id:ffffffff
key                               normalized raw
program_fail_count              : 100%       0
erase_fail_count                :   0%       190215511605248
wear_leveling                   :  86%       min: 1086, max: 47104, avg: 0
end_to_end_error_detection_count:   0%       429496780544
crc_error_count                 :   0%       281470688296960
timed_workload_media_wear       :   0%       4194240,000%
timed_workload_host_reads       : 100%       65535%
timed_workload_timer            :   0%       263882790666240 min
thermal_throttle_status         :   0%       0%, cnt: 6553600
retry_buffer_overflow_count     :   0%       0
pll_lock_loss_count             :   0%       0
nand_bytes_written              :   0%       sectors: 0
host_bytes_written              :   0%       sectors: 0
```

- `nand_bytes_written`: The bytes written to NAND cells
- `host_bytes_written`: The bytes written to the NVMe storage from the system
- **Write Amplification Factor (WAF)** = `nand_bytes_written` / `host_bytes_written` 

:warning: Not all NVMe devices support it:

```bash
The following are all installed plugin extensions:
  intel           Intel vendor specific extensions
  lnvm            LightNVM specific extensions
  memblaze        Memblaze vendor specific extensions
  wdc             Western Digital vendor specific extensions
  huawei          Huawei vendor specific extensions
  micron          Micron vendor specific extensions
```