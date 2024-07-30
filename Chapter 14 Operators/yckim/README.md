# Operators

리액티브 프로그래밍에서 Operator는 데이터를 변환하고 조작하기 위한 기본 구성 요소입니다.

리액티브 스트림의 데이터를 처리하는 데 사용되며, 데이터를 필터링, 변환, 결합, 재시도, 버퍼링 등 다양한 방식으로 처리할 수 있습니다.

Operator는 스트림에서 발생하는 이벤트를 입력으로 받아들이고, 이를 변환하여 다음 단계로 전달합니다.

## Sequence 생성을 위한 Operator

### justOrEmpty

justOrEmpty()는 just()의 확장 Operator로서, just() Operator와 달리, emit할 데이터가 null일 경우 NullPointerException이 발생하지 않고 onComplete Signal을 전송합니다.

그리고 emit할 데이터가 null이 아닐 경우 해당 데이터를 emit하는 Mono를 생성합니다.

```java
Mono.justOrEmpty(null)
        .subscribe(
                data -> {
                },
                error -> {
                },
                () -> log.info("# onComplete")
        );
```

위의 코드를 실행하면 NullPointerException 없이 onComplete Signal만 전송하는 것을 볼 수 있습니다.

```java
# onComplete
```

### fromIterable

fromIterable() Operator는 Iterable에 포함된 데이터를 emit하는 Flux를 생성합니다.

즉, Java에서 제공하는 Iterable을 구현한 구현체를 fromIterable()의 파라미터로 전달할 수 있습니다.

```java
List<String> itemList = Arrays.asList("item1", "item2", "item3", "item4");

Flux<String> itemFlux = Flux.fromIterable(itemList);

itemFlux.subscribe(item -> log.info("Next item: {}", item));
```

```java
20:39:17.441 [main] INFO org.example.webfluxplayground.operator.FromIterable -- Next item: item1
20:39:17.443 [main] INFO org.example.webfluxplayground.operator.FromIterable -- Next item: item2
20:39:17.443 [main] INFO org.example.webfluxplayground.operator.FromIterable -- Next item: item3
20:39:17.443 [main] INFO org.example.webfluxplayground.operator.FromIterable -- Next item: item4
```

### fromStream

fromStream() Operator는 Stream에 포함된 데이터를 emit하는 Flux를 생성합니다.

Java의 Stream 특성상 Stream은 재사용할 수 없으며 cancel, error, complete 시에 자동으로 닫히게 됩니다.

```java
Flux.fromStream(() -> Stream.of("item1", "item2", "item3", "item4"))
        .subscribe(item -> log.info("Next item: {}", item));
```

```java
20:48:12.970 [main] INFO org.example.webfluxplayground.operator.FromStream -- Next item: item1
20:48:12.971 [main] INFO org.example.webfluxplayground.operator.FromStream -- Next item: item2
20:48:12.971 [main] INFO org.example.webfluxplayground.operator.FromStream -- Next item: item3
20:48:12.971 [main] INFO org.example.webfluxplayground.operator.FromStream -- Next item: item4
```

### range

range() Operator는 n부터 1씩 증가한 연속된 수를 m개 emit하는 Flux를 생성합니다.

range() Operator는 명령형 언어의 for문처럼 특정 횟수만큼 어떤 작업을 처리하고자 할 경우에 주로 사용됩니다.

```java
Flux.range(5, 10)
        .subscribe(data -> log.info("Next item: {}", data));
```

```java
20:51:34.722 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 5
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 6
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 7
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 8
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 9
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 10
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 11
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 12
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 13
20:51:34.724 [main] INFO org.example.webfluxplayground.operator.Range -- Next item: 14
```

### defer

defer() Operator는 Operator를 선언한 시점에 데이터를 emit하는 것이 아니라 구독하는 시점에 데이터를 emit하는 Flux 또는 Mono를 생성합니다.

defer()는 데이터 emit을 지연시키기 때문에 꼭 필요한 시점에 데이터를 emit하여 불필요한 프로세스를 줄일 수 있습니다.

```java
log.info("# start : {}", LocalDateTime.now());
Mono<LocalDateTime> justMono = Mono.just(LocalDateTime.now());
Mono<LocalDateTime> deferMono = Mono.defer(() -> Mono.just(LocalDateTime.now()));

Thread.sleep(2000);

justMono.subscribe(data -> log.info("# justMono1 : {}", data));
deferMono.subscribe(data -> log.info("# deferMono1 : {}", data));

Thread.sleep(2000);

justMono.subscribe(data -> log.info("# justMono2 : {}", data));
deferMono.subscribe(data -> log.info("# deferMono2 : {}", data));
```

```java
21:24:13.458 [main] INFO org.example.webfluxplayground.operator.Defer -- # start : 2024-07-24T21:24:13.458100
21:24:15.507 [main] INFO org.example.webfluxplayground.operator.Defer -- # justMono1 : 2024-07-24T21:24:13.459910
21:24:15.507 [main] INFO org.example.webfluxplayground.operator.Defer -- # deferMono1 : 2024-07-24T21:24:15.507749
21:24:17.513 [main] INFO org.example.webfluxplayground.operator.Defer -- # justMono2 : 2024-07-24T21:24:13.459910
21:24:17.514 [main] INFO org.example.webfluxplayground.operator.Defer -- # deferMono2 : 2024-07-24T21:24:17.514547
```

실행 결과를 보면 deferMono의 경우 emit된 데이터가 2초 간격으로 emit된 것을 볼 수 있습니다.

justMono() 같은 경우 지연 시간과는 상관없이 동일한 시간을 출력하는 것을 볼 수 있는데 이는 just() Operator가 Hot Publisher이기 때문에 Subscriber의 구독 여부와는 상관없이 데이터를 emit하게 됩니다.

### using

using() Operator는 파라미터로 전달받은 resource를 emit하는 Flux를 생성합니다.

첫 번째 파라미터는 읽어 올 resource이고, 두 번째 파라미터는 읽어 온 resource를 emit하는 Flux입니다.

마지막 세 번째 파라미터는 종료 Signal이 발생할 경우, resource를 해제하는 등의 후처리를 할 수 있게 해줍니다.

```bash
Path path = Paths.get("src/main/resources/using.txt");

Flux.using(() -> Files.lines(path), Flux::fromStream, Stream::close)
        .subscribe(log::info);
```

