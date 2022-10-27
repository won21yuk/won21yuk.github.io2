---
title: Hadoop Deep Inside - Ch.2 구글 파일 시스템(Google File System) (2)
categories: [Hadoop, Deep Inside]
---

# 3. GFS의 상호작용(Interactions)

GFS의 설계자들은 모든 작업에 대해 마스터의 참여가 최소화 되도록 시스템을 설계했습니다. 이 배경을 바탕으로 어떻게 클라이언트, 마스터 그리고 청크서버 간의 상호작용에서 마스터의 개입이 최소화되는지를 알아보겠습니다.

![GFS-architecture3](/images/GFS-architecture3.jpg "GFS-architecture3")

## 1) Leases and Mutation Order

변화(Mutation)는 쓰기(write) 또는 추가(append) 작업과 같이 청크의 내용이나 메타데이터를 바꾸는 작업이고, 각각의 변화는 모든 청크 복제본에 적용되어야합니다. 이때, 복제본들을 거쳐 일관된 변화 순서를 유지하기 위해서 GFS에서 사용하는 방식을 임대(Lease)라고 합니다.

![primary-secondary](/images/primary-secondary.jpg "primary-secondary")

마스터는 복제본 중 하나에 청크 리스를 부여하는데, 이를 프라이머리(Primary)라고 합니다. 프라이머리는 청크에 대한 모든 변화에 일련의 순서를 지정하는데, 모든 복제본은 변화를 적용할 때 이 순서를 따르게 됩니다. 따라서 전역 변화(Global Mutation) 순서는 마스터가 선택한 리스 허가 순서에 의해서 먼저 정의가 되고, 리스 내에서는 프라이머리에 의해 할당된 일련 번호로 정의됩니다.

이러한 임대 메커니즘은 마스터에서 관리 오버헤드를 최소화하도록 설계되었습니다. 만약 마스터가 프라이머리와의 통신이 끊기게 되더라도 새로운 리스를 다른 복제본에 부여하는 방식으로 대처할 수 있고, 자연스럽게 이전의 리스는 만료되게됩니다.

아래의 그림은 쓰기(write)의 제어흐름을 도식화한 그래프입니다.

![flow](/images/flow.jpg "flow")

1. 클라이언트는 어떤 청크 서버가 현재 청크에 대한 리스를 보유하고 있는지와 다른 복제본의 위치를 마스터에 요청합니다. 아무도 리스를 가지고 있지않다면, 마스터는 하나의 복제본에 리스를 부여합니다.
2. 요청을 받은 마스터는 프라이머리(Primary) 복제본의 ID와 다른(Secondary) 복제본의 위치로 응답합니다. 클라이언트는 이후 변화에 대비해 이 응답을 캐싱합니다.
3. 클라이언트는 데이터를 모든 복제본에 푸시합니다. 각 청크 서버는 데이터가 사용되거나 만료될 때까지 내부 LRU(Least Recently Used) 버퍼 캐시에 데이터를 저장합니다. 데이터 흐름(Data flow)를 제어 흐름(control flow)에서 분리함으로써, 어떤 청크서버가 프라이머리인지와 상관없이 네트워크 토폴로지를 기반으로 값비싼 데이터 흐름(data flow)를 스케줄링함으로써 성능을 향상시킬 수 있습니다.

> **LRU(Least Recently Used)**
: 가장 오래전에 사용된 것은 디스크에 저장하고 메모리에는 가장 최근에 사용된 데이터를 저장함으로써, 디스크 I/O를 줄이고, 데이터베이스 시스템의 성능은 증가하도록 하는 관리 기법이다.
>

> **스케줄링(scheduling)**
: 여러 프로세스가 번갈아가며 사용하는 자원을 어떤 시점에 어떤 프로세스에게 자원을 할당할지 결정하는 것이다.
>

