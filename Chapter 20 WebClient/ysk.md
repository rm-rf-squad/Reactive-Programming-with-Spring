# Chapter 20 WebClient

##  Webclient란

webclient는 spring5부터 지원하는 non-blocking client로써, 블로킹, 논블로킹 둘다 지원한다.

restTemplete을 대체하기도하지만 이제는 그럴 필요가 없다. 스프링부트 3.2부터 RestClient가 있기 때문이다.

## Builder
```java
public interface Builder {
    Builder baseUrl(String baseUrl);

    Builder defaultHeaders(Consumer<HttpHeaders> headersConsumer);

    Builder defaultCookies(Consumer<MultiValueMap<String, String>> cookiesConsumer);

    Builder defaultRequest(Consumer<WebClient.RequestHeadersSpec<?>> defaultRequest);

    Builder filters(Consumer<List<ExchangeFilterFunction>> filtersConsumer);

    Builder clientConnector(ClientHttpConnector connector);

    Builder codecs(Consumer<ClientCodecConfigurer> configurer);

    Builder exchangeFunction(ExchangeFunction exchangeFunction);

    WebClient build();
}

```
### 메서드 설명

- **baseUrl(String baseUrl)**: `WebClient`의 기본 URL을 설정합니다. 모든 요청 경로에 이 URL이 접두사로 추가됩니다.
- **defaultHeaders(Consumer<HttpHeaders> headersConsumer)**: 기본으로 제공되는 헤더를 설정합니다.
- **defaultCookies(Consumer<MultiValueMap<String, String>> cookiesConsumer)**: 기본으로 제공되는 쿠키를 설정합니다.
- **defaultRequest(Consumer<WebClient.RequestHeadersSpec<?>> defaultRequest)**: 기본 요청 설정을 지정합니다.
- **filters(Consumer<List<ExchangeFilterFunction>> filtersConsumer)**: 요청을 보내기 전후에 동작하는 필터를 설정합니다.
- **clientConnector(ClientHttpConnector connector)**: `WebClient`에서 사용되는 `clientConnector`를 변경합니다.
- **codecs(Consumer<ClientCodecConfigurer> configurer)**: 클라이언트의 코덱 설정을 지정합니다.
- **exchangeFunction(ExchangeFunction exchangeFunction)**: HTTP 요청을 수행하는 `exchangeFunction`을 변경합니다.
- **build()**: 설정된 `WebClient`를 생성합니다.



### WebClient.Builder 인터페이스 정리

`WebClient.Builder` 인터페이스는 Spring WebFlux의 `WebClient`를 생성하면서 필요한 설정을 제공하는 빌더 패턴입니다. 이를 통해 `WebClient`의 기본 설정을 간편하게 구성할 수 있습니다.

```
public interface Builder {
    Builder baseUrl(String baseUrl);

    Builder defaultHeaders(Consumer<HttpHeaders> headersConsumer);

    Builder defaultCookies(Consumer<MultiValueMap<String, String>> cookiesConsumer);

    Builder defaultRequest(Consumer<WebClient.RequestHeadersSpec<?>> defaultRequest);

    Builder filters(Consumer<List<ExchangeFilterFunction>> filtersConsumer);

    Builder clientConnector(ClientHttpConnector connector);

    Builder codecs(Consumer<ClientCodecConfigurer> configurer);

    Builder exchangeFunction(ExchangeFunction exchangeFunction);

    WebClient build();
}
```

### 메서드 설명

- **baseUrl(String baseUrl)**: `WebClient`의 기본 URL을 설정합니다. 모든 요청 경로에 이 URL이 접두사로 추가됩니다.
- **defaultHeaders(Consumer<HttpHeaders> headersConsumer)**: 기본으로 제공되는 헤더를 설정합니다.
- **defaultCookies(Consumer<MultiValueMap<String, String>> cookiesConsumer)**: 기본으로 제공되는 쿠키를 설정합니다.
- **defaultRequest(Consumer<WebClient.RequestHeadersSpec<?>> defaultRequest)**: 기본 요청 설정을 지정합니다.
- **filters(Consumer<List<ExchangeFilterFunction>> filtersConsumer)**: 요청을 보내기 전후에 동작하는 필터를 설정합니다.
- **clientConnector(ClientHttpConnector connector)**: `WebClient`에서 사용되는 `clientConnector`를 변경합니다.
- **codecs(Consumer<ClientCodecConfigurer> configurer)**: 클라이언트의 코덱 설정을 지정합니다.
- **exchangeFunction(ExchangeFunction exchangeFunction)**: HTTP 요청을 수행하는 `exchangeFunction`을 변경합니다.
- **build()**: 설정된 `WebClient`를 생성합니다.

