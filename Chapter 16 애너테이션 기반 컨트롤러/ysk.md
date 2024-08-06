# Chapter 16 애너테이션 기반 컨트롤러

Spring MVC와 같이 어노테이션 기반으로 요청을 핸들링한다.

```java
@RestController
@RequestMapping("/v1/books")
public class BookController {
    private final BookService bookService;
    private final BookMapper mapper;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono postBook(@RequestBody BookDto.Post requestBody) {
        Mono<Book> book =
                bookService.createBook(mapper.bookPostToBook(requestBody));

        Mono<BookDto.Response> response = mapper.bookToBookResponse(book);
        return response;
    }
  
     @PatchMapping("/{book-id}")
    public Mono patchBook(@PathVariable("book-id") long bookId,
                                    @RequestBody BookDto.Patch requestBody) {
        requestBody.setBookId(bookId);
        Mono<Book> book =
                bookService.updateBook(mapper.bookPatchToBook(requestBody));

        return mapper.bookToBookResponse(book);
    }

    @GetMapping("/{book-id}")
    public Mono getBook(@PathVariable("book-id") long bookId) {
        Mono<Book> book = bookService.findBook(bookId);

        return mapper.bookToBookResponse(book);
    }
}
```

그러나 여기에 Blocking 요소가 있다 무엇일까 ?

```java
@RestController("bookControllerV2")
@RequestMapping("/v2/books")
public class BookController {
    private final BookService bookService;
    private final BookMapper mapper;

    public BookController(BookService bookService, BookMapper mapper) {
        this.bookService = bookService;
        this.mapper = mapper;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono postBook(@RequestBody Mono<BookDto.Post> requestBody) {
        Mono<Book> result = bookService.createBook(requestBody);

        return result.flatMap(book -> Mono.just(mapper.bookToResponse(book)));
    }

    @PatchMapping("/{book-id}")
    public Mono patchBook(@PathVariable("book-id") long bookId,
                                    @RequestBody Mono<BookDto.Patch> requestBody) {
        Mono<Book> result = bookService.updateBook(bookId, requestBody);
        return result.flatMap(book -> Mono.just(mapper.bookToResponse(book)));
    }

    @GetMapping("/{book-id}")
    public Mono getBook(@PathVariable("book-id") long bookId) {
        return bookService.findBook(bookId)
                .flatMap(book -> Mono.just(mapper.bookToResponse(book)));
    }
}
```

1. method argument로 리액티브 타입을 지원한다.
2. getBook에서 findBook으로부터 받은 Mono를 mapper를 이용해 중간에 block하지말고, flatMap으로 변환해서 전달한다.

#### Mono로 받지 않는 경우

- **블로킹 가능성**: 요청 본문을 동기적으로 처리하므로, 큰 요청 본문을 처리할 때 스레드가 블로킹될 수 있다.
- **성능 저하**: 동기적으로 처리하는 동안 다른 요청을 처리하는 데 필요한 리소스가 부족해질 수 있어 전체적인 성능이 저하될 가능성이 있다.
- **리액티브 파이프라인 단절**: 리액티브 처리 파이프라인이 중간에 동기적으로 변환되므로, 비동기 처리의 이점을 상실할 수 있다.