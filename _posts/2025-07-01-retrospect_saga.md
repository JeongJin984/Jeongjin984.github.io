---
title: Saga Orchestrator에 대해서
description: >-
  Saga Orchestrator에 대해서
author: jay
date: 2025-05-18 20:55:00 +0800
categories: [Retrospect, Geeson]
tags: [kafka, saga, spring]
pin: false
media_subpath: ''
---

# Saga Orchestrator

## Monolithic vs MSA

**Monolithic**

Monolithic의 최대 장점은 Transaction, Component(프로세스, 서버)를 관리하기 쉽다는 것이다. 중간에 뭐가 잘못되었다면 DB에서 지원하는 Transantion 관리 기능을 사용하면 된다. 그러면 Rollback도 쉽다. 

하지만 장점이 뚜렷한 만큼 단점도 뚜렷하다. 보통 일반적인 서비스는 지속적으로 확장되고 변경된다. 초기에는 큰 문제가 되지 않지만 이것 저것 서비스에 기능들을 추가하게 되면 점점 서비스는 무거워 지고 유지보수가 어렵게 된다. 게다가 하나의 장애가 여러 다른 기능들에 전파될 수 있는 여지도 충분히 있다.

**MSA**

MSA에 대해서는 사실 최근까지 안좋은 인식이 있었던 것이 사실이다. 왜냐하면 MSA는 그만큼 도메인에 대한 설계도 잘 되어있어야할 뿐아니라 각 컴포넌트를 관리하는 기술(대포적으로 k8s)에 대한 이해도도 높아야한다. 그리고 하나의 Transaction을 분리한다는 것은 그만큼 서비스가 불안정해 진다는 것이다. 따라서 굳이 이런 단점을 가져가면서도 MSA는 해야하는 이유에 대해서 의심스러웠다.

하지만 이번 프로젝트를 통해 어렴풋이나마 필요성에 대해서 느끼게 되었다. MSA의 최대 장점은 서비스를 분리하고 Kafka같은 기술을 활용해서 서비스를 서로간에 약하게 결합한다는 것이다. 약하게 결합된 서비스는 서비스를 확장시키는 것에 대해서 굉장한 편의성을 제공한다. 단순히 이벤트를 구독하고 소비하는 것으로 신규 서비스를 만들
수 있을 것이다.

## Saga Orchestration 
  
이번 프로젝트는 MSA 아키텍처를 구성하는 것이기 때문에 분산된 트랜잭션을 컨트롤 할 필요가 있다. 내가 선택한 시나리오는 주문 -> 결제 -> 재고 의 과정에서의 분산된 트랜잭션을 관리하는 것이 목표였다.

이를 해결하기 위해 3가지 방법이 있는데 Two-Phase Commit(2PC), 보상 트랜잭션 기반의 Saga 패턴(Orchestration(중앙 조정자), Choreography(각 서비스가 이벤트 기반으로 반응))이 있다. 

### 2PC

이 방법은 모든 참여자에 prepare 요청을 하고 모두 OK하면 Saga를 허용하는 것이다. 이 방법은 모든 참여 시스템으로 동기적으로 연결되어야 한다는 특징이 있는데 사실 가장 간단하다 즉 장애를 허용하지 않겠다는 것인데 사실 MSA에서는 하나의 Saga에 참여하고 있는 Component가 한두개도 아니고 그 중에서는 작동하지 않아도 괜찮은(내부 알럿같은) Component가 있을 수 있고 결국 확인 -> commit의 과정을 거치는데 확인 과정에서 문제가 없다가 commit에서 문제가 발생할 수 있는 여지는 충분히 있다.

그리고 하나의 Component가 응답이 느리면 모든 서비스가 응답이 느려지기 때문에 굉장히 비효율 적이라고 생각한다.

이럴거면 그냥 차라리 그 Saga를 위한 하나의 서버를 구성하는 것이 훨씬 효율적일 수 있다는 생각을 했다.

