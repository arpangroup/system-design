# Build LRU & LFU Cache From Scratch in Java

A step-by-step guide to designing and implementing a production-oriented in-memory cache from scratch.

We will start with a simple cache and progressively evolve it into a configurable caching framework supporting:

- Basic `get()` / `put()`
- Maximum capacity
- LRU eviction
- LFU eviction
- Pluggable eviction policies
- Thread safety
- TTL / expiration
- Cache statistics
- Generic key/value types
- Atomic operations
- Cache loaders
- Cache-aside pattern
- Per-cache configuration
- MiniSpring dependency injection
- `@Cacheable`
- `@CacheEvict`
- `@CachePut`
- Multiple named caches
- Future distributed-cache extensions

---

# 1. What Are We Building?

We want an API similar to:

```java
Cache<String, User> cache = new LruCache<>(100);

cache.put("user:1", user);

User user = cache.get("user:1");

cache.remove("user:1");

cache.clear();
```

The cache should automatically remove entries when its capacity is reached.

For example:

```text
Capacity = 3

put(A)
put(B)
put(C)

Cache:

A B C

put(D)

One entry must be evicted.
```

For LRU:

```text
A = least recently used
B
C = most recently used

put(D)

Evict A

B
C
D
```

For LFU:

```text
A -> frequency 1
B -> frequency 5
C -> frequency 2

put(D)

Evict A
```

---

# 2. Learning Objectives

By completing this project you should understand:

```text
HashMap
   ↓
Linked List
   ↓
Doubly Linked List
   ↓
LRU
   ↓
Frequency Tracking
   ↓
LFU
   ↓
Eviction Strategy
   ↓
Thread Safety
   ↓
Locks
   ↓
TTL
   ↓
Statistics
   ↓
Cache Loader
   ↓
SOLID
   ↓
MiniSpring
   ↓
@Cacheable
```

This is an excellent project for understanding how frameworks such as Spring caching and libraries such as Caffeine/Guava approach caching.

---

# 3. Project Structure

Start with:

```text
mini-cache/
│
├── src/main/java/
│   └── com/example/cache/
│
│       ├── api/
│       │   ├── Cache.java
│       │   ├── CacheStats.java
│       │   └── CacheLoader.java
│       │
│       ├── core/
│       │   ├── LruCache.java
│       │   ├── LfuCache.java
│       │   └── AbstractCache.java
│       │
│       ├── eviction/
│       │   ├── EvictionPolicy.java
│       │   ├── LruEvictionPolicy.java
│       │   └── LfuEvictionPolicy.java
│       │
│       ├── concurrency/
│       │   └── CacheLock.java
│       │
│       ├── config/
│       │   └── CacheConfig.java
│       │
│       ├── annotation/
│       │   ├── Cacheable.java
│       │   ├── CacheEvict.java
│       │   └── CachePut.java
│       │
│       └── minispring/
│           ├── CacheAutoConfiguration.java
│           └── CacheProxy.java
│
└── src/test/java/
    └── com/example/cache/
        ├── LruCacheTest.java
        ├── LfuCacheTest.java
        ├── ConcurrentCacheTest.java
        └── CacheIntegrationTest.java
```

Do not create everything at once.

Build it incrementally.

---

# 4. Step 1 — Define the Cache API

Create:

```java
public interface Cache<K, V> {

    V get(K key);

    void put(K key, V value);

    V remove(K key);

    void clear();

    int size();

    boolean containsKey(K key);
}
```

Basic usage:

```java
Cache<String, String> cache =
        new LruCache<>(3);

cache.put("A", "Apple");
cache.put("B", "Banana");

System.out.println(cache.get("A"));
```

---

# 5. Step 2 — Implement a Simple Cache

Before implementing eviction, create a basic cache.

```java
public class SimpleCache<K, V> implements Cache<K, V> {

    private final Map<K, V> store =
            new HashMap<>();

    @Override
    public V get(K key) {
        return store.get(key);
    }

    @Override
    public void put(K key, V value) {
        store.put(key, value);
    }

    @Override
    public V remove(K key) {
        return store.remove(key);
    }

    @Override
    public void clear() {
        store.clear();
    }

    @Override
    public int size() {
        return store.size();
    }

    @Override
    public boolean containsKey(K key) {
        return store.containsKey(key);
    }
}
```

This gives us:

```text
Cache API
   ↓
HashMap
```

But it has no maximum capacity.

---

# 6. Why Do We Need Eviction?

Suppose:

```java
Cache<String, User> cache =
        new SimpleCache<>();
```

and the application processes millions of users.

The cache can grow indefinitely:

```text
1 entry
10 entries
1000 entries
1 million entries
10 million entries
...
```

Eventually:

```text
Heap Memory
    ↓
GC pressure
    ↓
OutOfMemoryError
```

Therefore:

```text
Maximum Capacity
        +
Eviction Policy
        =
Bounded Cache
```

---

# 7. Step 3 — Understand LRU

LRU means:

> Least Recently Used.

Suppose:

```text
Capacity = 3
```

Operations:

```text
put(A)
put(B)
put(C)
```

State:

```text
A B C
```

Then:

```java
get(A);
```

A becomes recently used.

Conceptually:

```text
B C A
```

Then:

```java
put(D);
```

Evict:

```text
B
```

Result:

```text
C A D
```

---

# 8. LRU Data Structure

A naive implementation might scan all entries:

```text
HashMap
+
timestamp
```

Every `get()` updates timestamp.

Eviction requires:

```text
Find minimum timestamp
```

Complexity:

```text
get()      O(1)
put()      O(n)
eviction   O(n)
```

We can do better.

Use:

```text
HashMap + Doubly Linked List
```

This gives:

```text
get()       O(1)
put()       O(1)
remove()    O(1)
eviction    O(1)
```

---

# 9. LRU Architecture

Use:

```text
             HashMap
          ┌───────────┐
          │ A → Node  │
          │ B → Node  │
          │ C → Node  │
          └─────┬─────┘
                │
                ▼
HEAD ⇄ A ⇄ B ⇄ C ⇄ TAIL
```

Convention:

```text
HEAD = least recently used
TAIL = most recently used
```

---

# 10. Step 4 — Create Node

```java
class Node<K, V> {

    K key;
    V value;

    Node<K, V> prev;
    Node<K, V> next;

    Node(K key, V value) {
        this.key = key;
        this.value = value;
    }
}
```

---

# 11. Step 5 — Implement Doubly Linked List

Create:

```java
class DoublyLinkedList<K, V> {

    private final Node<K, V> head;
    private final Node<K, V> tail;

    DoublyLinkedList() {
        head = new Node<>(null, null);
        tail = new Node<>(null, null);

        head.next = tail;
        tail.prev = head;
    }
}
```

The dummy nodes simplify insertion/removal.

---

# 12. Add Node to Tail

```java
void addLast(Node<K, V> node) {

    Node<K, V> previous =
            tail.prev;

    previous.next = node;
    node.prev = previous;

    node.next = tail;
    tail.prev = node;
}
```

---

# 13. Remove Node

```java
void remove(Node<K, V> node) {

    node.prev.next = node.next;
    node.next.prev = node.prev;

    node.prev = null;
    node.next = null;
}
```

