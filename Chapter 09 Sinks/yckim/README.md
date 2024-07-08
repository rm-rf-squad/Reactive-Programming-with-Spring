# Sinks

Sinks는 리액티브 스트림즈의 Signal을 프로그래밍 방식으로 푸시할 수 있는 구조이며, Flux 또는 Mono의 의미 체계를 가집니다.

쉽게 말해 Flux 또는 Mono가 onNext 같은 Signal을 내부적으로 전송해주는 방식이 아닌 Sinks를 통해 프로그래밍 코드를 통해 명시적으로 Signal을 전송할 수 있습니다.

### 기존 방식과 Sinks를 사용한 방식의 차이

Reactor에서는 generate() Operator나 create() Operator를 사용하여 프로그래밍 방식으로 Signal을 전송할 수 있습니다.

이들과 Sinks의 차이는 다음과 같습니다.

- generate(), create()
    - 싱글 스레드 기반에서 사용됨

    ```java
    public static void main(String[] args) throws Exception {
            int tasks = 6;
    
            Flux.create((FluxSink<String> sink) -> {
                        IntStream.range(1, tasks)
                                .forEach(n -> sink.next(doTasks(n)));
                    })
                    .subscribeOn(Schedulers.boundedElastic())
                    .doOnNext(n -> log.info("# create() : {}", n))
                    .publishOn(Schedulers.parallel())
                    .map(result -> result + " success!")
                    .doOnNext(n -> log.info("# map() : {}", n))
                    .publishOn(Schedulers.parallel())
                    .subscribe(data -> log.info("# onNext : {}", data));
    
            Thread.sleep(500);
        }
    
        private static String doTasks(int taskNumber) {
            return "task " + taskNumber + " result";
        }
    ```

  위 코드는 결과 값을 반환하는 doTask() 메서드가 싱글 스레드가 아닌 여러 개의 스레드에서 각각의 전혀 다른 작업들을 처리한 후 처리 결과를 반환하는 상황이 발생할 수 있습니다.

- Sinks
    - 멀티 스레드 방식으로  Signal을 전송함

    ```java
        public static void main(String[] args) throws InterruptedException {
            int tasks = 6;
    
            Sinks.Many<String> unicastSink = Sinks.many().unicast().onBackpressureBuffer();
    
            Flux<String> fluxView = unicastSink.asFlux();
            IntStream.range(1, tasks)
                    .forEach(n -> {
                        try {
                            new Thread(() -> {
                                unicastSink.emitNext(doTask(n), Sinks.EmitFailureHandler.FAIL_FAST);
                                log.info("#emitted : {}", n);
                            }).start();
                            Thread.sleep(100L);
                        } catch (InterruptedException e) {
                            log.error(e.getMessage());
                        }
                    });
            fluxView.publishOn(Schedulers.parallel())
                    .map(result -> result + " success!")
                    .doOnNext(n -> log.info("# map() : {}", n))
                    .publishOn(Schedulers.parallel())
                    .subscribe(data -> log.info("# onNext : {}", data));
    
            Thread.sleep(200L);
        }
    
        private static String doTask(int taskNamber) {
            return "task " + taskNamber + " result";
        }
    ```

  create()를 사용한 예제와는 달리 doTask() 메서드가 루프를 돌 때마다 새로운 스레드에서 실행됩니다.

  그리고 작업 처리 결과를 Sinks를 통해 Downstream에 emit 합니다.


### 스레드 안전성

Sinks는 프로그래밍 방식으로 Signal을 전송할 수 있으며, 멀티 스레드 환경에서 스레드 안전성을 보장받을 수 있는 장점이 있습니다.

스레드 안정성이란 함수나 변수 같은 공유 자원에 접근할 경우에도 프로그램의 실행에 문제가 없음을 나타냅니다.

공유 변수에 동시에 접근해서 올바르지 않은 값이 할당된다거나 공유 함수에 동시에 접근함으로써 교착 상태, 즉 Dead Lock에 빠지게 되면 스레드 안전성이 깨지게 됩니다.

Processor에서는 onNext, onComplete, onError 메서드를 직접적으로 호출함으로써 스레드 안전성이 보장되지 않을 수 있는데 Sinks의 경우에는 동시 접근을 감지하고, 동시 접근하는 스레드 중 하나가 빠르게 실패함으로써 스레드 안전성을 보장합니다.

## Sinks 종류 및 특징

### Sinks.One

Sinks.One은 sinks.one() 메서드를 사용해서 한 건의 데이터를 전송하는 방법을 정의해 둔 기능 명세입니다.

```java
public final class Sinks {
	public static <T> Sinks.One<T> one() {
		return SinksSpecs.DEFAULT_ROOT_SPEC.one();
	}
}
```

Sinks.one() 메서드를 호출하는 것은 한 건의 데이터를 프로그래밍 방식으로 emit하는 기능을 사용하고 싶으니 거기에 맞는 적당한 기능 명세를 달라고 요청하는 것 입니다.

다음은 Sinks.One을 사용하는 예제입니다.

```java
    public static void main(String[] args) throws Exception {
        Sinks.One<String> sinkOne = Sinks.one();
        Mono<String> mono = sinkOne.asMono();

        sinkOne.emitValue("Hello Reactor", FAIL_FAST);

        mono.subscribe(data -> log.info("# Subscriber1 : {}", data));
        mono.subscribe(data -> log.info("# Subscriber2 : {}", data));

    }
```

