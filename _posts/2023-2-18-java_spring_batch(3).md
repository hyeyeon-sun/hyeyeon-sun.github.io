---
layout: post
title: "spring batch를 multi-thread로 구현하는 여러가지 방법(2)"
subtitle: "parallel steps, remote chunking, remote partitioning으로 Spring batch를 구현하는 방법에 대해 알아봅시다"
date: 2023-02-18 10:45:13 -0400
background: "/img/bg-index.jpg"
---

## 이 글에서 얻을 수 있는 정보

---

- 배치를 multi-thread 방식으로 처리하는 다양한 방법들과, 그것들의 비교분석
- multi-thread 방식으로 배치 코드를 구현할 때 주의할 점
  <br><br>

## 지난 글을 복습해보자 😌

---

- 지금까지의 과정을 요약해보자.

- 먼저 AsyncItemProcessor / AsyncItemWrite를 이용하는 방식은 기본적으로 read는 동기적으로 처리하면서, processor와 write만 비동기적으로 처리하는 것이었다.

- 다음으로 multi-thread는 read, process, write를 모두 비동기적으로 처리하는 것으로, 한 step 내에서 reader, processor, writer를 chunk단위로 병렬수행해야 할 때 사용한다.

- 그렇다면 위 두가지 방식의 공통점은 무엇일까 ? 바로 병렬처리가 **step내부 단위**에서 일어난다는 것이다 !

- 그런데 Job에는 여러가지의 Step이 있을 수 있으므로, 이 Step들 또한 병렬적으로 처리해야 하는 경우가 있다. 지금부터 이런 경우에 대해 알아보자.

  ## <br>

##### 3. Parallel Steps

---

- Paralle Steps가 바로 Step을 병렬로 처리할 때 사용하는 방법이다.

- Job은 기본적으로 step을 한번에 하나씩 선형적으로 실행하지만, 이 방법을 사용해서 병렬로 step을 처리할 수 있다.

![parallelSteps](/assets/image/parallelSteps.png){: width="90%" }

- 전반적인 구성은 상단과 같다. SplitState를 사용해 여러개의 Flow를 병렬적으로 실행하는 구조인데, 실행이 다 완료되면 FlowExecutionStatus의 결과들을 취합해 다음 단계를 결정한다.

![parallelSteps2](/assets/image/parallelSteps2.png){: width="60%" }

- parallel steps 에서는, flow라는 개념을 적극적으로 사용한다. step을 flow로 한번 감싸주고, 이 flow들이 병렬로 수행된다고 이해하면 좋을 것이다.

- 그런데 여기서 특이한 건 step4다. 기본적으로 step4는 병렬처리가 되지 않으며, step1과 step2, step3가 동시에 병렬처리가 되며, 이 모든게 끝나나 뒤에 step1의 후속작업인 step4가 수행되는 것이다.

<br>
#### ❗️ parallel steps를 수행할 때 주의해야 할 점

- 각 스레드가 동시에 같은 데이터에 접근하게 되면 동시성 문제가 발생할 수 있다. 따라서 이런 문제가 발생하지 않도록 lock을 사용하여 동시성으로 인한 문제가 발생하지 않도록 해야한다 !
  <br>

```java
private Object lock = new Object();

synchronized (lock) {
    for (int i = 0; i < 1000000000; i++) {
        sum++;
    }
    // thread 이름 출력
    System.out.println(String.format("%s has been executed on thread %s",
            chunkContext.getStepContext().getStepName(),
            Thread.currentThread().getName()));
    System.out.println(String.format("sum : %d", sum));
}
```

- 상단과 같이 lock 객체를 생성한 뒤, synchorinized 블럭 내부에 동시에 접근하는 데이터를 위치시킴으로써 여러 thread간에 동시에 같은 데이터에 접근하여 발생하는 문제를 예방할 수 있다.

<br>

##### 4. Externalizing Batch Process Execution

---

- 외부 remote 서버에서 병렬 수행해야 할 때 사용한다.

- 지금까지는 모두 스프링 배치를 Spring Integration이 감싸서 프로세스가 수행되는 식이었다.

- 그러나 이런 Spring Integration도 스프링 배치 내부에서 사용될 수 있다 !

