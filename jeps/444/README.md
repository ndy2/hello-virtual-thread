---
title: "JEP 444: Virtual Threads"
---

- [openjdk.org > jeps > 444](https://openjdk.org/jeps/444)

## Summaray

`Java Platform` 에 *virtual threads* 를 도입한다. Virtual threads 는 높은 처리량을 가진 동시성 애플리케이션의 작성, 유지보수, 운영 비용을 극적으로 낮춘다.

## History

- https://openjdk.org/jeps/444#History

## Goals

- 하드웨어 운용량을 near-optimal 수준까지 끌어올리는 thread-per-request 스타일의 server application 을 작성할 수 있다.
- 최소한의 변경으로 기존 `java.lang.Thread` 를 사용하던 코드가 virtual threads 를 이용할 수 있다.
- 기존 JDK tools 로 virtual threads 에 쉬운 트러블슈팅, 디버깅, 프로파일링을 제공할 수 있다.

## Non-Goals

- 전통적인 threads 를 제거하거나 silently migrate 하는 것은 목표가 아니다.
- Java 의 기본적인 동시성 모델을 변경하는 것은 목표가 아니다.
- Java 나 Java libraries 에 새로운 데이터 병렬성을 제공하는것은 목표가 아니다.

## Motivation

Java 개발자는 거의 30 여년을 concurrent server application 을 구성하기 위해 `thread` 에 의존해 왔다. 모든 메서드의 코드 한줄 한줄은 thread 내부에서 실행된다. Java 는 멀티스레드 이므로 여러 스레드의 실행은 동시에 이루어 진다. Thread 는 Java 의 `unit of concurrency` 이다. 즉 **서로 독립적이면서, 동시에 실행되는 일련의 코드 조각**이다. 각 Thread 는 지역변수를 저장하고, method 호출과 context (thread context) 를 조정하기위한 stack 을 제공한다. e.g.) 한 스레드 안에서 던져진 예외는 동일한 스레드의 다른 method 에 의해  caught 된다. 개발자는 thread 의 `stack trace` 를 통해 이를 추적할 수 있다. Thread 는 또한 Debuggers/ Profilers 에게도 중요한 개념이다.

### The thread-per-request style

Server 애플리케이션은 일반적으로 서로 독립적인 사용자의 요청을 처리합니다. 따라서, server 애플리케이션이 Thread 하나를 요청 하나를 처리하기 위해 전적으로 활용하는 것은 그럴싸 합니다. 이러한 `thread-per-request style` 은 이해하기 쉽고, 프로그램을 작성하고, 디버깅, 프로파일링 하기에도 쉽습니다. 그 이유는 `thread-per request style` 은 platform 의 `unit of concurrency - thread` 를 그대로 애플리케이션의 `unit of concurrency - thread` 로 사용하기 때문입니다.

Server 애플리케이션의 확장성 `Little's Law` 의 영향을 받습니다. e.g.) 
- 평균 지연율이 50ms 인 애플리케이션은 동시에 10 개의 요청을 동시에 처리하는 concurrency 를 통해 처리량 (TPS) 200 TPS 를 가집니다.
- 이 애플리케이션이 2000 TPS 를 가지기 위해서는 동시에 100개의 요청을 처리하는 concurrency 를 제공해야 합니다.

불행히도, JDK 는 thread 를 OS threads 의 wrapper 로 구현하기 때문에 (Native Thread) 전체 thread 의 개수는 제한되어 있습니다. OS 의 thread 는 비쌉니다. 따라서 많은 경우 thread-per-request style 을 부적합 합니다. 만약 하나의 요청이 하나의 Java thread -> 즉, OS thread 를 차지한다면, 많은 경우 Thread 의 개수는 CPU/ network connection 등의 다른 resource 에 비해 엄청한 limiting factor 가 될 것입니다. JDK 의 현재 구현은 server 애플리케이션의 처리량을 server 하드웨어 자원에 bound 되게 합니다. 이는 Thread Pool 과 같은 기법을 도입해도 마찬가지입니다. 왜냐하면 Thread Pool 은 Thread 의 (초기) 생성 비용을 해결하기 위한 방법이며 전체 사용가능한 Thread 개수는 늘려줄 수 없습니다.

