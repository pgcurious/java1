# Chapter 3: Streams: Declarative Data Processing

*Lazy evaluation, internal vs external iteration, parallel streams and their hidden costs*

---

## The Question That Started Everything

Consider this simple task: from a list of transactions, find the IDs of all large transactions (over $1000), sorted by amount.

**Java 7 approach:**

```java
List<Transaction> largeTxns = new ArrayList<>();
for (Transaction txn : transactions) {
    if (txn.getAmount() > 1000) {
        largeTxns.add(txn);
    }
}
Collections.sort(largeTxns, new Comparator<Transaction>() {
    @Override
    public int compare(Transaction a, Transaction b) {
        return Double.compare(a.getAmount(), b.getAmount());
    }
});
List<String> ids = new ArrayList<>();
for (Transaction txn : largeTxns) {
    ids.add(txn.getId());
}
```

**Java 8 approach:**

```java
List<String> ids = transactions.stream()
    .filter(txn -> txn.getAmount() > 1000)
    .sorted(comparing(Transaction::getAmount))
    .map(Transaction::getId)
    .collect(toList());
```

Same result. But something fundamental changed.

The first version tells the computer **how** to iterate: create a temporary list, loop through, check each element, add to list, sort, loop again, extract IDs. You're the foreman directing every step.

The second version tells the computer **what** you want: large transactions, sorted by amount, give me their IDs. You're describing the transformation, not the execution.

This is **declarative versus imperative** programming. And it's not just about fewer lines of code.

---

## First Principles: What Is a Stream?

A stream is a **sequence of elements supporting sequential and parallel aggregate operations**. But that definition hides the revolutionary bits. Let's unpack it.

### Streams Are Not Data Structures

A `List<Transaction>` stores transactions. A `Stream<Transaction>` represents a computation on transactions. The stream doesn't hold data—it describes a pipeline of operations to be performed on data.

```
┌─────────────────────────────────────────────────────────────────────┐
│  SOURCE          INTERMEDIATE OPS          TERMINAL OP             │
│                                                                     │
│  Collection  →  [filter] → [map] → [sort] → [limit]  →  [collect]  │
│  Array                                                              │
│  Generator       ← ← ← ← Elements flow this way → → → →            │
│  I/O Channel                                                        │
└─────────────────────────────────────────────────────────────────────┘
```

### Internal vs External Iteration

**External iteration** (the traditional way): You control the iteration explicitly.

```java
Iterator<String> it = names.iterator();
while (it.hasNext()) {
    String name = it.next();
    // process name
}
```

**Internal iteration** (the stream way): The library controls iteration; you just describe operations.

```java
names.stream()
    .filter(n -> n.length() > 3)
    .forEach(n -> /* process */);
```

Why does this matter? Because when you control iteration, you're locked into sequential, one-at-a-time processing. When the library controls iteration, it can:

- Parallelize across multiple cores
- Short-circuit when results are found
- Reorder operations for efficiency
- Fuse multiple operations together

You describe what you want; the library figures out how to do it efficiently.

---

## The Stream Pipeline: Anatomy of a Query

Every stream operation falls into one of two categories:

**Intermediate operations** return a new stream. They're lazy—nothing happens until a terminal operation is invoked.

```java
stream
    .filter(...)    // intermediate
    .map(...)       // intermediate
    .sorted(...)    // intermediate
    .limit(...)     // intermediate
```

**Terminal operations** produce a result or side effect, and consume the stream.

```java
    .collect(...)   // terminal - returns a collection
    .forEach(...)   // terminal - performs side effect
    .count()        // terminal - returns long
    .findFirst()    // terminal - returns Optional
    .reduce(...)    // terminal - returns reduced value
```

A stream can only be consumed once. After a terminal operation, the stream is "spent."

```java
Stream<String> stream = names.stream();
long count = stream.count();           // OK
List<String> list = stream.collect(toList());  // IllegalStateException!
```

---

## Laziness: The Hidden Superpower

Here's code that looks wasteful but isn't:

```java
List<String> result = hugeList.stream()       // millions of elements
    .filter(s -> s.startsWith("A"))           // might filter most
    .map(String::toUpperCase)
    .limit(10)                                // only need 10
    .collect(toList());
```

How many elements does this actually process? Not millions. It processes only enough elements to find 10 that start with "A". This is **laziness**—operations are deferred until absolutely necessary.

