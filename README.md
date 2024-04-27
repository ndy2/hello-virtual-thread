# hello-virtual-thread

## Vritual Thread

- 자바의 경량 쓰레드
- 경량 쓰레드란 - 기존 언어의 스레드 모델보다 더 작은 단위로 실행 단위를 나누어 컨텍스트 스위칭 비용과 Blocking 타임을 낮추는 개념
- Thread Per Request 구조의 서버 애플리케이션 HW 최적 적용
- 최소 변경으로 가상 쓰레드를 제공하자
- Platform-Thread 에 비해 가벼우며 수만 ~ 수백만개의 Thread 를 활용할 수 있다.

![[virtual-thread.excalidraw.png]]

## Platform Thread

- 기존 Java 의 활용방식
- OS Thread 와 1:1 매핑된다.
	- Platform Thread 는 OS Thread 의 thin wrapper
- Blocking 이 발생하는 것을 줄이기 위해 Reactive/Non-Blocking/비동기 방식을 활용해야한다.
- Platform Thread 의 개수는 OS Thread 의 개수에 제한되며 비교적 작다.
- 보통 Thread Pool 기법을 함께 활용한다.

![[platform-thread.excalidraw.png]]

## Virtual Thread vs Platform Thread

|           | Platform Thread | Virtual Thread |
| --------- | --------------- | -------------- |
| Stack 사이즈 | ~2MB            | ~10KB          |
| 생성시간      | ~1ms            | ~1µs           |
| 컨텍스트 스위칭  | ~100µs          | ~10µs          |
## 주요 특징

- CPU 바운드 작업이 아닌 IO 바운드 작업에 강하다.
- 처리 시간을 줄여주는 것이 아닌 처리량을 늘리는 방식이다.
- Pool 을 생성하지 말라

## 관련 자료

- [openjdk.org > jeps > 444](https://openjdk.org/jeps/444)
- Jakob Jenkov > Java Virtual Threads
- Jakob Jenkob > Java ExecutorService Using Virtual Threads
- 최범균 > 자바21 주요 특징3 - 가상 쓰레드
- 최범균 > 자바21 가상 쓰레드 쓰면 정말 성능 좋아?