### Choreography

이 방식은 각 서비스가 이벤트로 반응하며 Saga를 컨트롤 하는데 이 방식은 전체적인 흐름이 잘 보이지가 않는다. MSA의 각 Architecure의 독립성이라는 목적에는 부합하지만 이때문에 전체적인 흐름을 한눈에 파악하기 어렵고 장애나 지연이 발생하면 디버깅이 힘들다

물론 각 서비스를 느슨하게 결합한다는 원칙을 지키고 있기 때문에 서비스간의 독립성을 보장하고 확장성과 유지 보수성이 우수하며 탄력성이 높다는 점에서 MSA의 정의의 관점에서는 더욱 적합한 방식으로 판단 할 수 있다.

### Orchestration

이 방법은 Saga를 처리하는 중앙화된 조정자가 따로 존재하고 그 조정자는 전체적인 Saga의 흐름을 관리하는 역할만 하는 것이다. 이 방법은 복잡한 로직을 중앙 집중적으로 관리 가능하다는 장점이 있지만 SPOF가 발생할 수 있는 여지가 있고 전체적인 서비스가 조정자 아래 결합도가 상대적으로 높아진다는 단점이 있다.

이 방법을 사용해서 Saga를 구현하였는데 MSA는 Monolithic에 비해서 장애에 취약하다. MSA는 굉장히 효율적이고 튼튼하다고 착각할 수 있다고 생각하는데 오히려 MSA는 비효율적이고(서비스가 간단하면 간단할수록) 장애에 취약하다. 

MSA에서는 오히려 장애를 원천적으로 막는 것 보다는 추적 및 복구가 가능한 서비스를 구성하는 것이 더욱 중요한 과제이다. 이러한 측면에서 복잡한 Saga 흐름을 구현하는 것은 Choreography 방식보다 Orchestration 방식이 더 견고한 시스템을 만드는데 적합하다고 판단했다.   

**정리하자면**

```markdown
🟢 Orchestrator 적합한 경우

전체 흐름이 복잡하거나 정형화되어 있는 경우 (예: 주문 → 결제 → 재고 → 배송)

실패 보상 로직이 복잡하고, 한눈에 통제할 필요가 있을 때

빠른 개발 및 단순한 운영/디버깅 환경이 중요한 경우

팀 역량상 tracing/log aggregation을 아직 준비하지 않았을 때
```

```markdown
🔵 Choreography + OpenTelemetry 적합한 경우

마이크로서비스 간 결합도를 낮추고자 할 때

이벤트 중심 아키텍처가 기존에 자리잡고 있을 때

서비스가 독립적으로 진화하고, 다양한 소비자가 생길 가능성이 높을 때

**관측성(Observability)**을 강화하려는 전략이 명확할 때
```

따라서 Backbone 시스템(주문, 결제, 재고, 정산)등의 시스템은 MSA의 독립성, 탄련석을 어느정도 포기하고 Orchestrator 기반으로 구현하고 결제 notification, 쿠폰 발급, 이메일 전송등의 Saga는 Choreography를 사용하는 것이 맞다고 판단했다.


## 분산 트랜잭션

> 그래서 분산 트랜잭션을 Orchestrator에서 어떻게 구현할 건데?

### Kafka

Kafka는 Http와는 다르게 응답을 기다리지 않는다. 이벤트는 발송되기만 할 뿐 그에 따른 응답을 얻을 순 없다. 모든 것이 비동기적이라는 말이다. Orchestrator는 Kafka의 어쩌면 단점일 수 있는 점을 해결하기 위해 성공과 실패를 이벤트로 수신한다. kafka 메시지를 command와 event로 나누어서 마치 Kafka가 동기식으로 운영되는 것 처럼 구현하고 있다.

동기식으로 운영되는 것을 기대하기 때문에 어쩌면 결합력이 높아졌다고 볼 수 있는데 이건 어쩔 수 없는 것 같다.(대신 Transaction 관리가 용이해 졌으니...)