5. 모든 복제본이 데이터 수신을 승인하면, 클라이언트는 프라이머리 복제본에 쓰기 요청을 보내게 됩니다. 이 요청은 모든 복제본에서 이전에 푸시된 데이터를 식별합니다. 프라이머리는 여러 클러이언트에서 수신되는 모든 변화에 대해 일련 번호를 할당합니다. 이 일련 번호 순서로 자신의 로컬 상태에 변화를 적용합니다.
6. 프라이머리는 이후 쓰기 요청을 모든 다른(Secondary) 복제본으로 전달합니다. 각 세컨더리 복제본은 프라이머리 복제본에 의해 할당된 동일한 일련 번호 순서로 변화를 적용합니다.
7. 변화 적용이 끝난 복제본들은 다시 프라이머리에 작업이 완수되었음을 알립니다.
8. 프라이머리는 최종적으로 클라이언트에게 응답합니다.

어플리케이션에 의한 쓰기가 크거나 청크의 경계에 걸치는 경우 GFS 클라이언트 코드는 이를 여러 쓰기 작업으로 분할합니다. 이 과정은 모두 위에서 설명한 제어 흐름을 따르지만, 다른 클라이언트들의 동시 작업으로 인터리빙(Interleaced)되거나 덮어써질 수 있습니다.

> **인터러브(interleave)**
: 컴퓨터 하드디스크의 성능을 높이기 위해 데이터를 서로 인접하지 않게 배열하는 방식. (interleave : 교차로 배치하다)
>

## 2) 데이터 흐름(Data flow)

앞선 내용에서 데이터 흐름과 제어흐름을 분리하였다고 했는데, 이는 네트워크를 효율적으로 사용하기 위함이였습니다. 제어 흐름이 클라이언트에서 프라이머리 그리고 다른(Secondary) 모든 복제본들 순서로 흐르는 동안 데이터는 파이프라인 방식으로 신중하게 선택된 청크 서버의 체인을 따라 선형적으로 푸시 됩니다.

> **데이터 파이프라인(data pipeline)**
: 다양한 데이터 소스에서 원시 데이터를 수집한 다음 분석을 위해 데이터 레이크 또는 데이터 웨어하우스와 같은 데이터 저장소로 이전하는 방법. (생성, 변환, 저장 등)
>

이를 통해 GFS는 각 시스템의 네트워크 대역폭(bandwidth)을 최대한 활용하고, 네트워크 병목 현상과 지연시간이 긴 연결을 방지하고, 모든 데이터를 전송하는 지연 시간을 최소화하여 모든 데이터를 처리합니다.

## 3) Atomic Record Appends

GFS는 레코드 추가(Record append)라고 불리는 원자성 추가 작업(Atomic append operation)을 제공합니다. 기존의 쓰기방식에서는 클라이언트가 데이터를 쓸 오프셋을 지정하지만, 동일한 영역에 대한 동시 쓰기 작업은 직렬화(serializable)할 수 없습니다. 각 영역은 결국 다수의 클라이언트로부터 발생한 데이터 조각들을 가질 뿐입니다.

그러나

레코드 추가 작업은 서로 다른 머신의 다수의 클라이어트가 동시에 동일한 파일에 쓰는 작업을 가능하게 하며, 이로인해 분산 어플리케이션에서 많이 사용됩니다. 따라서 GFS 또한 레코드 추가 작업이 빈번하게 일어납니다.

레코드 추가는 일종의 변화(Mutation)입니다. 따라서 위에서 설명한 제어 흐름(Control flow)을 따르고, 프라이머리에서 약간의 추가적인 로직이 필요할 뿐입니다. 클라이언트는 파일의 마지막 청크의 복제본들에 데이터를 푸시하고, 프라이머리에 요청을 보냅니다. 요청을 받은 프라이머리는 추가(appending) 작업이 청크의 최대 사이즈(64MB)를 넘는지 확인하고, 만약 넘는다면 추가적인 작업을 수행하게 됩니다.

