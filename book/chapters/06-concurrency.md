# Chapter 6: Concurrency Without Tears

*CompletableFuture, virtual threads (Loom), structured concurrency—the journey from threads to fibers*

---

## The Fundamental Problem

Your server receives an HTTP request. To handle it, you need to:
1. Query the user service
2. Query the product catalog
3. Check inventory
4. Calculate pricing (which requires step 1 and 2)
5. Compose the response

These operations take time—network calls, database queries, file I/O. Meanwhile, a thread sits idle, waiting.

The thread-per-request model, while simple, doesn't scale. With 10,000 concurrent users, you need 10,000 threads. Each thread costs ~1MB of stack memory. That's 10GB just for thread stacks, plus context-switching overhead.

This chapter traces Java's evolution in addressing this problem: from raw threads to executors, from callbacks to CompletableFuture, and finally to virtual threads that change the game entirely.

---

## The Evolution: A Brief History

```
1996: Java 1.0        Thread, synchronized, wait/notify
2004: Java 5          ExecutorService, Future, concurrent collections
2011: Java 7          Fork/Join framework
2014: Java 8          CompletableFuture, parallel streams
2021: Java 17         Sealed classes enable state machines
2023: Java 21         Virtual threads (Project Loom)
Future                Structured concurrency (preview)
```

Each step addressed limitations of the previous approach while trying to make concurrency more accessible.

---

## CompletableFuture: Async Programming Before Virtual Threads

### The Problem with Future

Java 5's `Future<V>` represented an asynchronous computation:

```java
ExecutorService executor = Executors.newFixedThreadPool(10);

Future<String> future = executor.submit(() -> {
    // Long-running task
    return fetchFromRemoteService();
});

// Later...
String result = future.get();  // Blocks until complete!
```

`Future` had critical limitations:
1. **Blocking get()**: To use the result, you block
2. **No composition**: Can't chain futures or combine them
3. **No exception handling**: Exceptions are wrapped, hard to handle
4. **No callbacks**: Can't register completion handlers

### CompletableFuture to the Rescue

`CompletableFuture` (Java 8) is `Future` evolved into a proper async programming primitive:

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return fetchFromRemoteService();
});

// Non-blocking composition
future.thenApply(String::toUpperCase)
      .thenAccept(System.out::println);
```

### Creating CompletableFutures

```java
// From async supplier (uses ForkJoinPool.commonPool())
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> compute());

// With custom executor
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(
    () -> compute(),
    customExecutor
);

// For void operations
CompletableFuture<Void> cf3 = CompletableFuture.runAsync(() -> doWork());

// Already completed
CompletableFuture<String> cf4 = CompletableFuture.completedFuture("result");

// Completed exceptionally
CompletableFuture<String> cf5 = CompletableFuture.failedFuture(new RuntimeException());
```

### Transforming Results: thenApply, thenAccept, thenRun

```
┌──────────────────────────────────────────────────────────────────┐
│              Transformation Methods                               │
├─────────────────┬───────────────────┬────────────────────────────┤
│ Method          │ Input             │ Output                     │
├─────────────────┼───────────────────┼────────────────────────────┤
│ thenApply       │ Function<T, R>    │ CompletableFuture<R>       │
│ thenAccept      │ Consumer<T>       │ CompletableFuture<Void>    │
│ thenRun         │ Runnable          │ CompletableFuture<Void>    │
└─────────────────┴───────────────────┴────────────────────────────┘
```

```java
CompletableFuture<User> userFuture = fetchUser(userId);

// thenApply: transform the result
CompletableFuture<String> nameFuture = userFuture
    .thenApply(user -> user.getName());

// thenAccept: consume the result (returns Void)
CompletableFuture<Void> logged = nameFuture
    .thenAccept(name -> logger.info("Fetched user: {}", name));

// thenRun: run action after completion (doesn't receive result)
CompletableFuture<Void> done = logged
    .thenRun(() -> metrics.increment("user.fetch.count"));
```

### Chaining Async Operations: thenCompose

When your transformation itself returns a `CompletableFuture`, use `thenCompose` (like `flatMap`):

```java
// Wrong: nested futures
CompletableFuture<CompletableFuture<Order>> nested =
    fetchUser(userId).thenApply(user -> fetchOrders(user.getId()));

// Right: flattened
CompletableFuture<Order> orders =
    fetchUser(userId).thenCompose(user -> fetchOrders(user.getId()));
