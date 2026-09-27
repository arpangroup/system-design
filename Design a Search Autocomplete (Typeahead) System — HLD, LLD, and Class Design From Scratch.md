# Design a Search Autocomplete (Typeahead) System — HLD, LLD, and Class Design From Scratch

> **The interview question this guide answers:**
>
> *"Design a search autocomplete / typeahead system, like the suggestions that appear as you type into a search box. Cover both high-level design (data pipeline, sharding, caching) and low-level design (the actual data structure and algorithm that make a suggestion list fast). Follow SOLID design principles, and be ready to justify every decision when I push back — starting with why a naive prefix search isn't fast enough, no matter how good your data structure is."*
>
> This guide is structured exactly as that interview unfolds: build the obvious first data structure, show precisely where it's too slow, fix that with the one optimization that actually matters, then follow the same escalating sequence of **follow-up questions** a real interview always asks next — how the underlying popularity data gets there at all, how to shard a data structure that only hash-based sharding usually applies to, and what actually breaks first at global scale.

---

# 1. What We Are Building

We are building **MiniTypeahead** — a search autocomplete system covering:

- **Functional requirements**: given a partial query prefix, return the top-K most relevant completions, ranked by popularity, fast enough to update on every keystroke.
- **High-level design**: a Trie-based index, an offline pipeline that turns raw search-query logs into that index, prefix-aware sharding for a structure too large to fit on one machine, and caching for the small set of prefixes that receive the overwhelming majority of all traffic.
- **Low-level design**: a real Trie implementation with precomputed top-K suggestions cached at every node — the one optimization that turns "scan every completion under this prefix" into "read a pre-sorted list in constant time."
- **Scaling concerns**: why hash-based sharding, the default answer for almost everything else in this series, actively breaks a prefix-based data structure, and what to do instead.

```text
   User types "g" -> "go" -> "goo" -> "goog" ...          (one request per keystroke, debounced client-side, §29)
                        |
              Prefix Cache (hot prefixes, §26-§27)
                        | miss
              Prefix-Range Shard Router (§23-§24)  -->  Trie shard (§13-§17)
                        |                                   ^
                        v                                   |
              top-K suggestions, ALREADY sorted    Offline Trie-Build Pipeline (§19-§20)
              (no scan, no ranking at query time)   <-- aggregated search-query logs
```

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Explain why a Trie is the natural data structure for prefix lookup, and precisely where a naive Trie implementation is still too slow for a real autocomplete workload.
- Implement the single optimization that fixes it: precomputing and maintaining a sorted top-K suggestion list at every Trie node, updated incrementally as popularity data changes.
- Explain why autocomplete's read:write ratio means the index should be rebuilt from an offline pipeline, not updated synchronously on every search.
- Explain precisely why hash-based sharding — the default answer this series reaches for almost everywhere else — actively breaks a prefix-based structure, and design the sharding strategy that actually works instead.
- Design a caching layer for the small number of extremely hot prefixes that dominate real-world traffic, and a client-side debouncing strategy that keeps a keystroke-driven UI from overwhelming the backend.

---

# 3. Why This Matters (The Interview, Framed)

"Design a search autocomplete system" is a favorite systems-design question because the obvious data structure (a Trie) is correct, well-known, and *still* not the actual answer on its own — which makes it an excellent test of whether a candidate stops at "I know the textbook data structure" or keeps reasoning past it:

- **The data structure alone doesn't solve the problem** — a Trie answers "does this prefix exist" efficiently, but the actual requirement is "give me the top-K *most popular* completions, ranked," which a plain Trie has no way to answer without an expensive scan at query time. Naming this gap, and fixing it, is the real test.
- **Sharding this specific structure inverts the usual answer** — every other guide in this series reaches for hash-based (or consistent) sharding as the default; a prefix-based structure is the specific, instructive counter-example where that default actively breaks the workload, and knowing why is a genuine, transferable systems-design insight.
- **The read:write ratio argument shows up again, with a twist** — like URL shortening, reads vastly outnumber writes here too, but the fix isn't caching alone; it's recognizing that "writes" (new popularity data) don't need to be reflected instantly at all, which changes the entire pipeline architecture.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Language | Java 21 | Matches this guide's class diagrams and pattern implementations. |
| Core index | A Trie, with a precomputed top-K list cached at every node | Turns prefix lookup plus ranking into one `O(prefix length)` operation, with zero scanning or sorting at query time (§14-§17). |
| Index construction | An offline batch pipeline over aggregated search-query logs | The read:write ratio means the index can be rebuilt periodically rather than updated synchronously per search (§19-§20). |
| Sharding | Prefix-range partitioning, never a hash | A hash destroys the prefix locality every lookup depends on — explained precisely in §22-§24. |
| Cache | An in-memory LRU cache for hot prefixes | Real query traffic is heavily Zipfian — a small number of short, common prefixes account for a large share of all requests (§26-§27). |

---

# 5. Project Structure

