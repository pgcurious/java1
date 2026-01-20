# Chapter 9: The Hidden Machinery

*JVM internals that changed: G1/ZGC/Shenandoah, JIT improvements, escape analysis*

---

## Why Understanding the JVM Matters

You can write excellent Java without understanding the JVM's internals. But understanding them transforms how you think about code:

- Why does immutability often *improve* performance?
- Why do short-lived objects cost almost nothing?
- When does creating an object allocate no memory at all?
- Why might a simple loop outperform a parallel stream?

The JVM has evolved dramatically. Optimizations that seemed magical in 2010 are now routine. Techniques you learned to "help" the JVM might now hurt it. Let's explore the machinery.

---

## Memory Layout: How Objects Actually Live

### Heap Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                            JVM Heap                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────┐  ┌──────────────────┐  │
│  │              Young Generation           │  │  Old Generation  │  │
│  │                                         │  │                  │  │
│  │  ┌───────┐  ┌─────────┐  ┌─────────┐   │  │                  │  │
│  │  │ Eden  │  │Survivor │  │Survivor │   │  │  (long-lived     │  │
│  │  │       │  │  (S0)   │  │  (S1)   │   │  │   objects)       │  │
│  │  │ (new  │  │         │  │         │   │  │                  │  │
│  │  │objects)│  │(survived│  │(survived│   │  │                  │  │
│  │  │       │  │ 1 GC)   │  │ 2+ GCs) │   │  │                  │  │
│  │  └───────┘  └─────────┘  └─────────┘   │  │                  │  │
│  └─────────────────────────────────────────┘  └──────────────────┘  │
│                                                                      │
│  Objects flow: Eden → Survivor → Old Generation                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

This is the **generational hypothesis**: most objects die young. The JVM exploits this:

1. **Eden**: New objects allocate here (very fast—just bump a pointer)
2. **Minor GC**: When Eden fills, live objects copy to Survivor, dead ones vanish
3. **Aging**: Objects surviving multiple GCs get promoted to Old Generation
4. **Major GC**: Eventually, Old Generation needs collection (slower, less frequent)

### Object Headers

Every object has a header before its data:

```
┌────────────────────────────────────────────────────────────────┐
│                        Object in Memory                         │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Mark Word (8 bytes)                                       │  │
│  │ - Hash code, GC age, lock state                           │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Class Pointer (4-8 bytes)                                 │  │
│  │ - Pointer to class metadata                               │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Array Length (4 bytes, arrays only)                       │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Instance Data                                             │  │
│  │ - Your actual fields                                      │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │ Padding                                                   │  │
│  │ - Alignment to 8-byte boundary                            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Minimum object size: 16 bytes (even for empty object)         │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

This overhead matters. An `Integer` holding 4 bytes of data takes 16 bytes total. A `HashMap<Integer, Integer>` entry is ~50 bytes for 8 bytes of actual data.

---

## Garbage Collection: The Evolution

### G1 (Garbage-First) Collector

G1, default since Java 9, replaced the older CMS collector. Its innovation: **regionalized heap**.

```
┌─────────────────────────────────────────────────────────────────┐
│                        G1 Heap Regions                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐   │
│  │ E │ │ E │ │ S │ │ O │ │ O │ │ E │ │ O │ │ H │ │ H │ │ O │   │
│  └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘ └───┘   │
│                                                                  │
│  E = Eden, S = Survivor, O = Old, H = Humongous (large objects) │
│                                                                  │
│  Key insight: Collect regions with most garbage first            │
│  (hence "Garbage-First")                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

G1 characteristics:
- **Pause time target**: `-XX:MaxGCPauseMillis=200` (default)
- **Incremental**: Collects subsets of the heap
- **Compacting**: Reduces fragmentation
- **Concurrent marking**: Much work happens while application runs

### ZGC (Z Garbage Collector)

ZGC, available since Java 11 (production in 15), targets **sub-millisecond pauses** regardless of heap size.

```bash
java -XX:+UseZGC -Xmx16g MyApplication
```

ZGC achieves this through:
- **Colored pointers**: Metadata stored in pointer bits
- **Load barriers**: Small check on every reference load
- **Concurrent everything**: Almost all GC work concurrent with application

| Heap Size | ZGC Pause | G1 Pause (typical) |
|-----------|-----------|-------------------|
| 4 GB | < 1 ms | 10-50 ms |
| 16 GB | < 1 ms | 50-200 ms |
| 128 GB | < 1 ms | 200-1000 ms |

