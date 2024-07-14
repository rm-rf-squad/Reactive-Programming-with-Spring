# Chapter 11 Context

Context의 사전적의미 : 문맥

특정 프로그램이 실행되는 어떤 상태, 정보 등을 기록하기 위한 용어로도 많이 쓰인다.

* ApplicationContext, SecurityContext 등

Reactor API 문서에서는 Context를 다음과 같이 정의한다

> A key/value store that is propagated between components such as operators via the context protocol.
>
> 컨텍스트 프로토콜을 통해 연산자와 같은 구성 요소 간에 전파되는 키/값 저장소.

여기서의 전파는 Downstream에서 Upstream으로 Context가 전파되어 Operator 체인상 각 Operator가 해당 Context의 정보를 동일하게 이용할 수 있음을 의미한다.

ThreadLocal가 비슷하지만, 실행 스레드와 1:1로 매핑되는것이 아닌, `Subscriber와 매핑된다!!`

* **즉 구독이 발생할 때마다 해당 구독과 연결된 하나의 Context가 생긴다.**
* 이거 모르면 꽤나 고생한다.. 트랜잭셔널과 시큐리티 컨텍스트가 어떻게 동작할까? 코루틴과 JPA와 MVC Security는 어떨까? 

```java
/**
 * Context 기본 예제
 *  - contextWrite() Operator로 Context에 데이터 쓰기 작업을 할 수 있다.
 *  - Context.put()으로 Context에 데이터를 쓸 수 있다.
 *  - deferContextual() Operator로 Context에 데이터 읽기 작업을 할 수 있다.
 *  - Context.get()으로 Context에서 데이터를 읽을 수 있다.
 *  - transformDeferredContextual() Operator로 Operator 중간에서 Context에 데이터 읽기 작업을 할 수 있다.
 */
@Slf4j
public class Example11_1 {
    public static void main(String[] args) throws InterruptedException {
        Mono
            .deferContextual(ctx ->
                Mono.just("Hello" + " " + ctx.get("firstName"))
                    .doOnNext(data -> log.info("# just doOnNext : {}", data))
            )
            .subscribeOn(Schedulers.boundedElastic())
            .publishOn(Schedulers.parallel())
            .transformDeferredContextual( // here
                    (mono, ctx) -> mono.map(data -> data + " " + ctx.get("lastName"))
            )
            .contextWrite(context -> context.put("lastName", "Jobs"))
            .contextWrite(context -> context.put("firstName", "Steve"))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}
```

`Context` 객체를 직접 생성

```java
package reactor.util.context;

Context context = Context.of("key1", "value1", "key2", "value2");
```



### 컨텍스트에 데이터 쓰기

CotextWrite() 오퍼레이터를 보면 Context에 데이터를 쓰고있다. 

```java
/**
 * 업스트림 연산자의 이익을 위해 다운스트림에서 보이는 Context를
 * 풍부하게 하기 위해, 다운스트림 Context에 Function을 적용합니다.
 * 
 * Function은 편의를 위해 Context를 받아들이며,
 * 이를 통해 쉽게 Context의 put 메서드를 호출하여
 * 새로운 Context를 반환할 수 있습니다.
 * 
 * Context (및 그 ContextView)는 특정 구독에 연결되며,
 * 다운스트림 Subscriber를 쿼리하여 읽습니다. 컨텍스트를 풍부하게 하지 않는
 * Subscriber는 대신 자신의 다운스트림 컨텍스트에 접근합니다.
 * 결과적으로, 이 연산자는 체인의 아래쪽(다운스트림)에서 오는
 * Context를 개념적으로 풍부하게 하여 새로운 풍부해진 Context를
 * 체인의 위쪽(업스트림) 연산자에게 보이도록 합니다.
 *
 * @param contextModifier 다운스트림 Context에 적용할 Function.
 * 이를 통해 업스트림에서 보이는 더 완전한 Context가 생성됩니다.
 *
 * @return 컨텍스트화된 Mono
 */
public final Mono<T> contextWrite(Function<Context, Context> contextModifier) {
    if (ContextPropagationSupport.shouldPropagateContextToThreadLocals()) {
        return onAssembly(new MonoContextWriteRestoringThreadLocals<>(
                this, contextModifier
        ));
    }
    return onAssembly(new MonoContextWrite<>(this, contextModifier));
}
```

