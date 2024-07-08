# Chapter 10 Scheduler

Reactor Scheduler는 Reactor에서 사용되는 스레드를 관리해주는 관리자 역할을 한다.

* 용어 정리
* 듀얼코어 4스레드 -> 4 물리 스레드, 4 논리 코어 (하드웨어 스레드)

* 논리적인 스레드 -> 하드웨어 스레드가 아닌 애플리케이션 스레드. 

무수히 많은 논리적인 스레드가 물리 스레드에게 선택당하여 아주 빠르게 처리되고 있는것이다.



## Scheduler 전용 Operator

* subcribeOn
* publishOn
* parallell

### subscribeOn

subscribeOn() Operator는 그 이름처럼 구독이 발생한 직후 실행될 스레드를 지정하는 Operator

`subscribeOn` 연산자는 소스 시퀀스의 구독이 어느 Scheduler에서 실행될지를 결정한다. 

주로 I/O 바운드 작업이나 CPU 바운드 작업을 구분하여 적절한 Scheduler에서 구독을 처리하도록 사용한다.

```java
/**
 * subscribeOn() 기본 예제
 *  - 구독 시점에 Publisher의 실행을 위한 쓰레드를 지정한다
 */
@Slf4j
public class Example10_1 {
    public static void main(String[] args) throws InterruptedException {
        Flux.fromArray(new Integer[] {1, 3, 5, 7})
                .subscribeOn(Schedulers.boundedElastic()) // publisher의 동작을 처리하기 위한 스레드 할당 
                .doOnNext(data -> log.info("# doOnNext: {}", data)) // 처리 
                .doOnSubscribe(subscription -> log.info("# doOnSubscribe")) // 추가 처리 
                .subscribe(data -> log.info("# onNext: {}", data)); // 구독

        Thread.sleep(500L);
    }
}
```

* doOnsubscribe는 코드를 실행한 스레드에서 동작한다.
* subscribeon()을 추가하지 않았다면 원본 Plux의 처리 동작은 여전히 main 스 레드에서 실행되겠지만, subscribeOn()에서 Scheduler를 지정했기 때문에 구 독이 발생한 직후부터는 원본 Plux의 동작을 처리하는 스레드가 바뀌게 된다. 

### publishOn

`publishOn` 연산자는 데이터 처리가 어느 Scheduler에서 이루어질지를 결정한다.

특정 작업이 다른 스레드에서 처리되도록 할 때 유용하다. 

publishOn()이라는 Operator는 `Downstream`으로 Signal을 전송할 때 실행되는 스레드를 제어하는 역할을 하는 Operator라고 할 수 있다. 

* publishOn() Operator는 `코드상에서 publishOn()을 기준으로 아래쪽인 Downstream의 실행 스레드를 변경한다. `
* 그리고 subscribeOn()과 마찬가지로 파라미터로 Scheduler를 지정함으로써 해당 Scheduler의 특성을 가진 스레드로 변경할 수 있다.

```java
@Slf4j
public class Example10_2 {
    public static void main(String[] args) throws InterruptedException {
        Flux.fromArray(new Integer[] {1, 3, 5, 7})
                .doOnNext(data -> log.info("# doOnNext: {}", data))
                .doOnSubscribe(subscription -> log.info("# doOnSubscribe"))
                .publishOn(Schedulers.parallel()) // 페러렐로 지정. onNext() 실행 쓰레드가 페러렐로 변경 
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}
// 로그를 보면 publishOn다음부터 스레드가 바뀜 
```

* doOnNext의 경우 subscribeOn() 오퍼레이터를 사용하지 않았기때문에 요청이 실행된 스레드에서 실행 



### Parallel

parallel() Operator는 동시성이 아닌 병렬성을 가지는 물리적인 스레드이다. 

`parallel` 연산자는 시퀀스를 라운드 로빈 방식으로 CPU 코어만큼 스레드를 병렬로 실행한다. 

특히 `parallel` 연산자와 `runOn` 연산자를 결합하여 병렬 처리를 최적화할 수 있다. 