## Webclient 생성 예제
#### WebClient 생성 예제

1. **기본 WebClient 생성**:

   ```java
   WebClient webClient = WebClient.create();
   ```

2. **Builder를 이용한 WebClient 생성**:

   ```java
   WebClient.Builder webClientBuilder = WebClient.builder();
   WebClient webClient = webClientBuilder
       .baseUrl("https://fastcampus.co.kr")
       .defaultHeaders(headers -> headers.add("Authorization", "Bearer token"))
       .defaultCookies(cookies -> cookies.add("myCookie", "cookieValue"))
       .filters(filters -> filters.add((request, next) -> {
           // 필터 로직 추가
           return next.exchange(request);
       }))
       .build();
   ```

3. **baseUrl을 제공하여 WebClient 생성**:

   ```java
   WebClient webClientWithUrl = WebClient.create("https://fastcampus.co.kr");
   ```



## WebClient method

```java
nterface RequestHeadersUriSpec<T extends RequestHeadersUriSpec<T>> {
    T get();
    T head();
    RequestBodyUriSpec post();
    RequestBodyUriSpec put();
    RequestBodyUriSpec patch();
    T delete();
    T options();
    RequestBodyUriSpec method(HttpMethod method);
}
```

- ﻿﻿get, head, post, put, patch, delete, options, method 제공
- ﻿﻿post, put, patch의 경우 RequestBodyUriSpec를 나머지는 RequestHeadersUriSpeC를 반환

### Response Body Spec

```java
interface RequestBodySpec extends RequestHeadersSpec<RequestBodySpec> {
    RequestBodySpec contentLength(long contentLength);
    RequestBodySpec contentType(MediaType contentType);
    RequestHeadersSpec<?> bodyValue(Object body);
    <T, P extends Publisher<T>> RequestHeadersSpec<?> body(P publisher, Class<T> elementClass);
    <T, P extends Publisher<T>> RequestHeadersSpec<?> body(P publisher, ParameterizedTypeReference<T> elementTypeRef);
    RequestHeadersSpec<?> body(BodyInserter<?, ? super ClientHttpRequest> inserter);
}
```

- ﻿﻿request의 body를 설정할 수 있는 메소드 제 공
- ﻿﻿body 호출시 RequestHeaderSpec 반환
- ﻿﻿bodyValue: 객체를 받아서 내부적으로 codec을 이용하여 변환
- ﻿﻿Publisher를 이용해서 변환하거나 ﻿﻿Bodyinserter를 이용해서 변환 가능

### RequestHeaderSpec

```java
interface RequestHeadersSpec<S extends RequestHeadersSpec<S>> {
    S accept(MediaType... acceptableMediaTypes);
    S acceptCharset(Charset... acceptableCharsets);
    S cookie(String name, String value);
    S cookies(Consumer<MultiValueMap<String, String>> cookiesConsumer);
    S header(String headerName, String... headerValues);
    S headers(Consumer<HttpHeaders> headersConsumer);
    S attribute(String name, Object value);
    S attributes(Consumer<Map<String, Object>> attributesConsumer);
    S httpRequest(Consumer<ClientHttpRequest> requestConsumer);
  
    ResponseSpec retrieve ();
   <V> Mono<V> exchangeToMono(Function<ClientResponse,? extends Mono<V> responseHandler);

   <V> Flux<V> exchangeToFlux(Function<ClientResponse, ? extends Flux<V>> responseHandler);
}
```

- ﻿﻿accept, acceptCharset: Accept 헤더 변경
- ﻿﻿cookie, cookies: 요청의 쿠키 변경
- ﻿﻿header, headers: 요청의 header 변경
- ﻿﻿attribute: webClient의 filter들에 제공되는 attribute 설정
- ﻿﻿httpRequest: HTTP 라이브러리의 요청에 직접 접근할 수 있게 callback 제공

