# Rusty Solutions to the RMQ Problem

<div align="right" style="text-align:right">
    <i>Okonda, Joseph L.</i>
    <br>
    <i>2021</i>
</div>

## A Naïve Solution

The most straightforward way to solve this problem is to create a lookup table with all the RMQ answers precomputed. This will allow us to answer any RMQ in constant time by doing a table lookup. How can we build such a table? The first thing to notice is that this is a discrete optimization problem - we are interested in the minimal (aka the optimal) value in a given range. A quick reference to [common algorithmic patterns](https://www.notion.so/A-note-on-algorithmic-design-patterns-20e50d39c99945e3ad8dfb804177ab3f) should tell us that we may be able to use dynamic programming to solve the problem. All we need to do is come up with an update rule. In particular, suppose our array is $A$ ,  if we know the smallest value in some range `(i, j)` to be $\alpha$, we can easily figure out the answer on a larger range $(i, j+1)$ by comparing $\alpha$ with $A[i + 1]$ . That is:

$$
RMQ_A(i, j) = \begin{cases}
      A[j], & \text{if}\ i = j \\
      \min\left(A[j], RMQ_A(i, j - 1)\right), & \text{otherwise}
\end{cases}
$$

We can do this for all possible values of `i` and `j` to fill up our lookup table. This takes quadratic time. Thus, with this approach, we cam solve the RMQ problem in $\left<\Theta(n^2), \Theta(1)\right>$.

The code below implements this approach. The only modification we make is that instead or calculating the actual minimal value, we calculate the index of the smallest value. That is $\argmin$ instead of `min`

```rust
/// An inclusive ([i, j]), 0 indexed range for specifying a range
/// query.
#[derive(Eq, PartialEq, Hash)]
pub struct RMQRange<'a, T> {
    /// The starting index, i
    start_idx: usize,

    /// The last index, j
    end_idx: usize,

    /// The array to which the indexes above refer. Keeping
    /// a reference here ensures that some key invariants are
    /// not violated. Since it is expected that the underlying
    /// array will be static, we'll never make a mutable reference
    /// to it. As such, storing shared references in many
    /// different RMQRange objects should be fine
    underlying: &'a [T]
}
```

To make it easy to construct new range objects, we implement the from trait. This implementation will allow us to construct an `RMQRange` object from a `3-Tuple`  by simply invoking `(a, b, c).into`. We also do error checking here to make sure that we can only ever create valid ranges.

```rust
impl <'a, T> From<(usize, usize, &'a [T])> for RMQRange<'a, T> {
    fn from(block: (usize, usize, &'a [T])) -> Self {
        let start_idx = block.0;
        let end_idx = block.1;
        let len = block.2.len();
        if start_idx > end_idx {
            panic!("Range start cannot be larger than the range's end")
        }
        if end_idx >= len {
            panic!("Range end cannot be >= the len of underlying array")
        }
        RMQRange {
            start_idx,
            end_idx,
            underlying: block.2
        }
    }
}
```

With the abstractions above, we can go ahead and implement our procedure.

```rust
type LookupTable<'a, T> = HashMap<RMQRange<'a, T>, usize>;

/// Computes the answers to all possible queries. Since the ending index of a query
/// is lower bounded by the starting index, the resulting lookup table is an
/// upper triangular matrix. Therefore, instead of representing it as a matrix,
/// we use a hashmap instead (to save space)
fn compute_rmq_all_ranges<T: Hash + Eq + Ord>(array: &[T]) -> LookupTable<'_, T> {
    let len = array.len();
    let mut lookup_table = HashMap::with_capacity((len * len) / 2);
    for start in 0..len {
        for end in start..len {
            if start == end {
                lookup_table.insert((start, end, array).into(), start);
            } else {
                let prev_range = (start, end - 1, array).into();
                let new_range = (start, end, array).into();
                let mut min_idx = *lookup_table.get(&prev_range).unwrap();
                if array[min_idx] > array[end] {
                    min_idx = end;
                }
                lookup_table.insert(new_range, min_idx);
            }
        }
    }
    lookup_table
}
```

You can play around with the code so far [in the playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=ccb49819827b6e1834765389f7ecf12b)

Can we do better than $\left<\Theta(n^2), \Theta(1)\right>$ ? The query time is the best we can ever hope for. However, we can reduce the processing time. Let's see how we can do that in the next section.

## Binary Representation & Sparse Tables

Any positive integer can be factored into a sum of powers of two. This binary factorization is the basis of binary representation. For instance, the decimal number 19 can be represented as <!-- $19 = 16 + 2 + 1 = 2^4 + 0*2^3 + 0*2^2 + 2^1 + 2^0 = (10011)_2$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/5zxNilT8JM.svg">. Given some range `[i, j]` we know that its length, `(j - i) + 1` is positive. We can therefore factor it using binary factorization to get shorter ranges. For instance, if our range is `(0, 18)` we see that it has a length of `19` which, as we saw above, can be be factored into `(0, 15) + (16, 17) + (18, 18)`. 

### Preprocessing

How can we use these observations to construct a solution to our problem? First, note that powers of two are sparsely distributed among positive integers. Also, because they can be combined to form any other number, if we had a table with answers to all possible ranges whose size is a power of two, we would be able to get answers for any range. How can we construct such sparse table? For an array of length `k`, there are `O(lg k)`  ranges whose size is a power of two (just as there are `lg x` bits in the binary representation of `x`).We shall thus construct the sparse table by computing answers to all `lg k` ranges for all `n` possible values of `k`. Therefore, the time needed to create the sparse table is <!-- $\mathcal{O}(n \log n)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/CEiOpX0Hzv.svg">.We implement this scheme below.

```rust
/// An index into our sparse table
pub struct SparseTableIdx {
    /// The index where the range in question begins
    start_idx: usize,

    /// The length of the range. This has to always be a power of
    /// two
    len: usize
}

impl From<(usize, usize)> for SparseTableIdx {
    fn from(idx_tuple: (usize, usize)) -> Self {
        let start_idx = idx_tuple.0;
        let len = idx_tuple.1;
        if !len.is_power_of_two() {
            panic!("Expected the length to be a power or 2")
        }
        SparseTableIdx { start_idx, len }
    }
}
```

```rust
/// A sparse table is simply a collection of rmq answers for ranges whose
/// length is a power of two. We precompute such ranges for all possible starting
/// positions
type SparseTable<'a, T> = HashMap<SparseTableIdx, RMQResult<'a, T>>;

/// The DP update rule for populating the sparse table. The
/// smallest value in a range whose length is 1 << k is
/// the min of the two ranges that form that range
/// each of length 1 << (k - 1)
fn get_prev_min<'a, T: Hash + Eq + Ord>(
    array: &'a [T],
    left_res: &RMQResult<T>,
    right_res: &RMQResult<T>,
) -> RMQResult<'a, T> {
    if left_res.min_value < right_res.min_value {
        (left_res.min_idx, &array[left_res.min_idx]).into()
    } else {
        (right_res.min_idx, &array[right_res.min_idx]).into()
    }
}

/// For each index `i`, compute RMQ answers for ranges starting at `i` of
/// size `1, 2, 4, 8, 16, …, 2^k` as long as the resultant ending index
/// fits in the underlying array in the array.
/// For each array index, we compute lg n ranges. Therefore,
/// the total cost of the procedure is O(n lg n)
fn compute_rmq_sparse<'a, T: Hash + Eq + Ord>(array: &'a [T]) -> SparseTable<'a, T> {
    let len = array.len();
    let mut sparse_table = HashMap::new();
    let mut power = 0;
    while 1 << power <= len {
        let mut start_idx = 0;
        let mut end_idx = start_idx + (1 << power) - 1;
        while end_idx < len {
            if start_idx == end_idx {
                let idx = (start_idx, 1 << power).into();
                let rmq_res = (start_idx, &array[start_idx]).into();
                sparse_table.insert(idx, rmq_res);
            } else {
                let idx = (start_idx, 1 << power).into();
                let prev_len = 1 << (power - 1);
                let left: SparseTableIdx = (start_idx, prev_len).into();
                let right: SparseTableIdx = (start_idx + prev_len, prev_len).into();
                let left_res = sparse_table.get(&left).unwrap();
                let right_res = sparse_table.get(&right).unwrap();
                let rmq_res = get_prev_min(array, left_res, right_res);
                sparse_table.insert(idx, rmq_res);
            }
            start_idx += 1;
            end_idx += 1;
        }
        power += 1;
    }
    sparse_table
}

```

### Querying the Sparse Table

Now that we have our sparse table, how can we query from it given an arbitrary range `R = [i, j]`? From our initial discussion of binary factorization, you can imagine computing all sub-ranges of `R` whose length is a power of 2 and then taking the min over these values. For an arbitrary length `n`, there are <!-- $O(\lg n)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/wfptrCDR1m.svg"> such sub-ranges. Thus, this scheme would give us a <!-- $\left<\Theta(n\lg n), \Theta(\lg n)\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/Z802R6ewbT.svg"> solution to the `RMQ`problem. 

Computing all sub-ranges, however, is overkill. All we need are two sub-ranges that fully cover the underlying segment. How do we find the two covering segments? First, observe that if the length of the range is an exact power of two, then we do not need to do any further computation since we already precomputed answers for all such ranges. If its not, we start by finding the largest sub-range that is an exact power of two. Specifically, we find the value `k` such that <!-- $2^k \leq (j - i) + 1$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/SyXeSDNkuL.svg">. Note that this value `k` is the index of the most significant bit of the range's length. The first range is thus <!-- $[i, i + 2^k - 1]$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/RsDV0G7oU5.svg">. That is, a range whose length is `2^(k) - 1`. After finding the largest sub-range, the remaining portion's length certainly be not be a power of two. To proceed, we use a neat trick: we construct a range whose length is the smallest power of two larger than the remaining portion length. To prevent this sub-range from overflowing the underlying range, we shift it over to the left, overlapping the first subr-ange, until it is full contained in the original range. The second range is thus <!-- $[j - 2^k + 1, j]$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/2eOUcbNTa4.svg">, with a length of `2^(k) - 1` as well.

To recapitulate, we query from the sparse table by finding the `argmin` of two overlapping ranges whose answers have already been computed. Figuring out which ranges to use involves finding the `MSB(n)` where `n` is the length of the range in the query. How so we calculate `MSB(n)`? To compute `MSB(n)` in constant time, we can use a lookup table. Later on, when discussing specialized integer containers, we'll implement a complex but straightforward method for finding `k` in constant time. For now, a lookup table suffices. Thus, with this scheme, we have a <!-- $\left<\Theta(n\lg n), \Theta(1)\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/WpCYNtMQVL.svg"> solution to the `RMQ` problem. Below, we implement a procedure to compute the lookup table.

```rust
/// Here's the scheme we shall use to implement the lookup table:
///     - First, we shall assume that the values we get are 8 bytes (64bits) wide
///     - We shall pre-compute all MSB(n) values for all n <= 2^16. This will use
///       65536 bytes which is approximately 66Kb.
///     - To find the MSB of any value, we combine answers from the 4 16 bit
///       portions using logical shifts and masks
pub struct MSBLookupTable([u8; 1 << 16]);

impl MSBLookupTable {
    /// Build the lookup table. To fill up the table, we simply subdivide
    /// it into segments whose sizes are powers of two. The MSBa in a segment
    /// are the same. For instance:
    ///   ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    /// n       |1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16|17|18|19|...
    /// msb(n)  |0|1|1|2|2|2|2|3|3|3 |3 |3 |3 |3 |3 |4 |4 |4 |4 |...
    ///   ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    pub fn build() -> Self {
        let mut lookup_table = [0; 1 << 16];
        let mut msb_idx = 0;
        for i in 0..lookup_table.len() {
            let n = i + 1;
            if n > 1 && n.is_power_of_two() {
                msb_idx += 1;
            }
            lookup_table[i] = msb_idx;
        }
        MSBLookupTable(lookup_table)
    }

    /// Get the most significant bit of the given 64 bit value. A 64 bit
    /// bit number can be subdivided into 4 16 bit portions. Since we have
    /// pre-calculated the msb values for all possible 16 bit integers,
    /// we can find the msb of the number by combining answers to each segment
    pub fn get_msb_idx_of(&self, n: usize) -> u8 {
        debug_assert!(n != 0);
        let bit_mask = 0xFFFF;
        if n >> 48 > 0 {
            let d_idx = (n >> 48) - 1;
            self.0[d_idx] + 48
        } else if n >> 32 > 0 {
            let c_idx = ((n >> 32) & bit_mask) - 1;
            self.0[c_idx] + 32
        } else if n >> 16 > 0 {
            let b_idx = ((n >> 16) & bit_mask) - 1;
            self.0[b_idx] + 16
        } else {
            let a_idx = (n & bit_mask) - 1;
            self.0[a_idx]
        }
    }
}
```
Once again, the query time is the best possible. However, even though the pre-processing time reduced from quadratic to `O(n lg n)`, we can still do better. In particular, we can shave off a log factor and arrive at a linear time pre-processing algorithm. To figure out how to do that, we shall take a detour to discuss the method of four russians.

If you'd like to take a breather, feel free to play around with the sparse table code in [the rust playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=7f9c152dee95816d7ef8ef9d14bc1f72).

---

##### The Method of Four Russians
We begin this detour by taking another detour. Let us discuss the algorithms used to find the median (or more generally, the `i_th` order statistic) of a collection of pairwise comparable items. `Quickselect` can solve this problem in expected linear time. However, if we want a worst case linear time solution, we need to use the `Median of Medians` procedure.

`MoM` is exactly similar `Quickselelect`  except, instead of randomly picking the index to partition around, we compute an approximate median value. We begin by dividing the input collection into blocks of `length=5`. This gives us <!-- $\lceil \frac{n}{5}\rceil$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/pF1PFO71La.svg"> blocks, with the final block possibly having `< 5` items. For each block, we calculate the median by first sorting and selection the lower median. For a single block, this always takes constant time, meaning that finding the median for all blocks takes linear time. We aggregate all the block-level medians into a single array. This array is of length <!-- $\lceil \frac{n}{5}\rceil$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/pF1PFO71La.svg">. Once we have aggregated the lock level medians, we are faced with the xeact same problem we started with -- just on a much smaller array. Therefore, we can recursively find the median of this new array. Once we have this value, we can proceed as usual, using the [prune and conquer](https://www.notion.so/A-note-on-algorithmic-design-patterns-20e50d39c99945e3ad8dfb804177ab3f) strategy. Below, we implement this scheme

```rust
/// The abstraction for a single block.
pub struct MedianBlock<'a, T> {
    /// The starting index. This is 0-indexed and should be
    /// less than or equal to the end_idx
    start_idx: usize,

    /// The ending index. This should be strictly less than the
    /// length of the underlying array. Further, end_idx - start_idx should
    /// be 5 for all except possibly the last block
    end_idx: usize,

    /// The index of the median value in the given range. To move from this
    /// index to an indx in the underlying, we simply calculate
    /// `start_idx + median_idx`
    median_idx: usize,

    /// The median of this block
    median: &'a T,
}

/// For quick construction and error checking. (start_idx, end_idx, underlying, median_idx)
impl<'a, T> From<(usize, usize, &'a [T], usize)> for MedianBlock<'a, T> {
    fn from(block: (usize, usize, &'a [T], usize)) -> Self {
        let start_idx = block.0;
        let mut end_idx = block.1;
        let len = block.2.len();
        let median_idx = block.3;
        if end_idx >= len {
            end_idx = len - 1;
        }
        debug_assert!(start_idx < end_idx);
        debug_assert!(median_idx >= start_idx && median_idx <= end_idx);
        if end_idx < len {
            debug_assert!(end_idx - start_idx == 5);
        }
        MedianBlock {
            start_idx,
            end_idx,
            median_idx,
            median: &block.2[median_idx]
        }
    }
}
```

With the above abstractions in place, we can go ahead and implement the main procedure

```rust
//! Median of Medians Helper Functions. These will come in handy when implementing
//! multi-level data structures that use the method of four russians

/// Block partitioning and aggregation. This is the same for all
/// Method of Four Russians Algorithms
fn generate_macro_array<'a, T: Ord>(array: &'a [T]) -> Vec<MedianBlock<'a, T>> {
    let mut blocks = Vec::with_capacity(array.len()/5);
    for start_idx in (0..array.len()).step_by(5) {
        let end_idx = (start_idx + 5) - 1;
        let block = &array[start_idx..=end_idx];
        let median_idx = get_kth_by_sorting(block, block.len() / 2);
        blocks.push((start_idx, end_idx, array, median_idx).into())
    }
    blocks
}

/// This solves the problem for a single block. This changes from problem to problem.
/// For instance, when we solving RMQ, this will be get_min_idx_by_scanning. You can
/// definitely think of a way of creating an abstraction that can solve any
/// method of four russians problem by having the client provide a function for
/// block partitioning and solving the block level problem. 
fn get_kth_by_sorting<T: Ord>(block: &[T], k: usize) -> usize {
    let kth = block
        .iter()
        .enumerate()
        .sorted_by_key(|x| x.1)
        .map(|x| x.0)
        .nth(k);
    kth.unwrap()
}

/// Reorient the elements of the array around the element at `pivot_idx` and
/// return the length of the left partition
fn partition_at_pivot<T: Ord>(array: &mut [T], pivot_idx: usize) -> usize {
    let last_idx = array.len() - 1;
    array.swap(pivot_idx, last_idx);
    let mut less_than_tail = 0;
    for cur_idx in 0..last_idx {
        if array[cur_idx] <= array[last_idx] {
            array.swap(less_than_tail, cur_idx);
            less_than_tail += 1;
        }
    }
    array.swap(less_than_tail, last_idx);
    less_than_tail + 1
}

/// recursively calculate the median of the median blocks. This will give us an
/// approximate median that guarantees us a roughly even split
fn get_approx_median_idx<'a, T: Ord + Clone>(macro_array: &mut [MedianBlock<'a, T>]) -> usize {
    let median_pos = macro_array.len() / 2;
    let mut medians: Vec<_> = macro_array.iter().map(|x| x.median.clone()).collect();
    let approx_median = kth_order_statistic(&mut medians, median_pos);
    let mut median_idx = 0;
    for block in macro_array {
        if block.median == approx_median {
            median_idx = block.start_idx + block.median_idx;
        }
    }
    median_idx
}

/// Computes the index of the k-th smallest element in the `array`. This is sometimes
/// referred to as the k-th order statistic. This procedure computes this value in
/// O(n). Note that this is a more general method for finding the median i.e the
/// (n/2)-th order statistic
fn kth_order_statistic<'a, T: Ord + Clone>(array: &'a mut [T], k: usize) -> &T {
    match 5.cmp(&array.len()) {
        Ordering::Less | Ordering::Equal => &array[get_kth_by_sorting(array, k)],
        Ordering::Greater => {
            let macro_array = generate_macro_array(array);
            let approx_median_idx = get_approx_median_idx(macro_array.as_slice());
            let left_range_size = partition_at_pivot(array, approx_median_idx);
            match k.cmp(&left_range_size) {
                Ordering::Equal => &array[k - 1],
                Ordering::Less => kth_order_statistic(&mut array[..left_range_size], k),
                Ordering::Greater => {
                    kth_order_statistic(&mut array[left_range_size..], k - left_range_size)
                }
            }
        }
    }
} 
```
You can play around with the code for computing the `kth_order_statistic` [in the playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=02f915a79be3e7b6aadf53cfc1f29156).

---
The median of medians procedure has a few key structures:
* The input is divided into blocks or equal size. This is called block partitioning and each block is called the micro array.
* The original problem (median in this case) is solved for each block using a naive method that works well for small input sizes. With this scheme, we are able to solve the problem for each block in constant time and for all blocks in linear time.
* The solutions to all blocks are aggregated into a single array. We call this the macro array. The macro array, just like the micro arrays, are smaller instances of the original problem.
* By combining, in some bespoke fashion, the macro and micro array solutions, we are able to solve the original problem with a log factor shaved off. In `MoM` we went from `Quickselect's` `O(n lg n)` to `O(n)` (For a rigorous runtime analysis of the median of medians method, please refer to CLRS chapter 9).  
  
The structures above are the four major motifs in the method of four russians. How can we use this method to reduce the pre-processing time of our RMQ algorithm? We discuss that after the following interlude.

---
Thus far, we've implemented procedures to solve the `rmq` problem as free standing functions. Before we move forward, lets take a step back and see if we can come up with a much more elegant abstraction that unifies all the different solution methods. This will become more crucial as we start talking about 2-level structures that use multiple solution methods.

```rust
#[derive(Debug, Eq, PartialEq, Hash)]
pub struct SparseTableIdx {
    /// The index where the range in question begins
    start_idx: usize,

    /// The length of the range. This has to always be a power of
    /// two
    len: usize,
}

type DenseTable<'a, T> = HashMap<RMQRange<'a, T>, RMQResult<'a, T>>;
type SparseTable<'a, T> = HashMap<SparseTableIdx, RMQResult<'a, T>>;

#[derive(Debug)]
pub struct RMQResult<'a, T> {
    min_idx: usize,
    min_value: &'a T,
}

impl<'a, T> From<(usize, &'a T)> for RMQResult<'a, T> {
    fn from((min_idx, min_value): (usize, &'a T)) -> Self {
        RMQResult { min_idx, min_value }
    }
}

/// All structures capable of answering range min queries should
/// expose the solve method.
pub trait RMQSolver<'a, T: Ord> {
    fn solve(&self, range: &RMQRange<'a, T>) -> RMQResult<T>;
}
```

We introduce a trait that encodes the necessary and sufficient API that any  `rmq` solver should expose. We need to be able to build the solver and to invoke the solve method with a given range. Below, we introduce the various solvers, all of which we have already seen before -- we simply present them here in a unified manner.

```rust

/// A solver that answers range min queries by  doing no preprocessing. At query time, it
/// simply does a linear scan of the range in question to get the answer. This is an
/// <O(1), O(n)> solver
#[derive(Debug)]
pub struct ScanningSolver<'a, T> {
    underlying: &'a [T],
}

impl<'a, T> ScanningSolver<'a, T> {
    pub fn new(underlying: &'a [T]) -> Self {
        ScanningSolver { underlying }
    }
}

/// A solver that answers `rmq` queries by first pre-computing
/// the answers to all possible ranges. At query time, it simply
/// makes a table lookup. This is the <O(n*n), O(1)> solver
#[derive(Debug)]
pub struct DenseTableSolver<'a, T> {
    underlying: &'a [T],
    lookup_table: DenseTable<'a, T>,
}

impl<'a, T: Ord + Hash> DenseTableSolver<'a, T> {
    pub fn new(underlying: &'a [T]) -> Self {
        let lookup_table = compute_rmq_all_ranges(underlying);
        DenseTableSolver {
            underlying,
            lookup_table,
        }
    }
}

/// A solver that answers rmq queries by first precomputing
/// the answers to ranges whose length is a power of 2
/// At query time, it uses a lookup table of `msb(n)` values to
/// factor the length of the requested query into powers of
/// 2 and then looks up the answers in the sparse table.
/// This is the <O(n lg n), O(1)> solver
#[derive(Debug)]
pub struct SparseTableSolver<'a, T> {
    underlying: &'a [T],
    sparse_table: SparseTable<'a, T>,

    /// The precomputed and cached array of msb(n)
    /// answers for all n between 1 and 1 << 16
    msb_sixteen_lookup: [u8; 1 << 16],
}

impl<'a, T: Ord + Hash> SparseTableSolver<'a, T> {
    pub fn new(underlying: &'a [T]) -> Self {
        let sparse_table = compute_rmq_sparse(underlying);
        let msb_sixteen_lookup = MSBLookupTable::build();
        SparseTableSolver {
            underlying,
            sparse_table,
            msb_sixteen_lookup,
        }
    }
}
```

Below, we implement the `RMQSolver` trait for each of our solvers. We leverage functions that we already implemented in preceding segments.

```rust
/// Calculates the location and value of the smallest element
/// in the given block by iterating over the elements. This takes
/// linear time and is extremely fast for small block sizes (or
/// more generally, blocks that can fit in a single cache line)
fn get_min_by_scanning<T: Ord>(block: &[T]) -> RMQResult<T> {
    let (min_idx, min_value) = block.iter().enumerate().min_by_key(|x| x.1).unwrap();
    RMQResult { min_idx, min_value }
}

impl<'a, T: Ord> RMQSolver<'a, T> for ScanningSolver<'a, T> {
    fn solve(&self, range: &RMQRange<'a, T>) -> RMQResult<T> {
        let range_slice = &self.underlying[range.start_idx..=range.end_idx];
        get_min_by_scanning(range_slice)
    }
}

impl<'a, T: Ord + Eq + Hash> RMQSolver<'a, T> for DenseTableSolver<'a, T> {
    fn solve(&self, range: &RMQRange<'a, T>) -> RMQResult<T> {
        let res = self.lookup_table.get(&range).unwrap();
        (res.min_idx, res.min_value.clone()).into()
    }
}

impl<'a, T: Ord + Eq + Hash> RMQSolver<'a, T> for SparseTableSolver<'a, T> {
    fn solve(&self, range: &RMQRange<'a, T>) -> RMQResult<T> {
        let (i, j) = (range.start_idx, range.end_idx);
        let range_len = (j - i) + 1;
        let k = self.msb_sixteen_lookup.get_msb_idx_of(range_len);
        let right_start = j - (1 << k) + 1;
        let left: SparseTableIdx = (i, 1 << k).into();
        let right: SparseTableIdx = (right_start, 1 << k).into();
        let left_res = self.sparse_table.get(&left).unwrap();
        let right_res = self.sparse_table.get(&right).unwrap();
        get_prev_min(self.underlying, left_res, right_res)
    }
}
```

---

## Two-Level Structures

To apply the method of four russians to the RMQ problem, we begin by dividing the input array into blocks of length <!-- $b$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/fCJxfN2Ysc.svg">. If the length of the array is <!-- $n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/k2Z6CavdDQ.svg">, this results in <!-- $\mathcal{O}(\frac{n}{b})$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/gwfsqBuBks.svg"> blocks. For each of these blocks, we find the index of the smallest value bu doing a simple scan. This takes <!-- $\mathcal{O}(b)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/Dqg8jnCq0o.svg"> in each block and <!-- $\mathcal{O}(\frac{n}{b}) * \mathcal{O}(b) = \mathcal{O}(n)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/oI85zP1xyG.svg"> for all the blocks. We aggregate these min values in a new macro array. Given a query range `[i, j]` how can we use the blocks and the macro array to satisfy the query? Also, what value of `b` should we use? To query, we start by figuring out which block the ends of the query fall into. We do that by dividing each end with the block size, i.e `start_block = i/b, end_block = j/b`. We then scan the items in `start_block` that appear after `i` and the items in `end_block` that appear before `j` and take the minimal value over them. Let's call this value, the smallest value at the ends of the range, <!-- $\lambda$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/w3irZZ0Jg1.svg"> Then we scan the macro array to find the minimal value among all blocks between `start_block` and `end_block`. let's call this value <!-- $\alpha$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/6bXEiZZAEX.svg">. The answer to our query is the `min` (or `argmin`) between these two values: <!-- $RMQ_A(i, j) = \min(\lambda, \alpha)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/KHSI90DMS3.svg"> How long does this take? Well, finding the `min` in the end blocks take `O(b)` and scanning the intermediate blocks takes `O(n/b)`. This gives us `O(b + n/b)`. Therefore, to properly characterize the runtime, we need to find the value of `b` that minimized the expression `b + n/b`. We do so below
<!-- $$
\begin{array}{c}
    f(b) = b + \dfrac{n}{b} \\
    f'(b) = 1 - \dfrac{n}{b^2}\\
    0 = 1 - \dfrac{n}{b^2}\\
    b^2 = n \\
    b = \sqrt{n}
\end{array}
$$ -->

<div align="center"><img style="background: white;" src="../svg/7i3PLJolgh.svg"></div>

So, we set `b` to the square root of `n`. This gives us a query time of <!-- $\mathcal{O}(\sqrt{n})$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/qSsMnsGYC0.svg"> and an overall time of <!-- $\left<O(n), O(n^{0.5})\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/cgFTCzGhAr.svg">. 

Since our two level structure solutions will eventually mix and match the solvers that they use at each level, we begin by introducing an abstraction to facilitate that. Below, we implement an object that can answer any range min query using parameters that can be set by the client. 

```rust
/// The abstraction for a single block.
#[derive(Debug)]
pub struct RMQBlock<'a, T> {
    /// The starting index. This is 0-indexed and should be
    /// less than or equal to the end_idx
    start_idx: usize,

    /// The ending index. This should be strictly less than the
    /// length of the underlying array. Further, end_idx - start_idx should
    /// be 5 for all except possibly the last block
    end_idx: usize,

    /// The index of the median value in the given range. To move from this
    /// index to an idx in the underlying, we simply calculate
    /// `start_idx + median_idx`
    min_idx: usize,

    /// The median of this block
    min: &'a T,
}

impl <'a, T: Ord> From<(usize, usize, RMQResult<'a, T>)> for RMQBlock<'a, T> {
    fn from((start_idx, end_idx, res): (usize, usize, RMQResult<'a, T>)) -> Self {
        RMQBlock {
            start_idx,
            end_idx,
            min_idx: res.min_idx,
            min: res.min_value
        }
    }
}
/// The primary solvers available.
pub enum RMQSolverKind {
    ScanningSolver,
    DenseTableSolver,
    SparseTableSolver,
}

type BlockLevelSolvers<'a, T> = HashMap<RMQBlock<'a, T>, Box<dyn RMQSolver<'a, T>>>;

/// Since we unified our various solve, we can succinctly represent
/// a solver that follows the method of four russians scheme.
/// Notice how we allow one to set the block_size, and solvers
pub struct FourRussiansRMQ<'a, T: Ord> {
    /// This is the entire array. The solvers only operate on slices of
    /// this array
    static_array: &'a [T],

    /// As discussed already, block decomposition is at the
    /// heart of the method of four russians. We thus
    /// allow the client to set how large a single block should be
    block_size: usize,

    /// We call the solve method of this object when we want to
    /// answer an `rmq` query over the macro array
    macro_level_solver: Box<dyn RMQSolver<'a, T>>,

    /// We call the solve method of this object when we want to
    /// answer an `rmq` query over a single block (ie a micro array)
    block_level_solvers: BlockLevelSolvers<'a, T>,
}

impl<'a, T: Ord> FourRussiansRMQ<'a, T> {
    /// Create a new RMQ solver for `static_array` that uses block decomposition
    /// with a block size of `b`. The solver will use the `macro_solver` to
    /// solve the instance of the problem on the array of aggregate solutions from
    /// the blocks and `micro_solver` to solve the solution in each individual block
    pub fn new(
        static_array: &'a [T],
        block_size: usize,
        macro_solver: RMQSolverKind,
        micro_solver: RMQSolverKind,
    ) -> Self {
        let blocks = Self::create_blocks(static_array, block_size);
        let macro_level_solver = Self::create_macro_solver(blocks, macro_solver);
        let block_level_solvers = Self::create_micro_solvers(static_array, block_size, micro_solver);
        FourRussiansRMQ {
            static_array,
            block_size,
            macro_level_solver,
            block_level_solvers,
        }
    }

    fn create_blocks(array: &[T], b: usize) -> Vec<RMQBlock<'_, T>> {
        let mut blocks = Vec::<RMQBlock<T>>::with_capacity(array.len() / b);
        for start_idx in (0..array.len()).step_by(b) {
            let end_idx = (start_idx + b) - 1;
            let block = array[start_idx..=end_idx].as_ref();
            let rmq_res = get_min_by_scanning(block);
            blocks.push((start_idx, end_idx, rmq_res).into())
        }
        blocks
    }

    fn create_macro_solver(array: Vec<RMQBlock<'a, T>>, kind: RMQSolverKind) -> Box<dyn RMQSolver<'_, RMQBlock<'_, T>>> {
        todo!()
    }

    fn create_micro_solvers(array: &[T], b: usize, kinds: RMQSolverKind) -> BlockLevelSolvers<'_, T> {
        todo!()
    }

    /// Find the smallest element in the range provided (i, j). This works by finding the
    /// minimum among three answers:
    ///     (a) The smallest value in the valid portion of i's block
    ///     (b) The smallest value in the valid portion of j's block
    ///     (c) The smallest value in the intermediate blocks
    pub fn rmq(range: RMQRange<'a, T>) -> RMQResult<'a, T> {
        todo!()
    }
}
```

With the above abstraction in place, we can implement the two-level solution discussed in the preceding section as an instance of the `FourRussiansRMQ` with both `macro_solver` and `micro_solver` set to `ScanningSolver` and `b` set to `sqrt(n)`. We do so below.

```rust
/// WIP: The FourRussiansRMQ that uses the ScanningSolver 
/// for both the micro and macro arrays along with a block
/// size of sqrt(n)
```

So, Block decomposition allowed us to have linear pre-processing time. However, in the process, we lost our constant query time? Can we do better than <!-- $\sqrt{n}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/k4Mu0nuOVP.svg"> while still maintaining a linear pre-processing time? Yes. We can use a mix of block decomposition and sparse tables to achieve this. Let's see how. 

### Hybrid Structures

When discussing block decomposition, after decomposing the input into micro arrays, we went ahead and solved the original problem on each block, treating each as a reduced instance of the original. We also did the same for the macro array. In the preceding section, we solved the problem by doing a linear scan. We can, however, use methods from previous sections -- sparse and dense lookup tables to solve the problem on the micro and macro arrays. When we do that, we end up with hybrid solutions that have faster query times. In this section, we shall explore a few hybrid structures and characterize their runtime.

To create a hybrid structure, we need to decide which method we want to use to solve the problem on the macro array and on each micro array. By mixing and matching methods, we get different hybrids with different runtimes as shown in the table below.
|Block Size|Macro Array Method|Micro Array Method|Runtime|
|----------|------------------|------------------|-------|
|<!-- $\lg n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/VfomraIoyi.svg">   | Sparse Table | Linear Scan |<!-- $\left<O(n), O(\lg n\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/iVybTYrhUA.svg">|
|<!-- $\lg n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/VfomraIoyi.svg">    | Sparse Table | Sparse Table |<!-- $\left<O(n \lg \lg n), O(1\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/0m8X2tOgH1.svg">|
|<!-- $\lg n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/VfomraIoyi.svg">    | The first hybrid in this table| Sparse Table|<!-- $\left<O(n), O(\lg \lg n\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/cUT7DO2po5.svg">|

Below, we implement the first hybrid method

```rust
/// WIP: The FourRussiansRMQ that uses the ScanningSolver 
/// for both the micro and a SparseTableSolver for the macro 
/// array along with a block size of lg (n)
```
By this point we have a cool and quite efficient algorithm for the offline range min query problem. However, the title of the note did promise an `<O(n), O(1)>` solution. We discuss that in the next section with the caveat that the added constant factors that give us asymptotic constant query time may slow down the algorithm in practice. As noted [here](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/01/Small01.pdf), the preceding `<O(n), O(lg)>` hybrid solution outperforms the `<O(n), O(1)>` solution in practice.

### Cartesian Trees & The LCA-RMQ Equivalence

To fully understand the upcoming `<O(n), O(1)>` solution, we need to to first get an intimate understanding of Cartesian Trees. They are largely responsible for the constant time lookup. In this section, we begin by discussing what cartesian trees are and how to efficiently construct them. We then implement a cartesian tree.

#### Cartesian Trees

A cartesian tree is a derivative data structure. It is derived from an underlying array. More formally, the cartesian tree `T` of an array `A` is a min binary heap of the elements of `A` organized such that an in order traversal of the tree yields the original array. How can we construct such a tree given some input array? The main observations that will guide our construction will be the requirement that an in-order traversal must yield the array elements in their positional order, and the requirement that the tree be a min heap. During an in-order traversal, the right child is retrieved after both the parent and the left child -- consequently, the right-most node will be the last node retrieved. We can thus build the tree [incrementally](https://www.notion.so/A-note-on-algorithmic-design-patterns-20e50d39c99945e3ad8dfb804177ab3f), adding each new element as the rightmost node of the tree.

More specifically,  we'll build the cartesian tree incrementally -- adding in elements in the order that they appear in the array. To add an element <!-- $\chi$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/EwDBYUim1S.svg">, we inspect the right spine of the tree starting with the right most node. We follow parent pointers until we find an element, <!-- $\psi$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/qPR3FOEHUn.svg">,  in the tree that is smaller than <!-- $\chi$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/EwDBYUim1S.svg">. We modify the tree, making <!-- $\chi$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/EwDBYUim1S.svg"> a right child of <!-- $\psi$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/qPR3FOEHUn.svg">. We also make the rest of the right subtree that is below <!-- $\chi$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/EwDBYUim1S.svg"> a left subtree of the new node. Traversing the right spine of tree from the right-most node can be done efficiently by keeping nodes on the right spine in a stack. That way, the rightmost node is always at the top of the stack. Below, we use this observation to implement a procedure for creating a cartesian tree from some array.

```rust
/// As always, we use the `wrapped index pattern`
///
/// An index into a collection of cartesian tree nodes
#[derive(Debug, Ord, PartialOrd, Eq, PartialEq, Clone)]
struct CartesianNodeIdx(usize);

#[derive(Debug)]
struct CartesianTreeNode<'a, T: Ord> {
    /// A reference to the array value that this node represents
    value: &'a T,

    /// The locations of the children and parent of this node.
    left_child_idx: Option<CartesianNodeIdx>,
    right_child_idx: Option<CartesianNodeIdx>,
}

impl<'a, T: Ord> std::ops::Index<CartesianNodeIdx> for Vec<CartesianTreeNode<'a, T>> {
    type Output = CartesianTreeNode<'a, T>;
    fn index(&self, index: CartesianNodeIdx) -> &Self::Output {
        &self[index.0]
    }
}

impl<'a, T: Ord> std::ops::IndexMut<CartesianNodeIdx> for Vec<CartesianTreeNode<'a, T>> {
    fn index_mut(&mut self, index: CartesianNodeIdx) -> &mut Self::Output {
        &mut self[index.0]
    }
}
/// A cartesian tree is a heap ordered binary tree
/// derived from some underlying array. An in-order
/// traversal of the tree yields the underlying tree.
#[derive(Debug)]
struct CartesianTree<'a, T: Ord> {
    nodes: Vec<CartesianTreeNode<'a, T>>,
    root_idx: Option<CartesianNodeIdx>,
    action_profile: Vec<CartesianTreeAction>,
}

/// When constructing a cartesian tree, we either
/// push a node to or pop a node from a stack.
/// We keep track of these actions because we can
/// use them to generate the cartesian tree number.
#[derive(Debug, Eq, PartialEq)]
enum CartesianTreeAction {
    Push,
    Pop,
}

impl<'a, T: Ord> From<&'a T> for CartesianTreeNode<'a, T> {
    fn from(value: &'a T) -> Self {
        CartesianTreeNode {
            value,
            left_child_idx: None,
            right_child_idx: None,
        }
    }
}

// To create the cartesian tree, we pop the stack until either
// it's empty or the element atop the stack has a smaller value
// than the element we are currently trying to add to the stack.
// Once we break out of the `pop` loop, we make the item we popped
// a left child of the new item we are adding. Additionally, we make
// this new item a right/left child of the item atop the stack
impl<'a, T: Ord> From<&'a [T]> for CartesianTree<'a, T> {
    fn from(underlying: &'a [T]) -> Self {
        let len = underlying.len();
        let mut nodes = Vec::with_capacity(len);
        let mut stack = Vec::<CartesianNodeIdx>::with_capacity(len);
        let mut action_profile = Vec::with_capacity(len * 2);
        for (idx, value) in underlying.iter().enumerate() {
            nodes.push(value.into());
            let node_idx = CartesianNodeIdx(idx);
            add_node_to_cartesian_tree(&mut nodes, &mut stack, &mut action_profile, node_idx);
        }
        let root_idx = stack.first().map(|min| min.clone());
        CartesianTree {
            nodes,
            root_idx,
            action_profile,
        }
    }
}

type Nodes<'a, T> = Vec<CartesianTreeNode<'a, T>>;
type Stack = Vec<CartesianNodeIdx>;
type Actions = Vec<CartesianTreeAction>;
/// Adds the node at the given idx into the tree by wiring up the
/// child and parent pointers. it is assumed that the
/// node has already been added to `nodes` the list of nodes.
/// This procedure returns an optional index value
/// that is populated if the root changed.
fn add_node_to_cartesian_tree<T: Ord>(
    nodes: &mut Nodes<T>,
    stack: &mut Stack,
    actions: &mut Actions,
    new_idx: CartesianNodeIdx,
) {
    let mut last_popped = None;
    loop {
        match stack.last() {
            None => break,
            Some(top_node_idx) => {
                // If the new node is greater than the value atop the stack,
                // we make the new node a right child of that value
                if nodes[top_node_idx.clone()].value < nodes[new_idx.clone()].value {
                    nodes[top_node_idx.clone()].right_child_idx = Some(new_idx.clone());
                    break;
                }
                last_popped = stack.pop();
                actions.push(CartesianTreeAction::Pop);
            }
        }
    }
    // We make the last item we popped a left child of the
    // new node
    if let Some(last_popped_idx) = last_popped {
        nodes[new_idx.clone()].left_child_idx = Some(last_popped_idx);
    }
    stack.push(new_idx);
    actions.push(CartesianTreeAction::Push);
}

impl<'a, T: Ord> CartesianTree<'a, T> {
    fn in_order_traversal(&self) -> Vec<&T> {
        let mut res = Vec::with_capacity(self.nodes.len());
        self.traversal_helper(&self.root_idx, &mut res);
        res
    }

    fn traversal_helper(&self, cur_idx: &Option<CartesianNodeIdx>, res: &mut Vec<&'a T>) {
        let nodes = &self.nodes;
        match cur_idx {
            None => {}
            Some(cur_sub_root) => {
                self.traversal_helper(&nodes[cur_sub_root.clone()].left_child_idx, res);
                res.push(&nodes[cur_sub_root.clone()].value);
                self.traversal_helper(&nodes[cur_sub_root.clone()].right_child_idx, res);
            }
        }
    }
}
```

You can play around with the code for constructing a cartesian tree in [the rust playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=c51356cba92f48f0434c64abd21d7162).

---

Why are cartesian trees important, and how are they related to the `RMQ` problem? First, notice that once we have a cartesian tree for an array, we can answer any `RMQ` on that array. In particular, <!-- $RMQ_A(i, j) = LCA_T(A[i], A[j])$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/s3WLVY6DAX.svg">. That is we can answer `RMQ` by doing lowest common ancestor searches in the cartesian tree. Although this idea is intrinsically interesting, we do not explore it further. Feel free to check out [this note for further details](http://courses.csail.mit.edu/6.851/fall17/scribe/lec15.pdf).

To fully appreciate the importance of cartesian trees and their relation to the data structure design problem at hand, we have to explore when and how two arrays have isomorphic trees. This will lead us to a way of figuring out when two blocks can share the same pre-processed index -- a thing that will lead us to an `RMQ` data structure with constant query time.

#### Cartesian Tree Isomorphisms

When do two cartesian trees for two different arrays, <!-- $B_1$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/pCi7uET35K.svg">, <!-- $B_2$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/1b0Fxslia6.svg"> have the same shape? How can we tell this efficiently? Put simply, if two blocks have the same shape, then their minimal values of any range in both blocks occur at the same index. This means that, the sequence of `Push` and `Pop` operations when constructing the cartesian trees for the two blocks are exactly the same. Therefore, to know if two blocks are isomorphic, we could simply compare their action profiles. Note, however, when we are only interested in whether two blocks have isomorphic trees, we don't even need to construct the tree. We also do not need to allocate space for the action profile vector. The idea is to create a a bitstring from the sequence of `Push` and `Pop` operations. The number formed by this bitstring is called the cartesian tree number. Therefore, with this scheme, two blocks have isomorphic trees if they have the same cartesian tree number. Below, we show how to calculate such a number from the action profile. 

```rust
impl<'a, T: Ord> CartesianTree<'a, T> {
    /// Calculates the cartesian tree number of this tree
    /// using the sequence of `push` and `pop` operations
    /// stored in the `action_profile`. Note that calculating this
    /// value only makes sense when the underlying array is small.
    /// More specifically, this procedure assumes that the underlying
    /// array has at most 32 items. This makes sense in our context
    /// since we're mostly interested in the cartesian tree numbers
    /// of RMQ blocks
    fn cartesian_tree_number(&self) -> u64 {
        let mut number = 0;
        let mut offset = 0;
        for action in &self.action_profile {
            if action == &CartesianTreeAction::Push {
                number |= 1 << offset;
            }
            offset += 1;
        }
        number
    }
}
```
A nice consequence of the preceding discussion is that we can say something about the number of possible cartesian trees for an array of a given length `b`. Note that, the maximum length of the action profile is `2b`. Therefore, the largest cartesian tree number that can be produced is <!-- $2^{2b} = 4^b$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/xIDYHrCQR2.svg">. This number will come in handy when we analyze the runtime of the `<O(n), O(1)>` solution.

#### The Fischer-Heun RMQ Structure
How does all this talk of cartesian trees and cartesian numbers translate into an `<O(n), O(1)>` range min query solution? Let's discuss that next.

As with the other methods, we begin by dividing the underlying array into blocks of `b` items. For each of our `ceil(n/b)` blocks, we find the location of the smallest element by scanning. We then aggregate the min locations for all blocks in what we call the macro array. To answer an `rmq` query, we simply return the smallest value from three smaller `rmqs`: (a) The block of the starting index, (b) The block of the ending index, and (c) The macro array. So far, this is just a recap from the previous discussion.

We are yet to answer two key questions though: (a) What should our block size `b` be? and (b) What methods (`SolverKinds`) should we use to answer queries on the macro and micro arrays? The judiciously picking the block size and cleverly applying our solvers, we shall arrive at an `<O(n), O(1)>` method.

We shall use the `SparseTableSolver` to solve queries on the macro array. For each block, we shall use the `DenseTableSolver`. However, if two blocks have the same cartesian tree number, we shall have them share solvers. We can do this because, as mentioned earlier, two blocks have the same cartesian numbers iff they have isomorphic cartesian trees which further implies that min values for both blocks occur at the exact same indexes. Since we use precomputed lookup tables, the query time is obviously `O(1)`. How about the preprocessing time?

The preprocessing time is an amalgamation of three terms:
  1. The time used to divide the input into blocks and create the blocks. This, as we saw earlier, is `O(n)`
  2. The time used to create the solver for the macro array. Since we are using the `SparseTableSolver` we know that this will be `O(n/b lg n)`. Note that in past sections, we picked `b = lg n`. We have yet to choose the value of `b`.
  3. And finally, the time used to create solvers for each block. Since we are using the `DenseTableSolver`, we know that this time will be quadratic. That is, the time to create a solver for a single block will be `b^2`. Since we are sharing solvers between blocks, we need to multiply the time we spend on each block by the number of distinct blocks of size `b`. As we saw earlier, this value is <!-- $2^{2b} = 4^b$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/xIDYHrCQR2.svg">. 

Therefore, our expression for the preprocessing time is <!-- $\mathcal{O}(n + \lceil\frac{n}{b}\rceil + b^24^b)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/MiWMTJnLcq.svg">. Our final task is thus to pick a value of `b` that will make this expression evaluate to `O(n)`. Long story short, we pick <!-- $b = 0.5 \lg_4 n = O.25 \lg_2 n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/u3vbuw4bn2.svg">.

To summarize, by choosing a block size of `0.25 lg n` and using shared dense table solvers to solve the micro level `rmqs` (where we use cartesian tree numbers to figure out when to share solvers), and a sparse table solver for the macro array, we're able to solve the `RMQ` problem in `<O(n), O(1)>`. However, the `O(n)` preprocessing time does hide a large constant factor. Think of the trouble we went through to compute the to calculate cartesian trees.

Thus, our final data structure has the following features:
|Block Size|Macro Array Method|Micro Array Method|Runtime|
|----------|------------------|------------------|-------|
|`0.25 lg n`| Sparse Table| Sparse Table with Cartesian Tree based caching|`<O(n), O(1)>`|

As discussed earlier, although this method has impressive  asymptotic numbers, it is often outperformed in practice by the hybrid with logarithmic query time. Furthermore, this method is a lot more complex. That is another reason, from an engineering standpoint, to prefer the `<O(n), O(lg)>` -- much less code, and just as fast.

We leave the implementation of this scheme as an exercise. Using the abstractions from above, the implementation should be a simple extension. We simply have to keep track of a mapping from blocks to cartesian tree numbers and modify `BlockLevelSolvers` to map from cartesian tree numbers instead of blocks.

## References

1. [CS 166 Lecture 1](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/00/Small00.pdf)
2. [CS 166 Lecture 2](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/01/Small01.pdf)
3.  [6.851](http://courses.csail.mit.edu/6.851/fall17/lectures/L15.pdf)

```latex
@article{jlikhuva2021rmq,
  title   = "Rusty Solutions to the Range-Min Query Problem.",
  author  = "Okonda, Joseph",
  journal = "https://github.com/jlikhuva/blog",
  year    = "2021",
  url     = "https://github.com/jlikhuva/blog/blob/main/posts/rmq.md"
}
```
