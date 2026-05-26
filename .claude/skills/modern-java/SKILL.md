---
name: modern-java
license: Apache-2.0
metadata:
  source: https://github.com/open-elements/claude-base
  author: Open Elements
description: Write idiomatic modern Java (11–25) by actively using newer language features and APIs instead of legacy patterns. Covers text blocks, records, sealed classes, pattern matching (instanceof, switch, record patterns, primitive patterns), unnamed variables, sequenced collections, structured concurrency, scoped values, stream gatherers, flexible constructor bodies, module imports, Markdown documentation comments, code snippets in Javadoc, and modern API additions (HttpClient, Stream.toList, String methods, RandomGenerator, Foreign Function & Memory API, cryptography APIs including EdDSA, KEM, ML-KEM, ML-DSA, KDF, PEM encoding, etc.). Trigger this skill whenever writing or reviewing Java code to ensure modern idioms are used instead of pre-Java-11 patterns.
---

# Modern Java

Write idiomatic Java by leveraging language features and APIs introduced in Java 11 through 25. Most training data and internet examples use Java 8–11 patterns. This skill ensures that generated code uses the best available constructs for the target Java version.

## Instructions

### 1. Determine the target Java version

Before writing code, check the project's Java version:

- Look at `pom.xml` (`<java.version>`, `<maven.compiler.release>`, `<maven.compiler.source>`).
- Look at `build.gradle` / `build.gradle.kts` (`sourceCompatibility`, `toolchain`).
- Look at `.sdkmanrc` or `Dockerfile` base images.

Only use features available in the project's target version. When the version is unclear, ask the user.

### 2. Language features by version

#### Java 11 — Local variable syntax for lambda parameters

```java
// Old: explicit types or no annotation possible
(String x, String y) -> x + y

// Modern: var in lambdas enables annotations
(@NonNull var x, @NonNull var y) -> x + y
```

#### Java 14 — Switch expressions

Replace traditional switch statements with switch expressions that return values.

```java
// Old
String label;
switch (status) {
    case ACTIVE:
        label = "Active";
        break;
    case INACTIVE:
        label = "Inactive";
        break;
    default:
        label = "Unknown";
        break;
}

// Modern
String label = switch (status) {
    case ACTIVE -> "Active";
    case INACTIVE -> "Inactive";
    default -> "Unknown";
};
```

Use `yield` for multi-line blocks:

```java
String description = switch (status) {
    case ACTIVE -> "Active";
    case PENDING -> {
        log.info("Pending status found");
        yield "Pending Review";
    }
    default -> "Unknown";
};
```

#### Java 15 — Text blocks

Replace string concatenation or `\n`-littered strings with text blocks for multi-line content.

```java
// Old
String json = "{\n" +
    "  \"name\": \"Alice\",\n" +
    "  \"age\": 30\n" +
    "}";

// Modern
String json = """
        {
          "name": "Alice",
          "age": 30
        }
        """;
```

Use text blocks for SQL, JSON, HTML, XML, log messages, and any multi-line string. Indentation is managed by the position of the closing `"""`.

#### Java 16 — Records

Use records for immutable data carriers instead of classes with boilerplate.

```java
// Old (25+ lines)
public final class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// Modern (1 line)
public record Point(int x, int y) {}
```

Use the compact constructor for validation:

```java
public record Range(int start, int end) {
    public Range {
        if (start > end) {
            throw new IllegalArgumentException("start must be <= end");
        }
    }
}
```

Defensive-copy mutable components:

```java
public record Team(String name, List<String> members) {
    public Team {
        Objects.requireNonNull(name, "name must not be null");
        members = List.copyOf(members);
    }
}
```

#### Java 16 — Pattern matching for instanceof

Eliminate explicit casts after `instanceof`.

```java
// Old
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}

// Modern
if (obj instanceof String s) {
    System.out.println(s.length());
}

// Also works with negation
if (!(obj instanceof String s)) {
    return;
}
// s is in scope here
System.out.println(s.length());
```

#### Java 17 — Sealed classes and interfaces

Use sealed types when the set of subtypes is known and finite.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double a, double b, double c) implements Shape {}
```

Sealed types enable exhaustive switch expressions (no `default` needed when all cases covered).

#### Java 21 — Pattern matching for switch

Combine sealed types and pattern matching for powerful, type-safe branching.

```java
// Old: instanceof chains
if (shape instanceof Circle c) {
    return Math.PI * c.radius() * c.radius();
} else if (shape instanceof Rectangle r) {
    return r.width() * r.height();
} else if (shape instanceof Triangle t) {
    double s = (t.a() + t.b() + t.c()) / 2;
    return Math.sqrt(s * (s - t.a()) * (s - t.b()) * (s - t.c()));
} else {
    throw new IllegalArgumentException("Unknown shape");
}

// Modern
return switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t -> {
        double s = (t.a() + t.b() + t.c()) / 2;
        yield Math.sqrt(s * (s - t.a()) * (s - t.b()) * (s - t.c()));
    }
};
```

Use guarded patterns with `when`:

```java
String format(Object obj) {
    return switch (obj) {
        case Integer i when i < 0 -> "negative: " + i;
        case Integer i -> "positive: " + i;
        case String s when s.isBlank() -> "(blank)";
        case String s -> s;
        case null -> "null";
        default -> obj.toString();
    };
}
```

`null` can be handled directly in switch (no more `NullPointerException` before the switch).

#### Java 21 — Record patterns

Destructure records directly in pattern matching.

```java
// Old
if (obj instanceof Point p) {
    int x = p.x();
    int y = p.y();
    System.out.println(x + ", " + y);
}

// Modern
if (obj instanceof Point(int x, int y)) {
    System.out.println(x + ", " + y);
}
```

Nested record patterns:

```java
record Point(int x, int y) {}
record Line(Point start, Point end) {}

// Deep destructuring
if (obj instanceof Line(Point(int x1, int y1), Point(int x2, int y2))) {
    double length = Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
}
```

Record patterns in switch:

```java
String describe(Shape shape) {
    return switch (shape) {
        case Circle(double r) when r > 100 -> "large circle";
        case Circle(double r) -> "circle with radius " + r;
        case Rectangle(double w, double h) when w == h -> "square " + w;
        case Rectangle(double w, double h) -> w + "x" + h + " rectangle";
    };
}
```

#### Java 21 — Sequenced collections

Use the new sequenced collection interfaces for ordered access.

```java
// Old
list.get(0);                        // first element
list.get(list.size() - 1);          // last element
var it = set.iterator();
it.next();                          // first of LinkedHashSet

// Modern
list.getFirst();
list.getLast();
set.getFirst();

// Also: addFirst, addLast, removeFirst, removeLast, reversed
list.reversed().forEach(System.out::println);
```

`SequencedCollection`, `SequencedSet`, and `SequencedMap` are new interfaces in the hierarchy.

For maps: `sequencedMap.firstEntry()`, `sequencedMap.lastEntry()`, `sequencedMap.reversed()`.

#### Java 21 — Virtual threads

Use virtual threads for I/O-bound concurrent work instead of platform threads or thread pools.

```java
// Old
ExecutorService pool = Executors.newFixedThreadPool(100);
pool.submit(() -> handleRequest(request));

// Modern: virtual threads for I/O tasks
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> handleRequest(request));

// Or directly
Thread.startVirtualThread(() -> handleRequest(request));

