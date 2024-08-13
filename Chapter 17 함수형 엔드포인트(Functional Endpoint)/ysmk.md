# Chapter 17 함수형 엔드포인트(Functional Endpoint)

## HandlerFunction을 사용한 Request 처리

Spring WelFlux의 함수형 엔드포인트는 들어오는 요청을 처리하기 위해

HandlerFunction이라는 함수형 기반의 핸들러를 사용한다.

```java
@FunctionalInterface
public interface HandlerFunction<T extends ServerResponse> {

  Mono<T> handle(ServerRequest request);

}
```
* `HandlerFunction` 인터페이스는 하나의 메서드, `handle`을 가지고 있으며, 이 메서드는 `ServerRequest`를 받아 `Mono<ServerResponse>`를 반환한다. 
* HandlerFunction은 RouterFunction을 통해 요청이 라우팅된 이후에 동작한다 

## Request 라우팅을 위한 RouterFunction

RouterFunction은 @RequestMapping 어노테이션과 동일한 기능을 한다.
RouterFunction 은 URL 경로를 `HandlerFunction`에 매핑하는 역할을 한다. 
기존 Controller를 RouterFunction으로 변환해보자

```java
@Configuration("bookRouterV1")
public class BookRouter {
    @Bean
    public RouterFunction<?> routeBookV1(BookHandler handler) {
        return route()
                .POST("/v1/books", handler::createBook)
                .PATCH("/v1/books/{book-id}", handler::updateBook)
                .GET("/v1/books", handler::getBooks)
                .GET("/v1/books/{book-id}", handler::getBook)
                .build();
    }
}

```

* route() 메서드는 RouterFunctionBuilder 객체를 리턴하며 해당 객체로 각각 HTTP 메소드에 매치되는 request를 처리하기 위한 라우트를 추가한다
* route : SeverRequest를 인자로 받아 HandlerFunction을 Mono로 반환하는 추상메서드. 
* path, method, predicate 등으로 handlerFunction과 연결하여 해당 요청이 들 어왔을때 handlerFunction을 반환

또한 request를 처리하기 위한 핸들러를 구현한다

```java
@Component("bookHandlerV1")
public class BookHandler {
    private final BookMapper mapper;

    public BookHandler(BookMapper mapper) {
        this.mapper = mapper;
    }

    public Mono<ServerResponse> createBook(ServerRequest request) {
        return request.bodyToMono(BookDto.Post.class)
                .map(post -> mapper.bookPostToBook(post))
                .flatMap(book ->
                        ServerResponse
                                .created(URI.create("/v1/books/" + book.getBookId()))
                                .build());
    }

    public Mono<ServerResponse> getBook(ServerRequest request) {
        long bookId = Long.valueOf(request.pathVariable("book-id"));
        Book book =
                new Book(bookId,
                        "Java 고급",
                        "Advanced Java",
                        "Kevin",
                        "111-11-1111-111-1",
                        "Java 중급 프로그래밍 마스터",
                        "2022-03-22",
                        LocalDateTime.now(),
                        LocalDateTime.now());
        return ServerResponse
                            .ok()
                            .bodyValue(mapper.bookToResponse(book))
                            .switchIfEmpty(ServerResponse.notFound().build());
    }

    public Mono<ServerResponse> updateBook(ServerRequest request) {
        final long bookId = Long.valueOf(request.pathVariable("book-id"));
        return request
                .bodyToMono(BookDto.Patch.class)
                .map(patch -> {
                    patch.setBookId(bookId);
                    return mapper.bookPatchToBook(patch);
                })
                .flatMap(book -> ServerResponse.ok()
                        .bodyValue(mapper.bookToResponse(book)));
    }

    public Mono<ServerResponse> getBooks(ServerRequest request) {
        List<Book> books = List.of(
                new Book(1L,
                        "Java 고급",
                        "Advanced Java",
                        "Kevin",
                        "111-11-1111-111-1",
                        "Java 중급 프로그래밍 마스터",
                        "2022-03-22",
                        LocalDateTime.now(),
                        LocalDateTime.now()),
                new Book(2L,
                        "Kotlin 고급",
                        "Advanced Kotlin",
                        "Kevin",
                        "222-22-2222-222-2",
                        "Kotlin 중급 프로그래밍 마스터",
                        "2022-05-22",
                        LocalDateTime.now(),
                        LocalDateTime.now())
        );
        return ServerResponse
                .ok()
                .bodyValue(mapper.booksToResponse(books));
    }
}

```

* NotFound를 반환하려면 ServerResponse.notfound().build() 반환
* ok는 ServerResponse.ok()



## 함수형 엔드포인트에서 request Body 유효성 검증

