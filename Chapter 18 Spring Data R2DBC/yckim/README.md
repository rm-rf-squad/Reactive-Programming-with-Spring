# Spring Data R2DBC

## R2DBC란?

R2DBC는 관계형 데이터베이스에 리액티브 프로그래밍 API를 제공하기 위한 개방형 사양이면서, 드라이버 벤드가 구현하고 클라이언트가 사용하기 위한 SPI입니다.

R2DBC 이전에는 몇몇 NoSQL 벤더만 리액티브 데이터베이스 클라이언트를 제공했습니다.

따라서 RDB를 사용하는 경우 JDBC API 자체가 Blocking API이기 때문에 완전한 Blocking I/O를 지원하는 것이 불가능했습니다.

하지만 R2DBC의 등장으로 관계형 데이터베이스를 사용하더라도 클라이언트의 요청부터 데이터베이스 액세스 까지 완전한 Non-Blocking 애플리케이션을 구현하는 것이 가능해졌습니다.

## Spring Data R2DBC

Spring Data R2DBC는 R2DBC 기반 Repository를 좀 더 쉽게 구현하게 해 주는 Spring Data 프로젝트의 일부입니다.

Spring Data R2DBC는 JPA 같은 ORM 프레임워크에서 제공하는 캐싱, 지연 로딩, 기타 ORM 프레임워크에서 가지고 있는 특징들이 제거되어 단순하고 심플한 방법으로 사용할 수 있습니다.

또한, 액세스 계층의 보일러플레이트 코드의 양을 대폭 줄일 수 있습니다.

## Spring Data R2DBC 설정

### build.gradle 설정

build.gradle에 다음과 같이 의존성을 추가합니다.

```java
implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
runtimeOnly 'io.r2dbc:r2dbc-h2'
```

### 테이블 스키마 정의

```java
CREATE TABLE IF NOT EXISTS BOOK (
    id bigint AUTO_INCREMENT PRIMARY KEY,
    title varchar(255) NOT NULL,
    author varchar(255) NOT NULL,
    description varchar(255) NOT NULL,
    isbn varchar(255) NOT NULL,
    created_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### application.yml 설정

```java
spring:
  sql:
    init:
      schema-locations: classpath*:db/h2/schema.sql

logging:
  level:
    org:
      springframework:
        r2dbc: DEBUG
```

### R2DBC Repository와 Auditing 기능 활성화

```java
@EnableR2dbcRepositories
@EnableR2dbcAuditing
public class WebfluxPlaygroundApplication {
```

## Spring Data R2DBC에서의 도메인 클래스 매핑

```java
@NoArgsConstructor
@Setter
public class Book {
    @Id
    private Long id;

    private String title;
    private String author;
    private String description;
    private String isbn;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column("last_modified_at")
    private LocalDateTime modifiedAt;
}
```

- Book 도메인 엔티티 클래스를 Book 테이블과 매핑하기 위해서 기본적으로 기본키에 해당하는 필드에 @Id 애너테이션을 추가해야 합니다.
- 클래스 레벨에 @Table 애너테이션을 생략하면 기본적으로 클래스 이름을 테이블 이름으로 사용합니다.
- 도서 정보가 테이블에 저장 또는 업데이트될 때, 생성 일시와 수정 일시를 자동으로 테이블에 반영하기 위해 @CreatedDate, @LastModifiedDate 애너테이션을 추가하여 Auditing 기능을 적용합니다.

## R2DBC Repositories를 이용한 데이터 액세스

### 레포지터리 클래스 구현

```java
public interface BookRepository extends ReactiveCrudRepository<Book, Long> {
    Mono<Book> findByIsbn(String isbn);
}
```

Spring Data R2DBC의 Repository와 다른 점은 리액티브를 지원하는 ReactiveCrudRepository를 상속한다는 점과 쿼리 메서드의 리턴 타입이 Mono 또는 Flux 라는 것입니다.

### 서비스 클래스 구현

```java
@Service
@RequiredArgsConstructor
public class BookService {

    private final BookRepository bookRepository;

    public Mono<Book> getBookByIsbn(String isbn) {
        return bookRepository.findByIsbn(isbn);
    }

    public Mono<Book> saveBook(Book book) {
        return bookRepository.save(book);
    }

    public Mono<List<Book>> getAllBooks() {
        return bookRepository.findAll().collectList();
    }

    public Mono<Book> getBookById(Long id) {
        return bookRepository.findById(id);
    }

    public Mono<Book> updateBook(Long id, Book updatedBook) {
        return bookRepository.findById(id)
                .flatMap(book -> {
                    book.setTitle(updatedBook.getTitle());
                    book.setAuthor(updatedBook.getAuthor());
                    book.setDescription(updatedBook.getDescription());
                    book.setIsbn(updatedBook.getIsbn());
                    return bookRepository.save(book);
                });
    }

    public Mono<Void> deleteBook(Long id) {
        return bookRepository.deleteById(id);
    }
}

```

## R2dbcEntityTemplate을 이용한 데이터 액세스

QuerDSL와 유사한 방식으로 메서드를 조합하여 데이터베이스와 상호작용할 수 있는 R2dbcEntityTemplate 방식도 존재합니다.

```java
@Service
@RequiredArgsConstructor
public class R2dbcEntityTemplateBookService {
    private final R2dbcEntityTemplate r2dbcEntityTemplate;
    public Mono<Book> getBookByIsbn(String isbn) {
        return r2dbcEntityTemplate
                .select(Book.class)
                .matching(Query.query(Criteria.where("isbn").is(isbn)))
                .one();
    }

