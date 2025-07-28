---
title: HotSpot JVM G1 GC
description: ''
date: Jan 7, 2024
---

## TL;DR
> - G1 GC는 CMS GC와 마찬가지로 Suspend Time을 분산시켜 응답시간을 개선하는데 초점을 맞춘 GC.
> - garbage(unreachable objects)들로만 이루어진 region 부터 공간을 확보한다는 의미에서 붙여진 이름.
> - 힙 영역을 동일한 크기의 Region으로 나누어 관리.
> - 언제나 꽉 찼거나 거의 다 찬 영역만을 대상으로 GC를 진행하여 시간 최적화.
> - 힙 영역이 작을수록 별로 좋은 성능을 발휘하지 못함.

---

## Garbage-First Garbage Collector(G1 GC)
G1 GC는 서버의 프로세서와 메모리의 사이즈가 커짐에 따라 개선된 GC로 일시 정지 시간을 최소화하면서, 따로 설정을 하지 않아도 가능한 한 처리율(throughput)도 확보하는 것이 목표이다. G1 GC(Garbage-First Garbage Collector)는 garbage(unreachable objects)들로만 이루어진 region 부터 공간을 확보한다는 의미에서 붙여진 이름이다.

- 대용량 메모리가 있는 다중 프로세서 시스템을 대상으로 하는 서버-스타일 가비지 컬렉터
- 높은 처리량과 낮은 Stop-The-World(STW) 지향
- CMS의 개선안으로 계획
  - CMS GC보다 효율적으로 동시에 Application과 GC를 진행할 수 있고, 메모리 Compaction 과정까지 지원
- 쓰레기 비중이 높은 heap region을 집중적으로 수집
- JDK 7에서 정식으로 G1 GC를 포함하여 제공되었고 Java 9부터 디폴트로 설정

**G1을 쓰면 도움이 되는 상황**
- Java heap의 50% 이상이 라이브 데이터.
- 시간이 흐르면서 객체 할당 비율과 프로모션 비율이 크게 달라진다.
- GC가 너무 오래 걸린다(0.5 ~ 1초).

### G1 GC 활성화
- Garbage-First 가비지 수집기는 java9부터 기본 수집기이므로 일반적으로 추가 작업을 수행할 필요가 없다. `-XX:+UseG1GC` 옵션을 통해 명시적으로 활성화할 수 있다.

---

## Basic Concepts
- G1은 이름을 보면 짐작할 수 있듯, 쓰레기로 가득찬 heap 영역을 집중적으로 수집한다.
- G1은 큰 메모리를 가진 멀티 프로세서 시스템에서 사용하기 위해 개발된 GC이다.
- GC 일시 정지 시간을 최소화하면서, 따로 설정을 하지 않아도 가능한 한 처리율(throughput)도 확보하는 것이 G1 GC의 목표이다.
- G1은 실시간(real time) GC가 아니다. 일시 정지 시간을 최소화하긴 하지만 완전히 없애지는 못한다.
- G1은 통계를 계산해가면서 GC 작업량을 조절한다.

### Heap Layout
이전 GC와 비교했을 때, 고정된 메모리 크기로 각 Generation을 구분했던 것과 달리 힙 영역을 동일한 크기의 Region으로 나누어 관리한다.
JVM 힙은 2048개의 region으로 나뉠 수 있으며, 해당 옵션을 통해 1MB ~ 32MB 사이로 지정될 수 있다.(`-XX:G1HeapResionSize`)

G1은 힙 내의 하나 이상의 Region을 단일 Region으로 객체를 복사하는데 이 때 메모리를 압축/해제 시킨다. 다중 프로세서에서 병렬 작동을 통해 STW 시간은 줄리고 처리량은 증가시킨다.

G1은 영역의 참조를 관리할 목적으로 remember set를 만들어 사용한다. remember set은 total heap의 5% 미만 크기.

새롭게 정의된 Humongous, Available/Unused 영역이 존재한다.
- Humongous : Region 크기의 50%를 초과하는 큰 객체를 저장하기 위한 공간. 이 Region에서는 GC가 최적으로 동작하지 않는다.
- Available/Unused : 아직 사용되지 않은 Region을 의미한다.