```

### Combining Independent Futures

**thenCombine**: Combine results of two independent futures:

```java
CompletableFuture<User> userFuture = fetchUser(userId);
CompletableFuture<List<Order>> ordersFuture = fetchOrders(userId);

CompletableFuture<UserWithOrders> combined = userFuture.thenCombine(
    ordersFuture,
    (user, orders) -> new UserWithOrders(user, orders)
);
```

**allOf**: Wait for all futures to complete:

```java
CompletableFuture<String> f1 = fetchFromServiceA();
CompletableFuture<String> f2 = fetchFromServiceB();
CompletableFuture<String> f3 = fetchFromServiceC();

CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2, f3);

// After all complete, gather results
all.thenAccept(v -> {
    String a = f1.join();  // Safe now - already complete
    String b = f2.join();
    String c = f3.join();
    process(a, b, c);
});
```

**anyOf**: Complete when first future completes:

```java
CompletableFuture<Object> fastest = CompletableFuture.anyOf(
    fetchFromPrimary(),
    fetchFromReplica(),
    fetchFromCache()
);
```

### Error Handling

```java
CompletableFuture<User> userFuture = fetchUser(userId);

// Handle exception, return fallback value
CompletableFuture<User> withFallback = userFuture
    .exceptionally(ex -> {
        logger.error("Failed to fetch user", ex);
        return User.ANONYMOUS;
    });

// Handle both success and failure
CompletableFuture<User> handled = userFuture
    .handle((user, ex) -> {
        if (ex != null) {
            return User.ANONYMOUS;
        }
        return user;
    });

// Run on exception but propagate it
CompletableFuture<User> logged = userFuture
    .whenComplete((user, ex) -> {
        if (ex != null) {
            logger.error("Failed", ex);
        } else {
            logger.info("Got user: {}", user);
        }
    });
```

### Timeout Handling (Java 9+)

```java
CompletableFuture<String> future = fetchFromSlowService()
    .orTimeout(5, TimeUnit.SECONDS)           // Throws TimeoutException
    .completeOnTimeout("default", 5, TimeUnit.SECONDS);  // Returns default
```

### The Executor Question

By default, `supplyAsync` uses the common Fork/Join pool. This is often wrong for I/O-bound operations:

```java
// BAD: blocking I/O in the common pool starves other tasks
CompletableFuture.supplyAsync(() -> {
    return httpClient.send(request);  // Blocks a common pool thread
});

// GOOD: use a dedicated executor for blocking operations
ExecutorService ioExecutor = Executors.newCachedThreadPool();

CompletableFuture.supplyAsync(() -> {
    return httpClient.send(request);
}, ioExecutor);
```

### Real-World Pattern: Parallel Service Calls

```java
public CompletableFuture<ProductPage> buildProductPage(String productId) {
    // Start all independent calls in parallel
    CompletableFuture<Product> productFuture = productService.get(productId);
    CompletableFuture<List<Review>> reviewsFuture = reviewService.getForProduct(productId);
    CompletableFuture<Inventory> inventoryFuture = inventoryService.check(productId);
    CompletableFuture<List<Product>> relatedFuture = recommendationService.getRelated(productId);

    // Combine when all complete
    return productFuture.thenCombine(reviewsFuture, ProductWithReviews::new)
        .thenCombine(inventoryFuture, (pwr, inv) ->
            new ProductPageData(pwr.product(), pwr.reviews(), inv))
        .thenCombine(relatedFuture, (data, related) ->
            new ProductPage(data, related));
}
```

---

## The Problems CompletableFuture Doesn't Solve

CompletableFuture is powerful, but it has costs:

1. **Callback Hell**: Complex flows become nested, hard to read
2. **Stack Traces**: When something fails, the stack trace is useless—it shows the executor thread, not your code
3. **Debugging**: You can't step through async code normally
4. **Context Propagation**: Thread-locals don't work—security context, tracing IDs get lost
5. **Structured Concurrency**: Nested tasks can outlive their parents, causing leaks

```java
// This looks simple but is hard to debug and maintain
fetchUser(id)
    .thenCompose(user -> fetchOrders(user.getId()))
    .thenCompose(orders -> enrichOrders(orders))
    .thenApply(this::calculateTotals)
    .thenCompose(totals -> saveToDatabase(totals))
    .exceptionally(this::handleError);
