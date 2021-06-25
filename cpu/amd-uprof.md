# AMD uProf

## 특징

- AMD CPU 기반 시스템 성능 분석 툴
- CPU Profiling 및 Power Profiling 기능 제공
- CPU Profiling 시 Instruction 기반 또는 시간 기반 샘플링 가능
    - L2, L3 캐시의 상태, Java 어플리케이션 Profiling, Call Stack 샘플링 등
- Power Profiling 시 실시간 전력 상태 등의 분석 가능
- 시간 기반 샘플링을 통해 전체 시스템에 대한 CPU Hotspot 및 Function Level Analysis 파악 가능
    - 프로파일링 시작 - 어플리케이션 실행 - 프로파일링 종료

## 설치

1. [AMD uProf](https://developer.amd.com/amd-uprof/#download) 설치

2. 원격 접속인 경우, X11 포워딩 후 실행 가능:

```bash
$ ssh -X server01
$ cd /opt/AMDuProf_3.4-475/bin
$ ./AMDuProf
```

## 예시

![cpu](https://i.imgur.com/XG3QuwB.png)

- 사용한 CPU Time 순으로 정렬한 결과임