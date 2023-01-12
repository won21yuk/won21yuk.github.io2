---
title: Kubernetes - 컨테이너 오케스트레이션(Container Orchestration)
categories: [Kubernetes, k8s Notion]
---

# 컨테이너 이전의 시대

서버 관리는 언제나 어려운 과제 중에 하나였습니다. 어떤 장애가 발생할지 어떻게 해야 서버가 문제없지 잘 돌아갈 수 있을지 그리고 이것들이 도대체 언제 일어날지 예측불가능한 변수들이 워낙 많았기 때문입니다. 그래서 서버 관리자들은 항상 더 편하게 그리고 더 효율적이게 서버를 운영할 수 있는 방안에 대한 욕구가 많았습니다.

## 문서화

서버를 관리하기 위해 했던 가장 원초적인 노력 중 하나는 바로 문서화였습니다. 프로그램을 설치하는 방법을 하나하나 캡쳐하고 글로 적어 메뉴얼을 작성한 것입니다. 그러나 아무리 잘 만든 문서라도 문제는 여전히 존재했습니다. 가령 OS나 다른 소프트웨어 버전과의 호환문제 같이 배포하는 환경의 차이로 인해 업데이트가 안되거나 설치 자체가 안되는 경우들이 빈번했다는 것입니다.

## IaC

![container-orchestration0](/images/container-orchestration0.png)

이런 문서화에 대한 불완전성으로 인해 등장한 것이 코드로 인프라를 관리하는 방법인 IaC입니다.  문서화를 캡쳐를 뜨거나 글로 작성하는 대신 코드로 하기 시작하였다고 이해하면 정확합니다. 사용자가 직접 서버에 들어가서 다양한 커맨드를 실행시키는 것이는 것이나라 Terraform, Ansible, Puppet, Chef, Saltstack 등과 같은 IaC 툴들이 작성된 스크립트 대로 명령어를 날리는 방식으로 바뀐 것이죠.

코드로 서버를 관리하기 시작하면서 얻는 이점은 뚜렷했습니다. 코드로 작성된 대로 인프라가 형성되기 때문에 구성을 변경할 때마다 코드를 수정해줘야했고 이는 서버의 구성관리가 훨씬 더 유기적으로 진행된다는 것을 의미합니다. 또한 스크립트가 작성되어 있기 때문에 이 스크립트를 배포하면 매번 동일한 환경을 구성할 수도 있습니다.

그러나 프로그램 활용법 자체를 새로이 익혀야했기 때문에 진입장벽이 존재했고 서버가 복잡해질 수록 코드가 지나치게 복잡해져서 프로그램의 효용성이 떨어지는 문제점이 존재했습니다.

## 가상 머신

가상머신을 활용하여 서버를 관리하는 방법도 있었습니다. 서버 하나에 가상머신을 여러개 띄우고 가상머신 마다 하나의 어플리케이션만 사용하면 어플케이션간 충돌 위험이 크게 감소했습니다. 또한 서버의 리소스를 관리하고 효율적으로 사용하는데 도움이 됐습니다. 가상머신의 스냅샷 기능을 통해 이미지를 생성하여 같은 가상머신을 여러 서버에서 만들어낼수도 있었습니다.

그런데 가성머신은 태생적으로 무겁고 느렸기 때문에 보통의 운영 환경에서 사용하기에는 다소 제한이 있었습니다.  그리고 특정 벤더(Virtual Box, VMware 등)에 의존성이 강해 점차 인기를 얻고 있던 클라우드 환경과는 맞지않는 부분들이 발생하기도 했습니다.

# 도커의 등장 그리고 컨테이너의 시대

![container-orchestration1](/images/container-orchestration1.png)

도커는 사용법도 쉬웠고 빨랐습니다. 특히나 가상머신과 비교했을때 컨테이너의 생성이 훨씬 쉽고 빨랐으며 리소스의 사용도 효율적이였습니다. 그러면서도 가상머신이 가지고 있는 스냅샷과 같은 이미지 기능을 지원하여 컨테이너를 배포하고 롤백하는 것도 쉬웠습니다.

