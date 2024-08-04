# 애너테이션 기반 컨트롤러

## Spring MVC 기반 Controller

```java

@GetMapping
public ResponseEntity<String> getAllUsers() {
    return new ResponseEntity<>("All users retrieved: " + users, HttpStatus.OK);
}

@PostMapping
public ResponseEntity<String> createUser(@RequestBody String user) {
    users.add(user);
    return new ResponseEntity<>("User created: " + user, HttpStatus.CREATED);
}

@PutMapping
public ResponseEntity<String> updateUser(@RequestBody String user) {
    if (users.contains(user)) {
        return new ResponseEntity<>("User updated: " + user, HttpStatus.OK);
    } else {
        return new ResponseEntity<>("User not found", HttpStatus.NOT_FOUND);
    }
}

@DeleteMapping
public ResponseEntity<String> deleteUser(@RequestBody String user) {
    if (users.remove(user)) {
        return new ResponseEntity<>("User deleted: " + user, HttpStatus.NO_CONTENT);
    } else {
        return new ResponseEntity<>("User not found", HttpStatus.NOT_FOUND);
    }
}
```

@RequestBody 애너테이션을 지정해서 클라이언트의 요청 데이터를 전달받고 ResponseEntity 클래스를 이용해 응답 데이터를 클라이언트에 전달하는 방식을 사용합니다.

## Spring WebFlux 기반 Controller

```java

@GetMapping
public Mono<List<String>> getAllUsers() {
    return Mono.just(users);
}

@PostMapping
public Mono<ResponseEntity<String>> createUser(@RequestBody String user) {
    users.add(user);
    return Mono.just(new ResponseEntity<>("User created: " + user, HttpStatus.CREATED));
}

@PutMapping
public Mono<ResponseEntity<String>> updateUser(@RequestBody String user) {
    if (users.contains(user)) {
        return Mono.just(new ResponseEntity<>("User updated: " + user, HttpStatus.OK));
    } else {
        return Mono.just(new ResponseEntity<>("User not found", HttpStatus.NOT_FOUND));
    }
}

@DeleteMapping
public Mono<ResponseEntity<String>> deleteUser(@RequestBody String user) {
    if (users.remove(user)) {
        return Mono.just(new ResponseEntity<>("User deleted: " + user, HttpStatus.NO_CONTENT));
    } else {
        return Mono.just(new ResponseEntity<>("User not found", HttpStatus.NOT_FOUND));
    }
}
```

Spring MVC와 비교했을 때 Mono 타입으로 반환한다는 것 이외에는 변경된 부분이 거의 없습니다.

## Blocking 요소 제거

현재 코드에 DTO 객체와 엔티티 객체를 서로 변환하는 지점에 Blocking 요소가 포함되어 있습니다.

처리 시간이 긴 작업은 아니지만 Blocking 요소가 없도록 개선하면 더 좋을 것 입니다.

```java

@PostMapping
public Mono<ResponseEntity<String>> createUser(@RequestBody Mono<String> userMono) {
    return userMono.flatMap(user -> {
        users.add(user);
        return Mono.just(new ResponseEntity<>("User created: " + user, HttpStatus.CREATED));
    });
}

@PutMapping
public Mono<ResponseEntity<String>> updateUser(@RequestBody Mono<String> userMono) {
    return userMono.flatMap(user -> {
        if (users.contains(user)) {
            return Mono.just(new ResponseEntity<>("User updated: " + user, HttpStatus.OK));
        } else {
            return Mono.just(new ResponseEntity<>("User not found", HttpStatus.NOT_FOUND));
        }
    });
}

@DeleteMapping
public Mono<ResponseEntity<String>> deleteUser(@RequestBody Mono<String> userMono) {
    return userMono.flatMap(user -> {
        if (users.remove(user)) {
            return Mono.just(new ResponseEntity<>("User deleted: " + user, HttpStatus.NO_CONTENT));
        } else {
            return Mono.just(new ResponseEntity<>("User not found", HttpStatus.NOT_FOUND));
        }
    });
}
```