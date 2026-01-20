# Chapter 8: Modules and Encapsulation at Scale

*JPMS, why classpath hell existed, strong encapsulation*

---

## The Problem: Classpath Hell

It's 3 AM. Production is down. The error message:

```
java.lang.NoSuchMethodError: com.google.common.collect.ImmutableList.of(...)
```

But you definitely have Guava on your classpath. You've *always* had Guava. What happened?

After two hours, you discover: one library depends on Guava 19, another on Guava 28. The classloader loaded Guava 19 first (because of alphabetical jar naming), and Guava 28's method doesn't exist in version 19.

Welcome to **classpath hell**.

### Why the Classpath Model Failed

For 20 years, Java's deployment model was simple:

1. Compile your code to `.class` files
2. Bundle them (optionally) into `.jar` files
3. Put all jars on the classpath
4. The classloader finds classes by searching jars in order

This model has fundamental problems:

**No versioning**: The classpath doesn't understand versions. `guava-19.jar` and `guava-28.jar` can't coexist—the classloader sees classes named `com.google.common.collect.ImmutableList` and picks one.

**No encapsulation**: Any public class is accessible to any code. Library authors mark implementation classes as `public` (because they need them in other packages), and users depend on those internal APIs. Then the library changes internals, and users' code breaks.

**No reliable configuration**: Split packages (same package in multiple jars) cause undefined behavior. Missing dependencies are discovered at runtime, not at startup.

**No dependency declarations**: A jar doesn't declare what it needs. You discover missing dependencies by getting `NoClassDefFoundError` at runtime.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      The Classpath Model                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   app.jar ─────────────────────────────────────────────────────┐    │
│                                                                │    │
│   lib-a.jar ─────────────────────┬───────────────────────────┐ │    │
│                                  │                           │ │    │
│   lib-b.jar ───────────────────────────────────────────────┐ │ │    │
│                                  │                         │ │ │    │
│   guava-19.jar ─────────────────────────────────────────┐  │ │ │    │
│                                  │                      │  │ │ │    │
│   guava-28.jar ──────────────────────────────────────┐  │  │ │ │    │
│                                  ▼                   │  │  │ │ │    │
│                            ┌──────────┐              │  │  │ │ │    │
│                            │Classloader│◄────────────┴──┴──┴─┴─┘    │
│                            └──────────┘                             │
│                            "First one wins"                         │
│                                                                     │
│   Problem: Which Guava? No one knows until runtime!                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## The Java Platform Module System (JPMS)

Java 9 introduced the Java Platform Module System—a fundamental restructuring of how Java code is organized, encapsulated, and deployed.

### The Core Concepts

A **module** is:
1. A named, versioned collection of packages
2. With explicit dependency declarations
3. And explicit accessibility controls

```java
// module-info.java
module com.myapp.orders {
    // What we need
    requires com.myapp.customers;
    requires com.myapp.inventory;
    requires java.sql;

    // What we expose
    exports com.myapp.orders.api;
    exports com.myapp.orders.model;

    // Internal packages are NOT exported (truly encapsulated)
}
```

### module-info.java: The Module Descriptor

Every module has a `module-info.java` at its root:

```
src/
├── module-info.java          ← Module descriptor
└── com/
    └── myapp/
        └── orders/
            ├── api/
            │   └── OrderService.java
            ├── model/
            │   └── Order.java
            └── internal/
                └── OrderRepository.java   ← Not exported!
```

### The requires Directive

Declares dependencies:

```java
module com.myapp.orders {
    requires com.myapp.customers;      // Regular dependency
    requires transitive java.sql;      // Transitive: our consumers also get java.sql
    requires static com.myapp.metrics; // Optional: only needed at compile time
}
```

**requires transitive** is powerful: if your public API exposes types from another module, your consumers need access to that module. `requires transitive` makes this automatic.

```java
// com.myapp.orders exposes Customer in its API
module com.myapp.orders {
    requires transitive com.myapp.customers;  // Consumers get Customer too
    exports com.myapp.orders.api;
}

// com.myapp.webapp depends on orders
module com.myapp.webapp {
    requires com.myapp.orders;
    // Automatically gets com.myapp.customers due to transitive
}
```

### The exports Directive

Controls what's accessible:

```java
module com.myapp.orders {
    exports com.myapp.orders.api;                           // Public to all
    exports com.myapp.orders.spi to com.myapp.premium;      // Qualified export
    // com.myapp.orders.internal is NOT exported—truly private
}
```

Non-exported packages are **strongly encapsulated**. Even public classes in non-exported packages are inaccessible to other modules. This is stronger than Java's traditional access control.

### The opens Directive (For Reflection)

Frameworks like Spring, Hibernate, and Jackson use reflection to access private fields. The `opens` directive allows this:

