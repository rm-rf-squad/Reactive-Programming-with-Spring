# Testing

## StepVerifier를 사용한 테스팅

Reactor에서 가장 일반적인 테스트 방식은 Flux 또는 Mono를 Reactor Sequence로 정의한 후, 구독 시점에 해당 Operator 체인이 시나리오대로 동작하는지를 테스트하는 것 입니다.

예를 들어 Reactor Sequence에서 다음에 발생할 Signal이 무엇인지, 기대하던 데이터들이 emit되었는지, 특정 시간 동안 emit된 데이터가 있는지 등을 단계적으로 테스트할 수 있습니다.

### Signal 이벤트 테스트

StepVerifier를 이용한 가장 기본적인 테스트 방식은 바로 Reactor Sequence에서 발생하는 Signal 이벤트를 테스트하는 것입니다.

```java
@Test
@DisplayName("시그널 테스트")
void test1() {
    StepVerifier.create(Mono.just("Hello Reactor")) // 테스트 대상 Sequence 생성
            .expectNext("Hello Reactor") // emit된 데이터 기댓값 평가
            .expectComplete() // onComplete Signal 기댓값 평가
            .verify(); // 검증 실행
}
```

StepVerifier API를 이용한 Operator 체인의 기본적인 테스트 방법은 다음과 같습니다.

- create()을 통해 테스트 대상 Sequence를 생성합니다.
- expectXXXX()를 통해 Sequence에서 예상되는 Signal의 기댓값을 평가합니다.
- verifyXXXX()를 호출함으로써 전체 Operator 체인의 테스트를 트리거합니다.

### expectXXXX() 메서드

| 메서드 | 설명 |
| --- | --- |
| expectSubscription() | 구독이 이루어짐을 기대합니다. |
| expectNext(T t) | onNext Signal을 통해 전달되는 값이 파라미터로 전달된 값과 같음을 기대합니다. |
| expectComplete() | onComplete Signal이 전송되기를 기대합니다. |
| expectError() | onError Signal이 전송되기를 기대합니다. |
| expectNextCount(long count) | 구독 시점 또는 이전(previous) expectNext()를 통해 기댓값이 평가된 데이터 이후부터 emit된 수를 기대합니다. |
| expectNoEvent(Duration duration) | 주어진 시간 동안 Signal 이벤트가 발생하지 않았음을 기대합니다. |
| expectAccessibleContext() | 구독 시점 이후에 Context가 전파되었음을 기대합니다. |
| expectNextSequence(Iterable <? extends T> iterable) | emit된 데이터들이 파라미터로 전달된 Iterable의 요소와 매치됨을 기대합니다. |

### VerifyXXXX() 메서드

verifyXXXX() 메서드가 호출되면 내부적으로 테스트 대상 Operator 체인의 대한 구독이 이루어지고 기댓값을 평가하게 됩니다.

| 메서드 | 설명 |
| --- | --- |
| verify() | 검증을 트리거합니다. |
| verifyComplete | 검증을 트리거 하고, onComplete Signal을 기대합니다. |
| verifyError() | 검증을 트리거하고, onError Signal을 기대합니다. |
| verifyTimeout(Duration duration) | 검증을 트리거하고, 주어진 시간이 초과되어도 Publisher가 종료되지 않음을 기대합니다. |

### as() 사용하여 설명 추가

```java
public static Flux<String> sayHello() {
    return Flux.just("Hello", "Reactor");
}

@Test
@DisplayName("시그널 테스트 2")
void test2() {
    StepVerifier.create(sayHello())
            .expectSubscription()
            .as("# expect subscription")
            .expectNext("Hi") // Hi가 아닌 Hello가 emit되므로 테스트 실패
            .as("# expect Hi")
            .expectNext("Reactor")
            .as("# expect Reactor")
            .verifyComplete();
}
```

as() 메서드는 이전 기댓값 평가 단계에 대해 설명을 추가할 수 있습니다.

만약 테스트에 실패하게 되면 다음과 같이 실패한 단계에 해당하는 설명이 출력됩니다.

```java
expectation "# expect Hi" failed (expected value: Hi; actual value: Hello)
java.lang.AssertionError: expectation "# expect Hi" failed (expected value: Hi; actual value: Hello)
```

### Error 시그널 테스트

