# Chapter 19 예외 처리

Spring MVC 기반의 애플리케이션에서 @ExceptionHandler나 @ControllerAdvice 등의 애너테이션을 이용하는 예외 처리 방식은 Spring WebFlux 기반 애플리케이션에서도 사용할 수 있는 방식이다.



## onErrorResume() 오퍼레이터를 통한 예외처리

```java
public Mono<ServerResponse> createBook(ServerRequest request) {
        return request.bodyToMono(BookDto.Post.class)
                .doOnNext(post -> validator.validate(post))
                .flatMap(post -> bookService.createBook(mapper.bookPostToBook(post)))
                .flatMap(book -> ServerResponse
                        .created(URI.create("/v9/books/" + book.getBookId()))
                        .build())
                .onErrorResume(BusinessLogicException.class, error -> ServerResponse
                            .badRequest()
                            .bodyValue(new ErrorResponse(HttpStatus.BAD_REQUEST,
                                                            error.getMessage())))
                .onErrorResume(Exception.class, error ->
                        ServerResponse
                                .unprocessableEntity()
                                .bodyValue(
                                    new ErrorResponse(HttpStatus.INTERNAL_SERVER_ERROR,
                                                        error.getMessage())));
}
```

* 에러발생시 에러시그널을 받아 처리하는 방식이다
* 그런데 매번 이렇게 알수없는 예외들을 보낸다는것은 매우 비효율적으로 보인다

## ErrorWebExceptionHandler를 이용한 글로벌 예외처리

클래스 내에 여러 개의 sequence가 존재 한다면 각 Sequence마다 onErrorResume() Operator를 일일이 추가해야 되 고 중복 코드가 발생할 수 있는 단점이 존재한다.

이런 단점을 보완할 수 있는 exceptionHandler가 존재한다.

```java
import org.springframework.boot.web.reactive.error.ErrorWebExceptionHandler;

@Order(-2)
@Configuration
public class GlobalWebExceptionHandler implements ErrorWebExceptionHandler {
    private final ObjectMapper objectMapper;

    public GlobalWebExceptionHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public Mono<Void> handle(ServerWebExchange serverWebExchange,
                             Throwable throwable) {
        return handleException(serverWebExchange, throwable);
    }

    private Mono<Void> handleException(ServerWebExchange serverWebExchange,
                                       Throwable throwable) {
        ErrorResponse errorResponse = null;
        DataBuffer dataBuffer = null;

        DataBufferFactory bufferFactory =
                                serverWebExchange.getResponse().bufferFactory();
        serverWebExchange.getResponse().getHeaders()
                                        .setContentType(MediaType.APPLICATION_JSON);

        if (throwable instanceof BusinessLogicException) {
            BusinessLogicException ex = (BusinessLogicException) throwable;
            ExceptionCode exceptionCode = ex.getExceptionCode();
            errorResponse = ErrorResponse.of(exceptionCode.getStatus(),
                                                exceptionCode.getMessage());
            serverWebExchange.getResponse()
                        .setStatusCode(HttpStatus.valueOf(exceptionCode.getStatus()));
        } else if (throwable instanceof ResponseStatusException) {
            ResponseStatusException ex = (ResponseStatusException) throwable;
            errorResponse = ErrorResponse.of(ex.getStatus().value(), ex.getMessage());
            serverWebExchange.getResponse().setStatusCode(ex.getStatus());
        } else {
            errorResponse = ErrorResponse.of(HttpStatus.INTERNAL_SERVER_ERROR.value(),
                                                            throwable.getMessage());
            serverWebExchange.getResponse()
                                    .setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
        }

        try {
            dataBuffer =
                    bufferFactory.wrap(objectMapper.writeValueAsBytes(errorResponse));
        } catch (JsonProcessingException e) {
            bufferFactory.wrap("".getBytes());
        }

        return serverWebExchange.getResponse().writeWith(Mono.just(dataBuffer));
    }
}
```

* ErrorWebFluxAutoConfiguration을 통해 등록된 DefaultErrorWebExceptionHandler 보다 높은 우선순위를 갖도록 Order(-2)를 지정하였다. 
* ErrorWebExceptionHandler를 구현하면 글로벌하게 동작한다. 
* ErrorResponse 객체를 DataBuffer로 래핑하여 response body를 구성한다. 
  * DataBuffer는 netty가 아닌 플랫폼에서도 사용할수있도록 추상화한 바이트 컨테이너이다.