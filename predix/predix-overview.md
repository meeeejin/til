# Predix Overview

> 수정 중..

이 문서는 [Predix Overview](https://docs.predix.io/en-US/content/platform/get_started/predix_overview/)를 일부 한글 번역한 문서로, 개인적인 학습 및 사용을 목적으로 번역된 비공식 문서이다.

## What is Predix Platform?

Predix 플랫폼은 산업용 인터넷을 위한 클라우드 기반 Platform-as-a-Service (PaaS)이다.

Predix 플랫폼은 기계, 데이터, 사람 및 기타 자산을 연결하기 때문에 분산 컴퓨팅, 빅데이터 분석, 자산 데이터 관리 및 기계 간 통신을 위한 선도적인 기술을 사용한다. 이 플랫폼은 기업이 생산성을 높일 수 있는 다양한 산업용 마이크로 서비스를 제공하며, 다음과 같은 이점을 제공한다:

- 산업용 어플리케이션을 빠르게 개발할 수 있다
- 규모(scale)와 하드웨의 관리에 개발자의 개입을 최소화한다
- 고객의 요구 사항에 신속하게 대응한다
- 고객이 소유한 자산에 대해 단일 지점 제어(a single point of control)을 제공한다
- 산업용 인터넷을 지원하는 기업 및 개발자의 생태계(ecosystem)를 위한 기반으로 사용된다

Predix 플랫폼을 이해하기 위해, 풍력 터빈과 같은 산업 자산을 예로 들어, Predix 기계, Predix 클라우드 및 Predix 서비스와 같은 기본 Predix 구성 요소가 산업 자산과 어떻게 연결되는지 살펴 보자.

![predix-turbine](https://docs.predix.io/NotusCloudApi/resources/image/@ZMDA2Vf0cHJlNDEZGlw4N@TktREFfMTE0dS/SD00311970)

### Predix Machine

Predix 기계는 산업 자산에서 데이터를 수집하고 Predix 클라우드로 푸시(push)하는 소프트웨어 계층이며, 엣지 분석(edge analytics)과 같은 로컬 어플리케이션을 실행한다. Predix 기계는 게이트웨이, 산업용 컨트롤러 및 센서에 설치된다.

### Predix Service

Predix는 개발자가 산업용 인터넷 어플리케이션을 작성, 테스트 및 실행할 때 사용할 수 있는 산업 서비스를 제공한다. 또한, 개발자가 자체 서비스를 게시하고 제 3자의 서비스를 사용할 수 있는 마이크로서비스 마켓을 제공한다. Predix 플랫폼을 통해 고객 및 파트너는 산업 비즈니스 프로세스를 최적화할 수 있다.

### Predix Cloud

Predix 클라우드는 산업 워크로드에 최적화 되었고, 헬스 케어 및 항공과 같은 산업에 대한 엄격한 규제 표준을 준수하는 글로벌 보안 클라우드 인프라이다.

## Predix Platform Architecture

Predix 플랫폼의 아키텍처를 보여주는 한 가지 예는 터빈에서 데이터를 수집하여 클라우드로 푸시하는 풍력 어플리케이션이다.

![predix-turbine-ex](https://docs.predix.io/NotusCloudApi/resources/image/@ZMDA2Vf0cHJlNDEZGlw4N@TktREFfMTE0dS/SD00311988)

풍력 터빈과 터빈 발전소는 Predix 플랫폼의 "엣지"에 있다. Predix 서비스를 사용하면, 풍력 터빈 센서의 데이터를 수신하고 분석할 수 있다. 또한 터빈 작동을 모니터링하고 최적화하여 이 자산의 최대 가치를 얻을 수도 있다. 이를 통해, 변칙을 감지하고 정전이 발생하기 전에 예측함으로써, 풍력 발전소의 신뢰성을 향상시키면서 적시 유지 보수(just-in-time maintenance)를 최적화하고, 터빈 가동 중단 시간을 줄일 수 있다.

Predix 기계는 센서에서 수집한 데이터를 사용하고 엣지 분석을 사용하여 산업 자산의 상태를 모니터링하는 하드웨어 및 소프트웨어 솔루션이다. 비정상적인 상황이 감지되면, Predix 기계는 손상이 발생하기 전에 터빈을 종료할 수 있다. Predix 기계를 사용하면, 데이터 과학자들이 풍력 발전 단지 전체에 걸친 데이터를 저장하고 분석할 수 있다. 데이터 과학자들은 시간에 따른 경향성을 찾고, 새로운 패턴을 확인하고, 새로운 엣지 분석을 생성하고, 그 정보를 모든 풍력 터빈으로 되돌려 보낼 수 있다. Predix 기계는 Predix 연결을 통해 인터넷이 다운된 경우에도 클라우드와 통신할 수 있다. 이 오프라인 지원 기능은 네트워크를 사용할 수 없는 경우에도 어플리케이션에 지속적인 데이터 엑세스를 제공한다.

어플리케이션 개발자는 Predix 클라우드의 산업 서비스를 사용하여 산업용 인터넷 어플리케이션을 구축, 테스트 및 배포할 수 있다. 이 맞춤형 클라우드 데이터 인프라 구조는 강화된 보안 제어 기능과 고급 데이터 처리 및 네트워킹 기능을 갖추고 있다. Predix 플랫폼은 향상된 분석, 실시간 자산 최적화 또는 예측 정비(predictive maintenance)를 통해 산업 비즈니스 프로세스의 지속적인 개선을 지원하도록 설계되었다.

## Predix Microservices

Predix 플랫폼은 마이크로서비스 사용 및 전달을 지원하는 Cloud Foundry 기반 아키텍처를 사용한다. 마이크로서비스는 매우 작고 세분화된 독립적인 협업 서비스의 집합으로서의 기능을 제공한다.

Predix 플랫폼 마이크로서비스는 배포된 어플리케이션의 운영 및 관리, IT 운영의 복잡성, 솔루션 통합 및 솔루션 관리를 단순화한다. 일단 솔루션이 배포되면, 코드 재컴파일 및 간소화 작업을 제거함으로써 업데이트가 보다 간단하고 효율적이다.

Predix 플랫폼은 광범위한 산업 마이크로서비스를 제공하여 예측 솔루션을 만드는데, 이는 자산 성능 관리, 운영 최적화, 자산 모델링, 데이터 수집, 데이터 저장 및 조작, 모든 백엔드 어플리케이션의 보안을 통해 기업이 생산성을 높일 수 있도록 한다.

## Predix and Cloud Foundry

Predix는 개발 비용을 절감하는 데 도움이 되는 오픈 소스 Platform-as-a-Service (PaaS)의 구현인 Cloud Foundry를 기반으로 한다.

Cloud Foundry는 초기 개발 단계부터 각 테스트 단계, 배포까지 소프트웨어 라이프 사이클을 지원한다. Cloud Foundry의 장점은 CD(continuous delivery) 소프트웨어 전략에 대한 강력한 지원이다. Cloud Foundry는 다음과 같은 이점을 제공한다:

- 어플리케이션 라이프 사이클 관리
- 어플리케이션의 중앙 집중식(centralized) 관리
- 분산 환경
- 쉬운 유지 보수

![predix-cloud-foundry](https://docs.predix.io/NotusCloudApi/resources/image/@ZMDA2Vf0cHJlNDEZGlw4N@TktREFfMTE0dS/DB00440648)