이뿐만 아닙니다. 언어나 프레임워크에 상관없이 어플리케이션을 동일한 방식으로 관리할 수 있었고 클라우드환경이든 내 로컬 환경이든 구애받지않고 일관성있는 환경을 구축할 수 있었습니다.

이제 도커는 인프라의 세계를 컨테이너 시대로 전환시켰습니다.

그렇게 모든것이 도커화 되었고,

![container-orchestration2](/images/container-orchestration2.png)

도커를 중심으로 개발 프로세스가 정형화 되었습니다.

![container-orchestration3](/images/container-orchestration3.png)

# 컨테이너의 시대, 그 이후

인프라의 세상은 모든걸 도커 컨테이너로 만드는 시대를 맞이했습니다. 그런데 컨테이너를 사용하게 되면서 여러 문제점도 등장하기 시작했습니다. 모든 것을 컨테이너로 만들다보니 관리해야하는 컨테이너의 수가 기하급수적으로 늘어나기 시작한 것이죠.

## 배포의 문제

![container-orchestration4](/images/container-orchestration4.png)

도커 컨테이너를 실행하기 위해서는 모든 서버에 직접 ssh 접속해서 docker run 이라는 도커 커맨드를 직접 입력해야합니다. 3개의 서버가 있다면 세번 해줘야하고 백개가 있으면 백번을 해줘야하는 겁니다. 이는 버전을 관리하는 부분에 있어서도 마찬가지였습니다. 즉 버전을 업그레이드하거나 롤백해야할 때도 서버에 하나씩 접속해서 처리해야 했다는 것입니다.

또한 새로운 컨테이너를 실행하려면 상대적으로 리소스에 여유가 있는 서버에서 실행하는 것이 바람직합니다. 그런데 이를 확인하기 위한 모니터링 툴이 없으면 유휴 서버를 일일이 찾아내야하는 어려움이 존재했습니다.

## 서비스 검색의 문제

![container-orchestration5](/images/container-orchestration5.png)

서비스 검색은 클라이언트가 어떤 웹서버에 접근해야할지 검색하는 것을 의미합니다. 이 서비스 검색 과정에서 프록시 서버가 로드밸런서를 바라보게하고 그 뒤에 여러 웹서버를 두어 부하를 분산시킬 수 있습니다.

그러나 마이크로 서비스가 유행하면서 내부 서비스간 통신이 늘어나기 시작했습다. 이는 그만큼 많은 로드밸런서가 서버를 관리해야하는 상황에 직면했다는 것을 의미합니다. 또한 웹서버의 ip들이 바뀌면 매번 로드밸런서의 설정을 변경해줘야하는 등 업무량도 비례해서 증가했습니다.

## 서비스 노출의 문제

![container-orchestration6](/images/container-orchestration6.png)

서비스 노출은 내부에 구축한 서비스를 외부에 어떻게 노출할 것인가에 대한 것입니다. 제일 쉬운 방법은 퍼블릭 영역에 nginx와 같은 프록시 서버를 두어 들어오는 호스트에 따라 프라이빗에 있는 컨테이너와 연결해주는 것입니다.

그런데 내부적으로 컨테이너로 서버를 하나 띄울때마다 nginx의 설정을 매번 건들여줘야합니다. 즉, 특정 호스트를 받을 때 해당 컨테이너로 연결시켜줘야하기 때문에 이는 업무에 있어서 부담으로 작용할 수 있습니다.

## 서비스 장애, 부하 모니터링의 문제

![container-orchestration7](/images/container-orchestration7.png)

컨테이너로 서버를 운영하다가 위의 그림처럼 5개의 컨테이너가 무슨이유에선지 다운되었다고 해봅시다. 그러면 서버 관리자는 다운된 컨테이너가 있는 서버에 접속을해서 무슨 이유인지 파악을하고 다시 컨테이너를 띄우는 작업을 일일이 수행해야합니다.

