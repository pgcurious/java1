# Chapter 5: Memory and References Reimagined

*var, records, sealed classes—reducing ceremony, increasing intent*

---

## The Verbosity Tax

For decades, Java developers paid a verbosity tax. Simple concepts required disproportionate code:

```java
// A simple data holder in Java 8
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int getX() { return x; }
    public int getY() { return y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Point point = (Point) o;
        return x == point.x && y == point.y;
    }

    @Override
    public int hashCode() {
        return Objects.hash(x, y);
    }

    @Override
    public String toString() {
        return "Point{x=" + x + ", y=" + y + "}";
    }
}
```

50+ lines for two integers. This wasn't just tedious—it was error-prone. Forget to update `equals` when adding a field? Bug. Generate `hashCode` incorrectly? Different bug. Copy-paste from another class and forget to change the field names? Yet another bug.

Java's designers recognized that ceremony obscures intent. Modern Java has addressed this through several features that don't just save typing—they fundamentally change how we express domain concepts.

---

## Local Variable Type Inference: var (Java 10)

### The Problem

```java
Map<String, List<Customer>> customersByCity = new HashMap<String, List<Customer>>();
```

The type appears twice. The right side literally tells you what type is being constructed. Why repeat it on the left?

### The Solution

```java
var customersByCity = new HashMap<String, List<Customer>>();
```

`var` tells the compiler: "Figure out the type from the initializer. I trust you."

### What var Is (and Isn't)

`var` is **not** dynamic typing. Java remains statically typed. The compiler infers the type at compile time, and the variable is that type forever:

```java
var name = "Alice";  // Compiler infers String
name = 42;           // Compile error! name is String, not Object
```

`var` is **not** `Object`. The inferred type is the specific type of the initializer:

```java
var list = new ArrayList<String>();
list.add("hello");   // Works - list is ArrayList<String>
list.add(42);        // Compile error! Wrong type
```

### When var Helps

**Complex generic types:**
```java
// Before
Map<String, Map<Integer, List<Customer>>> nested = new HashMap<>();

// After
var nested = new HashMap<String, Map<Integer, List<Customer>>>();
```

**Loop variables:**
```java
for (var entry : map.entrySet()) {
    var key = entry.getKey();
    var value = entry.getValue();
}
```

**Try-with-resources:**
```java
try (var reader = new BufferedReader(new FileReader(path))) {
    var line = reader.readLine();
}
```

**Anonymous classes (especially useful):**
```java
// The anonymous class has a type that's hard to write
var comparator = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.length() - b.length();
    }

    public String describe() {
        return "Length comparator";
    }
};

// Can call the extra method!
System.out.println(comparator.describe());
```

### When var Hurts

**Unclear types from method names:**
```java
var result = processor.process(data);  // What type is result?
var x = getValue();                    // Completely unclear
```

**Literals where type matters:**
```java
var count = 0;        // int? long? Integer?
var amount = 3.14;    // double? float? BigDecimal? (It's double)
```

**Readability is paramount:**
```java
// This is clear without seeing the right side
Customer customer = findCustomer(id);

// This requires understanding findCustomer's return type
var customer = findCustomer(id);
```

### var Rules

1. **Local variables only**: Not for fields, method parameters, or return types
2. **Must have initializer**: `var x;` is illegal
3. **Cannot be null**: `var x = null;` is illegal (what type would it be?)
4. **Cannot be lambda**: `var f = x -> x * 2;` is illegal (target type unknown)
5. **Cannot be array initializer**: `var arr = {1, 2, 3};` is illegal

### Style Guidelines

From the official Oracle style guide:

1. **Choose variable names carefully**: With `var`, the name carries more weight
2. **Minimize scope**: Use `var` for short-lived local variables
3. **Initialize with constructor or factory**: `var list = new ArrayList<>()` is clear
4. **Consider the reader**: If the type isn't obvious, spell it out

```java
// Good: type is obvious from constructor
var customers = new ArrayList<Customer>();
var formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");

// Questionable: type not obvious
var result = computeResult();

// Bad: misleading name
var string = getObject();  // Returns Object, not String!
```

---

## Records: Data Classes Done Right (Java 16)

### The Problem

Most classes in real applications are plain data carriers: hold some values, provide access, implement equality. The Point example from earlier showed 50+ lines for trivial functionality.

Lombok helped with `@Data`, but it's a compile-time annotation processor with its own complexity. Kotlin has `data class`. Scala has `case class`. Java needed a first-class solution.

### The Solution

```java
public record Point(int x, int y) { }
```

That's it. This single line gives you:
- Private final fields `x` and `y`
- Constructor `Point(int x, int y)`
- Accessor methods `x()` and `y()` (not `getX()`)
- `equals()` based on all fields
- `hashCode()` based on all fields
- `toString()` returning `Point[x=1, y=2]`

