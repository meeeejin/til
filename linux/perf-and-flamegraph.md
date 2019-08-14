# perf and FlameGraph

## perf

- [perf(1) - Linux manual page](http://man7.org/linux/man-pages/man1/perf.1.html)
- [Linux perf Examples](http://www.brendangregg.com/perf.html)

아래의 커맨드로 설치 가능:

```bash
$ sudo apt-get install linux-tools-common linux-tools-generic linux-tools-`uname -r`
```

## [FlameGraph](http://www.brendangregg.com/FlameGraphs)

- X축은 스택 프로파일 모집단을 보여줌 (시간 순 아님)
- Y축은 스택 깊이를 보여줌
- 프레임이 넓을수록 스택에 더 자주 존재함
- hottest code-path일수록 더 넓게 나타남
- 각 스택 사각형의 너비는 해당 함수가 CPU에 있었던 총 시간을 나타냄 (샘플 카운트 기반)
    - 넓은 사각형을 보이는 함수는 좁은 사각형을 보이는 함수보다 execution 당 CPU를 더 많이 소비한 것일 수도 있고, 또는 단순히 더 자주 호출된 것일 수도 있음
    - 호출 횟수는 표시되지 않음

## 분석 방법

```bash
$ sudo perf record -F 99 -p xxx -g -- sleep 1200
$ sudo perf script > out.perf
$ ./stackcollapse-perf.pl out.perf > out.folded
$ ./flamegraph.pl out.folded > test.svg
```

- perf로 call stack 수집함
- perf data를 파싱해서 FlameGraph로 그림

### PostgreSQL perf 사용 방법

`src/Makefile.global` 파일에서 `enable_debug = yes`로 변경하고 `CFLAGS`에 `-g` 옵션 추가한 후 다시 build:

```bash
$ vi src/Makefile.global
...
enable_debug  = yes
...
CFLAGS = -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wmissing-format-attribute -Wformat-security -fno-strict-aliasing -fwrapv -fexcess-precision=standard -g -O2
...

$ make -j8 install
``` 

아래의 perf 명령어로 call stack 수집 및 [FlameGraph](https://www.dropbox.com/s/k7kvingw77gofqt/without-fprintf.svg?dl=0) 그림:

```bash
$ sudo perf record -F 99 -p XXX --call-graph dwarf sleep XXX
$ sudo perf script > out.perf
$ ./stackcollapse-perf.pl out.perf > out.folded
$ ./flamegraph.pl out.folded > test.svg
```