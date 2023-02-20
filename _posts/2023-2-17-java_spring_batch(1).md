---
layout: post
title: "java spring batch multi-thread (1)"
subtitle: "java spring batch를 multi-thread 방식으로 수행하는 방법에 대해 알아봅시다"
date: 2023-02-17 10:45:13 -0400
background: "/img/bg-index.jpg"
---

## 이 글에서 얻을 수 있는 정보

---

- 스프링 배치가 무엇이고, 배치를 더욱 빨리 처리하려면 어떻게 해야하는지
- 배치를 multi-thread 방식으로 처리하는 다양한 방법들과, 그것들의 비교분석
- taskExcuter를 이용해, chunk를 multi-thread 방식으로 구현하는 코드

<br>

## spring batch란 ?

---

- 어플리케이션을 운영하다보면 사용자와의 상호작용 없이, 대량의 데이터를 한번에 가공해야할 일이 생긴다. 예를 들어 월초에 사용자의 실적에 따라 등급을 업그레이드하거나, 연말 정산등을 할 때도 한번에 수많은 자료를 일괄적으로 처리해야 한다.

- 그리고 이런 작업은 대부분 반복적으로 수행되며, 레코드를 읽고 → 데이터를 처리하고 → 데이터를 다시 쓰는 비슷한 방식으로 수행된다. 따라서 spring에서는 이러한 기본 배치 반복을 자동화하고, 객체화 하여 사용자가 보다 쉽게 배치 작업을 수행할 수 있도록 했다.

## spring batch의 기본 구현 방식

---

![성격유형 검사하기](/assets/image/spring_batch.png){: width="100%" }

- Spring batch의 기본 구현 방식은 상단과 같다. Job launcher가 사용자가 명시한 여러가지 Job들을 수행한다. 그리고 JOB은 여러가지의 Step으로 이루어져 있다.

- 유저 등급을 갱신하는 상황을 예시로 들어보면, “매월 1일 유저 등급을 갱신하고, 등급에 따라 쿠폰을 배부한다”가 Job이고, Step1은 “유저 테이블에 등급을 매긴다”, Step2는 “등급에 따라 쿠폰을 배부한다” 가 될 것이다.

그리고 Step을 구현하는 두가지 방식에 대해 알아보자

- tasklet 방식
  - tep 내부에 tasklet을 가지고 있다.
  - 하나의 tasklet 내부에서 자유롭게 데이터를 처리하는 것이다.
    - read, process, write를 하나의 tasklet에 넣을 수도 있고, read와 process를 하나의 tasklet에서, write를 하나의 tasklet에서 하는 등 자유롭게 찢을 수도 있다.
  - 배치 과정이 비교적 쉬울 경우 사용
  - 대량 처리를 하는 경우에는 코드가 오히려 더 복잡해질 수 있음.
- Chunk 방식
  - Chunk : Read, Process, Write로 이루어진 하나의 데이터 처리 단위
  - itemReader, itemProcessor, itemWriter에 대해 이해해야 구현할 수 있다.
  - 하나의 큰 덩어리를 작은 덩어리로 나누어 처리할 수 있다.

<br>

## Multi-thread 기반 Spring batch 처리

---

##### 💡 문제는, 너무 느려 !

- spring batch의 경우 애초에 대용량의 데이터를 한꺼번에 처리하기 위해 만들어졌으므로, 수많은 레코드들을 한번에 갱신해야 한다. 그런데 문제는 ,, 스프링 배치가 기본적으로 single thread 방식으로 진행된다는 것이다 !
- 하단은 1011개의 데이터를 single-thread 기반으로 처리할 때 걸린 시간을 측정한 것이다. 총 49초가 소요되었음을 알 수 있다..

![성격유형 검사하기](/assets/image/batch_result.png){: width="100%" }

- 물론 배치 작업이 자주 이루어지지 않거나, 서비스 장애가 크리티컬한 이슈를 만들어내지 않는 환경이라면 굳이 multi-thread 방식을 도입하지 않아도 될 것이다.
- 그러나 자주 배치 작업이 일어나고, 서비스 중단이 유저의 사용성에 큰 영향을 줄 경우 최대한 spring batch에 드는 시간을 줄여야하며, 이 경우 multi-thread 기반 spring batch 처리를 반드시 해야할 것이다 !

  <br>

## Multi-thread Spring batch 처리 종류

---

- Multi-therad 기반으로 Spring batch를 처리하는 데에는 4가지 방식이 있다. 네가지 방식에 대해 알아보고, 각각의 장단점을 분석해보자.

##### 1. AsyncItemProcessor / AsyncItemWrite를 이용하는 방식

- 한 Step 내에서 Processor만 병렬 수행해야할 때 사용한다.
- itemProcessor에게 별도의 스레드가 할당되어 작업을 처리한다.

![성격유형 검사하기](/assets/image/Async.png){: width="100%" }

- 전반적인 작동 방식은 상단과 같은데, 해당 방식의 포인트는 ‘위임’이라고 할 수 있다.
- 만약 프로세스가 동기적으로 수행된다면 read를 한 뒤, 기다렸다 processor로 처리하고, 기다렸다 writer로 처리하고,, 이렇게 순차적으로 진행이 될 것이다.
- 그런데 이 방식은 read까지는 기다리지만, process와 write는 기다리지 않고 바로바로 다음 작업을 처리 하는 것이다.
- 예를 들어 A유저, B유저, C유저의 등급을 업그레이드 한다고 하자. 이때 A유저의 등급을 읽어온 다음, processor에서는 A유저의 실적으로 등급을 매기고, 이 값을 writer로 써야 한다. mainthread, ItemProcessor, ItemWriter입장에 따라 단계를 보면 하단과 같다.

###### 💁🏻 main processor 입장

1. Reade가 A유저의 정보를 읽어온다.
2. 그런데 이때 AsyncItemProcessor는 A유저의 등급을 다 매길때까지 기다리지 않고 thread를 파 ItemProcessor에게 이 작업을 위임하고, AsyncItemWriter로 넘어간다.
3. AsyncItemWriter도 다 쓸때까지 기다리지 않고, ItemWiter에게 작업을 위임하고, 다시 read 작업을 넘어간다.

###### 🙆‍♂️ ItemProcessor(thread)입장

1. AsyncItemProcessor가 일을 시키면 열심히 처리하고, 이걸 Future라는 객체에 넣는다.
2. 일 다했으면 소멸 한다 ,,

###### 🙋‍♀️ ItemWriter(thread)입장

1. Future내에는 비동기 처리중인 작업들이 있을 것이다. 이때 비동기 처리된 실행의 결과값을 모두 받아올 때까지 기다린다.
2. 다 받아왔으면 이 값을 DB에 저장한다.

## 마무리

---

- 이번 포스팅에서는 java spring batch가 무엇인지 알아보고, spring batch를 multi-thread로 처리할 때의 이점에 대해 알아보았다.
- 다음 포스팅에서는 spring batch를 multi-thread로 처리하는 다른 방법들에 대해 알아보고, 실제로 코드에 적용해보자 !

<br>