---

# 14. Remove First Node

```java
Node<K, V> removeFirst() {

    Node<K, V> first =
            head.next;

    if (first == tail) {
        return null;
    }

    remove(first);

    return first;
}
```

---

# 15. Move Node to Tail

```java
void moveToLast(Node<K, V> node) {

    remove(node);
    addLast(node);
}
```

Now we have everything required for LRU.

---

# 16. Step 6 — Implement LRU Cache

```java
public class LruCache<K, V>
        implements Cache<K, V> {

    private final int capacity;

    private final Map<K, Node<K, V>> map =
            new HashMap<>();

    private final DoublyLinkedList<K, V> list =
            new DoublyLinkedList<>();

    public LruCache(int capacity) {

        if (capacity <= 0) {
            throw new IllegalArgumentException(
                    "Capacity must be greater than zero");
        }

        this.capacity = capacity;
    }

    @Override
    public V get(K key) {

        Node<K, V> node = map.get(key);

        if (node == null) {
            return null;
        }

        list.moveToLast(node);

        return node.value;
    }
}
```

---

# 17. Implement put()

```java
@Override
public void put(K key, V value) {

    Node<K, V> existing =
            map.get(key);

    if (existing != null) {

        existing.value = value;

        list.moveToLast(existing);

        return;
    }

    Node<K, V> node =
            new Node<>(key, value);

    map.put(key, node);

    list.addLast(node);

    if (map.size() > capacity) {

        Node<K, V> evicted =
                list.removeFirst();

        map.remove(evicted.key);
    }
}
```

---

# 18. Complexity of LRU

| Operation | Complexity |
|---|---:|
| get | O(1) |
| put | O(1) |
| remove | O(1) |
| eviction | O(1) |
| containsKey | O(1) |

Memory:

```text
O(capacity)
```

---

# 19. Step 7 — Test LRU

```java
@Test
void shouldEvictLeastRecentlyUsed() {

    Cache<String, Integer> cache =
            new LruCache<>(3);

    cache.put("A", 1);
    cache.put("B", 2);
    cache.put("C", 3);

    cache.get("A");

    cache.put("D", 4);

    assertNotNull(cache.get("A"));
    assertNull(cache.get("B"));
    assertNotNull(cache.get("C"));
    assertNotNull(cache.get("D"));
}
```

---

# 20. Important LRU Edge Cases

Test:

### Updating an existing key

```text
put(A, 1)
put(B, 2)
put(A, 10)
```

A must become recently used.

### Capacity = 1

```text
put(A)
put(B)
```

Only B remains.

### Same value

```text
put(A, 1)
put(A, 1)
```

Size should remain 1.

### Null keys

Decide whether you support:

```java
cache.put(null, value);
```

A production cache should explicitly document this behavior.

---

# 21. Alternative: LinkedHashMap

Java already provides a convenient implementation.

```java
Map<K, V> cache =
        new LinkedHashMap<>(
                16,
                0.75f,
                true
        );
```

The third argument:

```text
accessOrder = true
```

maintains access order.

You can override:

```java
protected boolean removeEldestEntry(
        Map.Entry<K,V> eldest) {

    return size() > capacity;
}
```

However, for this learning project, implement the data structure yourself first.

---

# 22. Step 8 — Understand LFU

LFU means:

> Least Frequently Used.

Instead of tracking recency, track access frequency.

Example:

```text
A → 10 accesses
B → 2 accesses
C → 5 accesses
```

If capacity is reached:

```text
B
```

is evicted.

---

# 23. LFU Problem

Suppose:

```text
A → frequency 5
B → frequency 5
C → frequency 5
```

Which one should be removed?

We need a tie-breaking rule.

Common solution:

```text
LFU
 ↓
frequency
 ↓
if frequency same
 ↓
LRU
```

So LFU can become:

> Least Frequently Used, with Least Recently Used as the tie breaker.

---

# 24. LFU Data Structure

A typical efficient implementation uses:

```text
HashMap<Key, Node>

+

HashMap<Frequency, DoublyLinkedList<Node>>

+

minimumFrequency
```

Architecture:

```text
keyMap

A → Node(freq=2)
B → Node(freq=1)
C → Node(freq=2)
```

Frequency buckets:

```text
freq 1 → B

freq 2 → A ⇄ C

freq 3 → ...
```

And:

```text
minFrequency = 1
```

---

# 25. LFU Node

```java
class LfuNode<K, V> {

    K key;
    V value;

    int frequency = 1;

    LfuNode<K, V> prev;
    LfuNode<K, V> next;
}
```

---

# 26. LFU Maps

```java
private final Map<K, LfuNode<K, V>> nodes =
        new HashMap<>();

private final Map<Integer,
        DoublyLinkedList<K, V>> frequencyLists =
        new HashMap<>();

private int minFrequency;
```

---

# 27. LFU get()

Pseudo-code:

```text
get(key)

    node = nodes.get(key)

    if node does not exist
        return null

    oldFrequency = node.frequency

    remove node from old frequency list

    node.frequency++

    add node to new frequency list

    if old frequency list becomes empty
        update minFrequency

    return node.value
```

---

# 28. LFU put()

Pseudo-code:

```text
put(key, value)

    if key already exists
        update value
        increase frequency
        return

    if cache is full

        find minFrequency list

        remove least recently used node

        remove from nodes

    create new node

    frequency = 1

    add to frequency 1 list

    minFrequency = 1
```

---

# 29. LFU Complexity

With the correct data structures:

| Operation | Complexity |
|---|---:|
| get | O(1) |
| put | O(1) |
| remove | O(1) |
| eviction | O(1) |

Memory:

```text
O(capacity)
```

---

# 30. Step 9 — Separate Cache From Eviction Policy

The first LRU implementation can tightly couple:

```text
Cache
+
LRU algorithm
```

But this violates the Open/Closed Principle.

Instead create:

```java
public interface EvictionPolicy<K> {

    void onGet(K key);

    void onPut(K key);

    void onRemove(K key);

    K evict();

    void clear();
}
```

Now:

```text
Cache
   │
   ▼
EvictionPolicy
   │
   ├── LRU
   ├── LFU
   ├── FIFO
   ├── Random
   └── Future policies
```

---

# 31. Improved Architecture

```text
             Cache
               │
       ┌───────┴────────┐
       │                │
   Storage          EvictionPolicy
       │                │
    HashMap       ┌─────┼─────┐
                  │     │     │
                 LRU   LFU   FIFO
```

This is much more extensible.

---

# 32. Storage Abstraction

You can also abstract storage:

```java
public interface CacheStore<K, V> {

    V get(K key);

    void put(K key, V value);

    V remove(K key);

    boolean containsKey(K key);

    int size();

    void clear();
}
```

Implementation:

```java
public class HashMapCacheStore<K, V>
        implements CacheStore<K, V> {
}
```

Now the cache becomes:

```text
Cache
 ├── CacheStore
 └── EvictionPolicy
```

---

# 33. Step 10 — Create Cache Configuration

```java
public class CacheConfig {

    private int capacity;

    private Duration ttl;

    private EvictionPolicyType evictionPolicy;

    private boolean statisticsEnabled;

    private boolean threadSafe;
}
```

