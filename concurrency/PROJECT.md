# Mini Project — Thread-Safe In-Memory Cache with TTL & Stats

## Overview

Build a production-grade in-memory cache from scratch.
This project deliberately touches every major concurrency concept from the notes.

**Estimated time**: 3–5 hours  
**Difficulty**: Ambitious but achievable in one session

---

## Requirements

### Core API
```java
public interface Cache<K, V> {
    void put(K key, V value);
    void put(K key, V value, long ttlMillis);   // with expiry
    Optional<V> get(K key);
    void evict(K key);
    void clear();
    CacheStats stats();
}
```

### CacheStats
```java
public record CacheStats(
    long hits,
    long misses,
    long evictions,
    long size
) {}
```

---

## Features to Implement (in order)

### Phase 1 — Basic thread-safe cache
- Back the cache with `ConcurrentHashMap`
- `put` / `get` / `evict` / `clear`
- Track hit/miss counts with `LongAdder` (not `AtomicLong` — think about why)

### Phase 2 — TTL (Time-To-Live)
Each entry should carry an expiry timestamp:
```java
private record CacheEntry<V>(V value, long expiresAt) {
    boolean isExpired() {
        return expiresAt > 0 && System.currentTimeMillis() > expiresAt;
    }
}
```
`get()` must check expiry and treat expired entries as misses (lazy eviction).

### Phase 3 — Background eviction
Expired entries sitting in the map waste memory.
Add a background thread (scheduled via `ScheduledExecutorService`) that sweeps the map every N seconds and removes expired entries.

```java
scheduler.scheduleAtFixedRate(this::evictExpired, 5, 5, TimeUnit.SECONDS);
```

**Concurrency challenge**: the sweep runs concurrently with reads and writes.
How do you safely iterate and remove from a `ConcurrentHashMap`?  
Hint: `map.entrySet().removeIf(e -> e.getValue().isExpired())` is safe on CHM.

### Phase 4 — Max size + LRU eviction
Add a `maxSize` limit. When the cache is full, evict the **Least Recently Used** entry.

Options:
1. Use `LinkedHashMap(capacity, 0.75f, true)` (access-order) wrapped in a `ReadWriteLock`
2. Maintain a `ConcurrentLinkedDeque` as an access-order tracker alongside the `ConcurrentHashMap`

Option 1 is simpler. Option 2 is more interesting concurrency-wise — pick based on how much time you have.

### Phase 5 — Loader function (cache-aside)
```java
V getOrLoad(K key, Function<K, V> loader);
V getOrLoad(K key, long ttlMillis, Function<K, V> loader);
```
The loader should be called **at most once per key** even under concurrent access.
This is the classic "thundering herd" problem.

**How to solve it**: Use `ConcurrentHashMap.computeIfAbsent()` — it is atomic per key.
But be aware: in Java 8, calling `computeIfAbsent` inside another `computeIfAbsent` on the same map can deadlock. In Java 9+ this is fixed.

### Phase 6 — Graceful shutdown
Implement `AutoCloseable` / `Closeable`:
```java
@Override
public void close() {
    scheduler.shutdown();
    try {
        scheduler.awaitTermination(5, TimeUnit.SECONDS);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
}
```
Always restore the interrupt flag when catching `InterruptedException`.

---

## Class Structure (suggested)

```
src/
  project/
    Cache.java               (interface)
    CacheStats.java          (record)
    ConcurrentLRUCache.java  (main implementation)
    CacheEntry.java          (private record / inner class)
    CacheTest.java           (manual tests / demo main)
```

---

## Testing Scenarios

Write a `main` method (or JUnit tests) that validates:

```java
// 1. Basic put/get
cache.put("user:1", new User("Alice"));
assert cache.get("user:1").isPresent();

// 2. TTL expiry
cache.put("session:abc", token, 200); // 200ms TTL
Thread.sleep(300);
assert cache.get("session:abc").isEmpty(); // should be expired

// 3. Concurrent writes — no data corruption
ExecutorService pool = Executors.newFixedThreadPool(20);
for (int i = 0; i < 1000; i++) {
    final int n = i;
    pool.submit(() -> cache.put("key:" + (n % 50), "value:" + n));
}
pool.shutdown();
pool.awaitTermination(10, SECONDS);
// stats.size() should be <= 50

// 4. Stats accuracy
CacheStats stats = cache.stats();
System.out.printf("Hits: %d, Misses: %d, Evictions: %d%n",
    stats.hits(), stats.misses(), stats.evictions());

// 5. Loader — called once per key under concurrency
AtomicInteger loaderCallCount = new AtomicInteger(0);
ExecutorService pool2 = Executors.newFixedThreadPool(10);
for (int i = 0; i < 10; i++) {
    pool2.submit(() -> cache.getOrLoad("shared-key", k -> {
        loaderCallCount.incrementAndGet();
        return expensiveCompute(k);
    }));
}
pool2.shutdown();
pool2.awaitTermination(5, SECONDS);
assert loaderCallCount.get() == 1; // loader called exactly once
```

---

## Stretch Goals (if you finish early)

- **Metrics endpoint**: expose stats as a JSON string via a tiny HTTP server (`com.sun.net.httpserver`)
- **Segmented locking**: partition the cache into N segments, each with its own `ReadWriteLock`, to reduce contention (mimic how `ConcurrentHashMap` works internally)
- **Weak references**: use `WeakReference<V>` for values so the GC can reclaim memory under pressure — handle the case where `get()` returns a null referent
- **Listeners**: add an `EvictionListener<K, V>` that fires a callback when an entry is evicted (expired, over-capacity, or manual)

---

## Concepts This Project Covers

| Concept | Where |
|---------|-------|
| `ConcurrentHashMap` + compound operations | Phase 1, 5 |
| `LongAdder` for high-throughput counters | Phase 1 |
| Lazy expiry / timestamp comparison | Phase 2 |
| `ScheduledExecutorService` background thread | Phase 3 |
| Safe concurrent iteration | Phase 3 |
| LRU with access-order `LinkedHashMap` + `ReadWriteLock` | Phase 4 |
| `computeIfAbsent` thundering herd prevention | Phase 5 |
| `InterruptedException` handling + thread interrupt flag | Phase 6 |
| `AutoCloseable` + executor shutdown | Phase 6 |

---

## Portfolio Note

This project is portfolio-worthy. When talking about it in interviews, lead with the problems you solved:

> "I built a concurrent cache that handles TTL expiry, LRU eviction, and prevents the thundering herd problem. I used ConcurrentHashMap.computeIfAbsent to ensure the loader function is called at most once per key under concurrent access, and LongAdder instead of AtomicLong for hit/miss counters because it reduces contention under high write load."

That's the kind of answer that demonstrates you understand *why*, not just *how*.