- ﻿﻿retrieve, exchangeToMono, exchange ToFlux를 실행하면 요청을 서버에 전달하고 응답을 받는다
- ﻿﻿retrieve: ResponseSpec을 반환
- ﻿﻿exchangeToMono: ClientResponse 인자로 받고 body를 Mono 형태로 반환하는 callback
- ﻿﻿exchange ToFlux: ClientResponse를 인자 로 받고 body를 Flux 형태로 반환하는 반환하는 callback

### URI Spec

```java
interface UriSpec<S extends RequestHeadersSpec<S>> {
    S uri(URI uri);
    S uri(String uri, Object... uriVariables);
    S uri(String uri, Map<String, ?> uriVariables);
    S uri(String uri, Function<UriBuilder, URI> uriFunction);
    S uri(Function<UriBuilder, URI> uriFunction);
}
```



- **uri(URI uri)**: URI를 설정
- **uri(String uri, Object... uriVariables)**: 문자열 URI와 가변 인자를 사용하여 URI를 설정
- **uri(String uri, Map<String, ?> uriVariables)**: 문자열 URI와 맵을 사용하여 URI를 설정
- **uri(String uri, Function<UriBuilder, URI> uriFunction)**: URI 빌더와 함수형 인터페이스를 사용하여 URI를 설정
- **uri(Function<UriBuilder, URI> uriFunction)**: URI 빌더와 함수형 인터페이스를 사용하여 URI를 설정

### Response Sepc

```java
interface ResponseSpec {
    ResponseSpec onStatus(Predicate<HttpStatus> statusPredicate,
                          Function<ClientResponse, Mono<? extends Throwable>> exceptionFunction);

    <T> Mono<T> bodyToMono(Class<T> elementClass);

    <T> Flux<T> bodyToFlux(Class<T> elementClass);

    <T> Mono<ResponseEntity<T>> toEntity(Class<T> bodyClass);

    <T> Mono<ResponseEntity<List<T>>> toEntityList(Class<T> elementClass);

    <T> Mono<ResponseEntity<Flux<T>>> toEntityFlux(Class<T> elementType);

    Mono<ResponseEntity<Void>> toBodilessEntity();
}
```

### 설명

- **onStatus**: 응답 상태를 확인하고 문제가 발생한 경우 `Mono.error`를 반환하여 에러를 전파
- **bodyToMono**: 응답을 `Mono`로 변환
- **bodyToFlux**: 응답을 `Flux`로 변환
- **toEntity**: 응답을 `ResponseEntity`로 변환
- **toEntityList**: 응답을 `ResponseEntity<List<T>>`로 변환
- **toEntityFlux**: 응답을 `ResponseEntity<Flux<T>>`로 변환
- **toBodilessEntity**: 응답을 `ResponseEntity<Void>`로 변환하고 본문은 포함하지 않는다.

이 인터페이스는 응답을 처리하고 다양한 형태로 변환할 수 있는 메서드를 정의한다

. `onStatus` 메서드는 응답 상태를 확인하고, 조건에 따라 예외를 발생시킨다. 

`bodyToMono` 및 `bodyToFlux` 메서드는 응답 본문을 각각 `Mono`와 `Flux`로 변환하며 `toEntity` 메서드들은 응답을 `ResponseEntity`로 변환하여 다양한 형태의 데이터를 포함할 수 있다.



## WebClient ConnectionTimeout 설정

커넥션 타임아웃은 클라이언트가 서버에 연결을 시도할 때, 연결이 성공할 때까지 대기하는 최대 시간을 정의하는것이다.

클라이언트가 서버에 연결을 시도한 후, 지정된 시간 내에 연결이 이루어지지 않으면 타임아웃 오류가 발생한다.

 클라이언트가 너무 오랫동안 서버에 연결하려고 기다리지 않도록 하여, 자원 낭비를 방지하는 목적으로 설정한다.

특정 서버 엔진의 Client Connector 설정을 통해 ConnectionTimeout 설정을 할 수 있다.

* 다양한 서버 엔진을 지원한다. 

