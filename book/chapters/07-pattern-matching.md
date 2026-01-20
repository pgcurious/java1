# Chapter 7: Pattern Matching and Algebraic Thinking

*Switch expressions, record patterns, instanceof patterns—Java's gradual move toward expressiveness*

---

## The Question Pattern Matching Answers

You have an object. You want to know: What *kind* of thing is it, and what data does it contain?

In classic Java, answering this question is verbose:

```java
if (shape instanceof Circle) {
    Circle circle = (Circle) shape;
    return Math.PI * circle.radius() * circle.radius();
} else if (shape instanceof Rectangle) {
    Rectangle rect = (Rectangle) shape;
    return rect.width() * rect.height();
} else if (shape instanceof Triangle) {
    Triangle tri = (Triangle) shape;
    double s = (tri.a() + tri.b() + tri.c()) / 2;
    return Math.sqrt(s * (s - tri.a()) * (s - tri.b()) * (s - tri.c()));
} else {
    throw new IllegalArgumentException("Unknown shape: " + shape);
}
```

Three operations repeated for each case:
1. **Test**: Is it this type?
2. **Cast**: Convert to that type
3. **Extract**: Get the data out

Pattern matching collapses this into a single operation: "Match this pattern, bind these variables, run this code."

---

## The Evolution of Pattern Matching in Java

```
Java 14: Switch expressions (preview → standard in 16)
Java 16: Pattern matching for instanceof
Java 21: Pattern matching for switch, record patterns
Future:  Array patterns, collection patterns, more
```

Java is adding pattern matching incrementally, each feature building on the last. Let's trace this evolution.

---

## Switch Expressions (Java 14+)

### The Old Switch: Statement-Based

```java
String dayType;
switch (day) {
    case MONDAY:
    case TUESDAY:
    case WEDNESDAY:
    case THURSDAY:
    case FRIDAY:
        dayType = "Weekday";
        break;
    case SATURDAY:
    case SUNDAY:
        dayType = "Weekend";
        break;
    default:
        throw new IllegalArgumentException();
}
```

Problems:
1. **Fall-through by default**: Forget `break`, bugs happen
2. **Verbose**: Each case needs explicit break
3. **Not an expression**: Can't assign result directly
4. **Variable scope issues**: Variables declared in cases leak across cases

### The New Switch: Expression-Based

```java
String dayType = switch (day) {
    case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Weekday";
    case SATURDAY, SUNDAY -> "Weekend";
};
```

Changes:
1. **Arrow syntax (`->`)**: No fall-through
2. **Expression**: Returns a value
3. **Exhaustiveness**: Must cover all cases (or have default)
4. **Multiple labels**: Comma-separated cases

### Arrow vs Colon Syntax

Both syntaxes work in switch expressions:

```java
// Arrow syntax: no fall-through, single expression or block
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY -> 7;
    case THURSDAY, SATURDAY -> 8;
    case WEDNESDAY -> 9;
};

// Colon syntax: traditional fall-through rules, use yield for value
int numLetters = switch (day) {
    case MONDAY:
    case FRIDAY:
    case SUNDAY:
        yield 6;
    case TUESDAY:
        yield 7;
    default:
        yield -1;
};
```

`yield` is the "return for switch expressions" keyword.

### Blocks and yield

When you need multiple statements:

```java
String formatted = switch (status) {
    case PENDING -> "⏳ Pending";
    case APPROVED -> {
        logApproval();
        notifyUser();
        yield "✅ Approved";  // yield returns value from block
    }
    case REJECTED -> {
        logRejection();
        yield "❌ Rejected";
    }
};
```

### Exhaustiveness

Switch expressions must be exhaustive:

```java
enum Status { PENDING, APPROVED, REJECTED }

// Compile error! Missing REJECTED
String text = switch (status) {
    case PENDING -> "pending";
    case APPROVED -> "approved";
};

// Fixed: cover all cases
String text = switch (status) {
    case PENDING -> "pending";
    case APPROVED -> "approved";
    case REJECTED -> "rejected";
};

// Or use default
String text = switch (status) {
    case APPROVED -> "approved";
    default -> "not approved";
};
```

For sealed types, the compiler knows all subtypes:

```java
sealed interface Shape permits Circle, Rectangle, Triangle { }

// No default needed—compiler knows this is exhaustive
double area = switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t -> triangleArea(t);
};
```

