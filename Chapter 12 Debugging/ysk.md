# Chapter 12 Debugging

일반 명령형 프로그래밍에서는 stacktrace나 breakpoint로 디버깅하지만,

리액터는 대부분 비동기적으로 실행되고 선언형이여서 디버깅이 쉽지 않다

때문에 다음과 같은 방법을 사용해야 한다

## Debug Mode를 사용한 디버깅

Reactor에서는 디버그 모드(Debug Mode)를 활성화해서 Reactor Sequence를 디버깅할 수 있다.

```java
/**
 * onOperatorDebug() Hook 메서드를 이용한 Debug mode 예
 * - 애플리케이션 전체에서 global 하게 동작한다.
 */
@Slf4j
public class Example12_1 {
    public static Map<String, String> fruits = new HashMap<>();

    static {
        fruits.put("banana", "바나나");
        fruits.put("apple", "사과");
        fruits.put("pear", "배");
        fruits.put("grape", "포도");
    }

    public static void main(String[] args) throws InterruptedException {
        // Hooks.onOperatorDebug(); 이거 주석처리하면 디버깅 할 수가 없다.. 

        Flux
                .fromArray(new String[]{"BANANAS", "APPLES", "PEARS", "MELONS"})
                .subscribeOn(Schedulers.boundedElastic())
                .publishOn(Schedulers.parallel())
                .map(String::toLowerCase)
                .map(fruit -> fruit.substring(0, fruit.length() - 1))
                .map(fruits::get)
                .map(translated -> "맛있는 " + translated)
                .subscribe(
                        log::info,
                        error -> log.error("# onError:", error));

        Thread.sleep(100L);
    }
}

```

주석을 풀고 Hooks.onOperatorDebug()를 키면 스택 트레이스로 드디어 읽을 수 있다! (이전에는 못읽음.. 어디서 발생했는지 알 수가 없다..)

```java
20:50:47.873 [parallel-1] ERROR- # onError:
java.lang.NullPointerException: The mapper [chapter12.Example12_1$$Lambda/0x000000a801126688] returned a null value.
	at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:115)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.FluxMapFuseable] :
	reactor.core.publisher.Flux.map(Flux.java:6580)
	chapter12.Example12_1.main(Example12_1.java:35)
Error has been observed at the following site(s):
	*__Flux.map ⇢ at chapter12.Example12_1.main(Example12_1.java:35)
	|_ Flux.map ⇢ at chapter12.Example12_1.main(Example12_1.java:36)
Original Stack Trace:
		at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:115)
		at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:129)
```

Operator 체인이 시작되기 전 디버그 모드를 활성화하면 에러가 발생한 지점을 명확하게 찾을 수 있다.

그러나. 한계가 있따

* Hooks.onOperatorDebug() 사용의 한계: 디버그 모드의 활성화는 다음과 같이 애플리케이션 내에서 비용이 많이 드는 동작 과정을 거친다. 

  *  애플리케이션 내에 있는 모든 Operator의 스택트레이스Stacktrace를 캡처 한다.

  *  에러가 발생하면 캡처한 정보를 기반으로 에러가 발생한 Assembly의 스택트레이스를 원본 스택트레이스 rgnal Stackrace 중간에 끼워 넣는다.

따라서 에러 원인을 추적하기 위해 처음부터 디버그 모드를 활성화하는 것은 권장하지 않고, 발생하면 로컬에서 켜서 해봐야 한다 

> **Assembly**
>
> Operator에서 리턴하는 새로운 Mono, Flux가 선언된 지점을 Assembly라고 한다.
>
> Reactor의 많은 내부 메커니즘은 "Assembly"와 관련되어 있으며, 이는 주로 디버깅, 로깅, 스레드 로컬(Context 전파) 등의 목적으로 사용된다.

그러면 Production 환경에서 로그를 어떻게 남길까?

**ReactorDebugAgent**를 이용하면 된다. 

Reactor에서는 애플리케이션 내 모든 Operator 체인의 스택트레이스 캡처 비용을 지불하지 않고 디버깅 정보를 추가할 수 있도록 별도의 Java 에이전트를 제공한다. 