그리고 app2 컨테이너가 있는 서버에 요청이 많이 들어와서 부하가 생긴 상황도 한번 생각해봅시다. 서버가 완전히 죽은 건 아닌데 응답속도가 현저히 느려지는 상황이 발생한 것입니다. 이러면 서버를 늘려서 부하를 분산시켜줘야하는데 이게 사실 언제 부하가 생길 것이라고 예측하기도 어렵고 모니터링 툴이 존재하지 않는다면 매번 확인하는 절차가 필요하기 때문에 상당히 업무에 부담이됩니다.

# 컨테이너 오케스트레이션의 등장

컨테이너는 원래 프로세스나 어플리케이션을 격리하기 위해 만들어졌습니다. 그래서 개별 컨테이너를 만들고 서버에 띄우는 것은 쉬운 일입니다. 하지만, 컨테이너의 수가 많아지게 되면 관리와 운영에 어려움이 따르게 됩니다. 그래서 서버 관리자들은 컨테이너라는 기술 자체는 좋으나 점점 더 많아지는 컨테이너를 관리하기 위한 기술이 필요했습니다.

이 수많은 컨테이너를 관리하기 위해서는 각각 독립적으로 배치된 컨테이너를 연결하고 관리하고 확장하면서 이 요소들 전체가 하나로 실행되록 할 수 있어야 합니다. 그리고 이렇게 다수의 컨테이너 실행을 관리하고 조율하는 것을 컨테이너 오케스트레이션이라고 부릅니다.

![container-orchestration8](/images/container-orchestration8.png)

컨테이너 오케스트레이션을 한 문장으로 정의하자면 **복잡한 컨테이너 환경을 효과적으로 관리하기 위한 도구**라고 할 수 있습니다. 좀 더 직설적으로 이야기하면 서버관리자들이 해야하는 일들을 대신해줄 도구입니다.

컨테이너 오케스트레이션을 통해 컨테이너의 생성과 소멸, 자동 배치 및 복제, 장애 복구, 스케줄링, 로드 밸런싱, 클러스터링 등 컨테이너로 어플리케이션을 구성하는 모든 과정을 관리할 수 있습니다. 상황이나 조건이 변할 때마다 코딩으로 대응하지 않고 원하는 상태로 인프라를 구축할 수 있습니다.

다만 컨테이너 오케스트레이션이 되기 위해서는 크게 6가지의 특징을 가지고 있어야 합니다.

## 클러스터

![container-orchestration9](/images/container-orchestration9.png)

컨테이너 오케스트레이션은 클러스터 단위로 서버들을 추상화해서 관리해야합니다. 다만, 클러스터에 속한 노드마다 하나하나 ssh 접속을 하여 관리하는 건 클러스터의 규모가 커질수록 비현실적입니다. 그래서 웹서버 앞에 프록시 서버를 두듯이 클러스터 앞에 마스터 노드를 두어 마스터 노드가 클러스터에 직접 명령을 내릴 수 있는 형태로 구성합니다. 이로써 서버 관리자는 마스터 노드를 관리하는 것 만으로도 전체 클러스터를 관리할 수 있어야합니다.

이외에도 클러스터 내에서는 가상 네트워킹 등을 통해 노드간에 네트워킹이 원할해야하며, 노드의 개수가 수천 수만개가 되더라도 마스터가 클러스터를 감당할 수 있도록 설계되어야 합니다.

## 상태 관리

![container-orchestration10](/images/container-orchestration10.png)

원하는 상태를 설정하는 것 만으로 별도의 명령 없이 해당 상태로 변경 되어야합니다. 가령 replicas가 2에서 3으로 변경이 됐다면 자동으로 app1 이미지를 활용하서 컨테이너를 3개 띄워야합니다. 나아가 3개의 컨테이너 중 하나가 다운되면 그 컨테이너를 죽이고 새로운 컨테이너를 띄우는 것으로 전체 replicas를 3으로 유지할 수 있어야합니다.

## 배포 관리

![container-orchestration11](/images/container-orchestration11.png)