Enum:

```java
public enum EvictionPolicyType {

    LRU,
    LFU,
    FIFO
}
```

Example:

```java
CacheConfig config =
        new CacheConfig();

config.setCapacity(1000);
config.setTtl(Duration.ofMinutes(10));
config.setEvictionPolicy(
        EvictionPolicyType.LRU
);
```

---

# 34. Step 11 — Add TTL

TTL:

> Time To Live.

Example:

```text
put("user:1", user)

TTL = 5 minutes
```

After five minutes:

```text
get("user:1")
       ↓
expired
       ↓
remove
       ↓
return null
```

---

# 35. Entry With Expiration

```java
class CacheEntry<V> {

    private final V value;

    private final long expiresAt;

    CacheEntry(
            V value,
            long expiresAt) {

        this.value = value;
        this.expiresAt = expiresAt;
    }

    boolean isExpired() {

        return System.currentTimeMillis()
                >= expiresAt;
    }
}
```

---

# 36. TTL Strategies

There are two common approaches.

## Lazy expiration

Check expiration during:

```text
get()
```

Advantages:

- Simple
- No background thread

Disadvantage:

Expired entries may occupy memory until accessed.

---

## Active expiration

Background cleanup:

```text
ScheduledExecutorService
        ↓
periodically scan
        ↓
remove expired entries
```

But scanning the entire cache is:

```text
O(n)
```

and can become expensive.

A future optimization is a priority queue ordered by expiration time.

---

# 37. Step 12 — Add Maximum Weight

Capacity does not necessarily need to mean number of entries.

For example:

```text
entry A = 10 KB
entry B = 50 KB
entry C = 1 MB
```

You may want:

```text
maximumWeight = 100 MB
```

instead of:

```text
maximumEntries = 10,000
```

Create:

```java
public interface Weigher<K, V> {

    long weigh(K key, V value);
}
```

Then:

```java
totalWeight += weigher.weigh(key, value);
```

Evict until:

```text
totalWeight <= maximumWeight
```

---

# 38. Step 13 — Cache Statistics

Create:

```java
public class CacheStats {

    private long hits;

    private long misses;

    private long puts;

    private long evictions;

    private long removals;
}
```

Useful metrics:

```text
hits
misses
hitRate
missRate
evictions
loadSuccess
loadFailure
loadTime
currentSize
```

---

# 39. Hit Rate

```text
hitRate =
    hits / (hits + misses)
```

Example:

```text
hits = 800
misses = 200

hitRate = 80%
```

Statistics can help identify:

```text
Bad cache size
Bad TTL
Wrong eviction policy
Poor cache keys
Low locality
```

---

# 40. Thread Safety — Do We Need Locking?

Yes, if the cache is shared between multiple application threads.

For example, a Spring Boot application may have:

```text
HTTP Request 1 → Thread 1
HTTP Request 2 → Thread 2
HTTP Request 3 → Thread 3
HTTP Request 4 → Thread 4
```

All could execute:

```java
cache.get(key);
cache.put(key, value);
```

simultaneously.

A normal:

```java
HashMap
```

is not sufficient for safe concurrent mutation.

---

# 41. Why ConcurrentHashMap Alone Is Not Enough

A common mistake is:

```java
ConcurrentHashMap<K, Node<K,V>>
```

and assuming the entire LRU is thread-safe.

It isn't.

Because LRU requires multiple operations:

```text
HashMap lookup
      +
remove node
      +
move node
      +
update linked list
```

These operations must maintain a consistent invariant.

For example:

```text
map says node exists
but linked list doesn't contain it
```

That is a corrupted cache state.

---

# 42. Option 1 — synchronized

Simplest implementation:

```java
public synchronized V get(K key) {
    ...
}

public synchronized void put(
        K key,
        V value) {
    ...
}
```

Advantages:

- Very simple
- Correct if all mutable operations are protected

Disadvantages:

- One lock for the entire cache
- Concurrent operations serialize

---

# 43. Option 2 — ReentrantLock

```java
private final ReentrantLock lock =
        new ReentrantLock();
```

Then:

```java
lock.lock();

try {

    // mutate cache

} finally {

    lock.unlock();
}
```

This gives more flexibility.

For example:

```java
lock.tryLock()
```

or:

```java
lock.lockInterruptibly()
```

---

# 44. Option 3 — Read/Write Lock

It may seem attractive to use:

```java
ReentrantReadWriteLock
```

But pure LRU has an important problem.

`get()` changes the LRU order:

```text
get(A)
    ↓
A becomes recently used
```

Therefore `get()` is not actually read-only.

So:

```text
get()
```

often requires an exclusive write lock.

This is an important concurrency design lesson.

---

# 45. Option 4 — ConcurrentHashMap + Lock

A possible architecture:

```text
ConcurrentHashMap
        +
ReentrantLock
        +
DoublyLinkedList
```

The map provides concurrent lookup characteristics, while the lock protects the ordering structure.

However, if almost every operation still needs the same lock, the `ConcurrentHashMap` may provide limited additional benefit.

Start with:

```text
HashMap + ReentrantLock
```

and benchmark before complicating the design.

---

# 46. Cache Invariants

A good cache implementation should explicitly define invariants.

For example:

```text
1. map.size == number of nodes in linked list

2. Every map node exists exactly once in the list

3. Every list node exists in the map

4. size <= capacity

5. head.prev == null

6. tail.next == null

7. No circular references

8. totalWeight <= maximumWeight
```

For LFU:

```text
1. Every node belongs to exactly one frequency list

2. node.frequency matches its list

3. minFrequency points to the lowest active frequency

4. Every node exists in nodes map
```

Add internal validation during development:

```java
assertInvariants();
```

This is extremely useful for debugging concurrency and eviction bugs.

---

# 47. Step 14 — Prevent Cache Stampede

Consider:

```text
cache miss
    ↓
database call
```

Suppose 100 requests arrive simultaneously.

All 100 see:

```text
MISS
```

Then:

```text
100 DB queries
```

This is called a:

> Cache Stampede / Cache Thundering Herd.

---

# 48. Cache Loader

Create:

```java
@FunctionalInterface
public interface CacheLoader<K, V> {

    V load(K key);
}
```

Then:

```java
V get(
        K key,
        CacheLoader<K, V> loader) {

    V value = get(key);

    if (value != null) {
        return value;
    }

    value = loader.load(key);

    put(key, value);

    return value;
}
```

But this is still vulnerable to concurrent duplicate loading.

---

# 49. Single-Flight Loading

Better:

```text
Thread 1 → MISS → loading
Thread 2 → MISS → wait
Thread 3 → MISS → wait

Thread 1 → DB
Thread 1 → cache.put()

Thread 2 → receives value
Thread 3 → receives value
```

You can implement this using:

```java
ConcurrentHashMap<K, CompletableFuture<V>>
```

Conceptually:

```text
loadingKeys

user:1 → CompletableFuture<User>
```

Only one thread performs the actual load.

---

# 50. Step 15 — Prevent Null Ambiguity

This API:

```java
V get(K key)
```

has ambiguity:

```text
null
```

could mean:

```text
key does not exist
```

or:

```text
cached value is null
```