### Short-Circuit Operations

Some operations can finish without processing all elements:

- `limit(n)` - stops after n elements
- `findFirst()` / `findAny()` - stops after finding one
- `anyMatch()` / `noneMatch()` / `allMatch()` - stops when result is determined

```java
boolean hasError = logs.stream()
    .filter(log -> log.getLevel() == ERROR)
    .findAny()
    .isPresent();

// This does NOT scan all logs. It stops at the first ERROR.
```

### The Execution Model

When you chain intermediate operations, nothing happens:

```java
Stream<String> intermediate = names.stream()
    .filter(n -> {
        System.out.println("Filtering: " + n);
        return n.length() > 3;
    })
    .map(n -> {
        System.out.println("Mapping: " + n);
        return n.toUpperCase();
    });

// Nothing printed yet! Stream is just a description.
```

Only when a terminal operation executes does the pipeline run:

```java
List<String> result = intermediate.collect(toList());
// NOW the prints happen, interleaved:
// Filtering: Alice
// Mapping: Alice
// Filtering: Bob
// Filtering: Charlie
// Mapping: Charlie
```

Notice: elements flow through the entire pipeline one at a time. It's not "filter all, then map all." It's "filter one, if passes map it, then next element." This is **loop fusion**—multiple operations combined into a single pass.

---

## Essential Operations

### Filtering

```java
// Basic filter
stream.filter(x -> x > 0)

// Distinct elements (uses equals/hashCode)
stream.distinct()

// Skip first n
stream.skip(10)

// Take first n
stream.limit(10)

// Filter by type (Java 16+)
stream.filter(Animal.class::isInstance)
      .map(Animal.class::cast)

// Or more elegantly with pattern matching preview
```

### Mapping (Transformation)

```java
// Transform each element
stream.map(String::toUpperCase)

// Transform to primitive stream (no boxing)
stream.mapToInt(String::length)

// One-to-many (flatten)
stream.flatMap(line -> Arrays.stream(line.split(" ")))
```

### flatMap: When Elements Become Streams

`flatMap` handles the case where each element maps to multiple elements:

```java
List<String> lines = List.of("hello world", "streams are powerful");

// map gives Stream<Stream<String>> - not what we want
lines.stream()
    .map(line -> Arrays.stream(line.split(" ")))
    // Result: Stream<Stream<String>>

// flatMap flattens to Stream<String>
lines.stream()
    .flatMap(line -> Arrays.stream(line.split(" ")))
    // Result: Stream<String> with: hello, world, streams, are, powerful
```

**Mental model**: `map` transforms each element. `flatMap` transforms each element into a stream, then concatenates all those streams.

### Sorting and Ordering

```java
// Natural order
stream.sorted()

// Custom comparator
stream.sorted(comparing(Person::getAge))

// Reversed
stream.sorted(comparing(Person::getAge).reversed())

// Multiple criteria
stream.sorted(
    comparing(Person::getLastName)
        .thenComparing(Person::getFirstName)
)
```

### Peeking (for Debugging)

```java
stream
    .filter(x -> x > 0)
    .peek(x -> System.out.println("After filter: " + x))
    .map(x -> x * 2)
    .peek(x -> System.out.println("After map: " + x))
    .collect(toList());
```

> **Warning**: `peek` is designed for debugging. Using it for side effects in production code is an anti-pattern—it makes the code dependent on the stream being fully consumed.

---

## Terminal Operations: Getting Results Out

### Collecting to Collections

```java
// To List
List<String> list = stream.collect(Collectors.toList());

// To unmodifiable List (Java 16+)
List<String> list = stream.toList();

// To Set
Set<String> set = stream.collect(Collectors.toSet());

// To specific collection
TreeSet<String> treeSet = stream.collect(
    Collectors.toCollection(TreeSet::new)
);

// To Map
Map<String, Person> byName = people.stream()
    .collect(Collectors.toMap(
        Person::getName,    // key mapper
        p -> p              // value mapper (identity)
    ));
```

### Reducing

Reduction combines all elements into a single result:

```java
// Sum
int total = numbers.stream().reduce(0, Integer::sum);

// Max
Optional<Integer> max = numbers.stream().reduce(Integer::max);

// String concatenation
String combined = strings.stream().reduce("", String::concat);
// (But use Collectors.joining() for strings—it's more efficient)
```

