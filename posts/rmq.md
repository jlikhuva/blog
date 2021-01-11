##### Introduction

##### A naive solution
The most straightforward way to solve this problem is to create a lookup table with all the RMQ answers precomputed. This will allow us to answer any RMQ in constant time by doing a table lookup. How can we build such a table? The first thing to notice is that this is a discrete optimization problem - we are interested in the minimal (aka the optimal) value in a given range. A quick reference to [common algorithmic patterns](https://www.notion.so/A-note-on-algorithmic-design-patterns-20e50d39c99945e3ad8dfb804177ab3f) should tell us that we may be able to use dynamic programming to solve the problem. All we need to do is come up with an update rule. In particular, suppose our array is $A$,  if we know the smallest value in some range <!-- $(i, j)$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=(i%2C%20j)"> to be <!-- $\alpha$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/pU0BwDfjZb.svg">, we can easily figure out the answer on a larger range <!-- $(i, j+1)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/BR28J51dHP.svg"> by comparing <img style="transform: translateY(0.1em); background: white;" src="../svg/pU0BwDfjZb.svg"> with <!-- $A[i + 1]$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/DVfAuKZf2r.svg">. That is:
<!-- $$
RMQ_A(i, j) = \begin{cases}
      A[j], & \text{if}\ i = j \\
      \min\left(A[j], RMQ_A(i, j - 1)\right), & \text{otherwise}
\end{cases}
$$ --> 

<div align="center"><img style="background: white;" src="../svg/vZtLroTQei.svg"></div>
We can do this for all possible values of `i` and `j` to fill up our lookup table. This takes quadratic time. Thus, with this approach, we cam solve the RMQ problem in <!-- $\left<\Theta(n^2), \Theta(1)\right>$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/8UBwqFAqMg.svg">

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

##### Binary Representation & Sparse Tables

##### Hybrid Structures & The Method of four Russians

##### Cartesian Trees & The LCA-RMQ Equivalence

##### The Fischer-Heun RMQ Structure