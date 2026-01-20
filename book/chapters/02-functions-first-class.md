# Chapter 2: Functions as First-Class Citizens

*Lambdas, functional interfaces, method references—what shifted in thinking*

---

## The World Before: Behavior Trapped in Objects

Before Java 8, if you wanted to pass behavior to a method, you had to wrap it in an object. Want to sort a list with custom logic? Create a `Comparator`. Want to run code in a thread? Create a `Runnable`. Want to handle a button click? Create an `ActionListener`.

```java
// Sorting strings by length in Java 7
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return Integer.compare(a.length(), b.length());
    }
});
```

Five lines of code to express what should be a single thought: "compare by length."

This pattern—anonymous inner classes—was Java's workaround for not having first-class functions. And it was painful:

1. **Verbose**: Ceremony overwhelms intent
2. **Scoping confusion**: `this` refers to the anonymous class, not the enclosing class
3. **Performance**: Each anonymous class generates a separate `.class` file
4. **Readability**: The actual logic drowns in boilerplate

Engineers were frustrated. Other JVM languages (Scala, Groovy, Kotlin) had concise function syntax. Even JavaScript had `function(a, b) { return a.length - b.length; }`. Java felt stuck in the past.

---

## The Breakthrough: What Is a Lambda?

A lambda expression is, at its heart, a concise way to represent behavior. Here's the same sorting logic:

```java
// Java 8+
Collections.sort(names, (a, b) -> Integer.compare(a.length(), b.length()));
```

One line. The intent is clear: compare by length.

But let's be precise about what a lambda *is* in Java. Unlike functional languages where functions are truly first-class values, Java lambdas are **syntactic sugar for instances of functional interfaces**. The compiler converts:

```java
(a, b) -> Integer.compare(a.length(), b.length())
```

Into something that implements `Comparator<String>`. Not literally into an anonymous inner class (more on this later), but into something the JVM treats as a Comparator instance.

### The Mental Model: Lambdas as Behavior Literals

Think of lambdas like other literals in Java:

- `42` is an int literal
- `"hello"` is a String literal
- `(x) -> x * 2` is a **behavior literal**—a piece of code you can pass around

Just as you can pass `42` to any method expecting an `int`, you can pass `(x) -> x * 2` to any method expecting a compatible functional interface.

---

## Functional Interfaces: The Contract

A **functional interface** is any interface with exactly one abstract method. That's it. The `@FunctionalInterface` annotation is optional—it just asks the compiler to verify your interface qualifies.

```java
@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);

    // Default methods don't count against the "one abstract method" rule
    default Comparator<T> reversed() {
        return (o1, o2) -> compare(o2, o1);
    }

    // Static methods don't count either
    static <T extends Comparable<? super T>> Comparator<T> naturalOrder() {
        return (a, b) -> a.compareTo(b);
    }
}
```

Why does Java use functional interfaces instead of true function types? **Backwards compatibility**. The existing Java ecosystem had millions of APIs expecting interfaces like `Runnable`, `Comparator`, and `Callable`. Making lambdas compatible with these interfaces meant instant adoption—no API changes required.

### The Core Functional Interfaces

Java 8 introduced `java.util.function` with standard functional interfaces. Understanding these is essential:

```
┌─────────────────┬──────────────────┬─────────────────────────────────┐
│ Interface       │ Method           │ Description                     │
├─────────────────┼──────────────────┼─────────────────────────────────┤
│ Predicate<T>    │ test(T) → bool   │ Takes T, returns boolean        │
│ Function<T,R>   │ apply(T) → R     │ Takes T, returns R              │
│ Consumer<T>     │ accept(T) → void │ Takes T, returns nothing        │
│ Supplier<T>     │ get() → T        │ Takes nothing, returns T        │
│ UnaryOperator<T>│ apply(T) → T     │ Takes T, returns same type T    │
│ BinaryOperator<T>│ apply(T,T) → T  │ Takes two Ts, returns T         │
│ BiFunction<T,U,R>│ apply(T,U) → R  │ Takes T and U, returns R        │
│ BiPredicate<T,U>│ test(T,U) → bool │ Takes T and U, returns boolean  │
│ BiConsumer<T,U> │ accept(T,U)→void │ Takes T and U, returns nothing  │
└─────────────────┴──────────────────┴─────────────────────────────────┘
```

**Memory trick**: The naming is systematic.
- `Bi-` prefix = two parameters
- `Predicate` = returns boolean
- `Consumer` = returns void
- `Supplier` = takes nothing
- `Function` = takes something, returns something else
- `Operator` = takes and returns the same type

### Primitive Specializations

For performance (avoiding boxing), there are primitive-specialized versions:

```java
IntPredicate       // int → boolean
LongFunction<R>    // long → R
DoubleSupplier     // () → double
IntToDoubleFunction // int → double
// ... and more
```