### generate

generate() Operator는 프로그래밍 방식으로 Signal 이벤트를 발생시키며, 특히 동기적으로 데이터를 하나씩 순차적으로 emit하고자 하는 경우 사용됩니다.

```java
Flux.generate(() -> 0, (state, sink) -> {
            sink.next(state);
            if (state == 10) {
                sink.complete();
            }
            return ++state;
        })
        .subscribe(data -> log.info("# onNext: {}", data));
```

generate()의 첫 번째 파라미터에서 초깃값을 0으로 지정했으며, 두 번째 파라미터에서 전달받은 SynchronousSink 객체로 상태값을 emit 합니다.

SyncronousSink는 하나의 Signal만 동기적으로 발생시킬 수 있으며, 최대 하나의 상태 값만 emit하는 인터페이스입니다.

```java
18:30:03.112 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 0
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 1
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 2
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 3
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 4
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 5
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 6
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 7
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 8
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 9
18:30:03.113 [main] INFO org.example.webfluxplayground.operator.Generate -- # onNext: 10
```

### create

create() Operator는 generate() Operator 처럼 프로그래밍 방식으로 Signal 이벤트를 발생시키지만 generate() Operator와 몇 가지 차이점이 있습니다.

generate() Operator는 데이터를 동기적으로 한 번에 한 건씩 emit할 수 있는 반면에, create() Operator는 한 번에 여러건의 데이터를 비동기적으로 emit할 수 있습니다.

```java
public class Create {
    private static int SIZE = 0;
    private static int COUNT = -1;

    private static final List<Integer> DATA_SOURCE = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    public static void main(String[] args) {
        log.info("# start");

        Flux.create((FluxSink<Integer> sink) -> {
            sink.onRequest(n -> {
                try {
                    Thread.sleep(1000L);
                    for (int i = 0; i < n; i++) {
                        if (COUNT >= 9) {
                            sink.complete();
                        } else {
                            COUNT++;
                            sink.next(DATA_SOURCE.get(COUNT));
                        }
                    }
                } catch (InterruptedException ignored) {
                }
            });
            sink.onDispose(() -> log.info("# clean up"));
        }).subscribe(new BaseSubscriber<Integer>() {
            @Override
            protected void hookOnSubscribe(Subscription subscription) {
                request(2);
            }

            @Override
            protected void hookOnNext(Integer value) {
                SIZE++;
                log.info("# onNext : {}", value);
                if (SIZE == 2) {
                    request(2);
                    SIZE = 0;
                }
            }

            @Override
            protected void hookOnComplete() {
                log.info("# onComplete");
            }
        });
    }
}
```

위의 예시코드는 Subscriber에서 요청한 경우에만 create() Operator 내에서 요청 개수만큼의 데이터를 emit하는 예제 코드입니다.

emit 대상인 데이터 소스는 1부터 10까지의 숫자를 원소로 포함하고 있는 dataSource List를 이용했습니다.

그리고 Subscriber가 직접 요청 개수를 지정하기 위해 BaseSubscriber를 사용했습니다.

위의 코드는 다음과 같은 순서로 작동합니다.

1. 구독이 발생하면 BaseSubscriber의 hookOnSubscribe() 메서드 내부에서 request(2)를 호출하여 한 번에 두 개의 데이터를 요청합니다.
2. Subscriber 쪽에서 request() 메서드를 호출하면 create() Operator 내부에서 sink.onRequest() 메서드의 람다 표현식이 실행됩니다.
3. Subscriber가 요청한 개수만큼 데이터를 emit 합니다.
4. BaseSubscriber의 hookOnNext() 메서드 내부에서 emit된 데이터를 로그로 출력한 후, 다시 reuqest(2)를 호출하여 두 개의 데이터를 요청합니다.
5. 2에서 4의 과정이 반복되다가 dataSource List의 숫자를 모두 emit하면 onComplete Signal을 발생시킵니다.
6. BaseSubscriber의 hookOnComplete() 메서드 내부에서 종료 로그를 출력합니다.
7. sink.onDispose()의 람다 표현식이 실행되어 후처리 로그를 출력합니다.

```java
22:38:11.668 [main] INFO org.example.webfluxplayground.operator.Create -- # start
22:38:12.721 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 1
22:38:12.722 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 2
22:38:13.728 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 3
22:38:13.728 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 4
22:38:14.733 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 5
22:38:14.733 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 6
22:38:15.737 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 7
22:38:15.738 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 8
22:38:16.740 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 9
22:38:16.740 [main] INFO org.example.webfluxplayground.operator.Create -- # onNext : 10
22:38:17.745 [main] INFO org.example.webfluxplayground.operator.Create -- # onComplete
22:38:17.747 [main] INFO org.example.webfluxplayground.operator.Create -- # clean up
```

### 정리

- just() Operator는 Hot Publisher이기 때문에 Subscriber의 구독 여부와는 상관없이 데이터를 emit하며, 구독이 발생하면 emit된 데이터를 다시 재생(replay)해서 Subscriber에게 전달합니다.
- defer() Operator는 구독이 발생하기 전까지 데이터의 emit을 지연시키기 때문에 just() Operator를 defer()를 감싸게 되면 실제 구독이 발생해야 데이터를 emit 합니다.
- using() Operator는 파라미터로 전달받은 resource를 emit하는 Flux를 생성합니다.
- generator() Operator는 프로그래밍 방식으로 Signal 이벤트를 발생시키며, 동기적으로 데이터를 하나씩 순차적으로 emit 합니다.
- create() Operator는 generate() Operator와 마찬가지로 프로그래밍 방식으로 Singal 이벤트를 발생시키지만 generate() Operator와 달리 한 번에 여러 건의 데이터를 비동기적으로 emit할 수 있습니다.

## Sequence 필터링 Operator

### filter

filter() Operator는 Upstream에서 emit된 데이터 중에서 일치하는 데이터만 Downstream으로 emit 합니다.

즉, filter() Operator의 파라미터로 입력 받은 Predicate의 리턴 값이 true인 데이터만 Downstream으로 emit 합니다.

```java
Flux.range(1, 20)
        .filter(num -> num % 2 == 0)
        .subscribe(data -> log.info("# onNext : {}", data));
```

```java
22:55:24.379 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 2
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 4
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 6
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 8
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 10
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 12
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 14
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 16
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 18
22:55:24.380 [main] INFO org.example.webfluxplayground.operator.Filter -- # onNext : 20
```