```java
var p1 = new Point(1, 2);
var p2 = new Point(1, 2);

System.out.println(p1.x());           // 1
System.out.println(p1);               // Point[x=1, y=2]
System.out.println(p1.equals(p2));    // true
System.out.println(p1 == p2);         // false (different objects)
```

### Records Are Immutable

Record fields are `final`. There are no setters. To "modify" a record, you create a new one:

```java
public record Point(int x, int y) {
    public Point withX(int newX) {
        return new Point(newX, this.y);
    }
}

var p1 = new Point(1, 2);
var p2 = p1.withX(10);  // Point[x=10, y=2]
// p1 is unchanged: Point[x=1, y=2]
```

This immutability is a feature, not a limitation. Immutable objects are thread-safe, can be freely shared, and don't suffer from aliasing bugs.

### Compact Constructors: Validation Without Repetition

Records can have a **compact constructor** for validation:

```java
public record Range(int start, int end) {
    public Range {
        // No parameter list - parameters are implicit
        if (start > end) {
            throw new IllegalArgumentException("start must be <= end");
        }
        // Implicit: this.start = start; this.end = end;
    }
}
```

The compact constructor runs before field assignment. You can validate and transform:

```java
public record Email(String value) {
    public Email {
        // Normalize
        value = value.toLowerCase().trim();
        // Validate
        if (!value.contains("@")) {
            throw new IllegalArgumentException("Invalid email");
        }
    }
}
```

### Adding Methods

Records can have additional methods, static members, and implement interfaces:

```java
public record Person(String name, LocalDate birthDate) implements Comparable<Person> {

    // Instance method
    public int age() {
        return Period.between(birthDate, LocalDate.now()).getYears();
    }

    // Static factory
    public static Person parse(String csv) {
        String[] parts = csv.split(",");
        return new Person(parts[0], LocalDate.parse(parts[1]));
    }

    // Interface implementation
    @Override
    public int compareTo(Person other) {
        return this.name.compareTo(other.name);
    }
}
```

### Records Cannot Extend Classes

Records implicitly extend `java.lang.Record`. They cannot extend other classes (though they can implement interfaces):

```java
// Illegal: records can't extend classes
public record Employee(String name) extends Person { }

// Legal: records can implement interfaces
public record Employee(String name) implements Serializable { }
```

### Records and Serialization

Records have improved serialization behavior:

```java
public record User(String name, int age) implements Serializable { }
```

Unlike regular classes, record serialization uses the canonical constructor. This means:
- No need for `serialVersionUID`
- Validation in the constructor runs during deserialization
- No risk of uninitialized fields

### When to Use Records

**Use records for:**
- DTOs (Data Transfer Objects)
- API responses/requests
- Value objects (Money, Coordinates, Range)
- Immutable domain entities
- Tuple-like structures

**Don't use records for:**
- Mutable state
- Complex inheritance hierarchies
- When you need to customize equality in non-obvious ways
- JPA entities (they require no-arg constructor and mutable fields)

---

## Sealed Classes: Controlled Inheritance (Java 17)

### The Problem

Imagine modeling arithmetic expressions:

```java
abstract class Expr { }
class Num extends Expr { int value; }
class Add extends Expr { Expr left, right; }
class Mul extends Expr { Expr left, right; }
```

Someone can add `class Div extends Expr` anywhere in the codebase. Your exhaustive switch statement over expression types is no longer exhaustive. Your pattern matching is incomplete. Maintenance becomes a game of whack-a-mole.

In functional languages, this is solved with **algebraic data types** (ADTs)—a type that can be exactly one of a fixed set of variants. Java now has this capability.

### The Solution

```java
public sealed class Expr permits Num, Add, Mul {
    // ...
}

public final class Num extends Expr {
    private final int value;
    // ...
}

public final class Add extends Expr {
    private final Expr left, right;
    // ...
}

public final class Mul extends Expr {
    private final Expr left, right;
    // ...
}
```

The `sealed` keyword means: "Only these specific classes can extend me." The `permits` clause lists them explicitly. Each permitted subclass must be:
- `final` (no further extension)
- `sealed` (controlled further extension)
- `non-sealed` (opens up extension again)

### The Compiler Knows Your Hierarchy

Because the set of subtypes is fixed and known at compile time, the compiler can:

1. **Verify exhaustiveness**: A switch over a sealed type must cover all permitted subtypes
2. **Enable pattern matching**: Safely deconstruct without instanceof checks
3. **Warn about impossible patterns**: If you handle a subtype that can't occur

```java
// Compiler knows this is exhaustive
int eval(Expr expr) {
    return switch (expr) {
        case Num n -> n.value();
        case Add a -> eval(a.left()) + eval(a.right());
        case Mul m -> eval(m.left()) * eval(m.right());
        // No default needed! Compiler verified completeness.
    };
}
```

