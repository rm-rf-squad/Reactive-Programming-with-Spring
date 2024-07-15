# Debugging

Reactor는 처리되는 작업들이 대부분 비동기적으로 실행되고 Reactor  Sequence는 선언형 프로그래밍 방식으로 구성되므로 디버깅이 쉽지 않습니다.

## Debug Mode를 사용한 디버깅

Reactor에서는 디버그 모드를 활성화해서 Reactor Sequence를 디버깅할 수 있습니다.

```java
    public static void main(String[] args) throws InterruptedException {
        Hooks.onOperatorDebug();

        Flux.fromArray(new String[]{"BANANAS", "APPLES", "PEARS", "MELONS"})
                .subscribeOn(Schedulers.boundedElastic())
                .publishOn(Schedulers.parallel())
                .map(String::toLowerCase)
                .map(fruit -> fruit.substring(0, fruit.length() - 1))
                .map(fruits::get)
                .map(translated -> "맛있는 " + translated)
                .subscribe(log::info, error -> log.error("# onError: ", error));

        Thread.sleep(100L);
    }
```

<aside>
💡 현재 코드에서는 map Operator가 많은 것을 볼 수 있는데 IDE에서 map Operator가 너무 많아서 성능상의 오버헤드가 발생할 수 있다는 경고 메시지를 표시합니다.

실무에서는 이런 식의 코드를 지양하는게 좋습니다.

</aside>

디버깅 모드를 활성화하면 다음과 같이 자세한 에러 메시지를 볼 수 있습니다.

```java
java.lang.NullPointerException: The mapper [org.example.webfluxplayground.debug.DebugExample$$Lambda/0x000000c80109b7d0] returned a null value.
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:115)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.FluxMapFuseable] :
	reactor.core.publisher.Flux.map(Flux.java:6580)
	org.example.webfluxplayground.debug.DebugExample.main(DebugExample.java:30)
Error has been observed at the following site(s):
	*__Flux.map ⇢ at org.example.webfluxplayground.debug.DebugExample.main(DebugExample.java:30)
	|_ Flux.map ⇢ at org.example.webfluxplayground.debug.DebugExample.main(DebugExample.java:31)
```

Operator 체인상에서 에러가 발생한 지점을 정확하게 가리키고 있으며, 에러가 시작된 지점부터 에러 전파 상태를 친절하게 표시해주고 있습니다.

이처럼 Operator 체인이 시작되기 전에 디버그 모드를 활성화하면 에러가 발생한 지점을 좀 더 명확하게 찾을 수 있습니다.

- Hooks.onOperatorDebug() 사용의 한계 : 디버그 모드의 활성화는 다음과 같이 애플리케이션 내에서 비용이 많이 드는 동작 과정을 거칩니다.
    - 애플리케이션 내에 있는 모든 Operator의 스택트레이스를 캡처합니다.
    - 에러가 발생하면 캡처한 정보를 기반으로 에러가 발생한 Assembly의 스택트레이스를 원본 스택트레이스 중간에 끼워 넣습니다.

따라서 에러 원인을 추적하기 위해 처음부터 디버그 모드를 활성화하는 것은 권장되지 않습니다.

### Assenbly

Reactor의 Operator 들은 대부분 Mono 또는 Flux를 리턴하기 때문에 Operator 체인을 형성할 수 있습니다.

이처럼 Operator에서 리턴하는 새로운 Mono 또는 Flux가 선언된 지점을 Assembly라고 합니다.

### Traceback

디버그 모드를 활성화하면 Operator의 Assembly 정보를 캡처하는데 이중에서 에러가 발생한 Operator의 스택트레이스를 캡처한 Assembly 정보를 TraceBack이라고 합니다.

Traceback은 Suppressed Exception 형태로 원본 스택트레이스에 추가됩니다.

### 프로덕션(Production) 환경에서의 디버깅 설정

Reactor에서는 애플리케이션 내 모든 Operator 체인의 스택트레이스 캡처 비용을 지불하지 않고 디버깅 정보를 추가할 수 있도록 별도의 Java 에이전트를 제공합니다.

예를 들어, 여러분이 Spring WebFlux 기반의 애플리케이션을 제작하여 프로덕션 환경에서 사용할 목적이라면 reactor-tools를 추가하여 Reactor Debug Agent를 활성화할 수 있습니다.



## checkpoint() Operator를 사용한 디버깅

checkpoint() Operator를 사용하면 특정 Operator 체인 내의 스택트레이스만 캡처할 수 있습니다.

### Traceback을 출력하는 방법

checkpoint()를 사용하면 실제 에러가 발생한 assembly 지점 또는 에러가 전파된 assembly 지점의 traceback이 추가됩니다.

