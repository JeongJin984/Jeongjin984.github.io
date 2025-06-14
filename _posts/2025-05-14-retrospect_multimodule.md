---
title: 멀티모듈 설계에 관하여
description: >-
  멀티 모듈 설계에 관한 회고록
author: jay
date: 2025-05-14 20:55:00 +0800
categories: [Retrospect, Geeson]
tags: [gradle, architecture]
pin: false
media_subpath: ''
---

## 🔷 개요

프로젝트를 본격적으로 시작하기 전 gradle을 활용한 멀티 모듈 설계를 진행하고 설계 결과를 분석하여 생길 수 있는 문제와 이를 해결할 수 있는 방안을 찾아본다.

**분석 방법**

1. chat-gpt
2. 여러 기술 블로그
3. 검색

## 🔷 문제 분석

### **관점**

1. 멀티 모듈은 라이브러리의 개념으로 구현 기술의 종속성에 따라 각각의 모듈을 설계

---

### **초기 설계 결과**

- api : 웹 구현, API 스펙 구현
  - inventory-api
  - order-api
  - ...
- domain : 도메인 로직 구현
  - inventory-app
  - order-app
  - ...
- infra : 인프라 관련 설정 및 종속 기술 구현
  - inventory-db
  - order-db
  - ...
  - kafka
- support : 유틸 관련 기능 구현
  - logging
  - messaging

이런 식으로 멀티 모듈을 설계하여 위에서 아래(api -> domain -> infra)로 의존하도록 설계

- Kafka Message Produce 및 consuming을 도메인 로직에 포함
- JPA는 db-layer에서 구현

---

### **문제점**

1. 웹, Kafka DB등 여러 기술 모듈이 내부 구현을 알아야 동작함
  - 각각의 모듈이 기술에 강하게 결합된다는 뜻
2. 비즈니스 로직이 기술에 종속됨
  - JPA에 강하게 결합되어 구현체 혹은 Infra가 바뀌었을 때 대응하기가 힘듬(도메인 로직에 jpaRepository를 직접 사용하며 강하게 결합)
  - 단위 테스트가 어려움(단위 테스트가 반드시 인프라 자원을 필요, Infra에 Entity가 구현되어 있으니까)
3. 의존성 폭발
  - 도메인 경계를 무너뜨리는 암묵적 의존성
  - 모듈간 책임 흐름이 명확하지 않음
  - 순환 의존 또는 트랜잭션 관리 혼란 발생 위험

## 🔷 문제 해결

### 설계 개선안 개요

1. JPA Entity는 domain 로직에 구현
2. Infra 자원을 필요로 할 경우(Repository, Messaging) Port를 Application layer에 정의하면서 구현은 InfraLayer에서 구현
3. Kafka 또한 도메인 별로 별도로 모듈 분리

---

### JPA Entity에 대한 고찰

**의문점**
도메인 로직에 Entity를 구현하는 것은 JPA에 강하게 의존하는 것이 아닌가

**결론**
의존이 나쁜건 아니다. 사실 의존이라는 것은 '어디에서 허용할 것'인가를 정하는 것이 더욱 중요한 문제이다.

Entity를 붙이는 순간 해당 클래스는 JPA에 종속된 '도메인 객체'가 된다. 즉 순수 도메인 모델이 아닌 것이다.

물론 순수한 도메인 모델을 완전히 분리하여 사용하면 이상적이겠지만 경험상 이런 식의 접근은 굉장히 많은 공수(Mapper 구현, 도메인 객체 설계 및 구현)가 들어가게 된다.  따라서 JPA를 의존하여 doamin layer를 구현하였다.

| 전략 | 설명 |
| --- | --- |
| ✅ **JPA 의존을 허용한 domain layer** | 실용적인 방법. `@Entity`를 도메인에 포함시키고 JPA 의존을 받아들임 |
| ✅ **Port-Adapter 패턴으로 JPA Entity와 Domain Model 분리** | 이상적인 방법. 도메인 모델은 JPA를 모르고, 매핑 객체(`JpaEntity`)를 따로 둠 |
| ❌ Entity를 infra에 둠 | 잘못된 접근. 도메인이 infra에 의존하게 되어 아키텍처가 무너짐 |

결론적으로 Entity는 다음의 특징을 가지기 때문에

