# Context

## Context 란?

Context는 어떤 상황에서 그 상황을 처리하기 위해 필요한 정보를 의미합니다.

### 프로그래밍에서의 Context 예시

- ServletContext는 Servlet이 Servlet Container와 통신하기 위해서 필요한 정보를 제공하는 인터페이스입니다.
- Spring Framework에서 ApplicationContext는 애플리케이션의 정보를 제공하는 인터페이스입니다. ApplicationContext가 제공하는 대표적인 정보가 Spring Bean 객체입니다.
- Spring Security에서 SecurityContextHolder는 SecurityContext를 관리하는 주체인데, 여기서의 SecurityContext는 애플리케이션 사용자의 인증 정보를 제공하는 인터페이스 입니다.

### Reactor에서의 Context

Reactor의 Context는 Operator 같은 Reactor 구성요소 간에 전파되는 key value 형태의 저장소라고 정의합니다.

<aside>
💡 여기서 전파는 Downstream에서 Upstream으로 Context가 전파되어 Operator 체인상의 각 Operator가 해당 Context의 정보를 동일하게 이용할 수 있음을 의미합니다.

</aside>

Reactor의 Context는 ThreadLocal과 다소 유사한면이 있지만, 각각의 실행 스레드와 매핑되는 ThreadLocal과 달리 실행 스레드와 매핑되는 것이 아니라 Subscriber와 매핑됩니다.

즉, 구독이 발생할 때마다 해당 구독과 연결된 하나의 Context가 생깁니다.

### 컨텍스트에 데이터 쓰기

contextWrite() Operator를 사용하여 컨텍스트에 데이터를 쓸 수 있습니다.

```java
public abstract class Mono<T> implements Corepublisher<T> {
	public final Mono<T> contextWrite(Function<Context, Context> contextModifier) {
		return onAssembly(new MonoContextWrite<>(this, contextModifier));
	}
}
```

contextWrite() 매개변수로 Function 함수형 인터페이스를 받아 Context의 데이터를 쓰는 방법을 전달할 수 있습니다.

### Context에 쓰인 데이터 읽기

Context에서 데이터를 읽는 방식은 크게 두가지 입니다.

- 원본 데이터 소스레벨에서 읽는 방식
    - deferContextual() Operator를 사용합니다.
        - defer() Operator와 같은 원리로 동작하며, Context에 저장된 데이터와 원본 데이터 소스의 처리를 지연시키는 역할을 합니다.
        - 파라미터로 ContextView 타입의 객체를 전달받습니다. Context의 데이터를 쓸 때는 Context를 사용하지만 Context에 저장된 데이터를 읽을 때는 ContextView를 사용합니다.
- Operator 체인의 중간에서 읽는 방식
    - trnasformDeferredContextual() Operator를 사용합니다.

### Reactor Context의 특징

- Reactor에서는 Operator 체인상의 서로 다른 스레드들이 Context의 저장된 데이터에 손쉽게 접근할 수 있습니다.
- context.put()을 통해 Context에 데이터를 쓴 후에 매번 불변 객체를 contextWrite() Operator로 전달함으로써 스레드 안전성을 보장합니다.

### 정리

- Context는 일반적으로 어떠한 상황에서 그 상황을 처리하기 위해 필요한 정보를 의미합니다.
- Reactor에서의 Context
    - Operator 체인에서 전파되는 키와 값 형태의 저장소입니다.
    - Subscriber와 매핑되어 구독이 발생할 때마다 해당 구독과 연결된 하나의 Context가 생깁니다.
    - contextWriter() Operator를 사용해서 Context에 데이터 쓰기 작업을 할 수 있습니다.
    - deferContextual() Operator를 사용해서 원본 데이터 소스 레벨에서 Context의 데이터를 읽을 수 있습니다.
    - transformDeferredContextual() Operator를 사용해서 Operator 체인의 중간에서 데이터를 읽을 수 있습니다.

## 자주 사용되는 Context 관련 API

### 자주 사용되는 Context API

