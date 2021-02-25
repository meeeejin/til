# NVMe Device Driver

## NVMe (Non-Volatile Memory Express)

![nvme-queues-per-core](https://i0.wp.com/www.osr.com/wp-content/uploads/NVMe_Intro_Fig2.png)

- Submission queue 및 completion queue 기반
    - Submission queue (SQ): command 전송
    - Completion queue (CQ): ending status 전송
    - Host 및 controller에서 접근 가능
    - Admin 커맨드 처리를 위해 단일 SQ/CQ 사용
    - I/O 커맨드 처리를 위해 1 ~ 64K 개의 SQ/CQ 사용
    - 각 큐는 2 ~ 64K slot을 가질 수 있으며, FIFO 기반으로 동작
- PCIe 트랜잭션을 통해 데이터 전송

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

