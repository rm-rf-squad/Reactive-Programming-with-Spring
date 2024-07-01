# Chapter 07 Cold Sequence와 Hot Sequence

Hot Swap : 컴퓨터가 커진 상태에서 디스크 등 교체하더라도 시스템 재시작 없이 바로 장치 인식

Hot Deploy : 서버를 재시작하지않고 변경사항 적용

> Cold는 무언가를 새로 시작하고, Hot은 무언가를 새로 시작하지 않는다.

## Cold Sequence

Cold Sequence는 스트림에 구독자가 구독할 때마다 새롭게 시작되는 데이터 시퀀스를 의미한다.

구독시점이 달라도, 데이터를 emit하는 과정을 처음부터 재시작하여 구독자들은 모두 동일한 데이터를 전달받는것을 Cold Sequence라고 한다.  

* 모든 구독자는 독립적인 데이터 시퀀스를 받는다.. 즉, 구독자가 스트림을 구독할 때마다 데이터가 처음부터 시작된다.

![image-20240601170742397](./images//image-20240601170742397.png)

* 그림을 보면, Subscriber A,B가 구독시점이 다르지만 같은 4개의 data를 emit하는것을 볼 수 있다. 즉 동일한 데이터를 전달받는다.

``````java
@Slf4j
public class Example7_1 {
    public static void main(String[] args) throws InterruptedException {
        Flux<String> coldFlux = Flux.fromIterable(Arrays.asList("KOREA", "JAPAN", "CHINESE"))
                    .map(String::toLowerCase);

        coldFlux.subscribe(country -> log.info("# Subscriber1: {}", country));
        System.out.println("----------------------------------------------------------------------");
        Thread.sleep(2000L);
        coldFlux.subscribe(country -> log.info("# Subscriber2: {}", country));
    }
}
``````

* 두 구독자가 발행자를 구독하는 시점이 다르다. 그래도 같은 데이터결과가 출력된다. 

Cold sequence는 데이터 재사용 및 반복 가능성을 제공한다

이는 데이터의 무결성을 보장하며, 테스트와 디버깅에서 매우 유용하며, 동일한 스트림을 다른 구독자가 각각 독립적으로 처리할 수 있다. 

즉 구독자가 각각 다른 시점에 같은 데이터를 다른 방식으로 처리하는경우에 유용하다 

이를 통해 데이터의 재사용성, 무결성, 리소스 효율성, 구독 관리의 유연성을 확보할 수 있다. 

* 예시로, 같은 파일에서 json과 xml을 각각 만들 수 있는것이 있겠다. 

## Hot Sequence

Hot Sequence는 스트림이 시작되면 구독자와 상관없이 데이터를 계속 발행하는 시퀀스를 의미한다.

Cold와 반대로, 데이터 스트림이 새로 시작되지 않고 중간에 구독에 끼여들면 구독 시점 이전에 데이터는 전달받지 못하고 시점부터의 데이터만 전달받게 된다.

![image-20240601171302085](./images//image-20240601171302085.png)

* 세번의 구독이 발생했지만, 타임라인은 1개밖에 생성되지 않았다.  

```java
@Slf4j
public class Example7_2 {
    public static void main(String[] args) throws InterruptedException {
        String[] singers = {"Singer A", "Singer B", "Singer C", "Singer D", "Singer E"};

        log.info("# Begin concert:");
        Flux<String> concertFlux =
                Flux
                    .fromArray(singers)
                    .delayElements(Duration.ofSeconds(1))
                    .share();

        concertFlux.subscribe(
                singer -> log.info("# Subscriber1 is watching {}'s song", singer)
        );

        Thread.sleep(2500);

        concertFlux.subscribe(
                singer -> log.info("# Subscriber2 is watching {}'s song", singer)
        );

        Thread.sleep(3000);
    }
}

```

* 초당 1개의 엘리먼트를 반환하는 Flux이고, 3초쯤부터 두번째 구독자가 구독하므로 C부터 받을 수 있게 된다

Hot Sequence는 실시간 데이터 처리에 유용하다. 또한 하나의 데이터 소스를 여러 구독자가 동시에 사용할 때 리소스를 절약할 수 있다.

share() Operator는 Cold Sequence를 Hot Sequence로 동작하게 해주는 Operator인데 

원본 Flux(전혀 아무것도 가공되지 않은 처음의 소스)를 여러 Subscriber가 공유해서 사용할 수 있게 된다. 

* 예시로 센서 데이터를 실시간으로 처리하는 스트림을 생성하고, 여러 구독자가 실시간으로 데이터를 받을 수 있겠다. 

## HTTP 요청 응답에서 Cold Sequence와 Hot Sequence의 동작 흐름

```java
@Slf4j
public class Example7_3 {
    public static void main(String[] args) throws InterruptedException {
        URI worldTimeUri = UriComponentsBuilder.newInstance().scheme("http")
                .host("worldtimeapi.org")
                .port(80)
                .path("/api/timezone/Asia/Seoul")
                .build()
                .encode()
                .toUri();

        Mono<String> mono = getWorldTime(worldTimeUri); // 주목 
        mono.subscribe(dateTime -> log.info("# dateTime 1: {}", dateTime));
        Thread.sleep(2000);
        mono.subscribe(dateTime -> log.info("# dateTime 2: {}", dateTime));

        Thread.sleep(2000);
    }

}
```

* getWorldTime내부에서는 그냥 webclient로 요청후 Mono를 반환한다.
* 아무 Operator가 없어 Cold Sequence로 동작해서, 두번째 구독이 일어나면 새로 다시 요청을 보내게 된다. 

```java
@Slf4j
public class Example7_4 {
    public static void main(String[] args) throws InterruptedException {
        URI worldTimeUri = UriComponentsBuilder.newInstance().scheme("http")
                .host("worldtimeapi.org")
                .port(80)
                .path("/api/timezone/Asia/Seoul")
                .build()
                .encode()
                .toUri();

        Mono<String> mono = getWorldTime(worldTimeUri).cache(); // here
        mono.subscribe(dateTime -> log.info("# dateTime 1: {}", dateTime));
        Thread.sleep(2000);
        mono.subscribe(dateTime -> log.info("# dateTime 2: {}", dateTime));

        Thread.sleep(2000);
    }
}
// 결과 - 이 둘이 완전히 같다. 
[reactor-http-nio-2] INFO - # dateTime 1: 2024-06-01T17:22:07.792904+09:00
[main] INFO -               # dateTime 2: 2024-06-01T17:22:07.792904+09:00
```

* cache() Operator를 Mono에서 추가했다. 이는 Cold Sequence가 HotSequence로 동작하게 한다.

* 결과적으로 emit된 데이터를 캐시한 뒤, 구독이 발생하면 캐시된 데이터를 전달한다. 

cache() Operator를 활용할 수 있는 예로 인증 토큰이 필요한 경우를 들 수 있다.

매번 인증서버로부터 토큰을 받아오기에 부담일 수 있으므로 캐시된것을 재사용 할 수 있다.

* cache() 오퍼레이터의 파라미터에 ttl 등을 설정할 수 있다. 