### Improving scalability with the asynchronous style

일부 개발자들은 hardware 자원을 최대로 활용하기 위해 `thread-sharing style` 의 `thread-per-request style` 을 포기했습니다. 하나의 thread 를 요청 -> 응답 처음부터 끝까지 활용하는것이 아니라 thread 가 I/O 를 위해 waiting 하게 되면 즉시 thread 를 pool 로 반환하여 그 thread 는 다른 요청을 처리할 수 있도록 하였습니다. 이러한 방식은 fine-graned 한 thread 공유를 제공합니다. 즉, thread 는 calcultation (연산) 을 처리하는 코드만 처리 합니다. I/O 를 대기하는 구간을 처리하지 않습니다. 이를 통해 더 적은 thread 로 더 많은 conccurent 수준을 제공할 수 있습니다. 이러한 방식으로 OS thread 의 부족 (제약) 에 따른 처리량의 제한을 제결 할 수 있지만 다른 수준의 높은 비용이 필요합니다. 이러한 방식을 위해서는 `asyncrhonous programming style` 이라고 알려진 방식을 필요로 합니다. 즉, 모든 I/O 메서드를 구분하여 I/O 연산 호출이후 대기 (waiting) 하지 않도록 해야 합니다. 이를 위해 모든 코드 조각을 callback 으로 구성하여 sequential 한 pipeline 으로 제공해야 합니다. e.g. CompletableFuture/ Reactive Frameworks ... 이를 위해 loop/ try-catch block 같은 직관적인 순차 합성 연산을 활용할 수 없습니다.

Asyncrhonous sylte 에서, 요청의 각 단계는 다른 thread 에서 실행 될 수 있습니다. 모든 thread 는 서로 다른 요청을 interleaved 하게 실행합니다. 이는 프로그램을 매우 어렵게 만듭니다. 단적으로 Stack traces, debuggers, profilers 는 전통적인 thread-per-request style 에서 처럼 동작 할 수 없습니다. Labmda 표현식으로 callback 을 작성하는것은 java 의 stream API 를 이용하면 할 수 있지만 application 의 요청 처리 코드 전체를 이런 식으로 작성을 하는것은 쉽지 않습니다. Asyncrhonous style 은 application 의 `unit of concurrency - aynchronous pipeline` 가 platform 의 `unit of conccurency - thread` 와 다르다는 점에서 Java Platform 에는 매우 이질적입니다.

### Preserving the thread-per-request style with virtual threads

Java Platform 과 조화로움을 유지하며 애플리케이션이 확장가능하도록 하기 위해, 우리는 `thread-per-request style` 을 보존하기 위해 노력해야 합니다. Thread 를 더 효율적으로 만들면 할 수 있습니다. OS thread 는 더 효율적으로 만들 수 없지만 Java Runtime 에서 Thread 를 OS - Thread 와 일대일 대응 하는 방식을 채택하지 않으면 Java Runtime 의 thread 를 더 효율적으로 만들 수 있습니다. OS 가 Virtaul Memory 로 제한된 물리적인 RAM 의 한계를 극복한 것 처럼 Java Runtime 은 Virtaul Thread 을 통해 제한된 OS thread 개수라는 한계를 극복할 수 있습니다. 

- `virtual thread` 는 특정 OS thread 와 강하게 결합되지 않은 `java.lang.Thread` 인스턴스입니다.
- `platform thread` 는 반면, OS thread 의 단순한 wrapper 인 `java.lang.Thread` 인스턴스입니다.

virtual thread 를 사용하면 애플리케이션 코드는 thread-per-request style 을 유지할수 있습니다. 그와 동시에 virtual thread 는 cpu 자원을 실제 연산시에만 활용하기 때문에 asyncrhonouse style 의 장점인 높은 처리량을 가질 수 있습니다. virtual thread 가 실행 중 I/O blocking 연산을 `java.*` API 에서 만난다면 runtime 은 non-blocking OS 호출을 수행하고 자동으로 virtural thread 를 실행가능한 시점 까지 suspend 합니다. Java 개발자에게, virtual threads 는 단순히 거의 무한히 생성 할 수 있는 값싼 thread 입니다. 이를 통해 Hardware utilization 을 almost-optimal 하게 유지하면서도, 애플리케이션을 Java Platform 과 기존 방식에 조화롭게 유지할 수 있습니다.