The reduce signature: `reduce(identity, accumulator)`

- **identity**: The starting value (and result for empty stream)
- **accumulator**: Function that combines running result with next element

```java
// Equivalent to:
T result = identity;
for (T element : stream) {
    result = accumulator.apply(result, element);
}
return result;
```

### Finding and Matching

```java
// Find any element matching
Optional<Person> any = people.stream()
    .filter(p -> p.getAge() > 65)
    .findAny();

// Find first matching (respects encounter order)
Optional<Person> first = people.stream()
    .filter(p -> p.getAge() > 65)
    .findFirst();

// Check if any match
boolean hasRetirees = people.stream()
    .anyMatch(p -> p.getAge() > 65);

// Check if all match
boolean allAdults = people.stream()
    .allMatch(p -> p.getAge() >= 18);

// Check if none match
boolean noMinors = people.stream()
    .noneMatch(p -> p.getAge() < 18);
```

### Counting and Statistics

```java
long count = stream.count();

IntSummaryStatistics stats = people.stream()
    .mapToInt(Person::getAge)
    .summaryStatistics();

stats.getCount();    // number of elements
stats.getSum();      // sum of ages
stats.getMin();      // minimum age
stats.getMax();      // maximum age
stats.getAverage();  // average age
```

---

## Advanced Collectors

The `Collectors` class is a goldmine of utility:

### Grouping

```java
// Group people by city
Map<String, List<Person>> byCity = people.stream()
    .collect(groupingBy(Person::getCity));

// Group and count
Map<String, Long> countByCity = people.stream()
    .collect(groupingBy(Person::getCity, counting()));

// Group and transform
Map<String, List<String>> namesByCity = people.stream()
    .collect(groupingBy(
        Person::getCity,
        mapping(Person::getName, toList())
    ));

// Nested grouping
Map<String, Map<Integer, List<Person>>> byCityThenAge = people.stream()
    .collect(groupingBy(
        Person::getCity,
        groupingBy(Person::getAge)
    ));
```

### Partitioning

```java
// Split into two groups: true and false
Map<Boolean, List<Person>> partitioned = people.stream()
    .collect(partitioningBy(p -> p.getAge() >= 18));

List<Person> adults = partitioned.get(true);
List<Person> minors = partitioned.get(false);
```

### Joining Strings

```java
String names = people.stream()
    .map(Person::getName)
    .collect(joining());           // "AliceBobCharlie"

String csv = people.stream()
    .map(Person::getName)
    .collect(joining(", "));       // "Alice, Bob, Charlie"

String formatted = people.stream()
    .map(Person::getName)
    .collect(joining(", ", "[", "]")); // "[Alice, Bob, Charlie]"
```

### Reducing Within Collectors

```java
// Find highest-paid person per department
Map<Department, Optional<Person>> topPaid = people.stream()
    .collect(groupingBy(
        Person::getDepartment,
        maxBy(comparing(Person::getSalary))
    ));

// Sum salaries per department
Map<Department, Long> salaryByDept = people.stream()
    .collect(groupingBy(
        Person::getDepartment,
        summingLong(Person::getSalary)
    ));
```

---

## Parallel Streams: The Promise and the Peril

```java
// Sequential
long count = list.stream()
    .filter(expensive::test)
    .count();

// Parallel
long count = list.parallelStream()
    .filter(expensive::test)
    .count();
```

One word change. Multiple cores engaged. What could go wrong?

### When Parallel Streams Help

Parallel streams use the Fork/Join framework to split work across CPU cores. They help when:

1. **Large data volume**: Thousands of elements minimum
2. **Expensive per-element operations**: CPU-bound work, not I/O
3. **No shared mutable state**: Operations are independent
4. **Splittable source**: ArrayList, arrays good; LinkedList, I/O streams bad

```java
// Good candidate: expensive computation on large array
double[] results = largeArray.parallelStream()
    .mapToDouble(this::expensiveCalculation)
    .toArray();
```

### When Parallel Streams Hurt