- 비즈니스의 개념을 표현
- 다른 기술로 전환하더라도 클래스 자체는 의미를 유지할 수 있기 때문(물론 @Entity 이런건 바꿔야 하겠지만)
  domain layer에 구현하는 것에 맞다고 생각한다.

---

### Application Layer에 대한 고찰

**개념**

> 유스케이스의 흐름(기능 흐름)을 담당하는 계층
>
> - 도메인 로직을 조합하여 위부 요청 또는 이벤트에 대응하는 계층

```
[Interface / Presentation Layer] ← Web Controller (@RestController)
         ↓
[Application Layer]              ← 유스케이스 흐름 제어 (@Service)
         ↓
[Domain Layer]                   ← 비즈니스 규칙, Entity, ValueObject, DomainService
         ↓
[Infrastructure Layer]           ← 기술 구현체 (DB, Kafka, 외부 API)

```

**도메인 서비스와의 차이점**

| 항목 | Application Service | Domain Service |
| --- | --- | --- |
| 🎯 목적 | 유스케이스 흐름 조립 (시나리오 담당) | 복잡한 도메인 규칙 캡슐화 |
| 🧩 위치 | Application Layer | Domain Layer |
| 📦 책임 | 도메인 객체 호출/조합, 트랜잭션, 외부 시스템 연계 | 도메인 객체로 표현하기 어려운 도메인 규칙 수행 |
| 🔄 의존성 | 도메인 객체/서비스, Repository, 인프라에 의존 가능 | Entity, Value Object에만 의존 (가능한 한 순수) |
| 💡 예시 | 주문 생성, 결제 승인, 출고 처리 등 | 할인 계산, 수수료 계산, 정책 판별 등 |

**왜 Application Layer에 Port를 구현해야 하는가**

| 관점 | 이유 |
| --- | --- |
| 🎯 **아키텍처 원칙** | 외부 기술에 대한 의존을 인터페이스(Port)로 추상화하여 DIP(의존성 역전 원칙)를 지킴 |
| 🧪 **단위 테스트 용이성** | Repository 구현체(JPA 등)에 의존하지 않고, Port 인터페이스를 모킹(mock)함으로써 **Application Layer 테스트를 독립적으로 수행 가능** |
| 🔁 **유연한 구현 교체** | DB, 캐시, 외부 API 등 구현을 변경해도 테스트 코드나 UseCase 로직은 수정할 필요 없음 |
| 🔌 **Infra 격리** | 테스트에서 실제 DB를 띄우지 않아도 되므로 빠르고 안정적인 테스트 가능 |

---

### 의존성 폭발 문제 해결

아래의 Flow로 이벤트가 처리되면서

```
[order-application]
   ↓ (이벤트를 트리거)
[infra-kafka-payment-consumer]
   ↓ (payment service 호출)
[payment-application]
   ↓
[payment-infra → payment-db]
```

아래와 같은 문제 발생

- 도메인 경계를 무너뜨리는 암묵적 의존성
- 모듈간 책임 흐름이 명확하지 않음
- 순환 의존 또는 트랜잭션 관리 혼란 발생 위험

이에 따라 Kafka 모듈을 도메인 별로 분리

## 🔷 구현 결과

### 모듈 리스트

- api
  - inventory-api : 재고 API
  - order-api : 주문 API
  - payment-api : 결제 API
  - product-api : 상품 API
- application
  - order-app
  - payment-app
- domain
  - inventory-domain : 재고 서비스
  - order-domain : 주문 서비스
  - payment-domain : 결제 서비스
  - product-domain : 상품 서비스
- infra
  - rdb
    - inventory-db : 재고 DB
    - order-db : 주문 DB
    - payment-db : 결제 DB
    - product-db : 상품 DB
  - queue
    - inventory-kafka : Kafka 설정 및 Producer, Consumer 구현(이벤트 발행)
    - order-kafka
    - payment-kafka
    - product-kafka
- module
  - client : 외부 종속성(외부 API등) 모듈
  - enum : 공통 enum 관리 모듈
  - file : 정적 파일 관련 모듈
- support
  - logging : 로그 관련 유틸
  - messaging : 이벤트(카프카) 관련 유틸(이벤트 객체 등)
- commander : UI 모듈
