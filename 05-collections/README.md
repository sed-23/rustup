# Stage 5: Collections Deep Dive 📦

> *Single values are nice. Collections of values are where the real work happens.*

## What You'll Learn

- Vectors (`Vec<T>`) — the workhorse of Rust collections
- String internals — UTF-8, capacity, len vs capacity
- HashMaps — key-value storage with O(1) lookups
- BTreeMaps — sorted key-value storage
- HashSets & BTreeSets — unique value collections
- VecDeque — double-ended queues
- When to use which collection (decision flowchart)
- Memory layout of every collection

## Prerequisites

- Completed [Stage 4: Structuring Data](../04-structuring-data/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Vectors — Vec<T>](./01-vectors.md) | Creating, reading, updating, iterating, memory growth |
| 2 | [Vector Memory Layout](./02-vector-memory-layout.md) | How Vec grows, capacity vs length, reallocation |
| 3 | [String Internals](./03-string-internals.md) | UTF-8, bytes vs chars vs graphemes, String operations |
| 4 | [HashMap<K, V>](./04-hashmaps.md) | Creating, inserting, accessing, entry API, hashing |
| 5 | [BTreeMap & Ordered Collections](./05-btreemap.md) | Sorted maps, range queries, when to use over HashMap |
| 6 | [HashSet & BTreeSet](./06-sets.md) | Unique values, set operations (union, intersection, etc.) |
| 7 | [VecDeque & Other Collections](./07-vecdeque-and-others.md) | Double-ended queue, LinkedList, BinaryHeap |
| 8 | [Choosing the Right Collection](./08-choosing-collections.md) | Decision flowchart, performance comparison, real-world tips |

## By the End of This Stage

You'll know every standard collection in Rust, understand their memory layouts, and always pick the right one for the job.

---

**Previous:** [← Stage 4: Structuring Data](../04-structuring-data/) | **Next:** [Stage 6: Closures & Iterators →](../06-closures-and-iterators/)