```java
contextWrite(context -> context.put("firstName", "Steve"))
```

### 컨텍스트에서 데이터 읽기

2가지가 있다.

1개는 원본 데이터 소스 레벨에서 읽는 방식

나머지는 Operator 체인의 중간에서 읽는 방식



원본 데이터 소스 레벨에서려면 deferContextual()이라는 오퍼레이터를 사용해야 한다

* deferContextual()은 defer() Operator와 같은 원리로 동작하는데, Context에 저장된 데이터와 원본 데이터 소스의 처리를 지연시키는 역할한다.

```java
Mono.deferContextual(ctx ->
        Mono.just("Hello" + " " + ctx.get("firstName"))
            .doOnNext(data -> log.info("# just doOnNext : {}", data))
    )
    .subscribeOn(Schedulers.boundedElastic())
```

그런데 기억해야 될 사항이 하나 있는데, deferContextual()의 파라미터로 정의된 람다 표현식의 람다 파라미터(ctx)는 Context 타입의 객체가 아니라 ContexView 타입의 객체라는 것이다. Context에 데이터를 쓸 때는 Context 를 사용하지만, Context에 저장된 데이터를 읽을 때는 ContextView를 사용한다 는 사실을 기억해야 한다

* ContextView는 읽기 전용 인터페이스다. 무분별한 수정으로부터 Context를 보호하기 위함이다. Context를 참조하는 동안 불변성을 보장할 수 있다.

context,put()을 통해 Context에 데이터를 쓴 후에 매번 불변객체를 다음 context Write() Operator로 전달함으로써 스레드 세이프를 보장한다.

## 자주 사용되는 Context 관련 API

| 메서드                                | 설명                                                      |
| ------------------------------------- | --------------------------------------------------------- |
| `put(key, value)`                     | key/value 형태로 Context에 값을 쓴다.                     |
| `of(key1, value1, key2, value2, ...)` | key/value 형태로 Context에 여러 개의 값을 쓴다.           |
| `putAll(ContextView)`                 | 현재 Context와 파라미터로 입력된 ContextView를 merge한다. |
| `delete(key)`                         | Context에서 key에 해당하는 value를 삭제한다.              |

* 최대 5개의 파라미터를 쓸 수 있다. 
* https://projectreactor.io/docs/core/release/api/index.html dㅔ서 검색 가능

```java
@Slf4j
public class Example11_3 {
    public static void main(String[] args) throws InterruptedException {
        final String key1 = "company";
        final String key2 = "firstName";
        final String key3 = "lastName";

        Mono
            .deferContextual(ctx -> // 읽을때는 ContextView
                    Mono.just(ctx.get(key1) + ", " + ctx.get(key2) + " " + ctx.get(key3))
            )
            .publishOn(Schedulers.parallel())
            .contextWrite(context -> // 쓰기 
                    context.putAll(Context.of(key2, "Steve", key3, "Jobs").readOnly())
            )
            .contextWrite(context -> context.put(key1, "Apple"))
            .subscribe(data -> log.info("# onNext: {}" , data));

        Thread.sleep(100L);
    }
}

```



### ContextView 관련 API

| 메서드                             | 설명                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `get(key)`                         | ContextView에서 key에 해당하는 value를 반환한다.             |
| `getOrEmpty(key)`                  | ContextView에서 key에 해당하는 value를 Optional로 래핑해서 반환한다. |
| `getOrDefault(key, default value)` | ContextView에서 key에 해당하는 value를 가져온다. key에 해당하는 value가 없으면 default value를 가져온다. |
| `hasKey(key)`                      | ContextView에서 특정 key가 존재하는지를 확인한다.            |
| `isEmpty()`                        | Context가 비어 있는지 확인한다.                              |
| `size()`                           | Context 내에 있는 key/value의 개수를 반환한다.               |