**Trade-off**: ~5-10% throughput overhead due to load barriers. Worth it for latency-sensitive applications.

### Shenandoah

Shenandoah, from Red Hat (available in OpenJDK), also targets low latency:

```bash
java -XX:+UseShenandoahGC MyApplication
```

Similar goals to ZGC, different implementation. Uses **Brooks pointers** (forwarding pointers) instead of colored pointers. Slightly different performance characteristics.

### Choosing a Collector

```
┌────────────────────────────────────────────────────────────────────┐
│                    GC Collector Decision Tree                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  What matters most?                                                 │
│  │                                                                  │
│  ├─► Throughput (batch processing, scientific computing)           │
│  │   └─► Use Parallel GC: -XX:+UseParallelGC                       │
│  │                                                                  │
│  ├─► Balanced (typical server, reasonable latency)                 │
│  │   └─► Use G1 (default): -XX:+UseG1GC                            │
│  │                                                                  │
│  └─► Ultra-low latency (trading, real-time)                        │
│      ├─► Heap < 4GB: G1 might suffice                              │
│      └─► Heap > 4GB: ZGC or Shenandoah                             │
│                                                                     │
│  Note: These are starting points. Always benchmark your workload!  │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### Tuning G1

Common G1 tuning options:

```bash
java \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \        # Target pause time (ms)
  -XX:G1HeapRegionSize=16m \        # Region size (1-32MB, auto-selected)
  -XX:InitiatingHeapOccupancyPercent=45 \  # Start concurrent marking
  -Xms4g -Xmx4g \                   # Heap size (fix for predictability)
  MyApplication
```

**Rule of thumb**: Set pause time target, let G1 figure out the rest. Only tune further if you understand your specific needs.

---

## JIT Compilation: Making Java Fast

Java code goes through stages:

```
┌────────────────────────────────────────────────────────────────────┐
│                     JIT Compilation Pipeline                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   .java  ──javac──►  .class (bytecode)                              │
│                         │                                           │
│                         ▼                                           │
│              ┌─────────────────────┐                                │
│              │     JVM Startup     │                                │
│              │   (Interpreter)     │  ◄── Slow but starts fast      │
│              └──────────┬──────────┘                                │
│                         │                                           │
│              Method called many times                               │
│                         │                                           │
│                         ▼                                           │
│              ┌─────────────────────┐                                │
│              │    C1 Compiler      │  ◄── Quick compilation         │
│              │  (Client Compiler)  │      Moderate optimization     │
│              └──────────┬──────────┘                                │
│                         │                                           │
│              Method still hot                                       │
│                         │                                           │
│                         ▼                                           │
│              ┌─────────────────────┐                                │
│              │    C2 Compiler      │  ◄── Aggressive optimization   │
│              │  (Server Compiler)  │      Peak performance          │
│              └─────────────────────┘                                │
│                                                                     │
│   This is "tiered compilation" - default since Java 8               │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### What the JIT Does

The JIT (Just-In-Time compiler) converts bytecode to native machine code, applying optimizations:

**Inlining**: Replace method call with method body
```java
// Before inlining
int result = square(5);

int square(int x) { return x * x; }

// After inlining
int result = 5 * 5;  // No method call overhead!
```

**Loop unrolling**: Reduce loop overhead
```java
// Before
for (int i = 0; i < 4; i++) { sum += array[i]; }

// After unrolling
sum += array[0];
sum += array[1];
sum += array[2];
sum += array[3];
```

**Dead code elimination**: Remove unreachable code

**Common subexpression elimination**: Compute once, reuse
```java
// Before
int a = x * y + z;
int b = x * y + w;

// After
int temp = x * y;
int a = temp + z;
int b = temp + w;
```