filterWhen() Operator를 사용하면 비동기적으로 필터링을 수행할 수도 있습니다.

### skip

skip() Operator는 Upstream에서 emit된 데이터 중에서 파라미터로 입력받은 숫자만큼 건너뛴 후, 나머지 데이터를 Downstream으로 emit 합니다.

```java
Flux.interval(Duration.ofSeconds(1))
        .skip(2)
        .subscribe(data -> log.info("# onNext : {}", data));

Thread.sleep(5500L);
```

```java
22:59:33.821 [parallel-1] INFO org.example.webfluxplayground.operator.Skip -- # onNext : 2
22:59:34.820 [parallel-1] INFO org.example.webfluxplayground.operator.Skip -- # onNext : 3
22:59:35.819 [parallel-1] INFO org.example.webfluxplayground.operator.Skip -- # onNext : 4
```

### take

take() Operator는 Upstream에서 emit되는 데이터 중에서 파라미터로 입력받은 숫자만큼만 Downstream으로 emit 합니다.

```java
Flux.interval(Duration.ofSeconds(1))
        .take(3)
        .subscribe(data -> log.info("# onNext : {}", data));

Thread.sleep(4000L);
```

```java
23:10:23.380 [parallel-1] INFO org.example.webfluxplayground.operator.Take -- # onNext : 0
23:10:24.374 [parallel-1] INFO org.example.webfluxplayground.operator.Take -- # onNext : 1
23:10:25.373 [parallel-1] INFO org.example.webfluxplayground.operator.Take -- # onNext : 2
```

- takeLast() Operator는 Upstream에서 emit된 데이터 중에서 파라미터로 입력한 개수만큼 가장 마지막에 emit된 데이터를 Downstream으로 emit합니다.
- takeUntil() Operator는 파라미터로 입력한 람다 표현식이 true가 될 때 까지 Upstream에서 emit된 데이터를 Downstream으로 emit 합니다.
    - Upstrem에서 emit된 데이터에는 Predicate를 평가할 때 사용한 데이터가 포함된다는 사실을 기억해야 합니다.
- takeWhile() Operator는 파라미터로 입력한 람다 표현식이 true가 되는 동안에만 Upstream에서 emit된 데이터를 Downstream으로 emit 합니다.
    - takeUntil() Operator와 달리 Predicate를 평가할 때 사용한 데이터가 Downstream으로 emit되지 않습니다.

### next

next() Operator는 Upstream에서 emit되는 데이터 중에서 첫 번째 데이터만 Downstream으로 emit합니다.

만일 Upstream에서 emit되는 데이터가 empty라면 Downstream으로 empty Mono를 emit 합니다.

```java
public static void main(String[] args) {
  Flux.fromIterable(() -> "Hello".chars().iterator())
          .next()
          .subscribe(data -> log.info("# onNext : {}", (char)((int)data)));
}
```

```java
23:36:13.564 [main] INFO org.example.webfluxplayground.operator.Next -- # onNext : H
```

## Sequence 변환 Operator

### map

map() Operator는 Upstream에서 emit된 데이터를 mapper Function을 사용하여 변환한 후 Downstream으로 emit 합니다.

map() Operator 내부에서 에러 발생시 Sequence가 종료되지 않고 계속 진행되도록 하는 기능을 지원합니다.

```java
public static void main(String[] args) {
    Flux.just("1-Circle", "3-Circle", "5-Circle")
            .map(circle -> circle.replace("Circle", "Rectangle"))
            .subscribe(data -> log.info(" onNext : {}", data));
}
```

```java
23:42:40.131 [main] INFO org.example.webfluxplayground.operator.Map --  onNext : 1-Rectangle
23:42:40.133 [main] INFO org.example.webfluxplayground.operator.Map --  onNext : 3-Rectangle
23:42:40.133 [main] INFO org.example.webfluxplayground.operator.Map --  onNext : 5-Rectangle
```

### flatMap

마블 다이어그램에서는 flatMap() Operator 영역 위쪽의 Upstream에서동그라미 모양의 데이터가 emit되어 flatMap() Operator로 전달되면 flatMap() 내부에서 InnerSequence를 생성한 후, 한 개 이상의 네모 모양으로 변환된 데이터를 emit 하고 있는 것을 볼 수 있습니다.

즉, Upstream에서 emit된 데이터 한 건이 Inner Sequence에서 여러 건의 데이터로 변환된다는 것을 알 수 있습니다.

그런데 Upstream에서 emit된 데이터는 이렇게 Inner Sequence에서 평탄화 작업을 거치면서 하나의 Sequence로 병합되어 Downstream으로 emit 됩니다.

```java
Flux.just("Good", "Bad")
        .flatMap(feeling -> Flux.just("Morning", "Afternoon", "Evening")
                .map(time -> feeling + " " + time))
        .subscribe(log::info);
```

```java
23:49:52.260 [main] INFO org.example.webfluxplayground.operator.FlatMap -- Good Morning
23:49:52.261 [main] INFO org.example.webfluxplayground.operator.FlatMap -- Good Afternoon
23:49:52.261 [main] INFO org.example.webfluxplayground.operator.FlatMap -- Good Evening
23:49:52.262 [main] INFO org.example.webfluxplayground.operator.FlatMap -- Bad Morning
23:49:52.262 [main] INFO org.example.webfluxplayground.operator.FlatMap -- Bad Afternoon
23:49:52.262 [main] INFO org.example.webfluxplayground.operator.FlatMap -- Bad Evening
```

### concat

concat() Operator는 파라미터로 입력되는 Publisher의 Sequence를 연결해서 데이터를 순차적으로 emit 합니다.

특히 먼저 입력된 Publisher의 Sequence가 종료될 때까지 나머지 Publisher의 Sequence는 subscribe 되지 않고 대기하는 특성을 가집니다.

```java
Flux.concat(Flux.just(1, 2, 3), Flux.just(4, 5))
        .subscribe(data -> log.info("# onNext : {}", data));
```

```java
21:53:35.832 [main] INFO org.example.webfluxplayground.operator.Concat -- # onNext : 1
21:53:35.833 [main] INFO org.example.webfluxplayground.operator.Concat -- # onNext : 2
21:53:35.833 [main] INFO org.example.webfluxplayground.operator.Concat -- # onNext : 3
21:53:35.833 [main] INFO org.example.webfluxplayground.operator.Concat -- # onNext : 4
21:53:35.833 [main] INFO org.example.webfluxplayground.operator.Concat -- # onNext : 5
```

