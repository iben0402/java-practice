# Java Concurrency — Study Notes

> Goal: understand the *why* behind each mechanism, not just the API.
> That's what separates good answers from great ones in interviews.

---

## Table of Contents
1. [The Memory Model — Foundation of Everything](#1-the-memory-model)
2. [Thread Basics](#2-thread-basics)
3. [synchronized](#3-synchronized)
4. [volatile](#4-volatile)
5. [java.util.concurrent.locks](#5-locks)
6. [Atomic Classes](#6-atomic-classes)
7. [Thread-Safe Collections](#7-thread-safe-collections)
8. [ExecutorService & Thread Pools](#8-executorservice--thread-pools)
9. [CompletableFuture](#9-completablefuture)
10. [Common Concurrency Problems](#10-common-concurrency-problems)
11. [Interview Q&A Cheatsheet](#11-interview-qa-cheatsheet)

---

## 1. The Memory Model

### Why it matters
Without a memory model you can't reason about visibility or ordering. Every `synchronized`, `volatile`, and `happens-before` rule is built on top of this.

### The Problem: CPU Caches & Reordering
Each CPU core has its own L1/L2 cache. A write by Thread A to a variable may sit in A's cache and **never be visible to Thread B** running on a different core — unless you force a memory barrier.

The JVM and CPU are also allowed to **reorder instructions** for performance, as long as the result is correct *within a single thread*. This breaks multi-threaded code.

### happens-before
The JVM guarantees visibility through the **happens-before** relationship.
If action A *happens-before* action B, then A's writes are visible to B.

Key happens-before rules:
- Unlock of a monitor → lock of that same monitor
- Write to a `volatile` field → any subsequent read of that field
- `Thread.start()` → any action in the started thread
- Any action in a thread → `Thread.join()` returning in another thread
- `x.wait()` → `x.notify()` / `x.notifyAll()`

**If no happens-before exists between two threads, you have a data race.**

---

## 2. Thread Basics

### Creating threads
```java
// Option 1: extend Thread
class MyThread extends Thread {
    public void run() { System.out.println("running"); }
}
new MyThread().start();

// Option 2: implement Runnable (preferred — doesn't waste your one inheritance slot)
Thread t = new Thread(() -> System.out.println("running"));
t.start();
```

### Thread states
`NEW` → `RUNNABLE` → (`BLOCKED` | `WAITING` | `TIMED_WAITING`) → `TERMINATED`

- **BLOCKED**: waiting to acquire a monitor lock
- **WAITING**: waiting indefinitely (e.g., `Object.wait()`, `Thread.join()`)
- **TIMED_WAITING**: waiting with a timeout

### Key methods
| Method | What it does |
|--------|-------------|
| `start()` | Schedules thread for execution (do NOT call `run()` directly) |
| `join()` | Caller waits until this thread finishes |
| `sleep(ms)` | Pauses current thread; does NOT release locks |
| `interrupt()` | Sets interrupt flag; throws `InterruptedException` on blocking calls |
| `yield()` | Hints scheduler to give other threads a turn (rarely used) |

> **Interview trap**: Calling `thread.run()` instead of `thread.start()` executes on the *calling* thread — no new thread is created.

---

## 3. synchronized

### What it does
- Acquires an intrinsic (monitor) lock before entering the block/method
- Releases it on exit (even on exception)
- Guarantees **mutual exclusion** AND **visibility** (establishes happens-before)

```java
// synchronized method — locks on 'this'
public synchronized void increment() {
    count++;
}

// synchronized block — explicit lock object (preferred for fine-grained control)
private final Object lock = new Object();

public void increment() {
    synchronized (lock) {
        count++;
    }
}

// synchronized static method — locks on the Class object
public static synchronized void staticOp() { ... }
```

### Reentrant
A thread that already holds a lock can re-acquire it without deadlocking.

```java
synchronized void outer() {
    inner(); // same thread re-acquires 'this' lock — works fine
}
synchronized void inner() { ... }
```

### What synchronized does NOT do
- It does not make compound operations atomic unless **all** accesses are synchronized
- It does not protect against calling from outside the synchronization boundary

### Performance note
Uncontended `synchronized` is cheap (JVM optimizes with biased locking). It's only expensive under high contention — which is when you'd consider `ReentrantLock`.

---

## 4. volatile

### What it guarantees
1. **Visibility**: writes to a `volatile` variable are immediately visible to all threads
2. **No caching**: forces reads/writes to go to main memory
3. **Prevents certain reorderings** around the volatile access

### What it does NOT guarantee
- **Atomicity**. `volatile int count; count++` is still a read-modify-write and is **not** thread-safe.

```java
// WRONG — count++ is not atomic even with volatile
private volatile int count = 0;
public void increment() { count++; } // race condition!

// RIGHT — use AtomicInteger or synchronized
private final AtomicInteger count = new AtomicInteger(0);
public void increment() { count.incrementAndGet(); }
```

### When to use volatile
- Single-writer, multiple-reader flags:
```java
private volatile boolean running = true;

// Thread 1:
while (running) { doWork(); }

// Thread 2:
running = false; // guaranteed to be seen by Thread 1
```
- Double-checked locking (requires volatile on the instance field):
```java
private volatile Singleton instance;

public Singleton getInstance() {
    if (instance == null) {                    // first check (no lock)
        synchronized (this) {
            if (instance == null) {            // second check (with lock)
                instance = new Singleton();    // volatile prevents partial construction visibility
            }
        }
    }
    return instance;
}
```

> **Key interview distinction**: `volatile` = visibility. `synchronized` = visibility + atomicity + mutual exclusion.

---

## 5. Locks

`java.util.concurrent.locks` — more flexible than `synchronized`.

### ReentrantLock
```java
private final ReentrantLock lock = new ReentrantLock();

public void doWork() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock(); // ALWAYS in finally!
    }
}
```

### Extra capabilities over synchronized
```java
// Try to acquire without blocking
if (lock.tryLock()) {
    try { ... } finally { lock.unlock(); }
}

// Try with timeout
if (lock.tryLock(100, TimeUnit.MILLISECONDS)) { ... }

// Interruptible lock — throws InterruptedException if interrupted while waiting
lock.lockInterruptibly();

// Fairness — threads acquire in order of waiting (lower throughput, prevents starvation)
ReentrantLock fairLock = new ReentrantLock(true);
```

### ReadWriteLock
Allows multiple concurrent readers, but exclusive access for writers.
```java
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
private final Lock readLock  = rwLock.readLock();
private final Lock writeLock = rwLock.writeLock();

public String read() {
    readLock.lock();
    try { return data; } finally { readLock.unlock(); }
}

public void write(String val) {
    writeLock.lock();
    try { data = val; } finally { writeLock.unlock(); }
}
```
Use when: reads are far more frequent than writes (caches, config, lookups).

### StampedLock (Java 8+)
Even more flexible — supports **optimistic reads** (no lock acquisition at all, just validate afterward):
```java
StampedLock sl = new StampedLock();

// Optimistic read — super fast, but must validate
long stamp = sl.tryOptimisticRead();
int x = point.x;
int y = point.y;
if (!sl.validate(stamp)) {
    // someone wrote — fall back to read lock
    stamp = sl.readLock();
    try { x = point.x; y = point.y; } finally { sl.unlockRead(stamp); }
}
```

---

## 6. Atomic Classes

`java.util.concurrent.atomic` — lock-free thread safety using CPU-level **CAS** (Compare-And-Swap).

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();          // ++counter
counter.getAndIncrement();          // counter++
counter.addAndGet(5);               // counter += 5
counter.compareAndSet(5, 10);       // if counter == 5, set to 10; returns boolean

AtomicLong, AtomicBoolean, AtomicReference<V>

// AtomicReference example — atomic reference swap
AtomicReference<String> ref = new AtomicReference<>("hello");
ref.compareAndSet("hello", "world");
```

### LongAdder / LongAccumulator (Java 8+)
Better than `AtomicLong` under **high contention** — maintains multiple cells and sums them on read.
```java
LongAdder adder = new LongAdder();
adder.increment();
long total = adder.sum(); // approximate — good for stats/counters
```
Use `AtomicLong` when you need exact CAS semantics. Use `LongAdder` for high-throughput counters.

---

## 7. Thread-Safe Collections

| Collection | Thread-Safe Version | Notes |
|-----------|---------------------|-------|
| `ArrayList` | `CopyOnWriteArrayList` | Copies array on every write; great for rare writes, many reads |
| `HashMap` | `ConcurrentHashMap` | Segment/bucket-level locking; never use `Collections.synchronizedMap()` |
| `LinkedList` as queue | `ConcurrentLinkedQueue` | Lock-free |
| Blocking queue | `LinkedBlockingQueue`, `ArrayBlockingQueue` | Producer-consumer; `put()` blocks when full, `take()` blocks when empty |
| `PriorityQueue` | `PriorityBlockingQueue` | Unbounded; `take()` blocks when empty |

### ConcurrentHashMap internals (Java 8+)
- No longer uses segment locking — uses **bucket-level CAS + synchronized on bin head**
- `size()` is approximate; use `mappingCount()` for large maps
- **Atomic compound operations**:
```java
map.putIfAbsent(key, value);
map.computeIfAbsent(key, k -> expensiveCompute(k));
map.merge(key, 1, Integer::sum); // great for frequency maps
```

> **Interview trap**: `Hashtable` and `Collections.synchronizedMap()` lock the *entire* map for every operation — terrible for concurrency. Always use `ConcurrentHashMap`.

---

## 8. ExecutorService & Thread Pools

Never create raw `Thread` objects in production — use thread pools.

```java
// Fixed pool — bounded, predictable resource usage
ExecutorService pool = Executors.newFixedThreadPool(4);

// Cached pool — grows unboundedly; good for short-lived async tasks
ExecutorService pool = Executors.newCachedThreadPool();

// Single thread — sequentializes tasks
ExecutorService pool = Executors.newSingleThreadExecutor();

// Scheduled
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
scheduler.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);
```

### Submitting work
```java
// Runnable — fire and forget
pool.submit(() -> doWork());
pool.execute(() -> doWork()); // same but no Future returned

// Callable — returns a result
Future<String> future = pool.submit(() -> computeSomething());
String result = future.get();           // blocks until done
String result = future.get(2, SECONDS); // with timeout
future.cancel(true);                    // attempt to interrupt
```

### Proper shutdown
```java
pool.shutdown();                         // stop accepting new tasks, finish existing
pool.awaitTermination(10, SECONDS);      // wait for finish
if (!pool.isTerminated()) pool.shutdownNow(); // interrupt running tasks
```

### ThreadPoolExecutor — custom pools
```java
new ThreadPoolExecutor(
    2,                          // corePoolSize
    10,                         // maximumPoolSize
    60L, TimeUnit.SECONDS,      // keepAlive for idle threads above core
    new ArrayBlockingQueue<>(100), // work queue
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy: caller runs the task
);
```

Rejection policies: `AbortPolicy` (default, throws), `DiscardPolicy`, `DiscardOldestPolicy`, `CallerRunsPolicy`.

---

## 9. CompletableFuture

Java 8's async pipeline — combines Future + callback chaining.

```java
// Basic async task
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> fetchData());

// Transform result
cf.thenApply(data -> parse(data))       // sync transform
  .thenApplyAsync(parsed -> enrich(parsed)) // async transform on ForkJoinPool

// Side effect (no return value)
cf.thenAccept(result -> log(result))
  .thenRun(() -> System.out.println("done"));

// Error handling
cf.exceptionally(ex -> "default value")
  .handle((result, ex) -> ex != null ? "error" : result); // always called

// Combining
CompletableFuture<String> a = CompletableFuture.supplyAsync(() -> "A");
CompletableFuture<String> b = CompletableFuture.supplyAsync(() -> "B");

// Wait for BOTH
CompletableFuture.allOf(a, b).thenRun(() -> {
    String ra = a.join();
    String rb = b.join();
});

// First to complete wins
CompletableFuture.anyOf(a, b).thenAccept(result -> use(result));

// Chain dependent futures
CompletableFuture<String> chained = a.thenCompose(resultA ->
    CompletableFuture.supplyAsync(() -> resultA + "B")
);
```

> `thenApply` = map. `thenCompose` = flatMap. Remember this — it comes up in interviews.

---

## 10. Common Concurrency Problems

### Deadlock
Two threads each hold a lock the other needs.
```
Thread A: lock(X) → waiting for Y
Thread B: lock(Y) → waiting for X
```
**Prevention**: always acquire locks in the same order. Use `tryLock()` with timeout.

### Race Condition
Outcome depends on thread scheduling. Classic example: `count++` without synchronization.

### Livelock
Threads keep responding to each other but make no progress (like two people in a hallway both stepping aside simultaneously).

### Starvation
A thread never gets CPU time because higher-priority threads always take precedence.

### False Sharing
Two threads write to different variables that happen to be on the same CPU cache line — causes cache line bouncing and destroys performance. Fix: pad the variables or use `@Contended` (Java 8+).

### Producer-Consumer Pattern
```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);

// Producer
void produce() throws InterruptedException {
    queue.put(new Task()); // blocks if full
}

// Consumer
void consume() throws InterruptedException {
    Task t = queue.take(); // blocks if empty
    process(t);
}
```

---

## 11. Interview Q&A Cheatsheet

**Q: What's the difference between `synchronized` and `volatile`?**
`synchronized` gives mutual exclusion + visibility + atomicity for compound operations. `volatile` gives visibility only — it guarantees reads/writes are from main memory but does not prevent race conditions on compound operations like `i++`.

**Q: What's the difference between `sleep()` and `wait()`?**
`sleep()` is on `Thread`, does not release any locks, and just pauses execution. `wait()` is on `Object`, must be called within a `synchronized` block, releases the monitor lock, and requires `notify()` or `notifyAll()` to resume.

**Q: What is a happens-before relationship?**
A guarantee from the JVM that if action A happens-before action B, all changes made by A are visible to B. Key sources: monitor unlock/lock, volatile writes/reads, thread start/join.

**Q: Why is `ConcurrentHashMap` preferred over `Hashtable`?**
`Hashtable` synchronizes every method on the entire map. `ConcurrentHashMap` uses bucket-level locking (Java 8: CAS + per-bin synchronization), allowing concurrent reads and concurrent writes to different buckets.

**Q: What is the double-checked locking pattern and why does it need volatile?**
It's a lazy singleton initialization pattern. Without `volatile`, the JVM can reorder the constructor execution and the reference assignment — another thread could see a non-null but partially constructed object.

**Q: What's the difference between `Future.get()` and `CompletableFuture.join()`?**
Both block until the result is available. `get()` throws checked `InterruptedException` and `ExecutionException`. `join()` throws unchecked `CompletionException` — easier to use in lambda chains.

**Q: What is thread starvation and how do you prevent it?**
A thread never gets scheduled because others always take priority. Prevention: use fair locks (`ReentrantLock(true)`), avoid holding locks for long periods, use bounded queues to apply back-pressure.

**Q: When would you use `ReadWriteLock` over `synchronized`?**
When reads are far more frequent than writes. Multiple threads can read simultaneously without blocking each other. Use case: caches, configuration stores, lookup tables.

**Q: What's the difference between `Callable` and `Runnable`?**
`Runnable.run()` returns void and cannot throw checked exceptions. `Callable.call()` returns a value and can throw checked exceptions.

**Q: What is CAS and how do AtomicInteger use it?**
Compare-And-Swap is a CPU instruction that atomically checks if a variable has an expected value and, only if so, updates it to a new value. AtomicInteger uses a loop: read current value → compute new value → CAS(current, new) → retry if CAS fails (another thread updated it).