Use these in performance-critical code to avoid the overhead of boxing/unboxing.

---

## Lambda Syntax: The Full Picture

Lambda syntax has several forms, from verbose to concise:

```java
// Full explicit form
(String a, String b) -> { return a.length() - b.length(); }

// Inferred parameter types
(a, b) -> { return a.length() - b.length(); }

// Single expression (no braces, implicit return)
(a, b) -> a.length() - b.length()

// Single parameter (no parentheses needed)
x -> x * 2

// No parameters
() -> System.out.println("Hello")
```

The rule: **the compiler infers whatever it can**. Type parameters, return statements, and parentheses are omitted when unambiguous.

### Block Lambdas vs Expression Lambdas

```java
// Expression lambda: single expression, implicit return
x -> x * 2

// Block lambda: multiple statements, explicit return required
x -> {
    int doubled = x * 2;
    System.out.println("Doubling: " + x);
    return doubled;
}
```

Prefer expression lambdas when possible—they're more concise and signal that the operation is side-effect-free.

---

## Variable Capture: The "Effectively Final" Rule

Lambdas can capture variables from their enclosing scope:

```java
String prefix = "User: ";
Function<String, String> addPrefix = name -> prefix + name;
```

But there's a restriction: captured variables must be **effectively final**—they can't be modified after the lambda is created.

```java
String prefix = "User: ";
// prefix = "Admin: ";  // Compile error if uncommented!
Function<String, String> addPrefix = name -> prefix + name;
```

### Why This Restriction?

Imagine you're building a concurrent application. A lambda might execute on a different thread, at a later time:

```java
for (int i = 0; i < 10; i++) {
    executor.submit(() -> System.out.println(i)); // Won't compile!
}
```

If this compiled and `i` were mutable, what would it print? By the time the lambdas execute, the loop might be done and `i` might be 10 for all of them—or it might be some random mix. The "effectively final" rule prevents this class of bugs.

**Workaround for loops**:

```java
for (int i = 0; i < 10; i++) {
    int index = i; // Create a new effectively-final variable each iteration
    executor.submit(() -> System.out.println(index));
}
```

### `this` in Lambdas vs Anonymous Classes

Here's a subtle but important difference:

```java
public class EventHandler {
    private String name = "Handler";

    public void setupWithAnonymous() {
        button.addActionListener(new ActionListener() {
            @Override
            public void actionPerformed(ActionEvent e) {
                // 'this' refers to the ActionListener instance
                System.out.println(this.getClass().getName());
            }
        });
    }

    public void setupWithLambda() {
        button.addActionListener(e -> {
            // 'this' refers to the EventHandler instance
            System.out.println(this.name); // Prints "Handler"
        });
    }
}
```

In lambdas, `this` refers to the enclosing class. In anonymous classes, `this` refers to the anonymous class itself. This is often more intuitive for lambdas.

---

## Method References: When a Lambda Just Calls a Method

Often, a lambda simply delegates to an existing method:

```java
names.forEach(name -> System.out.println(name));
names.stream().map(name -> name.toUpperCase());
numbers.stream().reduce(0, (a, b) -> Integer.sum(a, b));
```

Method references are a shorter syntax for these cases:

```java
names.forEach(System.out::println);
names.stream().map(String::toUpperCase);
numbers.stream().reduce(0, Integer::sum);
```

### The Four Types of Method References

```
┌─────────────────────────┬──────────────────────────┬─────────────────────────┐
│ Type                    │ Syntax                   │ Equivalent Lambda       │
├─────────────────────────┼──────────────────────────┼─────────────────────────┤
│ Static method           │ ClassName::staticMethod  │ (args) -> Class.method(args)│
│ Instance method (bound) │ instance::method         │ (args) -> instance.method(args)│
│ Instance method (unbound)│ ClassName::method       │ (obj, args) -> obj.method(args)│
│ Constructor             │ ClassName::new           │ (args) -> new ClassName(args)│
└─────────────────────────┴──────────────────────────┴─────────────────────────┘
```

Let's understand each:

**1. Static Method Reference**
```java
// Lambda
Function<String, Integer> parser = s -> Integer.parseInt(s);

// Method reference
Function<String, Integer> parser = Integer::parseInt;
```

**2. Bound Instance Method Reference**
```java
String prefix = "Hello, ";

// Lambda
Function<String, String> greeter = name -> prefix.concat(name);

// Method reference (bound to the specific 'prefix' instance)
Function<String, String> greeter = prefix::concat;
```

**3. Unbound Instance Method Reference** (the tricky one)
```java
// Lambda
Function<String, String> upper = s -> s.toUpperCase();

// Method reference
Function<String, String> upper = String::toUpperCase;
```

Wait—`toUpperCase()` is an instance method. How does `String::toUpperCase` work with `Function<String, String>`?