### merge

merge() Operator는 파라미터로 입력되는 Publisher의 Sequence에서 emit된 데이터를 인터리빙 방식으로 병합합니다.

merge() Operator는 concat() Operator 처럼 먼저 입력된 Publisher의 Sequence가 종료될 때까지 나머지 Publisher의 Sequence가 subscribe 되지 않고 대기하는 것이 아니라 모든 Publisher의 Sequence가 즉시 subscribe 됩니다.

<aside>
💡 컴퓨터 하드디스크상의 데이터를 서로 인접하지 않게 배열해 성능을 향상시키거나 인접한 메모리 위치를 서로 다른 메모리 뱅크에 두어 동시에 여러 곳을 접근할 수 있게 해주는 것을 인터리빙이라고 합니다.

주의해야 하는 점은 인터리빙 방식이라고 해서 각각의 Publisher가 emit하는 데이터를 하나씩 번갈아 가며 merge 하는 것이 아니라 emit된 시간 순서대로 merge 한다는 것 입니다.

</aside>

```java
Flux.merge(
                Flux.just(1, 2, 3, 4).delayElements(Duration.ofMillis(300L)),
                Flux.just(5, 6, 7).delayElements(Duration.ofMillis(500L))
        )
        .subscribe(data -> log.info("# onNext : {}", data));
Thread.sleep(2000L);
```

```java
22:12:57.535 [parallel-1] INFO org.example.webfluxplayground.operator.Merge -- # onNext : 1
22:12:57.729 [parallel-2] INFO org.example.webfluxplayground.operator.Merge -- # onNext : 5
22:12:57.842 [parallel-3] INFO org.example.webfluxplayground.operator.Merge -- # onNext : 2
22:12:58.147 [parallel-5] INFO org.example.webfluxplayground.operator.Merge -- # onNext : 3
22:12:58.235 [parallel-4] INFO org.example.webfluxplayground.operator.Merge -- # onNext : 6
22:12:58.452 [parallel-6] INFO org.example.webfluxplayground.operator.Merge -- # onNext : 4
22:12:58.740 [parallel-7] INFO org.example.webfluxplayground.operator.Merge -- # onNext : 7
```

결과를 보면 각각의 Publisher가 emit하는 시간이 빠른 데이터부터 차례대로 emit하고 있는 것을 볼 수 있습니다.

### zip

zip() Operator는 파라미터로 입력되는 Publisher Sequence에서 emit된 데이터를 결합하는데, 각 Publisher가 데이터를 하나씩 emit할 때까지 기다렸다가 결합합니다.

```java
Flux.zip(
                Flux.just(1, 2, 3).delayElements(Duration.ofMillis(300L)),
                Flux.just(4, 5, 6).delayElements(Duration.ofMillis(500L))
        )
        .subscribe(data -> log.info("# onNext : {}", data));
Thread.sleep(2500L);
```

```java
22:18:26.558 [parallel-2] INFO org.example.webfluxplayground.operator.Zip -- # onNext : [1,4]
22:18:27.065 [parallel-4] INFO org.example.webfluxplayground.operator.Zip -- # onNext : [2,5]
22:18:27.571 [parallel-6] INFO org.example.webfluxplayground.operator.Zip -- # onNext : [3,6]
```

두개의 Flux가 emit하는 시간이 다르지만 각 Flux에서 하나씩 emit할 때까지 기다렸다가 emit된 데이터를 Tuple2 객체로 묶어서 subscriber에게 전달하는 것을 볼 수 있습니다.

### and

and() Operator는 Mono의 Complete Signal과 파라미터로 입력된 Publisher의 Complete Signal과 결합하여 새로운 Mono<Void>를 반환합니다.

즉, Mono와 파라미터로 입력된 Publisher의 Sequence가 모두 종료되었음을 Subscriber에게 알릴 수 있습니다.

```java
Mono.just("Task 1")
        .delayElement(Duration.ofSeconds(1))
        .doOnNext(data -> log.info("# Mono doOnNext : {}", data))
        .and(Flux.just("Task 2", "Task 3")
                .delayElements(Duration.ofMillis(600))
                .doOnNext(data -> log.info("# Flux doOnNext : {}", data))
        )
        .subscribe(data -> log.info("# onNext : {}", data),
                error -> log.error("# onError : ", error),
                () -> log.info("# onComplete")
        );
Thread.sleep(5000);
```

위의 예제에서 and() Operator를 기준으로 Upstream에서 1초의 지연 시간을 가진 뒤 Task 1을 emit하고 and() Operator 내부의 Inner Sequence에서는 0.6초에 한 번씩 Task 2, Task 3을 emit 합니다.

결과적으로 Subscriber에게 onComplete Signal만 전달되고, Upstream에서 emit된 데이터는 전달되지 않습니다.

즉, and() Operator는 모든 Sequence가 종료되길 기다렸다가 최종적으로 onComplete Signal만 전송합니다.

```java
22:27:13.478 [parallel-2] INFO org.example.webfluxplayground.operator.And -- # Flux doOnNext : Task 2
22:27:13.873 [parallel-1] INFO org.example.webfluxplayground.operator.And -- # Mono doOnNext : Task 1
22:27:14.088 [parallel-3] INFO org.example.webfluxplayground.operator.And -- # Flux doOnNext : Task 3
22:27:14.088 [parallel-3] INFO org.example.webfluxplayground.operator.And -- # onComplete
```

### collectList

collectList() Operator는 Flux에서 emit된 데이터를 모아서 List로 변환한 후, 변환된 List를 emit하는 Mono를 반환합니다.

만약 Upstream Sequence가 비어있다면 비어 있는 List를 Downstream으로 emit 합니다.

```java
    public static void main(String[] args) {
        Flux.just("...", "---", "...")
                .map(CollectList::transformMorseCode)
                .collectList()
                .subscribe(list -> log.info(String.join("", list)));

    }

    public static String transformMorseCode(String code) {
        return code + "!";
    }
```

```java
22:33:19.045 [main] INFO org.example.webfluxplayground.operator.CollectList -- ...!---!...!
```

### collectMap