```text
minitypeahead/
├── src/main/java/com/example/minitypeahead/
│   ├── trie/
│   │   ├── TrieNode.java                                   // §14, §17
│   │   ├── Trie.java                                        // §14, §17
│   │   └── Suggestion.java
│   ├── pipeline/
│   │   └── TrieBuildPipeline.java                            // §20
│   ├── sharding/
│   │   ├── PrefixRangeShardRouter.java                        // §24
│   │   └── ShardRange.java
│   ├── cache/
│   │   └── PrefixCache.java (LRU)                              // §27
│   └── client/
│       └── DebouncedAutocompleteClient.java                    // §29
└── src/test/java/com/example/minitypeahead/
    ├── TopKPropagationTest.java
    ├── PrefixShardRoutingTest.java
    └── PrefixCacheEvictionTest.java
```

---

# 6. Step 1 — Clarifying Requirements Before Designing Anything

> **Candidate's clarifying questions:** *"How many distinct historical queries are we ranking over — millions, or billions? Does 'popular' mean globally popular, or personalized per user? What's the acceptable staleness for newly-popular queries to show up — seconds, or is a longer delay acceptable? What's the target end-to-end latency per keystroke?"*

Exactly as every prior guide in this series argues, narrowing an intentionally broad prompt before designing anything is the first move that separates a strong answer from a shallow one. For this guide, we settle on a concrete, realistic scope: **billions of distinct historical queries**, **global popularity ranking** as the core requirement (with personalization named as a real layered enhancement, §33-§34), **staleness on the order of minutes-to-hours acceptable** for newly-trending queries, and a **target of well under 100 milliseconds** end-to-end per keystroke.

---

# 7. Functional Requirements

- Given a **partial query prefix**, return the **top-K** completions, ranked by historical popularity.
- Update the underlying popularity data from real search-query traffic **continuously**, even though the served index itself refreshes periodically.
- Support **prefixes of any length**, from a single character up to a full query.
- Serve suggestions with **low enough latency** to update as a user types, not just after they stop.

---

# 8. Non-Functional Requirements

| Requirement | What it means concretely | Where this guide addresses it |
|---|---|---|
| **Very low per-keystroke latency** | A suggestion list must feel instantaneous, well under 100ms, on every single character typed | Precomputed top-K at every Trie node (§16-§17), hot-prefix caching (§26-§27) |
| **Massive read:write skew** | Autocomplete requests vastly outnumber the rate at which underlying popularity data actually needs to change | The offline, periodic rebuild pipeline (§19-§20) |
| **Horizontal scalability of the index itself** | Billions of distinct historical queries do not fit in one machine's memory | Prefix-range sharding (§22-§24) |
| **Acceptable staleness, not real-time writes** | A newly-trending query does not need to appear in suggestions within milliseconds of its first occurrence | The entire batch-pipeline architecture (§18-§20) |
| **Extensibility** | Adding personalization or fuzzy matching later must not require redesigning the core index | Both are built as layers on top of the same Trie, never mixed into it (§33-§36) |

---

# 9. Follow-up Question 1 — "What Are the Core Nouns Here, Before We Draw Any Boxes?"

> **Interviewer:** *"Before architecture — what actually exists in this system, and how does it relate?"*

This is the same deliberate pivot every prior guide in this series makes — naming the domain model before naming components keeps the design honest about what actually needs solving.

---

# 10. Identifying the Core Domain Entities

| Entity | Represents | Key relationships |
|---|---|---|
| **TrieNode** | One character position in the shared prefix space of every indexed query | Has children keyed by the next character, and a cached, pre-sorted top-K list (§14, §16-§17) |
| **Suggestion** | One (query text, popularity score) pair | What a top-K list actually stores and returns |
| **QueryLogEntry** | One raw, real search event | The raw input the offline pipeline aggregates into popularity scores (§19-§20) |
| **ShardRange** | A contiguous range of the prefix keyspace owned by one Trie shard | Never a hash bucket — a real character-range interval (§23-§24) |
| **PrefixCache** | An in-memory cache of hot prefixes' already-computed top-K lists | Consulted before any shard is ever contacted (§26-§27) |

Every section from §11 onward either builds infrastructure **around** these entities or builds the entities **themselves** as real, working class designs — nothing introduced later is untraceable back to this table.

---

# 11. High-Level Architecture Overview

```text
                        Search traffic (millions of real queries, continuously)
                                          |
                              Query Log Aggregation (§19)
                                          |
                              Offline Trie-Build Pipeline (§20)
                                          |
                              Published, versioned Trie shards
                                          |
   User keystroke -> DebouncedAutocompleteClient (§29) -> PrefixCache (§27) --miss--> ShardRouter (§24) -> Trie shard
                                          |
                              top-K suggestions, already ranked
```

Two structurally different data flows, deliberately kept separate: the **write side** (query logs accumulating, periodically aggregated and rebuilt into a new Trie) happens on its own schedule, entirely decoupled from user-facing latency; the **read side** (a keystroke resolving to a ranked suggestion list) is the only path with a real-time latency budget, and every optimization in this guide targets that path specifically.

---

# 12. Follow-up Question 2 — "What Data Structure Actually Answers 'Give Me Completions of This Prefix,' and Why?"

> **Interviewer:** *"Forget ranking for a second. Given a set of strings and a prefix, what's the right data structure to find every string starting with that prefix, efficiently?"*

§13 names it; §14 implements it.

---

# 13. Why a Trie Is the Natural Fit