// With Thread.Builder
Thread.ofVirtual().name("worker-", 0).start(() -> handleRequest(request));
```

Important rules for virtual threads:

- Use for I/O-bound work (HTTP calls, database queries, file I/O). Not beneficial for CPU-bound work.
- Do not pool virtual threads — create a new one per task.
- **Java 21–23**: Avoid `synchronized` blocks that do I/O inside — use `ReentrantLock` instead (to avoid pinning the carrier thread).
- **Java 24+** (JEP 491): `synchronized` no longer pins virtual threads. The JVM now allows virtual threads to unmount inside `synchronized` blocks, so the `ReentrantLock` workaround is no longer necessary. `synchronized` is safe to use with virtual threads on Java 24+.
- `Thread.sleep()` and blocking I/O work correctly and efficiently on virtual threads.

#### Java 19 — ExecutorService as AutoCloseable

Use try-with-resources for automatic executor shutdown.

```java
// Old: manual shutdown in finally block
ExecutorService exec = Executors.newCachedThreadPool();
try {
    exec.submit(task);
} finally {
    exec.shutdown();
    exec.awaitTermination(1, TimeUnit.MINUTES);
}

// Modern: auto-shutdown on close
try (var exec = Executors.newCachedThreadPool()) {
    exec.submit(task);
}
// shutdown() + awaitTermination() called automatically
```

#### Java 19 — Thread.sleep(Duration)

```java
// Old: what unit is 5000? milliseconds? microseconds?
Thread.sleep(5000);

// Modern: self-documenting time units
Thread.sleep(Duration.ofSeconds(5));
Thread.sleep(Duration.ofMillis(500));
```

#### Java 22 — Unnamed variables and patterns

Use `_` for variables that are intentionally unused.

```java
// Old
try {
    // ...
} catch (NumberFormatException ignored) {
    // exception intentionally ignored
}
map.forEach((key, value) -> processValue(value)); // key unused

// Modern
try {
    // ...
} catch (NumberFormatException _) {
    // exception intentionally ignored
}
map.forEach((_, value) -> processValue(value));

// In pattern matching
switch (obj) {
    case Point(int x, _) -> "x=" + x;  // y not needed
    default -> "other";
}
```

#### Java 23 — Primitive types in patterns (preview, standard in 25)

Pattern matching extends to primitive types, eliminating manual casts and range checks.

```java
// Modern (Java 25)
return switch (statusCode) {
    case 200 -> "OK";
    case 301 -> "Moved";
    case 404 -> "Not Found";
    case int code when code >= 500 -> "Server Error: " + code;
    case int code -> "Other: " + code;
};
```

#### Java 23 — Module import declarations (preview, standard in 25)

Import all exported packages of a module with a single declaration.

```java
// Old
import java.util.List;
import java.util.Map;
import java.util.Set;
import java.util.stream.Collectors;

// Modern (Java 25)
import module java.base;
```

#### Java 24 — Flexible constructor bodies

Code can execute before `super()` or `this()` calls for validation and transformation. Previously all pre-super code was illegal.

```java
// Old: had to validate after super or use static helper
public class PositiveRange extends Range {
    public PositiveRange(int start, int end) {
        super(start, end); // what if start < 0?
        if (start < 0) throw new IllegalArgumentException(); // too late if super has side effects
    }
}

// Modern (Java 24)
public class PositiveRange extends Range {
    public PositiveRange(int start, int end) {
        if (start < 0) throw new IllegalArgumentException("start must be >= 0");
        super(start, end);
    }
}
```

Restriction: code before `super()`/`this()` cannot access `this` (cannot read fields, call instance methods, or use `this` as an argument).

#### Java 18 — Code snippets in Javadoc (@snippet)

Replace `<pre>{@code ...}</pre>` blocks with the `{@snippet}` tag for code examples in documentation.

```java
// Old: error-prone, no validation, no syntax highlighting
/**
 * Example usage:
 * <pre>{@code
 * List<String> list = List.of("a", "b");
 * }</pre>
 */

// Modern: validated, supports highlighting and regions
/**
 * Example usage:
 * {@snippet :
 * List<String> list = List.of("a", "b");   // @highlight substring="List.of"
 * }
 */
```

External snippets reference real compilable code from separate files:

```java
/**
 * {@snippet file="ShowExample.java" region="usage"}
 */
```

Where `ShowExample.java` in the `snippet-files` directory contains:

```java
public class ShowExample {
    void demo() {
        // @start region="usage"
        List<String> list = List.of("a", "b");
        // @end
    }
}
```

#### Java 23 — Markdown documentation comments

Use `///` with Markdown syntax instead of `/** */` with HTML and JavaDoc tags.

```java
// Old: HTML markup, verbose
/**
 * Returns the user with the given <em>id</em>.
 * <p>
 * The lookup follows these steps:
 * <ul>
 *   <li>Check the cache</li>
 *   <li>Query the database</li>
 * </ul>
 *
 * @param id the user ID
 * @return the user, or {@code null} if not found
 * @see UserRepository
 */

// Modern: Markdown syntax, cleaner
/// Returns the user with the given _id_.
///
/// The lookup follows these steps:
///
/// - Check the cache
/// - Query the database
///
/// @param id the user ID
/// @return the user, or `null` if not found
/// @see UserRepository
```

Key differences:
- `///` line comments instead of `/** */` block comments.
- Markdown for formatting: `_italic_`, `**bold**`, `` `code` ``, `-` for lists, blank lines for paragraphs.
- Links to API elements use Markdown reference links: `[List]` links to `java.util.List` (if imported), `[String#chars()]` links to the method.
- Tables use GitHub Flavored Markdown pipe syntax.
- JavaDoc block tags (`@param`, `@return`, `@see`, `@throws`) remain unchanged.
- Fenced code blocks (`` ``` ``) replace `{@snippet}` for simple inline examples.

#### Java 25 — Stable features arriving

Java 25 (September 2025) finalizes several important features:

- **Primitive types in patterns** — full pattern matching support for primitives in `instanceof` and `switch`.
- **Module import declarations** — `import module java.base;` to import all public types from a module.
- **Structured concurrency** — `StructuredTaskScope` for managing concurrent subtasks as a unit (see API section below).
- **Scoped values** — `ScopedValue` as a modern replacement for `ThreadLocal` (see API section below).
- **Stream gatherers** — custom intermediate stream operations via `Stream.gather()` (see API section below).
- **Compact source files** (Simple source files) — allows writing Java programs without explicitly declaring a class. The implicit class has an implicit `main` method if a top-level `main` method is present. Primarily for learning and scripting.

### 3. Modern API additions by version

#### Java 11 — String methods

```java
"  hello  ".strip()          // "hello" (Unicode-aware, prefer over trim())
"  hello  ".stripLeading()   // "hello  "
"  hello  ".stripTrailing()  // "  hello"
"  ".isBlank()               // true (better than trim().isEmpty())
"line1\nline2".lines()       // Stream<String> of lines
"ha".repeat(3)               // "hahaha"
```

Always prefer `strip()` over `trim()` — it handles Unicode whitespace correctly.

#### Java 9 — Stream and Optional enhancements

```java
// Stream.ofNullable — zero-or-one element stream from nullable value
Stream<String> s = Stream.ofNullable(nullableValue);  // empty if null

// Stream.iterate with predicate — like a for-loop in stream form
Stream.iterate(1, n -> n < 1000, n -> n * 2)
      .forEach(System.out::println);
// stops when n >= 1000 — no more limit() guessing

// takeWhile / dropWhile — short-circuit on ordered streams
List<Integer> small = sorted.stream()
        .takeWhile(n -> n < 100)
        .toList();
List<Integer> afterSkip = sorted.stream()
        .dropWhile(n -> n < 10)
        .toList();

// Collectors.flatMapping — flatten inside a grouping collector
Map<String, Set<String>> tagsByDept = employees.stream()
        .collect(Collectors.groupingBy(
                Employee::dept,
                Collectors.flatMapping(e -> e.tags().stream(), Collectors.toSet())
        ));

// Optional.or — chain fallback Optionals lazily
Optional<Config> cfg = primary()
        .or(this::secondary)
        .or(this::defaults);

// Optional.ifPresentOrElse — handle both cases in one call
findUser(id).ifPresentOrElse(
        this::greet,
        this::handleMissing
);
```

#### Java 9 — I/O convenience methods

```java
// InputStream.transferTo — copy stream in one call
input.transferTo(output);  // replaces manual read/write loop

// Try-with-resources on effectively final variables
Connection conn = getConnection();
try (conn) {                 // no re-declaration needed (Java 9+)
    use(conn);
}
```

#### Java 9 — ProcessHandle API

Inspect and manage OS processes without `Runtime.exec()`.

```java
ProcessHandle ph = ProcessHandle.current();
long pid = ph.pid();
ph.info().command().ifPresent(System.out::println);
ph.children().forEach(c -> System.out.println(c.pid()));

// Async notification when process exits
process.toHandle().onExit().thenAccept(p -> System.out.println("Exited: " + p.pid()));
```

#### Java 9 — Objects.requireNonNullElse

```java
// Old: ternary null check
String name = input != null ? input : "default";

// Modern: clear intent, default also checked for null
String name = Objects.requireNonNullElse(input, "default");

// Lazy default with supplier
String name = Objects.requireNonNullElseGet(input, this::computeDefault);
```

#### Java 9 — Private interface methods

Extract shared logic between default methods without exposing it.

```java
interface Logger {
    private String format(String level, String msg) {
        return "[" + level + "] " + Instant.now() + " " + msg;
    }
    default void info(String msg) { System.out.println(format("INFO", msg)); }
    default void warn(String msg) { System.out.println(format("WARN", msg)); }
}
```

#### Java 10 — Optional.orElseThrow() no-arg

```java
// Old: get() hides the risk of NoSuchElementException
String value = optional.get();

// Modern: explicit intent that absence is unexpected
String value = optional.orElseThrow();
```

Prefer `orElseThrow()` over `get()` — it communicates intent and satisfies static analysis tools.

#### Java 11 — Predicate.not()

```java
// Old: lambda wrapper to negate a method reference
list.stream().filter(s -> !s.isBlank()).toList();

// Modern: negate method references directly
list.stream().filter(Predicate.not(String::isBlank)).toList();
```

#### Java 11 — Path.of() factory

```java
// Old: separate Paths utility class
Path path = Paths.get("src", "main", "java");

// Modern: factory on Path itself, consistent with List.of(), Set.of()
Path path = Path.of("src", "main", "java");
```

#### Java 11 — HttpClient

Replace `HttpURLConnection` and third-party HTTP clients for simple use cases.

```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.example.com/data"))
        .header("Accept", "application/json")
        .GET()
        .build();

HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.body());