```

---

## Virtual Threads: The Game Changer (Java 21)

### The Insight

Traditional threads are **platform threads**—each maps 1:1 to an OS thread. OS threads are expensive: ~1MB stack, kernel scheduling overhead, limited in number (thousands, not millions).

What if we had lightweight threads that:
- Cost ~1KB instead of ~1MB
- Are scheduled by the JVM, not the OS
- Can exist in millions
- Make blocking operations cheap again

This is **Project Loom**, and virtual threads are its crown jewel.

### Creating Virtual Threads

```java
// Old way: platform thread
Thread platformThread = new Thread(() -> {
    System.out.println("Platform thread");
});

// New way: virtual thread
Thread virtualThread = Thread.startVirtualThread(() -> {
    System.out.println("Virtual thread");
});

// Or using builder
Thread vt = Thread.ofVirtual()
    .name("my-virtual-thread")
    .start(() -> doWork());
```

### Virtual Thread Executors

The real power comes from executors:

```java
// Creates a new virtual thread for each task
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

// Submit a million tasks!
for (int i = 0; i < 1_000_000; i++) {
    executor.submit(() -> {
        // Each runs in its own virtual thread
        Thread.sleep(1000);  // Blocking is FINE
        return computeResult();
    });
}
```

A million concurrent sleeping threads? With platform threads, you'd need a million MB of RAM and crash. With virtual threads, it's fine.

### The Mental Model: Continuations

Virtual threads are implemented using **continuations**. When a virtual thread blocks (I/O, sleep, lock), the JVM:

1. **Unmounts** the virtual thread from its carrier (platform) thread
2. **Parks** the continuation (saves its stack)
3. **Frees** the carrier thread for other virtual threads
4. When the blocking operation completes, **mounts** the continuation on an available carrier

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Virtual Thread Scheduling                         │
│                                                                      │
│   Carrier Thread 1         Carrier Thread 2         Carrier Thread 3│
│   ┌───────────────┐       ┌───────────────┐       ┌───────────────┐ │
│   │ Virtual       │       │ Virtual       │       │ Virtual       │ │
│   │ Thread A      │       │ Thread B      │       │ Thread C      │ │
│   │ (running)     │       │ (running)     │       │ (running)     │ │
│   └───────────────┘       └───────────────┘       └───────────────┘ │
│                                                                      │
│   Virtual Thread D (parked - waiting for I/O)                        │
│   Virtual Thread E (parked - sleeping)                               │
│   Virtual Thread F (parked - waiting for lock)                       │
│   ... millions more parked virtual threads ...                       │
│                                                                      │
│   When VT-A blocks, it unmounts. VT-D can mount on Carrier 1.       │
└─────────────────────────────────────────────────────────────────────┘
```

### Write Synchronous Code, Get Async Performance

The revolutionary insight: **you don't need to change your code**:

```java
// This synchronous-looking code scales to millions of concurrent requests
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        User user = userService.fetch(userId);      // Blocks, but cheaply
        List<Order> orders = orderService.fetch(user.getId());  // Blocks
        Inventory inv = inventoryService.check(productId);      // Blocks
        return buildResponse(user, orders, inv);
    });
}
```

No callbacks. No `thenCompose`. No reactive streams. Just straightforward sequential code that happens to scale.

### When Virtual Threads Shine

**High-concurrency I/O-bound workloads:**
- HTTP servers handling many requests
- Database connection pools
- Microservices making many outbound calls
- Chat applications with many idle connections

**Traditional server pattern, now scalable:**
```java
ServerSocket serverSocket = new ServerSocket(8080);
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    while (true) {
        Socket socket = serverSocket.accept();
        executor.submit(() -> handleConnection(socket));
    }
}
```

Each connection gets its own virtual thread. Millions of connections? No problem.

### When Virtual Threads Don't Help

**CPU-bound computation:** If you're computing, not waiting, virtual threads don't help. You're limited by CPU cores, not threads.

**Synchronized blocks with long waits:** Virtual threads can be **pinned** to their carrier when inside a `synchronized` block or calling native code. Prefer `ReentrantLock`:

```java
// BAD: pins the virtual thread
synchronized (lock) {
    blockingOperation();  // Carrier thread blocked!
}

// GOOD: virtual thread can unmount while waiting
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    blockingOperation();  // Virtual thread unmounts, carrier freed
} finally {
    lock.unlock();
}
```

**Thread-local heavy code:** Virtual threads are cheap, so you might create millions. Thread-locals that allocate memory per thread can cause problems.

---

## Structured Concurrency (Preview)

### The Problem with Unstructured Concurrency