### Implications of virtual threads

virtual thread 는 값싸고 아주 풍족합니다. 따라서 절때 이를 pool 하지 마세요. 새로운 virtual thread 는 모든 application task 에 생성 되어야 합니다. 대부분의 virtual thread 는 짧게 유지되어야 하며 앝은 call stack 을 가져야 합니다. platform thread 는 반면에 무겁고 비쌉니다. 따라서 항상 pooled 되곤 합니다. 이들은 보통 오래 유지되며 깊은 call stack 을 가지고 많은 task 간 공유되곤 합니다.

요약하자면, virtual thread 는 Java Platform 에 조화로운 안정적인 thread-per-request style 을 유지하면서, hardware 자원을 최대한 많이 활용할 수 있도록 합니다. virtual thread 를 활용하기 위해 새로운 개념을 학습해야하지 않습니다. 하지만 전통적인 thread 를 중요하고 무겁게 다루던 습관을 덜어내는 것을 필요로 할 수는 있습니다. 

## Description

오늘날, 모든 `java.lang.Thread` 의 구현체는 *`platform thread`* 입니다. platform thread 는 Java code 를 OS thread 의 아래에서 실행하며 코드 전체 생명주기 동안 OS thread 를 capture 합니다. 전체 platform thread 의 개수는 OS thread 의 개수에 제한되어 있습니다.

*`virtual thread`* 는 OS thread 의 아래에서 실행되지만 코드 전체 생명주기 동안 OS thread 를 capture 하지 않습니다. 이것은, 많은 virtual threads 가 자신의 Java code 를 동일한 OS thread 에서 효율적으로, 공유하며 실행할 수 있다는 것을 의미합니다. 반면 platform thread 는 귀중한 OS, thread 를 독점합니다. virtual thread 의 개수는 OS thread 의 개수보다 훨씬 많아질 수 있습니다.

`virtual thread` 는 OS 가아닌 JDK 가 제공하는 경량 스레드입니다. 이러한 `user-mode thread` 방식은 다른 멀티스레드 언어 (e.g., goroutines in Go, koroutines in Kotlin) 에서 이미 성공적으로 활용 중입니다. 초기 Java 에서 User-mode thread 는 *`green thread`* 라고도 불렸습니다. 반면, Java 의 green thread 에서는 여러 스레드가 하나의 OS thread 를 공유하는 방식을 활용했습니다. (M:1 scheduling) 이 방식은 Thread 를 OS-Thread 에 1:1 로 scheduling 하는 platform-thread 방식에 완전히 대체되었습니다. virtual thread 는 M:N scheduling 을 제공합니다.

### Using virtual threads vs. platform threads

개발자는 virtual thread/ platform thread 사용을 선택할 수 있습니다. 아래 코드는 많은 수의 virtual threads 를 생성하는 코드입니다. 이 프로그램은 주어진 task 에 대해 새로운 virtual thread 를 생성하는 ExecutorService 를 생성합니다. 그 다음 10,000 개의 작업을 수행하고 대기합니다.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
}  // executor.close() is called implicitly, and waits
```

virtual thread 를 활용하면 위 코드는 매우 빨리 실행됩니다. `Executors.newCachedThreadPool()` 혹은 `Executors.newFixedThreadPool(200)` 를 사용한다고 이런 어마어마한 동시성을 획득할 수 없습니다.

하지만 작업이 만약 단순히 1초 자는 것이 아니라 높은 cpu 사용량을 필요로 하는 경우 virtual thread/ platform thread 여부는 크게 영향을 주지 않습니다.

> [!NOTE] NOTE!
> They exist to provide scale (higher throughput), not speed (lower latency)

Virtual thread 는 platform thread 가 실행할 수 있는 모든 코드를 실행 할 수 있습니다. 특히, virtual thread 는 thread-local 변수를 지원합니다.