GFS는 모든 복제본이 바이트 단위로 동일할 것을 보장하지 않고 데이터가 원자 단위로 한 번 이상 작성 된다는 것만 보장합니다. 이 속성은 작업이 성공했다는 것을 보고하기위해 데이터가 일부 청크의 모든 복제본에서 동일한 오프셋에 기록되어야한다는 앞선 관찰에 기인합니다.

## 4) 스냅샷(Snapshot)

스냅샷 작업은 진행 중인 변화(Mutation)의  중단을 최소화하면서 파일 또는 디텍토리 트리의 복사본을 거의 즉시 만듭니다. 사용자는 이 스냅샷을 사용하여 대용량 데이터 세트의 브런치(branch) 복사본을 신속하게 생성하거나 나중에 커밋하거나 쉽게 롤백할 수 있는 변경사항을 실험하기 전에 현재 상태를 확인합니다.

스냅샷은 전형적인 copy-on-write 기술을 사용하여 구현되었습니다. 마스터가 스냅샷 요청을 받을 때, 먼저 마스터는 해당 청크들에 대한 리스를 철회시킵니다. 이것은 이후의 쓰기 작업이 마스터에게 다시 리스를 찾도록 만다는 보증(Guarantee)입니다. 또한 마스터가 해당 청크에 새로운 복사본을 만들도록 합니다.

리스가 철회되거나 만료되면 마스터는 해당 작업을 디스크에 로깅합니다. 이 후 원본 파일 또는 디렉토리 트리의 메타데이터를 복제하여 이 로그 레코드를 메모리 내 상태에 적용합니다. 새로 생성된 스냅샷 파일이 원본 파일과 동일한 청크를 가리킵니다.

스냅샷 작업 후 클라이언트가 처음으로 청크에 쓰기를 할때, 마스터에게 현재 리스 홀더를 찾기 위한 요청을 보냅니다. 마스터는 청크에 대한 참조수가 1개보다 크다는 것을 인식합니다. 클라이언트 요청에 대한 응답을 연기하고, 대신 새 청크 핸들를 선택합니다. 이후 현재 본제본이 있는 각 청크서버에 새 청크를 만들도록 요청합니다.

원본이 존재하는 청크서버에 새 청크를 만들어 데이터를 네트워크를 통해서가 아닌 로컬로 복사할 수 있습니다. 이 시점부터 요청 처리는 모든 청크에 대한 요청 처리와 다르지 않습니다. 마스터는 복제본 중 하나에 새 청크에 대한 리스를 부여하고 클라이언트에 응답합니다. 클라이언트는 기존 청크에서 방금 생성된 것을 모르고 청크를 정상적으로 쓸 수 있습니다.

# 4. 마스터의 작동(Master Operation)

![GFS-architecture2](/images/GFS-architecture2.jpg "GFS-architecture2")

마스터는 모든 네임스페이스의 작업을 실행합니다. 또한 시스템 전체에서 청크 복제본을 관리합니다. 배치에 대한 결정을 내리고, 새로운 청크를 생성하며, 복제본을 생성하고, 청크를 완전히 복제하고, 모든 청크서버에서 로드의 균형을 맞추고, 사용되지 않는 스토리지를 회수하기 위한 다양한 시스템 전반의 작업을 조정합니다.

## 1) 네임스페이스 관리 및 Locking(Namespace Management and Locking)

많은 마스터 작업이 오래 걸릴 수 있기 때문에, 실행중인 다른 마스터 작업을 지연시키지 않기 위해서 여러 작업을 활성화하고 네임스페이스의 영역에 대한 잠금(lock)을 사용하여 올바른 직렬화(serialization)을 보장합니다.

> **잠금(lock)**
: 트랜잭션 처리의 순차성을 보장하기 위한 방법
>

> **트랜잭션(transaction)**
: DB의 나누어지지 않는 최소한의 처리 단위
>

