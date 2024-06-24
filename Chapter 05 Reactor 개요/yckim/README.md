# Reactor 개요

## Reactor란?

Reactor는 Spring Framework 팀의 주도하에 개발된 리액티브 스트림즈의 구현체입니다.

Spring Framework 5 버전 부터 리액티브 스택에 포함되어 Spring WebFlux 기반의 리액티브 애플리케이션을 제작하기 위한 핵심 역할을 담당합니다.

### Reactor의 특징

1. Reactive Streams
    - 리액티브 스트림즈 사양을 구현한 리액티브 라이브러리입니다.
2. Non-Blocking
    - Reactor는 JVM 위에서 실행되는 Non-Blocking 애플리케이션을 제작하기 위해 필요한 핵심 기술입니다.
3. Java’s Function API
    - Reactor에서 Publisher와 Subscriber 간의 상호 작용은 Java의 함수형 프로그래밍 API를 통해서 이루어집니다.
4. Flux[N]
    - Flux[N]은 N개의 데이터를 emit 한다는 의미입니다.
    - 다시 말해, Flux는 0부터 N개, 즉 무한개의 데이터를 emit할 수 있는 Reactor의 Publisher 입니다.
5. Mono[0|1]
    - Mono는 데이터를 한 건도 emit 하지않거나 단 한건만 emit하는 단발성 데이터 emit에 특화된 Publisher입니다.
6. Well-suited for microservices
    - Reactor는 Non-Blocking 특징을 가지기 때문에 마이크로 서비스 기반 시스템에서 지속적으로 발생하는 I/O를 처리하기에 적합한 기술입니다.
7. Backpressure-ready network
    - Reactor는 Publisher로 부터전달받은 데이터를 처리하는 데 있어 과부하가 걸리지 않도록 제어하는 Backpressure를 지원합니다.

## Hello Reactor 코드로 보는 Reactor 구성요소

### Reactor로 Hello World 출력

```java
public class HelloWorld {
    public static void main(String[] args) {
        Flux<String> sequence = Flux.just("Hello", "Reactor");
        sequence.map(String::toLowerCase)
                .subscribe(System.out::println);
    }
}

```

```java
Flux<String> sequence = Flux.just("Hello", "Reactor");
```

Flux는 Reactor에서 Publisher의 역할을 합니다.

위의 코드에서는 just 메서드의 파라미터로 전달한 “Hello”, “Reactor”가 입력으로 들어오는 데이터가 됩니다.

이 데이터는 Publisher가 최초로 제공하는 가공되지 않은 데이터로 **데이터 소스**라고 불립니다.

<aside>
💡 데이터 소스의 개수가 두개 이상이기 때문에 Mono가 아닌 Flux를 사용해야 합니다.

</aside>

```java
sequence.map(String::toLowerCase)
```

예제에서 위과 같이 문자열을 소문자로 변환하는 연산을 하여 데이터 소스를 가공합니다.



```java
.subscribe(System.out::println);
```

예제에서 Subscriber의 역할을 하는 코드는 위와 같습니다.