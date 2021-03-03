# NVMe Device Driver

## NVMe (Non-Volatile Memory Express)

![nvme-queues-per-core](https://i0.wp.com/www.osr.com/wp-content/uploads/NVMe_Intro_Fig2.png)

- Submission queue 및 completion queue 기반
    - 큐들은 모두 고정된 크기를 갖는 circular ring 형태
    - **Submission queue (SQ): command 전송**
    - **Completion queue (CQ): ending status 전송**
    - Host 및 controller에서 접근 가능
    - **Admin 커맨드** 처리를 위해 *단일* SQ/CQ 사용
    - **I/O 커맨드** 처리를 위해 *1 ~ 64K* 개의 SQ/CQ 사용
    - 각 큐는 *2 ~ 64K* slot을 가질 수 있으며, FIFO 기반으로 동작
- PCIe 트랜잭션을 통해 데이터 전송

### Submission Queue

- 호스트의 커맨드를 컨트롤러에게 전달
- SQ는 priority를 가질 수 있음
    - Round robin, weighted round robin
- 각 커맨드는 64 bytes 크기
- 커맨드를 SQ의 head에 삽입하고, 해당 head의 위치 (pointer)를 doorbell 레지스터에 등록

### Completion Queue

- 커맨드 수행 완료시 컨트롤러가 호스트에게 응답하는 용도로 사용
- CQ의 각 entry는 *Phase Tag* (`P`) bit를 갖고 있음
    - 레지스터 참조 없이 이 bit를 사용해 entry가 새로 posting 되었는지 여부를 확인
    - CQ 통과할 때마다 컨트롤러는 `P`를 invert

### Doorbell

- 큐에 업데이트 발생 시 이를 알리는 용도로 사용되는 32 bits 레지스터
- 호스트에 의해 업데이트 됨
- SQ/CQ 마다 하나씩 존재

## NVMe Command Processing

![nvme-command-processing](https://i1.wp.com/www.osr.com/wp-content/uploads/NVMe_Intro_Fig1.png)

> Doorbell: 큐의 업데이트를 알려주는 32bit 레지스터; 호스트가 기록

1. 생성된 명령어를 호스트 메모리의 특정 SQ에 기록
2. 새로운 명령어의 존재를 알리기 위해, 호스트는 NVMe 컨트롤러의 SQ Tail Doorbell 레지스터 업데이트
3. 컨트롤러가 큐의 명령어 fetch
4. 컨트롤러는 가져온 명령어 처리
5. 명령어 처리가 끝나면, 컨트롤러는 호스트 메모리의 해당 CQ에 CQ entry 기록
6. 컨트롤러가 호스트에게 인터럽트 발생 (pin-based, MSI or MSI-X)
7. 호스트는 CQ entry를 꺼내서 확인 및 처리
8. 호스트는 인터럽트 확인 및 CQ entry 처리를 마쳤다는 의미로, CQ Head Doorbell 레지스터 업데이트

## NVMe Queue 요청 과정

> perf results

```bash
mysqld 21016 65782.742483:     184709 cycles:
        ffffffffb363bd94 native_sched_clock+0x24 ([kernel.kallsyms])
        ffffffffb363c999 sched_clock+0x9 ([kernel.kallsyms])
        ffffffffb3796369 trace_clock_local+0x9 ([kernel.kallsyms])
        ffffffffb37b9723 function_graph_enter+0x73 ([kernel.kallsyms])
        ffffffffb366e26c prepare_ftrace_return+0x5c ([kernel.kallsyms])
        ffffffffb4201be8 ftrace_graph_caller+0x78 ([kernel.kallsyms])
        ffffffffb3851765 kmalloc_slab+0x5 ([kernel.kallsyms])
        ffffffffb38a978e __kmalloc+0x2e ([kernel.kallsyms])
        ffffffffb3829115 mempool_kmalloc+0x15 ([kernel.kallsyms])
        ffffffffb3828bb1 mempool_alloc+0x71 ([kernel.kallsyms])
        ffffffffc00f9d65 nvme_queue_rq+0xc5 ([kernel.kallsyms])
        ffffffffb3af2a19 __blk_mq_try_issue_directly+0x139 ([kernel.kallsyms])
        ffffffffb3af3dbb blk_mq_request_issue_directly+0x4b ([kernel.kallsyms])
        ffffffffb3af3e96 blk_mq_try_issue_list_directly+0x46 ([kernel.kallsyms])
        ffffffffb3af8617 blk_mq_sched_insert_requests+0xb7 ([kernel.kallsyms])
        ffffffffb3af3cbb blk_mq_flush_plug_list+0x1eb ([kernel.kallsyms])
        ffffffffb3ae8d91 blk_flush_plug_list+0xd1 ([kernel.kallsyms])
        ffffffffb3ae8dec blk_finish_plug+0x2c ([kernel.kallsyms])
        ffffffffb392b315 do_blockdev_direct_IO+0x1c65 ([kernel.kallsyms])
        ffffffffb392c4de __blockdev_direct_IO+0x2e ([kernel.kallsyms])
        ffffffffb39b3fd9 ext4_direct_IO+0x2e9 ([kernel.kallsyms])
        ffffffffb3827632 generic_file_direct_write+0xa2 ([kernel.kallsyms])
        ffffffffb38277ae __generic_file_write_iter+0xbe ([kernel.kallsyms])
        ffffffffb399ed6d ext4_file_write_iter+0xbd ([kernel.kallsyms])
        ffffffffb38dc185 new_sync_write+0x125 ([kernel.kallsyms])
        ffffffffb38dc249 __vfs_write+0x29 ([kernel.kallsyms])
        ffffffffb38dcf21 vfs_write+0xb1 ([kernel.kallsyms])
        ffffffffb38df6a6 ksys_pwrite64+0x76 ([kernel.kallsyms])
        ffffffffb38df6de __x64_sys_pwrite64+0x1e ([kernel.kallsyms])
        ffffffffb3604417 do_syscall_64+0x57 ([kernel.kallsyms])
        ffffffffb420008c entry_SYSCALL_64_after_hwframe+0x44 ([kernel.kallsyms])
            7f491f98c13f __libc_pwrite64+0x4f (/lib/x86_64-linux-gnu/libpthread-2.27.so)
        69c4190067c41900 [unknown] ([unknown])
```

1. `__libc_pwrite64`
2. `do_syscall_64` => `__x64_sys_pwrite64` => `ksys_pwrite64`
    - A system call is an interrupt; `syscall(number, arguments)`
3. `vfs_write` => `__vfs_write` => `new_sync_write`
    - `vfs_write(f.file, buf, count, &pos);`
4. `ext4_file_write_iter` => `__generic_file_write_iter` => `generic_file_direct_write` => `ext4_direct_IO` => `__blockdev_direct_IO` => `do_blockdev_direct_IO`
5. `blk_finish_plug` => `blk_flush_plug_list` => `blk_mq_flush_plug_list` => `blk_mq_sched_insert_requests` => `blk_mq_try_issue_list_directly` => `blk_mq_request_issue_directly` => `__blk_mq_try_issue_directly`
6. `nvme_queue_rq` => `nvme_setup_rw` => `blk_mq_start_request` => `nvme_submit_cmd` => `nvme_write_sq_db` => `writel` => `__raw_writel` => `IO_CONCAT`
    - `IO_CONCAT(__IO_PREFIX,writel)(b, addr);` => `IO_CONCAT(generic,writel)(b, addr);` => `_IO_CONCAT(generic,writel)(b, addr);` => `generic_writel(b, addr);`

    ```bash
    #define IO_CONCAT(a,b)  _IO_CONCAT(a,b)
    #define _IO_CONCAT(a,b) a ## _ ## b

    #define __IO_PREFIX             generic
    ```

## Reference

- [Introduction to NVMe Technology](https://www.osr.com/nt-insider/2014-issue4/introduction-nvme-technology)
- [Linux Kernel Networking](https://docplayer.net/12120292-Linux-kernel-networking-raoul-rivas.html)
- [NVMe Specification](https://nvmexpress.org/)