// Async
client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
        .thenApply(HttpResponse::body)
        .thenAccept(System.out::println);
```

#### Java 11 — Files convenience methods

```java
// Read/write strings directly
String content = Files.readString(Path.of("file.txt"));
Files.writeString(Path.of("output.txt"), content);

// With charset
String content = Files.readString(path, StandardCharsets.UTF_8);
```

#### Java 12 — String and Collectors

```java
// String.indent and transform
"hello".indent(4)                    // "    hello\n"
"42".transform(Integer::parseInt)    // 42

// Collectors.teeing — combine two collectors
var result = stream.collect(Collectors.teeing(
        Collectors.counting(),
        Collectors.summingInt(Item::price),
        (count, sum) -> new Summary(count, sum)
));
```

#### Java 12 — Files.mismatch()

```java
// Old: load both files into memory and compare
byte[] f1 = Files.readAllBytes(path1);
byte[] f2 = Files.readAllBytes(path2);
boolean equal = Arrays.equals(f1, f2);

// Modern: memory-efficient, returns position of first difference
long pos = Files.mismatch(path1, path2);
// -1 if identical, otherwise byte position of first difference
```

#### Java 14 — Helpful NullPointerExceptions

No code change needed — just run on Java 14+. The JVM automatically provides detailed NPE messages:

```
// Old: "NullPointerException" — which variable was null?!
// Modern: "Cannot invoke String.length() because user.address().city() is null"
```

The exact null variable is identified in chained method calls. Enabled by default since Java 14.

#### Java 15 — String.formatted()

```java
// Old: static method, template is an argument
String msg = String.format("Hello %s, you are %d", name, age);

// Modern: instance method on the template string itself
String msg = "Hello %s, you are %d".formatted(name, age);
```

Reads more naturally in left-to-right flow and is chainable with other string methods.

#### Java 16 — Static members in inner classes

Since Java 16, non-static inner classes can declare static fields, methods, and nested types.

```java
class Outer {
    class Inner {
        static int instanceCount = 0;  // allowed since Java 16
        Inner() { instanceCount++; }
    }
}
```

Previously, only static nested classes could contain static members.

#### Java 16 — Stream.toList()

```java
// Old
List<String> names = stream.collect(Collectors.toList());

// Modern — returns unmodifiable list
List<String> names = stream.toList();
```

Note: `Stream.toList()` returns an unmodifiable list (unlike `Collectors.toList()`). Use `Collectors.toCollection(ArrayList::new)` if you need a mutable list.

#### Java 16 — mapMulti

Replace `flatMap` with `mapMulti` when the mapping is conditional or produces few elements.

```java
// Old: flatMap creates intermediate streams
stream.flatMap(x -> x.isValid() ? Stream.of(x.value()) : Stream.empty());

// Modern: mapMulti avoids stream overhead
stream.<String>mapMulti((x, consumer) -> {
    if (x.isValid()) {
        consumer.accept(x.value());
    }
});
```

#### Java 17 — HexFormat

```java
// Old
String hex = String.format("%02x", byteValue);
byte[] bytes = DatatypeConverter.parseHexBinary(hexString); // removed in Java 11

// Modern
HexFormat hex = HexFormat.of();
String hexString = hex.formatHex(byteArray);
byte[] bytes = hex.parseHex(hexString);

// With delimiters
HexFormat.ofDelimiter(":").formatHex(macAddress);  // "aa:bb:cc:dd:ee:ff"
```

#### Java 18 — Simple web server (jwebserver)

`jwebserver` CLI tool for quick static file serving during development. Also available programmatically:

```java
var server = SimpleFileServer.createFileServer(
        new InetSocketAddress(8080),
        Path.of("/var/www"),
        SimpleFileServer.OutputLevel.INFO
);
server.start();
```

#### Java 21 — Structured concurrency

Manage concurrent subtasks as a unit — if one fails, the others are cancelled.

```java
// Old: manual CompletableFuture management, error-prone cleanup
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> fetchUser(id));
CompletableFuture<List<Order>> ordersFuture = CompletableFuture.supplyAsync(() -> fetchOrders(id));
// If fetchOrders fails, fetchUser keeps running...

// Modern (Java 25 — stable)
try (var scope = StructuredTaskScope.open()) {
    Subtask<User> user = scope.fork(() -> fetchUser(id));
    Subtask<List<Order>> orders = scope.fork(() -> fetchOrders(id));
    scope.join();
    return new UserProfile(user.get(), orders.get());
}
// All subtasks complete or are cancelled together
```

With failure handling policies:

```java
// Fail on first error (ShutdownOnFailure replacement via Joiner)
try (var scope = StructuredTaskScope.open(Joiner.awaitAllSuccessfulOrThrow())) {
    Subtask<User> user = scope.fork(() -> fetchUser(id));
    Subtask<List<Order>> orders = scope.fork(() -> fetchOrders(id));
    scope.join();
    return new UserProfile(user.get(), orders.get());
}