---

## Pattern Matching for instanceof (Java 16)

### The Problem

```java
if (obj instanceof String) {
    String s = (String) obj;  // Redundant!
    System.out.println(s.length());
}
```

You already checked it's a String. Why cast again?

### The Solution

```java
if (obj instanceof String s) {
    System.out.println(s.length());  // s is already String
}
```

The pattern `String s` does three things:
1. Tests if `obj` is a `String`
2. Casts to `String`
3. Binds to variable `s`

### Flow Scoping: The Smart Part

The pattern variable is in scope where the compiler can prove the pattern matched:

```java
// In the if-body, we know the pattern matched
if (obj instanceof String s) {
    System.out.println(s.length());
}

// With &&, if we reach the right side, left side matched
if (obj instanceof String s && s.length() > 5) {
    System.out.println(s);
}

// With ||, we might reach here even if pattern didn't match
if (obj instanceof String s || s.length() > 5) {  // Compile error!
    // s might not be bound
}

// Negation flips scope
if (!(obj instanceof String s)) {
    return;  // s not in scope here
}
// s IS in scope here—we only reach if pattern matched
System.out.println(s.length());
```

### Practical Uses

**Equals implementation:**
```java
@Override
public boolean equals(Object o) {
    return o instanceof Point p
        && this.x == p.x
        && this.y == p.y;
}
```

**Visitor pattern simplified:**
```java
void process(Node node) {
    if (node instanceof BinaryOp op) {
        process(op.left());
        process(op.right());
    } else if (node instanceof UnaryOp op) {
        process(op.operand());
    } else if (node instanceof Literal lit) {
        System.out.println(lit.value());
    }
}
```

---

## Pattern Matching for switch (Java 21)

### Combining Powers

Now patterns work in switch expressions:

```java
Object obj = ...;

String result = switch (obj) {
    case Integer i -> "Integer: " + i;
    case Long l -> "Long: " + l;
    case Double d -> "Double: " + d;
    case String s -> "String: " + s;
    case null -> "null";
    case default -> "Unknown: " + obj.getClass();
};
```

### Type Patterns

```java
String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "int " + i;
        case Long l -> "long " + l;
        case Double d -> "double " + d;
        case String s -> "String length " + s.length();
        case int[] arr -> "int array of length " + arr.length;
        case null -> "null";
        default -> "other";
    };
}
```

### Guarded Patterns (when clause)

Add conditions to patterns:

```java
String describe(Integer value) {
    return switch (value) {
        case Integer i when i < 0 -> "negative";
        case Integer i when i == 0 -> "zero";
        case Integer i when i > 0 -> "positive";
        case null -> "null";
    };
}
```

**Order matters!** More specific patterns must come before general ones:

```java
// Correct: specific before general
switch (obj) {
    case String s when s.isEmpty() -> "empty string";
    case String s -> "string: " + s;
    case null, default -> "other";
}

// Compile error: unreachable pattern
switch (obj) {
    case String s -> "string: " + s;
    case String s when s.isEmpty() -> "empty string";  // Never reached!
}
```

### null Handling

Historically, switch threw NPE on null. Now you can handle it:

```java
switch (str) {
    case null -> System.out.println("null!");
    case "yes" -> System.out.println("affirmative");
    case "no" -> System.out.println("negative");
    default -> System.out.println("unknown");
}

// Or combine with default
switch (str) {
    case "yes" -> "affirmative";
    case "no" -> "negative";
    case null, default -> "unknown";
}
```

---

## Record Patterns (Java 21)

### Deconstructing Records

Records have a **canonical constructor**—the pattern matching complement is a **deconstruction pattern**:

```java
record Point(int x, int y) { }

Object obj = new Point(3, 4);

if (obj instanceof Point(int x, int y)) {
    System.out.println("x=" + x + ", y=" + y);  // x=3, y=4
}
```

The pattern `Point(int x, int y)` destructures the record, binding its components to variables.

### Nested Patterns

Patterns compose:

```java
record Point(int x, int y) { }
record Line(Point start, Point end) { }

Object obj = new Line(new Point(0, 0), new Point(10, 10));

if (obj instanceof Line(Point(int x1, int y1), Point(int x2, int y2))) {
    double length = Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
}
```

One pattern destructures three objects!