Simplest approach:

```text
Do not allow null values.
```

Alternatively:

```java
Optional<V> get(K key);
```

or:

```java
CacheResult<V>
```

Choose explicitly.

---

# 51. Step 16 — Add Removal Causes

Instead of simply:

```java
remove(key)
```

track why the entry disappeared.

```java
public enum RemovalCause {

    EXPLICIT,
    REPLACED,
    SIZE,
    EXPIRED,
    COLLECTED,
    ERROR
}
```

Then:

```java
public interface RemovalListener<K,V> {

    void onRemoval(
            K key,
            V value,
            RemovalCause cause);
}
```

This becomes useful for:

```text
metrics
logging
debugging
resource cleanup
```

---

# 52. Step 17 — Add Cache Events

You can expose:

```java
CacheListener<K,V>
```

with:

```java
onPut()
onGet()
onEviction()
onExpiration()
onRemoval()
```

But be careful.

Do not execute expensive listeners while holding the cache lock.

Prefer:

```text
Cache operation
      ↓
record event
      ↓
release lock
      ↓
notify listener
```

or asynchronous event processing.

---

# 53. Step 18 — Build a Generic Cache

The final API could look like:

```java
public interface Cache<K, V> {

    V get(K key);

    V get(
        K key,
        CacheLoader<K, V> loader);

    void put(K key, V value);

    V remove(K key);

    void clear();

    boolean containsKey(K key);

    int size();

    CacheStats stats();
}
```

---

# 54. Builder API

Instead of a huge constructor:

```java
new Cache<>(
    1000,
    Duration.ofMinutes(5),
    LRU,
    true,
    ...
);
```

use Builder:

```java
Cache<String, User> cache =
        CacheBuilder.<String, User>newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(
                Duration.ofMinutes(5))
            .evictionPolicy(
                EvictionPolicyType.LRU)
            .statisticsEnabled(true)
            .build();
```

This is much easier to extend.

---

# 55. CacheBuilder

```java
public class CacheBuilder<K,V> {

    private int maximumSize = 1000;

    private Duration ttl;

    private EvictionPolicyType policy =
            EvictionPolicyType.LRU;

    private boolean statisticsEnabled;

    public static <K,V>
    CacheBuilder<K,V> newBuilder() {

        return new CacheBuilder<>();
    }

    public CacheBuilder<K,V>
    maximumSize(int size) {

        this.maximumSize = size;

        return this;
    }

    public CacheBuilder<K,V>
    expireAfterWrite(Duration ttl) {

        this.ttl = ttl;

        return this;
    }

    public CacheBuilder<K,V>
    evictionPolicy(
            EvictionPolicyType policy) {

        this.policy = policy;

        return this;
    }
}
```

---

# 56. Step 19 — Separate Policy From Mechanism

This is one of the most important design improvements.

The cache should not know:

```text
LRU algorithm
LFU algorithm
FIFO algorithm
```

Instead:

```java
interface EvictionPolicy<K>
```

The cache simply says:

```java
policy.onAccess(key);
```

and:

```java
K victim = policy.evict();
```

This follows:

```text
Strategy Pattern
```

---

# 57. Strategy Pattern

```text
             Cache
               │
               ▼
       EvictionPolicy
               │
       ┌───────┼────────┐
       │       │        │
      LRU     LFU      FIFO
```

You can add:

```text
Random
MRU
Second Chance
ARC
TinyLFU
Window-TinyLFU
```

without modifying the core cache.

---

# 58. Step 20 — Add Multiple Caches

MiniSpring applications often need:

```text
userCache
productCache
restaurantCache
configurationCache
permissionCache
```

Each can have different settings:

```yaml
cache:

  caches:

    users:
      maximum-size: 10000
      eviction-policy: LRU
      ttl: 10m

    products:
      maximum-size: 50000
      eviction-policy: LFU
      ttl: 30m

    configuration:
      maximum-size: 100
      eviction-policy: LRU
      ttl: 1h
```

---

# 59. CacheManager

Create:

```java
public interface CacheManager {

    <K,V> Cache<K,V> getCache(
            String name);

    void register(
            String name,
            Cache<?,?> cache);

    void remove(String name);
}
```

Implementation:

```java
public class SimpleCacheManager
        implements CacheManager {

    private final Map<String, Cache<?, ?>>
            caches = new ConcurrentHashMap<>();

    @Override
    public <K,V> Cache<K,V>
    getCache(String name) {

        return (Cache<K,V>)
                caches.get(name);
    }
}
```

---

# 60. Why CacheManager?

Without a manager:

```java
UserService
    → creates cache

ProductService
    → creates cache

OrderService
    → creates cache
```

This leads to duplicated configuration.

Instead:

```text
                 CacheManager
                 /     |      \
                /      |       \
        userCache  productCache orderCache
```

MiniSpring can manage this as a singleton bean.

---

# 61. Integrating With MiniSpring

Your MiniSpring project already has the concepts:

```text
@Component
@Service
@Repository
@Controller
Component Scanning
ApplicationContext
Dependency Injection
Bean Lifecycle
```

We can integrate the cache as another infrastructure module.

Architecture:

```text
MiniSpring
    │
    ├── BeanFactory
    │
    ├── ApplicationContext
    │
    ├── ComponentScanner
    │
    └── CacheManager
             │
             ├── userCache
             ├── productCache
             └── orderCache
```

---

# 62. Step 21 — Register CacheManager as Bean

Create:

```java
@Component
public class SimpleCacheManager
        implements CacheManager {
}
```

MiniSpring scans it:

```text
@Component
     ↓
ComponentScanner
     ↓
BeanDefinition
     ↓
ApplicationContext
     ↓
SimpleCacheManager instance
```

Now:

```java
@Service
public class UserService {

    private final CacheManager cacheManager;

    public UserService(
            CacheManager cacheManager) {

        this.cacheManager = cacheManager;
    }
}
```

MiniSpring injects it.

---

# 63. Step 22 — Add @Cacheable

Create:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Cacheable {

    String value();

    String key() default "";
}
```

Usage:

```java
@Cacheable(
    value = "users",
    key = "#userId"
)
public User findUser(Long userId) {

    return userRepository.findById(userId);
}
```

Desired behavior:

```text
findUser(10)
     ↓
Cache lookup
     ↓
HIT?
 ┌───┴────┐
YES      NO
 │        │
return    DB
          ↓
        cache
          ↓
        return
```

---

# 64. Step 23 — MiniSpring Proxy

Your MiniSpring can intercept methods using a proxy.

Conceptually:

```text
Caller
   │
   ▼
Proxy
   │
   ├── inspect @Cacheable
   │
   ├── generate cache key
   │
   ├── cache.get()
   │
   ├── HIT → return
   │
   └── MISS
          ↓
       target method
          ↓
       cache.put()
          ↓
       return