```java
/**
 * parallel() 기본 사용 예제
 * - parallel()만 사용할 경우에는 병렬로 작업을 수행하지 않는다.
 * - runOn()을 사용해서 Scheduler를 할당해주어야 병렬로 작업을 수행한다.
 * - **** CPU 코어 갯수내에서 worker thread를 할당한다. ****
 */
@Slf4j
public class Example10_3 {
    public static void main(String[] args) throws InterruptedException {
        Flux.fromArray(new Integer[]{1, 3, 5, 7, 9, 11, 13, 15, 17, 19})
                .parallel(4)
                .runOn(Schedulers.parallel()) // 중요! 할당해야함 
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}
// 로그를 보면 모든 log를 다른 스레드가 찎음 
```

## publishOn()과 subscribeOn()의 동작 이해

원본 Publisher의 동작과 나머지 동작을 역할에 맞게 분리하고자 두 오퍼레이터를 같이 사용할 수 있다. 

publishOn()과 subscribeOn()을 사용하지 않을 경우 Operator 체인에서는 같은 스레드로 실행된다. 

publisherOn()을 추가하면 지정한 해당 Scheduler 유형의 스레드가 실행된다 (해당 코드 아래로) 

```java
@Slf4j
public class Example10_6 {
    public static void main(String[] args) throws InterruptedException {
        Flux
            .fromArray(new Integer[] {1, 3, 5, 7})
            .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
            .publishOn(Schedulers.parallel())
            .filter(data -> data > 3)
            .doOnNext(data -> log.info("# doOnNext filter: {}", data))
            .map(data -> data * 10)
            .doOnNext(data -> log.info("# doOnNext map: {}", data))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}

결과
[main] INFO - # doOnNext fromArray: 1
[main] INFO - # doOnNext fromArray: 3
[main] INFO - # doOnNext fromArray: 5
[main] INFO - # doOnNext fromArray: 7
[parallel-1] INFO - # doOnNext filter: 5
[parallel-1] INFO - # doOnNext map: 50
[parallel-1] INFO - # onNext: 50
[parallel-1] INFO - # doOnNext filter: 7
[parallel-1] INFO - # doOnNext map: 70
[parallel-1] INFO - # onNext: 70
```

publishOn을 또다시 체이닝해서 사용할 수 있다. 이때 실행하는 스레드가 바뀐다

```java
/**
 * subscribeOn()과 publishOn()의 동작 과정 예
 *  - 두 개의 publishOn()을 사용한 경우
 *      - 다음 publishOn()을 만나기 전까지 publishOn() 아래 쪽 Operator들의 실행 쓰레드를 변경한다.
 *
 */
@Slf4j
public class Example10_7 {
    public static void main(String[] args) throws InterruptedException {
        Flux
            .fromArray(new Integer[] {1, 3, 5, 7})
            .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
            .publishOn(Schedulers.parallel())
            .filter(data -> data > 3)
            .doOnNext(data -> log.info("# doOnNext filter: {}", data))
            .publishOn(Schedulers.parallel())
            .map(data -> data * 10)
            .doOnNext(data -> log.info("# doOnNext map: {}", data))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}

// 결과

[main] INFO - # doOnNext fromArray: 1
[main] INFO - # doOnNext fromArray: 3
[main] INFO - # doOnNext fromArray: 5
[main] INFO - # doOnNext fromArray: 7
[parallel-2] INFO - # doOnNext filter: 5
[parallel-2] INFO - # doOnNext filter: 7
[parallel-1] INFO - # doOnNext map: 50 
[parallel-1] INFO - # onNext: 50
[parallel-1] INFO - # doOnNext map: 70
[parallel-1] INFO - # onNext: 70
```

* filter까지는 같은 스레드 실행. map -> doOnNext부터는 다른 스레드에서 실행 

subscribeOn과 publisheOn을 함께 사용할 경우에는 어떻게 될까?

