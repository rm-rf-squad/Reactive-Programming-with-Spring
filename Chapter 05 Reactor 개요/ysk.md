## Reactor

Reactor란 라이브러리로, 스프링 팀의 주도하에 개발된 구현체로서 스프링5부터 리액티브 스택에 포함되어 Webflux 기반의 코어를 담당한다.

리액터의 특징은 다음과 같다

1. 리액티브 스트림즈를 구현한 리액티브 라이브러리다
2. JVM위에서 실행되는 Nonblocking의 핵심 기술이다. 
3. 자바의 함수형 프로그래밍 API를 통해 이뤄진다
4. FLUX (0~N개의 데이터 제공), Mono (0~1개의 데이터 제공)을 제공한다
5. 백프레셔를 지원한다. 즉 부하가 걸리지 않게 데이터 흐름을 제어할 수 있도록 한다. 

## 코드로 보는 리액터의 구성요소

```java
@Slf4j
public class Example5_1 {
    public static void main(String[] args) {
        Flux<String> sequence = Flux.just("Hello", "Reactor");
        sequence.map(data -> data.toLowerCase())
                .subscribe(data -> System.out.println(data));
    }
}

```

* Flux는 Publisher의 역할을 한다. 즉 입력으로 들어오는 데이터를 제공한다. 
  * Publisher가 최초로 제공하는 가공되지 않는 "hello", "Reactor"는 데이터 소스라고 부른다.
* subscribe 메서드 파라미터로 전달된 람다 표현식이 Subscriber 역할을 한다. (소비)
* map 메서드는 Operator 메서드인데, 전달받은 데이터를 가공하는 역할을 한다. 
  * Operator는 반환값으로 Mono 또는 Flux를 반환하기 때문에 Operator 메서드 체이닝을 형성한다.