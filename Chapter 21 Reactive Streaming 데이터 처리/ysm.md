# Chapter 21 Reactive Streaming 데이터 처리

Spring WebFlux는 SSE(Server-Sent Events)를 이용해 데이터를 Streaming할 수 있다. 

SSE는 Spring 4.2 버전부터 지원되었으며, Spring 5 버전부터 Reactor의 Publisher 타 입인 FluX를 이용해 조금 더 편리한 방법으로 SSE를 사용할 수 있게 되었다.

* SSE(Server-Sent Events)는 클라이언트가 HTTP 연결을 통해 서버로부터 전송되는 업데 이트 데이터를 지속적으로 수신할 수 있는 단방향 서버 푸시 기술이다.
* SSE는 주로 클라이언트 측에서 서버로부터 전송되는 이벤트 스트림을 자동으로 수신하기 위해 사용된다.
*  Chunked Transfer-Encoding 기반
*  chunk 단위로 여러 줄로 구성된 문자열을 전달
*  new line으로 이벤트를 구분
*  문자열은 일반적으로 `<field>: <value>` 형태로 구성

```
id:0
event:add
:comment-i
data:data-0

id:1
event:add
:comment-i
data:data-1

id:2
event:add
:comment-i
data:data-2

id:3
event:add
:comment-i
data:data-3

id:4
event:add
:comment-i
data:data-4
```

* id: 이벤트의 id를 가리킨다
  *  `client에서는 이벤트의 id를 저장하고`
  *  `Last-Event-ID 헤더에 첨부하여 가장 마지막으로받은 이벤트가 무엇인지 전달`
  *  `이를 이용해서 서버는 lastEventId 보다 큰 이벤트만 전달 가능`
* event: 이벤트의 타입을 표현
* data: 이벤트의 data를 표현
  *  여러 줄의 data 필드를 이용하면 multi line data를 표현 가능
* retry: reconnection을 위한 대기 시간을 클라이언트에게 전달
* comment: field 부분이 빈 케이스. 기능을 한다기보다는 정보를 남기기 위한 역할

Return value

- ﻿﻿Servlet stack의 return value와 거의 비슷
- ﻿﻿하지만 ModelAndView 대신 Rendering을 지원
- ﻿﻿void를 반환할때 필요로 하는 필요로 하는 argument의 차이
- ﻿﻿Flux<ServerSentEvent>, Observable<ServerSentEvent> 를 지원 
- ﻿﻿Servlet stack에서는 HttpMessageConverter를 사용하지만  reactive stack0에서는 HttpMessageWriter



서버-발송 이벤트(SSE)에서 `id` 값은 여러 용도로 사용될 수 있다. 

다음은 SSE의 `id` 값을 사용하는 몇 가지 주요 사례이다. 

1. **이벤트 재연결 및 복원**:
   - 클라이언트가 연결을 끊었다가 다시 연결할 때, 마지막으로 받은 이벤트의 `id` 값을 서버에 보내어 누락된 이벤트를 복원할 수 있습니다.
   - 클라이언트는 `Last-Event-ID` 헤더를 사용하여 서버에 마지막으로 받은 이벤트의 `id` 값을 전달할 수 있습니다.
2. **이벤트 순서 관리**:
   - 이벤트의 순서를 보장하거나 추적할 수 있습니다. `id` 값을 사용하면 클라이언트가 이벤트의 순서를 유지하거나 특정 이벤트를 건너뛰지 않았는지 확인할 수 있습니다.
3. **이벤트 로그 및 디버깅**:
   - 서버 측에서 발생한 이벤트의 `id` 값을 로그에 기록하여 디버깅 및 모니터링에 사용할 수 있습니다.
   - 이벤트 ID를 사용하여 특정 이벤트를 쉽게 추적하고 문제를 진단할 수 있습니다.
4. **이벤트 필터링 및 선택**:
   - 클라이언트가 특정 `id` 값 이후의 이벤트만 필터링하여 처리할 수 있습니다. 이를 통해 클라이언트는 특정 이벤트만 선택적으로 수신하고 처리할 수 있습니다.



ServerSentEvent는 ServerSentEventHttpMessageWriter가 처리한다.