```java
/**
 * subscribeOn()과 publishOn()의 동작 과정 예
 *  - subscribeOn()과 publishOn()을 함께 사용한 경우
 *      - subscribeOn()은 구독 직후에 실행될 쓰레드를 지정하고, publishOn()을 만나기 전까지 쓰레드를 변경하지 않는다.
 *
 */
@Slf4j
public class Example10_8 {
    public static void main(String[] args) throws InterruptedException {
        Flux
            .fromArray(new Integer[] {1, 3, 5, 7})
            .subscribeOn(Schedulers.boundedElastic())
            .doOnNext(data -> log.info("# doOnNext fromArray: {}", data))
            .filter(data -> data > 3)
            .doOnNext(data -> log.info("# doOnNext filter: {}", data))
            .publishOn(Schedulers.parallel())
            .map(data -> data * 10)
            .doOnNext(data -> log.info("# doOnNext map: {}", data))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(500L);
    }
}

결과
[boundedElastic-1] INFO - # doOnNext fromArray: 1
[boundedElastic-1] INFO - # doOnNext fromArray: 3
[boundedElastic-1] INFO - # doOnNext fromArray: 5
[boundedElastic-1] INFO - # doOnNext filter: 5
[boundedElastic-1] INFO - # doOnNext fromArray: 7
[parallel-1] INFO - # doOnNext map: 50
[boundedElastic-1] INFO - # doOnNext filter: 7
[parallel-1] INFO - # onNext: 50
[parallel-1] INFO - # doOnNext map: 70
[parallel-1] INFO - # onNext: 70
```

* subscribeOn()은 구독 직후에 실행될 쓰레드를 지정하고, publishOn()을 만나기 전까지 쓰레드를 변경하지 않는다.
* publishOn()을 만나면 실행하는 스레드가 바뀐다.

이처럼 subscribe() Operator와 publishOn() Operator를 함께 사용하면 

원본 Publisher에서 데이터를 emit하는 스레드와 emit된 데이터를 가공 처리하는 스레드를 적절하게 분리할 수 있다. 

* 발행보다 처리가 오래걸린다면 분리하는게 유리하지 않을까?



﻿﻿publishOn()과 subscribeon()의 특징

- ﻿﻿publishOn() Operator는 한 개 이상 사용할 수 있으며, 실행 스레드를 목적에 맞게 적절하게 분리할 수 있다.
- ﻿﻿subscribe() Operator와 publishOn() Operator를 함께 사용해서 원본 Publisher에 서 데이터를 emit하는 스레드와 emit된 데이터를 가공 처리하는 스레드를 적절하게 분리할 수 있다.
- ﻿﻿subscribeOn()은 Operator 체인상에서 어떤 위치에 있든 간에 구독 시점 직후, 즉 Publisher가 데이터를 emit하기 전에 실행 스레드를 변경한다.



## Scheduler의 종류

* Schedulers.immediate()
* Schedulers.single()
* Schedulers.boundedElastic()
* Schedulers.parallel()

**Schedulers.immediate()**

Schedulers.immediate()은 별도의 스레드를 추가적으로 생성하지 않고, 현재 스레드에서 작업을 처리하고자 할 때 사용할 수 있다.

```java
public class Example10_9 {
    public static void main(String[] args) throws InterruptedException {
        Flux
            .fromArray(new Integer[] {1, 3, 5, 7})
            .publishOn(Schedulers.parallel())
            .filter(data -> data > 3)
            .doOnNext(data -> log.info("# doOnNext filter: {}", data))
            .publishOn(Schedulers.immediate()) // here
            .map(data -> data * 10)
            .doOnNext(data -> log.info("# doOnNext map: {}", data))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(200L);
    }
}
```

* parallel로 publish하지만, immediate로 바뀐 순간부터 특정 스레드로 고정되어 처리된다. 



**Schedulers.single()**

Schedulers.single()은 스레드 하나만 생성해서 Scheduler가 제거되기 전까지 재사용하는 방식.

```java
/**
 * Schedulers.single() 예
 *  - Scheduler가 제거될 때까지 동일한 쓰레드를 재사용한다.
 *
 */
@Slf4j
public class Example10_10 {
    public static void main(String[] args) throws InterruptedException {
        doTask("task1")
                .subscribe(data -> log.info("# onNext: {}", data));

        doTask("task2")
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(200L);
    }

    private static Flux<Integer> doTask(String taskName) {
        return Flux.fromArray(new Integer[] {1, 3, 5, 7})
                .publishOn(Schedulers.single())
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# {} doOnNext filter: {}", taskName, data))
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# {} doOnNext map: {}", taskName, data));
    }
}
```

