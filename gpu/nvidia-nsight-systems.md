# Nvidia Nsight Systems

## 특징

- Nvidia GPU 기반 시스템 성능 분석 툴
- 크게 시스템 전반적인 CPU 및 GPU 활용률과 CPU-GPU 간 상호작용을 확인할 수 있는 성능 Metric을 제공
    - CPU core 활용률, backtrace 샘플링, GPU 워크로드 트레이스, GPU 및 CPU의 context switch 샘플링, CUDA/OpenGL API 사용 현황 등
- 특정 프로세스를 명시해 Profiler 자체에서 Profiling 대상이 되는 프로세스를 실행하고 성능 지표 수집
    - e.g., 프로세스 시작 - 프로파일링 시작 - 프로파일링 종료
- 기존에 동작 중인 프로세스의 경우, Nsight Systems로 실행한 프로세스가 아니라면 Profiling 불가능

## 설치

1. Nvidia Developer 홈페이지에서 [Nsight Systems](https://developer.nvidia.com/gameworksdownload#?dn=nsight-systems-2021-2-1-58) 설치

2. 원격 접속인 경우, X11 포워딩 후 실행 가능:

```bash
$ ssh -X server01
$ cd /opt/nvidia/nsight-systems/2021.2.1/bin
$ ./nsys-ui
```

## 예시 (Nsight Systems 2021.2)

![gpu](https://i.imgur.com/eSgV8tA.png)

- 가로 축: 어플리케이션 수행 시간
- 세로 축:
    - GPU Metric (10 kHz)
    - GPC Clock Frequency
    - SYS Clock Frequency
    - GR Active
    - SM Active
    - SM Instructions
    - SM Warp Occupancy
    - DRAM Bandwidth
    - PCIe Bandwidth
- SM Warp Occupancy 그래프:
    - 노란색: 현재 Computing에 사용 중인 Warp의 개수
    - 회색: Active한 SM 중 사용 가능한 (현재 사용 중이지 않은) Warp의 개수
- 벤치마크 수행 중 모든 쿼리에 걸쳐 Active SM Warp의 약 50~60%를 Computing에 사용
    - 높은 Warp 활용률? 