- 이를 통해 아이템, 청크 처리를 외부 프로세스로 위임할 수 있다.

- 이와 같은 방식으로 remote Chunking과 remote Partitioning이 있는데, 둘을 각각 알아보자.

  <br>

###### remote Chunking

![remoteChunking](/assets/image/remoteChunking.png){: width="60%" }

- remote chunking은 상단과 같이 구성된다고 할 수 있다.

- remote chunking을 사용하기 위해서는 먼저 step을 여러 프로세스로 나눈다. 그리고 나눈 프로세스를 다른 미들웨어에 할당하여 사용한다.

- 여기서 step1, step2, step3를 수행하는 것을 매니저 컴포넌트, 하단의 step2a, step2b, step2c를 수행하는 것을 워커라고 해보자.

- 이때 매니저 컴포넌트는 싱글 프로세스이고 워커는 멀티 리모트 프로세스다. 이 패턴은 매니저에 병목이 없어야 가장 잘 동작하므로, 반드시 process가 read보다 훨씬 무거운 작업일 때 사용해야 한다. (프로세스가 가볍다면 굳이 나누어서 remote에 위임할 필요가 없겠지 ?)

```java
@EnableBatchIntegration
@EnableBatchProcessing
public class RemoteChunkingJobConfiguration {
    @Configuration
    public static class ManagerConfiguration {

        @Autowired
        private RemoteChunkingManagerStepBuilderFactory managerStepBuilderFactory;

        @Bean
        public TaskletStep managerStep() {
            return this.managerStepBuilderFactory.get("managerStep")
                       .chunk(100)
                       .reader(itemReader()) //**1**
                       .outputChannel(requests()) // **2** requests sent to workers
                       .inputChannel(replies())   // **7**replies received from workers
                       .build();
        }
    }

    @Configuration
    public static class WorkerConfiguration {

        @Autowired
        private RemoteChunkingWorkerBuilder workerBuilder;

        @Bean
        public IntegrationFlow workerFlow() {
            return this.workerBuilder
                       .itemProcessor(itemProcessor()) // **3**
                       .itemWriter(itemWriter()) // **4**
                       .inputChannel(requests()) // **5** requests received from the manager
                       .outputChannel(replies()) // **6** replies sent to the manager
                       .build();
        }
    }
}
```

- 코드를 살펴보자

&nbsp;&nbsp;&nbsp; 1. Reader가 A유저의 정보를 읽어온다.<br>
&nbsp;&nbsp;&nbsp; 2. 그런데 이때 AsyncItemProcessor는 A유저의 등급을 다 매길때까지 기다리지 않고 thread를 파 ItemProcessor에게 이 작업을 위임하고, AsyncItemWriter로 넘어간다.<br>
&nbsp;&nbsp;&nbsp;3~4. AsyncItemWriter도 다 쓸때까지 기다리지 않고, ItemWiter에게 작업을 위임하고, 다시 read 작업을 넘어간다.
&nbsp;&nbsp;&nbsp; 5~6. manager로부터 request를 받고, 이에 대한 reply를 보낸다.<br>
&nbsp;&nbsp;&nbsp; 7. manager는 worker로 부터 request를 받는다.

- 상단과 같이 진행된다. 이처럼 remote chunking을 이용하면 복잡한 처리가 필요한 process를 외부 프로세스에 위임하여 빠르게 처리할 수 있다.
  <br>
  <br>

###### remote Partitioning

![remotePartitioning](/assets/image/remotePartitioning.png){: width="60%" }

- remote partitioning은 remote chunking과는 달리, I/O가 병목일 때 (process보다도 읽고 쓰는 작업이 무거울 때) 유용하다.
- remote partitioning을 사용하면 스텝 자체를 spring batch에 완전히 맡길 수 있다.
- 각 워커마다 ItemReader, ItemProcessor, ItemWriter를 따로 가지고 있으며, 이를 통해서 manager가 부여한 일을 처리한다.
  <br><br>

## 마무리

---

- 이번 포스팅에서는 spring batch를 multi-thread 방식으로 구현하는 방식에 대해 알아보았다.
- 다음 포스팅에서는 이 중 multi-threaded step을 구현해보고, 서비스에 적용하는 과정에 대해 알아보자 !

<br>
