# 함수형 엔드포인트

## HandlerFunction을 사용한 request 처리

Spring WebFlux의 함수형 엔드포인트는 들어오는 요청을 처리하기 위해 HandlerFunction이라는 함수형 기반의 핸들러를 사용합니다.

```java
@FunctionalInterface
public interface HandlerFunction<T extends ServerResponse> {
	Mono<T> handle(ServerRequest request);
}

```

서블릿 기반의 요청 처리는 Servlet 인터페이스의 service(ServletRequest req, ServletResponse res) 메서드의 파라미터로 전달받는 ServletRequest, ServletResponse res) 메서드의 파라미터로 전달받는 ServletRequest, ServletResponse에 의해 이루어집니다.

반면에 HandlerFunction은 요청 처리를 위한 ServerRequest 하나만 handle() 메서드의 파라미터로 전달받으며, 요청 처리에 대한 응답은 Mono<ServerResponse>의 형태로 리턴됩니다.

### ServerRequest

ServerRequest는 HandlerFunction에 의해 처리되는 HTTP request를 표현합니다.

ServerRequest는 객체를 통해 HTTP headers, method, URI, query parameters에 접근할 수 있는 메서드를 제공하며, HTTP body 정보에 접근하기 위해 body(), bodyToMono(), bodyToFlux() 같은 별도의 메서드를 제공합니다.

### ServerResponse

ServerResponse는 HandlerFunction 또는 HandlerFilterFunction에서 리턴되는 HTTP response를 표현합니다.

ServerResponse는 BodyBuilder와 HeadersBuilder를 통해 HTTP response body와 header 정보를 추가할 수 있습니다.

## request 라우팅을 위한 RouterFunction

RouterFunction은 들어오는 요청을 해당 HandlerFunction으로 라우팅해 주는 역할을 합니다.

RouterFunction은 @RequestMapping 어노테이션과 동일한 역할을 한다고 생각하면 됩니다.

RouterFunction은 요청을 위한 데이터뿐만 아니라 요청 처리를 위한 동작까지 RouterFunction의 파라미터로 제공한다는 차이점이 있습니다.

```java
@FunctionalInterface
public interface RouterFunction<T extends ServerResponse> {
	Optional<HandlerFunction<T>> route(ServerRequest request);
}
```

RouterFunction 인터페이스는 router() 메서드 하나만 정의되어 있는 함수형 인터페이스로서 route() 메서드에서 파라미터로 전달받은 request에 매치되는 HandlerFunction을 리턴합니다.

### 함수형 엔드포인트로 작성하기

```java
@Configuration
public class BookHandler {

    @Bean
    public RouterFunction<ServerResponse> routeBook(BookHandler handler) {
        return route()
                .POST("/v1/books", handler::createBook)
                .build();
    }
}

```

@Configuration 어노테이션을 추가하고 요청에 대해 라우팅 해주는 클래스입니다.

- route() 메서드는 RouterFunctionBuilder 객체를 리턴합니다.
- RouterFunctionBuilder 객체로 각각의 HTTP Method에 매치되는 request를 처리하기 위한 라우트를 추가합니다.
- build() 메서드를 호출하여 최종적으로 RouterFunction을 생성합니다.

```java
@Component
public class BookHandler {
    public Mono<ServerResponse> createBook(ServerRequest request) {
        return Mono.just("Book created")
                .flatMap(data -> ServerResponse.ok().bodyValue(data));
    }

}
```

위는 요청을 매핑해주는 핸들러의 예시입니다.

## 함수형 엔드포인트에서의 request body 유효성 검증

### Custom Validator를 이용한 유효성 검증

함수형 엔드포인트는 Spring의 Validator 인터페이스를 구현한 CustomValidator를 이용해 request body에 유효성 검증을 적용할 수 있습니다.

```java
@Component
public class BookCustomValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return String.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        String book = (String) target;
        if (book.length() < 5) {
            errors.rejectValue("book", "book.length", "Book name must be at least 5 characters long");
        }
    }
}
```

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class BookHandler {

    private final BookCustomValidator bookCustomValidator;
    public Mono<ServerResponse> createBook(ServerRequest request) {
        return Mono.just("Book created")
                .doOnNext(this::validate)
                .flatMap(data -> ServerResponse.ok().bodyValue(data));
    }

    private void validate(String book) {
        Errors errors = new BeanPropertyBindingResult(book, book);

        bookCustomValidator.validate(book, errors);
        if (errors.hasErrors()) {
            log.error(errors.getAllErrors().toString());
            throw new ServerWebInputException(errors.toString());
        }
    }
}
```

### 표준 Bean Validation을 이용한 유효성 검증

Custom Validation 이용한 검증 방식은 비즈니스 로직의 코드와 유효성 검증을 위한 코드가 뒤섞여 있기 때문에 바람직한 방식이 아닙니다.

그리고 다른 핸들러가 추가된다면 유효성 검증을 위한 동일한 코드가 핸들러에서 반복적으로 나타날 수 있습니다.

표준 Bean Validation을 이용해 유효성 검증을 하기 위해서는 Validator 인터페이스가 필요합니다.

Spring WebFlux에서는 Spring에서 지원하는 Validator 인터페이스와 javax에서 지원하는 표준 Validator 인터페이스 중 하나를 사용해 Validator 구현체를 주입받아서 유효성 검증을 진행할 수 있습니다.

<aside>
💡 어느 쪽을 사용하더라도 유효성 검증을 진행하기 위해 Validator 구현체인 Hibernate Validator를 내부적으로 사용합니다.

</aside>

### Spring Validator 인터페이스 사용

```java
public void validate(T body) {
	Errors errors = new BeanPropertyBindingResult(body, body.getClass().getName());
	
	this.validator.validate(body, errors);
	
	if (!errors.getAllErrors().isEmpty()) {
		onValidationErrors(errors);
	}
}
```

validate() 메서드의 파라미터는 핸들러에서 validate() 메서드를 호출할 때 전달하는 유효성 검증 대상 DTO 클래스의 객체입니다.

### javax 표준 Validator 인터페이스 사용

```java
public void validate(T body) {
	Set<ConstraintViolation<T>> constraintViolations = validator.validate(body);
	
	if (!constraintViolations.isEmpty()) {
		onValidationErrors(constraintViolations);
	}
	
}
```