1. gradle 설정

```groovy
dependencies {
    // 다른 의존성들
    compile 'io.projectreactor:reactor-tools'
}
```

2. yml 설정

```yaml
# application.yml 또는 application.properties 파일에서 설정
spring:
  reactor:
    debug-agent:
      enabled: true # 디폴트로 true
```

3. main에서 수동으로도 가능

```java
import reactor.tools.agent.ReactorDebugAgent;

@SpringBootApplication
public class MySpringBootApplication {

    public static void main(String[] args) {
        // 수동으로 ReactorDebugAgent를 초기화할 수도 있다.
        // ReactorDebugAgent.init();

        SpringApplication.run(MySpringBootApplication.class, args);
    }
}
```



### checkpoint() Operator를 시용한 디버깅

ReactorDebugAgent는 모든 애플리케이션 전역에서 스택 트레이스를 캡쳐하지만, 

checkPoint() Operator는 특정 Operator 체인 내의 스텍 트레이스만 캡쳐한다.

checkPoint를 사용하는 방법은 세가지다

1. Traceback을 출력하는 방법

실제 에러가 발생한 assembly 지점 또는 에러가 전파된 assembly 지점의 traceback이 추가된다

```java
@Slf4j
public class Example12_2 {
    public static void main(String[] args) {
        Flux
            .just(2, 4, 6, 8)
            .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
            .map(num -> num + 2)
            .checkpoint() // 17번째줄임 
            .subscribe(
                    data -> log.info("# onNext: {}", data),
                    error -> log.error("# onError:", error)
            );
    }
}
//
java.lang.ArithmeticException: / by zero
	at chapter12.Example12_2.lambda$main$0(Example12_2.java:15)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.FluxMap] :
	reactor.core.publisher.Flux.checkpoint(Flux.java:3622)
	chapter12.Example12_2.main(Example12_2.java:17) // 17번째줄 
Error has been observed at the following site(s):
```

* checkPoint를 통해 map()Operator 다음에 추가한 checkpoint 지점까지는 에러가 전파된것을 알 수 있다. 

이번에는 더 정확하게 찾기 위해서 또 checkPoint()를 추가한다.

```java
@Slf4j
public class Example12_3 {
    public static void main(String[] args) {
        Flux
            .just(2, 4, 6, 8)
            .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
            .checkpoint() // 추가 
            .map(num -> num + 2)
            .checkpoint() // 2개임 총 
            .subscribe(
                    data -> log.info("# onNext: {}", data),
                    error -> log.error("# onError:", error)
            );
    }
}
//
java.lang.ArithmeticException: / by zero
	at chapter12.Example12_3.lambda$main$0(Example12_3.java:15)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.FluxZip] :
	reactor.core.publisher.Flux.checkpoint(Flux.java:3622)
	chapter12.Example12_3.main(Example12_3.java:16)
Error has been observed at the following site(s):
	*__checkpoint() ⇢ at chapter12.Example12_3.main(Example12_3.java:16) // 체크포인트지점 
	|_ checkpoint() ⇢ at chapter12.Example12_3.main(Example12_3.java:18) // 체크포인트지점 
Original Stack Trace:
		at chapter12.Example12_3.lambda$main$0(Example12_3.java:15)
		at reactor.core.publisher.FluxZip$PairwiseZipper.apply(FluxZip.java:1190)
		....
		at chapter12.Example12_3.main(Example12_3.java:19)
```

* 첫번째 체크포인트에서 zipWith()에서 에러가 발생했음을 알 수 있다.

2. Traceback 출력 없이 식별자를 포함한 Description을 출력해서 에러 발생 지점을 예상하는 방법

checkpoint (description)를 사용하면 에러 발생 시 Traceback을 생략하고 description을 통해 에러 발생 지점을 예상할 수 있다.