```java
        Flux.just(2, 4, 6, 8)
                .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x / y)
                .checkpoint()
                .map(num -> num + 2)
                .subscribe(
                        data -> log.info("# onNext: {}", data),
                        error -> log.error("# onError :", error)
                );
```

실행해보면 다음과 같이 trackback이 출력되는 것을 볼 수 있습니다.

```java
java.lang.ArithmeticException: / by zero
	at org.example.webfluxplayground.debug.DebugExample2.lambda$main$0(DebugExample2.java:10)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.FluxZip] :
	reactor.core.publisher.Flux.checkpoint(Flux.java:3622)
	org.example.webfluxplayground.debug.DebugExample2.main(DebugExample2.java:11)
Error has been observed at the following site(s):
	*__checkpoint() ⇢ at org.example.webfluxplayground.debug.DebugExample2.main(DebugExample2.java:11)
```

checkpoint를 통해 어디서 에러가 발생했는지 예측할 수 있기 때문에 해당 지점을 확인하여 문제를 해결할 수 있습니다.

### Traceback 출력 없이 식별자를 포함한 Description을 출력해서 에러 발생 지점을 예상하는 방법

checkpoint(description)를 사용하면 에러 발생시 Traceback을 생략하고 description을 통해 에러 발생 지점을 예상할 수 있습니다.

```java
        Flux.just(2, 4, 6, 8)
                .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x / y)
                .checkpoint("example.zipWith.checkpoint")
                .map(num -> num + 2)
                .checkpoint("example.map.checkpoint")
                .subscribe(data -> log.info("# onNext: {}", data), error -> log.error("# onError :", error));
```

checkpoin()에 description을 추가해 Traceback 대신에 description을 출력하도록 했습니다.

```java
Error has been observed at the following site(s):
	*__checkpoint ⇢ example.zipWith.checkpoint
	|_ checkpoint ⇢ example.map.checkpoint
```

실행 결과를 보면 checkpoin()의 파라미터로 입력한 description이 출력된 것을 확인할 수 있습니다.

### Traceback과 Description을 모두 출력하는 방법

checkpoint(description, forceStackTrace)를 사용하면 description과 Traceback을 모두 출력할 수 있습니다.

```java
        Flux.just(2, 4, 6, 8)
                .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x / y)
                .checkpoint("example.zipWith.checkpoint", true)
                .map(num -> num + 2)
                .checkpoint("example.map.checkpoint", true)
                .subscribe(data -> log.info("# onNext: {}", data), error -> log.error("# onError :", error));
```

traceback과 description이 모두 출력되는 것을 볼 수 있습니다.

```java
Error has been observed at the following site(s):
	*__checkpoint(example.zipWith.checkpoint) ⇢ at org.example.webfluxplayground.debug.DebugExample4.main(DebugExample4.java:11)
	|_     checkpoint(example.map.checkpoint) ⇢ at org.example.webfluxplayground.debug.DebugExample4.main(DebugExample4.java:13)
Original Stack Trace:
		at org.example.webfluxplayground.debug.DebugExample4.lambda$main$0(DebugExample4.java:10)
		at reactor.core.publisher.FluxZip$PairwiseZipper.apply(FluxZip.java:1190)
		at reactor.core.publisher.FluxZip$PairwiseZipper.apply(FluxZip.java:1179)
		at reactor.core.publisher.FluxZip$ZipCoordinator.drain(FluxZip.java:926)
		at reactor.core.publisher.FluxZip$ZipInner.onSubscribe(FluxZip.java:1094)
		at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:53)
		at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:59)
		at reactor.core.publisher.Flux.subscribe(Flux.java:8840)
		at reactor.core.publisher.FluxZip$ZipCoordinator.subscribe(FluxZip.java:731)
		at reactor.core.publisher.FluxZip.handleBoth(FluxZip.java:318)
		at reactor.core.publisher.FluxZip.handleArrayMode(FluxZip.java:273)
		at reactor.core.publisher.FluxZip.subscribe(FluxZip.java:137)
		at reactor.core.publisher.Flux.subscribe(Flux.java:8840)
		at reactor.core.publisher.Flux.subscribeWith(Flux.java:8961)
		at reactor.core.publisher.Flux.subscribe(Flux.java:8805)
		at reactor.core.publisher.Flux.subscribe(Flux.java:8729)
		at reactor.core.publisher.Flux.subscribe(Flux.java:8699)
		at org.example.webfluxplayground.debug.DebugExample4.main(DebugExample4.java:14)
```

### log() Operator를 사용한 디버깅

log() Operator는 Reactor Sequence의 동작을 로그로 출력하는데 이 로그를 통해 디버깅이 가능합니다.