```

This is conceptually similar to Spring's proxy-based infrastructure.

---

# 65. Cache Proxy Pseudocode

```java
public Object invoke(
        Object proxy,
        Method method,
        Object[] args) {

    Cacheable annotation =
            method.getAnnotation(
                    Cacheable.class);

    if (annotation == null) {

        return method.invoke(
                target,
                args);
    }

    Cache<Object,Object> cache =
            cacheManager.getCache(
                    annotation.value());

    Object key =
            keyGenerator.generate(
                    method,
                    args);

    Object cached =
            cache.get(key);

    if (cached != null) {
        return cached;
    }

    Object result =
            method.invoke(
                    target,
                    args);

    cache.put(key, result);

    return result;
}
```

---

# 66. Step 24 — Cache Key Generation

Do not simply concatenate arguments:

```java
userId + ":" + type
```

because collisions can occur.

Create:

```java
public interface CacheKeyGenerator {

    Object generate(
            Method method,
            Object[] arguments);
}
```

Possible default key:

```text
method + arguments
```

Example:

```text
UserService.findUser
+
[100]
```

Result:

```text
UserService.findUser(100)
```

---

# 67. Better Key Design

For:

```java
getUser(
    Long id,
    String country
)
```

Use:

```text
UserService:getUser:100:IN
```

or a structured immutable key:

```java
record CacheKey(
        String targetClass,
        String method,
        List<Object> arguments
) {}
```

Avoid mutable objects as HashMap keys.

---

# 68. Step 25 — Add @CacheEvict

Create:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CacheEvict {

    String value();

    String key() default "";

    boolean allEntries() default false;
}
```

Usage:

```java
@CacheEvict(
    value = "users",
    key = "#userId"
)
public void deleteUser(Long userId) {
}
```

Flow:

```text
deleteUser()
      ↓
DB delete
      ↓
cache.remove()
```

---

# 69. Cache Eviction Timing

There are two possibilities.

### Before method

```text
cache.remove()
    ↓
method()
```

### After successful method

```text
method()
    ↓
success
    ↓
cache.remove()
```

Usually you should think carefully about transaction semantics.

For example:

```text
DB update
   ↓
transaction rollback
```

If you evict before the transaction succeeds, the cache state may no longer correspond to the database state.

A more advanced MiniSpring version can integrate cache eviction with transaction completion.

---

# 70. Step 26 — Add @CachePut

`@CachePut` means:

```text
Always execute method
+
update cache with result
```

Example:

```java
@CachePut(
    value = "users",
    key = "#user.id"
)
public User updateUser(User user) {
    return repository.save(user);
}
```

Flow:

```text
method()
   ↓
DB update
   ↓
cache.put()
```

---

# 71. Final MiniSpring Cache Architecture

```text
                    MiniSpring
                        │
                 ApplicationContext
                        │
              ┌─────────┴─────────┐
              │                   │
         CacheManager         BeanFactory
              │
       ┌──────┼─────────┐
       │      │         │
     users products  orders
       │      │         │
      Cache  Cache     Cache
       │      │         │
       └──────┼─────────┘
              │
        Cache Engine
              │
      ┌───────┴────────┐
      │                │
 CacheStore       EvictionPolicy
      │                │
   HashMap       ┌──────┼──────┐
                 │      │      │
                LRU    LFU    FIFO
```

---

# 72. Configuration-Driven MiniSpring Cache

Example:

```yaml
mini-spring:

  cache:

    enabled: true

    caches:

      users:
        maximum-size: 10000
        eviction-policy: LRU
        expire-after-write: 10m
        statistics: true

      products:
        maximum-size: 50000
        eviction-policy: LFU
        expire-after-access: 30m
        statistics: true
```

MiniSpring should:

```text
Configuration
     ↓
Configuration Binder
     ↓
CacheProperties
     ↓
CacheAutoConfiguration
     ↓
CacheManager
     ↓
Named Caches
```

---

# 73. Step 27 — Cache Auto Configuration

Create:

```java
@Configuration
public class CacheAutoConfiguration {

    @Bean
    public CacheManager cacheManager(
            CacheProperties properties) {

        return new ConfigurableCacheManager(
                properties);
    }
}
```

If your MiniSpring doesn't yet support:

```java
@Configuration
@Bean
```

this becomes an excellent next feature.

---

# 74. MiniSpring Enhancement Roadmap

Your cache project can drive several MiniSpring improvements.

## Phase 1

Implement:

```text
@Component
@Service
@Repository
@Controller
```

## Phase 2

Add:

```text
@Bean
@Configuration
```

## Phase 3

Add:

```text
@Cacheable
@CacheEvict
@CachePut
```

## Phase 4

Add:

```text
Proxy
MethodInterceptor
```

## Phase 5

Add:

```text
@Around
@Before
@After
```

## Phase 6

Generalize into:

```text
AOP framework
```

Then caching becomes just one interceptor.

---

# 75. General AOP Architecture

Instead of hardcoding:

```text
CacheProxy
```

create:

```java
interface MethodInterceptor {

    Object invoke(
        MethodInvocation invocation);
}
```

Then:

```text
Method
 │
 ▼
Interceptor Chain
 │
 ├── LoggingInterceptor
 │
 ├── SecurityInterceptor
 │
 ├── TransactionInterceptor
 │
 ├── CacheInterceptor
 │
 └── MetricsInterceptor
 │
 ▼
Target Method
```

This is a major MiniSpring architectural improvement.

---

# 76. Example Interceptor Chain

```text
HTTP Request
     ↓
Controller Proxy
     ↓
Security
     ↓
Logging
     ↓
Cache
     ↓
Transaction
     ↓
Service
```

Now your cache isn't tightly coupled to the DI framework.

It is just another interceptor.

---

# 77. Step 28 — Add Cache Namespaces

Imagine:

```text
users
products
orders
```

Each should have independent capacity.

```text
users     → 10,000
products  → 50,000
orders    → 20,000
```

This prevents one cache from consuming all memory.

---

# 78. Step 29 — Add Per-Cache Statistics

Example:

```text
users

size             = 8,923
capacity         = 10,000
hits             = 1,200,340
misses           = 120,300
hit rate         = 90.9%
evictions        = 30,200
expirations      = 10,100
```

Expose:

```java
CacheStats stats();
```

Later expose an endpoint:

```text
GET /actuator/caches
```

for MiniSpring applications.

---

# 79. Step 30 — Add Metrics

Eventually integrate with:

```text
Micrometer-like abstraction
```

Metrics:

```text
cache.get
cache.put
cache.hit
cache.miss
cache.eviction
cache.expiration
cache.load
```

Possible tags:

```text
cache=userCache
policy=LRU
```

---

# 80. Step 31 — Add Refresh-After-Write

TTL:

```text
write
  ↓
10 minutes
  ↓
expired
```

Refresh:

```text
write
  ↓
10 minutes
  ↓
refresh asynchronously
```

The old value may continue serving while a background refresh happens.

This reduces latency spikes.

---

# 81. Step 32 — Add Stale-While-Revalidate

Flow:

```text
Request
   ↓
Entry expired
   ↓
Is stale value allowed?
   ├── YES → return stale value
   │          +
   │        async refresh
   │
   └── NO → synchronous load
```

Configuration:

```yaml
stale-while-revalidate: true
```

---

# 82. Step 33 — Add Negative Caching

Suppose:

```text
getUser(999999)
```

doesn't exist.

Without negative caching:

```text
Request 1 → DB
Request 2 → DB
Request 3 → DB
...
```

You can cache:

```text
USER_NOT_FOUND
```

