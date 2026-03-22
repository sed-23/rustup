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