collectMap() Operator는 Flux에서 emit된 데이터를 기반으로 key와 value를 생성하여 Map의 Element로 추가한 후, 최종적으로 Map을 emit하는 Mono를 반환합니다.

만약 Upstream Sequence가 비어있다면 비어 있는 Map을 Downstream으로 emit 합니다.

```java
Flux.range(0, 26)
        .collectMap(key -> (char) (key + 65),
                value -> String.valueOf((char) (value + 65)))
        .subscribe(map -> log.info("onNext : {}", map));
```

```java
22:37:01.603 [main] INFO org.example.webfluxplayground.operator.collectMap -- onNext : {A=A, B=B, C=C, D=D, E=E, F=F, G=G, H=H, I=I, J=J, K=K, L=L, M=M, N=N, O=O, P=P, Q=Q, R=R, S=S, T=T, U=U, V=V, W=W, X=X, Y=Y, Z=Z}
```

## Sequence의 내부 동작을 확인하기 위한 Operator

Reactor에서는 Upstream Publisher에서 emit되는 데이터 변경 없이 부수 효과만을 수행하기 위한 Operator들이 있는데, doOnXXXX() 으로 시작하는 Operator가 바로 그러한 역할을 하는 Operator 입니다.

doOnXXXX()으로 시작하는 Operator는 Consumer 또는 Runnable 타입의 함수형 인터페이스를 파라미터로 가지기 때문에 별도의 리턴 값이 없습니다.

따라서 Upstream Publisher로부터 emit되는 데이터를 이용해 Upstream Publisher의 내부 동작을 엿볼 수 있으며 로그를 출력하는 등의 디버깅 용도로 많이 사용됩니다.

또한 데이터 emit 과정에서 error가 발생하면 해당 에러에 대한 알림을 전송하는 로직을 추가하는 등 부수 효과를 위한 다양한 로직을 적용할 수 있습니다.



| Operator | 설명 |
| --- | --- |
| doOnSubscribe() | Publisher가 구독 중일 때 트리거되는 동작을 추가할 수 있습니다. |
| doOnRequest() | Publisher가 요청을 수신할 때 트리거되는 동작을 추가할 수 있습니다. |
| doOnNext() | Publisher가 데이터를 emit할 때 트리거되는 동작을 추가할 수 있습니다. |
| doOnComplete() | Publisher가 성공적으로 완료되었을 때 트리거되는 동작을 추가할 수 있습니다. |
| doOnError() | Publisher가 에러가 발생한 상태로 종료되었을 때 트리거되는 동작을 추가할 수 있습니다. |
| doOnCancel() | Publisher가 취소되었을 때 트리거되는 동작을 추가할 수 있습니다. |
| doOnTerminate() | Publisher가 성공적으로 완료되었을 때 또는 에러가 발생한 상태로 종료되었을 때 트리거되는 동작을 추가할 수 있습니다. |
| doOnEach() | Publisher가 데이터를 emit할 때, 성공적으로 완료되었을 때, 에러가 발생한 상태로 종료되었을 때 트리거되는 동작을 추가할 수 있습니다. |
| doOnDiscard() | Upstream에 있는 전체 Operator 체인의 동작 중에서 Operator에 의해 폐기되는 요소를 조건부로 정리할 수 있습니다. |
| doAfterTerminate() | Downstream을 성공적으로 완료한 직후 또는 에러가 발생하여 Publisher가 종료된 직후에 트리거되는 동작을 추가할 수 있습니다. |
| doFirst() | Publisher가 구독되기 전에 트리거 되는 동작을 추가할 수 있습니다. |
| doFinally() | 에러를 포함해서 어떤 이유이든 간에 Publisher가 종료된 후 트리거되는 동작을 추가할 수 있습니다. |

## 에러 처리를 위한 Operator

### error

error() Operator는 파라미터로 지정된 에러로 종료하는 Flux를 생성합니다.

error() Operator는 마치 Java의 throw 키워드를 사용해서 예외를 의도적으로 던지는 것 같은 역할을 하는데 주로 체크 예외를 캐치해서 다시 던져야 하는 경우 사용할 수 있습니다.

```java
Flux.range(1, 5)
        .flatMap(num -> {
            if ((num * 2) % 3 == 0) {
                return Flux.error(new IllegalArgumentException("Not allowed multiple of 3"));
            } else {
                return Mono.just(num * 2);
            }
        })
        .subscribe(data -> log.info("# onNext : {}", data),
                error -> log.error("# onError : ", error));
```

```java
22:58:25.103 [main] INFO org.example.webfluxplayground.operator.Error -- # onNext : 2
22:58:25.104 [main] INFO org.example.webfluxplayground.operator.Error -- # onNext : 4
22:58:25.106 [main] ERROR org.example.webfluxplayground.operator.Error -- # onError : 
java.lang.IllegalArgumentException: Not allowed multiple of 3
	at org.example.webfluxplayground.operator.Error.lambda$main$0(Error.java:13)
	at reactor.core.publisher.FluxFlatMap$FlatMapMain.onNext(FluxFlatMap.java:388)
	at reactor.core.publisher.FluxRange$RangeSubscription.slowPath(FluxRange.java:156)
	at reactor.core.publisher.FluxRange$RangeSubscription.request(FluxRange.java:111)
	at reactor.core.publisher.FluxFlatMap$FlatMapMain.onSubscribe(FluxFlatMap.java:373)
	at reactor.core.publisher.FluxRange.subscribe(FluxRange.java:69)
	at reactor.core.publisher.Flux.subscribe(Flux.java:8840)
	at reactor.core.publisher.Flux.subscribeWith(Flux.java:8961)
	at reactor.core.publisher.Flux.subscribe(Flux.java:8805)
	at reactor.core.publisher.Flux.subscribe(Flux.java:8729)
	at reactor.core.publisher.Flux.subscribe(Flux.java:8699)
	at org.example.webfluxplayground.operator.Error.main(Error.java:18)
```

### onErrorReturn

onErrorReturn() Operator는 에러 이벤트가 발생했을 때 에러 이벤트를 Downstream으로 전파하지 않고 대체 값을 emit 합니다.

Java에서 예외가 발생했을 때 try - catch 문의 catch 블록에서 예외에 해당하는 대체 값을 리턴하는 방식과 유사하다고 생각하면 됩니다.