![heap_layout](https://velog.velcdn.com/images/ahnjs/post/4cf72f5d-dcfb-40bb-a42f-7c2e0823afef/image.png)
- 빨간색 : Young 영역(Eden)
- 빨간색 + S : Young 영역(Survivor)
- 파란색 : Old 영역
- 파란색 + H : Old 영역, 여러 개의 region이 필요한 사이즈가 큰 객체(humongous object)

이러한 영역은 다른 컬렉터의 각 연속 공간과 동일한 기능을 제공하지만, G1에서는 이러한 영역이 일반적으로 메모리에서 비연속 패턴으로 배치된다는 차이점이 있다.

**작동 방식**
- 비어 있는 영역에만 새로운 객체가 들어간다.
- 쓰레기가 쌓여 꽉 찬 영역을 우선적으로 청소한다.
- 꽉 찬 영역에서 라이브 객체를 다른 영역으로 옮기고, 꽉 찬 영역은 깨끗하게 비운다.
- 이렇게 옮기는 과정이 조각 모음의 역할도 한다.

G1 GC는 일시 정지 시간을 줄이기 위해 병렬로 GC 작업을 한다. 각각의 스레드가 자신만의 영역을 잡고 작업하는 방식.

### Garbage Collection Cycle
G1은 두 페이즈를 번갈아 가며 GC 작업을 한다.

![garbage_collection_cycle](https://velog.velcdn.com/images/ahnjs/post/85636301-07c4-4273-81f8-24dad54969da/image.png)
사이클중 모든 원은 stop-the-world가 발생한 것을 나타낸 것이고, 원의 크기에 따라 소요 시간이 달라진다고 보면 된다.
- 파란 원은 Minor GC(=Young GC, Evacuation Phuse)가 진행함에따라 stop-the-world가 발생한 것이다.
- 주황 원은 Major GC(=Old GC, ConcurrentCycle)이 진행하면서 객체를 마킹 및 기타 과정을 하기 위해 stop-the-world가 발생한 것이다.
- 빨간 원은 Mixed GC를 진행함에 따라 stop-the-world가 발생한 것이다.

#### Young-only Phase
Young Only (Garbage Collection) Phase는 Minor GC만 수행하다가 `-XX:InitiatingHeapOccupancyPercent`(Old Generation 비율)에 지정된 값을 초과하는 순간 Major GC가 수행된다. 그림에서 Old gen occupancy exceeds threshold 부분
Major GC의 첫 단계는 Initial Mark이며 Minor GC와 동시에 수행되며 둘 다 STW를 수반하므로 다른 파란 원보다 크기가 크다. 그 이후에 애플리케이션 스레드, Minor GC, Concurrent Mark가 동시에 수행되는데 Remark가 수행되는 순간 다른 작업은 멈추게 된다. 그래서 Remark에 해당하는 주황색이 원이 큰 것을 알 수 있다. 그 이후에 자잘하게 Minor GC가 수행되다가 Major GC의 Cleanup이 발생한다.

- old gen의 점유율이 threshold 값을 넘어서면 Young-only 페이즈로 전환된다.
- Concurrent Start 단계: 도달할 수 있는 객체들에 마킹 작업을 한다.
  - 이 단계에서 Young GC를 수행하면서 동시에 Marking Phase가 진행된다.
  - Concurrent Mark는 Old Region을 GC하기 위해 현재 도달할 수 있는 live 객체(Object)를 결정한다.
  - Concurrent Mark가 진행되는 도중에, Young GC가 동작할 수 있으며 이로 인해 방해받을 수 있다.
  - Marking은 Remark 와 Cleanup 단계에서 STW를 발생시킬 수 있습니다.
- Remark: 마킹을 끝내고, 쓰레기 영역을 해지한다.
  - 이 일시중지는 Marking을 마무리하고 Reference 처리 및 Class Unloading을 수행하여 Empty Region을 회수하고 내부 데이터 구조를 정리한다.
  - Remark와 Cleanup 사이에서 G1은 Old Region에서 여유 공간을 회수 할 수 있도록 정보를 계산한다.
  - 이 계산은 Cleanup 단계에서의 STW에서 마무리 된다.
- Cleanup: Space-Reclamation 페이즈로 들어가야 할지 말지를 판단한다.
  - 이 단계는 Region 회수가 실제로 진행될지 결정합니다.
  - 만약 공간 재확보(Space-Reclamation) 단계가 온다면, young-only 단계는 1회의 Mixed-GC만 진행하고 완료된다.

#### Space-reclamation Phase
Young Only Phase가 끝나고 Space Reclamation(공간 회수) Phase가 시작된다. 해당 Phase에서는 Mixed GC가 수행되는데 Mark 단계가 없어서 STW 빈도가 Young Only Phase에 비해 줄어든 것을 알 수 있다. Space Reclamation Phase가 끝나면 다시 Young Only Phase로 돌아가서 Minor GC를 수행한다.

- Space-Reclamation: young/old 가리지 않고 라이브 객체를 적절한 곳으로 대피시킨다(Evacuation). 작업 효율이 떨어지게 되면 이 페이즈는 끝나고, 다시 Young-only 페이즈로 전환된다.
  - 이 단계는 Young Generation Region 말고 Old Region의 Live 객체도 비우는 여러번의 Mixed-GC로 구성되어 있습니다.
공간 재 확보 단계는 G1이 더이상 Old Region을 효율적으로 줄일 수 없겠다고 판단되면 종료됩니다.

만약 애플리케이션 메모리가 부족한 경우 G1GC는 다른 GC들처럼 Full GC를 수행한다.

---

## Garbage-First Internals
### Determining Initiating Heap Occupancy
IHOP: Initiating Heap Occupancy Percent
마킹 발동 기준이 되는 값으로, old gen 사이즈에 대한 백분율이다.

- Adaptive IHOP
  - G1 통계를 계산하며 최적의 IHOP 값을 찾아내 알아서 설정한다.
  - Adaptive IHOP 기능이 켜져 있을 때 `-XX:InitiatingHeapOccupancyPercent` 옵션을 주면 통계 자료가 충분하지 않은 초기 상태에서 이 옵션 값을 초기값으로 활용한다.
  - `-XX:-G1UseAdaptiveIHOP 옵션으로 Adaptive IHOP` 기능을 끌 수 있다.
    - Adaptive 기능을 끄면 통계를 게산하지 않으므로 `-XX:InitiatingHeapOccupancyPercent`로 지정한 IHOP 값을 계속 쓰게 된다.
  - Adaptive IHOP는 `-XX:G1HeapReservePercent`로 설정된 값 만큼의 버퍼를 제외하고 시작 heap 점유율을 설정한다.

### Marking
- G1 GC는 SATB(Snapshot-At-The-Beginning) 알고리즘을 써서 마킹 작업을 한다.
- SATB는 일시 정지가 일어난 시점 직후의 라이브 객체에만 마킹을 한다. 따라서 마킹하는 도중에 죽은 객체도 라이브 객체로 간주하는 보수적인 특징이 있다. 비효율적일 것 같지만 Remark 단계의 응답 시간(latency)이 다른 GC에 비해 더 빠른 경향이 있다.

### Humongous Objects
- 위의 체스판 모양의 heap 구조에서 파란색 H로 표시된 객체가 커다란 객체(Humongous Object) 이다.
- 한 영역의 절반 이상의 크기를 가진 객체를 말한다.
- 영역의 절반이 기준이므로 -XX:G1HeapRegionSize의 영향을 받는다
  - 기본값을 쓰고 있다면, 알아서 자동으로 결정된다.
  
**커다란 객체는 아무래도 크기가 있다 보니 특별하게 다뤄진다.**
- 연속된 영역을 순차적으로 차지하도록 할당된다.
- 마지막 꼬리 영역에 남는 공간이 생길 수 있는데, 그 잉여 공간은 아깝지만 사용하지 않는다.
  - 즉, 커다란 객체가 회수될 때까지는 잉여 공간을 사용할 수 없다.
- 커다란 객체가 할당되면, G1은 IHOP를 확인하고 IHOP가 초과된 상태라면 즉시 강제로(force) young collection을 시작한다.
  - (집에 커다란 가구가 들어오게 되면 가족들이 열심히 여기저기 치우고 옮기며 공간을 확보하는 모습이 연상된다.)
- 커다란 객체는 Full GC 중에도 옮겨지지 않는데, 이로 인해 조각화가 발생할 수 있다.
  - 그 결과, 공간이 충분한데도 메모리 부족 상태가 발생할 수 있다.
  - 그 결과, Full GC가 느려질 수 있다.

---

## Comparison to Other Collectors
### Parallel GC
- Parallel GC는 old gen의 공간에서만 재확보(reclaim)와 조각 모음(compaction)을 한다.
- G1은 이런 작업을 더 짧은 GC 작업들로 분배하여 수행하여, 전체적인 처리율이 줄어드는 대신 일시 정지 시간을 크게 단축한다.

### CMS
- G1도 CMS처럼 old gen 영역을 동시에(concurrently) 작업한다.
- CMS는 old gen의 조각 모음을 하지 않으므로 Full GC 시간이 길어지는 문제가 있다.

---

https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector.html  
https://johngrib.github.io/wiki/java/gc/g1gc  
https://steady-coding.tistory.com/590