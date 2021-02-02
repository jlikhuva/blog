# Word Level Parallelism

In this note, we shall explore a few bit level algorithms that we'll need when developing some of the specialized integer data structures. We begin by discussing a procedure that allows us to extract the first (the top) `k` bits of any integer in constant time. We then proceed to discuss procedures that allow us, in constant time, to operate on integers in parallel.

## Finding the `top(k)` bits of an integer
The first procedure is quite simple. The goal is, given a number `x` and a length `k`, to extract the first `k` bits of `x` in `O(1)`. A procedure that does this will be handy when implementing the x-fast trie. Below, we implement this procedure.

```rust
const USIZE_BITS: usize = 64;
pub fn top_k_bits_of(x: usize, k: usize) -> usize {
    assert!(k != 0);
    let mut mask: usize = 1;

    // Shift the 1 to the index that is `k`
    // positions from the last index location.
    // That is `k` away from 64
    mask <<= USIZE_BITS - k;

    // Turn that one into a zero. And all
    // the other 63 zeros into ones.
    mask = !mask;

    // I think this is the most interesting/entertaining part.
    // Adding a one triggers a cascade of carries that flip all
    // the bits (all ones) before the location of the zero from above into
    // zeros. The cascade stops when they reach the zero from
    // above. Since it is a zero, adding a 1 does not trigger a carry
    //
    // In the end, we have a mask where the top k bits are ones
    mask += 1;

    // This is straightforward
    x & mask
}
```

You can play around with the code so far [in the playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=186242f8f5e9267430438fcb3119606c)

## Background