```java
    public static Map<String, String> fruits = new HashMap<>();

    static {
        fruits.put("banana", "바나나");
        fruits.put("apple", "사과");
        fruits.put("pear", "배");
        fruits.put("grape", "포도");
    }

    public static void main(String[] args) {
        Flux.fromArray(new String[]{"BANANAS", "APPLES", "PEARS", "MELONS"})
                .map(String::toLowerCase)
                .map(fruit -> fruit.substring(0, fruit.length() - 1))
                .log()
                .map(fruits::get)
                .subscribe(log::info, error -> log.error("# onError: ", error));
    }
```

log() Operator를 추가하면 Subscriber에 전달된 결과 이외에 onSubscribe(), request(), onNext() 같은 Signal이 출력되었습니다.

```java
13:57:51.028 [main] INFO reactor.Flux.MapFuseable.1 -- | onSubscribe([Fuseable] FluxMapFuseable.MapFuseableSubscriber)
13:57:51.030 [main] INFO reactor.Flux.MapFuseable.1 -- | request(unbounded)
13:57:51.030 [main] INFO reactor.Flux.MapFuseable.1 -- | onNext(banana)
13:57:51.030 [main] INFO org.example.webfluxplayground.debug.DebugExample5 -- 바나나
13:57:51.030 [main] INFO reactor.Flux.MapFuseable.1 -- | onNext(apple)
13:57:51.030 [main] INFO org.example.webfluxplayground.debug.DebugExample5 -- 사과
13:57:51.030 [main] INFO reactor.Flux.MapFuseable.1 -- | onNext(pear)
13:57:51.030 [main] INFO org.example.webfluxplayground.debug.DebugExample5 -- 배
13:57:51.030 [main] INFO reactor.Flux.MapFuseable.1 -- | onNext(melon)
13:57:51.032 [main] INFO reactor.Flux.MapFuseable.1 -- | cancel()
13:57:51.032 [main] ERROR org.example.webfluxplayground.debug.DebugExample5 -- # onError: 
java.lang.NullPointerException: The mapper [org.example.webfluxplayground.debug.DebugExample5$$Lambda/0x0000000801086088] returned a null value.
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:115)
	at reactor.core.publisher.FluxPeekFuseable$PeekFuseableSubscriber.onNext(FluxPeekFuseable.java:210)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:129)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:129)
	at reactor.core.publisher.FluxArray$ArraySubscription.fastPath(FluxArray.java:171)
	at reactor.core.publisher.FluxArray$ArraySubscription.request(FluxArray.java:96)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.request(FluxMapFuseable.java:171)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.request(FluxMapFuseable.java:171)
	at reactor.core.publisher.FluxPeekFuseable$PeekFuseableSubscriber.request(FluxPeekFuseable.java:144)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.request(FluxMapFuseable.java:171)
	at reactor.core.publisher.LambdaSubscriber.onSubscribe(LambdaSubscriber.java:119)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onSubscribe(FluxMapFuseable.java:96)
	at reactor.core.publisher.FluxPeekFuseable$PeekFuseableSubscriber.onSubscribe(FluxPeekFuseable.java:178)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onSubscribe(FluxMapFuseable.java:96)
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onSubscribe(FluxMapFuseable.java:96)
	at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:53)
	at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:59)
	at reactor.core.publisher.Flux.subscribe(Flux.java:8840)
	at reactor.core.publisher.Flux.subscribeWith(Flux.java:8961)
	at reactor.core.publisher.Flux.subscribe(Flux.java:8805)
	at reactor.core.publisher.Flux.subscribe(Flux.java:8729)
	at reactor.core.publisher.Flux.subscribe(Flux.java:8699)
	at org.example.webfluxplayground.debug.DebugExample5.main(DebugExample5.java:26)
```

log() Operator에 파라미터를 다음과 같이 추가하면 로그 레벨과 description을 추가할 수 있습니다.

```java
    public static void main(String[] args) {
        Flux.fromArray(new String[]{"BANANAS", "APPLES", "PEARS", "MELONS"})
                .map(String::toLowerCase)
                .map(fruit -> fruit.substring(0, fruit.length() - 1))
                .log("Fruit.Substring", Level.FINE)
                .map(fruits::get)
                .subscribe(log::info, error -> log.error("# onError: ", error));
    }
```

<aside>
💡 Level.FINE은 Slf4j 로깅 프레임워크에서 로그 레벨중 DEBUG 레벨에 해당됩니다.

</aside>

log() Operator를 사용하면 에러가 발생한 지점에 단계적으로 접근할 수 있기 때문에 Reactor Sequence의 내부 동작을 좀 더 상세하게 분석하면서 디버깅할 수 있습니다.