A **Trie** (prefix tree) stores a set of strings by sharing common prefixes as shared paths through a tree of characters — the string `"google"` and `"goggles"` share the path `g -> o -> g`, diverging only at the fourth character. Finding every stored string starting with a given prefix becomes: walk the tree following the prefix's characters one node at a time — an `O(prefix length)` operation, entirely independent of how many total strings are stored — and then explore the subtree rooted at wherever that walk ends. That second half, exploring the subtree, is exactly where a naive implementation is still too slow, which §15 states precisely.

---

# 14. Implementing the Basic Trie

```java
public record Suggestion(String query, long popularity) { }
```

```java
public final class TrieNode {
    final Map<Character, TrieNode> children = new HashMap<>();
    boolean isEndOfQuery;
    long popularityIfEndOfQuery; // only meaningful when isEndOfQuery is true
    List<Suggestion> topKSuggestions = List.of(); // §16-§17 -- the optimization this entire guide is built around
}
```

```java
public final class Trie {

    private final TrieNode root = new TrieNode();

    public void insert(String query, long popularity) {
        TrieNode current = root;
        for (char c : query.toCharArray()) {
            current = current.children.computeIfAbsent(c, k -> new TrieNode());
        }
        current.isEndOfQuery = true;
        current.popularityIfEndOfQuery = popularity;
    }

    /** The NAIVE version -- correct, and exactly what §15 shows is too slow at real scale. */
    public List<Suggestion> searchNaive(String prefix, int k) {
        TrieNode current = root;
        for (char c : prefix.toCharArray()) {
            current = current.children.get(c);
            if (current == null) return List.of(); // no stored query has this prefix at all
        }
        List<Suggestion> allCompletions = new ArrayList<>();
        collectAllCompletions(current, prefix, allCompletions); // walks the ENTIRE subtree -- §15's problem
        return allCompletions.stream()
                .sorted(Comparator.comparingLong(Suggestion::popularity).reversed())
                .limit(k)
                .toList();
    }

    private void collectAllCompletions(TrieNode node, String prefixSoFar, List<Suggestion> out) {
        if (node.isEndOfQuery) out.add(new Suggestion(prefixSoFar, node.popularityIfEndOfQuery));
        for (Map.Entry<Character, TrieNode> entry : node.children.entrySet()) {
            collectAllCompletions(entry.getValue(), prefixSoFar + entry.getKey(), out);
        }
    }
}
```

`insert` is genuinely `O(query length)` — inserting is never the problem. `searchNaive`, however, is `O(size of the entire subtree under this prefix)`: for a short, common prefix like `"a"`, that subtree can contain millions of stored queries, every one of which gets visited, collected, and sorted **on every single request** — the exact cost §15 names precisely.

---

# 15. Follow-up Question 3 — "A Naive Trie Search Still Has to Scan Every Completion Under a Prefix to Rank Them. How Do You Make That Fast?"

> **Interviewer:** *"§14's `searchNaive` walks the entire matching subtree and sorts it, every single request. For a one-character prefix with millions of matches, that's a real, unacceptable cost per keystroke. Fix it."*

The fix doesn't touch the search path at all — it moves the entire cost to **insert time**, where it's paid once, rarely, instead of on every read.

---

# 16. The Real Optimization: Precomputed Top-K at Every Node

Instead of computing a ranked list at query time, **cache the already-sorted top-K list directly on every node**, computed incrementally as data is inserted. A search for a prefix then becomes: walk to the node (`O(prefix length)`, unchanged), and simply **read** that node's already-sorted `topKSuggestions` field — no subtree traversal, no sorting, at query time, ever. The cost this defers is real, not eliminated — every node along an inserted query's path now has to potentially update its cached top-K list — but it's paid exactly once, at insert time, which (per §18's read:write ratio argument) happens at a tiny fraction of the rate reads do.

---

# 17. Implementing Top-K Propagation on Insert

```java
public final class Trie {

    private final TrieNode root = new TrieNode();
    private final int topK;

    public Trie(int topK) { this.topK = topK; }

    public void insert(String query, long popularity) {
        TrieNode current = root;
        Suggestion suggestion = new Suggestion(query, popularity);
        updateTopK(current, suggestion); // the ROOT's own top-K covers every query with an EMPTY prefix -- i.e. everything

        for (char c : query.toCharArray()) {
            current = current.children.computeIfAbsent(c, k -> new TrieNode());
            updateTopK(current, suggestion); // every ancestor along the path is a valid PREFIX of this query
        }
        current.isEndOfQuery = true;
        current.popularityIfEndOfQuery = popularity;
    }

    /** Inserts `candidate` into this node's cached top-K list if it belongs there -- O(topK), never O(subtree size). */
    private void updateTopK(TrieNode node, Suggestion candidate) {
        List<Suggestion> updated = new ArrayList<>(node.topKSuggestions);
        updated.removeIf(s -> s.query().equals(candidate.query())); // a re-inserted (updated-popularity) query replaces its old entry
        updated.add(candidate);
        updated.sort(Comparator.comparingLong(Suggestion::popularity).reversed());
        node.topKSuggestions = updated.size() > topK ? updated.subList(0, topK) : updated;
    }

    /** The FAST version -- a pure read, no traversal beyond the prefix itself, no sorting. */
    public List<Suggestion> search(String prefix) {
        TrieNode current = root;
        for (char c : prefix.toCharArray()) {
            current = current.children.get(c);
            if (current == null) return List.of();
        }
        return current.topKSuggestions; // ALREADY sorted, ALREADY limited to topK -- this IS the entire query cost
    }
}
```

