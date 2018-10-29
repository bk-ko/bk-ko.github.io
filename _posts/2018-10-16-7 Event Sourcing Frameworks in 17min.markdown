---
layout: post
title: "Event Sourcing Frameworks in 7min. feat JVM "
description: "Event Sourcing Frameworks 에 관한 글입니다."
date:  2018-10-16
tags: [eda, eventsourcing, concept, kafka, eventually]
comments: true
share: false
---
# Event Sourcing Frameworks in 7min. feat JVM 



> #### Terminology
>
> `Event sourcing`,  `Snapshot `,  `SAGA`,  `Eventual Consistency`,  `CQRS`

## AKKA

![](https://akka.io/resources/images/akka_full_color.svg)

* akka-actors

  * ![](https://doc.akka.io/docs/akka/2.5/guide/diagrams/serialized_timeline_invariants.png)
  * [행위자 모델](https://ko.wikipedia.org/wiki/%ED%96%89%EC%9C%84%EC%9E%90_%EB%AA%A8%EB%8D%B8)
    * Erlang, ericsson AXD301 nine, nine; [history](https://techblog.vn/actor-model)
  * Actor contains
    * A mailbox (the queue where messages end up).
    * A behavior (the state of the actor, internal variables etc.).
    * Messages (pieces of data representing a signal, similar to method calls and their parameters).
    * An execution environment (the machinery that takes actors that have messages to react to and invokes their message handling code).
    * An address
  * ![](https://doc.akka.io/docs/akka/2.5/guide/diagrams/actor_top_tree.png)

* Akka persistence, Akka persistence Query, FSM(SAGA)

  * Akka persistence enables stateful actors to persist their internal state so that it can be recovered when an actor is started

  * ```java
    class AccountPersistentActor extends AbstractPersistentActor {
    ...
        @Override
        public String persistenceId() {
            return persistenceId;
        }
    
        @Override
        public Receive createReceiveRecover() {
            return receiveBuilder()
                .match(Event.class, e -> state.update(e))
                .match(SnapshotOffer.class, 
                       ss -> state = (AccountState) ss.snapshot())
                .build();
        }
    
        @Override
        public Receive createReceive() {
            return receiveBuilder()
                .match(Command.class, c -> {
                    final String desc = "[" + getNumEvents() + "]";
                    final Event evt = new Event(c.getType(), c.getAmount(), desc);
                    persist(evt, (Event event) -> {
                        state.update(event);
                        getContext().system().eventStream().publish(event);
                    });
                })
                .matchEquals("snap", s -> saveSnapshot(state.copy()))
                .matchEquals("print", s -> System.out.println(state))
                .build();
        }
    
    }
    
    public static void main(String... args) throws Exception {
            final ActorSystem system = ActorSystem.create("example");
            final ActorRef persistentActor = system.actorOf(
                Props.create(AccountPersistentActor.class, "bk.ko"), 
                "persistentActor-4-java8");
            persistentActor.tell(new Command(DEPOSIT, 100), null);
            persistentActor.tell(new Command(DEPOSIT, 200), null);
            persistentActor.tell(new Command(DEPOSIT, 300), null);
            persistentActor.tell("snap", null);
            persistentActor.tell(new Command(WITHDRAW, 400), null);
            persistentActor.tell("print", null);
    
            Thread.sleep(10000);
            system.terminate();
        }
    ```

  * AKKA 의 특징인 병렬 수행, 메시지 기반 통신을 활용

  * Scala 의 Eventsourced 라이브러리 에서 파생

  * Java Serialization 보다는 Protobuf 를 권장

  * level db - default

  * 장점

    * Eventsourcing의 모든 구현이 제공됨 (DLQ, Recovery)

  * 단점

    * 결국 Scala



 * ![Alt text](https://monosnap.com/image/yVccu1AIc94Ry4AKovCIwMD6x44pRP.png)




## LAGOM

![](https://i2.wp.com/blog.knoldus.com/wp-content/uploads/2017/03/logom.png?resize=752%2C192&ssl=1)





![](https://d3ansictanv2wj.cloudfront.net/drms_0501-69c9d37c376d632b7dcef06451085d58.png)

* AKKA  + Paly + Cassandra + Kafka +….(Guice, Jackson)

  * akka streams : backpressure -  java9 flow 

* Microservices 를 위한 딱 알맞는(lagom) 프레임워크.

* Maven, SBT, no no gradle plugin


## Spring Cloud Stream

![](https://cloud.spring.io/spring-cloud-stream/img/SCSt-overview.png)

* ```java
  @SpringBootApplication
  @EnableBinding(Sink.class)
  public class LoggingConsumerApplication {
  
  	public static void main(String[] args) {
  		SpringApplication.run(LoggingConsumerApplication.class, args);
  	}
  
  	@StreamListener(Sink.INPUT)
  	public void handle(Person person) {
  		System.out.println("Received: " + person);
  	}
  
  	public static class Person {
  		private String name;
  		public String getName() {
  			return name;
  		}
  		public void setName(String name) {
  			this.name = name;
  		}
  		public String toString() {
  			return this.name;
  		}
  	}
  }
  
  ```

* `@EnableBinding({Source.class, Sink.class})`

* 미들 웨어 독립성

* Spring Integration (Message) + test

* Spring Cloud !

  * Avro 지원 (not Confluent)

* ES의 개념에 대한 구현이 없음 , Snapshot, Aggregate, Command, Event (장점 & 단점)



## Axon Framwork

![](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-L9ehuf5wnTxVIo9Rle6%2F-LF1MF4RtQNtSyAx0VL4%2F-L9eiEg8i2dLcK2ovEZU%2Fdetailed-architecture-overview.png?generation=1529048013327425&alt=media)

* 7년차 Framework

* 기본적으로 성숙한 구현 제공

  * validated request

    ```java
    return commandGateway.send(createArticle)
                         .thenApply(id -> …);
    ```

  * ```java
    @Aggregate
    public class Account {
    
        @AggregateIdentifier
        private UUID accountId;
    
        private Double balance;
    
        // Required for Axon to re-create the aggregate
        public Account() {
        }
    
        @CommandHandler
        public Account(CreateAccountCommand command) {
            apply(new AccountCreatedEvent(command.getAccountId(), command.getName()));
        }
    
        @CommandHandler
        public void handle(DepositMoneyCommand command) {
            apply(new MoneyDepositedEvent(command.getAccountId(), command.getAmount()));
        }
    
        @CommandHandler
        public void handle(WithdrawMoneyCommand command) {
            if (balance - command.getAmount() >= 0) {
                apply(new MoneyWithdrawnEvent(command.getAccountId(), command.getAmount()));
            } else {
                throw new IllegalArgumentException("Amount to withdraw is bigger than current balance on account");
            }
        }
    
        @EventSourcingHandler
        protected void on(AccountCreatedEvent event) {
            this.accountId = event.getAccountId();
            this.balance = 0.0;
        }
    
        @EventSourcingHandler
        protected void on(MoneyDepositedEvent event) {
            this.balance = balance + event.getAmount();
        }
    
        @EventSourcingHandler
        protected void on(MoneyWithdrawnEvent event) {
            this.balance = balance - event.getAmount();
        }
    
    }
    ```

* Annotation!! 

* Spring boot starter 제공, 

  * https://springoneplatform.io/2018/sessions/bootiful-cqrs-and-event-sourcing-with-axon-framework

* CQRS oriented Framework

* SAGA, Snapshot, Fullset

* no Message Queue;



## Kafka Interactive Queries

![](https://knoldus.files.wordpress.com/2018/05/remote-state-stores.png?w=720)

* ben stopford  예제
* kafka partition에 종속적
* Kafka 라는  Single source of truth 가 존재
* RPC Layer 가 문제인가? fallback 구현?
* `num.stream.threads` 에 따른 지각변동 , 문제인가?
* Avro, Json, String
* Data locality, 관리
  * ㅡ,.ㅡ;;;;



## Eventuate

![](https://eventuate.io/i/logo.gif)

![](https://raw.githubusercontent.com/eventuate-local/eventuate-local/master/i/Eventuate%20Local%20Big%20Picture.png)

* link https://github.com/eventuate-local/eventuate-local
* Chris Rischardson 님
* 돌아가는 구현이 SAAS 로 제공 되고 있음
  * Eventuate™ SaaS runs on AWS. All you need to use this version is an API key that you obtain by [signing up](https://signup.eventuate.io/).
* CDC base (mysql + kafka connect)
* Orchestration SAGA
* Json Serialization
* ES의 모든 구현
  * 너무!! 많은 예제




> **double-entry booking**
>
> * https://en.wikipedia.org/wiki/Double-entry_bookkeeping_system
> * 현대의 방식과 비슷한 최초의 복식회계는 13세기 말 피렌체 상인인 아마티노 마누치(Amatino Manucci)의 기록에서 발견된다. 다른 오래된 기록은 1299-1300 경 파롤피(Farolfi)의 거래원장에서 발견된다. 지오바니노 파롤피 앤 컴퍼니(Giovanino Farolfi & Company)는 피렌체 상인들의 회사였는데, 본사는 니메스(Nîmes)에 있었으며 그들은 아를의 대주교를 가장 큰 고객으로 둔 대금업자였다. 어떤 기록에 따르면, 지오바니 디 빈치 데 메디치(Giovanni di Bicci de' Medici)가 이 방법을 14세기에 메디치 은행에 소개했다고 한다.

![](https://t1.daumcdn.net/cfile/tistory/24244C345741C06830)



[***" Events are How the Real World Works."***](https://skp-devinnovation.slack.com/archives/D3M2K98JJ/p1539647207000100)
