If you add `Div` to the permits clause later, every switch statement that doesn't handle it becomes a compile error. This is **compile-time safety for evolving hierarchies**.

### Sealed Interfaces

Interfaces can be sealed too:

```java
public sealed interface JsonValue permits JsonString, JsonNumber, JsonBoolean, JsonNull, JsonArray, JsonObject { }

public record JsonString(String value) implements JsonValue { }
public record JsonNumber(double value) implements JsonValue { }
public record JsonBoolean(boolean value) implements JsonValue { }
public record JsonNull() implements JsonValue { }
public record JsonArray(List<JsonValue> elements) implements JsonValue { }
public record JsonObject(Map<String, JsonValue> members) implements JsonValue { }
```

Pattern matching becomes elegant:

```java
String describe(JsonValue json) {
    return switch (json) {
        case JsonString s -> "String: " + s.value();
        case JsonNumber n -> "Number: " + n.value();
        case JsonBoolean b -> "Boolean: " + b.value();
        case JsonNull ignored -> "Null";
        case JsonArray arr -> "Array with " + arr.elements().size() + " elements";
        case JsonObject obj -> "Object with " + obj.members().size() + " keys";
    };
}
```

### Sealed Classes + Records = Algebraic Data Types

The combination is powerful:

```java
public sealed interface Shape permits Circle, Rectangle, Triangle { }

public record Circle(double radius) implements Shape { }
public record Rectangle(double width, double height) implements Shape { }
public record Triangle(double a, double b, double c) implements Shape { }
```

This is a **sum type** (one of Circle, Rectangle, or Triangle) made of **product types** (records with named components). Classic algebraic data types, now in Java.

```java
double area(Shape shape) {
    return switch (shape) {
        case Circle(var r) -> Math.PI * r * r;
        case Rectangle(var w, var h) -> w * h;
        case Triangle(var a, var b, var c) -> {
            var s = (a + b + c) / 2;
            yield Math.sqrt(s * (s - a) * (s - b) * (s - c));
        }
    };
}
```

### non-sealed: The Escape Hatch

Sometimes you want controlled inheritance at one level but open extension below:

```java
public sealed class Account permits CheckingAccount, SavingsAccount, InvestmentAccount { }

public final class CheckingAccount extends Account { }
public final class SavingsAccount extends Account { }

// Allow third parties to create specific investment account types
public non-sealed class InvestmentAccount extends Account { }

// Now anyone can extend InvestmentAccount
public class MutualFundAccount extends InvestmentAccount { }
public class BrokerageAccount extends InvestmentAccount { }
```

---

## Text Blocks: Readable Multi-Line Strings (Java 15)

### The Problem

```java
String json = "{\n" +
    "    \"name\": \"Alice\",\n" +
    "    \"age\": 30,\n" +
    "    \"email\": \"alice@example.com\"\n" +
    "}";
```

Escape sequences everywhere. Line continuations. Error-prone concatenation.

### The Solution

```java
String json = """
    {
        "name": "Alice",
        "age": 30,
        "email": "alice@example.com"
    }
    """;
```

Text blocks use `"""` delimiters. The content's indentation is determined by the position of the closing `"""`.

### Incidental vs Essential Whitespace

```java
String html = """
              <html>
                  <body>
                      <p>Hello</p>
                  </body>
              </html>
              """;
```

The leading whitespace that aligns with the closing delimiter is **incidental** (stripped). Whitespace beyond that is **essential** (preserved).

Result:
```
<html>
    <body>
        <p>Hello</p>
    </body>
</html>
```

### String Interpolation (Sort Of)

Text blocks support `formatted()`:

```java
String name = "Alice";
int age = 30;

String json = """
    {
        "name": "%s",
        "age": %d
    }
    """.formatted(name, age);
```

> **Note**: Java doesn't have true string interpolation (like `$variable` in Kotlin/Scala). You still use `%s` format specifiers.

---

## Pattern Variables in instanceof (Java 16)

### The Problem

```java
if (obj instanceof String) {
    String s = (String) obj;  // Redundant cast
    System.out.println(s.length());
}
```

You checked that `obj` is a String, but you still have to cast it.

### The Solution

```java
if (obj instanceof String s) {
    System.out.println(s.length());  // s is already a String
}
```

The pattern variable `s` is in scope only where the compiler knows the `instanceof` succeeded:

```java
if (obj instanceof String s && s.length() > 5) {
    // s is in scope
}

if (obj instanceof String s || s.length() > 5) {
    // Compile error! s might not be bound if the instanceof failed
}

if (!(obj instanceof String s)) {
    return;  // s is not in scope here
}
// s IS in scope here (because we returned if it wasn't a String)
```

This pattern variable scoping, called **flow scoping**, is smart about control flow.