with a short TTL.

Example:

```text
negative TTL = 30 seconds
```

Be careful with rapidly changing data.

---

# 83. Step 34 — Add Serialization

For local in-memory cache, objects can remain as Java objects.

For future distributed caching:

```text
Object
 ↓
Serializer
 ↓
byte[]
```

Create:

```java
public interface Serializer {

    byte[] serialize(Object value);

    <T> T deserialize(
        byte[] data,
        Class<T> type);
}
```

Possible implementations:

```text
Java Serialization
JSON
Jackson
Kryo
Protobuf
Avro
```

For production systems, choose serialization based on compatibility, performance, and security requirements rather than defaulting to Java native serialization.

---

# 84. Step 35 — Add Persistence/Distributed Backend

Your current cache:

```text
Application
     ↓
Local Memory
```

Future architecture:

```text
Application
     ↓
Cache API
     ↓
CacheStore
     ├── InMemoryStore
     ├── RedisStore
     ├── DatabaseStore
     └── DistributedStore
```

This is another reason to separate:

```text
CacheStore
```

from:

```text
EvictionPolicy
```

---

# 85. Important Distributed Cache Question

LRU/LFU implemented inside each application instance behaves differently when there are multiple instances.

Example:

```text
             Load Balancer
             /           \
            /             \
       Server A        Server B
          │                │
       LRU Cache         LRU Cache
```

The caches are independent.

Therefore:

```text
Server A knows key X
Server B doesn't
```

A distributed cache such as Redis can provide shared state.

But then:

```text
network latency
serialization
distributed consistency
failure handling
connection pooling
```

become part of the problem.

---

# 86. Step 36 — Cache Stampede Protection

Use:

```java
ConcurrentHashMap<K, CompletableFuture<V>>
```

Example concept:

```text
loading:
user:100 → Future<User>
```

Algorithm:

```text
get(user:100)

    cache hit?
        YES → return

    loading map contains key?
        YES → wait for Future

    otherwise:
        create Future
        register Future
        load DB
        cache.put()
        complete Future
        remove loading entry
```

Be extremely careful to remove failed futures so one failed load doesn't permanently poison the key.

---

# 87. Step 37 — Handle Exceptions

What happens when loader fails?

```text
DB unavailable
```

Do not cache the exception indefinitely by default.

Possible strategies:

```text
FAIL
RETRY
CACHE_FAILURE_FOR_SHORT_TTL
RETURN_STALE
```

Make this configurable.

---

# 88. Step 38 — Backpressure

If many keys miss simultaneously:

```text
10,000 requests
     ↓
10,000 DB calls
```

You can introduce:

```text
Semaphore
```

or:

```text
ThreadPoolExecutor
```

to limit concurrent cache loads.

Example:

```text
Maximum concurrent loads = 100
```

This protects the database.

---

# 89. Step 39 — Thread Pool Considerations

Do not blindly create:

```java
Executors.newCachedThreadPool()
```

inside every cache.

Prefer application-managed resources.

For MiniSpring:

```text
ApplicationContext
      ↓
TaskExecutor
      ↓
Cache refresh/load
```

This allows:

```yaml
cache:
  refresh:
    threads: 4
```

---

# 90. Step 40 — Avoid Locking During Slow Operations

Never do:

```java
lock.lock();

try {

    database.load();

    cache.put();

} finally {

    lock.unlock();
}
```

This is dangerous.

Instead:

```text
lock
 ↓
check cache
 ↓
unlock

database.load()

lock
 ↓
put
 ↓
unlock
```

For stampede protection, use per-key coordination rather than holding the global cache lock during I/O.

---

# 91. Concurrency Architecture

Recommended starting point:

```text
                     Cache
                       │
                 ReentrantLock
                       │
             ┌─────────┴─────────┐
             │                   │
          HashMap           EvictionPolicy
                                │
                          DoublyLinkedList
```

Later:

```text
ConcurrentHashMap
       +
Fine-grained locking
       +
Per-key loading
```

Only after benchmarking.

---

# 92. Step 41 — Add Unit Tests

Minimum test suite:

```text
LruCacheTest
LfuCacheTest
CacheExpirationTest
CacheStatsTest
CacheConcurrencyTest
CacheLoaderTest
CacheEvictionTest
CacheManagerTest
MiniSpringCacheableTest
```

---

# 93. LRU Test Matrix

Test:

```text
empty cache
single item
capacity 1
capacity N
get existing
get missing
put existing
put new
remove
clear
eviction
access changes ordering
```

---

# 94. LFU Test Matrix

Test:

```text
frequency increment
frequency tie
LRU tie breaker
minimum frequency update
eviction
update existing value
remove
clear
capacity 1
```

---

# 95. Concurrency Tests

Create:

```java
ExecutorService executor =
        Executors.newFixedThreadPool(20);
```

Run:

```text
100,000 puts
100,000 gets
100,000 removes
```

from multiple threads.

After completion validate:

```text
size <= capacity
```

and:

```text
assertInvariants()
```

---

# 96. Race Condition Test

A particularly important test:

```text
Thread 1 → get(A)
Thread 2 → put(A)
Thread 3 → remove(A)
Thread 4 → get(A)
Thread 5 → put(B)
```

Run this repeatedly.

Use:

```java
CountDownLatch
```

to start many threads simultaneously.

---

# 97. Benchmarking

Do not assume:

```text
ConcurrentHashMap = faster
```

or:

```text
Lock-free = faster
```

Benchmark.

Measure:

```text
throughput
latency
p50
p95
p99
memory
GC
lock contention
```

Use:

```text
JMH
```

rather than a simple:

```java
System.currentTimeMillis()
```

benchmark.

---

# 98. Benchmark Scenarios

Benchmark:

```text
100% get
90% get / 10% put
50% get / 50% put
high contention
large cache
small cache
LRU
LFU
TTL enabled
statistics enabled
loader enabled
```

---

# 99. Important Optimization: Avoid Boxing Where Appropriate

LFU uses:

```java
Integer frequency
```

which can create overhead.

For a learning implementation this is perfectly acceptable.

Later you can consider specialized structures or primitive collections if profiling shows boxing is significant.

Do not optimize this prematurely.

---

# 100. Important Optimization: Avoid Full Cache Scans

Bad:

```text
every 1 second
    scan every entry
```

Complexity:

```text
O(n)
```

Better:

```text
PriorityQueue
```

ordered by:

```text
expiration time
```

Then:

```text
earliest expiration
       ↓
remove
       ↓
next expiration
```

---

# 101. Important Optimization: Timer Wheel

For very large TTL workloads, investigate:

```text
Hierarchical Timing Wheel
```

Instead of:

```text
one timer per entry
```

This is a more advanced topic.

---

# 102. Important Optimization: Admission Policy

Eviction asks:

> Which existing item should we remove?

Admission asks:

> Should this new item even enter the cache?

This is a more advanced optimization.

Example:

```text
Cache almost full

New item appears once
Existing item is accessed 1000 times
```

Blindly admitting the new item may hurt hit rate.

This leads toward:

```text
TinyLFU
Window TinyLFU
```

and more sophisticated cache designs.

---

# 103. LRU vs LFU

