# Iterator Design Pattern — Complete Step-by-Step Interview Guide

> A progressive Java interview guide that starts with a simple `for` loop and gradually evolves into custom iterators, iterator composition, lazy evaluation, infinite iterators, and senior-level design questions.

---

# Table of Contents

1. [How to Use This Guide](#1-how-to-use-this-guide)
2. [Start With a Simple for Loop](#2-start-with-a-simple-for-loop)
3. [Why Do We Need an Iterator?](#3-why-do-we-need-an-iterator)
4. [What Happens Inside Enhanced for?](#4-what-happens-inside-enhanced-for)
5. [Iterable vs Iterator](#5-iterable-vs-iterator)
6. [Basic Iterator Pattern](#6-basic-iterator-pattern)
7. [Implement a Basic Iterator](#7-implement-a-basic-iterator)
8. [Iterator Contract](#8-iterator-contract)
9. [OddIterator](#9-odditerator)
10. [EvenIterator](#10-eveniterator)
11. [NegativeIterator](#11-negativeiterator)
12. [PositiveIterator](#12-positiveiterator)
13. [Iterator Depending on Another Iterator](#13-iterator-depending-on-another-iterator)
14. [Generic FilteringIterator](#14-generic-filteringiterator)
15. [ForwardIterator](#15-forwarditerator)
16. [BackwardIterator](#16-backwarditerator)
17. [RangeIterator](#17-rangeiterator)
18. [SkipIterator](#18-skipiterator)
19. [LimitIterator](#19-limititerator)
20. [ZigZagIterator](#20-zigzagiterator)
21. [InterleavingIterator](#21-interleavingiterator)
22. [RoundRobinIterator](#22-roundrobiniterator)
23. [ConcatIterator](#23-concatiterator)
24. [FlattenIterator](#24-flatteniterator)
25. [MapIterator](#25-mapiterator)
26. [PeekIterator](#26-peekiterator)
27. [DistinctIterator](#27-distinctiterator)
28. [TakeWhileIterator](#28-takewhileiterator)
29. [DropWhileIterator](#29-dropwhileiterator)
30. [ZipIterator](#30-zipiterator)
31. [PartitionIterator](#31-partitioniterator)
32. [CycleIterator](#32-cycleiterator)
33. [CycleIterator Without Caching](#33-cycleiterator-without-caching)
34. [Lazy Evaluation](#34-lazy-evaluation)
35. [Iterator Pipeline](#35-iterator-pipeline)
36. [Operation Ordering](#36-operation-ordering)
37. [Iterator State Machine](#37-iterator-state-machine)
38. [Multiple hasNext() Calls](#38-multiple-hasnext-calls)
39. [Calling next() Without hasNext()](#39-calling-next-without-hasnext)
40. [Null Elements](#40-null-elements)
41. [Empty Source](#41-empty-source)
42. [remove()](#42-remove)
43. [Fail-Fast Iterator](#43-fail-fast-iterator)
44. [Concurrent Modification](#44-concurrent-modification)
45. [Thread Safety](#45-thread-safety)
46. [Infinite Iterators](#46-infinite-iterators)
47. [Complexity Analysis](#47-complexity-analysis)
48. [Which Iterators Need Caching?](#48-which-iterators-need-caching)
49. [Iterator Composition](#49-iterator-composition)
50. [SOLID Design](#50-solid-design)
51. [Iterator vs Stream](#51-iterator-vs-stream)
52. [Common Interview Mistakes](#52-common-interview-mistakes)
53. [Progressive Interview Question Set](#53-progressive-interview-question-set)
54. [Final Senior-Level Challenge](#54-final-senior-level-challenge)
55. [Final Revision Checklist](#55-final-revision-checklist)

---

# 1. How to Use This Guide

The recommended interview progression is:

```text
Simple for loop
      |
      v
Why Iterator?
      |
      v
Iterable vs Iterator
      |
      v
Basic Iterator
      |
      v
Filtering Iterator
      |
      v
Iterator wrapping Iterator
      |
      v
Traversal Strategies
      |
      v
Iterator Composition
      |
      v
Lazy Evaluation
      |
      v
Cycle / Infinite Iterator
      |
      v
Caching / Random Access
      |
      v
Concurrency / Fail-Fast
      |
      v
Senior-Level Design
```

The objective is not to memorize implementations.

The objective is to understand:

> **What traversal problem are we solving, what state is required, and what capabilities does the source provide?**

---

# 2. Start With a Simple `for` Loop

## Interview Question 1

Given:

```java
List<Integer> numbers =
        List.of(10, 20, 30, 40, 50);
```

How would you print all values?

### Expected Answer

```java
for (Integer number : numbers) {
    System.out.println(number);
}
```

Now the interviewer asks:

> What is happening internally?

This is where the Iterator discussion begins.

---

# 3. Why Do We Need an Iterator?

Suppose we have:

```java
class UserCollection {

    private User[] users;

}
```

One simple design would expose:

```java
public User[] getUsers() {
    return users;
}
```

The client can then do:

```java
User[] users = collection.getUsers();

for (User user : users) {
    // ...
}
```

Ask:

> What problems do you see?

## Problem 1 — Internal Representation Leaks

The client knows the collection is implemented using:

```text
User[]
```

If we change it to:

```text
List<User>
```

the client may need to change.

---

## Problem 2 — Tight Coupling

The client is coupled to the storage structure.

---

## Problem 3 — Traversal Logic Is Exposed

The collection is responsible for storing data, while the client is responsible for understanding how to traverse it.

---

## Problem 4 — Different Traversal Strategies

Suppose we want:

```text
Forward
Backward
Odd only
Even only
Negative only
ZigZag
Limited
Filtered
```

We don't want the collection API to expose every possible traversal implementation.

---

# 4. What Happens Inside Enhanced `for`?

Conceptually:

```java
for (Integer number : numbers) {
    System.out.println(number);
}
```

works like:

```java
Iterator<Integer> iterator =
        numbers.iterator();

while (iterator.hasNext()) {

    Integer number =
            iterator.next();

    System.out.println(number);
}
```

The important transition is:

```text
for-each
   |
   v
Iterable
   |
   | iterator()
   v
Iterator
   |
   +---- hasNext()
   |
   +---- next()
```

Now we can ask:

> What exactly is an Iterator?

---

# 5. Iterable vs Iterator

This is one of the most important interview concepts.

## Iterator

An `Iterator<T>` represents:

> One traversal over a source.

Example:

```java
Iterator<Integer> iterator =
        numbers.iterator();
```

The iterator owns traversal state.

For example:

```text
current index = 5
```

---

## Iterable

An `Iterable<T>` represents:

> Something from which an iterator can be obtained.

Conceptually:

```java
interface Iterable<T> {

    Iterator<T> iterator();
}
```

Therefore:

```text
Iterable
    |
    | iterator()
    v
Iterator
```

---

## Important Difference

An `Iterable` can usually produce multiple independent iterators:

```java
Iterator<Integer> a =
        collection.iterator();

Iterator<Integer> b =
        collection.iterator();
```

Each can have independent traversal state.

---

# 6. Basic Iterator Pattern

The Iterator pattern separates:

```text
Collection
```

from:

```text
Traversal
```

Conceptually:

```text
+----------------------+
|      Collection      |
|----------------------|
| Internal Data        |
+----------+-----------+
           |
           | creates
           v
+----------------------+
|       Iterator       |
|----------------------|
| Traversal State      |
| hasNext()            |
| next()               |
+----------------------+
```

The client does not need to know whether the underlying structure is:

```text
Array
List
LinkedList
Tree
Database Cursor
Remote Data Source
```

It only depends on the iterator contract.

---

# 7. Implement a Basic Iterator

## Interview Question 2

Given:

```java
class UserCollection {

    private final User[] users;

    public UserCollection(User[] users) {
        this.users = users;
    }
}
```

Design:

```java
Iterator<User> iterator =
        collection.iterator();

while (iterator.hasNext()) {

    User user =
            iterator.next();
}
```

---

## Step 1 — Iterator Interface

For learning purposes:

```java
interface MyIterator<T> {

    boolean hasNext();

    T next();
}
```

---

## Step 2 — Iterator Implementation

```java
class UserIterator
        implements MyIterator<User> {

    private final User[] users;

    private int index;

    public UserIterator(User[] users) {
        this.users = users;
    }

    @Override
    public boolean hasNext() {
        return index < users.length;
    }

    @Override
    public User next() {

        if (!hasNext()) {
            throw new NoSuchElementException();
        }

        return users[index++];
    }
}
```

---

## Step 3 — Collection

```java
class UserCollection {

    private final User[] users;

    public UserCollection(User[] users) {
        this.users = users;
    }

    public MyIterator<User> iterator() {
        return new UserIterator(users);
    }
}
```

---

## Step 4 — Client

```java
MyIterator<User> iterator =
        collection.iterator();

while (iterator.hasNext()) {

    User user =
            iterator.next();

    System.out.println(user);
}
```

---

# 8. Iterator Contract

The basic contract is:

```text
hasNext()
    |
    +---- true
    |       |
    |       v
    |     next()
    |
    +---- false
            |
            v
        iteration ends
```

A Java-style iterator normally behaves like:

```java
if (!hasNext()) {
    throw new NoSuchElementException();
}
```

---

# 9. OddIterator

## Interview Question 3

Input:

```text
1 2 3 4 5 6 7 8 9 10
```

Design:

```text
OddIterator
```

Expected:

```text
1 3 5 7 9
```

---

## Key Requirement

Do not do:

```java
List<Integer> oddNumbers = new ArrayList<>();
```

and populate it first.

We want:

```text
Lazy traversal
```

---

## Architecture

```text
Original Iterator
       |
       v
OddIterator
       |
       v
1 3 5 7 9
```

---

## Implementation

```java
class OddIterator
        implements MyIterator<Integer> {

    private final MyIterator<Integer> source;

    private Integer nextValue;

    private boolean prepared;

    public OddIterator(
            MyIterator<Integer> source) {

        this.source = source;
    }

    private void prepare() {

        if (prepared) {
            return;
        }

        while (source.hasNext()) {

            Integer value =
                    source.next();

            if (value % 2 != 0) {

                nextValue = value;
                prepared = true;

                return;
            }
        }

        prepared = true;
        nextValue = null;
    }

    @Override
    public boolean hasNext() {

        prepare();

        return nextValue != null;
    }

    @Override
    public Integer next() {

        prepare();

        if (nextValue == null) {
            throw new NoSuchElementException();
        }

        Integer result = nextValue;

        nextValue = null;
        prepared = false;

        return result;
    }
}
```

---

# 10. EvenIterator

## Interview Question 4

Input:

```text
1 2 3 4 5 6 7 8
```

Expected:

```text
2 4 6 8
```

Condition:

```java
value % 2 == 0
```

Architecture:

```text
Source Iterator
      |
      v
EvenIterator
      |
      v
2 4 6 8
```

---

# 11. NegativeIterator

## Interview Question 5

Input:

```text
10 -5 3 -8 -2 7
```

Expected:

```text
-5 -8 -2
```

Condition:

```java
value < 0
```

The important point is that the iterator should consume values from the source lazily.

---

# 12. PositiveIterator

Input:

```text
-10 5 -3 8 -2
```

Expected:

```text
5 8
```

Condition:

```java
value > 0
```

---

# 13. Iterator Depending on Another Iterator

This is an important interview progression.

## Interview Question 6

Input:

```text
1 2 3 4 5 6 7 8 9 10 11
```

Apply:

```text
OddIterator
```

Result:

```text
1 3 5 7 9 11
```

Now apply another:

```text
OddIterator
```

Result:

```text
1 5 9
```

Architecture:

```text
Original Iterator
       |
       v
OddIterator #1
       |
       v
OddIterator #2
       |
       v
Client
```

The second iterator does not need to know where the first iterator gets its data.

It only knows:

```text
Iterator<T>
```

This leads to an important design principle:

> **An iterator can wrap another iterator.**

---

# 14. Generic FilteringIterator

At this point ask:

> Are OddIterator, EvenIterator, NegativeIterator and PositiveIterator duplicating the same algorithm?

Yes.

The only thing changing is the condition.

Therefore create:

```text
FilteringIterator<T>
```

---

## Generic Design

```java
class FilteringIterator<T>
        implements MyIterator<T> {

    private final MyIterator<T> source;

    private final Predicate<T> predicate;

    private T nextValue;

    private boolean prepared;

    public FilteringIterator(
            MyIterator<T> source,
            Predicate<T> predicate) {

        this.source = source;
        this.predicate = predicate;
    }

    private void prepare() {

        if (prepared) {
            return;
        }

        while (source.hasNext()) {

            T candidate =
                    source.next();

            if (predicate.test(candidate)) {

                nextValue = candidate;

                prepared = true;

                return;
            }
        }

        nextValue = null;

        prepared = true;
    }

    @Override
    public boolean hasNext() {

        prepare();

        return nextValue != null;
    }

    @Override
    public T next() {

        prepare();

        if (nextValue == null) {
            throw new NoSuchElementException();
        }

        T result = nextValue;

        nextValue = null;
        prepared = false;

        return result;
    }
}
```

---

## Odd

```java
new FilteringIterator<>(
        iterator,
        value -> value % 2 != 0
);
```

---

## Even

```java
new FilteringIterator<>(
        iterator,
        value -> value % 2 == 0
);
```

---

## Negative

```java
new FilteringIterator<>(
        iterator,
        value -> value < 0
);
```

---

# 15. ForwardIterator

## Interview Question 7

Input:

```text
1 2 3 4 5
```

Output:

```text
1 2 3 4 5
```

State:

```java
private int index;
```

Algorithm:

```text
next():
    return data[index]
    index++
```

Space:

```text
O(1)
```

---

# 16. BackwardIterator

## Interview Question 8

Input:

```text
1 2 3 4 5
```

Expected:

```text
5 4 3 2 1
```

For a `List<T>`:

```java
class BackwardIterator<T>
        implements MyIterator<T> {

    private final List<T> list;

    private int index;

    public BackwardIterator(List<T> list) {

        this.list = list;

        this.index =
                list.size() - 1;
    }

    @Override
    public boolean hasNext() {
        return index >= 0;
    }

    @Override
    public T next() {

        if (!hasNext()) {
            throw new NoSuchElementException();
        }

        return list.get(index--);
    }
}
```

---

# 17. Can BackwardIterator Work With `Iterator<T>`?

Ask:

> Can you implement BackwardIterator if the only input is:

```java
Iterator<T>
```

Generally:

```text
No.
```

Why?

A normal iterator provides:

```text
hasNext()
next()
```

It does not provide:

```text
previous()
```

or:

```text
get(index)
```

Therefore backward traversal requires one of:

```text
Random-access collection
        OR
Bidirectional iterator
        OR
Cached data
```

This is a very important interview concept:

> **The iterator algorithm is constrained by the capabilities of the source.**

---

# 18. RangeIterator

## Interview Question 9

Design:

```java
RangeIterator(5, 10)
```

Expected:

```text
5 6 7 8 9 10
```

---

## Implementation

```java
class RangeIterator
        implements MyIterator<Integer> {

    private int current;

    private final int end;

    public RangeIterator(
            int start,
            int end) {

        this.current = start;
        this.end = end;
    }

    @Override
    public boolean hasNext() {
        return current <= end;
    }

    @Override
    public Integer next() {

        if (!hasNext()) {
            throw new NoSuchElementException();
        }

        return current++;
    }
}
```

---

# 19. RangeIterator With Step

Follow-up:

```java
RangeIterator(0, 10, 2)
```

Output:

```text
0 2 4 6 8 10
```

Reverse:

```java
RangeIterator(10, 0, -2)
```

Output:

```text
10 8 6 4 2 0
```

---

## Interview Follow-ups

Ask:

1. What if step is zero?
2. What if start > end?
3. Should the end be inclusive?
4. Should negative steps be allowed?
5. What exception should invalid arguments produce?
6. Can this iterator be infinite?

---

# 20. SkipIterator

## Interview Question 10

Input:

```text
1 2 3 4 5 6 7
```

Skip first three.

Expected:

```text
4 5 6 7
```

Architecture:

```text
Source
  |
  v
SkipIterator(3)
  |
  v
4 5 6 7
```

---

## Implementation

```java
class SkipIterator<T>
        implements MyIterator<T> {

    private final MyIterator<T> source;

    public SkipIterator(
            MyIterator<T> source,
            int skip) {

        this.source = source;

        for (
            int i = 0;
            i < skip && source.hasNext();
            i++
        ) {
            source.next();
        }
    }

    @Override
    public boolean hasNext() {
        return source.hasNext();
    }

    @Override
    public T next() {

        if (!hasNext()) {
            throw new NoSuchElementException();
        }

        return source.next();
    }
}
```

---

# 21. LimitIterator

## Interview Question 11

Input:

```text
1 2 3 4 5 6 7 8
```

Limit:

```text
3
```

Expected:

```text
1 2 3
```

---

## Implementation

```java
class LimitIterator<T>
        implements MyIterator<T> {

    private final MyIterator<T> source;

    private final int limit;

    private int count;

    public LimitIterator(
            MyIterator<T> source,
            int limit) {

        this.source = source;
        this.limit = limit;
    }

    @Override
    public boolean hasNext() {

        return count < limit
                && source.hasNext();
    }

    @Override
    public T next() {

        if (!hasNext()) {
            throw new NoSuchElementException();
        }

        count++;

        return source.next();
    }
}
```

Space:

```text
O(1)
```

No caching is required.

---

# 22. ZigZagIterator

## Interview Question 12

Input:

```text
1 2 3 4 5 6 7
```

Expected:

```text
1 7 2 6 3 5 4
```

---

## State

```text
left  = 0
right = 6
```

Alternate:

```text
left
right
left
right
...
```

---

## Implementation

```java
class ZigZagIterator<T>
        implements MyIterator<T> {

    private final List<T> list;

    private int left = 0;

    private int right;

    private boolean takeLeft = true;

    public ZigZagIterator(List<T> list) {

        this.list = list;

        this.right =
                list.size() - 1;
    }

    @Override
    public boolean hasNext() {
        return left <= right;
    }

    @Override
    public T next() {

        if (!hasNext()) {
            throw new NoSuchElementException();
        }

        if (takeLeft) {

            takeLeft = false;

            return list.get(left++);
        }

        takeLeft = true;

        return list.get(right--);
    }
}
```

Space:

```text
O(1)
```

Requirement:

```text
Random access
```

for this implementation.

---

# 23. InterleavingIterator

## Interview Question 13

Given:

```text
A = 1 2 3 4

B = 10 20 30 40
```

Output:

```text
1 10 2 20 3 30 4 40
```

Architecture:

```text
Iterator A -----\
                  \
                   InterleavingIterator
                  /
Iterator B -----/
```

---

## Basic State

```java
private boolean useFirst;
```

Algorithm:

```text
if both have values:
    alternate

if A is exhausted:
    continue B

if B is exhausted:
    continue A
```

---

# 24. Interleaving With Unequal Sources

Input:

```text
A = 1 2

B = 10 20 30 40
```

Expected:

```text
1 10 2 20 30 40
```

The implementation must handle exhaustion of one child iterator.

---

# 25. RoundRobinIterator

Now generalize the problem.

## Interview Question 14

Input:

```text
A = 1 2 3

B = 10 20

C = 100 200 300
```

Expected:

```text
1 10 100 2 20 200 3 300
```

Use:

```java
Queue<Iterator<T>>
```

Conceptually:

```text
Queue

[A, B, C]

Take A
Queue -> [B, C, A]

Take B
Queue -> [C, A, B]

Take C
Queue -> [A, B, C]
```

If an iterator becomes exhausted, do not put it back into the queue.

---

# 26. ConcatIterator

## Interview Question 15

Input:

```text
A = 1 2 3

B = 10 20
```

Output:

```text
1 2 3 10 20
```

Architecture:

```text
Iterator A
    |
    v
consume completely
    |
    v
Iterator B
    |
    v
consume completely
```

For multiple iterators:

```java
List<Iterator<T>>
```

and maintain:

```java
int currentIteratorIndex;
```

---

# 27. FlattenIterator

## Interview Question 16

Input:

```text
[
    [1, 2],
    [3, 4, 5],
    [6]
]
```

Expected:

```text
1 2 3 4 5 6
```

Architecture:

```text
Outer Iterator
       |
       v
Current Inner Iterator
       |
       v
FlattenIterator
```

---

## State

```java
private final Iterator<List<T>> outer;

private Iterator<T> inner;
```

Algorithm:

```text
while inner is exhausted:

    if outer has another collection:
        inner = outer.next().iterator()
    else:
        no more elements

return inner.next()
```

---

# 28. MapIterator

## Interview Question 17

Input:

```text
1 2 3 4
```

Transformation:

```java
x -> x * x
```

Output:

```text
1 4 9 16
```

---

## Implementation

```java
class MapIterator<T, R>
        implements MyIterator<R> {

    private final MyIterator<T> source;

    private final Function<T, R> mapper;

    public MapIterator(
            MyIterator<T> source,
            Function<T, R> mapper) {

        this.source = source;
        this.mapper = mapper;
    }

    @Override
    public boolean hasNext() {
        return source.hasNext();
    }

    @Override
    public R next() {

        if (!hasNext()) {
            throw new NoSuchElementException();
        }

        return mapper.apply(
                source.next()
        );
    }
}
```

---

# 29. PeekIterator

## Interview Question 18

Design:

```java
T peek();

T next();

boolean hasNext();
```

Input:

```text
1 2 3
```

Calling:

```java
peek()
```

returns:

```text
1
```

Calling again:

```java
peek()
```

still returns:

```text
1
```

Then:

```java
next()
```

returns:

```text
1
```

The iterator needs a one-element buffer.

---

# 30. DistinctIterator

## Interview Question 19

Input:

```text
1 2 2 3 3 3 4
```

Expected:

```text
1 2 3 4
```

Typical design:

```java
Set<T> seen =
        new HashSet<>();
```

Algorithm:

```text
candidate
    |
    v
already seen?
   /       \
 yes       no
 |          |
skip      remember
             |
             v
           return
```

---

# 31. Can DistinctIterator Avoid Caching?

For arbitrary data:

```text
No.
```

Why?

The iterator needs to remember previously seen values.

Worst-case:

```text
O(n)
```

additional memory.

This is a useful contrast with:

```text
FilterIterator
```

which only needs:

```text
O(1)
```

additional state.

---

# 32. TakeWhileIterator

## Interview Question 20

Input:

```text
1 2 3 4 0 5 6
```

Condition:

```java
x > 0
```

Output:

```text
1 2 3 4
```

The first failed condition ends iteration.

---

## Difference Between Filter and TakeWhile

### Filter

Input:

```text
1 2 0 3 4
```

Condition:

```text
x > 0
```

Output:

```text
1 2 3 4
```

### TakeWhile

Output:

```text
1 2
```

because `0` terminates traversal.

---

# 33. DropWhileIterator

Input:

```text
1 2 3 0 4 5
```

Condition:

```java
x > 0
```

Drop while condition is true.

Output:

```text
0 4 5
```

After the first failure, return the remaining values without continuing to evaluate the predicate.

---

# 34. ZipIterator

## Interview Question 21

Given:

```text
A = 1 2 3

B = 10 20 30
```

Return:

```text
(1,10)
(2,20)
(3,30)
```

Conceptually:

```text
A.next() + B.next()
```

---

# 35. Zip With Unequal Sources

Input:

```text
A = 1 2 3 4

B = 10 20
```

Possible contracts:

### Stop at shortest

```text
(1,10)
(2,20)
```

### Continue with null

```text
(1,10)
(2,20)
(3,null)
(4,null)
```

### Throw exception

Possible if equal-length sources are required.

The key interview lesson:

> **Clarify the contract before implementing.**

---

# 36. PartitionIterator

## Interview Question 22

Input:

```text
1 2 3 4 5 6 7
```

Chunk size:

```text
3
```

Output:

```text
[1,2,3]
[4,5,6]
[7]
```

This iterator returns:

```java
Iterator<List<T>>
```

instead of:

```java
Iterator<T>
```

The iterator itself is still lazy, but each chunk requires temporary storage.

Space:

```text
O(chunkSize)
```

---

# 37. CycleIterator

## Interview Question 23

Input:

```text
1 2 3
```

Create:

```text
CycleIterator
```

Conceptual output:

```text
1 2 3 1 2 3 1 2 3 ...
```

This is an infinite iterator.

---

# 38. The Critical CycleIterator Question

Ask:

> Can you implement CycleIterator using only `Iterator<T>` and without caching?

Generally:

```text
No.
```

Why?

A normal iterator eventually reaches:

```text
1 -> 2 -> 3 -> exhausted
```

There is no generic:

```text
restart()
```

or:

```text
previous()
```

operation.

Therefore, without caching, the source must provide another way to restart traversal.

---

# 39. CycleIterator Without Caching

Use:

```java
Iterable<T>
```

instead of:

```java
Iterator<T>
```

because:

```java
source.iterator()
```

can create a new iterator.

Architecture:

```text
Iterable
   |
   +---- Iterator #1
   |
   +---- Iterator #2
   |
   +---- Iterator #3
   |
   +---- ...
```

---

## Implementation

A clean implementation can be:

```java
class CycleIterator<T>
        implements MyIterator<T> {

    private final Iterable<T> source;

    private Iterator<T> current;

    public CycleIterator(
            Iterable<T> source) {

        this.source = source;

        this.current =
                source.iterator();
    }

    @Override
    public boolean hasNext() {

        if (current.hasNext()) {
            return true;
        }

        current =
                source.iterator();

        return current.hasNext();
    }

    @Override
    public T next() {

        if (!current.hasNext()) {

            current =
                    source.iterator();
        }

        if (!current.hasNext()) {

            throw new NoSuchElementException(
                    "Cannot iterate an empty source"
            );
        }

        return current.next();
    }
}
```

No entire collection is cached.

---

# 40. Why `Iterable` Is Important for CycleIterator

Compare:

```text
Iterator<T>
```

with:

```text
Iterable<T>
```

### Iterator

```text
One traversal
     |
     v
1 -> 2 -> 3 -> END
```

### Iterable

```text
source.iterator()
       |
       v
1 -> 2 -> 3

source.iterator()
       |
       v
1 -> 2 -> 3

source.iterator()
       |
       v
1 -> 2 -> 3
```

Therefore:

```text
No caching
+
Restartable source
=
Cycle possible
```

---

# 41. CycleIterator and Empty Source

Suppose:

```java
List<Integer> list =
        List.of();
```

What should:

```text
CycleIterator
```

do?

An empty source cannot meaningfully cycle.

Possible API contracts:

### Option 1

Allow creation but:

```java
hasNext() == false
```

### Option 2

Throw during construction.

Either is possible if explicitly documented.

The important thing is:

> Do not accidentally create an infinite iterator whose `hasNext()` always returns `true` for an empty source.

---

# 42. Cycle + Limit

Input:

```text
1 2 3
```

Pipeline:

```text
CycleIterator
      |
      v
LimitIterator(8)
```

Output:

```text
1 2 3 1 2 3 1 2
```

Architecture:

```text
Iterable
   |
   v
CycleIterator
   |
   v
LimitIterator(8)
   |
   v
Client
```

This is a good demonstration of iterator composition.

---

# 43. Lazy Evaluation

## Interview Question 24

What does lazy iteration mean?

### Eager approach

```text
Source
  |
  v
Create filtered List
  |
  v
Create limited List
  |
  v
Create mapped List
  |
  v
Client
```

This may create many intermediate objects.

---

## Lazy approach

```text
Source
  |
  v
FilterIterator
  |
  v
LimitIterator
  |
  v
MapIterator
  |
  v
Client
```

Each element is processed only when the client requests it.

---

# 44. Why Lazy Evaluation Matters

Suppose:

```text
1,000,000,000 elements
```

We need:

```text
first 5 odd numbers
```

A lazy pipeline:

```text
Source
  |
  v
OddIterator
  |
  v
LimitIterator(5)
```

can stop as soon as five results are found.

It does not need to build a billion-element intermediate collection.

---

# 45. Iterator Pipeline

Consider:

```text
Range(1, 100)
      |
      v
Filter odd
      |
      v
Skip 5
      |
      v
Limit 10
      |
      v
Map x -> x * 10
      |
      v
Client
```

Result:

```text
110
130
150
170
190
210
230
250
270
290
```

No intermediate result list is required.

---

# 46. Operation Ordering

This is a very important interview follow-up.

Input:

```text
1 2 3 4 5 6 7 8 9 10
```

---

## Pipeline A

```text
Filter odd
   |
   v
Limit 3
```

Output:

```text
1 3 5
```

---

## Pipeline B

```text
Limit 3
   |
   v
Filter odd
```

Output:

```text
1 3
```

Therefore:

> **Iterator operations are not necessarily commutative.**

The order changes both:

```text
result
```

and:

```text
amount of source data consumed
```

---

# 47. How Many Elements Are Consumed?

Consider:

```text
Range(1, 1_000_000_000)
        |
        v
Filter odd
        |
        v
Limit 5
```

The iterator only needs to inspect enough values to find:

```text
1
3
5
7
9
```

It does not need to traverse the entire billion-element range.

This is the core benefit of lazy iteration.

---

# 48. Iterator State Machine

An iterator is fundamentally a stateful object.

For a basic iterator:

```text
START
  |
  | next()
  v
POSITION 1
  |
  | next()
  v
POSITION 2
  |
  | next()
  v
POSITION 3
  |
  v
EXHAUSTED
```

Complex iterators have more state.

For example:

```text
PeekIterator
```

needs:

```text
source state
+
buffer state
```

---

# 49. Multiple `hasNext()` Calls

Consider:

```java
iterator.hasNext();
iterator.hasNext();
iterator.hasNext();

iterator.next();
```

The three `hasNext()` calls should not consume three elements.

For a filtering iterator, one possible implementation is:

```text
hasNext()
    |
    v
prepare next matching value
    |
    v
cache one value
```

Then:

```text
next()
```

consumes that cached value.

---

# 50. Calling `next()` Without `hasNext()`

Consider:

```java
iterator.next();
```

without calling:

```java
hasNext();
```

A good iterator should normally support this.

Therefore `next()` should validate exhaustion itself:

```java
if (!hasNext()) {
    throw new NoSuchElementException();
}
```

Do not assume callers always call `hasNext()` first.

---

# 51. Null Elements

Suppose the source contains:

```text
1 null 2 null 3
```

A filtering iterator cannot safely use:

```java
nextValue == null
```

as the only indication of:

```text
No value available
```

because:

```text
null
```

might itself be a valid element.

Use an explicit state flag such as:

```java
private boolean prepared;

private boolean hasBufferedValue;
```

This is a good senior-level edge-case question.

---

# 52. Empty Source

Every iterator should define behavior for:

```text
[]
```

Ask:

### Basic Iterator

```text
hasNext() -> false
next() -> NoSuchElementException
```

### FilterIterator

```text
hasNext() -> false
```

### LimitIterator

```text
hasNext() -> false
```

### FlattenIterator

Skip empty inner collections.

### InterleavingIterator

Continue with the non-empty source.

### CycleIterator

Must explicitly define empty-source behavior.

---

# 53. `remove()`

Java's `Iterator` historically supports:

```java
boolean hasNext();

T next();

void remove();
```

Now ask:

> What should happen if `remove()` is called before `next()`?

Usually invalid.

Ask:

> What if `remove()` is called twice without another `next()`?

Also invalid.

Therefore an iterator supporting `remove()` has a richer state machine.

---

# 54. `remove()` State Machine

```text
START
  |
  | next()
  v
CAN_REMOVE
  |
  | remove()
  v
CANNOT_REMOVE
  |
  | next()
  v
CAN_REMOVE
```

Invalid:

```text
START
 |
 | remove()
 X
```

And:

```text
CAN_REMOVE
 |
 | remove()
 v
CANNOT_REMOVE
 |
 | remove()
 X
```

This is an excellent state-machine interview problem.

---

# 55. Fail-Fast Iterator

Suppose:

```java
List<Integer> list =
        new ArrayList<>(
                List.of(1, 2, 3)
        );

Iterator<Integer> iterator =
        list.iterator();
```

Then the list is modified:

```java
list.add(100);
```

while iteration is in progress.

What should happen?

Java collection iterators often use fail-fast behavior and may throw:

```text
ConcurrentModificationException
```

---

# 56. `modCount` Concept

Conceptually:

```java
class Collection {

    int modCount;
}
```

Iterator remembers:

```java
int expectedModCount;
```

When the collection changes structurally:

```text
modCount++
```

Iterator checks:

```java
if (expectedModCount != modCount) {

    throw new ConcurrentModificationException();
}
```

Important:

> Fail-fast behavior is a detection mechanism, not a thread-safety mechanism.

---

# 57. Concurrent Modification

Consider:

```text
Thread A
    |
    +---- iterator.next()


Thread B
    |
    +---- collection.remove(...)
```

Without appropriate synchronization or a concurrent collection, behavior may be unsafe or inconsistent.

A production API should explicitly define its concurrency semantics.

---

# 58. Thread Safety

Ask:

> Is an Iterator automatically thread-safe?

No.

Possible designs:

```text
1. Iterator is not thread-safe
2. Caller synchronizes externally
3. Source is immutable
4. Iterator uses a snapshot
5. Source is a concurrent collection
6. Iterator itself synchronizes
```

Do not automatically synchronize everything.

Thread safety is an API design decision.

---

# 59. Infinite Iterators

Examples:

```text
CycleIterator
```

and:

```text
RangeIterator(0, infinity)
```

For an infinite iterator:

```java
hasNext()
```

may always return:

```text
true
```

for a non-empty source.

Therefore:

```java
while (iterator.hasNext()) {
    iterator.next();
}
```

may never terminate.

Infinite iterators are generally combined with:

```text
LimitIterator
```

or another terminating operation.

---

# 60. Infinite Iterator Pipeline

Example:

```text
Infinite Range
      |
      v
EvenIterator
      |
      v
LimitIterator(10)
      |
      v
Client
```

Only ten results are produced.

---

# 61. Complexity Analysis

A strong interview answer should discuss:

```text
Time complexity
```

and:

```text
Additional space complexity
```

---

## Typical Complexity Table

| Iterator | Extra Space | Special Requirement |
|---|---:|---|
| Forward | O(1) | None |
| Filtering | O(1) | Predicate |
| Odd | O(1) | None |
| Even | O(1) | None |
| Negative | O(1) | None |
| Positive | O(1) | None |
| Range | O(1) | None |
| Skip | O(1) | None |
| Limit | O(1) | None |
| Map | O(1) | Function |
| Backward | O(1) | Random access/reverse source |
| ZigZag | O(1) | Random access |
| Interleave | O(1) | Multiple sources |
| Round Robin | O(k) | Queue of iterators |
| Concat | O(1) | Multiple sources |
| Flatten | O(depth/current state) | Nested source |
| Peek | O(1) | One-element buffer |
| Distinct | O(n) | Set of seen values |
| Cycle | O(1) | Restartable source |
| Partition | O(chunk size) | Chunk buffer |

Where:

```text
n = number of source elements

k = number of child iterators
```

---

# 62. Which Iterators Need Caching?

This is one of the strongest senior interview questions.

## Usually no caching

```text
ForwardIterator
FilteringIterator
OddIterator
EvenIterator
NegativeIterator
PositiveIterator
RangeIterator
SkipIterator
LimitIterator
MapIterator
TakeWhileIterator
DropWhileIterator
```

These can generally operate using:

```text
O(1)
```

additional state.

---

## May require caching or stronger source capabilities

### BackwardIterator

Requires:

```text
Random access
OR
Reverse traversal
OR
Cached elements
```

---

### CycleIterator

Without caching it requires:

```text
Restartable source
```

such as:

```text
Iterable<T>
```

---

### DistinctIterator

Requires remembering previously seen values:

```text
O(n)
```

---

### PeekIterator

Requires a small look-ahead buffer:

```text
O(1)
```

---

# 63. What Does "Without Caching" Mean?

Suppose an interviewer says:

> Implement CycleIterator without caching.

Do not immediately code.

Ask:

> What type of source do I receive?

If:

```java
Iterator<T>
```

then:

```text
Cannot generally restart.
```

If:

```java
Iterable<T>
```

then:

```java
source.iterator()
```

can create a new traversal.

Therefore:

```text
Iterator<T>
+
No caching
=
Generally impossible to restart
```

while:

```text
Iterable<T>
+
No caching
=
Restart possible
```

This is a key abstraction-level answer.

---

# 64. Iterator Composition

The real power of the pattern is composition.

Example:

```text
Source
  |
  v
Filter
  |
  v
Skip
  |
  v
Limit
  |
  v
Map
  |
  v
Client
```

Each component has one responsibility.

---

# 65. Example Complete Pipeline

Input:

```text
Range(1, 100)
```

Pipeline:

```text
Filter odd
     |
     v
Skip 5
     |
     v
Limit 10
     |
     v
Map x -> x * 10
```

Result:

```text
110
130
150
170
190
210
230
250
270
290
```

No intermediate collection is necessary.

---

# 66. SOLID Design

The design naturally follows the Single Responsibility Principle.

```text
FilteringIterator
    -> filtering

LimitIterator
    -> limiting

SkipIterator
    -> skipping

MapIterator
    -> mapping

CycleIterator
    -> cycling

FlattenIterator
    -> flattening
```

New behaviors can be introduced by creating new iterator decorators rather than modifying existing iterators.

---

# 67. Iterator and Decorator Pattern

Consider:

```java
new LimitIterator<>(
    new FilteringIterator<>(
        source,
        predicate
    ),
    10
);
```

Conceptually:

```text
Source
  |
  v
FilteringIterator
  |
  v
LimitIterator
```

Each layer wraps the same abstraction:

```text
Iterator<T>
```

This is structurally similar to the Decorator pattern.

---

# 68. Iterator and Strategy Pattern

Different iterators represent different traversal strategies:

```text
Forward
Backward
ZigZag
Odd
Even
```

The client can use the common interface while the iterator encapsulates the traversal algorithm.

Therefore Iterator and Strategy can naturally overlap in implementation.

---

# 69. Iterator vs Stream

Java Streams provide higher-level lazy pipelines.

For example:

```java
numbers.stream()
       .filter(x -> x % 2 != 0)
       .skip(5)
       .limit(10)
       .map(x -> x * 10);
```

Conceptually this resembles:

```text
Iterator
    |
    v
Filter
    |
    v
Skip
    |
    v
Limit
    |
    v
Map
```

However, they are not identical abstractions.

An iterator gives direct, explicit traversal:

```java
hasNext()
next()
```

A Stream provides a richer processing model.

---

# 70. Why Custom Iterator Instead of Stream?

A custom iterator can make sense when:

```text
1. You need a custom traversal algorithm.

2. An API specifically requires Iterator.

3. You need explicit traversal state.

4. You are implementing a custom collection.

5. You need a reusable traversal abstraction.

6. You are implementing an external cursor-like source.
```

---

# 71. Multiple Independent Iterators

Consider:

```java
Iterator<Integer> a =
        collection.iterator();

Iterator<Integer> b =
        collection.iterator();
```

Expected:

```text
Iterator A -> independent state

Iterator B -> independent state
```

For example:

```text
Collection
   |
   +---- Iterator A
   |        index = 3
   |
   +---- Iterator B
            index = 0
```

The collection data can be shared.

The traversal state should normally belong to each iterator.

---

# 72. Important Principle

> **Traversal state belongs to the Iterator, not the Collection.**

Bad design:

```java
class Collection {

    private int currentIndex;
}
```

Then two clients interfere with each other.

Better:

```text
Collection
   |
   +---- Iterator A -> index A
   |
   +---- Iterator B -> index B
```

---

# 73. Resource-Backed Iterator

Suppose an iterator reads from:

```text
Database ResultSet
```

or:

```text
InputStream
```

Now ask:

> Is `Iterator<T>` enough?

Not necessarily.

The underlying source may require:

```text
close()
```

or:

```text
try-with-resources
```

or:

```text
transaction management
```

This introduces a broader design question:

> How should traversal resources be managed?

---

# 74. External Source Iterator

Imagine:

```text
Database
    |
    v
ResultSet
    |
    v
Iterator<T>
    |
    v
Client
```

Potential concerns include:

```text
Connection lifetime
Transaction lifetime
Network failure
Timeout
Resource cleanup
Pagination
Fetch size
Retry
```

This is where a simple Iterator pattern can evolve into a real system-design problem.

---

# 75. Pull-Based Nature of Iterator

Iterator processing is naturally pull-based:

```text
Consumer
   |
   | next()
   v
Source
```

The consumer decides:

> "Give me the next item."

This is different from push-based processing:

```text
Producer
   |
   | event
   v
Consumer
```

This distinction becomes useful when discussing:

```text
Reactive Streams
Backpressure
Message queues
Event processing
```

---

# 76. Common Mistakes

## Mistake 1 — Exposing Internal Data

```java
public List<T> getData()
```

when the client only needs traversal.

---

## Mistake 2 — Creating Intermediate Lists

Bad:

```java
List<Integer> result = new ArrayList<>();

for (...) {
    if (...) {
        result.add(...);
    }
}
```

when the requirement is lazy traversal.

---

## Mistake 3 — Advancing in `hasNext()`

Bad:

```java
boolean hasNext() {
    return source.next() != null;
}
```

This consumes an element.

---

## Mistake 4 — Returning `null` After Exhaustion

Usually prefer:

```java
throw new NoSuchElementException();
```

for Java-style iterators.

---

## Mistake 5 — Ignoring Empty Input

Especially dangerous for:

```text
CycleIterator
FlattenIterator
InterleavingIterator
```

---

## Mistake 6 — Ignoring Null Values

Do not assume:

```text
null == exhausted
```

unless the API explicitly forbids null elements.

---

## Mistake 7 — Assuming Reverse Traversal Is Always Possible

A normal:

```java
Iterator<T>
```

is forward-only.

---

## Mistake 8 — Assuming Cycle Always Works Without Caching

A plain iterator cannot generally rewind.

You need:

```text
restartable source
```

or:

```text
cached elements
```

---

## Mistake 9 — Ignoring Complexity

A correct implementation is not enough.

Always discuss:

```text
Time
Space
Source requirements
```

---

# 77. Progressive Interview Question Set

The following sequence can be used directly in an interview.

---

## Level 1 — Fundamentals

### Question 1

> Given a list, print every element using a `for` loop.

### Follow-up

> What happens internally in enhanced `for`?

---

### Question 2

> Why do we need Iterator?

Expected discussion:

```text
Encapsulation
Decoupling
Traversal abstraction
Multiple traversal strategies
```

---

### Question 3

> What is the difference between Iterable and Iterator?

Expected:

```text
Iterable -> can create iterator

Iterator -> represents one traversal
```

---

## Level 2 — Basic Iterator

### Question 4

> Implement an Iterator over an array.

### Follow-up

> Where should the current index live?

Expected:

```text
Iterator
```

---

### Question 5

> What happens when `next()` is called after exhaustion?

Expected:

```text
NoSuchElementException
```

for Java-style behavior.

---

### Question 6

> What happens if `hasNext()` is called five times?

Expected:

```text
It should not accidentally consume five elements.
```

---

# 78. Level 3 — Filtering

### Question 7

> Implement OddIterator.

### Question 8

> Implement EvenIterator.

### Question 9

> Implement NegativeIterator.

### Question 10

> Implement PositiveIterator.

---

### Follow-up

> Can OddIterator consume another OddIterator?

Expected:

```text
Yes.
```

Architecture:

```text
Iterator
   |
   v
OddIterator
   |
   v
OddIterator
```

---

### Follow-up

> Can we generalize all these iterators?

Expected:

```text
FilteringIterator<T>
```

with:

```java
Predicate<T>
```

---

# 79. Level 4 — Traversal

### Question 11

> Implement ForwardIterator.

### Question 12

> Implement BackwardIterator.

### Follow-up

> Can BackwardIterator work with only Iterator<T>?

Expected:

```text
Generally no.
```

---

### Question 13

> Implement ZigZagIterator.

Expected:

```text
left
right
left
right
```

---

# 80. Level 5 — Generation and Limiting

### Question 14

> Implement RangeIterator.

### Question 15

> Add step support.

### Question 16

> Implement SkipIterator.

### Question 17

> Implement LimitIterator.

---

# 81. Level 6 — Composition

### Question 18

> Implement MapIterator.

### Question 19

> Implement ConcatIterator.

### Question 20

> Implement InterleavingIterator.

### Question 21

> Generalize interleaving to N iterators.

Expected:

```text
RoundRobinIterator
```

using:

```java
Queue<Iterator<T>>
```

---

### Question 22

> Implement FlattenIterator.

---

# 82. Level 7 — Advanced

### Question 23

> Implement CycleIterator.

### Follow-up

> Can you implement it without caching?

Expected discussion:

```text
Iterator<T> -> generally no

Iterable<T> -> yes, because iterator can restart
```

---

### Question 24

> Implement PeekIterator.

### Question 25

> Implement DistinctIterator.

### Follow-up

> Can DistinctIterator work with O(1) memory?

Expected:

```text
Not for arbitrary input.
```

---

# 83. Level 8 — Java Behavior

### Question 26

> Explain remove().

### Question 27

> What happens if the collection changes during iteration?

### Question 28

> What is fail-fast behavior?

### Question 29

> What is modCount?

### Question 30

> Is Iterator thread-safe?

---

# 84. Level 9 — Architecture

### Question 31

> How would you build a reusable iterator pipeline?

Expected:

```text
Source
  |
  v
Filter
  |
  v
Skip
  |
  v
Limit
  |
  v
Map
```

---

### Question 32

> Which iterators require caching?

---

### Question 33

> Which require random access?

---

### Question 34

> Which operations are lazy?

---

### Question 35

> What are the time and space complexities?

---

### Question 36

> How does Iterator relate to SOLID?

---

### Question 37

> How does Iterator relate to Decorator and Strategy?

---

# 85. Final Senior-Level Challenge

## Problem

Design a reusable lazy iterator framework supporting:

```text
Range
Filter
Map
Skip
Limit
Cycle
Concat
Flatten
Interleave
```

Requirements:

```text
1. Avoid unnecessary caching.

2. Support iterator composition.

3. Preserve lazy evaluation.

4. Define empty-source behavior.

5. Define exhaustion behavior.

6. Discuss complexity.

7. Explain source requirements.

8. Explain which operations need random access.

9. Explain which operations need caching.

10. Explain whether Iterator<T> is enough.

11. Explain whether Iterable<T> is required.

12. Explain thread-safety behavior.
```

---

# 86. Example Final Architecture

```text
                         +----------------+
                         |    Iterable    |
                         +-------+--------+
                                 |
                            iterator()
                                 |
                                 v
                         +----------------+
                         |    Iterator    |
                         +-------+--------+
                                 |
             +-------------------+-------------------+
             |                   |                   |
             v                   v                   v
          Filter              Skip                Limit
             |                   |                   |
             +-------------------+-------------------+
                                 |
                                 v
                                Map
                                 |
                                 v
                               Client
```

---

# 87. Example Advanced Architecture

```text
                     +-------------------+
                     |      Source       |
                     +---------+---------+
                               |
                               v
                     +-------------------+
                     | FilterIterator    |
                     +---------+---------+
                               |
                               v
                     +-------------------+
                     | SkipIterator      |
                     +---------+---------+
                               |
                               v
                     +-------------------+
                     | LimitIterator     |
                     +---------+---------+
                               |
                               v
                     +-------------------+
                     | MapIterator       |
                     +---------+---------+
                               |
                               v
                            Client
```

---

# 88. Final Iterator Comparison

| Iterator | Purpose | Lazy | Extra Space | Source Requirement |
|---|---|---|---:|---|
| Forward | Forward traversal | Yes | O(1) | None |
| Backward | Reverse traversal | Yes | O(1) | Random access/reverse source |
| Odd | Odd filtering | Yes | O(1) | None |
| Even | Even filtering | Yes | O(1) | None |
| Negative | Negative filtering | Yes | O(1) | None |
| Positive | Positive filtering | Yes | O(1) | None |
| Filtering | Generic filtering | Yes | O(1) | Predicate |
| Range | Generate range | Yes | O(1) | None |
| Skip | Skip prefix | Yes | O(1) | None |
| Limit | Restrict output count | Yes | O(1) | None |
| Map | Transform values | Yes | O(1) | Function |
| ZigZag | Alternate ends | Yes | O(1) | Random access |
| Interleave | Alternate sources | Yes | O(1) | Multiple sources |
| Round Robin | N-source interleave | Yes | O(k) | Queue |
| Concat | Sequential sources | Yes | O(1) | Multiple sources |
| Flatten | Flatten nested sources | Yes | O(depth) | Nested source |
| Peek | Look ahead | Yes | O(1) | One-value buffer |
| Distinct | Remove duplicates | Yes | O(n) | Seen set |
| TakeWhile | Stop at predicate failure | Yes | O(1) | Predicate |
| DropWhile | Drop prefix by predicate | Yes | O(1) | Predicate |
| Zip | Pair sources | Yes | O(1) | Two sources |
| Partition | Group elements | Yes | O(chunk size) | Chunk buffer |
| Cycle | Repeat source | Yes | O(1) without cache | Restartable source |

---

# 89. Most Important Concepts

## Concept 1 — Iterator Owns Traversal State

The collection owns:

```text
Data
```

The iterator owns:

```text
Traversal state
```

---

## Concept 2 — Iterable Creates Iterators

```text
Iterable
    |
    v
iterator()
    |
    v
Iterator
```

---

## Concept 3 — Iterator Can Wrap Iterator

```text
Source
  |
  v
OddIterator
  |
  v
LimitIterator
  |
  v
MapIterator
```

---

## Concept 4 — Lazy Evaluation

Do not unnecessarily create:

```text
List A
List B
List C
```

Instead:

```text
Iterator
   |
   v
Iterator
   |
   v
Iterator
```

---

## Concept 5 — Source Capability Matters

A traversal algorithm cannot do something the source does not support.

For example:

```text
Iterator<T>
```

does not provide:

```text
previous()
```

Therefore backward traversal is not generally possible.

---

## Concept 6 — No Caching Does Not Mean No Restart

If the source is:

```java
Iterable<T>
```

we can request:

```java
source.iterator()
```

again.

Therefore:

```text
No caching
+
Restartable source
```

can support cycling.

---

## Concept 7 — Some Operations Inherently Need Memory

For example:

```text
DistinctIterator
```

needs to remember previously seen values.

---

## Concept 8 — Operation Order Matters

```text
Filter -> Limit
```

is not necessarily equivalent to:

```text
Limit -> Filter
```

---

# 90. Final Interview Mental Model

Whenever an interviewer asks:

> "Design an iterator."

Immediately think:

```text
                What is my source?
                       |
                       v
              What capabilities?
                       |
          +------------+-------------+
          |            |             |
       Forward      Random       Restartable
       only         access         source
          |            |             |
          v            v             v
       Filter       Reverse        Cycle
       Map          ZigZag
       Skip
       Limit
```

Then ask:

```text
What state do I need?

Do I need buffering?

Do I need caching?

Do I need random access?

Can I wrap another iterator?

Is the operation lazy?

What happens at exhaustion?

What happens for empty input?

What happens with null?

What is the complexity?

What happens if the source changes?

Is it thread-safe?
```

---

# 91. Final Interview Checklist

When designing any iterator, walk through these questions:

```text
1. What is the input/source?

2. Is it:
   - Iterator<T>
   - Iterable<T>
   - List<T>
   - Array
   - Stream
   - External source?

3. What is the traversal order?

4. What state does the iterator need?

5. Does hasNext() advance the source?

6. What does next() do after exhaustion?

7. Can the source contain null?

8. Is the iterator lazy?

9. Does it require buffering?

10. Does it require caching?

11. Does it require random access?

12. Can it wrap another iterator?

13. Can it compose with another iterator?

14. What happens with an empty source?

15. What happens with an infinite source?

16. What is the time complexity?

17. What is the additional space complexity?

18. Is it thread-safe?

19. What happens if the source changes?

20. Does it support remove()?

21. Can the collection create multiple independent iterators?

22. Does the source need to be restartable?

23. Can the iterator terminate?

24. Is operation ordering important?
```

---

# 92. Ultimate Interview Question

After the candidate has implemented several individual iterators, ask:

> **Design a generic lazy iterator framework where every iterator can be composed with another iterator. Support filtering, mapping, skipping, limiting, flattening, concatenation, interleaving, cycling, and range generation. Avoid unnecessary caching. For every iterator, explain the required source capabilities, time complexity, space complexity, and whether the operation is lazy.**

The candidate should eventually arrive at something conceptually similar to:

```text
                         +----------------+
                         |    Iterable    |
                         +-------+--------+
                                 |
                            iterator()
                                 |
                                 v
                         +----------------+
                         |    Iterator    |
                         +-------+--------+
                                 |
             +-------------------+-------------------+
             |                   |                   |
             v                   v                   v
          Filter              Skip                Limit
             |                   |                   |
             +-------------------+-------------------+
                                 |
                                 v
                                Map
                                 |
                                 v
                               Client
```

---

# 93. Final Takeaway

The Iterator pattern is not fundamentally about:

```java
hasNext()
next()
```

Those are only the interface.

The real design problem is:

> **How can we expose controlled traversal over a data source while keeping the source's internal representation hidden and allowing different traversal strategies to be composed efficiently and lazily?**

Once that idea is clear, the different iterator problems become variations of the same problem:

```text
OddIterator
EvenIterator
NegativeIterator
PositiveIterator
ForwardIterator
BackwardIterator
RangeIterator
SkipIterator
LimitIterator
ZigZagIterator
InterleavingIterator
RoundRobinIterator
ConcatIterator
FlattenIterator
MapIterator
PeekIterator
DistinctIterator
TakeWhileIterator
DropWhileIterator
ZipIterator
CycleIterator
```

For every one of them, ask:

```text
                 +---------------------+
                 |   What is source?   |
                 +----------+----------+
                            |
                            v
                 +---------------------+
                 | What traversal is   |
                 | required?           |
                 +----------+----------+
                            |
                            v
                 +---------------------+
                 | What state is       |
                 | required?           |
                 +----------+----------+
                            |
                            v
                 +---------------------+
                 | Need caching?       |
                 | Need buffering?     |
                 | Need random access? |
                 +----------+----------+
                            |
                            v
                 +---------------------+
                 | Can it compose with |
                 | another iterator?   |
                 +----------+----------+
                            |
                            v
                 +---------------------+
                 | Complexity +        |
                 | edge cases          |
                 +---------------------+
```

That is the mental model that turns an Iterator interview from a collection of coding exercises into a **design problem**.