> **직렬화(serialization)**
: 데이터를 네트워크로 전송하기 위해 구조화된 객체를 바이트 스트림으로 전환하는 과정.   \
⇒ CSV, XML, JSON 등의 형식은 알아보기 쉽다는 장점이 있지만, 데이터 구조상 파싱하는 시간이 오래 걸린다. 이를 binary로 변경하면 파싱 시간이 짧아지기 때문에 성능상 이득을 볼 수 있다.
>

많은 전통적인 파일 시스템과 달리, GFS는 디렉터리의 모든 파일을 나열하는 디렉토리별 데이터 구조를 가지고 있지 않습니다. 또한 동일한 파일이나 디렉터리에 대한 별칭(유닉스에서의 심볼릭 링크)도 지원하지 않습니다.

대신 GFS는 전체 경로 이름을 메타데이터에 매핑하는 룩업 테이블로 네임스페이스를 논리적으로 나타냅니다. 이 테이블은 prefix compression을 통해 메모리에 효율적으로 나타낼 수 있습니다. 네임스페이스 트리의 각 노드(절대 파일 이름 또는 절대 디렉토리 이름)에는 관련 읽기-쓰기 잠금(read-wrtie lock)이 있습니다.

각 마스터 작업은 실행되기 전에 일련의 잠금을 획득합니다. 아래의 그림과 그에 대한 설명을 간단히 하겠습니다.

![lock](/images/lock.jpg "lock")

- 스냅샷 작업은 `/home` 및 `/save`에 대한 read lock과 `/home/user` 및 `/save/user`에 대한 write lock을 획득한다.
- 파일 생성은 `/home` 및 `/home/user`에 대한 read lock와 `/home/user/foo`에 대한 write lock을 획득한다.
- 위 두 작업은 `/home/user`에서 충돌하는 lock을 얻으려고 하기 때문에 올바르게 직렬화이 된다.

네임스페이스는 많은 노드를 가질 수 있기 때문에 read-write 개체는 느리게 할당되고 사용되지 않으면 삭제됩니다. 잠금(lock)은 데드락(deadlock)을 방지하기 위해 일관된 전체 순서로 획득됩니다. 잠금은 먼저 네임스페이스 트리의 레벨별로 정렬되고 사전순(lexicographically)으로 동일한 레벨 내에 배치됩니다.

> **교착상태(deadlock)**
: 프로세스가 자원을 얻지 못해 다음 처리를 하지 못하는 상태로, 시스템 적으로 한정된 자원을 여러 곳에서 사용하려고 할 때 발생합니다.
>

> **사전식 나열(lexicographically)**
: 말그대로 사전에 배열된 순서대로 나열하는 방법. 영어는 알파벳 순서(a, b, c, …)대로 숫자는 작은숫자부터 나열한다.
ex. abcde-abced-abdce-abdec-abecd-abedc-...edcba
>

## 2) 복제본 배치(Replica Placement)

GFS 클러스터는 일반적으로 수백 개의 청크 서버가 많은 랙(rack)에 분산되어있는 구조로 되어있습니다.

> **랙(rack)**
: 서버 또는 네트워크 장비들을 넣어두는 철체 프레임. 데이터센터나 서버 룸과 같이 다수의 서버/네트워크 장비들을 두는 곳에서 사용한다.
>

이러한 청크 서버는 동일한 랙 또는 다른 랙에서 수백 대의 클라이언트에서 차례로 접근 가능합니다. 서로 다른 랙에 있는 두 시스템 간의 통신은 하나 이상의 네트워크 스위치를 통과할 수 있습니다.

청크 복제본 배치 정책은 데이터 안정성과 가용성을 극대화하고 네트워크 대역폭 활용도를 극대화하는 두 가지 용도로 사용됩니다. 두 가지 모두 복제본을 여러 시스템에 분산시키는 것만으로는 충분하지 않으므로 디스크나 시스템 장애에 대비하고 각 시스템의 네트워크 대역폭을 충분히 활용할 수 있습니다.