| Context API | 설명 |
| --- | --- |
| put(key, value) | key / value 형태로 Context에 값을 씁니다. |
| of(key1, value2, key2, value2, …) | key / value 형태로 Context에 여러 개의 값을 씁니다. |
| putAll(ContextView) | 현재 Context와 파리미터로 입력된 ContextView를 merge 합니다. |
| delete(key) | Context에서 key에 해당하는 value를 삭제합니다. |

### 자주 사용되는 Context View API

| ContextView API | 설명 |
| --- | --- |
| get(key) | ContextView에서 key에 해당하는 value를 반환합니다. |
| getOrEmpty(key) | ContextView에서 key에 해당하는 value를 Optional로 래핑해서 반환합니다. |
| getOrDefault(key, default value) | ContextView에서 key에 해당하는 value를 가져옵니다. key에 해당하는 value가 없으면 default value를 가져옵니다. |
| hasKey(key) | ContextView에서 특정 key가 존재하는지를 확인합니다. |
| isEmpty() | Context가 비어 있는지 확인합니다. |
| size() | Context 내에 있는 key/value의 개수를 반환합니다. |

### 정리

- Context에 데이터를 쓰기 위해서는 Context API를 사용해야 합니다.
- Context에서 데이터를 읽기 위해서는 ContextView API를 사용해야 합니다.

## Context의 특징

### Context는 구독이 발생할 때마다 하나의 Context가 해당 구독에 연결됩니다.

```java
    public static void main(String[] args) throws Exception {
        String key = "company";

        Mono<String> mono = Mono.deferContextual(ctx -> Mono.just("Company: " + " " + ctx.get(key)))
                .publishOn(Schedulers.parallel());

        mono.contextWrite(context -> context.put(key, "Apple"))
                .subscribe(data -> log.info("# subscribe1 onNext: {}", data));

        mono.contextWrite(context -> context.put(key, "Microsoft"))
                .subscribe(data -> log.info("# subscribe2 onNext: {}", data));

        Thread.sleep(100L);
    }
```

두 개의 데이터가 하나의 Context에 저장될 것 같지만, Context는 구독별로 연결되는 특징이 있기 때문에 구독이 발생할 때마다 해당하는 하나의 Context가 하나의 구독에 연결됩니다.

### Context는 Operator 체인의 아래에서 위로 전파됩니다.

```java
    public static void main(String[] args) throws Exception {
        String key1 = "company";
        String key2 = "name";

        Mono.deferContextual(ctx -> Mono.just(ctx.get(key1))
                        .publishOn(Schedulers.parallel())
                        .contextWrite(context -> context.put(key2, "John"))
                        .transformDeferredContextual((mono, context) -> mono.map(data -> data + ", " + context.getOrDefault(key2, "Doe")))
                )
                .contextWrite(context -> context.put(key1, "Apple"))
                .subscribe(data -> log.info("# onNext: {}", data));
        Thread.sleep(100L);
    }
```

Context에서 John을 저장했지만 Doe가 출력되는 것을 볼 수 있는데 Context의 경우 Operator 체인 상의 아래에서 위로 전파되는 특징이 있기 때문입니다.

ContextView 객체를 통해 name이라는 키로 데이터를 조회하지만 해당 시점에는 Context에 값이 존재하지 않아 Doe가 출력됩니다.

일반적으로 모든 Operator에서 Context에서 저장된 데이터를 읽을 수 있도록 contextWrite()을 Operator 체인의 맨 마지막에 둡니다.

또한 이러한 특성때문에 Operator 체인상에서 가장 위쪽에 위치한 contextWrite()이 저장한 값으로 덮어쓰게 됩니다.

### Inner Sequence 내부에서는 외부 Context에 저장된 데이터를 읽을 수 있습니다.
Inner Sequence 외부에서는 Inner Sequence 내부 Context에 저장된 데이터를 읽을 수 없습니다.