```java
/**
 * ContextView API 사용 예제
 */
@Slf4j
public class Example11_4 {
    public static void main(String[] args) throws InterruptedException {
        final String key1 = "company";
        final String key2 = "firstName";
        final String key3 = "lastName";

        Mono
            .deferContextual(ctx ->
                    Mono.just(ctx.get(key1) + ", " +
                            ctx.getOrEmpty(key2).orElse("no firstName") + " " +
                            ctx.getOrDefault(key3, "no lastName"))
            )
            .publishOn(Schedulers.parallel())
            .contextWrite(context -> context.put(key1, "Apple"))
            .subscribe(data -> log.info("# onNext: {}" , data));

        Thread.sleep(100L);
    }
}
```



추가 예제

#### 예제 1: 사용자 인증 정보를 Context에 저장 및 전파

```java
Mono<String> getUserInfo(String userId) {
    return Mono.just("User Info for " + userId);
}

Mono<String> secureMethod() {
    return Mono.deferContextual(ctx -> {
        if (ctx.hasKey("authToken")) {
            String authToken = ctx.get("authToken");
            // 인증 토큰을 사용하여 사용자 정보를 가져옴
            return getUserInfo(authToken);
        } else {
            return Mono.error(new RuntimeException("No auth token in context"));
        }
    });
}

Mono<String> result = secureMethod()
    .contextWrite(Context.of("authToken", "user123"));

result.subscribe(System.out::println, Throwable::printStackTrace); // 출력: User Info for user123
```

#### 예제 2: 설정 값을 Context에 저장 및 전파

```java
Mono<String> configurationMethod() {
    return Mono.deferContextual(ctx -> {
        String configValue = ctx.getOrDefault("configKey", "defaultConfig");
        return Mono.just("Configuration: " + configValue);
    });
}

Mono<String> result = configurationMethod()
    .contextWrite(Context.of("configKey", "customConfig"));

result.subscribe(System.out::println); // 출력: Configuration: customConfig
```

## Context 특징

Context를 사용할때 주의할점이 있다. 매우 버그를 만날 가능성이 높으므로 주의하자.



* Context는 구독이 발생할 때마다 하나의 Context가 해당 구독에 연결된다.

```java
/**
 * Context의 특징 예제
 *  - Context는 각각의 구독을 통해 Reactor Sequence에 연결 되며 체인의 각 Operator는 연결된 Context에 접근할 수 있어야 한다.
 */
@Slf4j
public class Example11_5 {
    public static void main(String[] args) throws InterruptedException {
        final String key1 = "company";

        Mono<String> mono = Mono.deferContextual(ctx ->
                        Mono.just("Company: " + " " + ctx.get(key1))
                )
                .publishOn(Schedulers.parallel());


        mono.contextWrite(context -> context.put(key1, "Apple"))
                .subscribe(data -> log.info("# subscribe1 onNext: {}", data));

        mono.contextWrite(context -> context.put(key1, "Microsoft"))
                .subscribe(data -> log.info("# subscribe2 onNext: {}", data));

        Thread.sleep(100L);
    }
}

// 결과
[parallel-2] INFO - # subscribe2 onNext: Company:  Microsoft
[parallel-1] INFO - # subscribe1 onNext: Company:  Apple
```

* 2개 구독 (mono.contextWrite(context -> context.put(key1, "Apple")))의 subscribe에 저장한 context 값은 다르다.

```java
@Slf4j
public class Example11_6 {
    public static void main(String[] args) throws InterruptedException {
        String key1 = "company";
        String key2 = "name";

        Mono
            .deferContextual(ctx ->
                Mono.just(ctx.get(key1))
            )
            .publishOn(Schedulers.parallel())
            .contextWrite(context -> context.put(key2, "Bill"))
            .transformDeferredContextual((mono, ctx) ->
                    mono.map(data -> data + ", " + ctx.getOrDefault(key2, "Steve"))
            )
            .contextWrite(context -> context.put(key1, "Apple"))
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}
// [parallel-1] INFO - # onNext: Apple, Steve

/**
 * Context의 특징 예제
 *  - Context는 Operator 체인의 아래에서부터 위로 전파된다.
 *      - 따라서 Operator 체인 상에서 Context read 메서드가 Context write 메서드 밑에 있을 경우에는 write된 값을 read할 수 없다.
 */
```

