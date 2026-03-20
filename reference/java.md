# Java Code Review Guide

Focus: Java 17/21 features, Spring Boot 3 practices, concurrency (virtual threads), JPA performance, and maintainability.

## Contents

- [Modern Java (17/21+)](#modern-java-1721)
- [Stream API & Optional](#stream-api--optional)
- [Spring Boot practices](#spring-boot-practices)
- [JPA & database performance](#jpa--database-performance)
- [Concurrency & virtual threads](#concurrency--virtual-threads)
- [Lombok guidelines](#lombok-guidelines)
- [Exception handling](#exception-handling)
- [Testing](#testing)
- [Review Checklist](#review-checklist)

---

## Modern Java (17/21+)

### Records

```java
// ❌ Classic POJO/DTO — lots of boilerplate
public class UserDto {
    private final String name;
    private final int age;

    public UserDto(String name, int age) {
        this.name = name;
        this.age = age;
    }
    // getters, equals, hashCode, toString...
}

// ✅ Record — concise, immutable, clear intent
public record UserDto(String name, int age) {
    // compact constructor for validation
    public UserDto {
        if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
    }
}
```

### Switch expressions & pattern matching

```java
// ❌ Classic switch: easy to miss break; verbose and error-prone
String type = "";
switch (obj) {
    case Integer i: // Java 16+
        type = String.format("int %d", i);
        break;
    case String s:
        type = String.format("string %s", s);
        break;
    default:
        type = "unknown";
}

// ✅ Switch expression: no fall-through, value must be produced
String type = switch (obj) {
    case Integer i -> "int %d".formatted(i);
    case String s  -> "string %s".formatted(s);
    case null      -> "null value"; // Java 21
    default        -> "unknown";
};
```

### Text blocks

```java
// ❌ Concatenated SQL/JSON strings
String json = "{\n" +
              "  \"name\": \"Alice\",\n" +
              "  \"age\": 20\n" +
              "}";

// ✅ Text blocks — readable, minimal escaping
String json = """
    {
      "name": "Alice",
      "age": 20
    }
    """;
```

---

## Stream API & Optional

### Don’t overuse streams

```java
// ❌ Simple loop doesn’t need a stream (overhead + harder to read)
items.stream().forEach(item -> {
    process(item);
});

// ✅ Plain for-each for simple cases
for (var item : items) {
    process(item);
}

// ❌ Overly long pipeline
List<Dto> result = list.stream()
    .filter(...)
    .map(...)
    .peek(...)
    .sorted(...)
    .collect(...); // hard to debug

// ✅ Break into named steps
var filtered = list.stream().filter(...).toList();
// ...
```

### Using Optional well

```java
// ❌ Optional as parameter or field (serialization, awkward callers)
public void process(Optional<String> name) { ... }
public class User {
    private Optional<String> email; // discouraged
}

// ✅ Optional mainly for return types
public Optional<User> findUser(String id) { ... }

// ❌ isPresent() + get() after adopting Optional
Optional<User> userOpt = findUser(id);
if (userOpt.isPresent()) {
    return userOpt.get().getName();
} else {
    return "Unknown";
}

// ✅ Functional style
return findUser(id)
    .map(User::getName)
    .orElse("Unknown");
```

---

## Spring Boot practices

### Dependency injection

```java
// ❌ Field injection (@Autowired)
// Harder to test (reflection), hides large dependency lists, not immutable-friendly
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo;
}

// ✅ Constructor injection
// Dependencies explicit, easy to mock, fields can be final
@Service
public class UserService {
    private final UserRepository userRepo;

    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
// 💡 Lombok @RequiredArgsConstructor helps — watch for circular dependencies
```

### Configuration

```java
// ❌ Hard-coded secrets/settings
@Service
public class PaymentService {
    private String apiKey = "sk_live_12345";
}

// ❌ Many scattered @Value fields
@Value("${app.payment.api-key}")
private String apiKey;

// ✅ Type-safe @ConfigurationProperties
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(String apiKey, int timeout, String url) {}
```

---

## JPA & database performance

### N+1 queries

```java
// ❌ FetchType.EAGER or lazy loading in a loop
@Entity
public class User {
    @OneToMany(fetch = FetchType.EAGER) // risky
    private List<Order> orders;
}

List<User> users = userRepo.findAll(); // may already be heavy
for (User user : users) {
    // Lazy: N extra queries here
    System.out.println(user.getOrders().size());
}

// ✅ @EntityGraph or JOIN FETCH
@Query("SELECT u FROM User u JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

### Transactions

```java
// ❌ @Transactional on controllers (holds connections too long)
// ❌ @Transactional on private methods (AOP proxy won’t apply)
@Transactional
private void saveInternal() { ... }

// ✅ @Transactional on public service methods
// ✅ readOnly = true for queries
@Service
public class UserService {
    @Transactional(readOnly = true)
    public User getUser(Long id) { ... }

    @Transactional
    public void createUser(UserDto dto) { ... }
}
```

### Entity design

```java
// ❌ Lombok @Data on entities
// equals/hashCode over all fields can touch lazy collections → bugs / perf hits
@Entity
@Data
public class User { ... }

// ✅ @Getter / @Setter only
// ✅ equals/hashCode based on stable id (typical pattern)
@Entity
@Getter
@Setter
public class User {
    @Id
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        return id != null && id.equals(((User) o).id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

---

## Concurrency & virtual threads

### Virtual threads (Java 21+)

```java
// ❌ Large fixed pool for massive blocking I/O (thread exhaustion)
ExecutorService executor = Executors.newFixedThreadPool(100);

// ✅ Virtual threads for I/O-heavy workloads
// Spring Boot 3.2+: spring.threads.virtual.enabled=true
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// Blocking calls (DB, HTTP) consume little OS-thread budget on virtual threads
```

### Thread safety

```java
// ❌ SimpleDateFormat is not thread-safe
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// ✅ DateTimeFormatter (Java 8+)
private static final DateTimeFormatter dtf = DateTimeFormatter.ofPattern("yyyy-MM-dd");

// ❌ HashMap under concurrent writes — corruption / lost updates
// ✅ ConcurrentHashMap
Map<String, String> cache = new ConcurrentHashMap<>();
```

---

## Lombok guidelines

```java
// ❌ @Builder without enforcing required fields
@Builder
public class Order {
    private String id;   // required
    private String note; // optional
}
// Caller can omit id: Order.builder().note("hi").build();

// ✅ For critical domains: hand-written builder/constructor or validation in build()
// (e.g. @Builder.Default, custom builder class)
```

---

## Exception handling

### Centralized handling

```java
// ❌ Scattered try/catch that swallows or only prints
try {
    userService.create(user);
} catch (Exception e) {
    e.printStackTrace(); // not for production
    // return null; // caller loses error context
}

// ✅ Domain exceptions + @ControllerAdvice (Spring Boot 3 ProblemDetail)
public class UserNotFoundException extends RuntimeException { ... }

@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail handleNotFound(UserNotFoundException e) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
    }
}
```

---

## Testing

### Unit vs integration

```java
// ❌ “Unit” test that boots full stack / hits real DB
@SpringBootTest // whole context — slow
public class UserServiceTest { ... }

// ✅ Unit tests with Mockito
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service;

    @Test
    void shouldCreateUser() { ... }
}

// ✅ Integration tests with Testcontainers
@Testcontainers
@SpringBootTest
class UserRepositoryTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    // ...
}
```

---

## Review Checklist

### Language & style
- [ ] Modern Java 17/21 where appropriate (switch expressions, records, text blocks)
- [ ] Avoid legacy APIs (`Date`, `Calendar`, `SimpleDateFormat`) for new code
- [ ] Collections: streams or `Collections` helpers where they improve clarity
- [ ] `Optional` for returns only — not fields or parameters

### Spring Boot
- [ ] Constructor injection, not `@Autowired` fields
- [ ] Typed config via `@ConfigurationProperties`
- [ ] Thin controllers; logic in services
- [ ] Global errors via `@ControllerAdvice` / `ProblemDetail`

### Database & transactions
- [ ] Read paths use `@Transactional(readOnly = true)` when transactional
- [ ] No N+1 (eager fetch abuse or lazy in loops)
- [ ] Entities: no `@Data`; sane `equals`/`hashCode`
- [ ] Indexes match real query predicates

### Concurrency & performance
- [ ] Virtual threads considered for I/O-bound work
- [ ] Correct concurrent structures (`ConcurrentHashMap` vs `HashMap`)
- [ ] Lock scope: avoid I/O and long work inside locks

### Maintainability
- [ ] Important logic covered by fast unit tests
- [ ] Logging via SLF4J — not `System.out`
- [ ] Magic strings/numbers replaced with constants or enums