또한, 청크 복제본을 랙에 분산 시킴으로서, 청크 복제본을 랙에 분산시켜 전체 랙이 손상되거나 오프라인 상태일 경우에도 청크의 일부 복제본을 사용 가능한 상태로 유지할 수 있게 됩니다.

## 3) 생성, 재복제, 재조정(Creation, Re-replication, Rebalancing)

청크 복제본은 청크 생성, 재복제, 재조정이라는 3가지 이유로 인해 생성됩니다. 마스터는 청크를 생성할 때, 처음에 비어있는 복제본을 배치할 위치를 선택합니다. 이때 몇가지를 고려하게 되는데 이는 아래와 같습니다.

- 평균 이하의 디스크 공간 활용률을 가진 청크 서버에 새 복제본을 배치하려고 한다.
- 각 청크 서버의 “최근” 생성 수를 제한하려고 한다.
- 청크의 복제본을 랙에 분산시키려 한다.

마스터는 기존의 유효한 복제본에서 직접 청크 데이터를 복사하도록 청크 서버에 지시합니다. 그리고 가장 높은 우선순위 청크를 선택하고 이를 복제합니다.

복제 트래픽이 클라이언트 트래픽을 넘어서지 않도록 마스터는 클러스터와 각 청크 서버 모두에 대해 활성 복제 작업 수를 제한합니다. 또한 각 청크 서버는 소스 청크 서버에 대한 읽기 요청을 제한하여 각 복제 작업에 사용되는 대역폭의 양을 제한합니다.

마지막으로 마스터는 주기적으로 복제본을 재조정합니다. 현재 복제본 분포를 검사하고 더 나은 디스크 공간과 부하 분산(load balancing)을 위해 복제본을 이동시킵니다. 또한 이 프로세스를 통해 마스터는 즉시 새로운 청크와 함께 제공되는 대량의 쓰기 트래픽으로 서버를 가득 채우는 대신, 점차 새로운 청크 서보로 채우게 됩니다. 그리고 마스터는 삭제할 기존 복제본을 선택해야 합니다. 일반적으로 디스크 공간 사용을 균등화 하기 위해서, 사용가능한 공간이 평균 미만인 청크 서버에서 이를 제거하는 것을 선호합니다.

## 4) Garbage Collection

파일을 삭제한 후 GFS는 사용 가능한 물리적 스토리지를 즉시 회수하지 않습니다. 이 접근 방식이 시스템을 훨씬 더 단순하고 신뢰할 수 있게 만든다는 것을 발견했기 때문입니다.

## 5) 오래된 복제본 탐지(Stale Replica Detection)

청크 서버가 다운된 동안 청크에 대한 변화가 누락되거나 청크 서버에 장애가 발생하게 된다면, 청크 복제본은 오래된(stale) 복제본이 됩니다. 이는 복제본이 쓸모가 없어졌다는 것을 의미합니다. 그래서 마스터는 각 청크에 대해 최신 복제본과 오래된 복제본을 구분하기 위해 청크 버전 번호(chunk version number)를 유지합니다.

마스터는 청크에 대해 새로운 리스를 부여할 때마다 청크 버전 번호를 증가시키고 최신 복제본에 알립니다. 마스터 및 복제본은 모두 새 버전 번호를 영구적인 상태로 기록합니다. 마약 마스터가 레코드 버전 번호보다 큰 버전 번호를 볼 경우, 마스터는 리스를 허용할 때 실패한 것으로 인지하고 상위 버전을 최신버전으로 간주합니다. 그리고 마스터는 일반 가비지 컬렉션에서 오래된 복제본을 제거합니다.

# 5. 내고장성과 진단(Fault Tolerance And Diagnosis)

GFS 뿐만아니라 어떠한 시스템을 설계할 때 가장 힘든 일 중 하나는 잦은 컴포넌트들의 장애를 처리하는 일입니다. GFS는 앞선 관찰과 설계 가정에서 이러한 문제는 예외라기 보단, 일반적인 것으로 간주했습니다. 실제로 수많은 구성요소의 품질과 양은 이런 문제들을 일반적인 것으로 만들기 마련이죠.