    public Mono<Book> saveBook(Book book) {
        return r2dbcEntityTemplate.insert(Book.class)
                .using(book);
    }

    public Mono<List<Book>> getAllBooks() {
        return r2dbcEntityTemplate
                .select(Book.class)
                .all()
                .collectList();
    }

    public Mono<Book> getBookById(Long id) {
        return r2dbcEntityTemplate
                .select(Book.class)
                .matching(Query.query(Criteria.where("id").is(id)))
                .one();
    }

    public Mono<Book> updateBook(Long id, Book updatedBook) {
        return r2dbcEntityTemplate
                .select(Book.class)
                .matching(Query.query(Criteria.where("id").is(id)))
                .one()
                .flatMap(existingBook -> {
                    existingBook.setTitle(updatedBook.getTitle());
                    existingBook.setAuthor(updatedBook.getAuthor());
                    existingBook.setDescription(updatedBook.getDescription());
                    existingBook.setIsbn(updatedBook.getIsbn());
                    existingBook.setModifiedAt(LocalDateTime.now());
                    return r2dbcEntityTemplate.update(existingBook);
                });
    }

    public Mono<Void> deleteBook(Long id) {
        return r2dbcEntityTemplate
                .delete(Book.class)
                .matching(Query.query(Criteria.where("id").is(id)))
                .all()
                .then();
    }
}
```

### Entrypoint method

R2dbcTemplate은 SQL 쿼리문의 시작 구문인 SELECT, INSERT, UPDATE, DELETE 등에 해당하는 select(), insert(), update(), delete() 메서드를 Entrypoint method라고 부릅니다.

### Terminating method

SQL문을 생성하고 최종적으로 SQL 문을 실행하는 메서드를 Terminating method라고 부릅니다.

| method | 설명 |
| --- | --- |
| first() | 조건에 일치하는 result row 중에서 first row를 얻고자 할 경우 사용할 수 있습니다. 만약 조건에 일차하는 row가 없다면 Mono<Void>를 리턴합니다. |
| one() | 조건에 일차하는 result row가 단 하나일 경우 사용할 수 있습니다. 만약 조건에 일차하는 row가 없다면 Mono<Void>를 리턴하며, result row가 한 건보다 많을 경우 Exception이 발생합니다. |
| all() | 조건에 일치하는 모든 result row를 얻고자 할 경우 사용할 수 있습니다. |
| count() | 조건에 일치하는 데이터의 건수만 조회할 경우 사용할 수 있습니다. 리턴 타입은 Mono<Long>입니다. |
| exists() | 조건에 일치하는 result row가 존재하는지 여부를 확인하고자 할 경우 사용할 수 있습니다. 리턴 타입은 Mono<Boolean> 입니다. |

### Criteria method

R2dbcEntityTemplate은 SQL 연산자에 해당하는 다양한 Criteria method를 표와 같이 지원합니다.

| method | 설명 |
| --- | --- |
| and(String column) | SQL 쿼리문에서 and 연산자에 해당됩니다. |
| or(String column) | SQL 쿼리문에서 or 연산자에 해당됩니다. |
| greaterThan(Object o) | SQL 쿼리문에서 > 연산자에 해당됩니다. |
| GreaterThanOrEquals
(Object o) | SQL 쿼리문에서 ≥ 연산자에 해당됩니다. |
| in(Object… o) 또는
in(Collection<?> collection) | SQL 쿼리문에서 IN 연산자에 해당됩니다. |
| is(Object o) | SQL 쿼리문에서 = 연산자에 해당됩니다. |
| isNull() | SQL 쿼리문에서 IS NULL 연산자에 해당됩니다. |
| isNotNull() | SQL 쿼리문에서 IS NOT NULL 연산자에 해당됩니다. |
| lessThan(Object o) | SQL 쿼리문에서 < 연산자에 해당됩니다. |
| lessThanOrEquals
(Object o) | SQL 쿼리문에서 ≤ 연산자에 해당됩니다. |
| like(Object o) | SQL 쿼리문에서 LIKE 연산자에 해당됩니다. |
| not(Object o) | SQL 쿼리문에서 ≠ 또는 NOT 연산자에 해당됩니다. |
| notIn(Object… o) 또는
notIn(Collection<?> collection) | SQL 쿼리문에서 NOT IN 연산자에 해당됩니다. |

## Spring Data R2DBC에서의 페이지네이션(Pagination) 처리

### Repository에서의 페이지네이션

Spring Data R2DBC에서 데이터 액세스를 위해 Repository를 이용할 경우, 페이지네이션 처리는 다른 Spring Data 프로젝트에서의 페이지네이션 처리와 별반 다를게 없습니다.

```java
Flux<Book> findAllBy(Pageable pageable);
```

### R2dbcEntityTemplate 에서의 페이지네이션 처리

R2dbcEntityTemplate은 limit(), offset(), sort() 등의 쿼리 빌드 메서드를 조합하면 페이지네이션 처리를 간단히 적용할 수 있습니다.

```java
    return r2dbcEntityTemplate
            .select(Book.class)
            .all() 
            .skip(offset) 
            .take(size);
```