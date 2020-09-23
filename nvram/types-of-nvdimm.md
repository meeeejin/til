# NVDIMM Comparison

![nvdimm-comparison](https://i.imgur.com/tjtJLBk.png)

## NVDIMM-F

- Memory mapped Flash
- NAND + Controller
- Access methods: Block-oriented access through a shared command buffer (i.e., a mounted drive)
- Capacity = NAND (100's GB - 1's TB)
- Latency = NAND (10's of microseconds)

## NVDIMM-N

- Memory mapped DRAM
- DRAM + NAND + Controller + Battery
- Access methods: Direct byte- or block-oriented access to DRAM
- Capacity = DRAM DIMM (1's - 10's GB)
- Latency = DRAM (10's of nanoseconds)
- Energy source for backup; 주기적으로 또는 전원 손실 시 백업됨

## NVDIMM-P

- Memory mapped Flash and memory mapped DRAM
- DRAM + NAND + Controller
- Access methods: Persistenc DRAM (-N) and block oriented drive access (-F); (-F)와 (-N)의 두 가지 동작 모드 지원, 하이브리드 방식
- Load/Store, emulated block supported
- Capacity = NVM (100's GB - 1's TB)
  - DRAM DIMM에 비해 많은 용량과 낮은 소비전력 제공
    - e.g., DRAM DIMM 16GB 모듈과 같은 수의 메모리칩을 장착했을 때, NAND 플래시는 최대 1TB 용량까지 구현
- Latency = NVM (100's of nanoseconds)

## Reference

- [NVDIMM-N cookbook: A soup-to-nuts primer on using NVDIMM-ns to improve your storage performance](https://dev.snia.org/sites/default/orig/FMS2015/Chang-Sainio_NVDIMM_Cookbook.pdf)