# WebClient

## WebClient란?

WebClient는 Spring 5부터 지원하는 Non-Blocking HTTP request를 위한 리액티브 웹 클라이언트로서 함수형 기반의 향상된 API를 제공합니다.

WebClient는 내부적으로 HTTP 클라이언트 라이브러리에게 HTTP request를 위임하며, 기본 HTTP 클라이언트 라이브러리는 Reactor Netty 입니다.

## WebClient 사용 예시

```java
    public Mono<String> createExample(Object requestData) {
        return webClient.post() // 메서드 지정
                        .uri("/endpoint") // 요청을 날릴 URI 지정
                        .bodyValue(requestData) // 요청 본문 지정
                        .retrieve() // 응답 수신 -> 응답 상태 확인 -> 응답 처리 진행
                        .bodyToMono(String.class); // 응답을 어떤 형태로 변환할지 지정
    }
```

## WebClient Connection Timeout 설정

```java
HttpClient httpClient = HttpClient.create()
                            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000) // 연결 타임아웃 설정 (5000ms = 5초)
                            .doOnConnected(conn -> 
                                conn.addHandlerLast(new ReadTimeoutHandler(10))
                                    .addHandlerLast(new WriteTimeoutHandler(10))
                            ) // 연결 후 읽기 및 쓰기 타임아웃 설정
                            .responseTimeout(Duration.ofSeconds(10)); // 전체 응답 타임아웃 설정 (10초)

return WebClient.builder()
          .clientConnector(new ReactorClientHttpConnector(httpClient))
          .baseUrl("https://api.example.com")
          .build();
```

- option()
    - Connection 설정을 위한 Timeout 시간을 설정
- responseTimeout()
    - 응답을 수신하기까지의 timeout 시간을 설정
- doOnConnected()
    - 파라미터로 전달되는 람다 표현식을 통해 Connection이 연결된 이후에 수행할 동작을 정의
    - 여기서는 특정 시간동안 읽을 수 있는 데이터가 없을 경우 ReadTimeoutException을 발생시키는 ReadTimeoutHandler와 특정 시간동안 쓰기 작업을 종료할 수 없을 경우 WriteTimeoutException을 발생시키는 WriteTimeoutHandler를 등록합니다.
- clientConnector()
    - HTTP Client Connector를 설정합니다.

## exchangeToMono()를 사용한 응답 디코딩

retrive() 대신에 exchangeToMon()나 exchangeToFlux() 메서드를 이용하면 response를 사용자의 요구조건에 맞게 제어할 수 있습니다.

```java
WebClient webClient = WebClient.builder()
        .baseUrl("https://api.example.com")  // 요청할 API의 base URL
        .build();

// GET 요청을 보내고 응답을 MyResponseDTO 클래스로 디코딩
Mono<MyResponseDTO> responseMono = webClient.get()
        .uri("/endpoint")  // 요청할 API의 endpoint
        .exchangeToMono(clientResponse -> {
            // 상태 코드 확인 후 적절히 처리
            if (clientResponse.statusCode().is2xxSuccessful()) {
                return clientResponse.bodyToMono(MyResponseDTO.class);  // 성공 시, 응답을 MyResponseDTO로 디코딩
            } else {
                return clientResponse.createException()
                        .flatMap(Mono::error);  // 오류 시, 예외 발생
            }
        });

// Mono 구독 (응답 처리)
responseMono.subscribe(
        response -> System.out.println("성공: " + response),  // 성공 시 처리 로직
        error -> System.err.println("실패: " + error.getMessage())  // 실패 시 처리 로직
);
```