```java
public static Flux<Integer> divideByTwo(Flux<Integer> source) {
    return source.zipWith(Flux.just(2, 2, 2, 2, 0), (x, y) -> x / y);
}
@Test
@DisplayName("시그널 테스트 3")
void test3() {
    Flux<Integer> source = Flux.just(2, 4, 6, 8, 10);

    StepVerifier.create(divideByTwo(source))
            .expectNext(1)
            .expectNext(2)
            .expectNext(3)
            .expectNext(4)
            .expectError(ArithmeticException.class)
            .verify();
}
```

2, 4, 6, 8은 2로 나누기 때문에 정상적으로 결과가 나오지만 5번째에 0으로 나누기 때문에 ArithmeticException이 발생합니다.

이때 expectError()를 통해 에러를 기대하기 때문에 최종 테스트 결과는 passed가 됩니다.

### StepVerifierOptions 예시

```java
public static Flux<Integer> takeNumber(Flux<Integer> source, long n) {
  return source.take(n);
}

@Test
@DisplayName("시그널 테스트 4")
void test4() {
    Flux<Integer> source = Flux.range(0, 1000);
    StepVerifier.create(takeNumber(source, 500), StepVerifierOptions.create().scenarioName("Verify from 0 to 499"))
            .expectSubscription()
            .expectNext(0)
            .expectNextCount(498)
            .expectNext(500)
            .expectComplete()
            .verify();
}
```

StepVerifierOptions는 이름 그대로 StepVerifer에 옵션, 즉 추가적인 기능을 덧붙이는 작업을 하는 클래스인데, 예제 코드에서는 테스트에 실패할 경우 파라미터로 입력한 시나리오명을 출력합니다.

다음과 같이 에러가 발생하면 시나리오 이름이 출력됩니다.

```java
[Verify from 0 to 499] expectation "expectNext(500)" failed (expected value: 500; actual value: 499)
```

## 시간 기반(Time-based) 테스트

StepVerifier는 가상의 시간을 이용해 미래에 실행되는 Reactor Sequence의 시간을 앞당겨 테스트할 수 있는 기능을 지원합니다.

```java
public static Flux<Tuple2<String, Integer>> getCOVID19Count(Flux<Long> source) {
    return source
            .flatMap(notUse -> Flux.just(
                    Tuples.of("서울", 10),
                    Tuples.of("경기도", 5),
                    Tuples.of("강원도", 3),
                    Tuples.of("충청도", 6),
                    Tuples.of("경상도", 5),
                    Tuples.of("전라도", 8),
                    Tuples.of("인천", 2),
                    Tuples.of("대전", 1),
                    Tuples.of("대구", 2),
                    Tuples.of("부산", 3),
                    Tuples.of("제주도", 0)
            ));
}

@Test
@DisplayName("타임 기반 테스트 1")
void test1() {
    StepVerifier.withVirtualTime(() -> getCOVID19Count(Flux.interval(Duration.ofHours(1)).take(1)))
            .expectSubscription()
            .then(() -> VirtualTimeScheduler
                    .get()
                    .advanceTimeBy(Duration.ofHours(1)))
            .expectNextCount(11)
            .expectComplete()
            .verify();
}
```

VirtualTimeScheduler를 사용하여 실제 1시간 기다려야 하는 상황을 기다리지 않고 바로 처리할 수 있게 됩니다.

### 지정한 시간 내에 테스트가 완료되는지 테스트

```java
@Test
@DisplayName("타임 기반 테스트 2")
void test2() {
    StepVerifier.create(getCOVID19Count(Flux.interval(Duration.ofMinutes(1)).take(1)))
            .expectSubscription()
            .expectNextCount(11)
            .expectComplete()
            .verify(Duration.ofSeconds(3));
}
```

verify(Duration)으로 3초 이내 테스트가 완료되는 것을 기대했지만 1분뒤에 emit되도록 설정했기 때문에 시간 초과로 인해 에러가 발생합니다.

```java
VerifySubscriber timed out on reactor.core.publisher.FluxFlatMap$FlatMapMain@76012793
```

### expectNoEvent 이용하기