* 구독은 1개다 (subscribe)
* bill은 읽을 수 없다. 14번에서 getOrDefault(key2)로 읽지만, 그 위에서 context.put(key2)를 하고있기 때문이다. 
  * get을 호출하면 없기때문에 noSucheElementException이 발생해버린다. 
* Context는 Operator 체인의 아래에서 위로 전파된다.
* 동일한 키에 대한 값을 중복해서 저장하면 Operator 체인에서 가장 위쪽에 위치한 contextWrite()이 저장한 값으로 덮어쓴다.

따라서 일반적으로 모든 Operator에서, Context에 저장된 데이터를 읽을 수 있도록 contextWirte()를 Operator 체인의 맨 마지막에 두는것이 좋다.

또한 Operator 체인상에 동일한 키를 가지는 값을 각각다른 값으로 저장한다면, Context가 아래에서 위로 전파되는 특성 때문에 Operator 체인상에서 가장 위 쪽에 위치한 contextWrite()이 저장한 값으로 덮어쓰게 된디는 사실 역시 중요하다. 

**또한 다음 내용도 참고하자.**

```java
@Slf4j
public class Example11_7 {
    public static void main(String[] args) throws InterruptedException {
        String key1 = "company";

        Mono.just("Steve")
            .transformDeferredContextual((stringMono, ctx) ->
                    // 여기서 'role' 키를 참조하려고 합니다.
                    // 그러나 현재 'role' 키가 Context에 없기 때문에 예외가 발생.
                                         // 여기는 외부 시퀀스 
                    ctx.get("role"))
            .flatMap(name ->
                Mono.deferContextual(ctx ->
                    Mono.just(ctx.get(key1) + ", " + name)
                        .transformDeferredContextual((mono, innerCtx) ->
                            // 이 부분이 이너 시퀀스
                            // 이너 시퀀스는 외부 시퀀스의 컨텍스트를 상속받지 않기 때문에 
                            // 별도의 컨텍스트를 사용다.
                            mono.map(data -> data + ", " + innerCtx.get("role"))
                        )
                        // 여기서 'role' 키를 추가하지만, 이는 이너 시퀀스에서만 적용.
                        .contextWrite(context -> context.put("role", "CEO"))
                )
            )
            .publishOn(Schedulers.parallel())
            .contextWrite(context -> context.put(key1, "Apple")) // 'company' 키를 Context에 추가
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}

```

- 이너 시퀀스는 **플랫맵(flatMap)** 안에 포함된 시퀀스를 의미.﻿﻿
- Inner Sequence 내부에서는 외부 Context에 저장된 데이터를 읽을 수 있다.
- Inner Sequence 외부에서는 Inner Sequence 내부 Context에 저장된 데이터를 읽을 수 없다.

왜그럴까?

**이너 시퀀스는 외부 시퀀스의 Context를 상속받지 않기 때문**. 

* **이너 시퀀스**: 별도의 구독과 Context를 가지며, 외부 시퀀스의 Context를 상속받지 않는다. 이너 시퀀스 내에서 별도로 Context를 설정해야 한다.

어떻게 해결할 수 있을까?

이를 해결하려면 Context 설정과 참조의 순서를 조정해야 한다.. 

특히, `transformDeferredContextual` 내에서 Context를 참조하기 전에 Context를 설정해야 한다 

```java
@Slf4j
public class Example11_7 {
    public static void main(String[] args) throws InterruptedException {
        String key1 = "company";

        Mono.just("Steve")
            // 첫 번째 transformDeferredContextual에서 'role' 키를 미리 설정
            .contextWrite(context -> context.put("role", "CEO"))
            .flatMap(name ->
                Mono.deferContextual(ctx ->
                    Mono.just(ctx.get(key1) + ", " + name)
                        .transformDeferredContextual((mono, innerCtx) ->
                            mono.map(data -> data + ", " + innerCtx.get("role"))
                        )
                        // 이너 시퀀스에서도 'role' 키를 설정
                        .contextWrite(context -> context.put("role", "CEO"))
                )
            )
            .publishOn(Schedulers.parallel())
            .contextWrite(context -> context.put(key1, "Apple")) // 'company' 키를 Context에 추가
            .subscribe(data -> log.info("# onNext: {}", data));

        Thread.sleep(100L);
    }
}

```

