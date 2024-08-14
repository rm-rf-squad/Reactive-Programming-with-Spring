# 예외처리

## onErrorResume() Operator를 이용한 예외처리

onErrorResume() Operator는 에러 이벤트가 발생했을 때 에러 이벤트를 Downstream으로 전파하지 않고 대체 Publisher를 통해 에러 이벤트에 대한 대체 값을 emit하거나 발생한 에러 이벤트를 래핑한 후에 다시 에러 이벤트를 발생시키는 역할을 합니다.

```java
@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public Mono<ServerResponse> getUserById(ServerRequest request) {
        String userId = request.pathVariable("id");

        return userService.getUserById(userId)
            .flatMap(user -> ServerResponse.ok().bodyValue(user))
            .switchIfEmpty(ServerResponse.badRequest().bodyValue("User not found"))
            .onErrorResume(e -> ServerResponse.status(500).bodyValue("Internal Server Error"));
    }
}
```

## ErrorWebExceptionHandler를 이용한 글로벌 예외 처리

onErrorResume() 같은 Operator를 통한 이용한 인라인 예외 처리는 사용하기 간편하다는 장점이 있습니다.

하지만 클래스 내에 여러개의 Sequence가 존재한다면 각 Sequence마다 onErrorResume() Operator를 일일이 추가해야하고 중복 코드가 발생할 수 있는 단점이 존재합니다.

```java
@Configuration
@Order(-2)  // 우선순위를 설정합니다. 낮을수록 높은 우선순위입니다.
public class GlobalErrorWebExceptionHandler implements ErrorWebExceptionHandler {

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        // 상태 코드와 메시지를 설정
        HttpStatus status = HttpStatus.INTERNAL_SERVER_ERROR;
        String message = "An unexpected error occurred";

        if (ex instanceof UserNotFoundException) {
            status = HttpStatus.BAD_REQUEST;
            message = ex.getMessage();
        }

        // 에러 응답을 JSON 형식으로 작성
        Map<String, Object> errorResponse = new HashMap<>();
        errorResponse.put("status", status.value());
        errorResponse.put("error", status.getReasonPhrase());
        errorResponse.put("message", message);
        errorResponse.put("path", exchange.getRequest().getPath().toString());

        // 응답 설정
        exchange.getResponse().setStatusCode(status);
        exchange.getResponse().getHeaders().setContentType(MediaType.APPLICATION_JSON);

        return exchange.getResponse().writeWith(
            Mono.just(exchange.getResponse()
                .bufferFactory()
                .wrap(errorResponse.toString().getBytes()))
        );
    }
}
```

- Spring Boot의 ErrorWebFluxAutoConfiguration을 통해 등록된 DefaultErrorWebExceptionHandler Bean의 우선순위보다 높은 우선순위인 -2를 지정합니다.