Every character of `query` walked during `insert` calls `updateTopK` on that node — because every prefix of `query` (empty string, first character, first two characters, ..., the full query) is exactly the set of nodes whose cached list `query` might now belong in. `updateTopK` itself is bounded by `topK` (typically 5-10), never by how many total queries share that prefix, which is what keeps insert cost proportional to `query length x topK` — small and constant-ish, regardless of how popular or rare the query turns out to be. This one method is the entire mechanism that turns §14's per-request scan-and-sort into §17's per-request constant-time read.

---

# 18. Follow-up Question 4 — "Where Does the Frequency Data Actually Come From, and How Often Does the Trie Update?"

> **Interviewer:** *"§17's `insert` needs a popularity number for every query. Where does that number come from in a real system, and does the Trie update the instant someone searches something new?"*

§19 answers with the same read:write asymmetry this project's own URL-shortening guide already leans on, applied here with a twist: the "write" isn't just infrequent relative to reads — it's explicitly allowed to be *stale*.

---

# 19. Read:Write Ratio, Again: Why the Trie Is Batch-Rebuilt, Not Updated Per Keystroke

Every real search a user performs is a raw signal — one more data point toward "this query is popular." Feeding every single one of those signals into `insert` synchronously, live, would mean every search anywhere in the world triggers an update walking potentially dozens of Trie nodes' top-K lists, an enormous write amplification for a structure that's read on every keystroke of every *other* user, constantly. The actual, industry-standard answer: **aggregate raw query logs continuously, but only rebuild the served Trie periodically** — every few minutes to every few hours, depending on how quickly "trending" needs to surface — accepting bounded staleness (§8) explicitly, in exchange for making "write" mean "run one batch job occasionally," not "update a live data structure on every search, everywhere."

---

# 20. Implementing the Offline Trie-Build Pipeline

```java
public record QueryLogEntry(String query, Instant searchedAt) { }
```

```java
public final class TrieBuildPipeline {

    private final int topK;

    public TrieBuildPipeline(int topK) { this.topK = topK; }

    /** Runs periodically (e.g. hourly), over a batch of raw log entries accumulated since the last run. */
    public Trie buildFrom(List<QueryLogEntry> recentLogEntries, Map<String, Long> priorAggregatedCounts) {
        Map<String, Long> updatedCounts = new HashMap<>(priorAggregatedCounts);
        for (QueryLogEntry entry : recentLogEntries) {
            updatedCounts.merge(entry.query(), 1L, Long::sum); // a real pipeline aggregates by exact query text first
        }

        Trie newTrie = new Trie(topK); // a BRAND NEW Trie -- built fresh, then swapped in atomically, §20's own note below
        updatedCounts.forEach(newTrie::insert);
        return newTrie;
    }
}
```

Building a **brand-new** Trie from the full, updated counts — rather than mutating a live, currently-serving Trie in place — is deliberate: the newly-built Trie is validated and warmed up completely offline, then published and **atomically swapped in** as the new "current" version once ready, the identical "build fresh, then swap the pointer" discipline this project's own database-internals guides already use for a page's own atomic route-table swap. A search request in flight during a rebuild is never served a half-updated, inconsistent structure — it's served either the complete old version or the complete new one, never something in between.

---

# 21. Follow-up Question 5 — "A Trie Holding Billions of Distinct Queries Doesn't Fit on One Machine. How Do You Shard It?"

> **Interviewer:** *"Every other system in this series, you'd reach for hash-based or consistent-hash sharding by now. Does that work here?"*

No — and explaining precisely why not is one of the most instructive moments this entire question offers.

---

# 22. Why Hash-Based Sharding Breaks a Prefix Trie

Every hash-based sharding scheme this series has built so far shares one assumption: **the full key is known before routing**. A URL shortener's short code, a cache key, a database row's primary key — every one of them is known in its entirety at lookup time, so hashing it and routing to the owning shard works perfectly. An autocomplete request has **only a prefix** — the user hasn't finished typing yet, by definition. Hashing `"appl"` gives no information at all about which shard holds the *completions* of `"appl"` (`"apple"`, `"application"`, `"applesauce"`), because those completions' hashes bear no relationship whatsoever to the prefix's own hash. A hash-sharded Trie would have to **fan every single request out to every shard**, merge the partial results, and re-rank — turning one cheap, local lookup into a full scatter-gather across the entire cluster, on every keystroke.

---

# 23. Prefix-Range Sharding: The Correct Approach

The fix: shard by **prefix range**, not by hash — a scheme that preserves exactly the locality property a hash destroys. Queries starting `"a"` through `"f"` live on shard 0; `"g"` through `"m"` on shard 1; and so on, with ranges sized (and re-balanced) according to how many actual stored queries fall in each range, not merely by dividing the alphabet evenly (query volume is wildly uneven across starting letters — many more real queries start with common letters than rare ones). A request for **any** prefix now routes to **exactly one shard**, determined directly from the prefix's own leading characters, with no fan-out and no merge step required at all.

---

# 24. Implementing Prefix-Range Shard Routing

```java
public record ShardRange(String startInclusive, String endExclusive, int shardId) { }
```