---

## Putting It All Together: Modern Java Style

Here's a realistic example combining these features:

```java
// Domain model with sealed types and records
public sealed interface PaymentMethod permits CreditCard, BankTransfer, DigitalWallet { }

public record CreditCard(String number, YearMonth expiry, String cvv) implements PaymentMethod {
    public CreditCard {
        if (number.length() != 16) throw new IllegalArgumentException("Invalid card number");
        if (expiry.isBefore(YearMonth.now())) throw new IllegalArgumentException("Card expired");
    }

    public String maskedNumber() {
        return "**** **** **** " + number.substring(12);
    }
}

public record BankTransfer(String accountNumber, String routingNumber) implements PaymentMethod { }

public record DigitalWallet(String provider, String email) implements PaymentMethod { }

// Processing with pattern matching
public class PaymentProcessor {

    public PaymentResult process(PaymentMethod method, Money amount) {
        return switch (method) {
            case CreditCard card -> processCreditCard(card, amount);
            case BankTransfer transfer -> processBankTransfer(transfer, amount);
            case DigitalWallet wallet -> processDigitalWallet(wallet, amount);
        };
    }

    private PaymentResult processCreditCard(CreditCard card, Money amount) {
        var gateway = new CreditCardGateway();
        var response = gateway.charge(card.maskedNumber(), amount);

        if (response instanceof SuccessResponse success) {
            return new PaymentResult.Success(success.transactionId());
        } else if (response instanceof FailureResponse failure) {
            return new PaymentResult.Failure(failure.reason());
        }
        throw new IllegalStateException("Unexpected response type");
    }

    // ... other methods
}
```

Notice:
- `sealed interface` defines the payment type hierarchy
- `record` for immutable, validated data
- `switch` with pattern matching for exhaustive handling
- `var` for local variables where type is obvious
- `instanceof` with pattern variable for safe casting

---

## What Interviewers Actually Want to Know

1. **What is `var` in Java?** Local variable type inference. The compiler deduces the type from the initializer. It's still statically typed, just less verbose.

2. **What are records?** Immutable data carriers with auto-generated constructors, accessors, equals, hashCode, and toString. Like Kotlin data classes.

3. **What's the point of sealed classes?** They restrict which classes can extend them, enabling exhaustive pattern matching and safer hierarchies. It's algebraic data types for Java.

4. **When would you use sealed classes vs regular inheritance?** Sealed when you want a closed set of subtypes (sum types, state machines, AST nodes). Regular when extension should be open.

5. **Can records extend classes?** No. Records implicitly extend `java.lang.Record`. They can implement interfaces though.

---

## Common Misconceptions

**"var is dynamic typing"**

No. `var` is static type inference. Once assigned, the type is fixed at compile time.

**"Records are just for DTOs"**

Records are great for DTOs, but they're also perfect for value objects, domain concepts, and anywhere you want immutable data with value semantics.

**"Sealed classes are like final classes"**

No. `final` means no subclasses. `sealed` means specific subclasses only. A sealed class explicitly permits certain extensions.

**"Text blocks add runtime overhead"**

No. Text blocks are processed at compile time. The runtime sees the same `String` as traditional concatenation.

**"Pattern variables in instanceof are just syntactic sugar"**

True, but powerful. They enable flow scoping and set up for future pattern matching features (switch expressions, record patterns, etc.).

---

## The Philosophy: Intent Over Mechanism

These features share a common theme: **expressing intent more directly**.

- `var`: "I care about the value, not repeating the type"
- Records: "This is immutable data with these components"
- Sealed classes: "These are the only valid subtypes"
- Text blocks: "This is the literal content, with natural formatting"
- Pattern instanceof: "Match this type and give me a typed reference"

Traditional Java forced you to express mechanism (constructor bodies, getter implementations, cast operations) when you wanted to express intent (what the data is, how types relate).

Modern Java bridges this gap. Your code says what you mean, and the compiler handles the details. The result is code that's not just shorter—it's clearer, more maintainable, and less prone to errors that come from repetitive boilerplate.

---

## Summary

| Feature | Java Version | Key Benefit |
|---------|--------------|-------------|
| `var` | 10 | Less redundant type declarations |
| Text Blocks | 15 | Readable multi-line strings |
| Records | 16 | Immutable data classes in one line |
| Pattern instanceof | 16 | Combine type check with cast |
| Sealed Classes | 17 | Controlled inheritance, exhaustive matching |

The combination enables patterns previously awkward in Java:
- Algebraic data types (sealed + records)
- Safe type narrowing (pattern instanceof)
- Domain modeling that's concise yet type-safe

Java's evolution continues to reduce the gap between what you want to express and what you have to write.

---

*Next Chapter: [Concurrency Without Tears](06-concurrency.md)*