```java
public static Flux<Tuple2<String, Integer>> getVoteCount(Flux<Long> source) {
    return source
            .zipWith(Flux.just(
                    Tuples.of("중구", 15400),
                    Tuples.of("서초구", 20020),
                    Tuples.of("강서구", 32040),
                    Tuples.of("강동구", 14506),
                    Tuples.of("서대문구", 35650)
            ))
            .map(Tuple2::getT2);
}

@Test
@DisplayName("타임 기반 테스트 3")
void test3() {
    StepVerifier.withVirtualTime(() -> getVoteCount(Flux.interval(Duration.ofMinutes(1))))
            .expectSubscription()
            .expectNoEvent(Duration.ofMinutes(1))
            .expectNoEvent(Duration.ofMinutes(1))
            .expectNoEvent(Duration.ofMinutes(1))
            .expectNoEvent(Duration.ofMinutes(1))
            .expectNoEvent(Duration.ofMinutes(1))
            .expectNextCount(5)
            .expectComplete()
            .verify();
}
```

expectNoEvent()를 사용해서 지정한 시간 동안 어떤 Signal 이벤트도 발생하지 않았음을 기대하는 예제입니다.

expectNoEvent()의 파리미터로 시간을 지정하면 지정한 시간 동안 어떤 이벤트도 발생하지 않을 것이라고 기대하는 동시에 지정한 시간만큼 시간을 앞당깁니다.

결과적으로 5분의 시간을 앞당길 수 있게 됩니다.

## Backpressure 테스트

```java
public static Flux<Integer> generateNumber() {
    return Flux.create(emitter -> {
        for (int i = 1; i <= 100; i++) {
            emitter.next(i);
        }
        emitter.complete();
    }, FluxSink.OverflowStrategy.ERROR);
}
@Test
void generateNumberTest() {
    StepVerifier.create(generateNumber(), 1L)
            .thenConsumeWhile(num -> num >= 1)
            .verifyComplete();
}
```

위의 테스트는 generateNumber() 메서드에서 한 번에 100개의 숫자 데이터를 emit하는데 StepVerifer의 create() 메서드에서 데이터의 요청 개수를 1로 지정해서 오버플로우가 발생했기 때문에 테스트 결과는 failed가 됩니다.

thenConsumeWhile() 메서드를 이용해서 emit되는 데이터를 소비하고 있지만 예상한 것보다 더 많은 데이터를 수신함으로써 결국에 오버플로우가 발생하게 됩니다.

다음과 같이 수정하면 성공하게 됩니다.

```java
@Test
void generateNumberTest2() {
    StepVerifier.create(generateNumber(), 1L)
            .thenConsumeWhile(num -> num >= 1)
            .expectError()
            .verifyThenAssertThat()
            .hasDroppedElements();
}
```

## Context 테스트

```java
public static Mono<String> getSecretMessage(Mono<String> keySource) {
    return keySource.zipWith(Mono.deferContextual(ctx -> Mono.just((String) ctx.get("secretKey"))))
            .filter(tp -> tp.getT1().equals(new String(Base64.getDecoder().decode(tp.getT2()))))
            .transformDeferredContextual((mono, ctx) -> mono.map(notUse -> ctx.get("secretMessage")));
}

@Test
void getSecretMessageTest() {
    Mono<String> source = Mono.just("hello");

    StepVerifier.create(getSecretMessage(source).contextWrite(context -> context.put("secretMessage", "Hello, Reactor"))
                    .contextWrite(context -> context.put("secretKey", "aGVsbG8=")))
            .expectSubscription()
            .expectAccessibleContext()
            .hasKey("secretKey")
            .hasKey("secretMessage")
            .then()
            .expectNext("Hello, Reactor")
            .expectComplete()
            .verify();
}
```

위의 코드는 다음과 같은 단계로 기댓값을 평가합니다.

- expectSubscription()으로 구독이 발생함을 기대합니다.
- expectAccessibleContext()로 구독 이후, Context가 전파됨을 기대합니다.
- hasKey()로 전파된 Context에 “secretKey” 키에 해당하는 값이 있음을 기대합니다.
- hasKey()로 전파된 Context에 “secretMessage” 키에 해당하는 값이 있음을 기대합니다.
- then() 메서드로 Sequence의 다음 Signal 이벤트의 기댓값을 평가할 수 있도록 합니다.
- expectNext()로 “Hello, Reactor” 문자열이 emit 되었음을 기대합니다.
- expectComplete()으로 onComplete Signal이 전송됨을 기대합니다.

## Record 기반 테스트

Reactor Sequence를 테스트하다보면 expectNext()로 emit된 데이터의 단순 기댓값만 평가하는 것이 아니라 조금 더 구체적인 조건으로 Assertion 해야하는 경우가 많습니다.