### Statemachine

여기서 하나의 문제가 더 발생한다. 바로 보상 트랜잭션이다. 예를들어 결제가 성공하고 재고에서 실패가 발생했을 때 결제를 실패 처리 해줘야한다. 이러한 보상 트랜잭션을 섬세하게 다뤄야만 견고한 시스템을 만들 수 있다. 아래는 내가 작성한 시나리오이다.

![Build source](/assets/img/retrospect_saga_img1.png){: .light .border .normal .center w=600' h='200' }

**Saga의 상태가 이렇게 변하는데 Saga의 상태를 어떻게 관리해야할까?** 

처음에는 SagaInstance와 각 SagaStep(command 발행)을 DB에 저장하는 것으로 해결하고자 했다. 하지만 그럼에도 각 Listener에서 다음 이벤트를 발행하고 if문을 통해 분기를 처리하다 보니 결국 전체적인 흐름은 커녕 Orchestrator가 점점 복잡해 지며 디버깅이 힘들어 졌다. 

그래서 ChatGPT를 통해 방안을 모색한 결과 유한상태머신을 구현할 수 있는 Spring-Statement라는 라이브러리를 통해 이 문제를 해결할 수 있었다. statemachine은 전체적인 흐름을 Configuration을  각 이벤트에 따라 command를 발행하고 이벤트에 따라 어떤 state로 Transition 되어야 하는지 Configuration을 통해 명시할 수 있다. 아래는 위의 흐름을 Configuration을 통해 명시한 코드이다.

```java
@Override
public void configure(StateMachineTransitionConfigurer<OrderSagaState, OrderSagaEvent> transitions) throws Exception {
  transitions
    .withExternal()
    .source(ORDER_CREATED)
    .event(START_ORDER)
    .target(PAYMENT_REQUESTED)
    .action(paymentCommandGatewayGW.paymentRequestCommand())

    .and()
    .withExternal()
    .source(PAYMENT_REQUESTED)
    .event(PAYMENT_SUCCESS)
    .target(PAYMENT_COMPLETED)
    .action(inventoryCommandGateway.inventoryReserveCommand())

    .and()
    .withExternal()
    .source(PAYMENT_REQUESTED)
    .event(PAYMENT_FAILURE)
    .target(FAILED)

    .and()
    .withExternal()
    .source(PAYMENT_COMPLETED)
    .event(INVENTORY_SUCCESS)
    .target(INVENTORY_RESERVED)

    .and()
    .withExternal()
    .source(PAYMENT_COMPLETED)
    .event(INVENTORY_FAILURE)
    .target(COMPENSATING_PAYMENT)
    .action(paymentCommandGatewayGW.inventoryFailurePaymentCompensateCommand())

    .and()
    .withExternal()
    .source(COMPENSATING_PAYMENT)
    .event(PAYMENT_COMPENSATED)
    .target(COMPENSATING_INVENTORY)
    .action(inventoryCommandGateway.inventoryFailureInventoryCompensateCommand())

    .and()
    .withExternal()
    .source(COMPENSATING_INVENTORY)
    .event(INVENTORY_COMPENSATED)
    .target(COMPENSATED)

    .and()
    .withExternal()
    .source(COMPENSATING_PAYMENT)
    .event(PAYMENT_COMPENSATE_FAIL)
    .target(FAILED)
    .action(paymentCommandGatewayGW.paymentInventoryCompensateFailDLQ())

    .and()
    .withExternal()
    .source(COMPENSATING_INVENTORY)
    .event(INVENTORY_COMPENSATE_FAIL)
    .target(FAILED)
    .action(inventoryCommandGateway.inventoryInventoryCompensateFailDLQ())

    .and()
    .withExternal()
    .source(INVENTORY_RESERVED)
    .event(COMPLETE)
    .target(ORDER_COMPLETED);
}
```