* ServerSentEventHttpMessageWrite r는 Flux에 주어진 type에 따라서 ServerSentEvent라면 그대로 사용하고 ﻿﻿아니라면 data에 집어넣는다

## ServerSentEventController 구현

```java
@Controller
@RequestMapping("/sse")
public class SseController {

    @ResponseBody
    @GetMapping(path = "/simple", produces = "text/event-stream")
    public Flux<String> simpleSse() {
        return Flux.interval(Duration.ofMillis(100))
                   .map(i -> "Hello " + i);
    }
}
```

- ﻿@ResponseBody를 추가
- ﻿GetMapping에서 produces를 통해서  response | Content-TypeO| "text/event-stream” 임을 명시
- ﻿Flux.interval을 이용해서 100ms 간격으로 하 나씩 증가된 값을 무한히 만드는 Flux 생성
- ﻿ServerSentEvent 객체를 반환한게 아니기 때 문에 ﻿id가 없고 event 또한 default 값인  "message”로 전달된다



## ServerSentEvent객체

```java
public final class ServerSentEvent<T> {

    @Nullable
    private final String id; // 이벤트의 고유 식별자

    @Nullable
    private final String event; // 이벤트의 타입

    @Nullable
    private final Duration retry; // 클라이언트가 이벤트 스트림 연결을 재시도하기 전에 기다려야 하는 시간

    @Nullable
    private final String comment; // 이벤트에 대한 설명 또는 주석

    @Nullable
    private final T data; // 이벤트의 실제 데이터

    public interface Builder<T> {

        Builder<T> id(String id);

        Builder<T> event(String event);

        Builder<T> retry(Duration retry);

        Builder<T> comment(String comment);

        Builder<T> data(@Nullable T data);

        ServerSentEvent<T> build();
    }
}
```



## db에서 모든 book을 읽어 Streaming하는 예시

```java
public Flux<Book> streamingBooks() {
    return template
            .select(Book.class)
            .all()
            .delayElements(Duration.ofSeconds(2L));
}

@Configuration
public class BookRouter {

  	@Bean
    public RouterFunction<?> routeStreamingBook(BookService bookService,
                                                BookMapper mapper) {
        return route(RequestPredicates.GET("/v11/streaming-books"),
                request -> ServerResponse
                        .ok()
                        .contentType(MediaType.TEXT_EVENT_STREAM)
                        .body(bookService
                                        .streamingBooks()
                                        .map(book -> mapper.bookToResponse(book))
                                ,
                                BookDto.Response.class));
    }
    
}

```

* sse를 사용하려면 Content-Type이 text/event-stream 이여야 한다.
* response는 Flux<Book>으로 지정한다.

다음처럼 요청하면 데이터를 계속 보내게 된다.

```java
@Slf4j
@Configuration
public class BookWebClient {
    @Bean
    public ApplicationRunner streamingBooks() {
        return (ApplicationArguments arguments) -> {
            WebClient webClient = WebClient.create("http://localhost:8080");
            Flux<BookDto.Response> response =
                    webClient
                            .get()
                            .uri("http://localhost:8080/v11/streaming-books")
                            .retrieve()
                            .bodyToFlux(BookDto.Response.class);

            response.subscribe(book -> {
                        log.info("bookId: {}", book.getBookId());
                        log.info("titleKorean: {}", book.getTitleKorean());
                        log.info("titleEnglish: {}", book.getTitleEnglish());
                        log.info("description: {}", book.getDescription());
                        log.info("author: {}", book.getAuthor());
                        log.info("isbn: {}", book.getIsbn());
                        log.info("publishDate: {}", book.getPublishDate());
                        log.info("=======================================");
                    },
                    error -> log.error("# error happened: ", error));
        };
    }
}

```

Javascript 기반의 웹 앱에서 수신하기 위해서는 Javascriot APl인 EventSource 객체를 이용하면 된다.

Javascript 기반의 웹 앱에서 Stream Event를 수신하는 방법에 대해 더 알고 싶다면 아래 링크를 참고

https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events

# WebSocket

- ﻿﻿0SI 7계층에 위치하는 프로토콜
    - ﻿﻿4계층의 TCP에 의존
