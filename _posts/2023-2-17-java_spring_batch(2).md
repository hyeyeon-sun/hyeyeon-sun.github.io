---
layout: post
title: "spring batch를 multi-thread로 구현하는 여러가지 방법(1)"
subtitle: "Async, Multi-threaded Step 으로 Spring batch를 구현하는 방법에 대해 알아봅시다"
date: 2023-02-17 10:45:13 -0400
background: "/img/bg-index.jpg"
---

## 이 글에서 얻을 수 있는 정보

---

- 배치를 multi-thread 방식으로 처리하는 다양한 방법들과, 그것들의 비교분석
- multi-thread 방식으로 배치 코드를 구현할 때 주의할 점
  <br>

## Multi-thread Spring batch 처리 종류

---

- Multi-therad 기반으로 Spring batch를 처리하는 데에는 4가지 방식이 있다.
- AsyncItemProcessor / AsyncItemWrite 이용, multi-Threaded Step, Paralle Steps, Externalizing Batch Process Execution 이다.
- 네가지 방식에 대해 알아보고, 각각의 장단점을 분석해보자.
  <br><br>

##### 1. AsyncItemProcessor / AsyncItemWrite를 이용하는 방식

- 한 Step 내에서 Processor만 병렬 수행해야할 때 사용한다.
- itemProcessor에게 별도의 스레드가 할당되어 작업을 처리한다.

![성격유형 검사하기](/assets/image/Async.png){: width="100%" }

<div style = "line-height: 40px; margin-bottom: 30px;">
&nbsp;&nbsp;&nbsp;&nbsp;•&nbsp;&nbsp;전반적인 작동 방식은 상단과 같은데, 해당 방식의 포인트는 ‘위임’이라고 할 수 있다.<br>
&nbsp;&nbsp;&nbsp;&nbsp;•&nbsp;&nbsp;만약 프로세스가 동기적으로 수행된다면 read를 한 뒤, 기다렸다 processor로 처리하고, 기다렸다 writer로 처리하고,, 이렇게 순차적으로 진행이 될 것이다.<br>
&nbsp;&nbsp;&nbsp;&nbsp;•&nbsp;&nbsp;그런데 이 방식은 read까지는 기다리지만, process와 write는 기다리지 않고 바로바로 다음 작업을 처리 하는 것이다.<br>
&nbsp;&nbsp;&nbsp;&nbsp;•&nbsp;&nbsp;예를 들어 A유저, B유저, C유저의 등급을 업그레이드 한다고 하자. 이때 A유저의 등급을 읽어온 다음, processor에서는 A유저의 실적으로 등급을 매기고, 이 값을 writer로 써야 한다. mainthread, ItemProcessor, ItemWriter입장에 따라 단계를 보면 하단과 같다.
</div>

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

<br>

##### 2. Multi-threaded Step

![multi-threaded Step](/assets/image/multi-threaded_Step.png){: width="100%" }

- Multi-threaded Step은 한 step 내에서 reader, processor, writer를 chunk단위로 병렬수행해야 할 때 사용한다.

- 구체적인 순서를 살펴보면 다음과 같다.

<div style = "line-height : 40px">
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;1. 먼저 Step은 TaskExecutorRepeatTemplate을 이용해 일을 한다. 이때에 Thread 만큼 Runnable을 만들고, 각 쓰레드는 일을 수행한다.<br>
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2. Runnable은 RepeatCallBack 함수를 수행한다.<br>
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;3. RepeatCallBack 함수는 ChunkOrientedTasklet을 수행한다. 이 Tasklet은 read, process, write로 이루어져 있다 !<br>
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;4. 각 Thread는 독립적으로 ItemReader / ItemProcessor / ItemWrite를 이용한다.<br>
</div>
<br>

#### ❗️ Multi-threaded Step 을 사용할 때 주의할 점

<br>

##### 1. thread-free한 (동기화 처리가 되는) reader를 사용하자 !

- TaskExecutoerRepeatTemplate이 여러개의 일꾼(Thread)를 만들고, 일을 시키는 상황을 상상해보자. 일단 일꾼 A가 일을 잡으면, process하고 write하는 것도 오로지 그 일꾼 A가 하기 때문에 동기화 문제가 발생할 염려가 없다.

- 그러나 맨처음 일을 잡으러 일꾼 A와 일꾼 B가 달려가는 상황을 상상해보자 ,, 그리고 만약 둘이 달려가서 같은 일을 잡아버린다면 ? 🫢🫢🫢

- 두명의 일꾼이 동시에 process와 write를 처리해야 하다보니 데이터의 일관성에 문제가 생길 수도 있고, 불필요한 노동력이 낭비 된다.

- 따라서 이런 문제를 피하기 위해 Reader는 반드시 동기화 처리가 되는 reader를 써야한다 !

- 동기화 처리가 되는 reader로는 **JdbcPagingItemReader / JpaPagingItemReader / SynchronizedItemStreamReader**등이 있다.

<br>

##### 2. 대량의 데이터를 동시에 처리할 때 connection pool 고갈 문제가 발생할 수 있다 🥹

![db connection pool](/assets/image/dbConnectionPool.png){: width="100%" }

- java에는 커넥션 풀이라는 좋은 방식이 있다. 상단의 그림과 같은데, 기본적으로 web application과 db를 연결하는 작업은 많은 비용이 드는 작업이므로, 미리 application과 db와의 연결을 만들어두고 이것을 pool로 관리하는 것이다 !

예를 들어보자

🙆‍♂️ Thread1이라는 일꾼 : “db 연결하고 싶어요 !!” 라고 하면<br>
🙋‍♀️ 풀 프레임워크(풀 관리해주는 친구) : “그래, 여기 있어 !”

라고 하면서 pool에 있는 connection을 하나 꺼내 주는 것이다.

- 그런데 ,, 만약 connection pool보다 많은 일꾼이 connection을 요청한다면 ? 일꾼은 일이 끝날 때까지 계속 기다려야 하고, 대기 시간이 매우 길어질 수 있다 😱

- 이뿐 아니라 만약 하나의 thread가 두개의 connection pool을 필요로 한다면 ? 최악의 경우 deadlock 문제가 발생할 수 있다 !

- 따라서 Multi-threaded Step 방식을 사용할때는, 사용가능한 thread의 수와 connection pool의 최대 개수를 반드시 잘 맞추어줘야한다.

<br>

##### 그래서, connection pool을 몇개로 설정해야 하는데 ??

_pool size = Tn x (Cm - 1) + 1_

- *Tn* : 전체 Thread 갯수<br>
- *Cm* : 하나의 Task에서 동시에 필요한 Connection 수

<div style = "line-height : 40px; ">
- hikariWiki의 글에 따르면, 통상적으로 상단의 개수를 설정하면 된다고 한다.<br>

- 상단의 공식을 따르면 deadlock 문제를 피하면서, pool size를 최적화할 수 있다.
</div>

<br>

## 마무리

---

- 이번 포스팅에서는 spring batch를 multi-thread 방식으로 구현하는 방식에 대해 알아보았다.
- 다음 포스팅에서는 이번 글에 이어서 multi-thread 기반 spring batch 구현 방식에 대해 더 알아보자 !

<br>