```java
public final class PrefixRangeShardRouter {

    private final List<ShardRange> orderedRanges; // sorted by startInclusive -- enables a direct binary search

    public PrefixRangeShardRouter(List<ShardRange> orderedRanges) { this.orderedRanges = orderedRanges; }

    public int shardFor(String prefix) {
        // Binary search for the range whose [start, end) interval contains this prefix -- O(log shard count),
        // never a linear scan, and CRUCIALLY never a fan-out to more than one shard.
        int low = 0, high = orderedRanges.size() - 1;
        while (low <= high) {
            int mid = (low + high) / 2;
            ShardRange range = orderedRanges.get(mid);
            if (prefix.compareTo(range.startInclusive()) < 0) {
                high = mid - 1;
            } else if (prefix.compareTo(range.endExclusive()) >= 0) {
                low = mid + 1;
            } else {
                return range.shardId(); // FOUND -- exactly one shard, no matter how short or long the prefix is
            }
        }
        throw new IllegalStateException("No shard range covers prefix: " + prefix);
    }
}
```

Sorting ranges by `startInclusive` and binary-searching them is what keeps routing itself cheap (`O(log(shard count))`) even though the underlying partitioning is uneven and re-balanceable — adding a new shard, or splitting an overloaded range, only ever means inserting or adjusting entries in this one sorted list, never rehashing anything.

---

# 25. Follow-up Question 6 — "How Do You Make the Common Case — a Few Very Popular Prefixes — Even Faster?"

> **Interviewer:** *"Real query traffic is heavily skewed — a handful of short, extremely common prefixes account for a huge share of all requests. Even a fast shard lookup is extra work for those. What do you do?"*

Cache them — the identical hot-key insight this project's own CDN, rate-limiter, and URL-shortening guides all lean on, applied here to prefixes instead of segments, tokens, or short codes.

---

# 26. Caching Hot Prefixes

Query-prefix popularity follows a heavily Zipfian distribution: single- and double-character prefixes (`"a"`, `"th"`, `"ho"`) are requested constantly, by nearly every user, on nearly every search — while the space of all possible longer prefixes is requested comparatively rarely. An in-memory LRU cache sitting directly in front of the shard router turns the overwhelming majority of real-world requests into a pure, local, in-memory lookup, with the underlying shard never even contacted.

---

# 27. Implementing a Prefix Cache

```java
public final class PrefixCache {

    private final int capacity;
    private final LinkedHashMap<String, List<Suggestion>> cache;

    public PrefixCache(int capacity) {
        this.capacity = capacity;
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) { // accessOrder=true -- the entire LRU mechanism
            @Override
            protected boolean removeEldestEntry(Map.Entry<String, List<Suggestion>> eldest) {
                return size() > PrefixCache.this.capacity;
            }
        };
    }

    public synchronized Optional<List<Suggestion>> get(String prefix) {
        return Optional.ofNullable(cache.get(prefix));
    }

    public synchronized void put(String prefix, List<Suggestion> suggestions) {
        cache.put(prefix, suggestions);
    }
}
```

This is the same `LinkedHashMap(accessOrder=true)` zero-extra-code LRU mechanism this project's own database-internals and URL-shortening guides already reuse — applied here for the identical reason: a small, bounded cache in front of a real backend, absorbing the specific, heavily-skewed traffic shape this workload actually has.

---

# 28. Follow-up Question 7 — "Typing Fires a Request Per Keystroke. How Do You Avoid Hammering the Backend?"

> **Interviewer:** *"A five-character query typed quickly fires five separate requests, most of them for prefixes the user has already moved past by the time the response arrives. How do you avoid that waste?"*

**Debounce** on the client — deliberately wait for a brief pause in typing before firing a request at all, and **cancel** any still-in-flight request that a newer keystroke has already made obsolete.

---

# 29. Client-Side Debouncing and Request Cancellation

```java
public final class DebouncedAutocompleteClient {

    private final AutocompleteApi api;
    private final Duration debounceWindow;
    private final ScheduledExecutorService scheduler;
    private ScheduledFuture<?> pendingCall;
    private CompletableFuture<List<Suggestion>> inFlightRequest;

    public DebouncedAutocompleteClient(AutocompleteApi api, Duration debounceWindow, ScheduledExecutorService scheduler) {
        this.api = api;
        this.debounceWindow = debounceWindow;
        this.scheduler = scheduler;
    }

    /** Called on EVERY keystroke -- most calls to this method never actually reach the network at all. */
    public void onKeystroke(String currentPrefix, Consumer<List<Suggestion>> onResult) {
        if (pendingCall != null) pendingCall.cancel(false); // a newer keystroke supersedes whatever was scheduled
        if (inFlightRequest != null) inFlightRequest.cancel(true); // and supersedes whatever was ALREADY in flight

        pendingCall = scheduler.schedule(() -> {
            inFlightRequest = api.suggest(currentPrefix);
            inFlightRequest.thenAccept(onResult);
        }, debounceWindow.toMillis(), TimeUnit.MILLISECONDS);
    }
}
```

Cancelling a still-**in-flight** request, not just a still-**pending** (not-yet-fired) one, matters specifically for a fast typist: without it, an earlier keystroke's slow response could arrive **after** a later keystroke's response, overwriting the correct, current suggestion list with stale results for a prefix the user has already typed past — a real, user-visible bug debouncing alone (without cancellation) does not fix.

---

# 30. Class Diagram: The Trie and Query Path