```java
@Slf4j
public class Example12_4 {
    public static void main(String[] args) {
        Flux
            .just(2, 4, 6, 8)
            .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
            .checkpoint("Example12_4.zipWith.checkpoint") //
            .map(num -> num + 2)
            .checkpoint("Example12_4.map.checkpoint") // 여길보면 
            .subscribe(
                    data -> log.info("# onNext: {}", data),
                    error -> log.error("# onError:", error)
            );
    }
}

// 
java.lang.ArithmeticException: / by zero
	at chapter12.Example12_4.lambda$main$0(Example12_4.java:16)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Error has been observed at the following site(s):
	*__checkpoint ⇢ Example12_4.zipWith.checkpoint // 여기도 
	|_ checkpoint ⇢ Example12_4.map.checkpoint  // 여기; 적혀있음 
Original Stack Trace:
		at chapter12.Example12_4.lambda$main$0(Example12_4.java:16)
		....
		at chapter12.Example12_4.main(Example12_4.java:20)
```

* 즉 로그로 설명을 남기는것

3. Traceback과 Description을 모두 출력하는방법

checkpoint(description, forcestackTrace)를 사용하면 description 과 Traceback을 모두 출력할 수 있다.

```java
@Slf4j
public class Example12_5 {
    public static void main(String[] args) {
        Flux
            .just(2, 4, 6, 8)
            .zipWith(Flux.just(1, 2, 3, 0), (x, y) -> x/y)
            .checkpoint("Example12_4.zipWith.checkpoint", true)
            .map(num -> num + 2)
            .checkpoint("Example12_4.map.checkpoint", true)
            .subscribe(
                    data -> log.info("# onNext: {}", data),
                    error -> log.error("# onError:", error)
            );
    }
}
//
java.lang.ArithmeticException: / by zero
	at chapter12.Example12_5.lambda$main$0(Example12_5.java:16)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.FluxZip], described as [Example12_4.zipWith.checkpoint] :
	reactor.core.publisher.Flux.checkpoint(Flux.java:3687)
	chapter12.Example12_5.main(Example12_5.java:17)
Error has been observed at the following site(s):
	*__checkpoint(Example12_4.zipWith.checkpoint) ⇢ at  chapter12.Example12_5.main(Example12_5.java:17) / 여기도 남고 
	|_     checkpoint(Example12_4.map.checkpoint) ⇢ at  chapter12.Example12_5.main(Example12_5.java:19) / 여기도 남고 
Original Stack Trace:
		at chapter12.Example12_5.lambda$main$0(Example12_5.java:16)
		..
		at chapter12.Example12_5.main(Example12_5.java:20)

```

* 근데 그렇게 썩 좋진않다. 여러곳에 흩어져있으면 매번 checkpoint 오퍼레이터를 추가해야 하기 때문이다.

### log() Operator를 사용한 디버깅

log() Operator는 Reactor Sequence의 동작을 로그로 출력하는데, 이 로그를 통해 디버깅이 가능하다.

```java
@Slf4j
public class Example12_7 {
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
//                .log("Fruit.Substring", Level.FINE)
                .map(fruits::get)
                .subscribe(
                        log::info,
                        error -> log.error("# onError:", error));
    }
}
//
INFO - | onSubscribe([Fuseable] FluxMapFuseable.MapFuseableSubscriber)
INFO - | request(unbounded)
INFO - | onNext(banana)
INFO - 바나나
INFO - | onNext(apple)
INFO - 사과
INFO - | onNext(pear)
INFO - 배
INFO - | onNext(melon)
INFO - | cancel()
ERROR- # onError:
java.lang.NullPointerException: The mapper [chapter12.Example12_7$$Lambda/0x000000f00110f058] retu
```

* log() 오퍼레티러를 추가하면 Subscriber에 전달된 결과 이외에 onNext, request 등 시그널들이 출력된다.
  * 바나나,사과,배 다음 포도가 출력되야 하는데 포도가 출력이 되지 않고 melon을 처리하려는데 hashmap에 없으므로 npe가 발생했단것을 알 수 있다. 그렇다면 map에서 발생했단걸 의심할 수 있다.
  * 그러나 로그 레벨이 모두 똑같다보니 분석하기가 쉽지 않다면?
* ..log("Fruit.Substring", Level.FINE)로 처리하면 더 보기 쉽다. (DEBUG 레벨)



그러나 여전히 리액터의 디버깅은 쉽지 않다. 