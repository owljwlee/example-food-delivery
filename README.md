![image](https://user-images.githubusercontent.com/24615790/223068675-18f65545-553f-4893-aa1c-2f4144ae1ed5.png)

# 항공 예약 시스템(MSA Air)


본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents
- [항공 예약 시스템]
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오


기능적 요구사항
1. 항공사관리자는 항공편 일정을 등록한다.
2. 항공사관리자는 등록된 항공편 일정을 취소할 수 있다.
3. 고객이 항공편을 선택하여 결재까지 완료시 항공편 예약이 생성된다.
4. 예약 생성시 마일리지가 적립된다.
5. 고객이 예약한 항공편을 취소할 수 있다.
6. 예약 취소시 예약결재시의 마일리지 적립이 취소된다.
7. 예약상태가 변경될 때마다, 고객에게 알림을 보낸다.
8. 고객은 적립된 마일리지를 조회할 수 있다.

비기능적 요구사항
1. 트랜잭션
   1. 존재하는 Flight에 대해서만 예약이 가능하다.
   2. 예약완료시 예약건에 대한 마일리지가 적립된다.
   3. 예약취소시 예약건에 대한 마일리지가 취소된다.
2. 장애격리
   1. 마일리지 적립/취소와 상관없이 항공편 예약은 24/365로 가능해야 한다
   2. 예약상태 변경관련 고객알림 기능의 정상동작여부와 상관없이 항공편 예약이 가능해야 한다.
3. 성능
   1. 고객은 본인의 예약이력을 확인할 수 있어야 한다.(CQRS)
   2. 예약상태 변경시마다 고객에게 알림이 발송되어야 한다(Event Driven)
   3. 성수기에는 예약건수가 폭주하기 때문에, 예약서비스에 대한 Scalability가 보장되어야 한다.

# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계

### Stakeholder 도출

name|역할|비고
:---|:--|:---
AirFlightMgr|항공편관리자
Customer|고객

### Command 도출
Stakeholder|command|기능
:---|:---|:---
AirFlightMgr|create schedule|항공편 일정을 등록한다.
AirFlightMgr|delete schedule|항공편 일정을 취소한다.
Customer|create reservation|항공편을 예약한다.
Customer|delete reservation|예약한 항공편을 취소한다.
Customer|inquiry reservations|예약정보를 본다.
Customer|inquery mileage|고객이 마일리지를 조회한다.
Customer|inquery history|고객이 예약,예약취소,마일리지적립기록등의 모든 history를 조회한다.

### Bounded Context
Bounded Context|설명|비고
:---|:---|:---
scheduleMgmt|항공편 일정 관리|
reservationMgmt|항공편 예약 관리|
customerMgmt|고객 mileage 적립관리|
notiMgmt|고객알림 관리|
reservationhist|history 관리|

### Event 도출 및 Event와 Bounded Context간 pub/sub 관계 도출
Event Name|scheduleMgmt|reservationMgmt|customerMgmt|notiMgmt|reservationhist|
:---|:---|:---|:---|:---|:---
ReservationCreated||Pub.|Sub.|Sub.|Sub.
MileageIncreased|||Pub.|Sub.|Sub.
ReservationCancelled||Pub.|Sub.|Sub.|Sub.
MileageDecreased|||Pub.|Sub.|Sub.


### 완성된 모델
![image](https://user-images.githubusercontent.com/24615790/223145528-1eb36776-8c98-416e-8b47-7c0e479d9747.png)
### 기능 요구사항에 대한 검증
요구사항|충족여부|비고
:---|:---|:---
항공사관리자는 항공편 일정을 등록한다.|Y|schedulemgmt 서비스의 create schedule api를 통해 처리 가능
항공사관리자는 등록된 항공편 일정을 취소할 수 있다.|Y|schedulemgmt 서비스의 delete schedule api를 통해 처리 가능
고객이 항공편을 선택하여 결재까지 완료시 항공편 예약이 생성된다.|Y|reservationmgmt의 create reservation api를 통해 처리 가능
예약 생성시 마일리지가 적립된다.|Y|reservationmgmt에서 pub., reservationCreated Event를 cusotomermgnt에서 sub.하여 처리
고객이 예약한 항공편을 취소할 수 있다.|Y|reservationmgmt의 delete reservation api를 통해 처리 가능
예약 취소시 예약결재시의 마일리지 적립이 취소된다.|Y|reservationmgmt에서 pub.한 reservationCanclled Event를 cusotomermgnt에서 sub.하여 처리
예약상태가 변경될 때마다, 고객에게 알림을 보낸다.|Y|발생하는 모든 event를 notimgmt service가 sub.하여 상태변경시마다 알림 처리
고객은 적립된 마일리지를 조회할 수 있다.|Y|customermgmt에서 제공하는 api를 통해 처리 가능

### 비기능 요구사항에 대한 검증
구분|요구사항|충족여부|비고
:---|:---|:---|:---
트랜잭션|존재하는 Flight에 대해서만 예약이 가능하다.|Y|schedulemgmt에서 제공하는 API를 통해 검증처리
트랜잭션|예약완료시 예약건에 대한 마일리지가 적립된다.|Y|Saga Pattern으로 구현
트랜잭션|예약취소시 예약건에 대한 마일리지가 취소된다.|Y|Saga Pattern으로 구현
장애격리|마일리지 적립/취소와 상관없이 항공편 예약은 24/365로 가능해야 한다|Y|마일리지 적립기능이 Saga Pattern으로 구현되어, 마일리지 적립서비스에 지연이 발생해도 예약가능
장애격리|예약상태 변경관련 고객알림 기능의 정상동작여부와 상관없이 항공편 예약이 가능해야 한다.|Y|알림기능이 Saga Pattern으로 구현되어,알림기능에 지연이 발생해도 예약가능
성능|고객은 본인의 예약이력을 확인할 수 있어야 한다.(CQRS)|Y|CQRS Patternd으로 구현
성능|예약상태 변경시마다 고객에게 알림이 발송되어야 한다(Event Driven)|Y|Saga pattern으로 구현
성능|성수기에는 예약건수가 폭주하기 때문에, 예약서비스에 대한 Scalability가 보장되어야 한다.|Y|예약기능을 담당하는 reservationmgmt가 별도의 Bounded Context로 분리되어. 예약기능만 scale out 처리 가능

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/487999/79684772-eba9ab00-826e-11ea-9405-17e2bf39ec76.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다.
microService name|comment|dir.|할당 port
:---|:---|:---|:--
schedulemgmt|항공편 일정 관리|schedulemgmt|8081
reservationmgmt|항공편 예약 관리|reservationmgmt|8083
customermgmt|고객 mileage 적립관리|customermgmt|8082
notimgmt|고객알림 관리|notimgmt|8084
reservationhist|history 관리|reservationhist8085

적용항목|구현여부|비고
:---|:---|:---
DDD적용|Y|
Saga (Pub-Sub)|Y| 
CQRS|Y|
Compensation & Correlation|Y|
동기식호출|Y|

각 서비스를 로컬에서 수행하는 방법
```
cd schedulemgmt
mvn spring-boot:run

cd reservationmgmt
mvn spring-boot:run 

cd customermgmt
mvn spring-boot:run  

cd notimgmt
mvn spring-boot:run 

cd reservationhis
mvn spring-boot:run 
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```
package fooddelivery;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="결제이력_table")
public class 결제이력 {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String orderId;
    private Double 금액;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getOrderId() {
        return orderId;
    }

    public void setOrderId(String orderId) {
        this.orderId = orderId;
    }
    public Double get금액() {
        return 금액;
    }

    public void set금액(Double 금액) {
        this.금액 = 금액;
    }

}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package fooddelivery;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface 결제이력Repository extends PagingAndSortingRepository<결제이력, Long>{
}
```
- 적용 후 REST API 의 테스트
```
# app 서비스의 주문처리
http localhost:8081/orders item="통닭"

# store 서비스의 배달처리
http localhost:8083/주문처리s orderId=1

# 주문 상태 확인
http localhost:8081/orders/1

```


## 폴리글랏 퍼시스턴스

앱프런트 (app) 는 서비스 특성상 많은 사용자의 유입과 상품 정보의 다양한 콘텐츠를 저장해야 하는 특징으로 인해 RDB 보다는 Document DB / NoSQL 계열의 데이터베이스인 Mongo DB 를 사용하기로 하였다. 이를 위해 order 의 선언에는 @Entity 가 아닌 @Document 로 마킹되었으며, 별다른 작업없이 기존의 Entity Pattern 과 Repository Pattern 적용과 데이터베이스 제품의 설정 (application.yml) 만으로 MongoDB 에 부착시켰다

```
# Order.java

package fooddelivery;

@Document
public class Order {

    private String id; // mongo db 적용시엔 id 는 고정값으로 key가 자동 발급되는 필드기 때문에 @Id 나 @GeneratedValue 를 주지 않아도 된다.
    private String item;
    private Integer 수량;

}


# 주문Repository.java
package fooddelivery;

public interface 주문Repository extends JpaRepository<Order, UUID>{
}

# application.yml

  data:
    mongodb:
      host: mongodb.default.svc.cluster.local
    database: mongo-example

```

## 폴리글랏 프로그래밍

고객관리 서비스(customer)의 시나리오인 주문상태, 배달상태 변경에 따라 고객에게 카톡메시지 보내는 기능의 구현 파트는 해당 팀이 python 을 이용하여 구현하기로 하였다. 해당 파이썬 구현체는 각 이벤트를 수신하여 처리하는 Kafka consumer 로 구현되었고 코드는 다음과 같다:
```
from flask import Flask
from redis import Redis, RedisError
from kafka import KafkaConsumer
import os
import socket


# To consume latest messages and auto-commit offsets
consumer = KafkaConsumer('fooddelivery',
                         group_id='',
                         bootstrap_servers=['localhost:9092'])
for message in consumer:
    print ("%s:%d:%d: key=%s value=%s" % (message.topic, message.partition,
                                          message.offset, message.key,
                                          message.value))

    # 카톡호출 API
```

파이선 애플리케이션을 컴파일하고 실행하기 위한 도커파일은 아래와 같다 (운영단계에서 할일인가? 아니다 여기 까지가 개발자가 할일이다. Immutable Image):
```
FROM python:2.7-slim
WORKDIR /app
ADD . /app
RUN pip install --trusted-host pypi.python.org -r requirements.txt
ENV NAME World
EXPOSE 8090
CMD ["python", "policy-handler.py"]
```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(app)->결제(pay) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (app) 결제이력Service.java

package fooddelivery.external;

@FeignClient(name="pay", url="http://localhost:8082")//, fallback = 결제이력ServiceFallback.class)
public interface 결제이력Service {

    @RequestMapping(method= RequestMethod.POST, path="/결제이력s")
    public void 결제(@RequestBody 결제이력 pay);

}
```

- 주문을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# 결제 (pay) 서비스를 잠시 내려놓음 (ctrl+c)

#주문처리
http localhost:8081/orders item=통닭 storeId=1   #Fail
http localhost:8081/orders item=피자 storeId=2   #Fail

#결제서비스 재기동
cd 결제
mvn spring-boot:run

#주문처리
http localhost:8081/orders item=통닭 storeId=1   #Success
http localhost:8081/orders item=피자 storeId=2   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


결제가 이루어진 후에 상점시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 상점 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package fooddelivery;

@Entity
@Table(name="결제이력_table")
public class 결제이력 {

 ...
    @PrePersist
    public void onPrePersist(){
        결제승인됨 결제승인됨 = new 결제승인됨();
        BeanUtils.copyProperties(this, 결제승인됨);
        결제승인됨.publish();
    }

}
```
- 상점 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package fooddelivery;

...

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void whenever결제승인됨_주문정보받음(@Payload 결제승인됨 결제승인됨){

        if(결제승인됨.isMe()){
            System.out.println("##### listener 주문정보받음 : " + 결제승인됨.toJson());
            // 주문 정보를 받았으니, 요리를 슬슬 시작해야지..
            
        }
    }

}

```
실제 구현을 하자면, 카톡 등으로 점주는 노티를 받고, 요리를 마친후, 주문 상태를 UI에 입력할테니, 우선 주문정보를 DB에 받아놓은 후, 이후 처리는 해당 Aggregate 내에서 하면 되겠다.:
  
```
  @Autowired 주문관리Repository 주문관리Repository;
  
  @StreamListener(KafkaProcessor.INPUT)
  public void whenever결제승인됨_주문정보받음(@Payload 결제승인됨 결제승인됨){

      if(결제승인됨.isMe()){
          카톡전송(" 주문이 왔어요! : " + 결제승인됨.toString(), 주문.getStoreId());

          주문관리 주문 = new 주문관리();
          주문.setId(결제승인됨.getOrderId());
          주문관리Repository.save(주문);
      }
  }

```

상점 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 상점시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# 상점 서비스 (store) 를 잠시 내려놓음 (ctrl+c)

#주문처리
http localhost:8081/orders item=통닭 storeId=1   #Success
http localhost:8081/orders item=피자 storeId=2   #Success

#주문상태 확인
http localhost:8080/orders     # 주문상태 안바뀜 확인

#상점 서비스 기동
cd 상점
mvn spring-boot:run

#주문상태 확인
http localhost:8080/orders     # 모든 주문의 상태가 "배송됨"으로 확인
```


# 운영
항목|구현여부|비고
:---|:---|:---
Gateway/Ingress|Y|spring gateway 사용
Deploy/Pipeline|Y|• Image를 생성하여 docker.io에 push<br>• docker.io의 image를 사용하여 aws eks에 배포
Autoscale (HPA)|Y|• 특정 Service(reservationmgmt)에 대해서 AutoScale 설정<br>• 트래픽 과다발생시 scale out 확인
Self-healing|Y|• Liveness probe 설정하여 배포 
Zero-downtime deploy|Y|• Readiness probe 설정하여 배포<br>• 신규버젼 배포시 무정지 배포 확인
Persistence Volume/ConfigMap/Secret|Y|
Apply Service Mesh|Y|• Micro service들이 배포된 namespace에 Initio-enabled 처리<br>• Micro service 재기동하여 sidecar injection 처리
Loggregation / Monitoring|Y|EFK stack 설치하여, 로그모니터링 처리

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 GCP를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 cloudbuild.yml 에 포함되었다.


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (pay) 결제이력.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

* **Gateway를 서비스 접근 정상처리 확인**
![image](https://user-images.githubusercontent.com/24615790/223391959-32259257-0bdd-4c1b-85cc-2dc43d55f0fc.png)
---

### Zero-downtime deploy (Readiness probe)
* 신규 버젼의 이미지 배포시 무중단 처리를 위해 수행.
* 이미지의 version을 변경하여, docker.io에 push
```
gitpod /workspace/msaair (main) $ docker images
REPOSITORY                  TAG                IMAGE ID       CREATED       SIZE
owljw/reservationmgmt       4                  00f71ff365c9   5 hours ago   417MB
owljw/reservationmgmt       3                  fbdb9d2a4393   6 hours ago   417MB
owljw/reservationmgmt       2                  e6b148cbf6cd   6 hours ago   417MB
owljw/gateway               1                  611c7abde2ee   7 hours ago   130MB
owljw/reservationhist       1                  2b3eafd72d2b   7 hours ago   417MB
owljw/notimgmt              1                  8de93786b2df   7 hours ago   417MB
owljw/customermgmt          1                  ee6d6268bb28   7 hours ago   417MB
owljw/reservationmgmt       1                  643513d571f0   7 hours ago   417MB
owljw/schedulemgmt          1                  504b7b3c30fb   9 hours ago   417MB
confluentinc/cp-kafka       latest             da23a46211ad   2 weeks ago   832MB
confluentinc/cp-zookeeper   latest             8a091961522e   2 weeks ago   832MB
openjdk                     15-jdk-alpine      f02adfce91a2   2 years ago   343MB
openjdk                     8u212-jdk-alpine   a3562aa0b991   3 years ago   105MB
```
* deployment.yaml에 Readiness Probe 설정 확인 및 신규버젼으로 배포되도록 버젼 변경
```
    spec
      containers:
        - name: reservationmgmt
          image: owljw/reservationmgmt:3
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
```         
* siege를 통해 대상 서비스에 부하발생 처리
```
root@siege:/# siege -c1 -t60S -v abc9cccd4c17d41af9bf9e37f59548a2-256813655.ap-southeast-2.elb.amazonaws.com:8080/reservations/ 
HTTP/1.1 200     0.01 secs:     460 bytes ==> GET  /reservations/
HTTP/1.1 200     0.02 secs:     460 bytes ==> GET  /reservations/
HTTP/1.1 200     0.01 secs:     460 bytes ==> GET  /reservations/
HTTP/1.1 200     0.02 secs:     460 bytes ==> GET  /reservations/
HTTP/1.1 200     0.02 secs:     460 bytes ==> GET  /reservations/
HTTP/1.1 200     0.01 secs:     460 bytes ==> GET  /reservations/
```
* 신규버젼으로 배포처리
```
gitpod /workspace/msaair/reservationmgmt (main) $ kubectl apply -f kubernetes/deployment.yaml
deployment.apps/reservationmgmt configured
gitpod /workspace/msaair/reservationmgmt (main) $ kubectl get pods -w
NAME                               READY   STATUS    RESTARTS      AGE
customermgmt-98c4cfcfc-82nk5       2/2     Running   7 (24m ago)   4h1m
gateway-67977d88f8-ttr74           2/2     Running   0             3h39m
my-kafka-0                         1/1     Running   0             6h50m
my-kafka-zookeeper-0               1/1     Running   0             6h50m
notimgmt-65d6c5bf45-cx5bw          2/2     Running   7 (23m ago)   3h27m
reservationhist-574585f56d-npmtc   2/2     Running   7 (23m ago)   3h27m
reservationmgmt-6dbf9c9449-mp2gh   2/2     Running   7 (23m ago)   4h18m
reservationmgmt-76bc697969-fmcdj   1/2     Running   0             8s
schedulemgmt-5f69f79b6f-x2fjk      2/2     Running   7 (23m ago)   3h26m
siege                              1/1     Running   0             5h22m
```
* siege 최종 결과 확인 --> Availability 100% 확인
```
Lifting the server siege...
Transactions:                   1386 hits
Availability:                 100.00 %
Elapsed time:                  59.02 secs
Data transferred:               0.61 MB
Response time:                  0.04 secs
Transaction rate:              23.48 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                    0.92
Successful transactions:        1386
Failed transactions:               0
Longest transaction:            0.87
Shortest transaction:           0.01
```
---

### Autoscale (HPA) ###
* 성수기에는 특정 기능(항공편 예약)에 대한 트래픽이 폭발적으로 증가할 수 있다. scalability가 요구되는 서비스에 대해서 auto scale 설정
```
gitpod /workspace/msaair (main) $ kubectl get hpa
NAME              REFERENCE                    TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
reservationmgmt   Deployment/reservationmgmt   3%/30%    1         5         1          4h47m
```
* siege 서비스를 이용하여 대상 서비스에 대해서 부하 발생 처리시 scale up이 발생하는 것 확인
```
root@siege:/# siege -c20 -t40S -v abc9cccd4c17d41af9bf9e37f59548a2-256813655.ap-southeast-2.elb.amazonaws.com:8080/reservations/
HTTP/1.1 200     5.75 secs:     460 bytes ==> GET  /reservations/
HTTP/1.1 200     9.67 secs:     460 bytes ==> GET  /reservations/
HTTP/1.1 200    10.34 secs:     460 bytes ==> GET  /reservations/
HTTP/1.1 200     4.11 secs:     460 bytes ==> GET  /reservations/

Lifting the server siege...
Transactions:                   2303 hits
Availability:                 100.00 %
Elapsed time:                  41.97 secs
Data transferred:               1.01 MB
Response time:                  0.28 secs
Transaction rate:              54.87 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                   15.29
Successful transactions:        2313
Failed transactions:               0
Longest transaction:           11.73
Shortest transaction:           0.02

gitpod /workspace/msaair (main) $ kubectl get pods
NAME                               READY   STATUS    RESTARTS      AGE
customermgmt-98c4cfcfc-82nk5       1/2     Running   1 (64s ago)   3h24m
gateway-67977d88f8-ttr74           2/2     Running   0             3h2m
my-kafka-0                         1/1     Running   0             6h13m
my-kafka-zookeeper-0               1/1     Running   0             6h13m
notimgmt-65d6c5bf45-cx5bw          1/2     Running   1 (90s ago)   170m
reservationhist-574585f56d-npmtc   1/2     Running   1 (87s ago)   170m
reservationmgmt-6dbf9c9449-mp2gh   1/2     Running   1 (90s ago)   3h41m
reservationmgmt-6dbf9c9449-nv699   1/2     Running   0             104s
reservationmgmt-6dbf9c9449-pnjqb   1/2     Running   0             119s
reservationmgmt-6dbf9c9449-ptk49   1/2     Running   0             119s
reservationmgmt-6dbf9c9449-rbhzk   2/2     Running   0             119s
schedulemgmt-5f69f79b6f-x2fjk      1/2     Running   1 (88s ago)   169m
siege                              1/1     Running   0             4h45m
```

---
### Persistence Volume/ConfigMap/Secret

* **persistent volume**
```
gitpod /workspace/msaair (main) $ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                 STORAGECLASS   REASON   AGE
air-volume                                 1Gi        RWX            Recycle          Bound    default/air-claim                                     gp2                     154m
pvc-315b6349-78cd-4d8d-aa03-26095682a9b3   8Gi        RWO            Delete           Bound    default/data-my-kafka-0                               gp2                     4h4m
pvc-8f4eb224-ccdb-48d6-b88c-dd9369f2351f   30Gi       RWO            Delete           Bound    logging/elasticsearch-master-elasticsearch-master-0   gp2                     125m
pvc-b4b4a929-a4da-46b1-93b7-d6c4bff4a0d6   8Gi        RWO            Delete           Bound    default/data-my-kafka-zookeeper-0                     gp2                     4h4m
```

* **persistent volume claim**
```
gitpod /workspace/msaair (main) $ kubectl get pvc
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
air-claim                   Bound    air-volume                                 1Gi        RWX            gp2            153m
data-my-kafka-0             Bound    pvc-315b6349-78cd-4d8d-aa03-26095682a9b3   8Gi        RWO            gp2            4h6m
data-my-kafka-zookeeper-0   Bound    pvc-b4b4a929-a4da-46b1-93b7-d6c4bff4a0d6   8Gi        RWO            gp2            4h6m
```

* **mount to schedulemgmt**
```
gitpod /workspace/msaair (main) $ kubectl describe deployment schedulemgmt
Name:                   schedulemgmt
Namespace:              default
CreationTimestamp:      Tue, 07 Mar 2023 03:54:18 +0000
Labels:                 app=schedulemgmt
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=schedulemgmt
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=schedulemgmt
  Containers:
   schedulemgmt:
    Image:        owljw/schedulemgmt:1
    Port:         8080/TCP
    Host Port:    0/TCP
    Liveness:     http-get http://:8080/actuator/health delay=120s timeout=2s period=5s #success=1 #failure=5
    Readiness:    http-get http://:8080/actuator/health delay=10s timeout=2s period=5s #success=1 #failure=10
    Environment:  <none>
    Mounts:
      /tmp/air from air-volume (rw)
  Volumes:
   air-volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  air-claim
    ReadOnly:   false
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   schedulemgmt-5f69f79b6f (1/1 replicas created)
Events:          <none>
```
---
### Apply Service Mesh : istio-gateway ###

istio 설치후, Microservice가 설치된 namespace(default)의 istio-enabled 설정후 재기동 수행
재기동 후 sidecar 정상 탑재확인
![image](https://user-images.githubusercontent.com/24615790/223364367-b316a6a9-b4be-4aa1-b9dd-0f929d5c81a8.png)

---
### Loggregation / Monitoring
EFK stack 설치후 모니터링 수행
![image](https://user-images.githubusercontent.com/24615790/223363581-5ad19711-b855-4ee8-a03b-0052ad88c6a7.png)
