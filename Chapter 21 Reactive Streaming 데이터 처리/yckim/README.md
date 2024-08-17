# Reactive Streaming 데이터 처리

Spring WebFlux는 SSE를 이용해 데이터를 Streaming할 수 있습니다.

SSE는 Spring 4.2 부터 지원되었으며, Spring 5 버전부터 Reactor의 Publisher 타입인 Flux를 이용해 조금 더 편리한 방법으로 SSE를 사용할 수 있게 되었습니다.

### SSE란?

SSE는 클라이언트가 HTTP 연결을 통해 서버로부터 전송되는 업데이트 데이터를 지속적으로 수신할 수 있는 단방향 서버 푸시 기술입니다.

SSE는 주로 클라이언트 측에서 서버로부터 전송되는 이벤트 스트림을 자동으로 수신하기 위해 사용됩니다.

## Streaming 데이터 처리 예시

```java
public Flux<Book> streamingBooks() {
	return template
		.select(Book.class)
		.all()
		.delayElements(Duration.ofSeconds(2L))'
}
```

- Spring WebFlux에서 Streaming 방식으로 데이터를 전송하기 위한 response body의 타입은 Flux 입니다.
- R2dbcEntityTemplate의 all() 메서드를 이용해 별도의 페이지네이션 없이 데이터베이스에 저장된 모든 도서 정보를 조회합니다.
- delayElements() Operator를 이용해 Time 기반의 이벤트가 발생하도록 했습니다. 예제 코드에서는 2초에 한번씩 데이터를 emit 합니다.

```java
@Bean
public RouterFunction<?> routeStreamingBook(BookService bookService, BookMapper mapper) {
	return route(RequestPredicateds.GET("/v11/streming-books"),
		request -> ServerResponse.ok()
			.contentType(MediaType.TEXT_EVENT_STREAM)
			.body(bookService.streamingBooks()
				.map(book -> mapper.bookToResponse(book)), BookDto.Response.class));
}
```

- SSE를 이용해 이벤트 스트림을 전송하기 위해서는 Content-Type이 “text/event-stream”이어야 합니다.

### ReactiveStreaming 데이터 수신을 위한 클라이언트

```java
Flux<BookDto.Response> response = webClient.get()
	.uri("http://localhost:8080/v11/streaming-books")
		.retrieve()
		.bodyToFlux(BookDto.Response.class);

response.subscribe(book -> {
	log.info("bookId: {}", book.getBookId());
},
error -> log.error("# error happend : ", error));
```

- WebClient를 이용해 Streaming 데이터 수신하기 위해서는 bodyToFlux 메서드를 사용합니다.