### In Switch Expressions

```java
sealed interface Expr permits Num, Add, Mul { }
record Num(int value) implements Expr { }
record Add(Expr left, Expr right) implements Expr { }
record Mul(Expr left, Expr right) implements Expr { }

int eval(Expr expr) {
    return switch (expr) {
        case Num(int value) -> value;
        case Add(var left, var right) -> eval(left) + eval(right);
        case Mul(var left, var right) -> eval(left) * eval(right);
    };
}
```

Notice: `var` works in patterns too. The compiler infers the type.

### Combining with Guards

```java
String classifyTriangle(Triangle t) {
    return switch (t) {
        case Triangle(var a, var b, var c) when a == b && b == c -> "equilateral";
        case Triangle(var a, var b, var c) when a == b || b == c || a == c -> "isosceles";
        case Triangle(_, _, _) -> "scalene";  // _ is unnamed pattern
    };
}
```

### Unnamed Patterns (_)

When you don't need a binding:

```java
if (obj instanceof Point(int x, _)) {
    // Only care about x coordinate
}

switch (expr) {
    case Add(_, _) -> "addition";  // Don't need operands
    case Mul(_, _) -> "multiplication";
    case Num(_) -> "number";
}
```

---

## Algebraic Data Types in Java

### The Concept

**Algebraic data types** (ADTs) combine:
- **Sum types**: "One of these types" (A | B | C)
- **Product types**: "Contains these fields" (A × B × C)

Functional languages have had these forever. Now Java does too:

```java
// Sum type: sealed interface with permits
sealed interface Json permits JsonNull, JsonBool, JsonNumber, JsonString, JsonArray, JsonObject { }

// Product types: records
record JsonNull() implements Json { }
record JsonBool(boolean value) implements Json { }
record JsonNumber(double value) implements Json { }
record JsonString(String value) implements Json { }
record JsonArray(List<Json> elements) implements Json { }
record JsonObject(Map<String, Json> fields) implements Json { }
```

### Processing ADTs

Pattern matching makes processing elegant:

```java
String toJsonString(Json json) {
    return switch (json) {
        case JsonNull() -> "null";
        case JsonBool(boolean b) -> String.valueOf(b);
        case JsonNumber(double n) -> String.valueOf(n);
        case JsonString(String s) -> "\"" + escape(s) + "\"";
        case JsonArray(var elements) -> elements.stream()
            .map(this::toJsonString)
            .collect(joining(", ", "[", "]"));
        case JsonObject(var fields) -> fields.entrySet().stream()
            .map(e -> "\"" + e.getKey() + "\": " + toJsonString(e.getValue()))
            .collect(joining(", ", "{", "}"));
    };
}
```

Every case is handled. Add a new Json variant? The compiler tells you every switch that needs updating.

### State Machines

```java
sealed interface ConnectionState permits Disconnected, Connecting, Connected, Failed { }

record Disconnected() implements ConnectionState { }
record Connecting(String host, int port) implements ConnectionState { }
record Connected(Socket socket) implements ConnectionState { }
record Failed(Exception error) implements ConnectionState { }

class Connection {
    private ConnectionState state = new Disconnected();

    public void connect(String host, int port) {
        state = switch (state) {
            case Disconnected() -> {
                startConnect(host, port);
                yield new Connecting(host, port);
            }
            case Connecting(_, _) -> throw new IllegalStateException("Already connecting");
            case Connected(_) -> throw new IllegalStateException("Already connected");
            case Failed(_) -> {
                startConnect(host, port);
                yield new Connecting(host, port);
            }
        };
    }

    public void onConnected(Socket socket) {
        state = switch (state) {
            case Connecting(_, _) -> new Connected(socket);
            case Disconnected(), Connected(_), Failed(_) ->
                throw new IllegalStateException("Unexpected connection");
        };
    }
}
```

The compiler enforces that every state handles every transition.

---

## Real-World Example: Expression Evaluator

Let's build a complete expression language:

```java
// The AST
sealed interface Expr permits Literal, Var, BinOp, UnaryOp, Let { }

record Literal(double value) implements Expr { }
record Var(String name) implements Expr { }
record BinOp(String op, Expr left, Expr right) implements Expr { }
record UnaryOp(String op, Expr operand) implements Expr { }
record Let(String name, Expr value, Expr body) implements Expr { }

// The evaluator
class Evaluator {
    public double eval(Expr expr, Map<String, Double> env) {
        return switch (expr) {
            case Literal(double value) -> value;

            case Var(String name) -> {
                if (!env.containsKey(name)) {
                    throw new RuntimeException("Undefined: " + name);
                }
                yield env.get(name);
            }

            case BinOp(String op, var left, var right) -> {
                double l = eval(left, env);
                double r = eval(right, env);
                yield switch (op) {
                    case "+" -> l + r;
                    case "-" -> l - r;
                    case "*" -> l * r;
                    case "/" -> l / r;
                    case "^" -> Math.pow(l, r);
                    default -> throw new RuntimeException("Unknown op: " + op);
                };
            }

            case UnaryOp(String op, var operand) -> {
                double v = eval(operand, env);
                yield switch (op) {
                    case "-" -> -v;
                    case "sqrt" -> Math.sqrt(v);
                    case "abs" -> Math.abs(v);
                    default -> throw new RuntimeException("Unknown op: " + op);
                };
            }

            case Let(String name, var value, var body) -> {
                double v = eval(value, env);
                var newEnv = new HashMap<>(env);
                newEnv.put(name, v);
                yield eval(body, newEnv);
            }
        };
    }
}
```

Usage:

```java
// let x = 3 in let y = 4 in sqrt(x^2 + y^2)
Expr pythagorean = new Let("x", new Literal(3),
    new Let("y", new Literal(4),
        new UnaryOp("sqrt",
            new BinOp("+",
                new BinOp("^", new Var("x"), new Literal(2)),
                new BinOp("^", new Var("y"), new Literal(2))))));

double result = new Evaluator().eval(pythagorean, Map.of());
// result = 5.0
```

---

## What Interviewers Actually Want to Know

1. **What's the difference between switch statement and switch expression?** Expression returns a value, requires exhaustiveness, arrow syntax prevents fall-through.

2. **What is pattern matching in Java?** Combining type test, cast, and variable binding in one operation. Works in instanceof and switch.

3. **What are record patterns?** Patterns that destructure records, extracting their components. Can be nested.

4. **Why does exhaustiveness matter?** The compiler verifies you handle all cases. With sealed types, adding a new case becomes a compile-time change, not a runtime bug.

5. **How do sealed types and pattern matching work together?** Sealed types define a closed set of subtypes. Switch can use this knowledge to verify exhaustiveness without default case.

---

## Common Misconceptions

**"Switch expressions are just syntactic sugar"**

They enable exhaustiveness checking and remove fall-through bugs. That's semantic improvement, not just syntax.

**"Pattern matching replaces the Visitor pattern"**

For simple cases, yes. For complex operations that need to add new behaviors without modifying types, Visitor still has a place.

**"Record patterns only work with records"**

Currently true. Future Java may allow custom deconstructors for any class.

**"Unnamed patterns (_) can be used anywhere"**

They're for patterns where you don't need the value. You can't use `_` as a regular variable name (it's now reserved).

**"when guards make patterns turing-complete"**

Guards are just conditions. The pattern part (type + structure) is still static.

---

## The Future

Java continues to evolve pattern matching:

- **Array patterns**: `case int[] { first, second, rest... }`
- **Collection patterns**: `case List.of(first, second)`
- **And-patterns**: `case String s && Integer i` (not real syntax)
- **Or-patterns**: `case Circle c | Rectangle r`
- **More deconstruction**: Custom deconstruction patterns for non-records

The direction is clear: make Java more expressive for data-oriented programming while maintaining static type safety.

---

## Summary

| Feature | Java Version | Purpose |
|---------|--------------|---------|
| Switch expressions | 14 (16 final) | Expression-based switch, no fall-through |
| instanceof patterns | 16 | Combine test + cast + binding |
| Switch patterns | 21 | Type patterns in switch |
| Record patterns | 21 | Deconstruct records in patterns |
| when guards | 21 | Add conditions to patterns |

Pattern matching transforms how you think about code:
- **Old**: "Check type, cast, extract fields, process"
- **New**: "Match this structure, run this code"

Combined with sealed types and records, Java now supports algebraic data types—a cornerstone of functional programming, finally available in a mainstream language.

---

*Next Chapter: [Modules and Encapsulation at Scale](08-modules.md)*