```text
TrieNode                                          Trie
+ children: Map<Character, TrieNode>              + insert(query, popularity)
+ isEndOfQuery, popularityIfEndOfQuery             + search(prefix): List<Suggestion>
+ topKSuggestions: List<Suggestion> (CACHED, §16)       |
                                                          | routes through
PrefixRangeShardRouter                            PrefixCache
+ shardFor(prefix): shardId                       + get(prefix): Optional<List<Suggestion>>
      ^                                            + put(prefix, suggestions)
      | consulted on a cache MISS
      |
TrieBuildPipeline                                 DebouncedAutocompleteClient
+ buildFrom(logEntries, priorCounts): Trie          + onKeystroke(prefix, onResult)
      | produces a NEW Trie, offline                     | the ONLY entry point a real UI ever calls
      v
Trie (swapped in atomically, §20)
```

Every arrow here traces a real call this guide built code for — a keystroke debounces, hits the cache, falls through to the shard router only on a miss, and the Trie it eventually reaches was built entirely offline, on its own schedule, by a pipeline the read path never waits on.

---

# 31. Follow-up Question 8 — "A Trie Has Real Memory Overhead for Sparse Data. How Do You Make It More Memory-Efficient?"

> **Interviewer:** *"A long, rarely-branching query like `'international shipping regulations'` allocates one full `TrieNode` per character, each with its own `HashMap`, even though most of those nodes have exactly one child. That's a lot of memory for very little branching. What's the fix?"*

§32 names the standard fix: **compress chains of single-child nodes into one edge carrying a substring**, rather than one node per character.

---

# 32. Compressed (Radix) Tries: Collapsing Single-Child Chains

```text
Plain Trie (one node per character):        Compressed (radix) Trie:
i-n-t-e-r-n-a-t-i-o-n-a-l                    "international" (one edge, one node)
                                                      |
                                              (branches only where the data actually branches)
```

A **radix trie** merges any run of nodes that each have exactly one child into a single edge labeled with the entire shared substring, rather than one node per character — a node only stays "split out" on its own where the data genuinely branches (two different queries diverging at that point). This is a real, meaningful memory reduction for exactly the common case named in the follow-up — long, low-branching queries — at the cost of a genuinely more complex insert/split algorithm (an edge sometimes needs to be split mid-substring when a new query diverges partway through an existing compressed edge). Named here as the correct, standard answer to the follow-up, and named honestly as real, additional implementation complexity beyond this guide's own core `Trie` (§14, §17), rather than built out in full (§45).

---

# 33. Follow-up Question 9 — "How Do You Personalize Suggestions Instead of Showing the Same Global Top-K to Everyone?"

> **Interviewer:** *"Two different users type the same prefix. Should they always see the identical suggestion list?"*

Not necessarily — §34 layers personalization **on top of** the global Trie, rather than building a separate per-user structure.

---

# 34. Personalization as a Layered Re-Ranking Step

```java
public final class PersonalizedSuggestionRanker {

    private final Trie globalTrie;                                  // §14, §17 -- UNCHANGED, shared across all users
    private final Function<String, List<Suggestion>> userHistoryLookup; // a user's own recent/frequent queries

    public PersonalizedSuggestionRanker(Trie globalTrie, Function<String, List<Suggestion>> userHistoryLookup) {
        this.globalTrie = globalTrie;
        this.userHistoryLookup = userHistoryLookup;
    }

    public List<Suggestion> suggestFor(String userId, String prefix, int k) {
        List<Suggestion> global = globalTrie.search(prefix);                       // the fast, shared, cached path
        List<Suggestion> personal = userHistoryLookup.apply(userId).stream()
                .filter(s -> s.query().startsWith(prefix))
                .toList();

        // Personal matches are boosted ahead of global ones, but the GLOBAL Trie itself is never touched --
        // personalization is a small, per-request re-ranking step layered on top, not a redesign of storage.
        List<Suggestion> merged = new ArrayList<>(personal);
        global.stream().filter(g -> personal.stream().noneMatch(p -> p.query().equals(g.query()))).forEach(merged::add);
        return merged.stream().limit(k).toList();
    }
}
```

Keeping the shared, cached, offline-built `globalTrie` completely untouched — and layering personalization as a small, per-request merge on top of its already-fast result — is what keeps personalization from requiring a redesign of §16-§20's entire storage and pipeline architecture; a user's own history is a comparatively tiny amount of data, cheap to look up and merge in per request, in a way that re-deriving a full per-user Trie would not be.

---

# 35. Follow-up Question 10 — "What About Typos — Should a Search for 'gogle' Still Suggest 'Google'?"

> **Interviewer:** *"A prefix search for a misspelled query returns nothing, because the Trie has no entry for that exact character sequence. Is that acceptable?"*

Not for a production-quality experience — but fuzzy matching (edit-distance-tolerant lookup) is a genuinely separate algorithmic problem from everything this guide has built so far, and deserves to be named honestly as such rather than bolted on superficially.

---

# 36. Fuzzy Matching and Spell Correction (Named as Real, Separate Work)

A prefix Trie answers "what completions share this **exact** prefix" — a fundamentally different question from "what popular queries are within a small edit distance of what was typed." Real systems solve the fuzzy case with a genuinely different mechanism layered alongside the exact-prefix Trie — commonly a small edit-distance search over a bounded candidate set (near-matches to the typed prefix, gathered via a BK-tree or a Levenshtein-automaton-driven traversal), triggered specifically when the exact-prefix Trie returns few or no results. This guide names it explicitly as real, valuable, separate work (§45) rather than folding a half-built version of it into the core Trie design and calling the result complete.

