![주차장](https://user-images.githubusercontent.com/64656963/86247292-39378200-bbe7-11ea-8a41-24a6d413f645.JPG)

# 개인 과제 : 일일주차권 관리

본 과제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계에 대한 내용입니다.

[ 업무 개요 ]
- 회사에서 보유한 주차장은 인별로 배정된 정기 주차권 이외에
- 일정 수량에 대해서는 일일주차권을 신청 받아 발급함
- 일일 주차권 예약, 처리, 주차권 배송을 관리하는 시스템

# Table of contents

- [과제 - 일일주차권 관리](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
    - [AS-IS 조직-Horizontally-Aligned](#AS-IS-조직-Horizontally-Aligned)
    - [TO-BE 조직-Vertically-Aligned](#TO-BE-조직-Vertically-Aligned)
    - [Event Storming 결과](#Event-Storming-결과)
    - [헥사고날 아키텍처 다이어그램 도출](#헥사고날-아키텍처-다이어그램-도출)
    - [신규 개발 조직의 추가](#신규-개발-조직의-추가)
    - [CQRS](#CQRS)
  - [구현](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [적용 후 REST API의 테스트](#적용-후-REST-API의-테스트)
    - [API Gateway](#API-Gateway)
    - [동기식 호출과 Fallback 처리](#동기식-호출과-Fallback-처리)
    - [비동기식 호출과 최종 일관성 테스트](#비동기식-호출과-최종-일관성-테스트)
  - [운영](#운영)
    - [CI-CD 설정](#CI-CD-설정)
    - [무정지 재배포](#무정지-재배포)
    - [ConfigMap 적용](#ConfigMap-적용)
    - [Secret 적용](#Secret-적용)
    - [동기식 호출과 서킷 브레이킹](#동기식-호출과-서킷-브레이킹)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [운영 모니터링](#운영-모니터링)

# 서비스 시나리오

기능적 요구사항
1. 직원이 특정 일자에 대한 일일 주차권을 예약/신청한다.
1. 일별 최대 배정 가능한 수량 초과 상태시, 예약 불가 이벤트를 보낸다.
1. 주차장 예약 처리가 되면, 신청 내역이 주차 관리자에게 전달된다.
1. 관리자는 주차권(주차장 출입시 차에 부착)을 신청자에게 전달한다.
1. 신청자는 일일 주차권 신청을 취소할 수 있다.
1. 신청이 취소되면, 주차장 일별 예약 가능 상태가 수정되고, 주차권 전달이 취소된다.
1. 주차권 예약/처리 진행 상태를 조회할 수 있다.

비기능적 요구사항
1. 트랜잭션
    1. 주차권 배송이 취소되지 않은 건은 주차장 예약 상태가 취소되지 않아야 한다.  Sync 호출 
1. 장애격리
    1. 주차장 상태 변경 기능이 수행되지 않더라도 주차권 예약은 365일 24시간 받을 수 있어야 한다.  Async (event-driven), Eventual Consistency
    1. 주차권 배송 취소가 과중되면 주차장은 잠시동안 취소를 받지 않고 잠시후에 하도록 유도한다.  Circuit breaker, fallback
1. 성능
    1. 직원은 주차권 예약, 처리, 배달 상태를 확인할 수 있어야 한다.  CQRS  

# 체크포인트

- 분석 설계 (40)
  - 이벤트스토밍 (15): 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - Readability: 인바운드와 아웃바운드 어댑터들의 위치가 왼쪽 상단과 우측하단에 배치되었는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?
  - 서브 도메인, 바운디드 컨텍스트 분리 (10)
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 팀(kpi)을 분리
      - 라일락 스티커의 위치사 팀간 이익에 따라 배치되었는가
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 (10)
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?
    - 취소에 따른 보상 트랜잭션을 설계하였는가(Saga Pattern)
    - 하나이상의 데이터 소스에서 데이터를 프로젝션하는 아키텍처가 보이는가?(CQRS)
  - 헥사고날 아키텍처 (5)
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
  
- 구현 (35)
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가? (5)
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 개발하였거나 자동 생성하였는가) (2)
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가? (2)
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가? (1)
  - Request-Response 방식의 서비스 중심 아키텍처 구현 (5)
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient) (3)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가? (2)
      - Spring Hystrix 를 이용
      - 운영에서 istio 로 보여주어도 됨
  - 이벤트 드리븐 아키텍처의 구현 (15)
    - 카프카를 이용하여 Pub/Sub 으로 하나 이상의 서비스가 연동되었는가? (7)
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가? (3)
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - 취소에 따른 보상 트랜잭션을 설계하였는가(Saga Pattern)
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
  - API 게이트웨이 (5)
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가? (2)
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가? (3)
  - 폴리글랏 프로그래밍 / 퍼시스턴스 (5)
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가? (2)
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가? (3)
  - 개발 능숙도
    - spring-boot 로 개발하는 방법을 이해하고 있다.
    - spring-boot 의 설정파일인 application.yaml 에 대해서 알고 있다.
    - 빌드시스템(maven, gradle)을 사용하여 라이브러리를 추가 할 수있다.

- 운영 (25)
  - SLA 준수 (10)
    - Pod를 생성시, Liveness와 Readiness Probe를 적용하여 준비가 되지않은 상태에서 요청을 받지 않도록 조치가 되었는가?
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가? 
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
      - 서킷브레이커 or 레이트리밋 적용
      - 리트라이 (retry) 적용
      - 풀-이젝션 적용
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
      - 실제 워크로드를 테스트 환경으로 생성하여 (seige  or JMeter) Replicas 의 증감을 보여줄 수 있는가?
    - 모니터링, 앨럿팅: 
      - 장애나 성능저하의 서비스 요청 건들에 대한 분산 추적을 할 수 있는가?
      - SLA 지표를 수립하여 원하는 대시보드를 구성할 수 있는가?
      - SLA 위협 임계치를 설정하여 자동으로 Alert 받도록 설정 할 수 있는가?
    - Stateless 한 구현이 되었는가? 하나이상의 레플리카가 동일한 데이터를 제공하는가? 퍼시스턴스 볼륨 혹은 분산 데이터베이스 인스턴스로 연결하였는가
  - 무정지 운영 CI/CD (10)
    - 플랫폼에서 제공하는 파이프라인을 적용하여 서비스를 클라우드에 배포하였는가? 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?
    - Advanced
      - Canary Deploy :  Istio 등을 이용하여 신규버전으로의 유입률을 높혀가면서 신규버전을 서서히 배포하고 문제가 발견될 때 최소한의 노출로 롤백할 수 있는가? 
      - Shadow Deploy, A/B Testing 등 각 2점
  - 운영 유연성 (5)
    - 데이터 저장소를 분리하기 위한 Persistence Volume과 Persistence Volume Claim 을 적절히 사용하였는가? 
    - ConfigMap과 Secret 을 사용할 수 있는가?



# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/64656963/86252726-8834e580-bbee-11ea-8708-1335aecd503d.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/64656963/86253117-0c876880-bbef-11ea-92cb-8f4df3fa76c3.png)

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://msaez.io/#/storming/dvf709nLm2hYUDGoOvvHX68wGlu2/mine/bdb33ed086647e2f2ed95e906dcb02e3/-MARmdGHbI-AKpTK-bTM


### 이벤트 도출
![image](https://user-images.githubusercontent.com/64656963/86260715-9982ef80-bbf8-11ea-92f9-7c1dc961e995.png)

### 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/64656963/86260801-bb7c7200-bbf8-11ea-81db-f71b0d2295b9.png)

### 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/64656963/86260856-cdf6ab80-bbf8-11ea-80fc-db665b891320.png)

    - TicketReservation의 일일주차권 예약, ParkingLot의 주차장 관리, TicketDelivery/TicketCancellation의 주차권 배송은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/64656963/86260918-e23aa880-bbf8-11ea-979f-33c0fa9d884c.png)

    - 도메인 서열 분리 
        - Core Domain: TicketReservation : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포 주기는 1주일 1회 미만
        - Supporting Domain: ParkingLot : 경쟁력을 내기 위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포 주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함
        - General Domain: TicketDelivery : 주차권 배송은 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (외부 근무자 사송 전달, 내부 근무자 아르바이트 직원 활용)

### 폴리시 부착

![image](https://user-images.githubusercontent.com/64656963/86260987-f67ea580-bbf8-11ea-9573-0521f8d4d089.png)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Res)

![image](https://user-images.githubusercontent.com/64656963/87327919-b06a0000-c56f-11ea-8ea9-62678cb0f0ba.png)


### 완성된 모델 검증 및 변경

![image](https://user-images.githubusercontent.com/64656963/87327955-bcee5880-c56f-11ea-8ba4-3016c6393e19.png)

    - Aggregate은 자신의 생명주기를 가지며, 데이터 변경의 단위로, 하나의 트랜잭션에서는 하나의 Aggregate만 수정되어야 함
    - TicketCancellation은 TicketDelivery의 상태를 변경하며, 자신의 식별성이나 수명주기가 없음
    - TicketCancellation을 TicketDelivery로 통합하는 것이 적절한 모델로 판단하여 모델 변경함

### 완성된 모델에 대한 기능적 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/64656963/87327989-caa3de00-c56f-11ea-9de7-48a5b10535a4.png)

    - 직원이 특정 일자에 대한 일일 주차권을 예약/신청한다. (ok)
    - 주차장 예약 처리가 되면, 신청 내역이 주차 관리자에게 전달된다. (ok)
    - 관리자는 주차권(주차장 출입시 차에 부착)을 신청자에게 전달한다. (ok)

![image](https://user-images.githubusercontent.com/64656963/87328046-e313f880-c56f-11ea-8734-3b0aa7b0d858.png)

    - 직원이 특정 일자에 대한 일일 주차권을 예약/신청한다. (ok)
    - 일별 최대 배정 가능한 수량 초과 상태시, 예약 불가 이벤트를 보낸다. (ok)

![image](https://user-images.githubusercontent.com/64656963/87328087-ee672400-c56f-11ea-87c0-84a1154be634.png)

    - 신청자는 일일 주차권 신청을 취소할 수 있다. (ok)
    - 신청이 취소되면, 주차장 일별 예약 가능 상태가 수정되고, 주차권 전달이 취소된다. (ok)

![image](https://user-images.githubusercontent.com/64656963/86266285-f1712480-bbff-11ea-9cf3-22ee215fce6a.png)

    - 주차권 예약/처리 진행 상태를 조회할 수 있다. (ok)
    
### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/64656963/87328133-0048c700-c570-11ea-834c-5229c92629cf.png)

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        1 트랜잭션(주차장 예약 상태 취소) : 주차권을 회수하지 않은 상태에서 주차장 공간을 가용 상태로 변경할 경우, 중복 예약 문제가 발생할 수 있으므로, ACID 트랜잭션 적용. Request-Response 방식 처리
        2 장애 격리 : 나머지 모든 inter-microservice의 이벤트는 데이터 일관성의 시점이 크리티컬하지 않은 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함
        3 성능: 주차권 예약, 처리, 배달 상태 조회 시, 개별 aggregate 통합 조회로 인한 성능 저하를 막기 위해 CQRS 적용

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/64656963/86526561-9295fe80-bed0-11ea-8ed1-005fefe1d910.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출 관계에서 Pub/Sub 과 Req/Res 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 관심 구현 스토리를 나눠가짐

## 신규 개발 조직의 추가

  ![image](https://user-images.githubusercontent.com/64656963/86277859-65b4c380-bc12-11ea-91c8-1a1a9baf63a6.png)


### 홍보팀의 추가
    - KPI: SV(Social Value) 창출을 위해, 주민들에게 업무 시간 이후에 사내 주차장 빈 공간을 제공
    - 구현 계획 마이크로 서비스: 주차장 빈 공간 조회, 예약 신청, 상태 알림 서비스를 제공할 예정

### 헥사고날 아키텍처 변화 

![image](https://user-images.githubusercontent.com/64656963/86526716-5ebbd880-bed2-11ea-9275-d207a32b4be6.png)

    - 폴리그랏 퍼시스턴스/프로그래밍
      . 서비스 특성, 각 팀의 기술 역량을 고려하여, DB, 프로그래밍 언어를 선택할 수 있음
      . 기존 서비스는 모두 mySQL, 자바를 사용했으나, 추가된 홍보팀은 Elastic Search와 파이썬을 사용하기로 함


### 아키텍처 변화에 따른 영향

기존의 마이크로 서비스에 수정을 발생시키지 않도록 Inbund 요청을 REST 가 아닌 Event 를 Subscribe 하는 방식으로 구현. 기존 마이크로 서비스에 대하여 아키텍처나 기존 마이크로 서비스들의 데이터베이스 구조와 관계없이 추가됨

### 운영과 Retirement

Request/Response 방식으로 구현하지 않았기 때문에 서비스가 더이상 불필요해져도 Deployment 에서 제거되면 기존 마이크로 서비스에 어떤 영향도 주지 않음

* [비교] 주차권 배송 (TicketDelivery) 마이크로서비스의 경우 API 변화나 Retire 시에 ParkingLot(주차장) 마이크로 서비스의 변경을 초래함:

예) API 변화시
```
# ParkingLot.java (Entity)

    @PostUpdate
    public void onPostUpdate(){

	parkingTicket.external.TicketDelivery ticketDelivery = new parkingTicket.external.TicketDelivery();
	// mappings goes here
	ticketDelivery.setParkingLotId(this.getId());
	ticketDelivery.setTicketReservationId(this.getTicketReservationId());
	ticketDelivery.setReservationDate(this.getReservationDate());
	ticketDelivery.setStatus(StatusType.Vacated);

	Application.applicationContext.getBean(parkingTicket.external.DeliveryCancelationService.class)
	    .cancel(ticketDelivery);
	
	----> 
	
	Application.applicationContext.getBean(parkingTicket.external.DeliveryCancelationService.class)
	    .cancel222(ticketDelivery);	    
    }       
```

예) Retire 시
```
# ParkingLot.java (Entity)

    @PostUpdate
    public void onPostUpdate(){
	/**
	parkingTicket.external.TicketDelivery ticketDelivery = new parkingTicket.external.TicketDelivery();
	// mappings goes here
	ticketDelivery.setParkingLotId(this.getId());
	ticketDelivery.setTicketReservationId(this.getTicketReservationId());
	ticketDelivery.setReservationDate(this.getReservationDate());
	ticketDelivery.setStatus(StatusType.Vacated);

	Application.applicationContext.getBean(parkingTicket.external.DeliveryCancelationService.class)
	    .cancel(ticketDelivery);
	**/
    } 
```

## CQRS

주차권 예약 상태 조회를 위한 서비스를 CQRS 패턴으로 구현
- 주차권 예약, 처리, 배달 상태 조회 시, 개별 aggregate 통합 조회로 인한 성능 저하를 막을 수 있음
- 모든 정보는 비동기 방식으로 발행된 이벤트를 수신하여 처리됨
- 별도의 서비스(reservationStatus), 저장소(AWS RDS-mySQL)로 구현

### CQRS 설계
- View(reservationStatus) 속성

![image](https://user-images.githubusercontent.com/64656963/86331157-a8fb4a80-bc83-11ea-8648-e65fabfe570e.png)

- 일일주차권 예약 이벤트 발생 시, reservationStatus create

![image](https://user-images.githubusercontent.com/64656963/86331209-bd3f4780-bc83-11ea-9147-7c75d1409e0f.png)

- 이벤트 발생 시, 상태 update

![image](https://user-images.githubusercontent.com/64656963/86331265-d0521780-bc83-11ea-9728-5959edd65d35.png)

![image](https://user-images.githubusercontent.com/64656963/86331313-e069f700-bc83-11ea-8632-660d9175e076.png)

![image](https://user-images.githubusercontent.com/64656963/86331385-fc6d9880-bc83-11ea-8a42-18fe45b900db.png)

- DELETE 이벤트 : 매핑하지 않음, 상태 정보는 삭제하지 않고 유지함


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084 이다)

```
- local -
cd TicketReservation
mvn spring-boot:run

cd ParkingLot
mvn spring-boot:run 

cd TicketDelivery
mvn spring-boot:run  

cd reservationStatus
mvn spring-boot:run 

- EKS -
CI/CD 통해 빌드/배포 ("운영 > CI-CD 설정" 부분 참조)

```

## DDD 의 적용

- 각 서비스 내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: 
(예시는 TicketReservation 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 
하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 영문으로 사용했다. 
(Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```
package parkingTicket;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

import java.util.Date;

@Entity
@Table(name="TicketReservation_table")
public class TicketReservation {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long memberId;
    private Date reservationDate;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getMemberId() {
        return memberId;
    }

    public void setMemberId(Long memberId) {
        this.memberId = memberId;
    }
    public Date getReservationDate() {
        return reservationDate;
    }

    public void setReservationDate(Date reservationDate) {
        this.reservationDate = reservationDate;
    }
}
```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package parkingTicket;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface TicketReservationRepository extends PagingAndSortingRepository<TicketReservation, Long>{

}
```
## 적용 후 REST API의 테스트

- 일일주차권 예약
1. ticketReservation POST 호출
```
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations memberId=3117 reservationDate=2020-07-13
```
2. ticketReservation POST 호출 : 실행 결과
![image](https://user-images.githubusercontent.com/64656963/86491507-f7086f00-bda5-11ea-8bfc-9037ad70af8e.png)

3. ticketReservation POST 호출 : kafka 이벤트
![image](https://user-images.githubusercontent.com/64656963/86491876-0dfb9100-bda7-11ea-91e2-f5b6bf08a17b.png)
```
TicketReserved 이벤트(TicketReservation) -> Occupied 이벤트(ParkingLot) -> Shipped 이벤트(TicketDelivery) : 순차적으로 발생함
```

4. ticketReservation POST 호출 : status 조회
```
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses
```
![image](https://user-images.githubusercontent.com/64656963/86491636-33d46600-bda6-11ea-8ed0-0abe0b48cdc0.png)
```
TicketReserved 이벤트(TicketReservation) -> Occupied 이벤트(ParkingLot) -> Shipped 이벤트(TicketDelivery)
순차적으로 발생하여, status는 Shipped 이벤트 처리 상태임
```

```
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses
-> reservationStatuses 전체 목록이 조회 되어, ticketReservation 건수가 많아질 경우, 특정 ticketReservation 조회만 찾기 어려움
-> 특정 ticketReservation 조회 REST API 추가 구현함 : ReservationStatusController class 추가, ReservationStatus Entity에 toString Override
-> http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses/reservationID?id={parameter} 형태로 호출하면 됨
```

- 일일주차권 취소
1. ticketReservation DELETE 호출
```
http DELETE http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations/2
```
2. ticketReservation DELETE 호출 : 실행 결과
![image](https://user-images.githubusercontent.com/64656963/86491676-536b8e80-bda6-11ea-874d-e8061ef949ca.png)

3. ticketReservation DELETE 호출 : kafka 이벤트
![image](https://user-images.githubusercontent.com/64656963/86491908-2ff51380-bda7-11ea-96ce-dbbdc8396951.png)
```
TicketReservationCanceled 이벤트(TicketReservation) -> DeliveryCanceled 이벤트(TicketDelivery) ->  Vacated 이벤트(ParkingLot)
ParkingLot -> TicketDelivery 동기 호출로, TicketDelivery의 이벤트가 먼저 발생함
```

4. ticketReservation DELETE 호출 : status 조회
```
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses
```
![image](https://user-images.githubusercontent.com/64656963/86491722-772ed480-bda6-11ea-99c5-59a36ac31498.png)
```
TicketReservationCanceled 이벤트(TicketReservation) -> DeliveryCanceled 이벤트(TicketDelivery) ->  Vacated 이벤트(ParkingLot)
ParkingLot -> TicketDelivery 동기 호출로, TicketDelivery의 이벤트가 먼저 발생하여
status는 Vacated 이벤트 처리 상태임
```

```
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses
-> reservationStatuses 전체 목록이 조회 되어, ticketReservation 건수가 많아질 경우, 특정 ticketReservation 조회만 찾기 어려움
-> 특정 ticketReservation 조회 REST API 추가 구현함 : ReservationStatusController class 추가, ReservationStatus Entity에 toString Override
-> http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses/reservationID?id={parameter} 형태로 호출하면 됨
```

- 일일주차 가능 대수 초과
1. ticketReservation POST 호출
```
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations memberId=6 reservationDate=2020-07-13
```
2. ticketReservation POST 호출 : 실행 결과
![image](https://user-images.githubusercontent.com/64656963/86494705-3ab4a600-bdb1-11ea-9cea-5f9cf84076f4.png)

3. ticketReservation POST 호출 : kafka 이벤트
![image](https://user-images.githubusercontent.com/64656963/86494754-6afc4480-bdb1-11ea-8c5e-49284665bbcf.png)
```
TicketReserved 이벤트(TicketReservation) -> FullyOccupied 이벤트(ParkingLot)  : 순차적으로 발생함
Shipped 이벤트(TicketDelivery)는 발생하지 않음
```

4. ticketReservation POST 호출 : status 조회
```
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses
```
![image](https://user-images.githubusercontent.com/64656963/86495075-c67b0200-bdb2-11ea-817a-c3cdc0776f6d.png)
```
TicketReserved 이벤트(TicketReservation) -> FullyOccupied 이벤트(ParkingLot)
순차적으로 발생하여, status는 FullyOccupied 이벤트 처리 상태임
Shipped 이벤트(TicketDelivery)는 발생하지 않아, ticketDeliveryId 값은 "null" 상태임
```

```
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses
-> reservationStatuses 전체 목록이 조회 되어, ticketReservation 건수가 많아질 경우, 특정 ticketReservation 조회만 찾기 어려움
-> 특정 ticketReservation 조회 REST API 추가 구현함 : ReservationStatusController class 추가, ReservationStatus Entity에 toString Override
-> http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses/reservationID?id={parameter} 형태로 호출하면 됨
```


## API Gateway

API Gateway를 통하여, 마이크로 서비스들의 진입점을 통일한다.

```
# application.yml 파일에 라우팅 경로 설정

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: TicketReservation
          uri: http://ticketreservation:8080
          predicates:
            - Path=/ticketReservations/** 
        - id: ParkingLot
          uri: http://parkinglot:8080
          predicates:
            - Path=/parkingLots/** 
        - id: TicketDelivery
          uri: http://ticketdelivery:8080
          predicates:
            - Path=/ticketDeliveries/**, /deliveryCancelations/**
        - id: reservationStatus
          uri: http://reservationstatus:8080
          predicates:
            - Path= /reservationStatuses/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080
  
```

- EKS에 배포 시, MSA는 Service type을 ClusterIP(default)로 설정하여, 클러스터 내부에서만 호출 가능하도록 한다.
- API Gateway는 Service type을 LoadBalancer로 설정하여 외부 호출에 대한 라우팅을 처리한다.

```
# buildspec.yml 설정

  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Service
  metadata:
    name: $_PROJECT_NAME
    labels:
      app: $_PROJECT_NAME
    spec:
    ports:
      - port: 8080
        targetPort: 8080
    selector:
      app: $_PROJECT_NAME
    type: LoadBalancer
  EOF
  
```

## 동기식 호출과 Fallback 처리

분석단계에서의 조건 중 하나로 주차장(ParkingLot)->주차권 배송(TicketDelivery) 간의 취소 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 주차권 배송(TicketDelivery) 서비스를 호출하기 위하여 FeignClient를 이용하여 Service 대행 인터페이스를 구현 

```
# (ParkingLot) DeliveryCancelationService.java

package parkingTicket.external;

//@FeignClient(name="TicketDelivery", url="http://localhost:8083") // fallback = DeliveryCancelationService.class)
@FeignClient(name="TicketDelivery", url="http://TicketDelivery:8080") // fallback = DeliveryCancelationService.class)
public interface DeliveryCancelationService {

    @RequestMapping(method= RequestMethod.POST, path="/deliveryCancelations")
    public void cancel(@RequestBody TicketDelivery ticketDelivery);

}
```

- 주차장 상태 변경 직후(@PostUpdate) 주차권 배송 취소를 요청하도록 처리
```
# ParkingLot.java (Entity)

    @PostUpdate
    public void onPostUpdate(){

	parkingTicket.external.TicketDelivery ticketDelivery = new parkingTicket.external.TicketDelivery();
	// mappings goes here
	ticketDelivery.setParkingLotId(this.getId());
	ticketDelivery.setTicketReservationId(this.getTicketReservationId());
	ticketDelivery.setReservationDate(this.getReservationDate());
	ticketDelivery.setStatus(StatusType.Vacated);

	Application.applicationContext.getBean(parkingTicket.external.DeliveryCancelationService.class)
	    .cancel(ticketDelivery);
    }    
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 주차권 배송(TicketDelivery) 시스템이 장애가 나면 주차장 취소도 처리되지 않는다.
- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)



## 비동기식 호출과 최종 일관성 테스트


주차권 예약/신청이 이루어진 후에 주차장 시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 주차장 시스템의 처리를 위하여 주차권 예약이 블로킹 되지 않도록 처리한다.
 
- 이를 위하여 주차권 예약에 기록을 남긴 후에 곧바로 예약이 되었다는 도메인 이벤트를 카프카로 송출한다.(Publish)
 
```
package parkingTicket;

@Entity
@Table(name="TicketReservation_table")
public class TicketReservation {

 ...
    @PostPersist
    public void onPostPersist(){
        TicketReserved ticketReserved = new TicketReserved();
        BeanUtils.copyProperties(this, ticketReserved);
        ticketReserved.publishAfterCommit();
    }

}
```
- 주차장 서비스에서는 주차권 예약 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.

```
package parkingTicket;

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverTicketReserved_Occupy(@Payload TicketReserved ticketReserved){

        if(ticketReserved.isMe()){
            System.out.println("##### listener Occupy : " + ticketReserved.toJson());

            ParkingLot parkingLot = new ParkingLot();
            parkingLot.setTicketReservationId(ticketReserved.getId());
            parkingLot.setReservationDate(ticketReserved.getReservationDate());

            // 일일 주차권 신규 요청의 예약일자 기준, 이미 예약된 주차 공간 수를 조회
            Date searchDate = ticketReserved.getReservationDate();
            //String searchStatus = "ParkingLot Occupied";
            String searchStatus = StatusType.Occupied;
            List<ParkingLot> searchParkingLots = parkingLotRepo.findByReservationDateAndStatus(searchDate, searchStatus);

            // 일일 주차권 발급 최대 가능 공간 수를 환경변수에서 추출
            String max =  env.getProperty("maxparking");
            System.out.println("##### max parking : " + max);
            System.out.println("##### searchParkingLots size : " + searchParkingLots.size());

            // 일일 주차권 발급 최대 가능 공간 수를 초과한 경우
            if(searchParkingLots.size() >= Long.valueOf(max)) {
                //parkingLot.setStatus("ParkingLot fully Occupied");
                parkingLot.setStatus(StatusType.FullyOccupied);
            } else { // 일일 주차권 발급 가능한 경우
                //parkingLot.setStatus("ParkingLot Occupied");
                parkingLot.setStatus(StatusType.Occupied);
            }

            parkingLotRepo.save(parkingLot);
        }
}

```
- 위 코드에서 status를 text로 하드 코딩하는 것은 (예: "ParkingLot fully Occupied") 좋지 않음
- 오타 등으로 인한 오류 발생 가능성 및 변경시 관련 소스 전체 변경 필요
- 아래 코드와 같이 class를 만들어 static final 변수로 사용함(예: StatusType.FullyOccupied)
```
public class StatusType {
    static final String FullyOccupied = "ParkingLot fully Occupied";
    static final String Occupied = "ParkingLot Occupied";
    static final String Vacated = "ParkingLot Vacated";
}
```

주차장 시스템은 주차권 예약과 완전히 분리되어 있으며, 이벤트 수신에 따라 처리되기 때문에, 주차장 시스템이 유지보수로 인해 잠시 내려간 상태라도 주차권 예약을 받는데 문제가 없다.
```
# 주차장 서비스 (ParkingLot) 를 잠시 내려놓음
  - kubectl delete svc parkinglot
  - kubectl delete deploy parkinglot

# 주차권 예약 처리
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations memberId=44 reservationDate=2020-07-13   #Success
```
![image](https://user-images.githubusercontent.com/64656963/86496110-d85ea400-bdb6-11ea-96f7-0c1e6fc7dc81.png)
```
# 주차권 예약 상태 확인
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses
```
![image](https://user-images.githubusercontent.com/64656963/86496148-00e69e00-bdb7-11ea-9780-ab7f54a7d698.png)
```
 - 주차권 예약 처리됨 : ticketReservationId, memberId, reservationDate 조회됨
 - 주차장 서비스 내려가 있어, 다른 값들은 모두 null 상태임
```

```
# 주차장 서비스 기동 : CI/CD(AWS Code Build) 통해 배포

# 주차권 예약 상태 확인
http http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/reservationStatuses
```
![image](https://user-images.githubusercontent.com/64656963/86496386-32ac3480-bdb8-11ea-9833-d78bec8fba33.png)
```
 - 주차권 서비스(ParkingLot) -> 주차권 배송 서비스(TicketDelivery) 모두 실행되어, Shipped 상태임
 - parkinglotId, ticketDeliveryId, status 값 생성/조회됨
```


# 운영

## CI-CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, CI/CD 플랫폼은 AWS Code Build를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.
- TicketReservation(일일주차권 예약) : https://github.com/maylover55/TicketReservation
- ParkiongLot(주차장) : https://github.com/maylover55/ParkingLot
- TicketDelivery(주차권 배송) : https://github.com/maylover55/TicketDelivery
- reservationStatus(주차권 예약 상태 조회) : https://github.com/maylover55/reservationStatus

![image](https://user-images.githubusercontent.com/64656963/86339987-3a23ee80-bc8f-11ea-8d73-617e919cba71.png)



## 무정지 재배포

* 쿠버네티스에서 어플리케이션을 업데이트 하는 방법은 Deployment를 활용하는 것임(spec > strategy에 type: RollingUpdate 설정)
* Rolling Update는 쿠버네티스의 default 배포 방식이며, 업데이트 프로세스 동안 무중단을 보장함
* Readiness 점검은 무중단을 제공하기 위한 Rolling Update에 중요한 요소임
* Liveness와 Readiness Probe는 Pod 내에서 실행되는 어플리케이션의 health를 조정하기 때문에 중요함
* Liveness Probe는 Pod의 상태를 체크하다가, Pod의 상태가 비정상인 경우 kubelet을 통해 재시작함
* Readiness Probe는 컨테이너가 비정상일 경우에는 서비스에서 제외함
 
```
# buildspec.yml 의 readinessProbe, livenessProbe 설정

    readinessProbe:
      httpGet:
	path: /actuator/health
	port: 8080
      initialDelaySeconds: 10
      timeoutSeconds: 2
      periodSeconds: 5
      failureThreshold: 10
    livenessProbe:
      httpGet:
	path: /actuator/health
	port: 8080
      initialDelaySeconds: 120
      timeoutSeconds: 2
      periodSeconds: 5
      failureThreshold: 5   

```
- 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 나 CB 설정을 제거함

- 새 버전으로의 배포 시작(AWS Code Build)
![image](https://user-images.githubusercontent.com/64656963/86499999-2c728400-bdc9-11ea-9896-ae29d6481043.png)

- siege로 워크로드 모니터링
```
siege -c100 -t180S --content-type "application/json" 'http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations POST {"memberId": "88", "reservationDate": "2020-07-13"}'
```

- siege 실행 결과, 배포기간 동안 Availability 100% 유지되어, 무정지 재배포가 성공한 것으로 확인됨
  (seige 180초, Code Build 65초 소요됨)
  
![image](https://user-images.githubusercontent.com/64656963/86506686-bb54c000-be0c-11ea-8110-5740749872f0.png)



## ConfigMap 적용

- 설정의 외부 주입을 통한 유연성을 제공하기 위해 ConfigMap을 적용
- 일별 최대 주차 가능 대수(maxparking), AWS RDS(mysql) 접속 정보(url)를 ConfigMap에 정의
- 설정 변경 시, 빌드/재배포가 필요 없으며, container 재기동만 하면 됨 (환경 변수 동적 로딩 가능할 경우, 재기동도 필요 없음) 

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: parking
data:
  maxparking: "5"
  urlstatus: "jdbc:mysql://xxxx.rds.amazonaws.com:3306/status?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8"
  urlparkinglot: "jdbc:mysql://xxxx.rds.amazonaws.com:3306/parkinglot?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8"
  urlticketreservation: "jdbc:mysql://xxxx.rds.amazonaws.com:3306/ticketreservation?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8"
  urlticketdelivery: "jdbc:mysql://xxxx.us-east-2.rds.amazonaws.com:3306/ticketdelivery?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8"
EOF
```

## Secret 적용

- 설정의 외부 주입을 통한 유연성을 제공하기 위해 Secret을 적용
- username, password와 같은 민감한 정보는 ConfigMap이 아닌 Secret을 적용
- etcd에 암호화 되어 저장되어, ConfigMap 보다 안전함
- value는 base64 인코딩 된 값으로 지정함 (echo root | base64)

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: parking
type: Opaque
data:
  username: xxxxxxxx== <- 보안 상, 임의의 값으로 표시함
  password: xxxxxxxx== <- 보안 상, 임의의 값으로 표시함
EOF
```


## 동기식 호출과 서킷 브레이킹

* 요청/응답 통신은 성능 저하와 장애 전파를 회피하기 위한 전략을 새워야 함
* 요청/응답 통신 중에 특정 서비스가 응답 시간이 느려진다거나, 오류가 특정치 이상 발생할때 서킷브레이커를 적용하여 장애 전파를 원천 차단 할 수 있음
* MSA에 서킷 브레이커 기능을 개발할 경우, 모든 서비스에 중복 코딩이 되어야 하며, 사용하는 개발 언어별로 다르게 구현됨
* MSA 서비스는 도메인 로직 구현에 집중해야 하며, 부가 기능(서킷브레이커, 로그, 모니터링, 트레이싱, 보안 등)은 사이드카 패턴을 적용하는 것이 좋음

* 쿠버네티스에 istio를 사용해서 서킷 브레이커 적용이 가능함
```
# istio 사용 시 좋은 점
  - L7 레이어를 사용, 성능이 좋음
  - Code 변경 없이 Cross-cutting 이슈를 다루어 줌
  - Main 서비스의 재배포 없이 사이드카를 관리 가능함
```

- 서비스를 istio로 배포(동기 호출하는 Request/Response 2개 서비스)
```
kubectl get deploy parkinglot -o yaml > parkinglot_deploy.yaml 
kubectl apply -f <(istioctl kube-inject -f parkinglot_deploy.yaml) 

kubectl get deploy ticketdelivery -o yaml > ticketdelivery_deploy.yaml 
kubectl apply -f <(istioctl kube-inject -f ticketdelivery_deploy.yaml) 

```

- istio 에서 서킷브레이커 설정(DestinationRule)
```
cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ticketdelivery
spec:
  host: ticketdelivery
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1           # 목적지로 가는 HTTP, TCP connection 최대 값. (Default 1024)
      http:
        http1MaxPendingRequests: 1  # 연결을 기다리는 request 수를 1개로 제한 (Default 
        maxRequestsPerConnection: 1 # keep alive 기능 disable
        maxRetries: 3               # 기다리는 동안 최대 재시도 수(Default 1024)
    outlierDetection:
      consecutiveErrors: 5          # 5xx 에러가 5번 발생하면
      interval: 1s                  # 1초마다 스캔 하여
      baseEjectionTime: 30s         # 30 초 동안 circuit breaking 처리   
      maxEjectionPercent: 100       # 100% 로 차단
EOF

```

* siege 툴을 통한 서킷 브레이커 동작 확인 : 동시사용자 100명, 60초 동안 실시

```
siege -c100 -t60S --content-type "application/json" 'http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations POST {"memberId": "88", "reservationDate": "2020-07-13"}'

- 호출 구조 : TicketReservation (kafka) -> ParkingLot (동기 호출) -> TicketDelivery
- TicketReservation 서비스에 부하를 주어도 ParkingLot에는 kafka를 통해 전달이 되어서 인지, 서킷 브레이커 동작 확인을 못함
- parkingLot은 POST REST API를 제공하지 않아, parkingLot에 직접 부하를 주지 못함

```



### 오토스케일 아웃

- 대부분의 워크로드는 시간이 지남에 따라 변화하는 동적 특성 때문에, 고정된 스케일링 설정을 적용하기가 어려움
- 그러나, 쿠버네티스에서는 변화하는 로드에 따라 적응하는 어플리케이션을 만들 수 있음
- 오토스케일링을 통해, 고정되어 있지 않으면서도 다른 로드를 충분히 처리할 수 있는 용량을 보장할 수 있음

- 주차권 예약 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 1프로를 넘어서면 replica 를 10개까지 늘려줌:
  (오토스케일링이 발생하지 않아, CPU 사용량을 극단적으로 작게 설정함)
```
kubectl autoscale deploy ticketreservation --min=1 --max=10 --cpu-percent=1
```
- siege 로 워크로드를 300초 동안 걸어줌
```
siege -c20 -t300S --content-type "application/json" 'http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations POST {"memberId": "88", "reservationDate": "2020-07-13"}'
```
- 사용자 20명(-c20)으로 걸었으나, 오류 발생함
  ![image](https://user-images.githubusercontent.com/64656963/86501666-eec82800-bdd5-11ea-9ba6-30568a765865.png)

- TicketReservation 서비스는 mySQL을 사용하고 있음 (프리티어 db.t2.micro 인스턴스로 생성)
- DB 연동 병목을 의심해서, DB를 H2(인메모리)로 변경함

- H2로 변경 후, 사용자 250명(-c250) 이상으로 설정 시는 동일 오류 발생 (Availability 70.54% 로 떨어짐)
```
siege -c250 -t180S --content-type "application/json" 'http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations POST {"memberId": "88", "reservationDate": "2020-07-13"}'
```
  ![image](https://user-images.githubusercontent.com/64656963/86506379-dd990e80-be09-11ea-8892-4630676bd7fe.png)

- 사용자 200명(-c200)으로 워크로드를 3분 동안 걸어줌 
```
siege -c200 -t180S --content-type "application/json" 'http://a63c1be5dd2104b04a6b0de92e51a7e8-1186153420.us-east-2.elb.amazonaws.com:8080/ticketReservations POST {"memberId": "88", "reservationDate": "2020-07-13"}'
```

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둠
```
kubectl get deploy ticketreservation -w

kubectl get hpa ticketreservation -w
```

- siege 실행 결과 오류 없이 수행됨(Availability 100%)
![image](https://user-images.githubusercontent.com/64656963/86507393-dcb8aa80-be12-11ea-9ab9-35cdb2ceee46.png)

- --cpu-percent=1로 설정했음에도, 오토스케일 발생하지 않음
![image](https://user-images.githubusercontent.com/64656963/86507426-0d98df80-be13-11ea-88f1-015644f0f43c.png)
![image](https://user-images.githubusercontent.com/64656963/86507435-2dc89e80-be13-11ea-9410-fb70427a41a6.png)



## 운영 모니터링

### 쿠버네티스 구조
쿠버네티스는 Master Node(Control Plane)와 Worker Node로 구성된다.

![image](https://user-images.githubusercontent.com/64656963/86503139-09a29880-bde6-11ea-8706-1bba1f24d22d.png)


### 1. Master Node(Control Plane) 모니터링
Amazon EKS 제어 플레인 모니터링/로깅은 Amazon EKS 제어 플레인에서 계정의 CloudWatch Logs로 감사 및 진단 로그를 직접 제공한다.

- 사용할 수 있는 클러스터 제어 플레인 로그 유형은 다음과 같다.
```
  - Kubernetes API 서버 컴포넌트 로그(api)
  - 감사(audit) 
  - 인증자(authenticator) 
  - 컨트롤러 관리자(controllerManager)
  - 스케줄러(scheduler)

출처 : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/logging-monitoring.html
```

- 제어 플레인 로그 활성화 및 비활성화
```
기본적으로 클러스터 제어 플레인 로그는 CloudWatch Logs로 전송되지 않습니다. 
클러스터에 대해 로그를 전송하려면 각 로그 유형을 개별적으로 활성화해야 합니다. 
CloudWatch Logs 수집, 아카이브 스토리지 및 데이터 스캔 요금이 활성화된 제어 플레인 로그에 적용됩니다.

출처 : https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/control-plane-logs.html
```

### 2. Worker Node 모니터링

- 쿠버네티스 모니터링 솔루션 중에 가장 인기 많은 것은 Heapster와 Prometheus 이다.
- Heapster는 쿠버네티스에서 기본적으로 제공이 되며, 클러스터 내의 모니터링과 이벤트 데이터를 수집한다.
- Prometheus는 CNCF에 의해 제공이 되며, 쿠버네티스의 각 다른 객체와 구성으로부터 리소스 사용을 수집할 수 있다.

- 쿠버네티스에서 로그를 수집하는 가장 흔한 방법은 fluentd를 사용하는 Elasticsearch 이며, fluentd는 node에서 에이전트로 작동하며 커스텀 설정이 가능하다.

- 그 외 오픈소스를 활용하여 Worker Node 모니터링이 가능하다. 아래는 istio, mixer, grafana, kiali를 사용한 예이다.

```
아래 내용 출처: https://bcho.tistory.com/1296?category=731548

```
- 마이크로 서비스에서 문제점중의 하나는 서비스가 많아 지면서 어떤 서비스가 어떤 서비스를 부르는지 의존성을 알기가 어렵고, 각 서비스를 개별적으로 모니터링 하기가 어렵다는 문제가 있다. Istio는 네트워크 트래픽을 모니터링함으로써, 서비스간에 호출 관계가 어떻게 되고, 서비스의 응답 시간, 처리량등의 다양한 지표를 수집하여 모니터링할 수 있다.

![image](https://user-images.githubusercontent.com/64656963/86347967-ff738380-bc99-11ea-9b5e-6fb94dd4107a.png)

- 서비스 A가 서비스 B를 호출할때 호출 트래픽은 각각의 envoy 프록시를 통하게 되고, 호출을 할때, 응답 시간과 서비스의 처리량이 Mixer로 전달된다. 전달된 각종 지표는 Mixer에 연결된 Logging Backend에 저장된다.

- Mixer는 위의 그림과 같이 플러그인이 가능한 아답터 구조로, 운영하는 인프라에 맞춰서 로깅 및 모니터링 시스템을 손쉽게 변환이 가능하다.  쿠버네티스에서 많이 사용되는 Heapster나 Prometheus에서 부터 구글 클라우드의 StackDriver 그리고, 전문 모니터링 서비스인 Datadog 등으로 저장이 가능하다.

![image](https://user-images.githubusercontent.com/64656963/86348023-14501700-bc9a-11ea-9759-a40679a6a61b.png)

- 이렇게 저장된 지표들은 여러 시각화 도구를 이용해서 시각화 될 수 있는데, 아래 그림은 Grafana를 이용해서 서비스의 지표를 시각화 한 그림이다.

![image](https://user-images.githubusercontent.com/64656963/86348092-25992380-bc9a-11ea-9d7b-8a7cdedc11fc.png)

- 그리고 근래에 소개된 오픈소스 중에서 흥미로운 오픈 소스중의 하나가 Kiali (https://www.kiali.io/)라는 오픈소스인데, Istio에 의해서 수집된 각종 지표를 기반으로, 서비스간의 관계를 아래 그림과 같이 시각화하여 나타낼 수 있다.  아래는 그림이라서 움직이는 모습이 보이지 않지만 실제로 트래픽이 흘러가는 경로로 에니메이션을 이용하여 표현하고 있고, 서비스의 각종 지표, 처리량, 정상 여부, 응답 시간등을 손쉽게 표현해 준다.

![image](https://user-images.githubusercontent.com/64656963/86348145-3a75b700-bc9a-11ea-8477-e7e7178c51fe.png)