The first parameter becomes the receiver (`this`). So `String::toUpperCase` means "call `toUpperCase()` on the String that's passed as the first argument."

This is particularly useful for comparisons:

```java
// Sorting strings by their natural order
list.sort(String::compareTo);

// Equivalent to
list.sort((a, b) -> a.compareTo(b));
```

**4. Constructor Reference**
```java
// Lambda
Supplier<ArrayList<String>> factory = () -> new ArrayList<>();

// Constructor reference
Supplier<ArrayList<String>> factory = ArrayList::new;

// With parameters
Function<Integer, ArrayList<String>> factory = ArrayList::new;
// Calls new ArrayList<>(initialCapacity)
```

---

## Composing Functions: The Power of Default Methods

Functional interfaces in `java.util.function` include powerful composition methods:

### Predicate Composition

```java
Predicate<String> isLong = s -> s.length() > 10;
Predicate<String> startsWithA = s -> s.startsWith("A");

// Combine predicates
Predicate<String> isLongAndStartsWithA = isLong.and(startsWithA);
Predicate<String> isLongOrStartsWithA = isLong.or(startsWithA);
Predicate<String> isNotLong = isLong.negate();

// Use in streams
strings.stream()
    .filter(isLong.and(startsWithA))
    .collect(toList());
```

### Function Composition

```java
Function<String, Integer> length = String::length;
Function<Integer, Integer> doubled = x -> x * 2;

// Compose: first apply length, then doubled
Function<String, Integer> doubleLength = length.andThen(doubled);

// Or in reverse order
Function<String, Integer> alsoDoubleLength = doubled.compose(length);

// Both give: "hello" → 5 → 10
```

### Comparator Composition

```java
Comparator<Person> byLastName = Comparator.comparing(Person::getLastName);
Comparator<Person> byFirstName = Comparator.comparing(Person::getFirstName);
Comparator<Person> byAge = Comparator.comparingInt(Person::getAge);

// Chain comparators
Comparator<Person> fullComparator = byLastName
    .thenComparing(byFirstName)
    .thenComparing(byAge);

// With nulls
Comparator<Person> nullSafe = Comparator.nullsFirst(byLastName);

// Reversed
Comparator<Person> oldestFirst = byAge.reversed();
```

---

## Under the Hood: How Lambdas Actually Work

Here's where Java's approach gets clever. Lambdas are **not** compiled to anonymous inner classes. Instead, they use `invokedynamic`, a bytecode instruction introduced in Java 7.

### The Problem with Anonymous Classes

If lambdas compiled to anonymous classes:
1. Each lambda would generate a `.class` file at compile time
2. JVM would have to load these classes, consuming metaspace
3. Instance creation would require `new`, triggering allocation

### The invokedynamic Solution

Instead, the compiler generates:
1. A private static method containing the lambda body
2. An `invokedynamic` instruction that, on first execution, calls a **bootstrap method** to create a function object

```
// Conceptually, this lambda:
x -> x * 2

// Gets compiled to something like:
private static Integer lambda$0(Integer x) {
    return x * 2;
}

// And an invokedynamic instruction that creates the Function object
```

The bootstrap method (part of `LambdaMetafactory`) decides at runtime the best way to implement the functional interface. It might:
- Generate a new class dynamically
- Return a singleton for stateless lambdas
- Use MethodHandles for efficiency

**Why does this matter?**

1. **Performance**: The JIT can inline lambda code more easily
2. **Memory**: Stateless lambdas can be cached as singletons
3. **Future flexibility**: The JVM can improve lambda implementation without recompiling code

---

## Practical Patterns

### Pattern 1: Strategy via Lambda

Before:
```java
interface DiscountStrategy {
    BigDecimal calculate(Order order);
}

class PercentageDiscount implements DiscountStrategy {
    private final BigDecimal percentage;

    PercentageDiscount(BigDecimal percentage) {
        this.percentage = percentage;
    }

    @Override
    public BigDecimal calculate(Order order) {
        return order.getTotal().multiply(percentage);
    }
}
```

After:
```java
Function<Order, BigDecimal> percentageDiscount(BigDecimal pct) {
    return order -> order.getTotal().multiply(pct);
}

Function<Order, BigDecimal> flatDiscount(BigDecimal amount) {
    return order -> amount;
}

// Usage
Function<Order, BigDecimal> discount = percentageDiscount(new BigDecimal("0.10"));
BigDecimal savings = discount.apply(order);
```

No interface needed. No implementation classes. Just functions.

### Pattern 2: Execute Around Method

When you have setup/cleanup that wraps variable logic:

```java
public <T> T withConnection(Function<Connection, T> operation) {
    Connection conn = dataSource.getConnection();
    try {
        return operation.apply(conn);
    } finally {
        conn.close();
    }
}

// Usage
List<User> users = withConnection(conn ->
    conn.prepareStatement("SELECT * FROM users")
        .executeQuery()
        .stream()
        .map(this::mapToUser)
        .collect(toList())
);
```

The caller provides the "what," the method handles the "around."

### Pattern 3: Lazy Initialization

```java
public class ExpensiveResource {
    private Supplier<Data> dataSupplier;
    private Data cachedData;

    public ExpensiveResource(Supplier<Data> dataSupplier) {
        this.dataSupplier = dataSupplier;
    }

    public Data getData() {
        if (cachedData == null) {
            cachedData = dataSupplier.get();
            dataSupplier = null; // Allow GC of the supplier
        }
        return cachedData;
    }
}

// Usage
ExpensiveResource resource = new ExpensiveResource(() -> loadFromDatabase());
// Data isn't loaded until getData() is called
```

### Pattern 4: Validation Pipelines

```java
@FunctionalInterface
interface Validator<T> {
    ValidationResult validate(T value);

    default Validator<T> and(Validator<T> other) {
        return value -> {
            ValidationResult result = this.validate(value);
            return result.isValid() ? other.validate(value) : result;
        };
    }
}

Validator<User> emailNotEmpty = user ->
    user.getEmail().isEmpty()
        ? ValidationResult.error("Email required")
        : ValidationResult.ok();

Validator<User> validEmailFormat = user ->
    user.getEmail().contains("@")
        ? ValidationResult.ok()
        : ValidationResult.error("Invalid email format");

Validator<User> userValidator = emailNotEmpty.and(validEmailFormat);

ValidationResult result = userValidator.validate(user);
```

---

## What Interviewers Actually Want to Know

1. **What is a lambda in Java?** Syntax for creating instances of functional interfaces. Not true first-class functions, but close enough with better ergonomics.

2. **What's a functional interface?** An interface with exactly one abstract method. It's what lambdas implement.

3. **Explain the four types of method references.** Static (`Class::static`), bound instance (`instance::method`), unbound instance (`Class::method`), and constructor (`Class::new`).

4. **Why must captured variables be effectively final?** To prevent race conditions and confusing behavior when lambdas execute later or on different threads.

5. **How do lambdas differ from anonymous classes?** Lambdas use invokedynamic (no class file per lambda), `this` refers to enclosing class, and syntax is much more concise.

6. **What's the difference between `Function` and `Consumer`?** Function returns a value; Consumer returns void. Function transforms; Consumer produces side effects.

---

## Common Misconceptions

**"Lambdas are just syntactic sugar for anonymous classes"**

Technically false. Lambdas use `invokedynamic` and `LambdaMetafactory`, which can be more efficient and allow the JVM to optimize differently.

**"You can use any interface with lambdas"**

No—only functional interfaces (one abstract method). `Runnable` works, but `Iterator` doesn't (it has `hasNext()` and `next()`).

**"Method references are always better than lambdas"**

Not when the method reference would be less clear. Compare:
```java
.filter(Objects::nonNull)         // Clear
.map(String::trim)                // Clear
.map(s -> s.substring(1))         // Clearer than would-be method reference
.filter(s -> s.length() > 5)      // No method reference possible
```

**"Lambdas have no performance overhead"**

There's still object creation (though potentially cached for stateless lambdas) and method call overhead. In extreme hot paths, an explicit loop might still be faster. But for 99% of code, the difference is negligible.

---

## The Paradigm Shift

Before Java 8, Java was a purely object-oriented language where behavior had to live in objects. This led to patterns like Strategy, Command, and Template Method—all ways to encapsulate behavior.

Lambdas don't eliminate these patterns, but they make lightweight versions trivial. You don't need a `DiscountStrategy` interface with `PercentageDiscount` and `FlatDiscount` implementations when a simple `Function<Order, BigDecimal>` will do.

This shift enables:
- **Higher-order functions**: Methods that take functions as parameters or return them
- **Composition over inheritance**: Building behavior from small functions
- **Deferred execution**: Passing behavior to be executed later
- **Declarative code**: Expressing *what* rather than *how*

The Streams API (next chapter) builds entirely on these foundations. Without lambdas, streams would require anonymous classes at every step—making them impractical.

---

## Summary

| Concept | Key Insight |
|---------|-------------|
| Lambda Expression | Concise syntax for creating functional interface instances |
| Functional Interface | Interface with exactly one abstract method |
| Method Reference | Even more concise when lambda just calls existing method |
| Variable Capture | Captured variables must be effectively final |
| `this` in Lambdas | Refers to enclosing class, not the lambda itself |
| Composition | Predicates, Functions, Comparators can be combined |
| invokedynamic | Lambdas aren't anonymous classes; they're more efficient |

---

*Next Chapter: [Streams: Declarative Data Processing](03-streams.md)*