```java
module com.myapp.orders {
    exports com.myapp.orders.api;

    // Allow reflection on these packages
    opens com.myapp.orders.model to com.fasterxml.jackson.databind;
    opens com.myapp.orders.entities to org.hibernate.orm.core;

    // Open everything to everything (not recommended)
    // open module com.myapp.orders { ... }
}
```

**exports vs opens**:
- `exports`: Compile-time access to public types
- `opens`: Runtime reflective access to all types (even private)

### The uses and provides Directives (Service Loading)

The module system formalizes Java's `ServiceLoader` pattern:

```java
// The API module
module com.myapp.payments.api {
    exports com.myapp.payments;
    uses com.myapp.payments.PaymentProcessor;  // We consume this service
}

// An implementation module
module com.myapp.payments.stripe {
    requires com.myapp.payments.api;
    provides com.myapp.payments.PaymentProcessor
        with com.myapp.payments.stripe.StripeProcessor;
}
```

At runtime, `ServiceLoader.load(PaymentProcessor.class)` finds all implementations.

---

## Modularizing the JDK

The JDK itself was modularized in Java 9. Previously, you had `rt.jar` (60+ MB) whether you used JDBC, Swing, or just `String`.

Now the JDK is ~75 modules:

```
java.base         ← Core (always present): Object, String, collections, IO
java.sql          ← JDBC
java.desktop      ← AWT, Swing
java.logging      ← java.util.logging
java.xml          ← XML processing
jdk.compiler      ← javac
jdk.jshell        ← JShell REPL
...
```

This enables:
- **Smaller runtimes**: `jlink` creates custom runtimes with only needed modules
- **Better security**: Unexported JDK internals are truly inaccessible
- **Faster startup**: Less to load

```bash
# Create minimal runtime for a command-line app
jlink --module-path $JAVA_HOME/jmods:myapp.jar \
      --add-modules com.myapp.cli \
      --output myapp-runtime

# Result: ~30MB runtime instead of ~300MB JDK
```

---

## Migration: The Unnamed Module

Not everything can be modularized at once. Java provides compatibility:

**Unnamed module**: All classpath content (non-modular jars) goes into a single "unnamed module" that:
- Can read all named modules
- Exports all packages
- Can access all opens packages

This lets you run existing code unchanged. But you lose module benefits.

**Automatic modules**: Put a regular jar on the module path, and it becomes an "automatic module":
- Module name derived from jar filename (or `Automatic-Module-Name` manifest entry)
- Exports all packages
- Requires all other modules
- Allows gradual migration

```bash
# Running with modules (module path)
java --module-path mods -m com.myapp/com.myapp.Main

# Running with classpath (unnamed module, legacy)
java -classpath libs/* com.myapp.Main

# Mixing: module path for some, classpath for others
java --module-path mods --add-modules com.myapp -classpath libs/* com.myapp.Main
```

---

## Practical Module Design

### When to Modularize

**Modularize when**:
- You're creating a library with clear public API boundaries
- You want to enforce encapsulation at scale
- You're building a custom runtime (embedded, serverless)
- You're maintaining a large codebase with unclear dependencies

**Don't modularize (yet) when**:
- Dependencies aren't modularized and you can't use automatic modules
- You're using frameworks that heavily rely on reflection (Spring, etc.—though they're improving)
- Your team is unfamiliar and you're under deadline pressure

### Module Naming Conventions

Use reverse-DNS naming, like packages:

```java
module com.company.project.component { }
```

Avoid generic names like `utils` or `common`—they conflict.

### Designing Module Boundaries

Think about:

1. **API vs Implementation**: Export API packages, hide implementation
2. **Dependency Direction**: Modules should depend downward (layers) or sideways (peers), not upward
3. **Cohesion**: A module should have a single, clear purpose

```
┌──────────────────────────────────────────────────────────────────┐
│                     Module Architecture                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│                    com.myapp.webapp                               │
│                          │                                        │
│              ┌───────────┼───────────┐                            │
│              ▼           ▼           ▼                            │
│      com.myapp.orders  com.myapp.customers  com.myapp.inventory   │
│              │           │           │                            │
│              └───────────┼───────────┘                            │
│                          ▼                                        │
│                   com.myapp.common                                │
│                          │                                        │
│                          ▼                                        │
│                      java.base                                    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### Example: A Complete Module

```java
// module-info.java
module com.myapp.orders {
    // Dependencies
    requires com.myapp.customers;
    requires com.myapp.inventory;
    requires transitive com.myapp.common;  // Our API uses common types
    requires java.sql;
    requires static lombok;  // Compile-time only

    // Public API
    exports com.myapp.orders.api;
    exports com.myapp.orders.model;
    exports com.myapp.orders.events;

    // SPI for custom order validators
    exports com.myapp.orders.spi to com.myapp.orders.premium;
    uses com.myapp.orders.spi.OrderValidator;