### Custom Validator

Custom Validator를 지원한다

```java
import org.springframework.stereotype.Component;
import org.springframework.validation.Errors;
import org.springframework.validation.ValidationUtils;
import org.springframework.validation.Validator;

@Component("bookValidatorV2")
public class BookValidator implements Validator {
    @Override
    public boolean supports(Class<?> clazz) {
        return BookDto.Post.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        BookDto.Post post = (BookDto.Post) target;

        ValidationUtils.rejectIfEmptyOrWhitespace(
                errors, "titleKorean", "field.required");

        ValidationUtils.rejectIfEmptyOrWhitespace(
                errors, "titleEnglish", "field.required");
    }
}

```

* Jakarta Validator가 아니다.

검증하려면 핸들러에서 호출한다

```java
@Slf4j
@Component("bookHandlerV2")
public class BookHandler {
    private final BookMapper mapper;
    private final BookValidator validator;

    public BookHandler(BookMapper mapper, BookValidator validator) {
        this.mapper = mapper;
        this.validator = validator;
    }

    public Mono<ServerResponse> createBook(ServerRequest request) {
        return request.bodyToMono(BookDto.Post.class)
                .doOnNext(post -> this.validate(post)) // here 호출한다. 
                .map(post -> mapper.bookPostToBook(post))
                .flatMap(book ->
                        ServerResponse
                                .created(URI.create("/v2/books/" + book.getBookId()))
                                .build());
    }

    private void validate(BookDto.Post post) {
        Errors errors = new BeanPropertyBindingResult(post, BookDto.class.getName());
        validator.validate(post, errors);
        if (errors.hasErrors()) {
            log.error(errors.getAllErrors().toString());
            throw new ServerWebInputException(errors.toString());
        }
    }
}

```



## BeanValidation을 이용한 유효성 검증

전처럼 커스텀을 사용하는 경우는 많지 않을 수 있따. 일단 유효성 검증을 위한 코드가 섞여 코드가 복잡해진다.

Webflux에서도 Bean VAlidation을 지원한다.

Spring WebFlux에서는

 Spring에서 지원하는 Validator 인터페이스와 

javax에 서 지원하는 표준 Validator 인터페이스 중 하나를 사용해 Validator 구현체를 주입받아서 유효성 검증을 진행할 수 있다.

1. Spring Validator 인터페이스 사용

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.validation.BeanPropertyBindingResult;
import org.springframework.validation.Errors;
import org.springframework.validation.Validator;
import org.springframework.web.server.ResponseStatusException;

@Slf4j
@Component("bookValidatorV3")
public class BookValidator<T> {
    private final Validator validator;

    public BookValidator(@Qualifier("springValidator") Validator validator) {
        this.validator = validator;
    }

    public void validate(T body) {
        Errors errors =
                new BeanPropertyBindingResult(body, body.getClass().getName());

        this.validator.validate(body, errors);

        if (!errors.getAllErrors().isEmpty()) {
            onValidationErrors(errors);
        }
    }

    private void onValidationErrors(Errors errors) {
        log.error(errors.getAllErrors().toString());
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST, errors.getAllErrors()
                .toString());
    }
}

```



2. javax(jakarta) Validator 인터페이스 사용

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ResponseStatusException;

import javax.validation.ConstraintViolation;
import javax.validation.Validator;
import java.util.Set;


@Slf4j
@Component("bookValidatorV4")
public class BookValidator<T> {
    private final Validator validator;

    public BookValidator(@Qualifier("javaxValidator") Validator validator) {
        this.validator = validator;
    }

    public void validate(T body) {
        Set<ConstraintViolation<T>> constraintViolations = validator.validate(body);
        if (!constraintViolations.isEmpty()) {
            onValidationErrors(constraintViolations);
        }
    }

    private void onValidationErrors(Set<ConstraintViolation<T>> constraintViolations) {
        log.error(constraintViolations.toString());
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST,
                                            constraintViolations.toString());
    }
}

```

마찬가지로 호출해서 사용하면 된다

```java
@Slf4j
@Component("bookHandlerV4")
public class BookHandler {
    private final BookMapper mapper;
    private final BookValidator validator;

    public BookHandler(BookMapper mapper, BookValidator validator) {
        this.mapper = mapper;
        this.validator = validator;
    }

    public Mono<ServerResponse> createBook(ServerRequest request) {
        return request.bodyToMono(BookDto.Post.class)
                .doOnNext(post -> validator.validate(post))
                .map(post -> mapper.bookPostToBook(post))
                .flatMap(book -> ServerResponse
                        .created(URI.create("/v4/books/" + book.getBookId()))
                        .build());

    }
}
```