이때 recordWith()를 사용할 수 있습니다.

recordWith()는 파라미터로 전달한 Java 컬렉션에 emit된 데이터를 추가하는 세션을 시작합니다.

```java
public static Flux<String> getCapitalizedCountry(Flux<String> source) {
    return source
            .map(country -> country.substring(0, 1).toUpperCase() + country.substring(1));
}

@Test
void getCityTest() {
    StepVerifier.create(getCapitalizedCountry(Flux.just("korea", "england", "canada", "india")))
            .expectSubscription()
            .recordWith(ArrayList::new)
            .thenConsumeWhile(country -> !country.isEmpty())
            .consumeRecordedWith(countries -> {
                assertThat(countries.stream()
                                .allMatch(country -> Character.isUpperCase(country.charAt(0))),
                        is(true));
            })
            .expectComplete()
            .verify();
}
```

테스트는 다음과 같이 진행됩니다.

- expectSubscription()으로 구독이 발생함을 기대합니다.
- recordWith()로 emit된 데이터에 대한 기록을 시작합니다.
- thenConsumeWhile()로 파라미터로 전달한 Predicate과 일치하는 데이터는 다음 단계에서 소비할 수 있도록 합니다.
- consumeRecordedWith()로 컬렉션에 기록된 데이터를 소비합니다. 여기서는 모든 데이터의 첫 글자가 대문자인지 여부를 확인함으로써 getCapitalizedCountry() 메서드를 Assertion 합니다.
- expectComplete()으로 onComplete Signal이 전송됨을 기대합니다.

## 정리

- StepVerifier를 사용한 테스트 유형
    - Reactor Sequence에서 발생한 Signal 이벤트를 테스트할 수 있습니다.
    - withVirtualTime()과 VirtualTimeScheduler() 등을 이용해서 시간 기반 테스트를 진행할 수 있습니다.
    - thenConsumeWhile()을 이용해서 Backpressure 테스트를 진행할 수 있습니다.
    - expectAccessibleContext()를 이용해서 Sequence에 전파되는 Context를 테스트할 수 있습니다.
    - recordWith(), consumeRecordedWith(), expectRecordedMatchers() 등을 이용해서 Record 기반 테스트를 진행할 수 있습니다.

## TestPublisher를 사용한 테스팅

### 정상 동작하는 TestPublisher

TestPublisher를 사용하면, 개발자가 직접 프로그래밍 방식으로 Signal을 발생시키면서 원하는 상황을 미세하게 재연하며 테스트를 진행할 수 있습니다.

```java
public static Flux<Integer> divideByTwo(Flux<Integer> source) {
    return source.zipWith(Flux.just(2, 2, 2, 2, 0), (x, y) -> x / y);
}
@Test
void divideByTwoTest() {
    TestPublisher<Integer> source = TestPublisher.create();

    StepVerifier.create(divideByTwo(source.flux()))
            .expectSubscription()
            .then(() -> source.emit(2, 4, 6, 8, 10))
            .expectNext(1, 2, 3, 4)
            .expectError()
            .verify();
}
```

위의 코드는 다음과 같이 테스트가 진행됩니다.

- Testpublisher.create()으로 TestPublisher를 생성합니다.
- 테스트 대상 클래스에 파라미터로 Flux를 전달하기 위해 flux() 메서드를 이용하여 Flux를 변환합니다.
- 마지막으로 emit() 메서드를 사용해서 테스트에 필요한 데이터를 emit 합니다.

### TestPublisher가 발생시키는 Signal 유형

- next(T) 또는 next(T, T…)
    - 1개 이상의 onNext Signal을 발생시킵니다.
- emit(T…)
    - 1개 이상의 onNext Signal을 발생시킨 후 onComplete Signal 발생시킵니다.
- complete()
    - onComplete Signal을 발생시킵니다.
- error(Throwable)
    - onError Signal을 발생시킵니다.

### 오동작하는(Misbehaving) Test Publisher

오동작하는 TestPublisher를 생성하여 리액티브 스트림즈의 사양을 위반하는 상황이 발생하는지를 테스트할 수 있습니다.

<aside>
💡 오동작하는 TestPublisher라는 말의 의미는 리액티브 스트림즈 사양 위반 여부를 사전에 체크하지 않는다는 의미입니다.
따라서, 리액티브 스트림즈 사양에 위반되더라도 TestPublisher는 데이터를 emit할 수 있습니다.