```java
void fetchUserData() throws Exception {
    ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

    Future<User> userFuture = executor.submit(() -> fetchUser());
    Future<Orders> ordersFuture = executor.submit(() -> fetchOrders());

    try {
        User user = userFuture.get();
        Orders orders = ordersFuture.get();
        process(user, orders);
    } catch (Exception e) {
        // If fetchUser fails, fetchOrders continues running!
        // Resource leak, wasted computation
        throw e;
    }
}
```

Tasks are **orphaned**—they outlive their logical scope. If one fails, others keep running. Cancellation is manual and error-prone.

### Structured Concurrency to the Rescue

The principle: **when a task starts subtasks, they should all complete (success or failure) before the task completes**.

```java
void fetchUserData() throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<User> userTask = scope.fork(() -> fetchUser());
        Subtask<Orders> ordersTask = scope.fork(() -> fetchOrders());

        scope.join();            // Wait for all tasks
        scope.throwIfFailed();   // Propagate any exception

        // Both succeeded
        process(userTask.get(), ordersTask.get());
    }
    // Scope closes: all tasks guaranteed complete
}
```

If `fetchUser()` fails:
1. The exception is captured
2. `fetchOrders()` is cancelled (interrupted)
3. `throwIfFailed()` rethrows the exception
4. No orphaned tasks

### Scopes for Different Strategies

**ShutdownOnFailure**: Cancel remaining tasks on first failure
```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    // First failure cancels others and is rethrown
}
```

**ShutdownOnSuccess**: Return first successful result, cancel others
```java
try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {
    scope.fork(() -> fetchFromPrimary());
    scope.fork(() -> fetchFromReplica());
    scope.fork(() -> fetchFromCache());

    scope.join();
    String result = scope.result();  // First to succeed
}
```

### Benefits of Structure

1. **Error propagation**: Child failures propagate to parent
2. **Cancellation**: Cancelled parent cancels children
3. **Observability**: Stack traces show the logical hierarchy
4. **No leaks**: All tasks complete before scope closes
5. **Thread-dump clarity**: Virtual threads show their logical owner

---

## Concurrent Collections: The Foundation

Regardless of threading model, you need thread-safe data structures.

### ConcurrentHashMap

```java
ConcurrentHashMap<String, Integer> counts = new ConcurrentHashMap<>();

// Atomic get-or-default
counts.getOrDefault("key", 0);

// Atomic put-if-absent
counts.putIfAbsent("key", 1);

// Atomic compute
counts.compute("key", (k, v) -> v == null ? 1 : v + 1);

// Atomic merge
counts.merge("key", 1, Integer::sum);

// Parallel forEach
counts.forEach(4, (key, value) -> process(key, value));
```

### CopyOnWriteArrayList

```java
// For read-heavy, write-rare scenarios
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

// Iteration never throws ConcurrentModificationException
// Because iterators see a snapshot
for (String item : list) {
    // Safe even if another thread modifies
}
```

### BlockingQueue Variants

```java
// Bounded buffer
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100);
queue.put(task);    // Blocks if full
Task t = queue.take();  // Blocks if empty

// Unbounded
BlockingQueue<Task> unbounded = new LinkedBlockingQueue<>();

// Priority-based
BlockingQueue<Task> priority = new PriorityBlockingQueue<>();

// Delayed execution
DelayQueue<DelayedTask> delayed = new DelayQueue<>();
```

---

## Synchronization Primitives

### ReentrantLock vs synchronized

```java
// synchronized: simple but limited
synchronized (object) {
    // critical section
}

// ReentrantLock: more control
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();
}

// Advantages of ReentrantLock:
// - tryLock() for non-blocking attempt
// - lockInterruptibly() for interruptible waiting
// - Fair mode (FIFO ordering)
// - Multiple condition variables
// - Doesn't pin virtual threads!
```

### ReadWriteLock

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();

// Multiple readers allowed simultaneously
rwLock.readLock().lock();
try {
    return readData();
} finally {
    rwLock.readLock().unlock();
}

// Writers get exclusive access
rwLock.writeLock().lock();
try {
    writeData();
} finally {
    rwLock.writeLock().unlock();
}
```

### StampedLock (Java 8)

```java
StampedLock lock = new StampedLock();