| Property | LRU | LFU |
|---|---|---|
| Tracks recency | Yes | Yes/tie-break |
| Tracks frequency | No | Yes |
| Implementation | Easier | More complex |
| Good for temporal locality | Yes | Depends |
| Long-term popular items | Can be evicted | Better retained in many workloads |
| Metadata | Low | Higher |
| Complexity | O(1) possible | O(1) possible |

Do not choose purely from theory.

Measure against your workload.

---

# 104. Eviction Policy Extension

Eventually:

```java
public interface EvictionPolicy<K> {

    void recordAccess(K key);

    void recordInsert(K key);

    void recordRemove(K key);

    K selectVictim();

    void clear();
}
```

Possible implementations:

```text
LruEvictionPolicy
LfuEvictionPolicy
FifoEvictionPolicy
MruEvictionPolicy
RandomEvictionPolicy
TinyLfuEvictionPolicy
```

---

# 105. Step 42 — Add Cache Lifecycle

Because MiniSpring manages beans, cache should support lifecycle:

```java
public interface Lifecycle {

    void start();

    void stop();
}
```

For caches with:

```text
background expiration
refresh threads
statistics reporters
```

the lifecycle becomes important.

MiniSpring:

```text
ApplicationContext.start()
        ↓
Cache.start()

ApplicationContext.stop()
        ↓
Cache.stop()
```

---

# 106. Step 43 — Graceful Shutdown

Never leave:

```text
ScheduledExecutorService
```

running after application shutdown.

Implement:

```java
public void close() {

    scheduler.shutdown();

}
```

Prefer:

```java
AutoCloseable
```

where appropriate.

---

# 107. Step 44 — Memory Safety

A cache is intentionally keeping objects alive.

Therefore:

```text
large cache
    ↓
large heap retention
```

Consider:

```text
maximumSize
maximumWeight
TTL
object size
value size
```

Future advanced options:

```text
soft references
weak references
off-heap storage
serialized values
```

But reference-based strategies have trade-offs and should not be used as a substitute for explicit cache sizing.

---

# 108. Step 45 — Add Cache Warmup

When application starts:

```text
Application startup
       ↓
load frequently required data
       ↓
populate cache
```

Example:

```text
countries
configuration
permissions
feature flags
```

MiniSpring could support:

```java
@PostConstruct
```

or a dedicated:

```java
CacheWarmup
```

interface.

---

# 109. Step 46 — Add Cache Refresh

Create:

```java
public interface CacheRefresher<K,V> {

    V refresh(K key);
}
```

Scheduler:

```text
ScheduledExecutorService
          ↓
refresh
          ↓
cache.put()
```

Do not refresh everything blindly.

Only refresh:

```text
popular keys
near-expiration keys
configured keys
```

---

# 110. Step 47 — Add Observability

Expose:

```text
/cache/stats
```

Example:

```json
{
  "users": {
    "size": 923,
    "capacity": 1000,
    "hits": 1200340,
    "misses": 120300,
    "evictions": 30200,
    "hitRate": 0.909
  }
}
```

For MiniSpring:

```text
CacheManager
     ↓
CacheMetrics
     ↓
HTTP Endpoint
```

---

# 111. Step 48 — Add Admin Operations

Useful endpoints:

```text
GET    /cache
GET    /cache/{name}
GET    /cache/{name}/stats
DELETE /cache/{name}
DELETE /cache/{name}/{key}
POST   /cache/{name}/clear
```

Protect these endpoints with authentication/authorization in real applications.

---

# 112. Step 49 — Configuration Validation

Invalid:

```yaml
maximum-size: -1
```

Invalid:

```yaml
ttl: -10m
```

Invalid:

```yaml
eviction-policy: UNKNOWN
```

MiniSpring should fail fast:

```text
Application startup
       ↓
Configuration validation
       ↓
Invalid
       ↓
clear error
       ↓
application does not start
```

---

# 113. Step 50 — Cache Consistency

Once caching database entities:

```text
Database
    +
Cache
```

you must define:

```text
What happens after update?
```

Possible approaches:

### Cache Aside

```text
Application
    ↓
Cache
    ↓
miss
    ↓
Database
    ↓
Cache
```

### Write Through

```text
Application
    ↓
Cache
    ↓
Database
```

### Write Behind

```text
Application
    ↓
Cache
    ↓
async database write
```

Each has different consistency and failure behavior.

For the first MiniSpring implementation, start with **cache-aside**.

---

# 114. Recommended MiniSpring First Version

Implement:

```text
@Cacheable
@CacheEvict
@CachePut
```

with:

```text
CacheManager
Cache
CacheStore
EvictionPolicy
CacheKeyGenerator
CacheStats
```

Architecture:

```text
              Application
                   │
                   ▼
              MiniSpring
                   │
              AOP Proxy
                   │
          ┌────────┴────────┐
          │                 │
     CacheInterceptor    Target
          │
     CacheManager
          │
       Cache
          │
   ┌──────┴─────────┐
   │                │
CacheStore     EvictionPolicy
   │                │
HashMap         LRU / LFU
```

---

# 115. Recommended Development Order

Do not implement everything simultaneously.

Follow this exact progression.

## Phase 1 — Basic Data Structure

Implement:

```text
1. Cache interface
2. SimpleCache
3. HashMap storage
4. Unit tests
```

---

## Phase 2 — LRU

Implement:

```text
5. Node
6. DoublyLinkedList
7. LruCache
8. O(1) eviction
9. Tests
```

---

## Phase 3 — LFU

Implement:

```text
10. Frequency node
11. Frequency buckets
12. minFrequency
13. LFU eviction
14. LRU tie breaker
15. Tests
```

---

## Phase 4 — Strategy Pattern

Implement:

```text
16. EvictionPolicy
17. LruEvictionPolicy
18. LfuEvictionPolicy
19. FIFO policy
20. Configurable policy
```

---

## Phase 5 — Thread Safety

Implement:

```text
21. ReentrantLock
22. Cache invariants
23. Concurrent tests
24. Race tests
```

---

## Phase 6 — Expiration

Implement:

```text
25. TTL
26. expire-after-write
27. expire-after-access
28. lazy expiration
29. active expiration
```

---

## Phase 7 — Observability

Implement:

```text
30. CacheStats
31. hit/miss
32. eviction statistics
33. removal causes
34. listeners
```

---

## Phase 8 — Loading

Implement:

```text
35. CacheLoader
36. get(key, loader)
37. single-flight loading
38. CompletableFuture
39. load failure handling
```

---

## Phase 9 — Cache Manager

Implement:

```text
40. CacheManager
41. Named caches
42. Cache configuration
43. Builder API
```

---

## Phase 10 — MiniSpring

Implement:

```text
44. CacheManager Bean
45. @Cacheable
46. @CacheEvict
47. @CachePut
48. CacheInterceptor
49. CacheKeyGenerator
```

---

## Phase 11 — MiniSpring AOP

Implement:

```text
50. MethodInterceptor
51. MethodInvocation
52. InterceptorChain
53. LoggingInterceptor
54. CacheInterceptor
55. MetricsInterceptor
```

---

## Phase 12 — Advanced

Implement:

