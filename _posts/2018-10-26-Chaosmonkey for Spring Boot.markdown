---
layout: post
title: "Chaos Monkey for Spring Boot"
description: "Chaos Monkey for Spring Boot에 관한 글입니다."
date:  2018-10-26
tags: [msa, chaos engineering, concept, resilient]
comments: true
share: true
---
# Chaos Monkey for Spring Boot

![logo](https://codecentric.github.io/chaos-monkey-spring-boot/images/sb-chaos-monkey-logo.png)

> 참고 자료
>
> - PRINCIPLES OF CHAOS ENGINEERING [link](http://channy.creation.net/blog/1173) 
>   - O'REILLY Chaos Engineering [pdf](http://channyblog.s3-ap-northeast-2.amazonaws.com/data/channy/2018/01/18023151/chaos-engineering.pdf)
> -  Netflix blog
>   - FIT: Failure Injection Testing [link](https://medium.com/netflix-techblog/fit-failure-injection-testing-35d8e2a9bb2)
>      - FIT : 가상의 장애를 주입하여 실제 서버가 어떻게 작동하는지 확인하려고 만듬
>      - ZUUL 헤더에 메타데이터를 주입하여 Fault 인경우 의도한 에러 발생
>      - 실패 주입점 :  Hystrix,Ribbon, EVCache, Astyanax 등을 통해 전체 시스템을 탄력있게 개선
>   - Automated Failure Testing [link](https://medium.com/netflix-techblog/automated-failure-testing-86c1b8bc841f)
>      - 어디에 주입해야 할지 찾는 방법, 자동화된 실패 주입,
>      - 각 실험은 요청 클래스로 범위가 지정되며 단시간 (20-30 초) 동안 실행되어 초당 실패 수 확인, 75% 이상 실패 하는가?
>      - 일 500회 실시 최악 의 경우 5,000명이 에러를 볼것으로 기대; 정상 요청은 수억건;;
>   - The Netflix Simian Army [link](https://medium.com/netflix-techblog/the-netflix-simian-army-16e57fbab116)
>      - Latency Monkey : 네트워크 레이어 지연
>      - Conformity Monkey : Autoscaling 하지 않는 서버군의 에러발생 → 재시작을 제대로 하는지        
>      - Doctor Monkey : CPU등 인스턴스 상태 모니터링, 알람, 비정상 앱 종료
>      - Janitor Monkey : 클러스터 자원이 낭비되지 않도록 검사하여 폐기
>      - Security Monkey : 보안 위반, AWS 상의 취약점 점검, 폐기
>      - 10–18 Monkey : Localization-Internationalization 검사
>      - Chaos Gorilla :  Chaos Monkey for Amazon availability zone,  re-balance

## Chaos Engineering 이란

> ### 카오스 엔지니어링(Chaos Engineering)이란 프로덕션 서비스의 각종 장애 조건을 견딜 수 있는 시스템의 신뢰성을 확보하기 위해 분산 시스템을 실험 하고 배우는 분야입니다.

위의 넷플릭스 블로그를 읽어 보면 알겠지만 Microservices 아키텍처 인 시스템을 실제 프로덕션 상황에서 임의의 장애, 지연, 서버다운 을 일으켜, 전체 MSA 시스템이 탄력있게 반응하는지 확인하는 Fault Injection Test(FIT) 에서 유래한다고 합니다. Chaos monkey 라는 이름으로 유명하지만, Spring Boot 에서 이를 쉽게 할수 있는 라이브러리가 있어 살펴 보겠습니다.

## Chaos Monkey for Spring Boot

**FIT**를 혼자 구현한다고 생각해보면 

- 에러가 생길 만한 부분을 찾기
- 원하는 시기에 원하는 빈도로 에러, 지연 를 발생
- 이걸 보려고 별도의 배포를 하지 않고, Runtime 시에 Flag, 설정들을 변경

이정도만 생각해도 온갖 `if`문으로 코드가 범벅이 될것 같습니다.

이런 부분을 쉽게 해결해 줄수 있는 공통 라이브러리 로 생각하면 될것 같습니다.

[Chaos Monkey Spring Boot]((https://codecentric.github.io/chaos-monkey-spring-boot/)) 를 적용한다면 실제로 Runtime 시에 여러 Error를 일으킴으로써 현재 시스템의 특징에 대해 더 잘 알수 있을겁니다.

- Fallback 이 원하는대로 작동하는지
- Network 지연에 대해서 어떤일이 발생하는지
- 실제 서비스, 또는 인스턴스 일부가 작동을 멈춘 다면
  - 그상황에서  Client Side Load Balance가 잘 작동하는지

그렇다면 해당 기능들이 우선 구현이 되어있어야 하고, 모니터링이나 평상시 상황(api 소요시간, 서버상태 등)을 잘 정의하여 Production에서 발생 시켰을때 어떤 일이 생기는지 (해도 되는지) 정의가 필요할겁니다.  

### 간단한 동작원리

![all](https://codecentric.github.io/chaos-monkey-spring-boot/images/sb-chaos-monkey-architecture.png)

이 라이브러리는 간단히 얘기하면 ***Annotaion으로 찾아가 문제를 일으키고, Acuator 로 실시간 조절 가능 하다*** 로 정리할수 있을거 같습니다.

처리 가능한 타입들은 위의 그림과 같이  `@RestController` 부터 `@Repository` 까지 친숙한 것들이 있는데 여기에 사전에 제공되는 공격(? assault) 방식에 따라 여러가지 장애가 주입됩니다.

- Latency Assault 
  - 정의된 최소값과 최대값의 사이의 시간만큼 지연을 일으키는 공격
  
- Exception Assault  
  - 해당 메소드에서 `RuntimeException`이 발생
  ```
    java.lang.RuntimeException: Chaos Monkey - RuntimeException
	    at de.codecentric.spring.boot.chaos.monkey.assaults.ExceptionAssault.attack(ExceptionAssault.java:52) ~[chaos-monkey-spring-boot-2.0.1.jar:na]
	    at de.codecentric.spring.boot.chaos.monkey.component.ChaosMonkey.chooseAndRunAttack(ChaosMonkey.java:70) ~[chaos-monkey-spring-boot-2.0.1.jar:na]
        ...
  ```
- AppKiller Assault 
  - 해당 Springboot App 을 종료 시켜 버림
  ```
  d.c.s.b.c.m.assaults.KillAppAssault : Chaos Monkey - I am killing your Application!
  ```

위의 세가지 Fault 를 원하는 빈도와 원하는 대상을 선택해 주입 가능합니다.

Spring Boot 에서 Actuator 로  손쉽게  설정을  확인하거나 변경 가능한데 Chaos monkey 가 제공하는 설정 값으 간단히 살펴 보겠습니다. 2.0.1 부터는 micro meter로 metric을 제공해 각종 모니터링 툴에서 실시간으로 확인이 가능해졌습니다. [[watcher_metrics](https://codecentric.github.io/chaos-monkey-spring-boot/2.0.1/#_watcher_metrics)] 

예제소스의 `application-chaos-monkey.yml` 파일입니다. 이 설정들은 모두 Runtime 에서 바꿀수 있습니다. ( Profile 을 `chaos-monkey`로 해야만 작동합니다.)

```yml
chaos:
  monkey:
    enabled: true
    assaults:
      level: 5
      latencyRangeStart: 1000
      latencyRangeEnd: 10000
      exceptionsActive: true
      killApplicationActive: true
      watchedCustomServices: ["sample.chaospilot.fake.FakeRemoteService.getNameAndLength"]
    watcher:
      controller: false
      restController: true
      repository: false
      service: false
      component: false
```

 각각의 속성에 대한 설명을 복사해왔습니다.

#### Properties

| Property                                    | Description                                                  | Values                               | Default |
| ------------------------------------------- | ------------------------------------------------------------ | ------------------------------------ | ------- |
| chaos.monkey.enabled                        | Determine whether should execute or not                      | TRUE or FALSE                        | FALSE   |
| chaos.monkey.assaults.level                 | How many requests are to be attacked. **1 each request**, 5 each 5th request is attacked | 1-10000                              | 5       |
| chaos.monkey.assaults.latencyRangeStart     | Minimum latency in ms added to the request                   | Integer.MIN_VALUE, Integer.MAX_VALUE | 3000    |
| chaos.monkey.assaults.latencyRangeEnd       | Maximum latency in ms added to the request                   | Integer.MIN_VALUE, Integer.MAX_VALUE | 15000   |
| chaos.monkey.assaults.latencyActive         | Latency assault active                                       | TRUE or FALSE                        | TRUE    |
| chaos.monkey.assaults.exceptionsActive      | Exception assault active                                     | TRUE or FALSE                        | FALSE   |
| chaos.monkey.assaults.killApplicationActive | AppKiller assault active                                     | TRUE or FALSE                        | FALSE   |
| chaos.monkey.watcher.controller             | Controller watcher active                                    | TRUE or FALSE                        | FALSE   |
| chaos.monkey.watcher.restController         | RestController watcher active                                | TRUE or FALSE                        | FALSE   |
| chaos.monkey.watcher.service                | Service watcher active                                       | TRUE or FALSE                        | TRUE    |
| chaos.monkey.watcher.repository             | Repository watcher active                                    | TRUE or FALSE                        | FALSE   |
| chaos.monkey.watcher.component              | Component watcher active                                     | TRUE or FALSE                        | FALSE   |



### 간단 구현

>  아래의 모든 예제는 [github](https://github.com/bk-ko/chaos-pilot)에서 확인 가능 합니다.

우선 필요한 의존성을 주입합니다.

```groovy
implementation('org.springframework.boot:spring-boot-starter-actuator')
implementation('de.codecentric:chaos-monkey-spring-boot:2.0.1')
```

(actuator 를 꽂지 않으면 jmx 기준으로 작동합니다.)

시작해보면 logo와 함께 시작이 됩니다. 

```
2018-10-26 16:46:50.095  INFO 29026 --- [ost-startStop-1] d.c.s.b.c.monkey.component.ChaosMonkey   : 
     _____ _                       __  __             _
    / ____| |                     |  \/  |           | |
   | |    | |__   __ _  ___  ___  | \  / | ___  _ __ | | _____ _   _
   | |    | '_ \ / _` |/ _ \/ __| | |\/| |/ _ \| '_ \| |/ / _ | | | |
   | |____| | | | (_| | (_) \__ \ | |  | | (_) | | | |   |  __| |_| |
    \_____|_| |_|\__,_|\___/|___/ |_|  |_|\___/|_| |_|_|\_\___|\__, |
                                                                __/ |
    _ready to do evil!                                         |___/

:: Chaos Monkey for Spring Boot                                    ::

```

Runtime 시 Chaos monkey 를 Control하기 위해서는 actuator가 필요합니다  우선 sample http 예제를 올려 놓기는 했지만 내부구현을 살펴보겠습니다. [Sample.http](https://github.com/bk-ko/chaos-pilot/blob/master/src/main/resources/sample-request.http)

가장 핵심 메소드를 같이 살펴 보면

```java
// de.codecentric.spring.boot.chaos.monkey.component.ChaosMonkey
public class ChaosMonkey {
    
    public void callChaosMonkey(String simpleName) { // simpleName : 현재 메소드 path
        if (isEnabled() && isTrouble()) { // 작동중인지, Assult를 일으킬 차례인지
    
            if (metricEventPublisher != null)
                metricEventPublisher.publishMetricEvent(MetricType.APPLICATION_REQ_COUNT, "type", "total");
    
            // Custom watched services can be defined at runtime, if there are any, only these will be attacked!
            // 설정을 통해 특정 클래스, 메소드만 문제를 일으키도록 제한 가능 - 'watchedCustomServices' property
            if (chaosMonkeySettings.getAssaultProperties().isWatchedCustomServicesActive()) {
                if (!chaosMonkeySettings.getAssaultProperties().getWatchedCustomServices().isEmpty() && chaosMonkeySettings.getAssaultProperties().getWatchedCustomServices().contains(simpleName)) {
                    // only all listed custom methods will be attacked
                    // 세가지 Assult중  선택, 실행!!
                    chooseAndRunAttack();
                }
            } else {
                // default attack if no custom watched service is defined
                chooseAndRunAttack();
            }
        }
    
    }
}
```



만약 Spring Cloud나 다른 Spring 기반의 MSA 구현 아래서, 아키텍쳐가 얼마나 Fault Tolerable 한지 확인하고 싶다면 상용에서 시도해볼만한 테스트라고 생각합니다.

Hystrix 의 Fallback, Ribbon Retry 등 과 함께 사용된다면, 대부분의 지연으로 인한 쓰레드 점유, 서버 다운등 상용에서도 별 문제 없이 작동되는 걸 확인할수 있을겁니다. 만약 여러대에서 전면적으로 수행한다면 Circuit이 원하는 대로 열릴지 이런것도 확인 가능 할거 같습니다. 

하지만 정말 어려운건, CUD 를 수행하는중에 실패에 어떻게 대처할지, Transaction 이 보장 안되는 네트워크 구간 사이에서는 멱등성에 대한 고민이 필요할거 같습니다. 

