```java
Flux.just(1, 2, 3, 4, 5)
      .<Integer>handle((num, sink) -> {
          if (num == 3) {
              sink.error(new IllegalArgumentException("Not allowed 3"));
              return;
          }
          sink.next(num * 2);
      })
      .onErrorReturn(-1)
      .subscribe(data -> log.info("# onNext : {}", data),
              error -> log.error("# onError : ", error));
```

```java
23:02:14.165 [main] INFO org.example.webfluxplayground.operator.OnErrorReturn -- # onNext : 2
23:02:14.167 [main] INFO org.example.webfluxplayground.operator.OnErrorReturn -- # onNext : 4
23:02:14.168 [main] INFO org.example.webfluxplayground.operator.OnErrorReturn -- # onNext : -1
```

### onErrorResume

onErrorResume() Operator는 에러 이벤트가 발생했을 때, 에러 이벤트를 Downstream으로 전파하지 않고 대체 Publisher를 리턴합니다.

onErrorResume() Operator 마블 다이어그램을 보면 제일 위쪽에 있는 Publisher에서 에러가 발생할 경우, onErrorResume() Operator 영역 안에서 새로운 Publisher로 대체되는 것을 볼 수 있습니다.

onErrorResume() Operator는 마치 Java에서 예외가 발생했을 때 try - catch문의 catch 블록에서 예외가 발생한 메서드를 대체할 수 있는 또 다른 메서드를 호출하는 방식이라고 볼 수 있습니다.

```java
Flux<Integer> source = Flux.just(1, 2, 3, 4)
        .concatWith(Flux.error(new RuntimeException("An error occurred!")))
        .concatWith(Flux.just(5));

Flux<Integer> fallback = Flux.just(10, 20, 30);

source
        .onErrorResume(e -> {
            System.out.println("Caught error: " + e.getMessage());
            return fallback;
        })
        .subscribe(
                data -> System.out.println("Next: " + data),
                error -> System.err.println("Error: " + error),
                () -> System.out.println("Completed")
        );
```

```java
Next: 1
Next: 2
Next: 3
Next: 4
Caught error: An error occurred!
Next: 10
Next: 20
Next: 30
Completed
```

에러 발생 후 기존의 Publisher가 새로운 Publisher로 대체된것을 볼 수 있습니다.

### onErrorContinue

onErrorContinue Operator()는 에러가 발생했을 때 에러 영역 내에 있는 데이터는 제거하고, Upstream에서 후속 데이터를 emit하는 방식으로 에러를 복구할 수 있도록 해줍니다.

onErrorContinue() Operator의 파라미터인 BiConsumer 함수형 인터페이스를 통해 에러 메시지와 에러가 발생했을 때 emit된 데이터를 전달받아서 로그를 기록하는 등의 후처리를 할 수 있습니다.

```java
Flux.just(1, 2, 4, 0, 6, 12)
      .map(num -> 12 / num)
      .onErrorContinue((error, num) -> log.error("Error : {}, num : {}", error.getMessage(), num))
      .subscribe(
              data -> log.info("# onNext : {}", data),
              error -> log.error("# onError : ", error)
      );
```

```java
20:30:34.607 [main] INFO org.example.webfluxplayground.operator.OnErrorContinue -- # onNext : 12
20:30:34.609 [main] INFO org.example.webfluxplayground.operator.OnErrorContinue -- # onNext : 6
20:30:34.609 [main] INFO org.example.webfluxplayground.operator.OnErrorContinue -- # onNext : 3
20:30:34.610 [main] ERROR org.example.webfluxplayground.operator.OnErrorContinue -- Error : / by zero, num : 0
20:30:34.610 [main] INFO org.example.webfluxplayground.operator.OnErrorContinue -- # onNext : 2
20:30:34.610 [main] INFO org.example.webfluxplayground.operator.OnErrorContinue -- # onNext : 1
```

### retry

retry() Operator는 Publisher가 데이터를 emit하는 과정에서 에러가 발생하면 파라미터로 입력한 횟수만큼 원본 Flux의 Sequence를 다시 구독합니다.

만약, 파라미터로 Long.MAX_VALUE를 입력하면 재구독을 무한 반복합니다.

```java
final int[] count = {1};

Flux.range(1, 3)
        .delayElements(Duration.ofSeconds(1))
        .map(num -> {
            try {
                if (num == 3 && count[0] == 1) {
                    count[0]++;
                    Thread.sleep(1000);
                }
            } catch (InterruptedException ignored) {
            }
            return num;
        })
        .timeout(Duration.ofMillis(1500))
        .retry()
        .subscribe(
                data -> log.info("# onNext : {}", data),
                error -> log.error("# onError : ", error),
                () -> log.info("# onComplete")
        );
Thread.sleep(7000);
```

```java
11:42:54 [parallel-2] INFO - # onNext: 1 
11:42:55 [parallel-4] INFO - # onNext: 2 
11:42:57 [parallel-6] DEBUG- onNextDropped: 3 
11:42:57 [paraUel-8] INFO - # onNext: 1 
11:42:58 [parallel-2] INFO - # onNext: 2 
11:42:59 [parallel-4] INFO - # onNext: 3 
11:42:59 [parallel-4] INFO - # onComplete
```

## Sequnece의 동작 시간 측정을 위한 Operator

### elapsed

elapsed() Operator는 emit된 데이터 사이의 경과 시간을 측정해서 Tuple<Long, T> 형태로 Downstream에 emit 합니다.

emit되는 첫 번째 데이터는 onSubscribe Signal과 첫 번째 데이터 사이를 기준으로 시간을 측정합니다.

측정된 시간의 단위는 milliseconds 입니다.

```java
public static void main(String[] args) throws InterruptedException {
    Flux.range(1, 5)
            .delayElements(Duration.ofSeconds(1))
            .elapsed()
            .subscribe(data -> log.info("# onNext : {}, time: {}", data.getT1(), data.getT2()));

    Thread.sleep(6000);
}
```

```java
20:40:27.962 [parallel-1] INFO org.example.webfluxplayground.operator.Elapsed -- # onNext : 1004, time: 1
20:40:28.965 [parallel-2] INFO org.example.webfluxplayground.operator.Elapsed -- # onNext : 1004, time: 2
20:40:29.968 [parallel-3] INFO org.example.webfluxplayground.operator.Elapsed -- # onNext : 1003, time: 3
20:40:30.972 [parallel-4] INFO org.example.webfluxplayground.operator.Elapsed -- # onNext : 1004, time: 4
20:40:31.978 [parallel-5] INFO org.example.webfluxplayground.operator.Elapsed -- # onNext : 1006, time: 5
```