```java
HttpClient httpClient =
        HttpClient
                .create()
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 500)
                .responseTimeout(Duration.ofMillis(500))
                .doOnConnected(connection ->
                    connection
                            .addHandlerLast(
                                    new ReadTimeoutHandler(500,
                                                        TimeUnit.MILLISECONDS))
                            .addHandlerLast(
                                    new WriteTimeoutHandler(500,
                                                        TimeUnit.MILLISECONDS)));
Flux<BookDto.Response> response =
        WebClient
                .builder()
                .baseUrl("http://localhost:8080")
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .build()
                .get()
                .uri(uriBuilder -> uriBuilder
                        .path("/v10/books")
                        .queryParam("page", "1")
                        .queryParam("size", "10")
                        .build())
                .retrieve()
                .bodyToFlux(BookDto.Response.class);
response
        .map(book -> book.getTitleKorean())
        .subscribe(bookName -> log.info("book name2: {}", bookName));
```

* **읽기/쓰기 시간 초과 핸들러 추가**: `.doOnConnected(connection -> ...)`를 사용하여 연결된 후 읽기와 쓰기 시간 초과 핸들러를 추가했다. 
* 500ms 가 초과되면 TimeoutExceptiondㅣ 발생한다. 



번외 - 주요 서버엔진은 다음과같은 종류가 있다.



**Reactor Netty**:

- 기본적으로 Spring WebFlux에서 사용하는 서버 엔진
- `reactor.netty.http.client.HttpClient`를 사용하여 설정

```java
HttpClient httpClient = HttpClient.create()
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
    .responseTimeout(Duration.ofSeconds(5));

WebClient webClient = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

**Apache HttpClient**:

- Apache의 HttpClient를 사용하여 설정
- `org.apache.http.impl.nio.client.HttpAsyncClients`를 사용하여 설정

```java
CloseableHttpAsyncClient client = HttpAsyncClients.custom()
    .setDefaultRequestConfig(RequestConfig.custom()
        .setConnectTimeout(5000)
        .setSocketTimeout(5000)
        .build())
    .build();

client.start();

WebClient webClient = WebClient.builder()
    .clientConnector(new HttpComponentsClientHttpConnector(client))
    .build();
```



**Jetty**:

- Jetty의 HttpClient를 사용할 수 있다.
- `org.eclipse.jetty.client.HttpClient`를 사용하여 설정

```java
HttpClient httpClient = new HttpClient();
httpClient.setConnectTimeout(5000);

WebClient webClient = WebClient.builder()
    .clientConnector(new JettyClientHttpConnector(httpClient))
    .build();
```



**OkHttp**:

- OkHttp는 Square에서 개발한 자바 기반의 HTTP 클라이언트로, 간단하고 효율적인 API를 제공.
- `okhttp3.OkHttpClient`를 사용하여 ConnectionTimeout을 설정할 수 있다.

```java
OkHttpClient okHttpClient = new OkHttpClient.Builder()
    .connectTimeout(5, TimeUnit.SECONDS)
    .build();

WebClient webClient = WebClient.builder()
    .clientConnector(new OkHttp3ClientHttpConnector(okHttpClient))
    .build();
```

## exchangeToMono를 사용한 응답 디코딩

`retrieve` 메서드는 `WebClient`에서 HTTP 요청을 보내고 응답 본문을 간단하게 추출하고 처리하는 메서드이다.

retrieve() 대신에 exchangeToMono()나 exchange Toflux() 메서드를 이용 하면 response를 사용자의 요구 조건에 맞게 제어할 수 있다.

```java
BookDto.Post post = new BookDto.Post("Java 중급",
        "Intermediate Java",
        "Java 중급 프로그래밍 마스터",
        "Kevin1", "333-33-3333-333-3",
        "2022-03-22");

WebClient webClient = WebClient.create();

webClient.post()
        .uri("http://localhost:8080/v10/books")
        .bodyValue(post)
        .exchangeToMono(response -> {
            if(response.statusCode().equals(HttpStatus.CREATED))
                return response.toEntity(Void.class);
            else
                return response
                        .createException()
                        .flatMap(throwable -> Mono.error(throwable));
        })
        .subscribe(res -> {
            log.info("response status2: {}", res.getStatusCode());
            log.info("Header Location2: {}", res.getHeaders().get("Location"));
            },
            error -> log.error("Error happened: ", error));
```

* status가 201 CREATED면 ResponseEntity를 리턴, 이외에는 exception을 Mono.error로 던진다.

* ClientResponse의 createException() 메서드는 request/response 정보를 포 함한 WebClientResponseException을 생성한다.