# How InnoDB performs a checkpoint

> 수정 중..

이 문서는 [InnoDB 체크포인트 관련 포스팅(2011/01/29)](https://www.xaprb.com/blog/2011/01/29/how-innodb-performs-a-checkpoint/)을 한글 번역한 문서로, 개인적인 학습 및 사용을 목적으로 번역된 비공식 문서이다.

## Introduction

InnoDB의 체크포인트 알고리즘은 문서화가 잘 안 되어있다. 체크포인트를 이해하기 위해서는 InnoDB가 수행하는 다른 많은 것들을 이해해야하기 때문에, 긴 블로그 포스트로도 설명하기에는 너무 복잡하다. 단순화 시킨 high 레벨의 용어를 사용해 InnoDB의 체크포인트 과정을 설명한 것이 많은 도움이 되기를 바란다.

약간의 배경 지식: [Gray and Reuter’s classic text on transaction processing](https://www.amazon.com/gp/product/1558601902/?tag=xaprb-20)은 605 페이지에서부터 두 가지 유형의 체크포인트를 소개했다. 즉, **Sharp checkpoint** 가 있고, **fuzzy checkpoint** 가 있다.

### Sharp checkpoint

**Sharp checkpoint** 는 커밋(commit)된 트랜잭션들의 수정된 모든 페이지들을 디스크에 플러시하고, `가장 최근에 커밋된 트랜잭션의 LSN(Log Sequence Number)`을 기록한다. 커밋되지 않은 트랜잭션들의 수정된 페이지들은 플러시되지 않아야 한다. 플러시 한다면, 이는 write-ahead logging 규칙을 위반하게 된다. 복구 시, 로그 REDO는 체크포인트가 발생한 LSN부터 시작할 수 있다. **Sharp checkpoint** 는 체크포인트 시 디스크로 플러시 되는 모든 것이 단일 시점(즉, 체크포인트 LSN)으로 일관되기 때문에 "sharp"라고 불린다.

### Fuzzy checkpoint

**Fuzzy checkpoint** 는 더 복잡하다. **Fuzzy checkpoint** 는 **sharp checkpoint** 가 플러시 했을 모든 페이지를 플러시 할 때까지, 시간이 지남에 따라 페이지를 플러시 한다. 체크포인트는 두 개의 LSN(`체크포인트가 시작된 시점`과 `종료된 시점`)을 기록함으로써 완료된다. 그러나 플러시 된 페이지는 각각 단일 시점에 플러시 되기 때문에 서로 일관성이 없을 수 있다. 따라서, 이 체크포인트는 "fuzzy"라고 불린다. 초기에 플러시 된 페이지는 그 이후로 수정되었을 수 있고, 늦게 플러시 된 페이지는 시작 LSN보다 새로운 LSN을 가질 수 있다. **Fuzzy checkpoint** 는 시작 LSN부터 종료 LSN까지 REDO를 수행함으로써, 개념적으로 **sharp checkpoint** 로 변환될 수 있다. 그리고나서, 복구 시, REDO는 체크포인트가 시작된 LSN부터 시작할 수 있다.

## InnoDB's checkpoint

InnoDB는 **fuzzy checkpoint** 를 한다고 알려져 있지만, 사실 InnoDB는 위의 두 체크포인트를 모두 수행한다. 데이터베이스가 종료될 때, InnoDB는 **sharp checkpoint** 를 수행한다. 정상 작동 중에는, **fuzzy checkpoint** 를 수행한다. 그리고 InnoDB의 **fuzzy checkpoint** 구현은 [Gray & Reuter의 TP 책](https://www.amazon.com/gp/product/1558601902/?tag=xaprb-20) 에서 설명한 것과 정확히 동일하지는 않다.

여기서 문제가 복잡해진다: 나는 약간의 미묘함을 설명하려고 노력할 것인데, InnoDB에서는 주기적으로 발생하는 중요한 이벤트로써의 체크포인트가 아니라, 거의 끊임없이 체크포인트를 수행함으로써 균일한 서비스 품질을 제공한다. 즉, 정상적인 동작 중에 InnoDB는 실제로 체크포인트를 수행하지 않는다고 말할 수 있다. 대신, 디스크 상의 데이터베이스 상태는 끊임없이 진행 중인(advancing) **fuzzy checkpoint** 상태다. 그 진행 상태(advance)는 데이터베이스의 정상적인 동작 중의 일부분으로, dirty 페이지를 주기적으로 플러시 함으로써 수행된다. 새로운 버전이 릴리즈 될 때마다 변경되는 사항이 많아서 세부 사항을 여기에 쓰기엔 너무 많고 복잡하지만, 개요를 간단히 설명하려고 한다.

InnoDB는 많은 데이터베이스 페이지를 가지는 큰 버퍼 풀을 메모리에 유지하며, 디스크에 수정 사항을 즉시 쓰진 않는다. 대신, dirty 페이지들이 디스크에 기록되기 전에 여러 번 수정되기를 바라면서, dirty 페이지를 메모리에 유지한다. 이를 **쓰기 결합**(**write combining**)이라고 하며, 성능 최적화 방안 중 하나다. InnoDB는 몇몇 list들을 통해 버퍼 풀의 페이지를 관리한다: `Free list`는 사용 가능한 페이지들을 모아놓은 list이고, `LRU list`는 가장 오랫동안 사용되지 않은(least recently used) 페이지를 tail에 유지하는 list이며, `flush list`는 모든 dirty 페이지를 가지고 있는 list로, LSN 순으로 가장 오랫동안 수정되지 않은(least recently modified) 페이지를 head에 유지하는 list이다.

이 list들은 모두 중요하지만, 여기서는 간략한 설명을 위해 flush list에 초점을 맞출 것이다. InnoDB는 제한된 크기의 버퍼 풀을 가지기 때문에, 만약 InnoDB가 디스크에서 읽을 필요가 있는 페이지를 저장할 공간이 없다면, 공간을 확보하기 위해 dirty 페이지를 비우고 해제(free)해야 한다. 하지만 이 과정은 느리다. 따라서, InnoDB는 dirty 페이지를 지속적으로 플러시 하여, 플러시 하지 않고도 대체될 수 있는 clean 페이지들을 유지한다. 이를 통해, 위의 과정(하나의 page read를 위해 dirty 페이지를 비우고 free하는 과정)을 피하려 한다. 즉, flush list에서 가장 오래 전에 수정된 페이지를 플러시 하여, 특정 high-water mark에 도달하지 않으려 한다. 이 때, 디스크 상의 물리적 위치와 LSN(페이지들의 수정 시간)을 기준으로 페이지를 선택한다.

High-water mark를 피하는 것 외에도, InnoDB는 매우 중요한 low-water mark를 피해야 한다. InnoDB의 트랜잭션 로그(일명, REDO 로그와 WAL 로그)는 고정 크기이며, 순환(circular) 방식으로 기록된다. 그러나 아직 플러시 되지 않은 dirty 페이지에 대한 변경 레코드를 포함한 경우, 해당 로그를 덮어 쓸 수 없다. 이러한 일이 발생하고 서버가 손상된 경우, 해당 변경 사항을 가지는 모든 레코드가 손실된다. 따라서, InnoDB는 주어진 페이지의 수정 사항을 기록하는 데 제한된 시간을 가지고 있다. 왜냐하면 진행 중인 트랜잭션 로깅은 로그의 빈 영역을 갈망하고 있기 때문이다. 로그의 크기는 제한되어 있다. 로그를 기록하는 활동이 한 바퀴를 돌아 로그의 tail에 부딪치게 되면, InnoDB가 로그의 일부 영역을 비우려고 서둘러서 일을 하는 동안, 서버는 정지해 있다(very bad stall). 이것이 InnoDB가 수정된지 오래된 순서대로 플러시 하려는 이유이다: 가장 오래 전에 수정된 페이지는 로그에서 가장 뒤 쪽에 있으며, 가장 먼저 이 상황에 부딪히는 페이지다. 플러시 되지 않은 dirty 페이지 중 가장 오래된 페이지의 LSN은 트랜잭션 로그의 low-water mark이고, InnoDB는 가능한 한 트랜잭션 로그에 많은 공간을 확보하기 위해 low-water mark를 높이려고 한다. 로그를 더 크게 만드는 것은 로그 영역을 급하게 확보해야 할 상황을 줄이고, 플러시를 보다 효율적으로 하기 위한 다양한 성능 최적화 방안을 사용할 수 있게 만든다.

이제 간단한 설명을 통해 InnoDB가 **fuzzy checkpoint** 를 수행하는 방식을 이해할 수 있다. InnoDB가 dirty 페이지를 디스크에 플러시 할 때, 가장 오래된 dirty 페이지의 LSN을 찾고, 그 LSN을 체크포인트의 low-water mark로 취급한다. 그런 다음, 트랜잭션 로그 헤더에 이 값을 쓴다. `log_checkpoint_margin()` 및 `log_checkpoint()` 함수에서 이를 확인할 수 있다. 즉, InnoDB가 flush list의 head에서부터 dirty 페이지를 플러시 할 때마다, 시스템에서 가장 오래된 LSN을 전진(advance)시켜 체크포인트를 만들고 있다. 그리고 이것이 연속적인 **fuzzy checkpoint** 가 별도의 이벤트로서 "체크포인트를 수행"하는 것 없이도 구현되는 방식이다. 서버가 비정상적으로 종료되면, 복구는 단순히 가장 오래된 LSN부터 진행된다.

InnoDB가 종료되면, 추가 작업이 수행된다. 먼저, 데이터에 대한 모든 업데이트를 중지한다. 그리고 나서 모든 dirty 버퍼를 디스크로 플러시 한다. 그 다음, 트랜잭션 로그에 현재 LSN을 기록한다. 이것은 **sharp checkpoint** 이다. 또한, 데이터베이스의 각 데이터 파일의 첫 번째 페이지에 LSN을 쓰는데, 이는 그 LSN까지 체크포인트를 수행했다는 표시다. 이렇게 하면 복구 중일 때와 데이터 파일을 열 때 추가 최적화가 가능하다.  

실제 시스템에서 이러한 체크포인트가 어떻게 수행되는지 세부적으로 알고 싶다면, 더 많은 공부가 필요하다. 이 과정에 도움이 되는 자료는 다음과 같다:

- [Gray and Reuter’s book](https://www.amazon.com/gp/product/1558601902/?tag=xaprb-20)
- [Mark Callaghan’s note on fuzzy checkpoints](http://www.facebook.com/note.php?note_id=408059000932)
- [Peter Zaitsev’s post on why the flushing algorithm in older InnoDB used to cause spikes](http://www.mysqlperformanceblog.com/2006/05/10/innodb-fuzzy-checkpointing-woes/)
- [Mark Callaghan’s slides from Percona Performance Conference 2009](http://www.percona.com/ppc2009/PPC2009_Life_of_a_dirty_pageInnoDB_disk_IO.pdf)

이와 비슷한 복잡한 주제로는 "InnoDB가 데이터베이스의 워크로드를 따라 잡기 위해 dirty 페이지를 적절한 속도로 플러시 하는 방법"이 있다. 속도가 너무 빠르면 서버가 너무 많은 작업을 수행한다. 속도가 너무 느리면 서버가 뒤쳐져 워크로드를 따라 잡으려고 서둘러 작업을 하고, 이는 격렬한 플러시 활동(`furious flushing`)과 서비스 품질 저하를 초래한다. 주장하건대, Percona Server의 XtraDB 스토리지 엔진(InnoDB의 변종)은 이와 관련된 가장 발전되고 효과적인 알고리즘을 가지고 있다. Percona Server는 이를 `adaptive checkpointing`이라 부른다. InnoDB도 비슷하게 구현했지만, 올바르게 조정하는 게 더 어렵다. InnoDB는 이를 보다 정확한 이름인 `adaptive flushing`이라 부른다. 이와 관련된 포스팅은 아주 많은데, 구현 방식과 동작 방식을 가장 간결하게 요약한 포스팅은 다음과 같다:

- [Dimitri Kravtchuk’s blog post about adaptive flushing and the innodb_io_capacity variable](http://dimitrik.free.fr/blog/archives/2010/07/mysql-performance-innodb-io-capacity-flushing.html)
- [Vadim’s benchmarks of Percona Server and MySQL 5.5.8](http://www.mysqlperformanceblog.com/2011/01/03/mysql-5-5-8-in-search-of-stability/), `adaptive flushing`이 잘 작동하도록 조정하는 방법을 보여줌
- [Percona Server documentation for adaptive checkpointing](http://www.percona.com/docs/wiki/percona-server:features:innodb_io)
- [My own blog post about balancing dirty page flushing and write combining](https://www.xaprb.com/blog/2010/05/25/dirty-pages-fast-shutdown-and-write-combining/)
