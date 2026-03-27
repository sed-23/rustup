# VecDeque, BinaryHeap, and Other Collections 📦

> **Beyond Vec, HashMap, and HashSet, Rust's standard library provides specialized collections: `VecDeque` for double-ended queues, `BinaryHeap` for priority queues, and `LinkedList` for rare edge cases.**

---

## Table of Contents

- [VecDeque — Double-Ended Queue](#vecdeque--double-ended-queue)
- [BinaryHeap — Priority Queue](#binaryheap--priority-queue)
- [LinkedList](#linkedlist)
- [When to Use What](#when-to-use-what)
- [Exercises](#exercises)
- [Summary](#summary)

---

## VecDeque — Double-Ended Queue

`VecDeque<T>` is a growable ring buffer. It supports efficient push/pop at **both ends**:

```
Vec:      push/pop only at back → O(1)
          insert/remove at front → O(n) (shifts everything)

VecDeque: push/pop at front AND back → O(1)
```

### Memory Layout

```
VecDeque (ring buffer):

Internal buffer: [_, _, 3, 4, 5, 1, 2, _]
                       ↑ tail      ↑ head
                       
Logical order:   [1, 2, 3, 4, 5]
                  ↑ front   ↑ back
```

### Basic Operations

```rust
use std::collections::VecDeque;

fn main() {
    let mut deque = VecDeque::new();
    
    // Push to back (like Vec)
    deque.push_back(1);
    deque.push_back(2);
    deque.push_back(3);
    // [1, 2, 3]
    
    // Push to front (O(1) — Vec would be O(n))
    deque.push_front(0);
    // [0, 1, 2, 3]
    
    // Pop from back
    println!("{:?}", deque.pop_back());   // Some(3)
    // [0, 1, 2]
    
    // Pop from front
    println!("{:?}", deque.pop_front());  // Some(0)
    // [1, 2]
    
    // Index
    println!("{}", deque[0]);  // 1
    
    // From vec! equivalent
    let deque: VecDeque<i32> = VecDeque::from([1, 2, 3, 4, 5]);
    
    // Convert to Vec
    let v: Vec<i32> = deque.into();
    
    // With capacity
    let deque: VecDeque<i32> = VecDeque::with_capacity(100);
}
```

### Rotation

```rust
use std::collections::VecDeque;

fn main() {
    let mut deque: VecDeque<i32> = (1..=5).collect();
    println!("{:?}", deque);  // [1, 2, 3, 4, 5]
    
    // Rotate left: move front elements to back
    deque.rotate_left(2);
    println!("{:?}", deque);  // [3, 4, 5, 1, 2]
    
    // Rotate right: move back elements to front
    deque.rotate_right(2);
    println!("{:?}", deque);  // [1, 2, 3, 4, 5]
}
```

### Use Case: Sliding Window

```rust
use std::collections::VecDeque;

fn max_in_sliding_window(nums: &[i32], k: usize) -> Vec<i32> {
    let mut result = Vec::new();
    let mut window: VecDeque<usize> = VecDeque::new();  // stores indices
    
    for i in 0..nums.len() {
        // Remove indices outside the window
        while let Some(&front) = window.front() {
            if front + k <= i {
                window.pop_front();
            } else {
                break;
            }
        }
        
        // Remove smaller elements from back
        while let Some(&back) = window.back() {
            if nums[back] <= nums[i] {
                window.pop_back();
            } else {
                break;
            }
        }
        
        window.push_back(i);
        
        if i >= k - 1 {
            result.push(nums[*window.front().unwrap()]);
        }
    }
    
    result
}

fn main() {
    let nums = vec![1, 3, -1, -3, 5, 3, 6, 7];
    println!("{:?}", max_in_sliding_window(&nums, 3));
    // [3, 3, 5, 5, 6, 7]
}
```

### Ring Buffers — The Data Structure Behind VecDeque

`VecDeque` is implemented as a **ring buffer** (also called a circular buffer) — a classic data structure dating back to the 1960s. Understanding how it works under the hood explains *why* it's O(1) at both ends.

#### How a Ring Buffer Works

A ring buffer is a fixed-size array with two pointers — **head** and **tail** — that *wrap around* when they reach the end:

- **`push_back`**: advances the tail pointer forward. If it reaches the end of the array, it wraps to index 0.
- **`push_front`**: moves the head pointer backward. If it goes below 0, it wraps to the last index.
- **`pop_front`**: advances head forward. **`pop_back`**: moves tail backward.

Because no elements ever shift, every operation is **O(1)**.

```
  Ring Buffer (capacity 8):

  Physical array:  [ _, _, C, D, E, A, B, _ ]
                     0  1  2  3  4  5  6  7
                              tail↑  head↑

  Logical view (reading head → tail, wrapping):

       head
        ↓
  ┌───┬───┬───┬───┬───┐
  │ A │ B │ C │ D │ E │    ← contiguous logical sequence
  └───┴───┴───┴───┴───┘
        wraps around ↻

  push_front('Z'):  head moves 5→4
  [ _, _, C, D, E, Z, A, B ]   ← no shifting!
                    ↑ new head

  push_back('F'):   tail moves 2→? (grows if needed)
```

Contrast this with `Vec`, which must shift *every element* forward when you insert at the front — O(n) work.

#### Ring Buffers in the Real World

Ring buffers are everywhere in systems programming:

| Domain | Use Case | Why Ring Buffers? |
|--------|----------|-------------------|
| **Linux kernel** | Network packet buffers (`sk_buff` rings) | Fixed-size, no allocation in hot path |
| **Audio processing** | Sample buffers between capture and playback | Producer-consumer with bounded latency |
| **Keyboard/serial I/O** | Input event buffers | Hardware writes, software reads at different rates |
| **Game engines** | Command queues, replay buffers | Bounded history with automatic overwrite |
| **Logging** | Circular log buffers | Keep last N entries without growing |

The **producer-consumer pattern** is the classic use case: one side writes data, the other side reads it. A ring buffer lets both sides operate independently at O(1) cost, with no allocation. In real-time systems (audio, video, embedded), this is critical — you pre-allocate the buffer once and never touch the allocator again.

#### VecDeque vs Vec — When to Choose Which

```
  Vec:      ▓▓▓▓▓▓▓▓░░░░       ← elements packed at front, push/pop at back
            fast →→→ back        slow ←←← front (shifts everything)

  VecDeque: ░░▓▓▓▓▓▓░░░░       ← elements can start anywhere in the buffer
            fast ←← front       fast →→→ back (both O(1))
```

- **Use `Vec`** when you only add/remove from the back (stack pattern, most common).
- **Use `VecDeque`** when you need FIFO (queue) behavior, or fast operations at *both* ends.
- `Vec` has slightly less overhead (one pointer instead of two), so default to `Vec` unless you need front operations.

---

## BinaryHeap — Priority Queue

`BinaryHeap<T>` is a max-heap. The largest element is always at the top:

```
              10          ← pop() returns this (the max)
             /  \
            8    9
           / \  / \
          4  5 6   7
```

### Basic Operations

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();
    
    // Push elements
    heap.push(3);
    heap.push(1);
    heap.push(5);
    heap.push(2);
    heap.push(4);
    
    // Peek at the max (doesn't remove)
    println!("Max: {:?}", heap.peek());  // Some(5)
    
    // Pop the max
    println!("{:?}", heap.pop());  // Some(5)
    println!("{:?}", heap.pop());  // Some(4)
    println!("{:?}", heap.pop());  // Some(3)
    println!("{:?}", heap.pop());  // Some(2)
    println!("{:?}", heap.pop());  // Some(1)
    println!("{:?}", heap.pop());  // None
    
    // From array
    let heap = BinaryHeap::from([3, 1, 4, 1, 5, 9]);
    
    // Into sorted vec (ascending)
    let sorted = heap.into_sorted_vec();
    println!("{:?}", sorted);  // [1, 1, 3, 4, 5, 9]
}
```

### Min-Heap with Reverse

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn main() {
    // Wrap values in Reverse for a min-heap
    let mut min_heap = BinaryHeap::new();
    min_heap.push(Reverse(3));
    min_heap.push(Reverse(1));
    min_heap.push(Reverse(5));
    
    println!("{:?}", min_heap.pop());  // Some(Reverse(1))  ← minimum!
    println!("{:?}", min_heap.pop());  // Some(Reverse(3))
    println!("{:?}", min_heap.pop());  // Some(Reverse(5))
}
```

### Use Case: Top K Elements

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn top_k(nums: &[i32], k: usize) -> Vec<i32> {
    // Use a min-heap of size k
    let mut heap: BinaryHeap<Reverse<i32>> = BinaryHeap::with_capacity(k + 1);
    
    for &n in nums {
        heap.push(Reverse(n));
        if heap.len() > k {
            heap.pop();  // Remove the smallest
        }
    }
    
    heap.into_iter().map(|Reverse(x)| x).collect()
}

fn main() {
    let nums = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5];
    let mut result = top_k(&nums, 3);
    result.sort();
    println!("Top 3: {:?}", result);  // [5, 6, 9]
}
```

### Use Case: Task Scheduler

```rust
use std::collections::BinaryHeap;

#[derive(Eq, PartialEq)]
struct Task {
    priority: u32,
    name: String,
}

// Higher priority = greater
impl Ord for Task {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        self.priority.cmp(&other.priority)
    }
}

impl PartialOrd for Task {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

fn main() {
    let mut scheduler = BinaryHeap::new();
    
    scheduler.push(Task { priority: 1, name: "Low priority".into() });
    scheduler.push(Task { priority: 10, name: "Critical!".into() });
    scheduler.push(Task { priority: 5, name: "Medium priority".into() });
    
    while let Some(task) = scheduler.pop() {
        println!("[Priority {}] {}", task.priority, task.name);
    }
    // [Priority 10] Critical!
    // [Priority 5] Medium priority
    // [Priority 1] Low priority
}
```

### Binary Heaps and Priority Queues — From Theory to Practice

Binary heaps were invented by **J.W.J. Williams in 1964** as part of the heapsort algorithm — one of the most elegant data structures in computer science. Understanding the theory unlocks a powerful problem-solving tool.

#### The Heap Property

A binary heap is a *complete binary tree* stored in an array, satisfying one rule:

- **Max-heap**: every parent is **≥** its children (Rust's `BinaryHeap`)
- **Min-heap**: every parent is **≤** its children

```
  Max-Heap (Rust's BinaryHeap):       Min-Heap (BinaryHeap<Reverse<T>>):

          50                                    1
         /  \                                  / \
       30    40                               5    3
      / \   / \                             / \  / \
    10  20 15  35                          10  8 7   6
```

Rust's `BinaryHeap` is a **max-heap** — `pop()` always returns the largest element. For a min-heap, wrap your values in `std::cmp::Reverse`:

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

let mut min_heap = BinaryHeap::new();
min_heap.push(Reverse(5));
min_heap.push(Reverse(1));
min_heap.push(Reverse(3));
assert_eq!(min_heap.pop(), Some(Reverse(1)));  // smallest first!
```

#### The Array Trick — No Pointers Needed

The key insight: a complete binary tree maps perfectly onto a flat array. For element at index `i`:

```
  Parent:      (i - 1) / 2
  Left child:  2 * i + 1
  Right child: 2 * i + 2

  Array:  [50, 30, 40, 10, 20, 15, 35]
  Index:    0   1   2   3   4   5   6

  Tree:         50 (0)
               /      \
          30 (1)      40 (2)
          /   \       /   \
      10 (3) 20 (4) 15 (5) 35 (6)

  Index 2's children: 2*2+1=5, 2*2+2=6  → 15, 35  ✓
  Index 4's parent:   (4-1)/2=1          → 30      ✓
```

No pointers, no allocations per node — just a contiguous `Vec` internally. This makes heaps **cache-friendly** and very fast in practice.

#### Heaps Across Languages

| Language | Type | Max or Min? | Backing Storage | Default Behavior |
|----------|------|-------------|-----------------|------------------|
| **Rust** | `BinaryHeap<T>` | Max | `Vec<T>` | Largest first |
| **Java** | `PriorityQueue<T>` | Min | Array | Smallest first |
| **Python** | `heapq` | Min | List | Smallest first |
| **C++** | `priority_queue<T>` | Max | `vector<T>` | Largest first |
| **Go** | `container/heap` | Min (by convention) | Slice | User-defined |

Notice: most languages default to min-heap. Rust and C++ are the exceptions with max-heap.

#### Real-World Applications

- **Dijkstra's shortest path**: process the closest unvisited node first → min-heap of (distance, node)
- **A\* pathfinding**: game AI, GPS navigation — priority queue ordered by estimated total cost
- **Job/task schedulers**: OS process scheduling, print queues — highest priority job executes first
- **K-largest / K-smallest problems**: maintain a heap of size K → O(n log k) total, much better than sorting O(n log n) when k ≪ n
- **Median-finding**: two heaps (max-heap for lower half, min-heap for upper half) → O(log n) per insertion, O(1) median query
- **Event-driven simulation**: events with timestamps enter a priority queue, always processing the earliest event next — used in network simulators, physics engines, and discrete-event systems

#### The Heap Guarantee

One subtlety: a `BinaryHeap` is **not fully sorted**. Only the top element is guaranteed to be the max. The internal order of other elements is arbitrary (they satisfy the heap property, but sibling order is undefined). If you need a fully sorted iteration, use `.into_sorted_vec()` — or just pop repeatedly.

---

## LinkedList

`LinkedList<T>` is a doubly-linked list. **Almost never the right choice:**

```rust
use std::collections::LinkedList;

fn main() {
    let mut list = LinkedList::new();
    
    list.push_back(1);
    list.push_back(2);
    list.push_front(0);
    
    println!("{:?}", list);  // [0, 1, 2]
    
    // Pop from either end
    println!("{:?}", list.pop_front());  // Some(0)
    println!("{:?}", list.pop_back());   // Some(2)
    
    // Append another list (O(1))
    let mut a = LinkedList::from([1, 2, 3]);
    let mut b = LinkedList::from([4, 5, 6]);
    a.append(&mut b);
    println!("{:?}", a);  // [1, 2, 3, 4, 5, 6]
    println!("{:?}", b);  // []  (moved into a)
}
```

**Why avoid LinkedList?**
- Poor cache locality (nodes scattered in memory)
- Every node has 2 extra pointers (16 bytes overhead on 64-bit)
- `Vec` and `VecDeque` are faster for almost everything
- Only advantage: O(1) split/append if you have a cursor

### LinkedList — Why You (Almost) Never Need It

Linked lists hold a special place in CS history. They were among the **first data structures ever invented** (1955–56, by Allen Newell, Cliff Shaw, and Herbert Simon at RAND Corporation, for their Logic Theory Machine). Every CS curriculum teaches them extensively.

But in modern practice, they are rarely the right choice. Here's why.

#### The Cache Locality Problem

Every node in a linked list is a **separate heap allocation**. Nodes end up scattered across memory:

```
  Vec (contiguous):
  ┌───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 3 │ 4 │ 5 │   ← all in one cache line, prefetcher loves this
  └───┴───┴───┴───┴───┘

  LinkedList (scattered):
  ┌───┐    ┌───┐    ┌───┐    ┌───┐    ┌───┐
  │ 1 │───→│ 2 │───→│ 3 │───→│ 4 │───→│ 5 │
  └───┘    └───┘    └───┘    └───┘    └───┘
  0x1A00   0x4F20   0x0830   0xBB10   0x72C0
     ↑ scattered across memory — cache miss on every node!
```

Modern CPUs are incredibly fast at sequential memory access (prefetching, cache lines of 64 bytes). Walking a `Vec` is essentially free after the first access. Walking a linked list triggers a **cache miss on nearly every node** — 50–100x slower per access.

#### Bjarne Stroustrup's Famous Benchmark

**Bjarne Stroustrup**, creator of C++, demonstrated something counterintuitive: inserting into the **middle** of a `Vec` (which requires shifting all subsequent elements) is *faster* than inserting into a linked list — even though the linked list insertion is "O(1)" once you have the position.

The reason: shifting elements in a contiguous array is a single `memcpy` — CPUs do this at memory bandwidth speed. Finding the insertion point in a linked list requires chasing pointers through scattered memory. The crossover point where linked lists actually win is around **500,000+ elements**, and only if you already have a pointer/cursor to the insertion point.

#### Memory Overhead

Rust's `LinkedList` is doubly-linked. Each node stores:

```
  Node<T>:
  ┌──────────┬──────────┬──────────┐
  │  *prev   │  value   │  *next   │
  │ (8 bytes)│ (T size) │ (8 bytes)│
  └──────────┴──────────┴──────────┘
     + allocator overhead (~16 bytes per allocation)

  For a LinkedList<i32>:  4 bytes of data + ~32 bytes of overhead = 8x waste!
  For a Vec<i32>:         4 bytes of data + ~0 bytes overhead per element
```

#### When LinkedList IS Useful

There are legitimate (rare) use cases:

- **O(1) split and append**: splitting a linked list at a cursor position or appending two lists is O(1) — `Vec` requires O(n) copies
- **Intrusive linked lists in kernel/OS code**: nodes embedded in larger structures, no separate allocation
- **Cursor-based editing** (unstable API): if you need to insert/remove at arbitrary positions while iterating, the cursor API provides true O(1) operations

```rust
// The unstable cursor API — the ONLY time LinkedList wins
// #![feature(linked_list_cursors)]
// let mut cursor = list.cursor_front_mut();
// cursor.move_next();     // O(1)
// cursor.insert_after(x); // O(1) — true constant time!
```

#### The Broader Lesson: Big-O Isn't Everything

This is one of the most important performance lessons in systems programming:

> **Theoretical complexity (Big-O) doesn't account for cache behavior. Real performance is about data layout.**

- O(n) operation on contiguous memory (Vec) often beats O(1) operation on scattered memory (LinkedList)
- The Rust community consensus: *"If you're reaching for `LinkedList`, reconsider. `VecDeque` or `Vec` is almost always faster."*
- Default to `Vec`. If you need front+back operations, use `VecDeque`. Only reach for `LinkedList` if you have a proven, measured need for cursor-based O(1) splicing.

---

## When to Use What

| Collection | Best For | Push/Pop | Index | Sorted |
|-----------|----------|----------|-------|--------|
| `Vec<T>` | General purpose | Back: O(1) | O(1) | No |
| `VecDeque<T>` | Queue, front+back operations | Both ends: O(1) | O(1) | No |
| `BinaryHeap<T>` | Priority queue, top-k | Push: O(log n), Pop max: O(log n) | No | Partial (max) |
| `LinkedList<T>` | Almost never | Both ends: O(1) | O(n) | No |
| `HashMap<K,V>` | Key-value lookup | N/A | By key: O(1) | No |
| `BTreeMap<K,V>` | Sorted key-value | N/A | By key: O(log n) | Yes |
| `HashSet<T>` | Unique values, membership | N/A | N/A | No |
| `BTreeSet<T>` | Sorted unique values | N/A | N/A | Yes |

Decision flowchart:

```
Need key-value pairs?
├── Yes → Need sorted keys? → Yes: BTreeMap
│                            → No:  HashMap
└── No → Need unique values only?
         ├── Yes → Need sorted? → Yes: BTreeSet
         │                       → No:  HashSet
         └── No → Need fast front+back operations?
                  ├── Yes → VecDeque
                  └── No → Need priority/max/min?
                           ├── Yes → BinaryHeap
                           └── No → Vec
```

---

## Exercises

### Exercise 1: Queue Implementation

Implement a simple task queue using VecDeque:

<details>
<summary>Solution</summary>

```rust
use std::collections::VecDeque;

struct TaskQueue {
    queue: VecDeque<String>,
}

impl TaskQueue {
    fn new() -> Self {
        TaskQueue { queue: VecDeque::new() }
    }
    
    fn enqueue(&mut self, task: String) {
        self.queue.push_back(task);
    }
    
    fn dequeue(&mut self) -> Option<String> {
        self.queue.pop_front()
    }
    
    fn peek(&self) -> Option<&String> {
        self.queue.front()
    }
    
    fn len(&self) -> usize {
        self.queue.len()
    }
}

fn main() {
    let mut q = TaskQueue::new();
    q.enqueue("Task A".into());
    q.enqueue("Task B".into());
    q.enqueue("Task C".into());
    
    while let Some(task) = q.dequeue() {
        println!("Processing: {}", task);
    }
}
```

</details>

### Exercise 2: Merge K Sorted Lists

Use a BinaryHeap to merge k sorted vectors into one sorted vector:

<details>
<summary>Solution</summary>

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

fn merge_sorted(lists: Vec<Vec<i32>>) -> Vec<i32> {
    // (value, list_index, element_index)
    let mut heap: BinaryHeap<Reverse<(i32, usize, usize)>> = BinaryHeap::new();
    
    // Push first element of each list
    for (i, list) in lists.iter().enumerate() {
        if !list.is_empty() {
            heap.push(Reverse((list[0], i, 0)));
        }
    }
    
    let mut result = Vec::new();
    
    while let Some(Reverse((val, list_idx, elem_idx))) = heap.pop() {
        result.push(val);
        
        let next_idx = elem_idx + 1;
        if next_idx < lists[list_idx].len() {
            heap.push(Reverse((lists[list_idx][next_idx], list_idx, next_idx)));
        }
    }
    
    result
}

fn main() {
    let lists = vec![
        vec![1, 4, 7],
        vec![2, 5, 8],
        vec![3, 6, 9],
    ];
    println!("{:?}", merge_sorted(lists));
    // [1, 2, 3, 4, 5, 6, 7, 8, 9]
}
```

</details>

---

## Summary

| Collection | Internal | Push Front | Push Back | Pop Front | Pop Back | Use Case |
|------------|----------|-----------|-----------|-----------|----------|----------|
| `Vec` | Array | O(n) | O(1)* | O(n) | O(1) | Default list |
| `VecDeque` | Ring buffer | O(1)* | O(1)* | O(1) | O(1) | Queue/deque |
| `BinaryHeap` | Binary heap | — | O(log n) | — | O(log n) | Priority queue |
| `LinkedList` | Doubly-linked | O(1) | O(1) | O(1) | O(1) | Rarely used |

\* amortized

---

**Previous:** [← Sets](./06-sets.md) · **Next:** [Choosing Collections →](./08-choosing-collections.md)

<p align="center"><i>Tutorial 7 of 8 — Stage 5: Collections</i></p>