- ﻿﻿하나의 TCP 연결로 실시간 양방향 통신 지원
- ﻿﻿HTTP streaming에서 서버에서 클라이언트로 만 이벤트를 전달했지만 ﻿﻿WebSocket은 양방향 통신이 가능
- ﻿﻿HTTP와 달리 지속적인 연결을 유지하면 오버 헤드가 적다

## WebSocket 요청과 응답

- ﻿﻿요청에서 Upgrade: websocket과  Connection: Upgrade 헤더를 추가
- ﻿﻿응답으로 status 101과 함께 Upgrade: websocket, Connection: Upgrade 헤더를 제공

```
GET /websocket HTTP/1.1
Upgrade: websocket
Connection: Upgrade

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
```

## WebSocketHandler

- ﻿﻿handle을 통해서 WebSocketSession을 받 고 Mono<Void> 를 반환한다
- ﻿﻿인자로 받는 부분만 제외하면 WebHandler와 비슷

```java
public interface WebSocketHandler {
	Mono<Void> handle (WebSocketSession session):
}
```

## WebSocket CloseStatus

- ﻿﻿1000: NORMAL. 정상적으로 webSocket이 종료
- ﻿﻿1001: GOING_AWAY. 서버가 예상치 못하게 종료되 거나 페이지에서 벗어난 경우
- ﻿﻿1002: PROTOCOL_ERROR. 프로토콜에 문제가 있 는 경우
- ﻿﻿1003: NOT_ACCEPTABLE. accept할 수 없는 데이 터를 받은 경우
- ﻿﻿1009: TOO_BIG_TO_PROCESS. 처리하기에 너무 큰 데이터를 받은 경우
- ﻿﻿1011: SERVER_ERROR. 예상하지 못한 에러로 서버 에서 요청을 처리하지 못하는 경우
- ﻿﻿1012: SERVICE_RESTARTED. 서비스가 재시작. 클 라이언트는 5~30초 사이로 랜덤하게 접근

## WebSocketMessage

```java
public class WebSocketMessage {

	private static final boolean reactorNetty2Present = ClassUtils.isPresent(
			"io.netty5.handler.codec.http.websocketx.WebSocketFrame", WebSocketMessage.class.getClassLoader());


	private final Type type;

	private final DataBuffer payload;

	@Nullable
	private final Object nativeMessage;

  public enum Type {
		TEXT, BINARY, PING. PONG
  }
}
```

- ﻿﻿type: WebSocketMessage 타입. TEXT, BINARY, PING, PONG 중 하나
- ﻿﻿payload: socketMessage의 값
- ﻿﻿getPayloadAsText: payload를 string 형태로 제공



## 핸들러 구현 및 웹소켓 등록

```java
@RequiredArgsConstructor
@Component
public class ChatWebSocketHandler implements WebSocketHandler {
    private final ChatService chatService;

    @Override
    public Mono<Void> handle(WebSocketSession session) {
        String iam = (String) session.getAttributes().get("iam");

        Flux<Chat> chatFlux = chatService.register(iam);
        chatService.sendChat(iam,
                new Chat(iam + "님 채팅방에 오신 것을 환영합니다", "system"));

        session.receive()
                .doOnNext(webSocketMessage -> {
                    String payload = webSocketMessage.getPayloadAsText();

                    String[] splits = payload.split(":");
                    String to = splits[0].trim();
                    String message = splits[1].trim();

                    boolean result = chatService.sendChat(to, new Chat(message, iam));
                    if (!result) {
                        chatService.sendChat(iam, new Chat("대화 상대가 없습니다", "system"));
                    }
                }).subscribe();

        return session.send(chatFlux
                .map(chat -> session.textMessage(chat.getFrom() + ": " + chat.getMessage()))
        );
    }
}

// 등록 
@Configuration
public class MappingConfig {
    @Bean
    SimpleUrlHandlerMapping simpleUrlHandlerMapping(
            ChatWebSocketHandler chatWebSocketHandler
    ) {
        Map<String, WebSocketHandler> urlMapper = Map.of(
                "/chat", chatWebSocketHandler
        );

        SimpleUrlHandlerMapping mapping = new SimpleUrlHandlerMapping();
        mapping.setOrder(1);
        mapping.setUrlMap(urlMapper);

        return mapping;
    }

}
```