// Race: return first successful result
try (var scope = StructuredTaskScope.open(Joiner.anySuccessfulResultOrThrow())) {
    scope.fork(() -> fetchFromPrimary(id));
    scope.fork(() -> fetchFromMirror(id));
    return scope.join();  // first successful result
}
```

#### Java 21 — Scoped values

Replace `ThreadLocal` with `ScopedValue` for sharing immutable data across call stacks, especially with virtual threads.

```java
// Old: ThreadLocal leaks, mutable, must be cleaned up
static final ThreadLocal<User> CURRENT_USER = new ThreadLocal<>();
CURRENT_USER.set(user);
try {
    process();
} finally {
    CURRENT_USER.remove();
}

// Modern (Java 25 — stable)
static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

ScopedValue.runWhere(CURRENT_USER, user, () -> {
    process();
});
// Automatically scoped, no cleanup needed, immutable within scope

// Reading
User user = CURRENT_USER.get();  // throws if not bound
CURRENT_USER.orElse(defaultUser);
```

Advantages over ThreadLocal: no memory leaks, immutable bindings, works naturally with virtual threads and structured concurrency, inheritable by child scopes.

#### Java 22 — Stream gatherers

Create custom intermediate stream operations with `Stream.gather()`.

```java
// Built-in gatherers (Java 25 — stable, from java.util.stream.Gatherers)

// Fixed-size windows
stream.gather(Gatherers.windowFixed(3))
      .forEach(window -> System.out.println(window));
// [1,2,3], [4,5,6], [7,8,9]

// Sliding windows
stream.gather(Gatherers.windowSliding(3))
      .forEach(window -> System.out.println(window));
// [1,2,3], [2,3,4], [3,4,5]

// Fold (stateful reduction as intermediate operation)
stream.gather(Gatherers.fold(() -> 0, Integer::sum));

// Scan (running accumulation)
stream.gather(Gatherers.scan(() -> 0, Integer::sum))
      .toList();
// [1, 3, 6, 10, 15] for input [1, 2, 3, 4, 5]

// mapConcurrent — process elements concurrently with virtual threads
stream.gather(Gatherers.mapConcurrent(10, this::fetchFromApi))
      .toList();
```

Custom gatherer example:

```java
// Distinct by key (not available in standard Stream API)
static <T, K> Gatherer<T, ?, T> distinctBy(Function<T, K> keyExtractor) {
    return Gatherer.ofSequential(
            HashSet::new,
            (seen, element, downstream) -> {
                if (seen.add(keyExtractor.apply(element))) {
                    return downstream.push(element);
                }
                return true;
            }
    );
}

// Usage
users.stream()
     .gather(distinctBy(User::email))
     .toList();
```

#### Java 21 — Other notable additions

```java
// String: StringBuilder and StringBuffer indexOf with range
// Character: emoji support
Character.isEmoji('😀');       // true
Character.isEmojiComponent(c); // true for skin tone modifiers etc.

// Math: clamp
Math.clamp(value, min, max);   // instead of Math.max(min, Math.min(max, value))

// Collections: unmodifiable sequenced views
Collections.unmodifiableSequencedCollection(collection);
Collections.unmodifiableSequencedSet(set);
Collections.unmodifiableSequencedMap(map);

// Repeat for StringBuilder
new StringBuilder().repeat("abc", 3);  // "abcabcabc"
```

#### Java 22 — Class-File API

For tools and frameworks that generate bytecode, `java.lang.classfile` replaces ASM/Byte Buddy for standard bytecode generation.

#### Java 15 — EdDSA digital signatures

Use the Edwards-Curve Digital Signature Algorithm (EdDSA) for modern, high-performance digital signatures.

```java
// Generate Ed25519 key pair
KeyPairGenerator kpg = KeyPairGenerator.getInstance("Ed25519");
KeyPair kp = kpg.generateKeyPair();

// Sign
Signature sig = Signature.getInstance("Ed25519");
sig.initSign(kp.getPrivate());
sig.update(message);
byte[] signature = sig.sign();

// Verify
sig.initVerify(kp.getPublic());
sig.update(message);
boolean valid = sig.verify(signature);
```

EdDSA offers better performance and security than ECDSA at the same security level. Prefer Ed25519 for new signature use cases.

#### Java 17 — Enhanced pseudo-random number generators

Use the `RandomGenerator` interface and `RandomGeneratorFactory` instead of directly instantiating `Random` or `ThreadLocalRandom`.

```java
// Old: limited to specific classes
Random random = new Random();
ThreadLocalRandom tlr = ThreadLocalRandom.current();

// Modern: uniform interface, interchangeable algorithms
RandomGenerator rng = RandomGenerator.getDefault();
RandomGenerator rng = RandomGenerator.of("L64X128MixRandom");

// Splittable generators for parallel streams
RandomGenerator.SplittableGenerator splittable =
        RandomGenerator.SplittableGenerator.of("L64X128MixRandom");
splittable.splits(10).forEach(splitRng -> {
    // each split has independent, non-overlapping sequences
});

// Stream of random numbers
rng.ints(100, 0, 1000).forEach(System.out::println);

// List available algorithms
RandomGeneratorFactory.all()
        .map(RandomGeneratorFactory::name)
        .sorted()
        .forEach(System.out::println);
```

New algorithm families: LXM algorithms (`L64X128MixRandom`, `L128X256MixRandom`, etc.) and Xoshiro/Xoroshiro variants. These are higher quality than the legacy `Random` algorithm.

#### Java 17 — Deserialization filters

Configure context-specific deserialization filters to protect against deserialization attacks.

```java
// Set a JVM-wide deserialization filter factory
ObjectInputFilter.Config.setSerialFilterFactory((current, next) -> {
    // Allow only specific classes
    return ObjectInputFilter.Config.createFilter(
            "com.example.*;java.base/*;!*");
});

// Per-stream filter
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(ObjectInputFilter.Config.createFilter(
        "com.example.dto.*;!*"));
```

Always configure deserialization filters when accepting serialized data from untrusted sources.

#### Java 18 — Internet-Address Resolution SPI

The `InetAddress` API now supports pluggable DNS resolvers via `java.net.spi.InetAddressResolverProvider`. Useful for testing (mock DNS) and custom resolution (DNS over HTTPS).

```java
// Custom resolver for testing
public class MockResolverProvider extends InetAddressResolverProvider {
    @Override
    public InetAddressResolver get(Configuration configuration) {
        return new MockResolver();
    }
    @Override
    public String name() { return "Mock DNS Resolver"; }
}
// Register via ServiceLoader (META-INF/services or module-info.java)
```

#### Java 21 — Key Encapsulation Mechanism API (KEM)

Use `javax.crypto.KEM` for modern key exchange instead of traditional RSA encryption with padding.

```java
// Generate key pair
KeyPairGenerator g = KeyPairGenerator.getInstance("DHKEM");
KeyPair kp = g.generateKeyPair();

// Sender: encapsulate
KEM kemS = KEM.getInstance("DHKEM");
KEM.Encapsulator enc = kemS.newEncapsulator(kp.getPublic());
KEM.Encapsulated encapsulated = enc.encapsulate();
SecretKey sharedSecretS = encapsulated.key();
byte[] keyEncapsulationMessage = encapsulated.encapsulation();

// Receiver: decapsulate
KEM kemR = KEM.getInstance("DHKEM");
KEM.Decapsulator dec = kemR.newDecapsulator(kp.getPrivate());
SecretKey sharedSecretR = dec.decapsulate(keyEncapsulationMessage);
// sharedSecretS and sharedSecretR are the same
```

#### Java 22 — Foreign Function & Memory API

Replace JNI with the `java.lang.foreign` API for calling native functions and managing off-heap memory.

```java
// Allocate and use off-heap memory with deterministic deallocation
try (Arena arena = Arena.ofConfined()) {
    // Allocate a native memory segment
    MemorySegment segment = arena.allocate(100);
    segment.set(ValueLayout.JAVA_INT, 0, 42);
    int value = segment.get(ValueLayout.JAVA_INT, 0);

    // Allocate a C string
    MemorySegment cString = arena.allocateFrom("Hello");
}
// Memory is deterministically deallocated here