```java
    public static void main(String[] args) throws Exception {
        String key = "company";

        Mono.just("Steve")
//                .transformDeferredContextual(((stringMono, contextView) -> contextView.get("role")))
                .flatMap(name -> Mono.deferContextual(ctx -> Mono.just(ctx.get(key) + ", " + name)
                        .transformDeferredContextual(((stringMono, innerCtx) -> stringMono.map(data -> data + ", " + innerCtx.get("role"))))
                        .contextWrite(context -> context.put("role", "CEO"))))
                .publishOn(Schedulers.parallel())
                .contextWrite(context -> context.put(key, "Apple"))
                .subscribe(data -> log.info("# onNext: {}", data));
        Thread.sleep(100L);
    }
```

flatMap 내부에 있는 Sequence를 Inner Sequence라고 하는데, Inner Sequence에서는 Inner Sequence 바깥쪽 Sequence에 연결된 Context의 값을 읽을 수 있습니다.

만약 주석을 해제하게 되면 Context에 role이라는 키가 없어 NoSuchElementException이 발생합니다.

따라서, Inner Sequence 외부에서는 Inner Sequence 내부 Context에 저장된 데이터를 읽을 수 없음을 알 수 있습니다.

### Context는 인증 정보 같은 직교성(독립성)을 가지는 정보를 전송하는 데 적합합니다.

```java
    public static final String HEADER_AUTH_TOKEN = "authToken";

    public static void main(String[] args) {
        Mono<String> mono = postBook(Mono.just(new Book("abcd-1111-3533-2809", "Reactors bible", "Kevin")))
                .contextWrite(Context.of(HEADER_AUTH_TOKEN, "eyJhbGciOi"));
        mono.subscribe(data -> log.info("# onNext: {}", data));
    }

    private static Mono<String> postBook(Mono<Book> book) {
        return Mono.zip(book, Mono.deferContextual(ctx -> Mono.just(ctx.get(HEADER_AUTH_TOKEN))))
                .flatMap(tuple -> {
                    String response = "POST the book(" + tuple.getT1().getBookName() + "," + tuple.getT1().getAuthor() + ") with token: " + tuple.getT2();
                    return Mono.just(response); // HTTP POST를 전송 했다고 가정
                });
    }

    @AllArgsConstructor
    @Data
    static class Book {
        private String isbn;
        private String bookName;
        private String author;
    }
```

위의 코드는 인증된 도서 관리자가 신규 도서를 등록하기 위해 도서 정보와 인증 토큰을 서버로 전송한다고 가정합니다.

- 도서 정보인 Book을 전송하기 위해 Mono<Book> 객체를 postBook() 메서드의 파라미터로 전달합니다.
- zip() Operator를 사용해 Mono<Book> 객체와 인증 토큰 정보를 의미하는 Mono<String> 객체를 하나의 Mono로 합칩니다. 이때 합쳐진 Mono는 Mono<Tuple2>의 객체가 됩니다.
- flatMap() Operator 내부에서 도서 정보를 전송합니다.

예제의 핵심은 Context에 저장한 인증 토큰을 두 개의 Mono를 합치는 과정에서 다시 Context로부터 읽어 와서 사용한다는 것 입니다.

Mono가 어떤 과정을 거치든 상관없이 가장 마지막에 리턴된 Mono를 구독하기 직전에 contextWrite()으로 데이터를 저장하기 때문에 Operator 체인의 위쪽으로 전파되고, Operator 체인 어느 위치에서든 Context에 접근할 수 있는 것 입니다.

### 정리

- Context는 구독이 발생할 때마다 하나의 Context가 해당 구독에 연결됩니다.
- Context는 Operator 체인의 아래에서 위로 전파됩니다.
- 동일한 키에 대한 값을 중복해서 저장하면 Operator 체인상에서 가장 위쪽에 위치한 contextWrite()이 저장한 값으로 덮어씁니다.
- Inner Sequence 내부에서는 외부 Context에 저장된 데이터를 읽을 수 있습니다.
- Inner Sequence 외부에서는 Inner Sequence 내부 Context에 저장된 데이터를 읽을 수 없습니다.
- Context는 인증 정보 같은 직교성(독립성)을 가지는 정보를 전송하는데 적합합니다.