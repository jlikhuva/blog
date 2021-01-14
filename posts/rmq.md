#### Introduction

#### A Naïve Solution
The most straightforward way to solve this problem is to create a lookup table with all the RMQ answers precomputed. This will allow us to answer any RMQ in constant time by doing a table lookup. How can we build such a table? The first thing to notice is that this is a discrete optimization problem - we are interested in the minimal (aka the optimal) value in a given range. A quick reference to [common algorithmic patterns](https://www.notion.so/A-note-on-algorithmic-design-patterns-20e50d39c99945e3ad8dfb804177ab3f) should tell us that we may be able to use dynamic programming to solve the problem. All we need to do is come up with an update rule. In particular, suppose our array is <!-- $A$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/fT4Y6ki4ih.svg">,  if we know the smallest value in some range <!-- $(i, j)$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=(i%2C%20j)"> to be <!-- $\alpha$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/pU0BwDfjZb.svg">, we can easily figure out the answer on a larger range <!-- $(i, j+1)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/BR28J51dHP.svg"> by comparing <img style="transform: translateY(0.1em); background: white;" src="../svg/pU0BwDfjZb.svg"> with <!-- $A[i + 1]$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/DVfAuKZf2r.svg">. That is:
<!-- $$
RMQ_A(i, j) = \begin{cases}
      A[j], & \text{if}\ i = j \\
      \min\left(A[j], RMQ_A(i, j - 1)\right), & \text{otherwise}
\end{cases}
$$ --> 

<div align="center"><img style="background: white;" src="../svg/vZtLroTQei.svg"></div>

We can do this for all possible values of `i` and `j` to fill up our lookup table. This takes quadratic time. Thus, with this approach, we cam solve the RMQ problem in <!-- $\left<\Theta(n^2), \Theta(1)\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/8UBwqFAqMg.svg">.

The code below implements this approach. The only modification we make is that instead or calculating the actual minimal value, we calculate the index of the smallest value. That is `argmin` instead of `min`

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
fn compute_rmq_all_ranges<'a, T: Hash + Eq + Ord>(array: &'a [T]) -> LookupTable<'a, T> {
    let len = array.len();
    let mut lookup_table = HashMap::with_capacity((len * len)/2);
    for start in 0..len {
        for end in start..len {
            if start == end {
                lookup_table.insert((start, end, array).into(), start);
            } else {
                let prev_range = (start, end-1, array).into();
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
Can we do better than <!-- $\left<\Theta(n^2), \Theta(1)\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/8UBwqFAqMg.svg">? The query time is the best we can ever hope for. However, we can reduce the processing time. Let's see how we can do that in the next section.

#### Binary Representation & Sparse Tables
Any positive integer can be factored into a sum of powers of two. This binary factorization is the basis of binary representation. For instance, the decimal number 19 can be represented as <!-- $19 = 16 + 2 + 1 = 2^4 + 0*2^3 + 0*2^2 + 2^1 + 2^0 = (10011)_2$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/5zxNilT8JM.svg">. Given some range `[i, j]` we know that its length, `(j - i) + 1` is positive. We can therefore factor it using binary factorization to get shorter ranges. For instance, if our range is `(0, 18)` we see that it has a length of `19` which, as we saw above, can be be factored into `(0, 15) + (16, 17) + (18, 18)`. 

**Preprocessing**

How can we use these observations to construct a solution to our problem? To see how, let us first consider  how we would construct a sparse table. To create the sparse table, we simply precompute the RMQ answers to all ranges whose length is a power of 2. For an array of length `n`, there are `O(lg n)` such ranges. To cover all possible ranges, we have to perform this precomputation for all `n` possible starting positions. Therefore, the time needed to create the sparse table is <!-- $\mathcal{O}(n \log n)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/CEiOpX0Hzv.svg">.We implement this scheme below.
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
/// A sparse table is simply a collection of lookup tables for ranges whose
/// length is a power of two. We precompute such ranges for all possible starting
/// positions
type SparseTable<'a, T> = HashMap<SparseTableIdx, LookupTable<'a, T>>;

/// For each index `i`, compute RMQ answers for ranges starting at `i` of
/// size `1, 2, 4, 8, 16, …, 2^k` as long as the resultant ending index
/// fits in the underlying array in the array.
/// For each array index, we compute lg n ranges. Therefore,
/// the total cost of the procedure is O(n lg n)
fn compute_rmq_sparse_table<'a, T: Hash + Eq + Ord>(array: &'a [T]) -> SparseTable<'a, T> {
    let len = array.len();
    let mut sparse_table = HashMap::new();
    for start_idx in 0..len {
        let mut power = 0;
        let mut end_idx = start_idx + (1 << power) - 1;
        while end_idx < len {
            let cur_range_idx: SparseTableIdx = (start_idx, 1 << power).into();
            let cur_range_lookup_table = compute_rmq_all_ranges(&array[start_idx..=end_idx]);
            sparse_table.insert(cur_range_idx, cur_range_lookup_table);
            power += 1;
            end_idx = start_idx + (1 << power) - 1;
        }
    }
    sparse_table
}
```
**Querying the Sparse Table**

Now that we have our sparse table, how can we query from it given an arbitrary range `R = [i, j]`? From our initial discussion of binary factorization, you can imagine computing all subranges of `R` whose length is a power of 2 and then taking the min over these values. For an arbitrary length `n`, there are <!-- $O(\lg n)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/wfptrCDR1m.svg"> such subranges. Thus, this scheme would give us a <!-- $\left<\Theta(n\lg n), \Theta(\lg n)\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/Z802R6ewbT.svg"> solution to the `RMQ`problem. 

Computing all subranges, however, is overkill. All we need are two sub-ranges that fully cover the underlying segment. How do we find the two covering segments? First, observe that if the length of the range is an exact power of two, then we do not need to do any further computation since we already precomputed answers for all such ranges. If its not, we start by finding the largest subrange that is an exact power of two. Specifically, we find the value `k` such that <!-- $2^k \leq (j - i) + 1$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/SyXeSDNkuL.svg">. Note that this value `k` is the index of the most significant bit of the range's length. The first range is thus <!-- $[i, i + 2^k - 1]$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/RsDV0G7oU5.svg">. After finding the largest subrange, the remaining portion's length certainly be not be a power of two. To proceed, we use a neat trick: we construct a range whose length is the smallest power of two larger than the remaining portion length. To prevent this subrange from overflowing the underlying range, we shift it over to the left, overlapping the first subrange, until it is full contained in the original range. The second range is thus <!-- $[j - 2^k + 1, j]$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/2eOUcbNTa4.svg">

To recapitulate, we query from the sparse table by finding the `argmin` of two overlapping ranges whose answers have already been computed. Figuring out which ranges to use involves finding the `MSB(n)` where `n` is the length of the range in the query. How so we calculate `MSB(n)`? To compute `MSB(n)` in constant time, we can use a lookup table. Later on, when discussing specialized integer containers, we'll implement a complex but straightforward method for finding `k` in constant time. For now, a lookup table suffices. Thus, with this scheme, we have a <!-- $\left<\Theta(n\lg n), \Theta(1)\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/WpCYNtMQVL.svg"> solution to the `RMQ` problem. Below, we implement a procedure to compute the lookup table.
```rust
/// Here's the scheme we shall use to implement the lookup table:
///     - First, we shall assume that the values we get are 8 bytes (64bits) wide
///     - We shall precompute all MSB(n) values for all n <= 2^16. This will use
///       65536 bytes which is approximately 66Kb.
///     - To find the MSB of any value, we combine answers from the 4 16 bit
///       portions using logical shifts and masks
pub struct MSBLookupTable([u8; 1 << 16]);

impl MSBLookupTable {
    /// Build the lookup table. To fill up the table, we simply subdivide
    /// it into segements whose sizes are powers of two. The MSB for a segment
    /// are the same for instance:
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
    pub fn msb_idx(&self, n: usize) -> u8 {
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
Once again, the query time is the best possible. However, even though the pre-processing time reduced from quadtratic to `O(n lg n)`, we can still do better. In particular, we can shave off a log factor and arrive at a linear time pre-processing algorithm. To figure out how to do that, we shall take a detour to discuss the method of four russians.

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
    let mut blocks = Vec::with_capacity(5);
    for start_idx in (0..array.len()).step_by(5) {
        let end_idx = (start_idx + 5) - 1;
        let median_idx = get_median_idx_by_sorting(&array[start_idx..=end_idx]);
        blocks.push((start_idx, end_idx, array, median_idx).into())
    }
    blocks
}

/// This solves the problem for a single block. This changes from problem to problem.
/// For instance, when we solving RMQ, this will be get_min_idx_by_scanning. You can
/// definitely think of a way of creating an abstraction that can solve any
/// method of four russians problem by having the client provide a funtion for
/// block partitioning and solving the block level problem. 
fn get_kth_by_sorting<T: Ord>(block: &[T], k: usize) -> usize {
    let sorted: Vec<_> = block
        .iter()
        .enumerate()
        .sorted_by_key(|x| x.1)
        .map(|x| x.0)
        .collect();
    sorted[k]
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
/// approximate median that guranteeds us a roughly even split
fn get_approx_median_idx<'a, T>(macro_array: &[MedianBlock<'a, T>]) -> usize {
    todo!()
}

/// Computes the index of the k-th smallest element in the `array`. This is sometimes
/// referred to as the k-th order statistic. This procedure computes this value in
/// O(n). Note that this is a more general method for finding the median i.e the
/// (n/2)-th order statistic
fn kth_order_statistic<'a, T: Ord>(array: &'a mut [T], k: usize) -> &T {
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
The median of medians procedure has a few key structures:
* The input is divided into blocks or equal size. This is called block partitioning and each block is called the micro array.
* The original problem (median in this case) is solved for each block using a naive method that works well for small input sizes. With this scheme, we are able to solve the problem for each block in constant time and for all blocks in linear time.
* The solutions to all blocks are aggregated into a single array. We call this the macro array. The macro array, just like the micro arrays, are smaller instances of the original problem.
* By combining, in some bespoke fashion, the macro and micro array solutions, we are able to solve the original problem with a log factor shaved off. In `MoM` we went from `Quickselect's` `O(n lg n)` to `O(n)`.  
  
The structures above are the major motifs in the method of four russians. How can we use this method to reduce the pre-processing time of our RMQ algorithm? We discuss that next.

**Two-Level Structures**

To apply the method of four russians to the RMQ problem, we begin by dividing the input array into blocks of length <!-- $b$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/fCJxfN2Ysc.svg">. If the length of the array is <!-- $n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/k2Z6CavdDQ.svg">, this results in <!-- $\mathcal{O}(\frac{n}{b})$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/gwfsqBuBks.svg"> blocks. For each of these blocks, we find the index of the smallest value bu doing a simple scan. This takes <!-- $\mathcal{O}(b)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/Dqg8jnCq0o.svg"> in each block and <!-- $\mathcal{O}(\frac{n}{b}) * \mathcal{O}(b) = \mathcal{O}(n)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/oI85zP1xyG.svg"> for all the blocks. We aggregate these min values in a new macro array. Given a qeury range `[i, j]` how can we use the blocks and the macro array to satisfy the query? Also, what value of `b` should we use? To query, we start by figuring out which block the ends of the query fall into. We do that by dividing each end with the block size, i.e `start_block = i/b, end_block = j/b`. We then scan the items in `start_block` that appear after `i` and the items in `end_block` that appear before `j` and take the minimal value over them. Let's call this value, the smallest value at the ends of the range, <!-- $\lambda$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/w3irZZ0Jg1.svg"> Then we scan the macro array to find the minimal value among all blocks between `start_block` and `end_block`. let's call this value <!-- $\alpha$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/6bXEiZZAEX.svg">. The answer to our query is the `min` (or `argmin`) between these two values: <!-- $RMQ_A(i, j) = \min(\lambda, \alpha)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/KHSI90DMS3.svg"> How long does this take? Well, finding the `min` in the end blocks take `O(b)` and scanning the intermediate blocks takes `O(n/b)`. This gives us `O(b + n/b)`. Therefpre, to properly characterize the runtime, we need to find the value of `b` that minimized the expression `b + n/b`. We do so below
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

So, we set `b` to the square root of `n`. This gives us a query time of <!-- $\mathcal{O}(\sqrt{n})$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/qSsMnsGYC0.svg"> and an overall time of <!-- $\left<O(n), O(n^{0.5})\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/cgFTCzGhAr.svg">. We implement this scheme below.
```rust
/// The abstraction for a single block. This is exactly like
/// we had for median_of_medians
pub struct RMQBlock<'a, T> {
    /// The starting index. This is 0-indexed and should be
    /// less than or equal to the end_idx
    start_idx: usize,

    /// The ending index. This should be strictly less than the
    /// length of the underlying array
    end_idx: usize,

    /// The index of the smallest value in the given range. To move from this
    /// index to an idx in the underlying, we simply calculate
    /// `start_idx + min_idx`
    min_idx: usize,

    /// The minimal value of this block
    min: &'a T,
}

/// We are essentially constructing a new data structure.
/// This abstracts the logic for constructing
/// the two level structure and answering range queries
pub struct RMQBlockDecomposition<'a, T> {
    /// An aggregation of the minimal values in each of our
    /// (n/b) blocks.
    macro_array: Vec<RMQBlock<'a, T>>,

    /// The size of each block
    block_size: usize,

    /// The static array onto which this data structure is layered
    underlying: &'a [T]
}

impl <'a, T> RMQBlockDecomposition<'a, T> {}

```
So, Block decomposition allowed us to have linear pre-processing time. However, in the process, we lost our constant query time? Can we do better than <!-- $\sqrt{n}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/k4Mu0nuOVP.svg"> while still maintaining a linear pre-processing time? Yes. We can use a mix of block decomposition and sparse tables to achieve this. Let's see how. 

#### Hybrid Strucures
When discussing block decomposition, after decomposing the input into micro arrays, we went ahead and solved the original problem on each block, treating each as a reduced instance of the original. We also did the same for the macro array. In the preceding section, we solved the problem by doing a linear scan. We can, however, use methods from previos sections -- sparse and dense lookup tables to solve the problem on the micro and macro arrays. When we do that, we end up with hybrid solutions that have faster query times. In this section, we shall explore a few hybrid structures and characterize their runtimes. 

To create a hybrid structure, we need to decide which method we want to use to solve the problem on the macro array and on each micro array. By mixing and matching methods, we get different hybrids with different runtimes as shown in the table below.
|Block Size|Macro Array Method|Micro Array Method|Runtime|
|----------|------------------|------------------|-------|
|<!-- $\lg n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/VfomraIoyi.svg">   | Sparse Table | Linear Scan |<!-- $\left<O(n), O(\lg n\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/iVybTYrhUA.svg">|
|<!-- $\lg n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/VfomraIoyi.svg">    | Sparse Table | Sparse Table |<!-- $\left<O(n \lg \lg n), O(1\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/0m8X2tOgH1.svg">|
|<!-- $\lg n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/VfomraIoyi.svg">    | The first hybrid in this table| Sparse Table|<!-- $\left<O(n), O(\lg \lg n\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/cUT7DO2po5.svg">|

Below, we implement the first hybrid method
```rust
/// WIP
```
By this point we have a nifty and quite efficient algorithm for the offline range min query problem. However, the title of the note did promise an `<O(n), O(1)>` solution. We discuss that in the next section with the caveat that the added constant factors that give us assymptotic constant query time may slow down the algorithm in practice. As noted [here](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/01/Small01.pdf), the preceding `<O(n), O(lg)>` hybrid solution outperforms the `<O(n), O(1)>` solution in practice.

#### Cartesian Trees & The LCA-RMQ Equivalence
To fully understand the upcoming `<O(n), O(1)>` solution, we need to to first get an intimate understanding of Cartesian Trees. They are largely responsible for the constant time lookup. In this section, we begin by discussing what cartesian trees are and how to efficently construct them. We then implement a cartesian tree.

#### The Fischer-Heun RMQ Structure