**Lock elision**: Remove unnecessary synchronization (when JIT proves it's safe)

### Escape Analysis: The Magic Optimization

**Escape analysis** determines whether an object "escapes" the method or thread that created it. If it doesn't escape, the JIT can:

1. **Stack allocation**: Allocate on stack instead of heap (no GC!)
2. **Scalar replacement**: Replace object with its fields (no object at all!)
3. **Lock elision**: Remove synchronization on non-escaping objects

```java
public int sumPoints() {
    Point p = new Point(3, 4);  // Does p escape?
    return p.x + p.y;           // No! It's local.
}

// JIT can transform to:
public int sumPoints() {
    int p_x = 3;
    int p_y = 4;
    return p_x + p_y;  // No object created!
}
```

**This is why short-lived objects are cheap**. The JIT often eliminates them entirely.

### When Escape Analysis Fails

```java
public Point createPoint() {
    Point p = new Point(3, 4);
    return p;  // p escapes! Caller has reference.
}

public void storePoint(List<Point> list) {
    Point p = new Point(3, 4);
    list.add(p);  // p escapes! Stored in collection.
}

public void logPoint() {
    Point p = new Point(3, 4);
    System.out.println(p);  // p escapes! Passed to another method.
}
```

**Key insight**: Objects passed to methods you don't control are assumed to escape. The JIT can't analyze the entire world.

### Viewing JIT Output

```bash
# Print compilation events
java -XX:+PrintCompilation MyApp

# Print inlining decisions
java -XX:+PrintInlining MyApp

# Full assembly output (requires hsdis library)
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly MyApp
```

Or use JITWatch (a visualization tool) for detailed analysis.

---

## String Optimizations

Strings are everywhere, so the JVM optimizes them heavily.

### String Interning

The **string pool** stores unique string instances:

```java
String s1 = "hello";           // Goes to pool
String s2 = "hello";           // Same object from pool
String s3 = new String("hello"); // New object, not pooled

System.out.println(s1 == s2);  // true (same object)
System.out.println(s1 == s3);  // false (different objects)

s3 = s3.intern();              // Now add to pool
System.out.println(s1 == s3);  // true (same object)
```

### Compact Strings (Java 9+)

Before Java 9, strings always used `char[]` (2 bytes per character). Now:
- Latin-1 strings (ASCII, most European languages): 1 byte per character
- Other strings: 2 bytes per character

Memory savings can be dramatic for English text-heavy applications.

### String Concatenation Evolution

```java
String result = "Hello, " + name + "!";
```

**Java 8**: Compiled to `StringBuilder.append()` chains
**Java 9+**: Uses `invokedynamic` to call a bootstrap method that chooses the best strategy at runtime

The new approach allows future JVM improvements without recompiling code.

---

## Value Objects: Project Valhalla (Future)

A fundamental JVM change is coming: **value types** (or "primitive objects").

Currently, even simple objects like `Point` require:
- Heap allocation
- Object header (16+ bytes)
- Indirection (reference to data)

Value types will be:
- Stored inline (like primitives)
- No identity (like primitives)
- Immutable
- No object header

```java
// Future syntax (preview)
value class Point {
    int x;
    int y;
}

// Memory layout: just 8 bytes for two ints, no header!
// Arrays of Point: contiguous memory, cache-friendly
```

This will dramatically improve:
- Memory efficiency (no headers, no references)
- Cache locality (contiguous layout)
- GC pressure (inline values don't need collection)

---

## Practical Performance Insights

### Why Immutability Can Be Faster

Counter-intuitive but true:

```java
// Mutable approach
StringBuilder sb = new StringBuilder();
for (String s : strings) {
    sb.append(s);
}
return sb.toString();

// Immutable approach (with streams)
return strings.stream().collect(Collectors.joining());
```

The immutable approach might be faster because:
1. **No defensive copying**: Caller can't modify, so callee doesn't copy
2. **Escape analysis**: Immutable temporaries often eliminated entirely
3. **Thread safety**: No synchronization needed
4. **JIT confidence**: Immutable means predictable—easier to optimize

### Why Allocation Is (Usually) Cheap

Modern allocation is just bumping a pointer:

```java
Object obj = new Object();  // Bump pointer, zero memory, done
```

No free-list traversal, no fragmentation worries. And if the object doesn't escape, it might not allocate at all.

**Expensive allocation** happens when:
- Eden is full (triggers minor GC)
- Object is large (goes to humongous region)
- GC is concurrent and memory is tight

### The Premature Optimization Problem

Old "wisdom" that the JIT makes obsolete:

**"Cache object in field to avoid allocation"**
```java
// "Optimization" that might hurt
class Calculator {
    private StringBuilder buffer = new StringBuilder(); // Reuse!

    public String format(int value) {
        buffer.setLength(0);  // Reset
        buffer.append("Value: ").append(value);
        return buffer.toString();
    }
}
```

Problems:
- Not thread-safe
- Object escapes (field), so can't be eliminated
- Might be slower than just creating a new StringBuilder (which can be escaped-analyzed)

**"Use primitives always"**
```java
// "Optimization"
int[] values = ...;  // Primitives!

// vs
List<Integer> values = ...; // Objects, boxed
```

Sometimes primitives are faster. But boxed objects benefit from:
- Caching small values (-128 to 127)
- Null semantics
- Generic compatibility
- Future value types will eliminate the penalty

### Measuring, Not Guessing

Use JMH (Java Microbenchmark Harness) for reliable benchmarks:

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5)
@Measurement(iterations = 10)
public class MyBenchmark {

    @Benchmark
    public int testMethod() {
        return performOperation();
    }
}
```

JMH handles:
- Warmup (let JIT optimize)
- Dead code prevention
- Statistics
- JVM quirks

Never benchmark without it.

---

## What Interviewers Actually Want to Know

1. **What is escape analysis?** JIT optimization that determines if an object escapes its scope. Non-escaping objects can be stack-allocated or eliminated entirely.

2. **How does G1 differ from older GCs?** Regionalized heap, pause time targets, concurrent marking. Trades some throughput for predictable latency.

3. **When would you use ZGC over G1?** Ultra-low latency requirements (< 1ms pauses), large heaps where G1 pauses would be unacceptable.

4. **What happens during a minor GC?** Eden space is scanned, live objects copied to Survivor, Eden wiped. Fast because most objects are dead.

5. **Why might immutable objects be faster?** No defensive copying, escape analysis can eliminate them, no synchronization needed, JIT can optimize more aggressively.

---

## Common Misconceptions

**"Creating objects is expensive"**

Allocation is cheap (pointer bump). Short-lived objects are often free (escape analysis). GC is generational—young objects are collected fast.

**"You should always use primitives over objects"**

Depends. Boxed types cache small values, enable generics, support null. JIT optimization can reduce the gap. Measure, don't assume.

**"Bigger heap is always better"**

Bigger heap means longer full GCs (though less frequent). For latency-sensitive apps, a right-sized heap with frequent small GCs might be better.

**"Turning off GC makes things faster"**

Epsilon GC (the no-op GC) exists for testing and ultra-short-lived apps. For anything else, you'll run out of memory and crash.

**"The JIT will optimize everything"**

The JIT needs time to warm up and makes conservative assumptions. Hot paths get optimized; cold paths don't. Megamorphic call sites (many different types) defeat inlining.

---

## JVM Flags Worth Knowing

```bash
# Memory
-Xms4g                          # Initial heap size
-Xmx4g                          # Maximum heap size
-XX:MaxMetaspaceSize=256m       # Metaspace limit

# GC Selection
-XX:+UseG1GC                    # G1 (default in Java 9+)
-XX:+UseZGC                     # ZGC (low latency)
-XX:+UseShenandoahGC            # Shenandoah (low latency)
-XX:+UseParallelGC              # Parallel (throughput)

# G1 Tuning
-XX:MaxGCPauseMillis=200        # Pause time target
-XX:G1HeapRegionSize=16m        # Region size

# Diagnostics
-XX:+PrintGCDetails             # GC logging (deprecated, use below)
-Xlog:gc*                       # Unified logging for GC
-XX:+HeapDumpOnOutOfMemoryError # Dump heap on OOM
-XX:HeapDumpPath=/tmp/          # Where to dump
```

---

## Summary

| Topic | Key Insight |
|-------|-------------|
| Generational GC | Most objects die young; exploit this for efficiency |
| G1 | Regionalized, pause-time-targeted, good default |
| ZGC/Shenandoah | Sub-millisecond pauses, for latency-sensitive apps |
| JIT Compilation | Bytecode → interpreted → C1 → C2, progressively optimized |
| Escape Analysis | Non-escaping objects can be eliminated entirely |
| Inlining | Hot methods inlined, enabling further optimizations |
| Object Overhead | 16+ bytes per object; primitives and future value types avoid this |

Understanding the JVM transforms how you write code:
- Don't fear allocations (short-lived objects are cheap)
- Prefer immutability (enables optimization)
- Write clear code first (JIT optimizes predictable patterns)
- Measure before optimizing (your intuition is often wrong)
- Trust the runtime (it's smarter than you think)

The JVM has evolved from a simple bytecode interpreter into a sophisticated adaptive optimization system. Write clean, idiomatic code, and let it do its job.

---

*This concludes "Java Through First Principles." May your understanding of Java be forever deeper than "that's just how it works."*