추가 컨테이너를 실행시키고자 할때, 서버 별 리소스를 체크해서 상대적으로 여유있는 서버에 컨테이너를 배치시킬 수 있어야합니다. 가령 위의 그림처럼 3개의 서버가 있고 APP1 이미지로 만든 컨테이너를 추가하고자하면 3번째 서버에 컨테이너가 배치되어야 합니다. 나아가 여유있는 서버가 없는 경우에는 자동으로 새로운 서버를 하나 추가시켜서 컨테이너를 띄울 수 있도록 스케쥴링 해줄 수 있는 기능이 필요합니다.

## 배포 버전관리

![container-orchestration12](/images/container-orchestration12.png)

클러스터의 모든 노드들에 대한 배포 버전관리를 중앙에서 일괄적으로 롤백/롤아웃이 가능해야합니다. 여기서 롤아웃은 버전을 이전과 다르게 변경하는 것을 의미하고 롤백은 롤아웃 이전 상태로 다시 되돌리는 것을 의미합니다.

## 서비스 등록 및 조회

![container-orchestration13](/images/container-orchestration13.png)

새로 개발된 서비스의 IP나 기존 서비스의 IP가 변경된 경우 자동으로  프록시서버는 이 설정을 반영할 수 있어야합니다. 새로운 서비스가 추가되면 자연스럽게 저장소에 있는 구성파일이 수정되고 저장소의 구성파일을 바라보고 있던 프록시 서버는 이를 인지하면 자신의 설정을 변경하고 프로세스를 재시작하는 것으로 변경사항을 자동으로 반영해야합니다.

## 볼룸 스토리지

![container-orchestration14](/images/container-orchestration14.png)

가령 위의 그림처럼 각 노드에 필요한 볼륨을 마운트해야할 수 있습니다. 컨테이너 오케스트레이션은 이러한 노드별 볼륨을 추상적인 레벨로 관리할 수 있어야합니다.

# 컨테이너 오케스트레이션 구현체 그리고 쿠버네티스

![container-orchestration15](/images/container-orchestration15.png)

컨테이너 오케스트레이션 구현체는 대표적으로 Docker의 Swarm, Apache의 MESOS, Hashicorp의 Nomad, 구글이 만들어 오픈소스로 공개한 Kubernetes가 있습니다. 이외에도 컨테이너 시대가 도래하면서 등장한 수많은 컨테이너 오케스트레이션 도구들이 있습니다.

그런데 불과 몇년 지나지않아 독보적으로 치고나가는 것으로도 모잘라 현 시점에서는 이미 컨테이너 오케스트레이션의 **De facto(사실상의 표준)**가 되어버린 도구가 하나 있습니다.

![container-orchestration16](/images/container-orchestration16.png)

[2022년 Redhat이 발표한 오픈소스 현황 보고서](https://www.redhat.com/en/resources/state-of-enterprise-open-source-report-2022?intcmp=701f2000000tjyaAAA&extIdCarryOver=true&sc_cid=701f2000001OH8HAAW)에서 전체 IT 리더들의 70%가 자신들의 조직에서 사용하고 있다고 응답한 그 기술.

바로 쿠버네티스가 그 주인공입니다.

어떻게 쿠버네티스는 이 수많은 컨테이너 오케스트레이션 도구들을 평정하고 독보적인 위치를 가지게 되었을까요? 무엇이 달랐길래 그리고 무엇이 특출났길래 컨테이너 오케스트레이션의 De facto가 되었을까요? 이에 관한 이야기는 다음 포스팅에서 이어가도록 하겠습니다.

# Reference

[초보를 위한 쿠버네티스 안내서 - 컨테이너 오케스트레이션이란? - YouTube](https://www.youtube.com/watch?v=Ia8IfowgU7s&list=PLIUCBpK1dpsNf1m-2kiosmfn2nXfljQgb&index=2&ab_channel=44BITS)

[클라우드의 시대, 컨테이너 오케스트레이션 툴이 반드시 필요한 이유 – ACCORDION (accordions.co.kr)](https://accordions.co.kr/it_trend/14778/)

[Kubernetes - 컨테이너 오케스트레이션 - wooody92’s blog](https://wooody92.github.io/kubernetes/Kubernetes-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EC%98%A4%EC%BC%80%EC%8A%A4%ED%8A%B8%EB%A0%88%EC%9D%B4%EC%85%98/)