```text
56. Refresh
57. Stale-while-revalidate
58. Weight-based eviction
59. Admission policy
60. TinyLFU
61. Distributed CacheStore
62. Metrics
63. Admin APIs
```

---

# 116. Final API

A mature version could expose:

```java
Cache<String, User> users =
    CacheBuilder.<String, User>newBuilder()
        .maximumSize(10_000)
        .evictionPolicy(
            EvictionPolicyType.LRU)
        .expireAfterWrite(
            Duration.ofMinutes(10))
        .statisticsEnabled(true)
        .build();
```

Usage:

```java
User user =
    users.get(
        userId,
        id -> userRepository.findById(id)
            .orElse(null)
    );
```

---

# 117. MiniSpring Usage

Application configuration:

```yaml
cache:

  enabled: true

  caches:

    users:
      maximum-size: 10000
      eviction-policy: LRU
      expire-after-write: 10m

    products:
      maximum-size: 50000
      eviction-policy: LFU
      expire-after-access: 30m
```

Service:

```java
@Service
public class UserService {

    @Cacheable(
        value = "users",
        key = "#id"
    )
    public User findUser(Long id) {

        return userRepository.findById(id)
                .orElse(null);
    }

    @CacheEvict(
        value = "users",
        key = "#id"
    )
    public void deleteUser(Long id) {

        userRepository.deleteById(id);
    }

    @CachePut(
        value = "users",
        key = "#user.id"
    )
    public User updateUser(User user) {

        return userRepository.save(user);
    }
}
```

---

# 118. Complete Concept Map

By the end of this project you should understand:

```text
                         CACHE
                           │
          ┌────────────────┼────────────────┐
          │                │                │
       Storage          Eviction         Expiration
          │                │                │
       HashMap       ┌──────┼──────┐      TTL
          │          │      │      │        │
          │         LRU    LFU    FIFO      │
          │          │      │               │
          └──────────┴──────┴───────────────┘
                           │
                      Thread Safety
                           │
                    ReentrantLock
                           │
                    Cache Statistics
                           │
                     Cache Loading
                           │
                   CompletableFuture
                           │
                     CacheManager
                           │
                       MiniSpring
                           │
                       AOP Proxy
                           │
               ┌───────────┼────────────┐
               │           │            │
          @Cacheable   @CachePut   @CacheEvict
```

---

# 119. Final Recommended Architecture

```text
                         MiniSpring Application
                                  │
                                  ▼
                           ApplicationContext
                                  │
                                  ▼
                            AOP Interceptor
                                  │
                     ┌────────────┴────────────┐
                     │                         │
               CacheInterceptor             Target
                     │
                     ▼
                CacheManager
                     │
              ┌──────┴────────┐
              │               │
          UserCache       ProductCache
              │               │
              ▼               ▼
          CacheEngine      CacheEngine
              │               │
       ┌──────┴───────┐ ┌─────┴────────┐
       │              │ │              │
   CacheStore   EvictionPolicy   CacheStore  EvictionPolicy
       │              │              │           │
    HashMap       LRU/LFU         HashMap      LFU/LRU
       │
       ├── TTL
       ├── Statistics
       ├── Loader
       ├── Weight
       ├── Listener
       └── Metrics
```

---

# 120. Most Important Design Lessons

## Lesson 1

Do not start with:

```text
Spring-like cache
```

Start with:

```text
HashMap
```

and build the abstractions gradually.

---

## Lesson 2

LRU teaches:

```text
HashMap
+
Doubly Linked List
```

and gives O(1) operations.

---

## Lesson 3

LFU teaches:

```text
HashMap
+
Frequency Buckets
+
Doubly Linked Lists
+
Minimum Frequency
```

---

## Lesson 4

Thread safety is not simply:

```text
ConcurrentHashMap
```

You must protect the **entire state transition** that maintains cache invariants.

---

## Lesson 5

`get()` in LRU/LFU can be a mutation.

Therefore:

```text
read operation
```

does not necessarily mean:

```text
read-only operation
```

---

## Lesson 6

Don't hold locks while performing:

```text
DB calls
HTTP calls
file I/O
slow callbacks
```

---

## Lesson 7

Use Strategy Pattern for eviction.

```text
EvictionPolicy
     ↓
LRU
LFU
FIFO
```

---

## Lesson 8

Separate:

```text
Storage
Eviction
Expiration
Loading
Statistics
Configuration
```

This keeps the system SOLID.

---

## Lesson 9

MiniSpring should not know how LRU or LFU works.

It should know:

```text
CacheManager
Cache
CacheInterceptor
```

The cache engine remains independent.

---

## Lesson 10

Caching is not just an algorithm problem.

A production cache involves:

```text
Data structure
+
Concurrency
+
Memory
+
Expiration
+
Consistency
+
Loading
+
Failure handling
+
Observability
+
Configuration
```

---

# 121. Suggested Next Enhancements

After completing the basic project, implement these in order:

```text
1. FIFO
2. MRU
3. Random eviction
4. TTL
5. expire-after-access
6. expire-after-write
7. Cache statistics
8. Removal listeners
9. Cache loader
10. Single-flight loading
11. Weight-based cache
12. Refresh-after-write
13. Stale-while-revalidate
14. Cache warmup
15. CacheManager
16. Configuration binding
17. @Cacheable
18. @CacheEvict
19. @CachePut
20. MiniSpring AOP
21. Metrics
22. Admin endpoints
23. TinyLFU
24. Distributed CacheStore
25. Redis implementation
```

---

# 122. Final Challenge

Once the basic implementation works, remove all library-based shortcuts.

Do **not** use:

```java
LinkedHashMap
```

for LRU.

Do **not** use:

```java
Guava Cache
```

or:

```text
Caffeine
```

for the core implementation.

Implement:

```text
LRU
LFU
TTL
Statistics
Thread safety
Cache loader
CacheManager
```

yourself.

Then compare your implementation against a mature cache library using JMH.

The purpose is not necessarily to outperform mature libraries.

The purpose is to understand **why those libraries are designed the way they are**.

---

# 123. Final Project Milestone

Your final MiniSpring project should eventually allow:

```java
@Service
public class ProductService {

    @Cacheable(
        value = "products",
        key = "#id"
    )
    public Product getProduct(Long id) {

        return productRepository.findById(id);
    }
}
```

with:

```yaml
cache:

  enabled: true

  defaults:
    eviction-policy: LRU
    maximum-size: 1000
    expire-after-write: 10m
    statistics: true

  caches:

    products:
      maximum-size: 50000
      eviction-policy: LFU
      expire-after-write: 30m

    users:
      maximum-size: 10000
      eviction-policy: LRU
      expire-after-access: 10m
```

And internally:

```text
@Cacheable
      ↓
MiniSpring AOP
      ↓
CacheInterceptor
      ↓
CacheManager
      ↓
Named Cache
      ↓
Thread Safety
      ↓
CacheStore
      +
EvictionPolicy
      +
Expiration
      +
Statistics
      ↓
HIT
      │
      └── return

MISS
  ↓
CacheLoader
  ↓
Database
  ↓
Cache
  ↓
return
```

That gives you a complete learning path from a **HashMap-based cache implementation** all the way to a **MiniSpring-integrated caching framework**.