이처럼 우리는 기계를 완전히 신뢰할 수 없고, 따라서 디스크도 완전히 신뢰 할 수 없습니다. 그러나 컴포넌트들의 오류는 시스템을 사용할 수 없게하거나 데이터를 손상시키는 등 치명적인 부작용을 초래합니다. 그렇기 때문에 GFS는 이러한 문제를 해결하고 진단하기 위해 시스템 내에 여러 장치들을 마련했습니다.

## 1) 고가용성(High Avaulability)

GFS 클러스터에 속ㅎ산 수백 대의 서버 중 일부는 사용 불가능할 수 있습니다. 이 경우 단순하면서 효과적인 두가지 전략으로 전체 시스템의 가용성을 향상시킬 수 있습니다.

첫째는 빠른 복구(Fast Recovery)입니다. GFS는 마스터 서버와 청크 서버 모두 상태를 복원하고 어떻게 종료되었는지에 관계없이 몇 초만에 시작하도록 설계되어 있습니다. 이는 정상종료와 비정상 종료를 구분하지 않는 다는 것을 의미하고, 프로세스를 중단하는 것 만으로도 서버가 종료된다는 것을 의미합니다.

둘째는 복제입니다. 복제는 또 두 가지 유형으로 나뉘는데 하나는 청크 복제 또 하나는 마스터 복제입니다.

앞서 설명했듯이 청크는 서로 다른 랙의 여러 청크 서버에 복제됩니다. 사용자는 파일 네임스페이스의 다른 부분에 대해 다른 복제 수준(복제본 숫자를 의미. 디폴트는 3개)를 지정할 수 있습니다. 마스터는 청크 서버가 오프라인으로 전환될 때, 각 청크를 완전히 복제하거나 체크섬 확인을 통해 손상된 복제본을 탐지합니다. 필요한 경우 기존 복제본을 복제하기도 합니다.

> **체크섬(checksum)**
: 중복 검사의 한 형태로, 오류 정정을 통해, 공간(전자 통신)이나 시간(기억 장치) 속에서 송신된 자료의 무결성을 보호하는 단순한 방법
>

안정성을 위해 마스터의 상태 또한 복제됩니다. 해당 작업 로그 및 체크포인트는 여러 시스템에 복제 됩니다. 상태 변환은 로그 레코드가 로컬 및 모든 마스터 복제본에서 디스크로 플러시된 후에만 커밋된 것으로 간주합니다. 단순성을 위해 하느이 마스터 프로세스가 모든 변형은 물론 내부적으로 시스템을 변경하는 가비지 컬렉션과 같은 백그라운드 활동을 담당합니다.

마스터에 장애가 발생하면 거의 즉시 재시작이 가능합니다. 머신 또는 디스크에서 장애가 발생할 때, GFS 외부의 모니터링 인프라는 이전 복제된 작업 로그를 사용하여 다른 머신에서 새 마스터 프로세스를 시작합니다. 클러이언트는 마스터의 표준 이름만 사용합니다. 이 이름은 마스터가 다른 시스템을 ㅗ재배치된 경우 변경할 수 있는 DNS 별칭입니다.

또한 쉐도우 마스터는 기본 마스터가 중단된 경우에도 파일 시스템에 대한 읽기 전용 접근 권한을 제공합니다. 일반적으로 조금 딜레이 시간있다는 점(그래봤자 1초 언저리)에서 거울이 아닌 그림자로 불리게 됐습니다. 이러한 기능은 현재 변경되지 않은 파일이나 다소 오래된 결과를 얻어도 상관하지 않는 어플리케이션에 대해 읽기 가용성을 향상시킵니다. 실제로 파일 내용은 청크서버에서 읽기 때문에 어플리케이션에서 오래된(stale) 파일 내용을 확인하지 않습니다.