* Schedulers.single()을 사용했기 때문에 doTask()를 두 번 호출 하더라도 첫 번째 호출에서 이미 생성된 스레드를 재사용하게 된다.
* Schedulers.single()을 통해 하나의 스레드를 재사용하면서 다수의 작업 을 처리할 수 있는데, 하나의 스레드로 다수의 작업을 처리해야 되므로 지연시 간이 짧은 작업을 처리하는 것이 효과적이다..



**Schedulers.newSingle()**

Schedulers.newSingle()은 호출할 때마다 매번 새로운 스레드 하나를 생성한다.

```java
@Slf4j
public class Example10_11 {
    public static void main(String[] args) throws InterruptedException {
        doTask("task1")
                .subscribe(data -> log.info("# onNext: {}", data));

        doTask("task2")
                .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(200L);
    }

    private static Flux<Integer> doTask(String taskName) {
        return Flux.fromArray(new Integer[] {1, 3, 5, 7})
                .publishOn(Schedulers.newSingle("new-single", true))
                .filter(data -> data > 3)
                .doOnNext(data -> log.info("# {} doOnNext filter: {}", taskName, data))
                .map(data -> data * 10)
                .doOnNext(data -> log.info("# {} doOnNext map: {}", taskName, data));
    }
}
```

첫 번째 파라미터에는 생성할 스레드의 이름을 지정하고, 두 번째 파라미터는 이 스레드를 데몬 스레드로 동작하게 할지 여부를 설정한다.

* 데몬스레드는 메인스레드가 종료되면 자동으로 종료된다. true로 설정해두는것이 좋다. 안그러면 메인이 죽어도 살아있다. 



**Schedulers.boundedElastic()**

Schedulers boundedIFlastic()은 ExecutorService 기반의 스레드 풀Thread Pool을 생성한 후,

 그 안에서 정해진 수만큼의 스레드를 사용하여 작업을 처리하고 작업 이 종료된 스레드는 반납하여 재사용하는 방식

* 디폴트로 CPU 코어 수 x 10 만큼의 스레드를 생성하며, 최대 10만개의 작업이 대기 큐에 있을 수 있다. 

Schedulers.DoundedElastic()은 바로 Blocking 1/O 작업을 효과적으로 처리하기 위한 방식이다.

다른 Non-blokcing 처리에 영향을 주지 않도록 전용 스레드를 할당해서 Blocking I/O 작업을 처리한다.

```java
Flux.just("Task")
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe(System.out::println);
```



**Schedulers.parallel()**

schedulers.boundedElastic()이 Blocking I/O 작업에 최적화되어 있는 반면에, 

Schedulers.parallel()은 Non-Blocking I/O에 최적화되어 있는 Scheduler로서 CPU 코어 수만큼의 스레드를 생성한다.

CPU 바운드 작업을 처리하기 위해 사용되며 그런 작업에 적합하다.

```java
Flux.just("Task")
    .subscribeOn(Schedulers.parallel())
    .subscribe(System.out::println);
```



**Schedulers.fromExecutorService()**

기존 exeutorService로부터 Scheduler를 생성하는 방식이다. 리액터에서는 권장하지 않는다.

* Reactor의 기본 스케줄러는 Reactor의 내부 메커니즘과 최적화된 방식으로 통합되어 있어서, Reactor의 일부 최적화 기능을 사용할 수 없을수도 있다. 
* . 예를 들어, `Schedulers.boundedElastic()`는 필요한 만큼 스레드를 생성하지만, 특정 한도를 넘지 않도록 제한하여 시스템 리소스를 과도하게 사용하지 않도록 하는데, executorService는 다르게 동작할 수도 있다. 



**Schedulers.newXXXX()**

필요하다면 newSingle(), newBoundedElastic(), newParallel()을 이용해서 새 Scheduler 인스턴스를 생성할 수 있다.

즉, 스레드 이름, 생성 가능한 디폴트 스레드의 개수, 스레드의 유휴 시간, 데몬스레드 동작 여부 등을 직접 커스텀하게 할 수 있다.