// Call a native C function (e.g., strlen)
Linker linker = Linker.nativeLinker();
SymbolLookup stdlib = linker.defaultLookup();
MethodHandle strlen = linker.downcallHandle(
        stdlib.find("strlen").orElseThrow(),
        FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS)
);

try (Arena arena = Arena.ofConfined()) {
    MemorySegment cString = arena.allocateFrom("Hello");
    long len = (long) strlen.invoke(cString);  // 5
}
```

Arena types: `Arena.ofConfined()` (single-thread, deterministic), `Arena.ofShared()` (multi-thread), `Arena.ofAuto()` (GC-managed), `Arena.global()` (never deallocated).

#### Java 24 — Quantum-resistant cryptography (ML-KEM, ML-DSA)

Use NIST-standardized post-quantum algorithms for key exchange and digital signatures.

```java
// ML-KEM: quantum-resistant key encapsulation (FIPS 203)
KeyPairGenerator g = KeyPairGenerator.getInstance("ML-KEM");
g.initialize(NamedParameterSpec.ML_KEM_768); // or ML_KEM_512, ML_KEM_1024
KeyPair kp = g.generateKeyPair();

KEM kem = KEM.getInstance("ML-KEM");
KEM.Encapsulator enc = kem.newEncapsulator(kp.getPublic());
KEM.Encapsulated encapsulated = enc.encapsulate();
// Use encapsulated.key() as shared secret

// ML-DSA: quantum-resistant digital signatures (FIPS 204)
KeyPairGenerator dsa = KeyPairGenerator.getInstance("ML-DSA");
dsa.initialize(NamedParameterSpec.ML_DSA_65); // or ML_DSA_44, ML_DSA_87
KeyPair dsaKp = dsa.generateKeyPair();

Signature sig = Signature.getInstance("ML-DSA");
sig.initSign(dsaKp.getPrivate());
sig.update(message);
byte[] signature = sig.sign();
```

Start planning migration to post-quantum algorithms now — an adversary can harvest encrypted data today and decrypt it when quantum computers are available.

#### Java 24 — Deprecation of sun.misc.Unsafe memory access

`sun.misc.Unsafe` memory-access methods are deprecated for removal. Use the Foreign Function & Memory API (`java.lang.foreign`) or `VarHandle` instead.

#### Java 25 — Key Derivation Function API (KDF)

Use `javax.crypto.KDF` for deriving cryptographic keys from shared secrets, replacing manual HKDF implementations.

```java
// HKDF: Extract-and-Expand
KDF hkdf = KDF.getInstance("HKDF-SHA256");

AlgorithmParameterSpec params =
        HKDFParameterSpec.ofExtract()
                .addIKM(initialKeyMaterial)
                .addSalt(salt)
                .thenExpand(info, 32);

SecretKey derivedKey = hkdf.deriveKey("AES", params);
```

#### Java 25 — PEM encoding/decoding of cryptographic objects (Preview)

Encode and decode cryptographic keys and certificates in PEM format without manual Base64/DER handling.

```java
// Old: manual Base64 encoding, tedious parsing
String pem = "-----BEGIN PUBLIC KEY-----\n" +
    Base64.getMimeEncoder().encodeToString(publicKey.getEncoded()) +
    "\n-----END PUBLIC KEY-----";

// Modern (Java 25, preview)
PEMEncoder encoder = PEMEncoder.of();
String pem = encoder.encodeToString(publicKey);

// Decode
PEMDecoder decoder = PEMDecoder.of();
PublicKey key = decoder.decode(pemString, PublicKey.class);

// Encrypt private key with password
String encryptedPem = encoder.withEncryption(password).encodeToString(privateKey);
```

### 4. Tooling improvements

#### Java 9 — JShell REPL

Test expressions interactively without creating files or a main method.

```
$ jshell
jshell> "hello".chars().count()
$1 ==> 5
jshell> List.of(1,2,3).reversed()
$2 ==> [3, 2, 1]
```

#### Java 11 — Single-file execution

Run single-file programs directly without explicit `javac` step:

```
$ java HelloWorld.java
```

Also supports Unix shebang lines (`#!/usr/bin/java --source 25`) for script-like usage.

#### Java 22 — Multi-file source launcher

Launch multi-file programs without a build tool — referenced classes are found and compiled automatically:

```
$ java Main.java
# automatically compiles classes referenced by Main.java
```

#### Java 9/11 — Java Flight Recorder (JFR)

Low-overhead profiling built into the JVM (~1% performance impact, safe for production):

```
$ java -XX:StartFlightRecording=filename=rec.jfr MyApp
$ jcmd <pid> JFR.start   # attach to running app
```

Captures CPU, memory, GC, I/O, thread, and lock events without external tools.

#### Java 25 — AOT class preloading

Cache class loading from a training run for faster startup:

```
# Training run
$ java -XX:AOTCacheOutput=app.aot -cp app.jar com.App
# Production — faster startup
$ java -XX:AOTCache=app.aot -cp app.jar com.App
```

#### Java 25 — Compact object headers

Enable with `-XX:+UseCompactObjectHeaders` to reduce per-object header overhead from 16 bytes to 8 bytes (50% reduction), improving memory density and cache utilization.

### 5. Migration patterns — old to modern

Always apply these substitutions when you encounter legacy patterns:

| Legacy pattern | Modern replacement | Since |
|---|---|---|
| `instanceof` + explicit cast | Pattern matching `instanceof` | 16 |
| if/else instanceof chains | Pattern matching `switch` | 21 |
| Switch statement with break | Switch expression with `->` | 14 |
| String concatenation (multi-line) | Text blocks `"""` | 15 |
| POJO with getters/equals/hashCode | `record` | 16 |
| `list.get(list.size()-1)` | `list.getLast()` | 21 |
| `stream.collect(Collectors.toList())` | `stream.toList()` | 16 |
| `"".trim()` | `"".strip()` | 11 |
| `"".trim().isEmpty()` | `"".isBlank()` | 11 |
| `new BufferedReader(new InputStreamReader(...)).lines()` | `Files.readString(path)` | 11 |
| `HttpURLConnection` | `HttpClient` | 11 |
| `DatatypeConverter.parseHexBinary` | `HexFormat` | 17 |
| `Math.max(min, Math.min(max, val))` | `Math.clamp(val, min, max)` | 21 |
| `ThreadLocal` for request-scoped data | `ScopedValue` | 25 |
| Thread pools for I/O concurrency | Virtual threads | 21 |
| `CompletableFuture` for concurrent subtasks | `StructuredTaskScope` | 25 |
| `flatMap` for simple conditional mapping | `mapMulti` | 16 |
| Unused catch variable `ignored` | Unnamed variable `_` | 22 |
| Validation after `super()` call | Pre-super validation | 24 |
| `/** */` with HTML in Javadoc | `///` with Markdown | 23 |
| `<pre>{@code ...}</pre>` in Javadoc | `{@snippet ...}` | 18 |
| `new Random()` | `RandomGenerator.of(...)` | 17 |
| JNI for native function calls | Foreign Function & Memory API | 22 |
| `sun.misc.Unsafe` for off-heap memory | `Arena` + `MemorySegment` | 22 |
| ECDSA for new signatures | EdDSA (Ed25519) | 15 |
| RSA encryption for key exchange | KEM API (`javax.crypto.KEM`) | 21 |
| Classical key exchange (RSA, DH) | ML-KEM (post-quantum) | 24 |
| Manual PEM Base64 encoding | `PEMEncoder`/`PEMDecoder` | 25 (preview) |
| Manual HKDF implementation | `KDF` API (`javax.crypto.KDF`) | 25 |
| `input != null ? input : default` | `Objects.requireNonNullElse()` | 9 |
| `optional.get()` | `optional.orElseThrow()` | 10 |
| `s -> !s.isBlank()` in filter | `Predicate.not(String::isBlank)` | 11 |
| `Paths.get(...)` | `Path.of(...)` | 11 |
| `String.format(template, args)` | `template.formatted(args)` | 15 |
| Manual read/write loop for streams | `InputStream.transferTo()` | 9 |
| Manual byte comparison of files | `Files.mismatch()` | 12 |
| `Stream.iterate(seed, op).limit(n)` | `Stream.iterate(seed, pred, op)` | 9 |
| Ternary for nullable stream | `Stream.ofNullable()` | 9 |
| Manual loop to take prefix | `Stream.takeWhile()` / `dropWhile()` | 9 |
| `isPresent()` + `get()` anti-pattern | `ifPresentOrElse()` / `or()` | 9 |
| Manual executor shutdown in finally | `try (var exec = ...)` (AutoCloseable) | 19 |
| `Thread.sleep(5000)` | `Thread.sleep(Duration.ofSeconds(5))` | 19 |
| `javac` + `java` two-step | `java File.java` single-file execution | 11 |
| `javax.xml.bind.*` (JAXB) | `jakarta.xml.bind:jakarta.xml.bind-api` | 11 (removed) |
| `javax.xml.ws.*` (JAX-WS) | `jakarta.xml.ws:jakarta.xml.ws-api` | 11 (removed) |
| `javax.activation` (JAF) | `jakarta.activation:jakarta.activation-api` | 11 (removed) |
| `SecurityManager` / `AccessController` | Framework-level auth (Spring Security) | 17 (dep. for removal) |
| `Object.finalize()` | `try-with-resources` or `Cleaner` | 18 (dep. for removal) |
| `Thread.stop()` / `suspend()` / `resume()` | `Thread.interrupt()` + cooperative shutdown | removed |
| `sun.misc.Unsafe` memory access | FFM API (`MemorySegment`) or `VarHandle` | 24 (dep. for removal) |
| `javax.security.cert.X509Certificate` | `java.security.cert.X509Certificate` | removed |
| `Pack200` | `jlink` or standard compression | 14 (removed) |
| `new Integer(42)` / `new Double(3.14)` etc. | `Integer.valueOf(42)` / `Double.valueOf(3.14)` | 9 (dep.) |
| `new URL("...")` | `URI.create("...")` / `URL.of(uri, null)` | 20 (dep.) |
| `new Locale("de", "DE")` | `Locale.of("de", "DE")` | 19 (dep.) |
| `Class.newInstance()` | `getDeclaredConstructor().newInstance()` | 9 (dep.) |
| `URLDecoder.decode(str)` (no charset) | `URLDecoder.decode(str, UTF_8)` | 10 |
| `Runtime.exec(String)` | `ProcessBuilder` | best practice |
| `Collections.unmodifiableList(new ArrayList<>(...))` | `List.of(...)` / `List.copyOf(...)` | 9/10 |
| `DatatypeConverter.printBase64Binary()` | `Base64.getEncoder().encodeToString()` | 11 (removed) |
| `@Deprecated` (no details) | `@Deprecated(since="...", forRemoval=true)` | 9 |

### 6. Usage rules

1. **Always check the project's Java version** before using any feature. Do not use Java 21 features in a Java 17 project.
2. **Prefer the modern pattern** whenever the project version supports it. Do not write legacy-style code on modern Java.
3. **Do not mix styles** — if a file uses switch expressions, do not add a switch statement. Keep the style consistent.
4. **Records replace data classes** — any class whose sole purpose is carrying data should be a record (unless it needs mutability or inheritance).
5. **Text blocks for all multi-line strings** — SQL, JSON, HTML, XML, log messages, error messages.
6. **Pattern matching first** — use `instanceof` patterns and switch patterns instead of explicit casts and if/else chains.
7. **Virtual threads for I/O** — when writing concurrent I/O code on Java 21+, default to virtual threads.
8. **Sequenced collection methods** — always use `getFirst()`/`getLast()` instead of index-based access for first/last elements on Java 21+.
9. **`Stream.toList()`** — always use instead of `Collectors.toList()` on Java 16+ unless a mutable list is needed.
10. **`strip()` over `trim()`** — always prefer `strip()` on Java 11+.
11. **Markdown Javadoc** — on Java 23+, write new documentation comments using `///` with Markdown syntax instead of `/** */` with HTML.
12. **`@snippet` over `<pre>{@code}`** — on Java 18+, use `{@snippet}` for code examples in documentation.
13. **`RandomGenerator` over `Random`** — on Java 17+, use `RandomGenerator.of()` for new code. Use LXM algorithms for better quality.
14. **Foreign Function & Memory API over JNI** — on Java 22+, use `Arena`, `MemorySegment`, and `Linker` instead of JNI for native interop.
15. **Post-quantum cryptography** — on Java 24+, evaluate ML-KEM and ML-DSA for new security-sensitive applications.

### 7. Deprecated constructors and methods — use modern replacements

Many common constructors and methods have been deprecated in favor of better alternatives. Always use the modern replacement.

#### Wrapper type constructors — use valueOf() or parse methods

All wrapper type constructors are deprecated since Java 9. Use static factory methods instead.

```java
// DEPRECATED — never use wrapper constructors
new Integer(42);          new Integer("42");
new Long(42L);            new Long("42");
new Double(3.14);         new Double("3.14");
new Float(3.14f);         new Float("3.14");
new Boolean(true);        new Boolean("true");
new Short((short) 1);     new Byte((byte) 1);
new Character('a');

// MODERN — static factory methods (cached, no unnecessary allocation)
Integer.valueOf(42);      Integer.valueOf("42");
Long.valueOf(42L);        Long.valueOf("42");
Double.valueOf(3.14);     Double.valueOf("3.14");
Float.valueOf(3.14f);     Float.valueOf("3.14");
Boolean.valueOf(true);    Boolean.valueOf("true");
Short.valueOf((short) 1); Byte.valueOf((byte) 1);
Character.valueOf('a');

// For parsing strings to primitives, prefer parse methods:
int n = Integer.parseInt("42");
double d = Double.parseDouble("3.14");
boolean b = Boolean.parseBoolean("true");
```

`valueOf()` uses caching (e.g., `Integer.valueOf()` caches values -128 to 127), avoiding unnecessary object creation.

#### URL constructor — use URI.create() or URL.of()

```java
// DEPRECATED (Java 20) — throws checked MalformedURLException
URL url = new URL("https://example.com");

// MODERN — use URI first, convert to URL when needed
URI uri = URI.create("https://example.com");
URL url = uri.toURL();

// Or use URL.of() (Java 20+)
URL url = URL.of(URI.create("https://example.com"), null);
```

`URI` is the preferred type for representing resource identifiers. Convert to `URL` only when an API requires it.

#### Date/Calendar constructors — use java.time

```java
// DEPRECATED — mutable, confusing API
new Date(year, month, day);       // year is offset from 1900!
Calendar.getInstance();

// MODERN — immutable, clear API
LocalDate.of(2025, Month.JANUARY, 15);
LocalDateTime.now();
Instant.now();
ZonedDateTime.now(ZoneId.of("Europe/Berlin"));
```

#### String constructors for encoding

```java
// DEPRECATED
new String(bytes, 0);  // deprecated hi-byte constructor

// MODERN — always specify charset explicitly
new String(bytes, StandardCharsets.UTF_8);
```

