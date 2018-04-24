---
layout: post
title:  "REST API로 MSA에서 CUD 처리하기"
date:   2018-04-24 20:31:01 +0900
categories: jekyll update
comments: true
---
# REST API로 MSA에서 CUD 처리하기


## 개요

MSA에서 멱등성 있게 API 호출하는 방법을 알아봤습니다.

500이나, timeout 으로 정상응답(200)으로 받지 않은경우, 진짜 Server의 요청 처리 실패인지, Network 문제인지 알 수 없습니다. 이때 재시도를 하는 방법들 입니다.

### 공감대 형성

- MSA에서는 Monolithic 수준의 강력한 일관성은 불가능합니다.
  - "Eventually Consistent" 하고, 이마저도 후처리(수작업 포함)가 없으면 데이터가 틀릴수도 있습니다.
  
    
- MSA에서의 Transaction 은 기존에 비해 고비용이므로 이걸 구현하는게 맞는지, 에러율과 서비스 중요도에 따라 판단하고, 모니터링과 후처리 로 빨리 Error를 드러내고 처리 하도록 해야합니다. 
  - 이미 결제도 REST 호출로 이루어 지고 있습니다.


- 기본적으로 CUD API 구현시에는 중복호출에 대한 방어로직이 추가되어 있어야 함을 인지 하셔야 합니다.

## PUT 으로 호출