섀도 마스터는 증가하는 작업 로그의 복제본을 읽고 프라이머리와 동일한 변경 순서를 데이터 구조에 적용합니다. 프라이머리 서버와 마찬가지로 시작 시(이후에는 드물게) 청크 서버를 폴링하여 청크 복제본을 찾고 상태를 모니터링하기 위해 자주 핸드셰이크 메시지를 교환합니다. 복제본 작성 및 삭제 결정에 따른 복제본 위치 업데이트에 대해서만 프라이머리 마스터에 따라 달라집니다.

## 2) 데이터 통합(Data integrity)

각 청크서버는 체크섬을 사용하여 저장된 데이터의 손상을 감지합니다. GFS클러스터 에는 수백개의 머신들에 존재하는 수천개의 디스크가 있습니다. 읽기 및 쓰기 과정 모두에서 데이터 손상 또는 손실을 일으키는 디스크 장애가 지속적으로 발생한다는 것 입니다.

다른 청크 복제본을 사용하여 손상을 복구할수도 있지만, 청크 서버간 복제본을 비교하여 손상을 탐지하는 것은 다분히 비현실적입니다. 따라서 체크섭 방식을 도입합니다.

하나의 청크는 64KM블록으로 분할 됩니다. 그리고 각각에 해당하는 32비트의 체크섬이 있습니다. 다른 메타데이터와 마찬가지로 체크섬은 메모리에 보관되며 사용자 데이터와 별도로 로깅되어 영구적으로 저장됩니다.

읽기 작업의 경우 청크서버는 데이터를 요청자에게 반환하기 전 읽기 범위와 겹치는 데이터 블록의 체크섬을 확인합니다. 이에 따라 청크서버는 손상된 것을 다른 머신에 전달하지 않습니다. 만약 블록이 기록된 체크섬과 일치하지 않으면, 청크서브는 요청자에게 오류를 반환하고, 마스터에게 체크섬 불일치를 보고합니다.

이에 요청자는 다른 복제본에서 읽는 반면, 마스터는 다른 복제본에서 청크를 복제합니다. 유효한 새 복제본이 있으면 마스터는 불일치를 보고한 청크서버의 복제본을 삭제하도록 지시합니다.

## 3) 진단 도구(Diagnostic Tools)

GFS서버는 많은 중요한 이벤트와 모든 RPC 요청 및 응답을 기록하는 진단 로그를 생성합니다. RPC로그에는 읽거나 쓰는 파일 데이터를 제외하고 와이어로 전송되는 정확한 요청과 응답이 포함됩니다. 요청을 응답과 일치시키고 서로 다른 장치에서 RPC 레코드를 취합함으로써 전체 상호 작용 기록을 재고성하여 문제를 진단할 수 있습니다. 로그는 부하테스트 및 성능 분석을 위한 추적 기능도 있습니다.

# 6. 정리

- GFS는 load balance나 fault tolerance을 위해 데이터를 투명하게 이동할 수 있는 위치 독립적인 네임스페이스를 제공한다.
- GFS는 총 성능 및 fault tolerance를 제공하기 위해 파일 데이터를 스토리지 서버에 분산한다.
- GFS는 파일 시스템 인터페이스 아래에 캐싱을 제공하지 않는다. 타겟 워크로드는 대규모 데이터 셋을 통해 스트리밍하거나 데이터 셋 내에서 무작위로 검색하여 매번 소량의 데이터를 읽기 때문에 단일 애플리케이션 실행 시 재사용이 거의 없다.
- GFS는 설계를 단순화(simplify)하고, 신뢰도(reliability)를 높이며, 유연성(flexibility)을 얻기 위해 중앙 접근 방식을 선택했다.
- 마스터 상태를 작게 유지하고 다른 시스템에서 완전히 복제하여 fault tolerance를 해결한다.
- 확장성(scalability)와 높은 가용성(availability)(읽기용)은 현재 섀도(shadow) 마스터 메커니즘에 의해 제공된다.