emitValue에 인자로 전달하는 FAIL_FAST는 람다 표현식으로 표현한 EmitFailureHandler 인터페이스의 구현 객체입니다.

FAIL_FAST를 사용하면 에러가 발생했을 때 재시도를 하지 않고 즉시 실패 처리를 하게 됩니다.

이렇게 빠른 실패 처리를 함으로써 스레드 간의 경합으로 발생하는 교착 상태 등을 미연에 방지할 수 있습니다.

### Sinks.Many

Sinks.Many는 sinks.many() 메서드를 사용해서 여러 건의 데이터를 여러 가지 방식으로 전송하는 기능을 정의해 둔 기능 명세입니다.

```java
public final class Sinks {
	public static ManySpec many() {
		return SinksSpecs.DEFAULT_ROOT_SPEC.many();
	}
}
```

Sinks.Many는 데이터 emit을 위한 ManySpec을 리턴하는 것을 볼 수 있습니다.

ManySpec 인터페이스는 다음과 같이 구성되어 있습니다.

```java
public final class Sinks {
	public interface ManySpec {
		UnicastSpec unicast();
		MulticastSpec multicast();
		MulticastReplaySpec replay();
	}
}
```

다음은 Sinks.Many 예제 코드입니다.

```java
    public static void main(String[] args) {
        Sinks.Many<Integer> unicastSinks = Sinks.many().unicast()
                .onBackpressureBuffer();
        Flux<Integer> fluxView = unicastSinks.asFlux();

        unicastSinks.emitNext(1, Sinks.EmitFailureHandler.FAIL_FAST);
        unicastSinks.emitNext(2, Sinks.EmitFailureHandler.FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));

        unicastSinks.emitNext(3, Sinks.EmitFailureHandler.FAIL_FAST);

//        fluxView.subscribe(data -> log.info("# Subscriber2 : {}", data));
    }
```

UnicaseSpec의 경우 하나의 Subscriber에게 데이터를 emit하는 것을 볼 수 있습니다.

만약 아래의 주석을 해제하면 두 번째 Subscriber에게 전달될 수 없기 때문에 예외가 발생합니다.

이번에는 Sinks.Many의 multicastSpec을 사용해보겠습니다.

```java
    public static void main(String[] args) {
        Sinks.Many<Integer> multicastSink = Sinks.many().multicast().onBackpressureBuffer();
        Flux<Integer> fluxView = multicastSink.asFlux();

        multicastSink.emitNext(1, Sinks.EmitFailureHandler.FAIL_FAST);
        multicastSink.emitNext(2, Sinks.EmitFailureHandler.FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));
        fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));

        multicastSink.emitNext(3, Sinks.EmitFailureHandler.FAIL_FAST);
    }
```

실행 결과를 보면 에러는 나지 않지만 Subscriber 2는 세번 째 데이터만 전달받는 것을 볼 수 있습니다.

이는 Sinks가 Publisher의 역할을 할 경우 기본적으로 Hot Publisher로 동작하며, 특히 onBackpressureBuffer() 메서드는 Warm up의 특징을 가지는 Hot Sequence로 동작하기 때문에 첫 번째 구독이 발생한 시점에 Downstream 쪽으로 데이터가 전달되는 것 입니다.

다음은 Sinks.Many의 replay() 메서드를 사용하는 예제입니다.

```java
    public static void main(String[] args) {
        Sinks.Many<Integer> replaySink = Sinks.many().replay().limit(2);
        Flux<Integer> fluxView = replaySink.asFlux();

        replaySink.emitNext(1, Sinks.EmitFailureHandler.FAIL_FAST);
        replaySink.emitNext(2, Sinks.EmitFailureHandler.FAIL_FAST);
        replaySink.emitNext(3, Sinks.EmitFailureHandler.FAIL_FAST);

        fluxView.subscribe(data -> log.info("# Subscriber1: {}", data));

        replaySink.emitNext(4, Sinks.EmitFailureHandler.FAIL_FAST);
        fluxView.subscribe(data -> log.info("# Subscriber2: {}", data));
    }
```

replay() 메서드를 사용하면 emit된 데이터의 최신순 데이터 부터 데이터를 받을 수 있습니다.

현재는 limit 2로 지정되어 있기 때문에 최신의 2개의 데이터를 불러오고 all() 메서드를 사용하게 되면 모든 데이터를 불러오게 됩니다.

## 정리

- Sinks는 Publisher와 Subscriber의 기능을 모두 지닌 Processor의 향상된 기능을 제공합니다.
- 데이터를 emit하는 Sinks에는 크게 Sinks.One과 Sinks.Many가 있습니다.
- Sinks.One은 한 건의 데이터를 프로그래밍 방식으로 emit 합니다.
- Sinks.Many는 여러 건의 데이터를 프로그래밍 방식으로 emit 합니다.
- Sinks.Many의 UnicastSpec은 단 하나의 Subscriber에게만 데이터를 emit 합니다.
- Sinks.Many의 MulticastSpec은 하나 이상의 Subscriber에게 데이터를 emit 합니다.
- Sinks.Many의 MulticastReplaySpec은 emit된 데이터 중에서 특정 시점으로 되돌린 데이터부터 emit 합니다.