#### Class.newInstance() — use Constructor.newInstance()

```java
// DEPRECATED (Java 9) — swallows checked exceptions
Object obj = MyClass.class.newInstance();

// MODERN — proper exception handling
Object obj = MyClass.class.getDeclaredConstructor().newInstance();
```

#### Locale constructor — use Locale.of()

```java
// DEPRECATED (Java 19)
new Locale("de");
new Locale("de", "DE");
new Locale("de", "DE", "POSIX");

// MODERN — factory method
Locale.of("de");
Locale.of("de", "DE");
Locale.of("de", "DE", "POSIX");
```

#### URLDecoder/URLEncoder — specify charset

```java
// DEPRECATED — uses platform default encoding
URLDecoder.decode(str);
URLEncoder.encode(str);

// MODERN — always specify charset (Java 10+)
URLDecoder.decode(str, StandardCharsets.UTF_8);
URLEncoder.encode(str, StandardCharsets.UTF_8);
```

#### Runtime.exec(String) — use ProcessBuilder

```java
// DEPRECATED pattern — fragile string splitting, no control
Process p = Runtime.getRuntime().exec("ls -la /tmp");

// MODERN — explicit arguments, configurable environment
Process p = new ProcessBuilder("ls", "-la", "/tmp")
        .directory(Path.of("/tmp").toFile())
        .redirectErrorStream(true)
        .start();
```

#### Thread constructors — use Thread.Builder (Java 21+)

```java
// Old: limited constructor options
Thread t = new Thread(runnable, "worker-1");
t.setDaemon(true);
t.start();

// Modern: fluent builder, supports virtual threads
Thread t = Thread.ofPlatform()
        .name("worker-1")
        .daemon(true)
        .start(runnable);

// Virtual thread
Thread t = Thread.ofVirtual()
        .name("handler-", 0)
        .start(runnable);
```

#### Collections.unmodifiable* — prefer List/Set/Map.of() and .copyOf()

```java
// Old: mutable collection wrapped as unmodifiable
List<String> list = Collections.unmodifiableList(new ArrayList<>(Arrays.asList("a", "b")));
Set<String> set = Collections.unmodifiableSet(new HashSet<>(Arrays.asList("a", "b")));
Map<String, Integer> map = Collections.unmodifiableMap(new HashMap<>() {{ put("a", 1); }});

// Modern: truly immutable from creation (Java 9+)
List<String> list = List.of("a", "b");
Set<String> set = Set.of("a", "b");
Map<String, Integer> map = Map.of("a", 1, "b", 2);

// Copy existing collection to immutable (Java 10+)
List<String> copy = List.copyOf(existingList);
Set<String> copy = Set.copyOf(existingSet);
Map<K, V> copy = Map.copyOf(existingMap);

// For larger maps (>10 entries)
Map<String, Integer> map = Map.ofEntries(
        Map.entry("a", 1),
        Map.entry("b", 2),
        Map.entry("c", 3)
);
```

#### Collections.synchronized* — prefer concurrent collections

```java
// Old: external synchronization wrapper
List<String> list = Collections.synchronizedList(new ArrayList<>());

// Modern: purpose-built concurrent collections
List<String> list = new CopyOnWriteArrayList<>();
Map<K, V> map = new ConcurrentHashMap<>();
Queue<T> queue = new ConcurrentLinkedQueue<>();
Deque<T> deque = new ConcurrentLinkedDeque<>();
```

#### Base64 encoding — use java.util.Base64

```java
// REMOVED (Java 11) — javax.xml.bind.DatatypeConverter
String encoded = DatatypeConverter.printBase64Binary(bytes);
byte[] decoded = DatatypeConverter.parseBase64Binary(str);

// Also deprecated: sun.misc.BASE64Encoder/BASE64Decoder

// MODERN (Java 8+)
String encoded = Base64.getEncoder().encodeToString(bytes);
byte[] decoded = Base64.getDecoder().decode(str);

// URL-safe variant
String urlSafe = Base64.getUrlEncoder().encodeToString(bytes);

// MIME variant (line-wrapped)
String mime = Base64.getMimeEncoder().encodeToString(bytes);
```

#### Logging — avoid java.util.logging for new projects

```java
// Old: JUL (java.util.logging) — verbose configuration
Logger logger = Logger.getLogger(MyClass.class.getName());
logger.log(Level.INFO, "Processing {0}", item);

// Modern for libraries: System.Logger (Java 9+) — no external dependency
System.Logger logger = System.getLogger(MyClass.class.getName());
logger.log(System.Logger.Level.INFO, "Processing {0}", item);

// Modern for applications: SLF4J
private static final org.slf4j.Logger log = LoggerFactory.getLogger(MyClass.class);
log.info("Processing {}", item);
```

#### Annotation Type changes

```java
// DEPRECATED (Java 9) — @Deprecated without information
@Deprecated
public void oldMethod() {}

// MODERN — include since and forRemoval
@Deprecated(since = "2.3", forRemoval = true)
public void oldMethod() {}
```

### 8. Removed and deprecated APIs — never use

These APIs have been removed from the JDK or are deprecated for removal. Never use them in new code. When encountering them in existing code, migrate to the listed replacement.

#### Removed modules and packages (Java 11+)

These Java EE and CORBA modules were removed in Java 11. Use standalone replacements from Maven Central.

| Removed module | Removed packages | Replacement (Maven) |
|---|---|---|
| java.xml.bind (JAXB) | `javax.xml.bind.*` | `jakarta.xml.bind:jakarta.xml.bind-api` + `org.glassfish.jaxb:jaxb-runtime` |
| java.xml.ws (JAX-WS) | `javax.xml.ws.*`, `javax.jws.*` | `jakarta.xml.ws:jakarta.xml.ws-api` + CXF or Metro |
| java.activation (JAF) | `javax.activation` | `jakarta.activation:jakarta.activation-api` |
| java.xml.ws.annotation | `javax.annotation` | `jakarta.annotation:jakarta.annotation-api` |
| java.transaction (JTA) | `javax.transaction` | `jakarta.transaction:jakarta.transaction-api` |
| java.corba (CORBA) | `org.omg.*`, `javax.rmi.*`, `javax.activity` | No direct replacement — redesign away from CORBA |

Additional removals:
- **`java.rmi.activation`** — RMI Activation (removed Java 17). No replacement needed for new code.
- **`java.security.acl`** — ACL API (removed Java 14). Use `java.security.Policy` or framework-specific authorization.
- **`javax.security.cert`** — Old certificate API. Use `java.security.cert` instead.

#### Removed classes

| Removed class | Since | Replacement |
|---|---|---|
| `java.lang.Compiler` | 12 | No replacement — JIT is handled by the JVM |
| `java.util.jar.Pack200` | 14 | Use `jlink` for modular JARs or standard compression |
| `javax.security.auth.Policy` | 12 | `java.security.Policy` (also deprecated) |
| `javax.management.remote.rmi.RMIIIOPServerImpl` | 15 | Not needed — CORBA/IIOP removed |

#### Removed methods

| Removed method | Replacement |
|---|---|
| `Thread.stop(Throwable)` | Cooperative interruption with `Thread.interrupt()` |
| `Thread.destroy()` | Not needed — use `interrupt()` and clean shutdown |
| `Thread.suspend()` / `resume()` | Use `java.util.concurrent` locks and conditions |
| `Thread.countStackFrames()` | `Thread.getStackTrace()` or `StackWalker` |
| `Runtime.runFinalizersOnExit()` | Use `try-with-resources` or `Cleaner` |
| `Runtime.traceInstructions()` / `traceMethodCalls()` | Use JFR or debugger |
| `FileInputStream.finalize()` / `FileOutputStream.finalize()` | Use `try-with-resources` |
| All `finalize()` overrides in JDK classes | Use `try-with-resources` or `java.lang.ref.Cleaner` |