</aside>

```java
        public static Flux<Integer> divideByTwo(Flux<Integer> source) {
            return source.zipWith(Flux.just(2, 2, 2, 2, 0), (x, y) -> x / y);
        }
        private static List<Integer> getDataSource() {
            return List.of(2, 4, 6, 8, null);
        }
        @Test
        void divideByTwoTest() {
            TestPublisher<Integer> source = TestPublisher.createNoncompliant(TestPublisher.Violation.ALLOW_NULL);

            StepVerifier.create(divideByTwo(source.flux()))
                    .expectSubscription()
                    .then(() -> {
                        getDataSource()
                                .forEach(source::next);
                        source.complete();
                    })
                    .expectNext(1, 2, 3, 4, 5)
                    .expectComplete()
                    .verify();
        }
```

오동작하는 TestPublisher로서 동작하도록 ALLOW_NULL 위반 조건을 지정하여 데이터의 값이 null이라도 정상동작하는 TestPublisher를 생성한 것을 볼 수 있습니다.

실행 결과를 보면 NullPointerException이 발생한 것을 볼 수 있습니다.

```java
java.lang.NullPointerException
```

### 오동작하는 TestPublisher를 생성하기 위한 위반 조건

- ALLOW_NULL
    - 전송할 데이터가 null이어도 NullPointerException을 발생시키지 않고 다음 호출을 진행할 수 있도록 합니다.
- CLEANUP_ON_TERMINATE
    - onComplete, onError, emit 같은 Terminal Signal을 연달아 여러 번 보낼 수 있도록 합니다.
- DEFER_CANCELLATION
    - cancel Signal을 무시하고 계속해서 Signal을 emit할 수 있도록 합니다.
- REQUEST_OVERFLOW
    - 요청 개수보다 더 많은 Signal이 발생하더라도 IllegalStateException을 발생시키지 않고 다음 호출을 진행할 수 있도록 합니다.

## 정리

- TestPublisher를 사용하면 개발자가 직접 프로그래밍 방식으로 Signal을 발생시키면서 원하는 상황을 미세하게 재연하며 테스트를 진행할 수 있습니다.
- 오동작하는 TestPublisher를 생성하여 리액티브 스트림즈의 사양을 위반하는 상황이 발생하는지를 테스트할 수 있습니다.

## PublisherProbe를 사용한 테스팅

PublisherProbe를 이용해 Sequence의 실행 경로를 테스트할 수 있습니다.

주로 조건에 따라 Sequence가 분기되는 경우, Sequence의 실행 경로를 추적해서 정상적으로 실행되었는지 테스트할 수 있습니다.

```java
    // 평소에는 주전력을 사용해서 작업하다 주전력이 끊겼을 때 예비 전력을 사용해 작업을 진행하는 상황을 시뮬레이션 하기 위한 메서드
    public static Mono<String> processTask(Mono<String> main, Mono<String> standby) {
        return main.flatMap(Mono::just)
                .switchIfEmpty(standby);
    }

    public static Mono<String> supplyMainPower() {
        return Mono.empty();
    }

    public static Mono<String> supplyStandbyPower() {
        return Mono.just("Standby Power");
    }

    @Test
    void publisherProbeTest() {
        PublisherProbe<String> probe = PublisherProbe.of(supplyStandbyPower());

        StepVerifier.create(processTask(supplyMainPower(), probe.mono()))
                .expectNextCount(1)
                .verifyComplete();

        probe.assertWasSubscribed();
        probe.assertWasRequested();
        probe.assertWasNotCancelled();
    }
```

위의 테스트 코드는 Sequence의 Signal 이벤트뿐만 아니라 switchIfEmpty() Operator로 인해 Sequence가 분기되는 상황에서 실제로 어느 Publisher가 동작하는지 해당 Publisher의 실행 경로를 테스트하는 것입니다.

PublisherProbe 클래스의 메서드들을 통해 구독을 했는지, 요청을 했는지, 중간에 취소가 되지 않았는지를 검증할 수 있게 됩니다.

## 정리

- PublisherProbe를 사용하면 Sequence의 실행이 분기되는 상황에서 Publisher가 어느 경로로 실행되는지 알 수 있습니다.
- PublisherProbe의 assertWasSubscribed(), assertWasRequested(), assertWasNotCancelled() 등을 통해 Sequence의 기대하는 실행 경로를 Assertion할 수 있습니다.