```java
// BAD: Shared mutable state
List<String> results = new ArrayList<>();
stream.parallel()
    .filter(s -> s.length() > 3)
    .forEach(s -> results.add(s));  // Race condition!

// BAD: I/O bound operation
urls.parallelStream()
    .map(this::fetchFromNetwork)    // Network is the bottleneck, not CPU
    .collect(toList());

// BAD: Small data
List<String> smallList = List.of("a", "b", "c");
smallList.parallelStream()          // Overhead exceeds benefit
    .map(String::toUpperCase)
    .collect(toList());

// BAD: Linked data structure
LinkedList<Integer> linked = ...;
linked.parallelStream()             // LinkedList can't split efficiently
    .map(x -> x * 2)
    .collect(toList());

// BAD: Order-dependent operations
stream.parallel()
    .limit(10)                      // Contention on maintaining order
    .collect(toList());
```

### The Fork/Join Pool Trap

By default, parallel streams use the **common Fork/Join pool**, shared across your entire application. If one parallel stream operation blocks (network I/O, waiting on locks), it starves all other parallel operations.

```java
// This can block the entire common pool!
urls.parallelStream()
    .map(url -> httpClient.get(url))  // Blocks waiting for HTTP response
    .collect(toList());
```

**Mitigation**: Use a custom pool for blocking operations:

```java
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() ->
    urls.parallelStream()
        .map(url -> httpClient.get(url))
        .collect(toList())
).join();
```

### Benchmarking: The Only Truth

Never assume parallel is faster. Measure.

```java
// Benchmark framework: JMH (Java Microbenchmark Harness)
@Benchmark
public long sequential() {
    return data.stream()
        .filter(x -> x > 0)
        .count();
}

@Benchmark
public long parallel() {
    return data.parallelStream()
        .filter(x -> x > 0)
        .count();
}
```

Typical results for simple operations on a 4-core machine:
- 100 elements: sequential 5x faster
- 10,000 elements: roughly equal
- 1,000,000 elements: parallel 2-3x faster

The break-even point depends on operation cost and hardware.

---

## Creating Streams

### From Collections

```java
List<String> list = List.of("a", "b", "c");
Stream<String> stream = list.stream();
```

### From Arrays

```java
String[] array = {"a", "b", "c"};
Stream<String> stream = Arrays.stream(array);

// Partial array
Stream<String> partial = Arrays.stream(array, 1, 3); // "b", "c"
```

### From Values

```java
Stream<String> stream = Stream.of("a", "b", "c");
```

### Infinite Streams

```java
// Generate with supplier (infinite)
Stream<Double> randoms = Stream.generate(Math::random);

// Iterate with seed and function (infinite)
Stream<Integer> naturals = Stream.iterate(0, n -> n + 1);

// Iterate with predicate (finite, Java 9+)
Stream<Integer> upTo100 = Stream.iterate(0, n -> n <= 100, n -> n + 1);
```

**Critical**: Infinite streams must be limited!

```java
Stream.iterate(0, n -> n + 1)
    .limit(100)                    // Without this, runs forever
    .forEach(System.out::println);
```

### From Files

```java
try (Stream<String> lines = Files.lines(Paths.get("data.txt"))) {
    lines.filter(line -> line.contains("ERROR"))
         .forEach(System.out::println);
}
// Stream is AutoCloseable—use try-with-resources
```

### IntStream, LongStream, DoubleStream

Primitive streams avoid boxing overhead:

```java
IntStream ints = IntStream.range(0, 100);      // 0 to 99
IntStream ints = IntStream.rangeClosed(0, 100); // 0 to 100

// Convert to object stream
Stream<Integer> boxed = IntStream.range(0, 10).boxed();

// Convert object stream to primitive
IntStream lengths = strings.stream().mapToInt(String::length);
```

---

## Real-World Patterns

### Pattern 1: Processing Records with Error Handling

```java
public List<ProcessedRecord> processRecords(List<RawRecord> records) {
    return records.stream()
        .map(this::tryProcess)                    // Returns Optional<ProcessedRecord>
        .flatMap(Optional::stream)                // Keep only successful ones
        .collect(toList());
}

private Optional<ProcessedRecord> tryProcess(RawRecord raw) {
    try {
        return Optional.of(transform(raw));
    } catch (ProcessingException e) {
        logger.warn("Failed to process: {}", raw, e);
        return Optional.empty();
    }
}
```

### Pattern 2: Building Indexes