### 쿼리 수행이 여러번 되어도 상관 없는경우
![Alt text](https://monosnap.com/image/OS57fwcwT9HVakxno21pPm8c1uZtFt.png)

PUT 방식으로 호출한다는 의미는 멱등성 있게 API 호출이 가능하다는 이야기입니다. 쿼리의 Update, Delete는 멱등하기 때문에 Caller는 200을 받을때까지 몇번이고 재시도가 가능합니다.

재시도 + timeout 설정으로 실패 판단을 할수 있습니다.

EDA의 장바구니 삭제 의 경우 이방식으로 개발 되어 있습니다.

> **Caller 가 할일**
> - retry, timeout에 대한 정의를 통해, 최종적으로 성공인지 실패인지 결정해야 합니다. retry 구현 필요 (Feign extenson) 최종 실패인경우 Throw
> 
> **Server가 할일**
> - 쿼리가 중복 수행되어도 문제 없는지 확인 (updated_time 필드)

---

### 쿼리 수행이 한번만 되어야 되는 경우
![Alt text](https://monosnap.com/image/3I7LXzZGfjljhT9Yj9KUfkqtnDphPz.png)
단순한 Update, Delete 가 아닌경우, Insert 와 같이 중복실행을 허용하지 않는 경우에는
Service Layer에서 기존 처리 여부를 확인후 저장하고 (메인 비지니스 로직과 같은 트랜잭션에서) 재요청시 이미 처리했으면 바로 정상응답하고 그렇지 않은경우에만 해당 요청을 처리 하는방식입니다.

이때도 마찬가지로 Caller 는 PUT 요청후 200 응답을 받을때까지 재시도 또는 timeout 관리를 해야할것입니다.

> **Caller 가 할일**
> - PK를 제공(주문번호, 상품번호, Transaction Id) 현재 요청에대한 고유값을 부여해 Server가 같은 요청인지 판단할수 있도록 제공해야함 
> - retry, timeout에 대한 정의로 성공실패 여부 결정 로직 retry 기능 구현 (feign-extension)
> 
> **Server가 할일**
> - 해당 Request에 대한 처리여부를 비지니스처리와 함께 저장해야함 (같은 Transaction) 
>   - 다음번 동일 요청이 왔을때 이미 처리된 요청 인지 체크하는 기능 구현 필요 중복 요청이라고 판단되면 재처리 하지 않고 바로 200 응답

---

## POST + Cache 조합으로 호출
![Alt text](https://monosnap.com/image/Ke5OfWHqEi0C0rkWSNtSecHY3sq833.png)
위의 구현과 비슷하지만 POST, Cache 를 사용하는 방법입니다.

POST는 Cache 할수 있는 Http Method입니다.

약간 다르지만 Write Through Cache 를 생각하시면 됩니다. 

Caller에서 요청을 처리후 결과가 Server의 Cache에 저장 되는 방식입니다 다만 Global Cache 가 구현되어 있어야만 가능하다는 전제가 있습니다. 보상처리 구현시 Cache Eviction 기능을 구현해야 합니다.


> **Caller 가 할일**
> - PK를 제공(주문번호, 상품번호, Transaction Id) 현재 요청에대한 고유값을 부여해 Server가 같은 요청인지 판단할수 있도록 제공해야함 
> - retry, timeout에 대한 정의로 성공실패 여부 결정 로직 retry 기능 구현 (feign-extension)
> 
> **Server가 할일**
> - 해당 Request에 대한 처리여부를 비지니스처리와 함께 Global Cache 에 저장해야함 (같은 Transaction) 다음번 동일 요청이 왔을때 Cache 된 값을 반환


---


## Long Transaction에서의 처리 (보상 API호출)
![Alt text](https://monosnap.com/image/zKfRQNvbaAx5t6Hdh1a2c8fmx55duw.png)
멱등성 있게 API를 호출하는 일과 더불어,동시에 여러가지 일들이 일어나는 긴 Caller 트랜잭션의 가운데에 REST 요청이 일어나는 경우 입니다.

예를 들면 상품등록 같은 경우입니다. (상품등록, 이미지등록, 쿠폰등록, 등 여러가지 일들이 동시에 일어나는경우)

위의 두경우와 달리, 중간의 REST 요청이 성공하였더라도, 그이후 Caller 의 로직에서 Exception이 발생하면 보상이 되도록 보상 API를 호출하여 원복해야 합니다.

이를 위해서는 두가지가 필요합니다.

- 요청에 대한 전체 트랜잭션을 관리할 Key (Transaction Id) 
- 요청 처리 원복을 위한 보상 API

보상 API 도 역시 실패 할수 있습니다.
Caller가 끝내 보상 API 정상 응답을 못받은경우 후처리를 통해 보상 API를 호출해야 합니다. (배치, 젠킨스 잡)

보상 API를 구현하기 위해서는 모든 CUD쿼리에 대한 원복용 데이터를 생성해야만 합니다. 비지니스의 CUD 쿼리 갯수가 많아질 수록 구현에 신경써야 합니다. 프로시저라면 보상을위한 프로시저를 만드는 별도의 구현을 해야할겁니다

이렇게 Remote Call 의 트랜잭션을 보장하는 것은 어렵기 때문에 정말 필요한지 생각해 봐야합니다.

> **Caller 가 할일**
> - API 호출시 고유의 Transaction Id가 있어야 합니다.
>   - 상품번호, 주문번호, UUID : 확인 또는 보상을 위한 Key Business API 호출이 200이 아닐경우 Exception을 Throw 해야 합니다.
> - 비지니스 API, Confirm API모두 최종실패인경우는 Exception Throw 하셔야 합니다.
> 
> **Server가 할일**
> - 보상 API, 서비스 구현
>   - 내부에서 수행하는 비지니스를 수행전 상태로 원복시킬수 있는 데이터를 조회, 저장하는 로직이 필요합니다.
>   - 다만 어느 수준까지 원복할지는 개발 시 선택, 결정하시면 됩니다. (변경한 값만 원래대로, 모든 필드를 원래대로?)
> - 후처리구현 (필요시)
>   - 배치로 돌면서 후처리 대상을 조회, 보상 API를 호출하도록 구현해야 합니다.
>   - 추가 작업이 필요한 건수, 에러 빈도를 고려해서 도입을 결정해야 합니다.


---


## FAQ

***Caller가 보상 API 호출도 못하고 Down되면 어떻게 해야 하나요?***

- Caller의 로직에서 중간에 보상 API호출 하지도 못하고 끝나버리는경우 이런 케이스 마저 방지하고 싶은경우, Caller 는 정상종료 되었음 을 알려주는 Confirm API 구현이 필요합니다. 이마저도 실패할경우는 후처리 배치가 처리하도록 합니다.

- 최종적인 API 호출이 실패(요청처리 실패, 네트워크 이슈) 하는 경우 역시 가능합니다. 꼭 필요한지..


***만약 REST Call이 Caller 로직의 맨마지막에 실행된다면 보상 API는 필요 한가요***

- 긴 트랜잭션에서 맨마지막에 REST Call이 위치한다면 별도의 보상 API 호출은 필요 없다고 봅니다. 다만 더 확실하게 하려면 후처리 배치는 필요합니다.
   - (Caller는 Throw 했지만, Server만 성공으로 알고 있는경우 처리를 위해...)
- 역시 각 API의 호출 빈도와 오류 상황으로 판단이 필요합니다.

***만약 응답시간이 10초 이상 걸리는 REST호출의 경우는 어떻게 해야하나요***

- Caller의 비지니스에 따라 구현이 달라질것 같습니다.
예를 들면, 비동기 처리후 3초마다 처리 완료 확인 API 반복호출 (4회) + timeout 으로 판단 내린다던지...


## 참고자료
- [Microservice Trade-Offs / martinfowler](https://martinfowler.com/articles/microservice-trade-offs.html#consistency)
- [caching-strategies-and-how-to-choose-the-right-one](https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/) 
- [restcookbook-idempotency](http://restcookbook.com/HTTP%20Methods/idempotency/)
- [Support-retry-with-idempotency](https://wiki.openstack.org/wiki/Support-retry-with-idempotency)  
- [idempotency-apis-and-retries](https://hackernoon.com/idempotency-apis-and-retries-34b161f64cb4) 
- [POST,Cache - RFC](https://tools.ietf.org/html/rfc7231#section-4.2.3) 