---

# 37. Follow-up Question 11 — "At Global Scale, What Breaks First?"

> **Interviewer:** *"Hundreds of millions of users, globally distributed, typing constantly. What's the actual bottleneck once everything in this guide is already built?"*

Physical distance to the nearest shard, exactly the same bottleneck this project's own video-streaming and file-storage guides already name for a different kind of content — and the fix is the identical one.

---

# 38. Scaling the Read Path: Replicated, Cached Shards Behind a CDN

Because a served Trie shard is **read-only** between rebuilds (§20's atomic swap), it can be freely **replicated** across every region that needs it, and **cached at the edge**, exactly like a static asset — a user in Tokyo should never wait on a round trip to a shard physically hosted in Virginia for a lookup that's identical for every user asking the same popular prefix. This is only safe *because* the underlying data is read-mostly and explicitly tolerant of bounded staleness (§19) — a design that needed real-time-fresh writes could never replicate this cheaply, which is precisely why the earlier decision to batch-rebuild rather than update live pays off again here, at a completely different layer of the system.

---

# 39. Full Worked Example: A User Types, End to End

```text
1.  User types "g"
2.  DebouncedAutocompleteClient.onKeystroke("g", ...) -- schedules a call, does NOT fire immediately            (§29)
3.  User types "o" within the debounce window -- the PENDING call for "g" is cancelled, a new one scheduled for "go"
4.  User pauses -- the scheduled call for "go" finally fires
5.  PrefixCache.get("go") -> HIT (an extremely common two-letter prefix)                                         (§27)
6.  Suggestions returned INSTANTLY -- the shard was never even contacted

7.  (later) User types a rare, specific prefix: "goph"
8.  PrefixCache.get("goph") -> miss                                                                                (§27)
9.  PrefixRangeShardRouter.shardFor("goph") -> shard 1 (owns "g".."m")                                              (§24)
10. Trie.search("goph") -> walks 4 nodes -> reads the CACHED topKSuggestions at that node -- no scan, no sort        (§17)
11. PrefixCache.put("goph", result) -- warms the cache for the next request with this same prefix                    (§27)
```

Steps 5 and 10 are the entire payoff of this guide's design: a hot prefix never leaves the local cache at all, and even a genuine cache miss resolves via a constant-time read of an already-ranked list — never the expensive subtree scan §14's naive version would have required.

---

# 40. Final Architecture Diagram

```text
                         Query logs (continuous)  -->  TrieBuildPipeline (§20, periodic/offline)
                                                                |
                                              New Trie shards, atomically swapped in
                                                                |
                    Region: US-East                                          Region: AP-South
              +------------------------+                              +------------------------+
              | Replicated Trie shards  |                              | Replicated Trie shards  |
              | (read-only, §38)         |                              | (read-only, §38)         |
              +-----------+------------+                              +-----------+------------+
                          |                                                        |
              PrefixRangeShardRouter (§24)                             PrefixRangeShardRouter (§24)
                          |                                                        |
                    PrefixCache (§27)                                        PrefixCache (§27)
                          ^                                                        ^
                          |                                                        |
              DebouncedAutocompleteClient (§29)                        DebouncedAutocompleteClient (§29)
```

---

# 41. Design Patterns Used Throughout This Guide

| Pattern | Where | Why |
|---|---|---|
| **Memoization / Precomputed Cache** | `TrieNode.topKSuggestions` (§16-§17) | The single optimization this entire guide is built around — moving a per-request cost to a rarely-paid insert-time cost. |
| **Facade** | `Trie.search(prefix)` (§17) | The bit-packing, top-K maintenance, and traversal are all invisible to the caller behind one method. |
| **Strategy (deployment-level)** | Global ranking (§17) vs. personalized re-ranking (§34) | Two interchangeable ranking strategies layered on the identical underlying Trie, never requiring two different storage designs. |
| **Range Partitioning** | `PrefixRangeShardRouter` (§23-§24) | The specific sharding strategy that preserves the locality this workload's queries actually need, unlike a hash. |
| **LRU Cache** | `PrefixCache` (§27) | The zero-extra-code `LinkedHashMap(accessOrder=true)` trick this project's own database-internals and URL-shortening guides already use for the identical purpose. |

---

# 42. SOLID Principles Applied

- **Single Responsibility**: `TrieBuildPipeline` only aggregates logs and builds a Trie; `PrefixRangeShardRouter` only routes; `PrefixCache` only caches. None of them know how to rank suggestions or debounce a keystroke.
- **Open/Closed**: adding personalization (§34) or fuzzy matching (§36) requires zero changes to `Trie` or `TrieNode` — both are layered on top, consuming the existing `search` method unchanged.
- **Liskov Substitution**: any ranking strategy consuming `Trie.search(prefix)`'s output is fully substitutable — `PersonalizedSuggestionRanker` (§34) and a plain, unranked pass-through are interchangeable from the client's point of view.
- **Interface Segregation**: `Trie` exposes exactly `insert`/`search` — no sharding detail, no caching detail, no personalization detail leaks into its surface.
- **Dependency Inversion**: `PersonalizedSuggestionRanker` (§34) depends on a `Function<String, List<Suggestion>>` for user history — an abstraction, never a concrete user-history store implementation directly.

---

# 43. Common Mistakes When Building This Yourself

- **Ranking at query time instead of precomputing top-K at insert time** (§14-§15) — correct, and catastrophically slow the moment a prefix has millions of completions, which common, short prefixes always do.
- **Updating the live Trie synchronously on every search** (§18-§19) — reintroduces an enormous write amplification for a structure that's read far more often than the underlying popularity data actually needs to change.
- **Sharding a prefix Trie by hash** (§22) — destroys prefix locality entirely, forcing a full cluster fan-out and merge on every single request, the opposite of what sharding is supposed to buy.
- **Debouncing without cancelling in-flight requests** (§29) — a fast typist can still see a stale, superseded response arrive *after* a newer one, overwriting the correct current suggestions.
- **Building a separate, full Trie per user for personalization** (§33-§34) — enormously more expensive than necessary; a small per-request re-ranking layer over one shared, global Trie captures nearly all the value at a fraction of the storage and build cost.
- **Bolting a half-built fuzzy-match feature onto the exact-prefix Trie** (§35-§36) — conflates two genuinely different algorithmic problems into one data structure neither is well-suited for on its own.

---

# 44. Testing Strategy

- **`Trie`** (§14, §17): inserting several queries sharing a prefix and searching that prefix returns exactly the correct top-K, sorted by popularity, and re-inserting an existing query with a higher popularity correctly re-ranks it.
- **`TopKPropagationTest`**: inserting a query updates the cached `topKSuggestions` list on **every** ancestor node along its path, not just the final node — the test that directly validates §17's core mechanism.
- **`PrefixRangeShardRouter`** (§24): every prefix in a representative sample routes to exactly one shard, consistently, across repeated calls; a prefix at the exact boundary between two ranges routes correctly to the range that actually contains it.
- **`PrefixCache`** (§27): a cache at capacity evicts the least-recently-used prefix, not an arbitrary one, verified by accessing an existing entry and confirming it survives longer than an unused one.
- **`DebouncedAutocompleteClient`** (§29): rapid, successive keystrokes result in exactly one network call, for the final prefix only; a slow, superseded in-flight response is confirmed cancelled and never delivered to the caller.
- **`TrieBuildPipeline`** (§20): a rebuild produces a new Trie whose top-K results correctly reflect the updated aggregated counts, and the previously-serving Trie remains fully intact and correct until the swap completes.

---

# 45. Suggested Future Enhancements

- **A real compressed (radix) Trie implementation** (§32) — the memory-efficiency refinement named but not built out in full here, genuinely valuable at billions of stored queries.
- **A real fuzzy-matching layer** (§35-§36) — a BK-tree or Levenshtein-automaton-based near-match search, triggered specifically when the exact-prefix Trie returns few or no results.
- **Trending/velocity-aware ranking** — boosting a query that's suddenly spiking in frequency ahead of one with a higher all-time total but flat recent trend, rather than ranking purely by cumulative historical popularity.
- **A/B-testable ranking strategies** — since global ranking and personalized re-ranking are already Strategy-shaped (§34), running two ranking approaches against different user cohorts and comparing engagement is a natural next step.
- **Multi-language/tokenization-aware prefixes** — this guide's Trie operates on raw characters, which works cleanly for space-delimited Latin-script queries but needs real, separate handling for languages without clear word-boundary characters.

---

# 46. Progressive Interview Question Set

1. Walk through exactly why a naive Trie search, correct as it is, becomes too slow at real scale — name the specific cost, not just "it's slow."
2. Explain precisely how inserting one query updates potentially many different nodes' cached top-K lists, and why that update is still cheap.
3. Why is the underlying popularity data batch-rebuilt rather than updated synchronously on every search, and what specific guarantee does bounded staleness buy in exchange?
4. Explain exactly why hash-based sharding, the default answer almost everywhere else, actively breaks a prefix Trie — what specific piece of information is missing at request time?
5. Why does prefix-range sharding preserve exactly the property a hash destroys?
6. Why does debouncing alone not fully solve the "stale response" problem, and what does cancellation add on top of it?
7. Why is personalization implemented as a re-ranking layer over one shared Trie, rather than a separate Trie built per user?
8. What's the real memory cost a compressed (radix) Trie is solving, and what specific case (long, low-branching queries) does it help most?
9. Why is fuzzy matching treated as a genuinely separate mechanism rather than an extension of the exact-prefix Trie?
10. If asked to support "trending now" queries that should rank higher than their raw historical popularity alone would suggest, how would you extend this design without redesigning the core Trie?

---

# 47. Final Takeaway

Every genuinely hard piece of this design traces back to one realization: ranking cannot be computed at query time and still be fast, so it has to be computed **somewhere else** — at insert time, cached directly on the data structure itself, paid for once by a rare write instead of repeatedly by a frequent read. Sharding by prefix range instead of by hash is the same insight wearing a different hat: preserve the locality the actual access pattern needs, rather than reaching for the sharding strategy that happens to work everywhere else. A search-autocomplete system that gets both of these right ends up remarkably simple at the read path — a tree walk and a list read, nothing more — which is exactly the point: the complexity this design has to manage lives entirely in how the data gets *there*, never in how it's read once it has.