Arithmetic and logical operations take, for all intents and purposes, constant time. Such operations operate on whole words.  (A word is the size of a single memory segment. In this exposition, we assume a word size width of `64`. For a more in-depth discussion of computer memory, refer to [this note](https://akkadia.org/drepper/cpumemory.pdf)). For instance, it takes constant time to add two `64` bit numbers. The central idea of the methods we're about to discuss is this: If you have a bunch of small integers -- each smaller that sixty four bits, e.g. a bunch of bytes, we can pack many of them into a single sixty four bit integer. Then, we can operate on that packed integer as if it were a single number. For example, we can fit 8 byte sized numbers in a single word. By operating on the packed integer, we are in effect operating on 8 different integers in parallel. This is what we call world level parallelism. Of course there are intricate details that have been elided, in this broad description. In the next sections, we take a detailed look at those details as we flesh out the different parallel operation.

## Motivation: B-Trees

Suppose we wish to store small sized integers in a B-Tree. And suppose too that we wish to take advantage of the fact that we can fit many of these integers in a single, larger integer. How would we go about designing such a B-Tree?

Recall that a B-Tree of order `b` is a multiway search tree in which each node is a bucket that must contain between `b - 1` and `2b - 1` keys. Furthermore, each node has one more child than the number of keys it contains. That is, each node must have between `b` and `2b` child nodes. Operations on B-Trees rely on a key operation: `node.rank(x)`. This operation searches through the keys of a single node (which are sorted) and either returns the location of `x` in the bucket, or the index of the child we need to descend into in order to complete the operation at hand. In run of the mill B-Trees, `node.rank(x)` is implemented using binary search and thus takes `O(lg b)`. If we know that our keys are small integers, can we perform `node.rank(x)` in `O(1)`? It turns out we can. Let's see how.

#### Parallel Compare
The `node.rank(x)` operation depends on an even more basic operation: `compare(x, y)`. This operation simply tells us if `x >= y`. Suppose `x` and `y` are `7-bits` wide (but stored in an 8 byte integer such that the final bit is unoccupied), how could we implement `compare(x, y)`? Well, we could do it the `C-`way - by subtraction. However, instead of taking the usual route where we calculate `z = x - y` and if `z` is negative we know that `x < y` otherwise, we conclude that `x >= y`, we'll adopt a much cooler approach. In particular, we'll first set the 8th bit of `y`, the 7-bit number we are comparing against to `1`. We'll also set the 8th bit of `x` to 0. For example, suppose <!-- $x = 0000111 \text{ and } y = 1100001$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/r00ard6i01.svg">. Below, we show the effect of setting the 8th bit.
<!-- $$
\begin{aligned}
    y = 1100001 \rightarrow \textcolor{red}{1}1100001 \\
    x = 0000111 \rightarrow \textcolor{cyan}{0}0000111
\end{aligned}
$$ --> 

<div align="center"><img style="background: white;" src="../svg/NlyOijlPKY.svg"></div>

Now, if we compute `y - x`, how will that 8th bit behave? Below, we show the result of this subtraction.

<!-- $$
\begin{aligned}
    y = \textcolor{red}{1}1100001 \\
    x = \textcolor{cyan}{0}0000111 \\
\text{------------------------} \\
    x - y = \textcolor{green}{1}1011010
\end{aligned}
$$ --> 

<div align="center"><img style="background: white;" src="../svg/mi1XPKf4L7.svg"></div>

As shown above, when `y >= x` the sentinel bit in the result is turned on. Had `x` been larger than `y`, that bit would have been tuned off. 

[Explain why. Discuss the effect of borrowing]

Therefore, to compare two 7-bit integers, we set the 8-th bit to `0` in one of the numbers and to `1` in the other. We then subtract the resultant number and check the value of the `8-th` bit.

So far, we've been talking about how to compare `x` with a single `7bit` number. However, for our procedure to be useful subroutine in computing `node.rank(x)`, we need to compare `x` with `b` 7-bit numbers. Note that we should choose `b` such that it fits in 64 bits, so we choose `b=7`. That is, a single node holds 7, 7bit numbers, each represented using 8 bits. If those seven numbers are organized such that each number has an associated sentinel bit that is set to `0`, we can compare `x` to all of them by comparing the entire word with a number formed by tiling `x` 7 times. 

To recap: Fundamental Primitive: Parallel Compare
 1. Pack a list of values x₁, …, xₖ into a
machine word X, separated by 1s.
 2. Pack a list of values y₁, …, yₖ into a
machine word Y, separated by 0s.
 3. Compute X – Y. The bit preceding
xᵢ – yᵢ is 1 if xᵢ ≥ yᵢ and 0 otherwise.
Assuming the packing can be done in O(1)
time, this compares all the pairs is O(1)
machine word operations.

Explain how to read off the results
Include latex example with parallel version

## Parallel Tile

How would we use the idea above to impl rank in a b-tree

 Store the (s-bit) keys in the B-tree node in
a single (w-bit) machine word, with zeros
interspersed:

Introduce the abstraction if a sardine can

How to perform a lookup in a sardine can -- To perform a lookup for the key k, form a number by
replicating k multiple times with 1s interspersed

Subtract the B-tree key number from it to do a parallel
comparison.

Count up how many of the sentinel bits in the resulting
number are equal to 1. This is the number of keys in the
node less than or equal to k.

How to tile by multiplication

## Parallel Rank

Define rank. With rank, we first compare, then descend. In a b-tree the cost of searching within the nodes dominates `O(logb n · log b) = O(log n)`.. If we can fit all the keys in a node into O(1)
machine words, we can determine rank(k) in time O(1)
for total lookup cost of `O(logb n)`.

## Parallel Add

After performing our subtraction, we’re left with a
number like this one, where the highlighted bits
are “interesting” to us.
Goal: Add up these “interesting” values using
O(1) word operations.

This is a series of shifts and
adds. It’s equivalent to
multiplying our original
number by some well-chosen
spreader!

## `O(1)` Most Significant Bit

Mathematically, msb(n) is the largest value of k
such that 2k ≤ n. 
There’s a simple O(w)-time algorithm
We can improve this runtime to O(log w) by using a
binary search:
Lookup table

parallel pack

## `O(1)` Integer LCP

- Given two integers m and n, the longest common
prefix of m and n, denoted lcp(m, n), is the length
of the longest bitstring they both start with.
- 63 - msb(m ⊕ n)

https://rust-lang.github.io/api-guidelines/type-safety.html#newtypes-provide-static-distinctions-c-newtype

## References

1. [CS 166 Lecture 15](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/15/Slides15.pdf)
2. [CS 166 Lecture 16](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/16/Slides16.pdf)
3. [CS 166 Lecture 17](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/17/Slides17.pdf)
4. [6.851](http://courses.csail.mit.edu/6.851/fall17/scribe/lec12.pdf)