    // Reflection access for frameworks
    opens com.myapp.orders.model to com.fasterxml.jackson.databind;
}
```

---

## Breaking Encapsulation (When You Must)

Sometimes you need to access non-exported APIs (testing, migration, etc.):

### --add-exports (Compile and Runtime)

```bash
# Allow my.module to access internal package
javac --add-exports java.base/sun.security.ssl=my.module ...
java --add-exports java.base/sun.security.ssl=my.module ...
```

### --add-opens (Runtime Reflection)

```bash
# Allow reflection into package
java --add-opens java.base/java.lang=ALL-UNNAMED ...
```

### --add-reads (Add Dependency)

```bash
# Add a reads edge (dependency)
java --add-reads my.module=other.module ...
```

> **Warning**: These flags are escape hatches for migration, not permanent solutions. They may stop working in future Java versions as the JDK continues to encapsulate internals.

---

## Common Migration Issues

### Split Packages

A package can only exist in one module. If two jars contain the same package, that's an error:

```
Error: Package com.example.util exists in multiple modules: lib.a, lib.b
```

**Fix**: Rename packages, merge jars, or use classpath (unnamed module) for one.

### Reflective Access Warnings

```
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by com.foo.Bar to field java.lang.String.value
```

A library is using reflection to access JDK internals.

**Fixes**:
1. Update the library (many have fixed this)
2. Use `--add-opens` as a workaround
3. Accept the warning (it's not an error—yet)

### Dependency Hell

```
Module A requires B version 1.0
Module C requires B version 2.0
```

The module system doesn't solve version conflicts—it just makes them explicit. You still need build tool (Maven/Gradle) version resolution.

---

## The Module System and Build Tools

### Maven

```xml
<!-- pom.xml -->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <release>21</release>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Maven handles `module-info.java` automatically when on the source path.

### Gradle

```kotlin
// build.gradle.kts
plugins {
    java
}

java {
    modularity.inferModulePath.set(true)
}
```

Gradle infers whether to use classpath or module path based on presence of `module-info.java`.

---

## jlink: Custom Runtime Images

With modularization, you can create stripped-down runtimes:

```bash
# Create minimal runtime for a specific application
jlink \
    --module-path $JAVA_HOME/jmods:app/target/classes \
    --add-modules com.myapp.main \
    --output custom-runtime \
    --strip-debug \
    --compress=2 \
    --no-header-files \
    --no-man-pages

# Result: self-contained runtime with only needed modules
./custom-runtime/bin/java -m com.myapp.main/com.myapp.Main
```

This is powerful for:
- **Containers**: Smaller Docker images
- **Embedded**: Minimal footprint
- **Security**: Reduced attack surface

---

## What Interviewers Actually Want to Know

1. **What problem does the module system solve?** Classpath hell—lack of encapsulation, version conflicts, no dependency declarations, runtime discovery of missing classes.

2. **What's the difference between exports and opens?** `exports` gives compile-time access to public types. `opens` gives runtime reflective access to all types, including private.

3. **What's the unnamed module?** All classpath content goes into it. Provides backward compatibility—existing code works without module-info.java.

4. **What's requires transitive?** When your public API exposes types from another module, `requires transitive` ensures your consumers can access those types.

5. **How does modularization affect reflection-heavy frameworks like Spring?** They need `opens` directives for packages they inject into. Modern versions handle this better with explicit configuration.

---

## Common Misconceptions

**"The module system replaces Maven/Gradle"**

No. Build tools handle building, testing, dependency resolution. Modules handle encapsulation and runtime configuration. They complement each other.

**"I have to modularize everything"**

No. The unnamed module (classpath) still works. You can adopt incrementally or not at all.

**"Modules solve dependency version conflicts"**

No. The module system doesn't understand versions. Two versions of the same module can't coexist. Build tools still handle version resolution.

**"Automatic modules are the same as proper modules"**

No. Automatic modules export everything and read everything—they're a migration aid, not a destination.

**"public means accessible"**

Not anymore! A public class in a non-exported package is inaccessible outside the module. Strong encapsulation.

---

## Summary

| Concept | Purpose |
|---------|---------|
| `module-info.java` | Module descriptor declaring dependencies and exports |
| `requires` | Declares a dependency on another module |
| `requires transitive` | Dependency that consumers also get |
| `exports` | Makes a package accessible at compile time |
| `opens` | Allows runtime reflection into a package |
| `uses`/`provides` | Service loading declarations |
| Unnamed module | Backward compatibility for classpath content |
| Automatic modules | Regular jars on module path (migration aid) |
| `jlink` | Creates custom, minimal runtime images |

The module system represents Java's answer to large-scale software engineering: explicit dependencies, strong encapsulation, and reliable configuration. It's optional for existing code but valuable for new projects that benefit from clear boundaries.

---

*Next Chapter: [The Hidden Machinery](09-hidden-machinery.md)*