## Flux Sequence 분할을 위한 Operator

### window

window(int maxSize) Operator는 Upstream에서 emit되는 첫 번째 데이터 부터 maxSize 숫자만큼 데이터를 포함하는 새로운 Flux로 분할합니다.

이렇게 분할된 Flux를 윈도우라고 부릅니다.

```java
public static void main(String[] args) {
  Flux.range(1, 11)
          .window(3)
          .flatMap(flux -> {
              log.info("====================");
              return flux;
          })
          .subscribe(new BaseSubscriber<>() {
              @Override
              protected void hookOnSubscribe(Subscription subscription) {
                  subscription.request(2);
              }

              @Override
              protected void hookOnNext(Integer value) {
                  log.info("# onNext : {}", value);
                  request(2);
              }
          });
}
```

```java
20:47:42.191 [main] INFO org.example.webfluxplayground.operator.Window -- ====================
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 1
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 2
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 3
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- ====================
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 4
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 5
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 6
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- ====================
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 7
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 8
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 9
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- ====================
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 10
20:47:42.193 [main] INFO org.example.webfluxplayground.operator.Window -- # onNext : 11
```

실행 결과를 보면 window() Operator의 파라미터로 입력한 maxSize의 데이터를 포함한 새로운 Flux로 분할되고, 분할된 Flux가 각각의 데이터를 Downstream으로 emit하고 있는 것을 확인할 수 있습니다.

### buffer

buffer(int maxSize) Operator는 Upstream에서 emit되는 첫 번째 데이터부터 maxSize의 숫자만큼의 데이터를 List 버퍼로 한 번에 emit 합니다.

마지막 버퍼에 포함된 데이터의 개수는 maxSize보다 더 적거나 같습니다.

```java
public static void main(String[] args) {
    Flux.range(1, 95)
            .buffer(10)
            .subscribe(buffer -> log.info("# onNext : {}", buffer));
}
```

```java
20:50:38.838 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [31, 32, 33, 34, 35, 36, 37, 38, 39, 40]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [51, 52, 53, 54, 55, 56, 57, 58, 59, 60]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [61, 62, 63, 64, 65, 66, 67, 68, 69, 70]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [71, 72, 73, 74, 75, 76, 77, 78, 79, 80]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [81, 82, 83, 84, 85, 86, 87, 88, 89, 90]
20:50:38.841 [main] INFO org.example.webfluxplayground.operator.Buffer -- # onNext : [91, 92, 93, 94, 95]
```

높은 처리량을 요구하는 애플리케이션이 있다면, 들어오는 데이터를 순차적으로 처리하기보다는 batch insert 같은 일괄 작업에 buffer() Operator를 이용해서 성능 향상을 기대할 수 있습니다.

### bufferTimeout

bufferTimeout(maxSize, maxTime) Operator는 Upstream에서 emit되는 첫번째 데이터부터 maxSize 숫자만큼의 데이터 또는 maxTime 내에 emit된 데이터를 List 버퍼로 한 번에 emit 합니다.

```java
public static void main(String[] args) {
    Flux.range(1, 20)
            .map(num -> {
                try {
                    if (num < 10) {
                        Thread.sleep(100);
                    } else {
                        Thread.sleep(300);
                    }
                } catch (InterruptedException ignored) {
                }
                return num;
            })
            .bufferTimeout(3, Duration.ofMillis(400))
            .subscribe(buffer -> log.info("# onNext : {}", buffer));
}
```

```java
20:54:41.439 [main] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [1, 2, 3]
20:54:41.751 [main] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [4, 5, 6]
20:54:42.061 [main] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [7, 8, 9]
20:54:42.770 [parallel-1] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [10, 11]
20:54:43.373 [parallel-1] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [12, 13]
20:54:43.981 [parallel-1] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [14, 15]
20:54:44.592 [parallel-1] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [16, 17]
20:54:45.199 [parallel-1] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [18, 19]
20:54:45.405 [main] INFO org.example.webfluxplayground.operator.BufferTimeout -- # onNext : [20]
```

maxSize가 3인 상태에서 10 이하의 숫자는 0.1초의 지연시간을 가지기 때문에 세 개씩 전달되고 10 이상의 숫자는 0.3초의 지연시간을 가지기 때문에 2개씩 전달되는 것을 볼 수 있습니다.

### group by

groupBy(keyMapper) Operator는 emit되는 데이터를 keyMapper로 생성한 key를 기준으로 그룹화한 GroupedFlux를 리턴하며, 이 GroupedFlux를 통해서 그룹별로 작업을 수행할 수 있습니다.

```java
public static void main(String[] args) {
    Flux<String> flux = Flux.just("apple", "banana", "orange", "apple", "banana", "grape");

    Flux<GroupedFlux<String, String>> groupedFlux = flux.groupBy(fruit -> fruit);

    groupedFlux.concatMap(group -> group.collectList()
                    .map(list -> group.key() + ": " + list))
            .subscribe(log::info);
}
```

같은 문자열을 group by로 묶은 것을 확인할 수 있습니다.

```java
20:59:29.869 [main] INFO org.example.webfluxplayground.operator.GroupBy -- apple: [apple, apple]
20:59:29.870 [main] INFO org.example.webfluxplayground.operator.GroupBy -- banana: [banana, banana]
20:59:29.870 [main] INFO org.example.webfluxplayground.operator.GroupBy -- orange: [orange]
20:59:29.870 [main] INFO org.example.webfluxplayground.operator.GroupBy -- grape: [grape]
```

## 다수의 subscriber에게 Flux를 멀티캐스팅(Multicasting) 하기 위한 Operator

Reactor에서는 다수의 Subscriber에게 Flux를 멀티캐스팅하기 위한 Operator를 지원합니다.

즉, Subscriber가 구독을 하면 Upstream에서 emit된 데이터가 구독중인 모든 Subscriber에게 멀티캐스팅되는데, 이를 가능하게 해주는 Operator들은 Cold Sequence를 Hot Sequence로 동작하게 하는 특징이 있습니다.

### publish

publish() Operator는 구독을 하더라도 구독 시점에 즉시 데이터를 emit하지 않고 connect()를 호출하는 시점에 비로소 데이터를 emit 합니다.

