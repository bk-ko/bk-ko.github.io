---
layout: post
title: "Snapshots in Eventsourcing"
description: "Eventsourcing 에서 Snapshot에 관한 글입니다."
date:  2018-08-09
tags: [eda, eventsourcing, concept, kafka, eventually]
comments: true
share: true
---
# Snapshots in Eventsourcing

[TOC]

## Snapshot

eventsourcing 에서는 하나의 aggregate 라는 건 그 **생성시점 부터 모든 event를 합한 결과** 라는 사실은 잘 알려져 있습니다. 하지만 오래되고, 많은 event 가 누적되어 있는 aggregate 라면 한번의 조회뿐으로도 성능면에서 비효일적일 수 밖에 없을것이고, 그래서 eventsourcing 에서는 snapshot 이라는 중간 상태를 통해 이 부분을 해결하고 있습니다.

snapshot 이라는건 말그대로 aggregate의 현재 상태를 스냅샷 찍듯 저장하고, 다음번에 조회할때는 snapshot 찍은 그부분에서부터의 event 만 조회하여 성능이나, 오래된 event의 retention 문제를 해결할 수 있다는 개념입니다.

![Alt text](https://monosnap.com/image/CWqMsUJBCOeq51iw5PINVXnHcxOl2J.png)

이제부터 eventsourcing의 snapshot 에 대해 더 알아 보겠습니다.

### 조회, Replay

eventsourcing에서는 **event란 불변하는 확고한 하나의 소스**로서 쌓여있고, 원하는 aggregate를 얻을때는 그저 기존의 event만 replay하면 된다는 말을 하는데 code로 한번 다시 보겠습니다.

Account와 Snapshot의 대략적인 모습입니다. `TODO :  그림 그려야함`

![Alt text](https://monosnap.com/image/yzJgwtyyHU2NAHRxlJTYCn7sbci4O4.png)

```java
// QueryService.java
public Account getAccount(String accountId) {
     // 최근의 snapshot이 존재하는지 확인
     Account account = recreateAccountFromSnapshot(accountId);

     // 모든 event 조회
     List<AccountEvent> accountEvents =
         repository.findAllByEntityId(accountId);

     // replay 를 통해 최신의 aggregate로 재구성
     return account.replay(accountEvents);
 }

// Account.java
public Account replay(List<AccountEvent> events) {
        events.stream()
              // snapshot 된 version(eventId) 이후 event만을 Filter
              .filter(e -> e.getEventId() > eventId)
              // apply -> state transition
              .forEach(e -> apply(this, e));

        this.events = events;
        return this;
    }
```

위의 sample 처럼 만약 계좌정보(Account)를 조회한다면

- 기존의 snapshot을 조회하고 (만약 기존 snapshot이 있다면 가장 최근 snapshot의 aggregate된 계좌정보가 있을겁니다.)

- 전체 event를 조회해서 (snapshot의 event version 이후만 조회해올 수 도 있겠죠? )

- snapshot 이후의 event만을 apply 하여 최신 상태를 구한다.

  - 이때 apply 는 상태(state)의 전이(transition)이지 action이 다시 일어나서는 안됩니다.

    ( 매우중요, 메일 발송을 생각하시면 됩니다.)

이전이라면 DB에 한번 물어보면 되던 일이 이렇게 복잡해졌다는게 놀랍지만, 그럼에도 불구하고 eventsourcing의 장점도 있습니다.

## KAFKA STREAM 에서의 Snapshot

KAFKA topic을 event store 로 사용하는 경우도 살펴보겠습니다. 우선 간단히 얘기하면 KAFKA (STREAM) 에서는 굳이 snapshot 이라는 개념이 필요 없습니다. 매번 흘러 들어오는 하나의 event를 aggregate 하여 고유의 Key Value Store를 생성한다거나, 아니면 aggregate 한 결과를 별도의 Topic 또는 noSql 이나 RDBMS 로 저장할수 있기 때문에, 마치 snapshot이 매번 생성되는 것과 같은 QueryService를 구축 할 수 있습니다. (이부분은 Materialized View 라는 말이 제일 맞는거 같지만, Global View, Memory View 여러가지 표현이 있습니다. )

![event-sourcing-cqrs](https://cdn-images-1.medium.com/max/1600/1*jyahaf0wO27PI52k_VqpQw.png)

[event-sourcing-cqrs-stream-processing](https://www.confluent.io/blog/event-sourcing-cqrs-stream-processing-apache-kafka-whats-connection/)



snapshot 이라는 것에 관해 살펴 봤고, 이것은 event sourcing에서 일부일 뿐입니다.

요새는 DDD,  Event Driven, CQRS, Eventsourcing 등 뭔가 어울리지만 *서로 다른* 여러가지 개념들이 섞이면서 이런 저런 어려움이 많은거 같습니다.

### 참고자료

- [Microservices Patterns: With examples in Java](https://www.amazon.com/Microservices-Patterns-examples-Chris-Richardson/dp/1617294543/ref=sr_1_1?s=books&ie=UTF8&qid=1533730373&sr=1-1&keywords=Microservices+Patterns%3A+With+examples+in+Java) by [Chris Richardson](https://www.amazon.com/s/ref=dp_byline_sr_book_1?ie=UTF8&text=Chris+Richardson&search-alias=books&field-author=Chris+Richardson&sort=relevancerank)

## FAQ

꼭 snapshot 관련된건 아니지만 생각해본 이슈들입니다.

### 만약 Event 가 발생하는데 Snapshot 생성하는 시간이랑 겹치면 어떡하나요?

이 질문은 단지 snapshot 뿐만 아니라 모든 종류의 지연과 관련된 이야기 같습니다. 그렇게 생각한다면, 결국 하나의 store에 순서대로 log를 쌓고 그 순서에 따라 처리하되, 만약 실패한  Command에 대해서는 보상 처리한다, 라는 SAGA 와 같은 상태관리에 관한 이야기와 같다고 생각됩니다. 참고로 로그를 통한 분산 트랜잭션 처리에 관한 글입니다.  [logs-for-data-infrastructure](http://martin.kleppmann.com/2015/05/27/logs-for-data-infrastructure.html)

### eventsourcing을 하면 schema evolution이 어려울것 같은데...

개념적으로는 eventstore는 영원히 보존하는것, 이라는 말을 듣습니다. 하지만 당연히 aggregate는 변하기 마련이고, 일반적인 필드, aggregate의 추가의 경우는 문제 없지만, 아래의 표로 정리를 한것과 같이 하위 호환이 안되는 case 들이 있습니다. 이는 기존의 SQL에서의 schema migration과 비슷한 경우라고 생각할 수 있을겁니다. eventsourcing에서는 **upcast** 라는 접근방식으로 개별의 event를 다시 재생산 하는 방식도 존재한다고 합니다.

![Alt text](https://monosnap.com/image/CLEz9hytyU2s70tmnzObej3coWifGm.png)

