// Optimistic read (no blocking!)
long stamp = lock.tryOptimisticRead();
int x = this.x;
int y = this.y;
if (!lock.validate(stamp)) {
    // Another thread modified, fall back to regular read
    stamp = lock.readLock();
    try {
        x = this.x;
        y = this.y;
    } finally {
        lock.unlockRead(stamp);
    }
}
return new Point(x, y);
```

### Semaphore

```java
// Limit concurrent access
Semaphore permits = new Semaphore(10);  // 10 concurrent allowed

permits.acquire();  // Wait for permit
try {
    accessLimitedResource();
} finally {
    permits.release();
}
```

### CountDownLatch

```java
// Wait for N events
CountDownLatch latch = new CountDownLatch(3);

// Workers count down
executor.submit(() -> { doWork1(); latch.countDown(); });
executor.submit(() -> { doWork2(); latch.countDown(); });
executor.submit(() -> { doWork3(); latch.countDown(); });

// Main thread waits
latch.await();  // Blocks until count reaches 0
proceed();
```

### CyclicBarrier

```java
// Reusable synchronization point
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All threads reached barrier");
});

// Each thread:
doPhase1Work();
barrier.await();  // Wait for others
doPhase2Work();
barrier.await();  // Reusable!
doPhase3Work();
```

---

## Choosing Your Concurrency Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Concurrency Decision Tree                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Is your workload CPU-bound?                                         │
│  ├── YES: Use parallel streams or ForkJoinPool                       │
│  │        (Limited by cores, not threads)                            │
│  │                                                                   │
│  └── NO (I/O-bound): How many concurrent operations?                 │
│       ├── Hundreds: Platform threads with ExecutorService fine       │
│       │                                                              │
│       └── Thousands+: Use virtual threads (Java 21+)                 │
│            │                                                         │
│            └── Need to compose results?                              │
│                 ├── Simple: Sequential code in virtual threads       │
│                 └── Complex: StructuredTaskScope for managed tasks   │
│                                                                      │
│  Still on Java 8-20? CompletableFuture for async composition         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## What Interviewers Actually Want to Know

1. **What's the difference between CompletableFuture and Future?** CompletableFuture supports non-blocking composition (thenApply, thenCompose), callbacks, and exception handling. Future only has blocking get().

2. **Explain thenApply vs thenCompose.** thenApply transforms the result (Function<T,R>). thenCompose chains async operations (Function<T, CompletableFuture<R>>). Like map vs flatMap.

3. **What are virtual threads?** Lightweight threads managed by the JVM, not the OS. They allow millions of concurrent threads by efficiently unmounting when blocked.

4. **When would you NOT use virtual threads?** CPU-bound work (no benefit), synchronized blocks (pins carrier), heavy thread-local usage.

5. **What is structured concurrency?** Ensuring subtasks complete before their parent, enabling proper error propagation and cancellation. Prevents orphaned tasks.

---

## Common Misconceptions

**"Virtual threads are faster than platform threads"**

Not individually. A single virtual thread isn't faster than a platform thread. The benefit is *scalability*—you can have millions, enabling high concurrency.

**"CompletableFuture is non-blocking"**

The *composition* is non-blocking. The underlying tasks still block if you use blocking I/O. You're just not blocking the calling thread.

**"I need to change my code for virtual threads"**

Usually not! Existing thread-based code works. The main changes: prefer ReentrantLock over synchronized for better unmounting, and be careful with thread-locals.

**"More threads = more speed"**

Only for I/O-bound work. For CPU-bound work, threads beyond core count add overhead. Parallel streams auto-size to available processors.

**"ConcurrentHashMap is always the right choice"**

For write-heavy workloads. For read-heavy with rare writes, CopyOnWriteArrayList or immutable maps might be better.

---

## Summary

| Feature | Java Version | Use Case |
|---------|--------------|----------|
| ExecutorService | 5 | Managing thread pools |
| CompletableFuture | 8 | Async composition without virtual threads |
| Parallel Streams | 8 | CPU-bound parallel processing |
| Virtual Threads | 21 | High-concurrency I/O-bound workloads |
| Structured Concurrency | 21+ (preview) | Managing task hierarchies safely |

The evolution continues toward making concurrency more accessible:
- **Java 5**: "Here are building blocks" (primitives)
- **Java 8**: "Here's composition" (CompletableFuture)
- **Java 21**: "Here's simplicity at scale" (virtual threads)
- **Future**: "Here's safety" (structured concurrency)

Write straightforward code. Let the runtime handle the complexity.

---

*Next Chapter: [Pattern Matching and Algebraic Thinking](07-pattern-matching.md)*
