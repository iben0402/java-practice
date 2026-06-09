# Java Concurrency — Practice Exercises

Work through these in order. Each section builds on the previous one.
Write your solutions in `src/exercises/` within the concurrency module.

---

## Level 1 — Thread Basics

### Exercise 1.1 — Thread vs Runnable
Create a counter that counts from 1 to 5, using both `Thread` subclass and `Runnable`.
Then call `.run()` instead of `.start()` on one of them and observe the difference.

**What to notice**: Which thread ID prints? Why?

---

### Exercise 1.2 — Race Condition (intentional)
```java
public class Counter {
    private int count = 0;
    public void increment() { count++; }
    public int get() { return count; }
}
```
Spawn 10 threads, each calling `increment()` 1000 times.
Print the final count. Run it several times.

**Questions to answer**:
- Is the result always 10000? Why not?
- What does `count++` actually compile to at the bytecode level? (Hint: it's 3 operations)

---

### Exercise 1.3 — Fix with synchronized
Fix Exercise 1.2 using:
1. A `synchronized` method
2. A `synchronized` block with a dedicated lock object

Verify the result is always 10000.

---

## Level 2 — volatile vs synchronized

### Exercise 2.1 — Visibility bug
```java
public class VisibilityBug {
    private static boolean running = true; // NOT volatile

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            while (running) {} // spin
            System.out.println("Stopped");
        }).start();

        Thread.sleep(100);
        running = false;
        System.out.println("Flag set to false");
    }
}
```
Run this. Does "Stopped" always print? (It may loop forever on some JVMs/hardware.)
Now add `volatile` to `running`. Does it always stop now?

**Explain why** in a comment in your code.

---

### Exercise 2.2 — volatile is NOT enough
Prove that `volatile` doesn't make `count++` atomic:
- Use a `volatile int count`
- Spawn 10 threads, each incrementing 1000 times
- Show the result is still inconsistent
- Fix it with `AtomicInteger`

---

### Exercise 2.3 — Double-Checked Locking
Implement a thread-safe lazy singleton using double-checked locking.
Then implement it using the **initialization-on-demand holder** pattern and explain why the holder pattern is simpler and equally safe.

```java
// Holder pattern skeleton:
public class Singleton {
    private Singleton() {}
    
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## Level 3 — Locks

### Exercise 3.1 — ReentrantLock basics
Rewrite the counter from Exercise 1.3 using `ReentrantLock`.
Then add a `tryLock(50, MILLISECONDS)` version that skips the increment if the lock isn't available within 50ms (and logs a message).

---

### Exercise 3.2 — Deadlock
Create a deliberate deadlock:
```java
Object lockA = new Object();
Object lockB = new Object();

// Thread 1: acquires A, then tries B
// Thread 2: acquires B, then tries A
```
Confirm it deadlocks (program hangs).
Then fix it by ensuring both threads acquire locks in the same order.

---

### Exercise 3.3 — ReadWriteLock cache
Implement a simple `ThreadSafeCache<K, V>` backed by a `HashMap` using `ReentrantReadWriteLock`.

Requirements:
- `get(K key)` — uses read lock
- `put(K key, V value)` — uses write lock
- `getOrCompute(K key, Function<K, V> loader)` — uses read lock first, upgrades to write lock only on miss

**Gotcha**: Java's `ReentrantReadWriteLock` does NOT support lock upgrading (read → write).
How do you handle the getOrCompute case correctly?

---

## Level 4 — ExecutorService

### Exercise 4.1 — Basic thread pool
Submit 20 tasks to a fixed pool of 4 threads.
Each task: sleep random 100–500ms, print `"Task N done by " + Thread.currentThread().getName()`.
Observe that only 4 thread names appear.

---

### Exercise 4.2 — Collect results with Future
Submit 5 `Callable<Integer>` tasks that each return a random number.
Collect all `Future<Integer>` results and print their sum.
Add a 2-second timeout to each `future.get()` call.

---

### Exercise 4.3 — invokeAll vs invokeAny
Using a list of `Callable<String>` tasks (some fast, some slow):
- `invokeAll()`: wait for all, print all results
- `invokeAny()`: print only the first result, cancel the rest

When would you use each in a real application?

---

### Exercise 4.4 — Rejection policy
Create a `ThreadPoolExecutor` with:
- 2 core threads
- 4 max threads
- Queue capacity of 2
- `CallerRunsPolicy`

Submit 10 tasks simultaneously and observe which thread runs the rejected ones.
Try each of the 4 rejection policies and note the behavior.

---

## Level 5 — CompletableFuture

### Exercise 5.1 — Basic pipeline
Simulate fetching a user, then their orders, then computing a total:
```
fetchUser(id) → fetchOrders(user) → computeTotal(orders)
```
Each step is async (use `supplyAsync` with a sleep to simulate I/O).
Chain them using `thenCompose`. Print the total.

---

### Exercise 5.2 — Parallel fetch + combine
Fetch 3 items in parallel (simulate with `supplyAsync`), wait for all 3, combine results:
```java
CompletableFuture<String> itemA = CompletableFuture.supplyAsync(() -> fetchA());
CompletableFuture<String> itemB = CompletableFuture.supplyAsync(() -> fetchB());
CompletableFuture<String> itemC = CompletableFuture.supplyAsync(() -> fetchC());

// Combine all three into one result
```

---

### Exercise 5.3 — Error handling
Build a pipeline that:
1. Fetches data (may throw an exception 50% of the time — use `Random`)
2. On success: transform and return the result
3. On failure: return a fallback value using `exceptionally`
4. Always log whether the call succeeded or failed using `whenComplete`

---

### Exercise 5.4 — Timeout
Java 9+ added `orTimeout` and `completeOnTimeout`. Use them:
```java
CompletableFuture.supplyAsync(() -> {
    Thread.sleep(3000); // slow operation
    return "result";
})
.orTimeout(1, TimeUnit.SECONDS)
.exceptionally(ex -> "timed out!");
```
What exception is thrown on timeout?

---

## Level 6 — Producer-Consumer

### Exercise 6.1 — BlockingQueue pipeline
Build a simple pipeline with `LinkedBlockingQueue`:
- **Producer**: generates integers 1–20, puts them in the queue
- **Consumer**: takes from queue, squares each number, prints it
- Use `null` or a sentinel value to signal "done"
- Run producer in 1 thread, consumer in 2 threads

---

### Exercise 6.2 — Bounded buffer with backpressure
Same as above but use `ArrayBlockingQueue(5)` — capacity of 5.
Make the producer sleep 0ms and the consumer sleep 100ms.
Observe that the producer is slowed down by `put()` blocking.
This is **backpressure** — a fundamental pattern in streaming systems.

---

## Level 7 — Synchronizers

### Exercise 7.1 — CountDownLatch: race start
Simulate a race: 5 runner threads wait at a start line, a starter thread counts down from 1.
All runners start simultaneously.
```java
CountDownLatch startSignal = new CountDownLatch(1);
// runners call startSignal.await()
// starter calls startSignal.countDown()
```

### Exercise 7.2 — CyclicBarrier: phase synchronization
Simulate 3 threads that must all finish phase 1 before any can start phase 2.
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("Phase complete!"));
// each thread calls barrier.await() at the end of each phase
```
Note: `CyclicBarrier` resets and can be reused. `CountDownLatch` cannot.

### Exercise 7.3 — Semaphore: connection pool
Simulate a database connection pool with max 3 connections using `Semaphore(3)`.
Spawn 8 threads that each try to "acquire a connection", do work (sleep), then release.
Verify no more than 3 threads are in the critical section at once.

---

## Challenge Exercises

### Challenge A — Thread-safe frequency map
Build a thread-safe word frequency counter.
Given a `List<String>` of words, count occurrences using `ConcurrentHashMap.merge()`.
Compare performance vs a `synchronized HashMap` under high concurrency (many threads).

### Challenge B — Custom thread pool with metrics
Extend `ThreadPoolExecutor` to track:
- Total tasks submitted
- Total tasks completed
- Average task execution time
Override `beforeExecute` and `afterExecute` to collect the metrics.

### Challenge C — Parallel merge sort
Implement parallel merge sort using `RecursiveTask<int[]>` and `ForkJoinPool`.
Split the array recursively; below a threshold (e.g., 1000 elements) use sequential sort.
Compare timing vs `Arrays.sort()`.

---

## Self-Check Questions

After each level, ask yourself:

1. Could I explain this to someone without using the code?
2. What happens if I remove the synchronization? (race condition? visibility bug? deadlock?)
3. What's the performance cost of my solution?
4. Is there a more idiomatic Java way to do this?