그리고 Hot Sequence로 변환되기 때문에 구독 시점 이후에 emit된 데이터만 전달받을 수 있습니다.

```java
public static void main(String[] args) throws InterruptedException {
    ConnectableFlux<Integer> flux = Flux.range(1, 5)
            .delayElements(Duration.ofMillis(300L))
            .publish();

    Thread.sleep(500L);
    flux.subscribe(data -> log.info("# subscriber1: {}", data));

    Thread.sleep(200L);
    flux.subscribe(data -> log.info("# subscriber2: {}", data));

    flux.connect();

    Thread.sleep(1000L);
    flux.subscribe(data -> log.info("# subscriber3: {}", data));

    Thread.sleep(2000L);
}
```

```java
22:42:25.075 [parallel-1] INFO org.example.webfluxplayground.operator.Publish -- # subscriber1: 1
22:42:25.080 [parallel-1] INFO org.example.webfluxplayground.operator.Publish -- # subscriber2: 1
22:42:25.386 [parallel-2] INFO org.example.webfluxplayground.operator.Publish -- # subscriber1: 2
22:42:25.386 [parallel-2] INFO org.example.webfluxplayground.operator.Publish -- # subscriber2: 2
22:42:25.690 [parallel-3] INFO org.example.webfluxplayground.operator.Publish -- # subscriber1: 3
22:42:25.690 [parallel-3] INFO org.example.webfluxplayground.operator.Publish -- # subscriber2: 3
22:42:25.992 [parallel-4] INFO org.example.webfluxplayground.operator.Publish -- # subscriber1: 4
22:42:25.993 [parallel-4] INFO org.example.webfluxplayground.operator.Publish -- # subscriber2: 4
22:42:25.993 [parallel-4] INFO org.example.webfluxplayground.operator.Publish -- # subscriber3: 4
22:42:26.294 [parallel-5] INFO org.example.webfluxplayground.operator.Publish -- # subscriber1: 5
22:42:26.294 [parallel-5] INFO org.example.webfluxplayground.operator.Publish -- # subscriber2: 5
22:42:26.294 [parallel-5] INFO org.example.webfluxplayground.operator.Publish -- # subscriber3: 5
```

### autoConnect

publish() Operator의 경우, 구독이 발생하더라도 connect()를 직접 호출하기 전까지는 데이터를 emit하지 않기 때문에 connect()를 직접 호출해야 합니다.

autoConnect() Operator는 파라미터로 지정하는 숫자만큼의 구독이 발생하는 시점에 Upstream 소스에 자동으로 연결되기 때문에 별도의 connect() 호출이 필요하지 않습니다.

```java
    public static void main(String[] args) throws InterruptedException {
        Flux<String> publisher = Flux.just("Concert part1", "Concert part2", "Concert part3")
                .delayElements(Duration.ofMillis(300L))
                .publish()
                .autoConnect(2);

        Thread.sleep(500L);
        publisher.subscribe(data -> log.info("# audience 1 is watching : {}", data));

        Thread.sleep(500L);
        publisher.subscribe(data -> log.info("# audience 2 is watching : {}", data));

        Thread.sleep(500L);
        publisher.subscribe(data -> log.info("# audience 3 is watching : {}", data));

        Thread.sleep(1000L);
    }
```

```java
22:52:36.409 [parallel-1] INFO org.example.webfluxplayground.operator.AutoConnect -- # audience 1 is watching : Concert part1
22:52:36.411 [parallel-1] INFO org.example.webfluxplayground.operator.AutoConnect -- # audience 2 is watching : Concert part1
22:52:36.716 [parallel-2] INFO org.example.webfluxplayground.operator.AutoConnect -- # audience 1 is watching : Concert part2
22:52:36.717 [parallel-2] INFO org.example.webfluxplayground.operator.AutoConnect -- # audience 2 is watching : Concert part2
22:52:36.717 [parallel-2] INFO org.example.webfluxplayground.operator.AutoConnect -- # audience 3 is watching : Concert part2
22:52:37.020 [parallel-3] INFO org.example.webfluxplayground.operator.AutoConnect -- # audience 1 is watching : Concert part3
22:52:37.021 [parallel-3] INFO org.example.webfluxplayground.operator.AutoConnect -- # audience 2 is watching : Concert part3
22:52:37.021 [parallel-3] INFO org.example.webfluxplayground.operator.AutoConnect -- # audience 3 is watching : Concert part3
```

### refCount

refCount() Operator는 파라미터로 입력된 숫자만큼의 구독이 발생하는 시점에 Upstream 소스에 연결되며, 모든 구독이 취소되거나 Upstream의 데이터 emit이 종료되면 연결이 해제됩니다.

refCount() Operator는 주로 무한 스트림 상황에서 모든 구독이 취소될 경우 연결을 해제하는 데 사용할 수 있습니다.

```java
public static void main(String[] args) throws InterruptedException {
    Flux<Long> publisher = Flux.interval(Duration.ofMillis(500))
            .publish().refCount(1);
    Disposable disposable = publisher.subscribe(data -> log.info("# subscriber 1: {}", data));

    Thread.sleep(2100L);
    disposable.dispose();

    publisher.subscribe(data -> log.info("# subscriber 2 : {}", data));

    Thread.sleep(2500L);
}
```

```java
23:00:37.280 [parallel-1] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 1: 0
23:00:37.772 [parallel-1] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 1: 1
23:00:38.274 [parallel-1] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 1: 2
23:00:38.772 [parallel-1] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 1: 3
23:00:39.382 [parallel-2] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 2 : 0
23:00:39.882 [parallel-2] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 2 : 1
23:00:40.382 [parallel-2] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 2 : 2
23:00:40.885 [parallel-2] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 2 : 3
23:00:41.381 [parallel-2] INFO org.example.webfluxplayground.operator.RefCount -- # subscriber 2 : 4
```

위의 예제에서는 refCount() Operator를 이용해 1개의 구독이 발생하는 시점에 Upstream 소스에 연결되도록 했습니다.

그런데 첫 번째 구독이 발생한 후 2.1초 후에 구독을 해제했습니다.

이 시점에는 모든 구독이 취소된 상태이기 때문에 연결이 해제되고, 두 번째 구독이 발생할 경우에는 Upstream 소스에 다시 연결됩니다.