#### Deprecated for removal — do not use in new code

**SecurityManager ecosystem** (deprecated for removal since Java 17):

The entire Security Manager is being removed. Do not use any of these in new code:
- `java.lang.SecurityManager`
- `System.getSecurityManager()` / `System.setSecurityManager()`
- `java.security.AccessController`, `AccessControlContext`, `AccessControlException`
- `java.security.Policy`, `PolicySpi`
- All `*Permission` classes (`RuntimePermission`, `FilePermission`, `NetPermission`, `PropertyPermission`, etc.)
- `javax.security.auth.Subject.doAs()` / `doAsPrivileged()` / `getSubject()`
- `java.security.DomainCombiner`, `SubjectDomainCombiner`

Use framework-level authorization (Spring Security, etc.) instead.

**Finalization** (deprecated for removal since Java 18):

- `Object.finalize()` — never override this method
- `System.runFinalization()` / `Runtime.runFinalization()`

Use `try-with-resources` for resource cleanup. For exotic cases, use `java.lang.ref.Cleaner`.

**Applet API** (deprecated for removal):

- `java.applet.Applet`, `AppletContext`, `AppletStub`, `AudioClip`
- `javax.swing.JApplet`

Applets are dead. Use web technologies or desktop frameworks (JavaFX, Swing without Applet).

**Legacy threading** (deprecated for removal):

- `Thread.stop()` — use cooperative interruption with `Thread.interrupt()` + checking `Thread.interrupted()`
- `Thread.checkAccess()` / `ThreadGroup.checkAccess()` — part of SecurityManager, being removed
- `ThreadGroup.destroy()` / `ThreadGroup.isDaemon()` / `ThreadGroup.setDaemon()`
- `ThreadDeath` — no longer thrown

**Legacy concurrency** (deprecated for removal):

- `Executors.privilegedCallable()`, `privilegedCallableUsingCurrentClassLoader()`, `privilegedThreadFactory()` — part of SecurityManager ecosystem

**Legacy security certificates**:

- `javax.security.cert.X509Certificate` — use `java.security.cert.X509Certificate`
- `SSLSession.getPeerCertificateChain()` — use `getPeerCertificates()`

**Other deprecated-for-removal**:

- `java.util.zip.ZipError` — use `ZipException`
- `java.rmi.RMISecurityManager` — SecurityManager removal
- `java.beans.beancontext.*` — entire BeanContext API (14 classes)
- `java.beans.AppletInitializer`
- `java.security.Identity`, `IdentityScope`, `Signer` — legacy identity model
- `java.security.Certificate` (interface) — use `java.security.cert.Certificate`

#### sun.misc.Unsafe

`sun.misc.Unsafe` memory-access methods are deprecated for removal (Java 24). Use:
- `java.lang.foreign.MemorySegment` and `Arena` for off-heap memory
- `java.lang.invoke.VarHandle` for atomic/volatile field access
- `java.lang.ref.Cleaner` for cleanup actions

### 9. Review checklist

When reviewing Java code, check for these modernization opportunities:

- [ ] Are `instanceof` checks followed by explicit casts? (Use pattern matching)
- [ ] Are there multi-way `if/else instanceof` chains? (Use pattern matching switch)
- [ ] Are switch statements used where switch expressions would work?
- [ ] Are multi-line strings using concatenation instead of text blocks?
- [ ] Are there data-carrier classes that should be records?
- [ ] Is `Collectors.toList()` used instead of `Stream.toList()`?
- [ ] Is `trim()` used instead of `strip()`?
- [ ] Are first/last element accesses using index arithmetic instead of `getFirst()`/`getLast()`?
- [ ] Are `ThreadLocal`s used where `ScopedValue` would be better? (Java 25+)
- [ ] Is thread pool-based concurrency used for I/O instead of virtual threads? (Java 21+)
- [ ] Are `CompletableFuture` patterns used where structured concurrency would be cleaner? (Java 25+)
- [ ] Are there unused variables that should use `_`? (Java 22+)
- [ ] Is `HttpURLConnection` used instead of `HttpClient`? (Java 11+)
- [ ] Are byte arrays formatted to hex manually instead of using `HexFormat`? (Java 17+)
- [ ] Are sealed types applicable for known type hierarchies? (Java 17+)
- [ ] Are Javadoc comments using HTML where `///` Markdown comments would be cleaner? (Java 23+)
- [ ] Are `<pre>{@code}</pre>` blocks used where `{@snippet}` would be better? (Java 18+)
- [ ] Is `new Random()` used where `RandomGenerator` would be more appropriate? (Java 17+)
- [ ] Is JNI used where the Foreign Function & Memory API could replace it? (Java 22+)
- [ ] Are deserialization filters configured when accepting untrusted serialized data? (Java 17+)
- [ ] Are classical (non-quantum-resistant) algorithms used where ML-KEM/ML-DSA should be considered? (Java 24+)
- [ ] Is `synchronized` avoided unnecessarily for virtual thread compatibility? (No longer needed on Java 24+ — pinning is fixed)
- [ ] Is `Paths.get()` used instead of `Path.of()`? (Java 11+)
- [ ] Is `String.format()` used where `String.formatted()` would be cleaner? (Java 15+)
- [ ] Is `optional.get()` used instead of `optional.orElseThrow()`? (Java 10+)
- [ ] Are manual stream copy loops used instead of `InputStream.transferTo()`? (Java 9+)
- [ ] Is `Predicate.not()` applicable for negated method references? (Java 11+)
- [ ] Are `Optional.or()` / `ifPresentOrElse()` applicable instead of `isPresent()` + `get()`? (Java 9+)
- [ ] Is `Stream.takeWhile()`/`dropWhile()` applicable instead of manual loops? (Java 9+)
- [ ] Are executors shut down manually instead of using try-with-resources? (Java 19+)
- [ ] Is `Thread.sleep(long)` used instead of `Thread.sleep(Duration)`? (Java 19+)
- [ ] Are any removed APIs used (`javax.xml.bind`, `javax.xml.ws`, CORBA, `Pack200`, `java.lang.Compiler`)?
- [ ] Is `SecurityManager` or `AccessController` used? (deprecated for removal)
- [ ] Is `Object.finalize()` overridden? (Use `try-with-resources` or `Cleaner`)
- [ ] Is `Thread.stop()` / `suspend()` / `resume()` used? (Use cooperative interruption)
- [ ] Is `sun.misc.Unsafe` used for memory access? (Use FFM API or `VarHandle`)
- [ ] Is `javax.security.cert.X509Certificate` used instead of `java.security.cert.X509Certificate`?
- [ ] Are `Subject.doAs()` / `doAsPrivileged()` used? (deprecated for removal)
- [ ] Are wrapper type constructors used (`new Integer()`, `new Double()`)? (Use `valueOf()`)
- [ ] Is `new URL(...)` used? (Use `URI.create()` + `toURL()` or `URL.of()`)
- [ ] Is `new Locale(...)` used? (Use `Locale.of()` since Java 19)
- [ ] Is `Class.newInstance()` used? (Use `getDeclaredConstructor().newInstance()`)
- [ ] Is `URLDecoder.decode()` / `URLEncoder.encode()` used without charset? (Always specify `UTF_8`)
- [ ] Is `Runtime.exec(String)` used? (Use `ProcessBuilder`)
- [ ] Are `Collections.unmodifiable*` wrappers used? (Prefer `List.of()` / `List.copyOf()`)
- [ ] Is `DatatypeConverter` used for Base64? (Use `java.util.Base64`)
- [ ] Is `@Deprecated` used without `since` and `forRemoval`?
