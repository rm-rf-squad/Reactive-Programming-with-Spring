# Chapter 09 Sinks

**Sinks**는 Project Reactor에서 신호(데이터, 오류, 완료 신호)를 리액티브 스트림에 프로그래밍 방식으로 밀어 넣을 수 있게 해주는 컴포넌트이다.  핫 퍼블리셔를 만들고 백프레셔를 수동으로 관리할 수 있다.

* 과거 Processor의 기능을 개선한 기능이다.

Sinks를 사용하면 명시적으로 Signal을 전송할 수 있다.

* Signal이란 리액티브 스트림에서 데이터 흐름을 제어하는 신호. `onNext`(데이터), `onError`(에러), `onComplete`(완료) 같은 이벤트를 의미

**create() Operator 사용 예제:**

```java
@Slf4j
public class Example9_1 {
    public static void main(String[] args) throws InterruptedException {
        int tasks = 6;
        Flux.create((FluxSink<String> sink) -> {
                IntStream
                        .range(1, tasks)
                        .forEach(n -> sink.next(doTask(n)));
            })
            .subscribeOn(Schedulers.boundedElastic())
            .doOnNext(n -> log.info("# create(): {}", n))
            .publishOn(Schedulers.parallel())
            .map(result -> result + " success!")
            .doOnNext(n -> log.info("# map(): {}", n))
            .publishOn(Schedulers.parallel())
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }

    private static String doTask(int taskNumber) {
        return "task " + taskNumber + " result";
    }
}
```

#### create() Operator

- **단일 쓰레드 기반**: 기본적으로 단일 스레드에서 데이터를 생성
- **데이터 생성**: `FluxSink`를 통해 데이터(`doTask` 결과)를 생성하고 `next`를 통해 신호를 전송
- **스케줄링**: `subscribeOn`과 `publishOn`을 사용하여 다른 스레드에서 데이터 처리를 수행

결과적으로 3개의 스레드가 동시에 실행되어 데이터를 처리한다.

* .subscribeOn(Schedulers.boundedElastic()), .publishOn(Schedulers.parallel()), .publishOn(Schedulers.parallel())

왜 문제가 될 수 있을까?

여러 스레드에서 데이터를 생성하고 signal을 전송하는 경우, create() Operator는 스레드 세이프하지 않다.

여러 스레드에서  여러 스레드가 `sink.next()`를 동시에 호출하면 데이터의 순서가 꼬이거나, 충돌이 발생할 수 있다. 



반면에 Sinks는 어떨까

```java
/**
 * Sinks를 사용하는 예제
 *  - Publisher의 데이터 생성을 멀티 쓰레드에서 진행해도 Thread safe 하다.
 */
@Slf4j
public class Example9_2 {
    public static void main(String[] args) throws InterruptedException {
        int tasks = 6;

        Sinks.Many<String> unicastSink = Sinks.many().unicast().onBackpressureBuffer();
        Flux<String> fluxView = unicastSink.asFlux();
        IntStream
                .range(1, tasks)
                .forEach(n -> {
                    try {
                        new Thread(() -> {
                            unicastSink.emitNext(doTask(n), Sinks.EmitFailureHandler.FAIL_FAST);
                            log.info("# emitted: {}", n);
                        }).start();
                        Thread.sleep(100L);
                    } catch (InterruptedException e) {
                        log.error(e.getMessage());
                    }
                });

        fluxView
                .publishOn(Schedulers.parallel())
                .map(result -> result + " success!")
                .doOnNext(n -> log.info("# map(): {}", n))
                .publishOn(Schedulers.parallel())
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(200L);
    }

    private static String doTask(int taskNumber) {
        // now tasking.
        // complete to task.
        return "task " + taskNumber + " result";
    }
}
```

**멀티스레드 지원**: 여러 스레드에서 안전하게 신호를 전송할 수 있다.

**스레드 안전성**: 내부적으로 동기화를 관리하여 예기치 않은 동작을 방지한다.