```java
// Index people by their email addresses
Map<String, Person> personByEmail = people.stream()
    .collect(toMap(
        Person::getEmail,
        Function.identity(),
        (existing, duplicate) -> existing  // Keep first on collision
    ));

// Index people by tags (one-to-many)
Map<String, List<Person>> peopleByTag = people.stream()
    .flatMap(p -> p.getTags().stream()
        .map(tag -> Map.entry(tag, p)))
    .collect(groupingBy(
        Map.Entry::getKey,
        mapping(Map.Entry::getValue, toList())
    ));
```

### Pattern 3: Pagination

```java
public List<Item> getPage(int page, int pageSize) {
    return items.stream()
        .skip((long) page * pageSize)
        .limit(pageSize)
        .collect(toList());
}
```

### Pattern 4: Running Totals

```java
// Calculate cumulative sums
List<Integer> cumulative = new ArrayList<>();
numbers.stream()
    .reduce(0, (sum, n) -> {
        int newSum = sum + n;
        cumulative.add(newSum);
        return newSum;
    });

// Note: This uses reduce for side effects, which is not ideal.
// For production, consider a simple loop or specialized library.
```

---

## Stream vs Loop: When to Use Which

Use **streams** when:
- Operations are transformations (filter, map, reduce)
- Code reads as a data transformation description
- You might want to parallelize later
- Operations are independent and side-effect-free

Use **loops** when:
- You need to modify external state
- Control flow is complex (break to outer loop, multiple returns)
- Performance is critical and benchmarks show loop is faster
- Operation has side effects that need precise ordering

```java
// Stream: clear transformation
List<String> upperNames = names.stream()
    .filter(n -> n.length() > 3)
    .map(String::toUpperCase)
    .collect(toList());

// Loop: complex control flow
String result = null;
outer: for (Category cat : categories) {
    for (Product prod : cat.getProducts()) {
        if (prod.matches(criteria)) {
            result = prod.getName();
            break outer;
        }
    }
}
```

---

## What Interviewers Actually Want to Know

1. **What's the difference between intermediate and terminal operations?** Intermediate operations are lazy and return streams; terminal operations are eager and produce results or side effects.

2. **Explain lazy evaluation with an example.** A chain of `filter().map().limit(1)` processes elements one at a time, stopping after finding the first match.

3. **When would you use `flatMap`?** When each element maps to multiple elements (one-to-many), like splitting strings into words.

4. **When is parallel stream actually faster?** Large data, CPU-bound operations, independent computations, splittable source (ArrayList, not LinkedList).

5. **What's dangerous about parallel streams?** Shared mutable state, I/O-bound operations, and the common Fork/Join pool can be blocked by one bad operation.

6. **How is reduce different from collect?** Reduce combines elements into a single immutable result (sum, max). Collect accumulates into a mutable container (List, Map).

---

## Common Misconceptions

**"Streams are always faster than loops"**

No. Streams have overhead. For simple operations on small collections, loops are often faster. Streams win on readability and composition.

**"Parallel streams are always faster than sequential"**

No. Parallelization has overhead. For small data or simple operations, sequential is faster. Always benchmark.

**"You can reuse a stream"**

No. Streams are consumed by terminal operations. Create a new stream for each use.

**"peek() is for general side effects"**

No. `peek()` is for debugging. Its execution depends on the terminal operation—it might not run for all elements.

**"map and flatMap are interchangeable"**

No. `map` is one-to-one. `flatMap` is one-to-many with flattening.

---

## Summary

| Concept | Key Insight |
|---------|-------------|
| Stream | Not a data structure—a description of computation |
| Lazy Evaluation | Operations don't run until terminal operation |
| Internal Iteration | Library controls iteration, enabling optimization |
| Intermediate vs Terminal | Intermediate returns stream (lazy); terminal produces result |
| Parallel Streams | Easy to write, hard to get right; always benchmark |
| Collectors | Powerful aggregation toolkit for terminal operations |

Streams changed how Java programmers think about data processing. Instead of writing loops, we describe transformations. Instead of mutating collections, we derive new ones. The code becomes a declaration of intent rather than a specification of mechanism.

But streams aren't magic. They're a tool. Used well, they make code clearer and potentially faster. Used poorly, they create confusion and performance problems. The key is understanding when each tool fits.

---

*Next Chapter: [Time Done Right](04-time-done-right.md)*
