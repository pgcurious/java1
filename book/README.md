# Java Through First Principles

*A book for engineers who refuse to accept "that's just how it works"*

---

## Who This Book Is For

You're not here to learn syntax. You're here because you want to understand *why* Java evolved the way it did, *what problems* each feature actually solves, and *how* to think about these concepts deeply enough to teach them to others—or ace a FAANG interview where the interviewer wants to see you *think*, not just recite.

This book assumes you can already write Java code. What it offers is the mental framework to understand Java at a level where features stop feeling arbitrary and start feeling inevitable.

---

## The Philosophy

Every language feature exists because engineers were frustrated. Someone, somewhere, was writing code that was verbose, error-prone, or impossible to parallelize. They complained. They proposed solutions. Committees debated trade-offs. And eventually, something shipped.

Understanding that process—the *why* behind the *what*—transforms you from a Java user into a Java thinker.

We'll use analogies extensively. Not because programming is like cooking or building houses, but because good analogies create mental hooks that survive under interview pressure and help you explain concepts to your team at 2 AM during a production incident.

---

## Table of Contents

### [Chapter 1: The Evolution of Handling Absence](chapters/01-handling-absence.md)
*Optional, null-safety patterns, and why the billion-dollar mistake matters*

The story of how Tony Hoare's "billion-dollar mistake" haunted Java for two decades, and how Optional represents not just a new class, but a fundamental shift in how we communicate intent through types.

### [Chapter 2: Functions as First-Class Citizens](chapters/02-functions-first-class.md)
*Lambdas, functional interfaces, method references—what shifted in thinking*

Before Java 8, behavior was trapped inside objects. We'll trace the journey from anonymous inner classes to lambdas, understand why `Runnable` is a functional interface, and see how method references complete the picture.

### [Chapter 3: Streams: Declarative Data Processing](chapters/03-streams.md)
*Lazy evaluation, internal vs external iteration, parallel streams and their hidden costs*

Streams aren't just a different way to loop. They represent a fundamental inversion of control—from telling the computer *how* to iterate to telling it *what* you want. We'll explore why laziness matters and when parallel streams are actually slower.

### [Chapter 4: Time Done Right](chapters/04-time-done-right.md)
*java.time API, why Date/Calendar was broken, immutability in temporal design*

The old Date API wasn't just inconvenient—it was fundamentally broken in ways that caused real production bugs. We'll understand the principles behind JSR-310 and why immutability isn't just a nice-to-have for temporal data.

### [Chapter 5: Memory and References Reimagined](chapters/05-memory-references.md)
*var, records, sealed classes—reducing ceremony, increasing intent*

Java's reputation for verbosity isn't unfounded. But recent versions have introduced features that aren't about being trendy—they're about making intent clearer and boilerplate disappear. We'll see how records and sealed classes enable algebraic data types in Java.

### [Chapter 6: Concurrency Without Tears](chapters/06-concurrency.md)
*CompletableFuture, virtual threads (Loom), structured concurrency—the journey from threads to fibers*

Concurrency in Java has evolved from "create a Thread and pray" to sophisticated abstractions. We'll trace this evolution, understand why CompletableFuture exists, and see how virtual threads change the game entirely.

### [Chapter 7: Pattern Matching and Algebraic Thinking](chapters/07-pattern-matching.md)
*Switch expressions, record patterns, instanceof patterns—Java's gradual move toward expressiveness*

Pattern matching isn't just syntactic sugar. It's Java's acknowledgment that sometimes you need to deconstruct data, and that checked exhaustiveness prevents bugs. We'll see how this connects to sealed classes and algebraic data types.

### [Chapter 8: Modules and Encapsulation at Scale](chapters/08-modules.md)
*JPMS, why classpath hell existed, strong encapsulation*

For 20 years, Java had no answer to "which version of this library are we actually using?" JPMS attempts to solve this, but understanding it requires understanding what was broken about the classpath model.

### [Chapter 9: The Hidden Machinery](chapters/09-hidden-machinery.md)
*JVM internals that changed: G1/ZGC/Shenandoah, JIT improvements, escape analysis*

You can write excellent Java without knowing how garbage collection works. But understanding the JVM's evolution helps you write code that plays to its strengths and avoid patterns that fight against it.

---

## How to Read This Book

Each chapter stands alone, but they build on each other conceptually. If you're preparing for interviews, Chapters 1-3 and 6-7 are highest-yield. If you're trying to understand modern Java idioms, read front-to-back.

Code examples are designed to be copied and run. Edge cases are highlighted because that's where understanding lives.

---

## A Note on Java Versions

This book covers Java 8 through Java 21+, but it's organized by *concept*, not by release. Features are introduced where they make conceptual sense, with clear notes about which version introduced them.

The goal is understanding that survives version upgrades.

---

*Let's begin.*
