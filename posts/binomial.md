# Heaps

In this note, we'll explore the implementation and analysis of a binomial heap. We'll begin by discussing traditional heaps — their uses and limitations, before introducing key ideas that we'll use to build our advanced heap. As with all other notes in this series, this post follows literate programming ethos; All code snippets are fully functional and runnable. As always, if you find any bugs, feel free to open a PR.

## The Binary Heap

Heaps are an important class of data structures that are used to represent data when we only ever need access to the extreme values. In particular, they give us O(1) access to the `max` or `min` value. In general, a heap is any data structure that exposes the API below:

|    Endpoint    |                Description                                               |
|----------------|--------------------------------------------------------------------------|
| `Make Heap`    |                    Creates a new heap                                    |
| `Insert(x)`    |        Adds the given item into the heap                                 |
| `Minimum`      |        Returns a reference to the smallest element in the heap           |
| `Extract Min`  |        Remove smallest element and return it                             |
| `Union`        | Merge the contents of this heap with those of another                    |
| `Decrease Key` | Reduce the value of the given key. This changes its position in the heap |
| `Delete(x)`    | Remove the given key from the heap                                       |

The most common kind of heap is the min binary heap. This can be understood conceptually as an almost complete binary tree in which the values of the child nodes are always larger than the value of the parent node. This way, the smallest value is always at the root. The concrete implementation of this data structure is often as an array, with the children of node `i` at `i * 2` and `i*2 + 1`. Note here that this means that all the nodes in the upper half of the array are leaves or trivial heaps.

```rust
/// WIP
```

We can use normal binary heaps to implement a host of other data structures and algorithms, such as an LRU cache, a Job scheduler, and finding the median of a stream of objects. Normal heaps are generally excellent when all we need is fast access to the largest or the smallest value in a dynamic set. They are not as efficient, however, when we want to merge 2 heaps. Joining the two arrays that underlie the heaps and then running min/max heapify takes liner time — this can be too slow for some operations. Binomial and Fibonacci heaps allow us to efficiently represent and implement merge-able heaps.

## The Binomial Tree

Each tree in the forest satisfies the binomial heap properties: (a) Each binomial tree is a min heap, and (b) For any non-negative integer `k`, there is at most `1` binomial tree in the forest whose root has degree `k` (the degree of a binomial tree is the number of nodes in the tree).

A binomial tree of order `k` consists of 2 binomial trees of order `k` linked in such a way that the root of the one with the larger root key is the leftmost child of the one with the smaller root key. A tree of order `k`has `2^k` nodes in it and a height of `k`. At each depth `i`, the number of nodes is $k \choose i$. The children of a root of an order `k` binomial tree are themselves trees of order $k-1, k-2 \ldots 0$, with the leftmost child being the tree of order `k - 1`.

```rust
/// WIP
```

## Binomial Trees & Binary Representation

Binomial heaps should be viewed as isometries of the binary representation of numbers. For instance, the binary representation of 13 is $(1101)_b = 2^3 + 2^2 + 2^0$. Analogously, a binary heap of `13` elements will contain `3` binomial trees of order $\left<3, 2, \text{ and } 0\right>$. In binomial heaps, we thus represent data in packets whose sizes are powers of `2`. In a heap with `n` items, we can have at most $\lfloor \log_2 n \rfloor + 1$  packets or trees. If each tree provides $\mathcal{O}(1)$ access to its min value — as binomial trees do, then we can find the min value of a heap in logarithmic time. After deleting an element from a packet, i.e a binomial tree, the size is no longer a power of 2.

To restore the invariant, we use the observation that $2^k - 1 = \prod_{i = 0}^{k - 1} 2^i$ to fracture a packet and recreate valid binomial trees out of them. Doing this, however, may violate the invariant that there can only be at most 1 tree of each order. To solve this, we have to merge trees of the same order. Fusing two trees of order $$k-1$$ results in a tree of order `k`

## Merging and Splitting Trees

```rust
/// WIP
```

## The Binomial Heap

A Binomial heap is a forest of Binomial Trees stored in ascending order of size.

### Union & Enqueue

Just as the children of a given node are linked together into a linked list via the right_sibling pointer, the roots of the heap are also linked together in a root list.These roots are organized such that their degrees increase as we traverse across the root list. A heap is accessed via the head of the root list. Thus to find the minimum of a heap, we simply search a list whose length can be at most $\lg_2n$. To meld/union two heaps, we begin by combining the root lists of the two heaps into a single linked list sorted by degree. This merge procedure is akin to the merge step of merge sort. After merging, we may have broken the invariant that we can only have at most one root of a particular degree. To restore it, we run a procedure to repeatedly merge trees of the same order until the invariant is restored.

```rust
/// WIP
```

### Finding the Minimum

Just as the children of a given node are linked together into a linked list via the `right_sibling` pointer, the roots of the heap are also linked together in a root list.These roots are organized such that their degrees increase as we traverse across the root list. A heap is accessed via the head of the root list. Thus to find the minimum of a heap, we simply search a list whose length can be at most $\lg_2n$. To meld/union two heaps, we begin by combining the root lists of the two heaps into a single linked list sorted by degree. This merge procedure is akin to the merge step of merge sort. After merging, we may have broken the invariant that we can only have at most one root of a particular degree. To restore it, we run a procedure to repeatedly merge trees of the same order until the invariant is restored.

To insert a node, we simply create a binomial tree of order `0`, with the element we wish to insert and then union the singleton tree with the heap.

The decrease key procedure is similar to the decrease key procedure for normal binary heaps. we simply change the value of the node to the new lower value and then bubble it up until the `min-heap` invariant is restored.

Extract min is also quite simple, we first find the minimum value by searching the root list. We then remove it from the heap leaving  the linked list of its children exposed. We reverse this linked list — doing so turns the child subtrees into a binomial tree of order `k - 1`. We meld this tree with the rest of the heap.
  
To delete from a binomial heap, we simply decrease the key of the node  we want to delete to the lowest possible value. We then run extract min on the heap. All these operations run in $\mathcal{O}(\lg_2 n)$.

What we have described above can be thought of as eager binomial heaps — each operation that may break an invariant promptly does the work to clean up the mess to restore it. We can get a heap with better than logarithmic performance by using an amortized version of the binomial heap where the work of restoring invariants is deferred until extract min is called. This is a Lazy Binomial Heap. Remember that in an amortized data structure, some operations can take much longer as long as previous operations did not take too long to finish. In a lazy binomial heap, we store the root list as a circular doubly linked list which we access via the node with the `min` value. To merge two heaps, we simply link their root list — we do not do any clean up after that. This gives us $\mathcal{\Theta}(1)$ for union. This also means that enqueue, which is defined in terms of meld, also takes constant time. Finding the minimum is obviously a constant operation — because we access the root list through it. We do all of the deferred work during the call to `extract-min`. We coalesce binomial trees in the heap to ensure that we have at most 1 tree of each order.

In the coalesce step, since the root list is not kept in sorted order, we use a variant of counting sort to map each tree to a direct addressable array with $\lceil \log_2(n + 1) \rceil$ slots. For each slot in the array, while there’s 2 or more trees in that slot, we merge them and place the result one bucket higher. Informally, because the number of treed grows slowly — `1` per enqueue, and drops precipitously once we run extract min, the amortized cost of `extract_min`  and `delete` — which is implemented using `extract_min` is $\mathcal{O}(\lg_2 n)$

### Decrease Key

### Extract Min & Delete

### Amortized Analysis

#### A Word on Fibonacci Heaps
