# NVDIMM Comparison

![nvdimm-comparison](https://i.imgur.com/tjtJLBk.png)

## NVDIMM-F

- NAND + Controller

## NVDIMM-N

- DRAM + NAND + Controller + Battery
- DRAM의 데이터를 NAND 플래시에 백업하는 DIMM (Dual-inline memory module) + 컨트롤러
- 주기적으로 또는 전원 손실 시 백업됨

## NVDIMM-P

- DRAM + NAND + Controller
- (-F)와 (-N)의 두 가지 동작 모드 지원, 하이브리드 방식
- DRAM DIMM에 비해 많은 용량과 낮은 소비전력 제공
  - e.g., DRAM DIMM 16GB 모듈과 같은 수의 메모리칩을 장착했을 때, NAND 플래시는 최대 1TB 용량까지 구현