즉 우리가 개발하는 멀티스레드 웹 어플리케이션 환경에서는 create()같은 thread-safe 하지 않은 오퍼레이터보다 Sinks같은 오퍼레이터를 사용하는것이 적절하다.  

## Sinks 종류 및 특징

**Sinks.One**: 단일 값 또는 완료 신호를 관리.

**Sinks.Many**: 여러 값을 관리하며, 다양한 백프레셔 전략을 지원.

**Sinks.Empty**: 즉시 완료되는 sink.



### Sinks.ONE

한건의 데이터를 emit하여 Mono 방식으로 사용한다.

```java
@Slf4j
public class Example9_4 {
    public static void main(String[] args) throws InterruptedException {
        Sinks.One<String> sinkOne = Sinks.one();
        Mono<String> mono = sinkOne.asMono();

        sinkOne.emitValue("Hello Reactor", FAIL_FAST);
        sinkOne.emitValue("Hi Reactor", FAIL_FAST);
        sinkOne.emitValue(null, FAIL_FAST);

        mono.subscribe(data -> log.info("# Subscriber1 {}", data));
        mono.subscribe(data -> log.info("# Subscriber2 {}", data));
    }
}
```

FAIL_FAST 파라미터는 emit도중 발생할 경우 어떻게 처리할지에 대한 핸들러이다.(상수같지만 함수형 인터페이스 상수임)

Sinks.One으로 아무리 많은 수의 데이터를 emit한다 하 더라도 처음 emit한 데이터는 정상적으로 emit되지만 나머지 데이터들은 Drop된다. (1개만 emit하니까)

###   SinksMany

Sinks.many() 메서드를 사용해서 여러 건의 데이터를 여러 가지 방식으로 전송하는 기능을 정의해 둔 기능 명세라고 볼 수 있다.

One의경우, 한건의 데이터만 emit하므로 별도 스펙이 필요없지만, Many의 경우 여러 스펙이 존재한다

```java
	public interface ManySpec {
	
		UnicastSpec unicast();

		MulticastSpec multicast();

		MulticastReplaySpec replay();
	}
```

#### UnicastSpec

- **의미**: 하나의 Subscriber에게만 데이터를 전송.
- **사용 예**: 단일 소비자가 있는 경우.
  - UnicastSpec에 여러 컨슈머를 연결하면 IllegalStateException이 발생함. 

#### MulticastSpec

- **의미**: 여러 Subscriber에게 동일한 데이터를 전송.
- **사용 예**: 여러 소비자가 동일한 데이터를 동시에 수신해야 하는 경우.
  - Hot Sequence로 동작한다. 

#### MulticastReplaySpec

- **의미**: 여러 Subscriber에게 동일한 데이터를 전송하며, 새로운 Subscriber가 구독 시 이전 데이터를 재생.
- **사용 예**: 새로운 구독자가 이전 데이터를 포함한 전체 스트림을 받아야 하는 경우.



Sinks가 Publisher의 역할을 할 경우 기본적으로 Hot Publisher로 동작하며, onBackpressureBuffer() 메서드는 Hot sequence로 동작한다. 

```java
@Slf4j
public class Example9_9 {
	public static void main(String[] args) {
		Sinks.Many<Integer> multicastSink =
			Sinks.many()
				.multicast()
				.onBackpressureBuffer();
		Flux<Integer> fluxView = multicastSink.asFlux();

		multicastSink.emitNext(1, FAIL_FAST);
		multicastSink.emitNext(2, FAIL_FAST);

		fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));
		fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));  // 1, 2는 출력하지 못함.

		multicastSink.emitNext(3, FAIL_FAST); 
	}
}

```

Cold Sequence로 동작하게 하려면 MulticastReplaySpec를 사용해야 한다. 

* Sinks.Many의 MulticastReplaySpec은 emit된 데이터 중에서 특정 시점으로 되돌린 (replay) 데이터부터 emit한다.