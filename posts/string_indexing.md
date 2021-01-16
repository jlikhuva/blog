# String Indexing
#### Introduction
I lied. We will not talk about Tries. We will, however, discuss two structures that are foundational to tasks that need to do substring matching: The Suffix Array and The Longest Common Prefix Array. We'll explore two linear time procedures (SA-IS & Kasai's Algorithm) for constructing these data structures given some underlying, fairly static string from some alphabet <!-- $\Sigma$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/pqSc9UzkQu.svg">.

#### Substring Search
A string <!-- $\alpha$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/2jiDZwz5ad.svg"> with length <!-- $n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/Kk7kPk2fRQ.svg"> is said to be a substring of another string <!-- $\Omega$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/xaqTdKWTXT.svg"> with length <!-- $m \text { and } n \leq m$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/dUhlY1dPvx.svg"> if, an only if <!-- $\alpha$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/2jiDZwz5ad.svg"> is a prefix of some suffix of <!-- $\Omega$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/xaqTdKWTXT.svg"> . This is the basic insight that explains why we weould want to construct a suffix array. Before we proceed any further, we need to explain what a suffix array is.

A suffix array for some string <!-- $\Omega$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/xaqTdKWTXT.svg">  is an array of all the suffixes of that string. Note that, since a suffix is fully defined by its starting index in the string, there are <!-- $m + 1$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/0Yio9PnxIZ.svg"> possible suffixes (where the extra 1 comes from counting the empty suffix). This array is sorted lexicographically. To save space, suffix arrays do not store the full suffixes, instead, they only store the starting index of each suffix. To sum up, a suffix array for some string is a lexicographically sorted array of all the indexes of all suffixes in the underlying string.

Finally, most algorithms that operate on strings assume that each string has a sentinel character appended at the end. This sentinel character should not appear anywhere else in the string and should be lexicographically smaller than all characters that could appear in the string. For the ASCII alphabet, we often use the character `$` as the sentinel.

#### The Suffix Array: A Naïve Solution via Sorting
How can we create the suffix array? The most straightforward way would be to first generate all possible suffixes, then sort them lexicographically. This runtime of this approach is at least `m lg m` as that is the comparison-based sorting lower bound. 

For instance, suppose out string is, <!-- $\Omega = \text{banana}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/IgqCBZV4Hg.svg">. We would start by appending the sentinel at the end to get <!-- $\Omega' = \text{banana\$}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/r2eMXBEDPR.svg">. Then we would generate all the 7 different suffixes of the string in linear time by doing a single scan over the string. This would give us the table on the left below:
<!-- $$
\begin{array}{lr}
    \hline \text{start index}&{\text{suffix}}\\ \hline
    \text{0}&{\text{banana\$}}\\
    \text{1}&{\text{anana\$}}\\
    \text{2}&{\text{nana\$}}\\
    \text{3}&{\text{ana\$}}\\
    \text{4}&{\text{na\$}}\\
    \text{5}&{\text{a\$}}\\
    \text{6}&{\text{\$}}\\ 
\end{array}
\quad
\begin{array}{lr}
    \hline \text{start index}&{\text{suffix}}\\ \hline
    \text{6}&{\text{\$}}\\
    \text{5}&{\text{a\$}}\\
    \text{3}&{\text{ana\$}}\\
    \text{1}&{\text{anana\$}}\\
    \text{0}&{\text{banana\$}}\\
    \text{4}&{\text{na\$}}\\
    \text{2}&{\text{nana\$}}\\ 
\end{array}
$$ --> 

<div align="center"><img style="background: white;" src="../svg/JPUtXuRKeP.svg"></div>

We would then sort these suffixes to get the table on the right. The start index column of that table is the suffix array. Right off the bat, we can notice a few salient features aboy the suffix array. these features are the source of its utility. First, notice that all suffixes that start with the same character occupy a contiguous slice in the array. In fact, all suffixes that begin with the same prefix are all next to one another.

We implement this scheme below. 
```rust
//! We will be working with indexes to different arrays a lot. Having many
//! raw indexes flying around increases the cognitive load as it requires
//! one to be super vigilant not to use an index of one array in another different array.
//! To solve this problem, we use the `NewType Index Pattern`. By giving each index
//! a concrete type, we offload the cognitive load to the compiler.

/// This is an index into the underlying string
#[derive(Debug, PartialEq, Eq, Hash, Clone)]
pub struct SuffixIndex(usize);

/// To make things even more ergonomic, we implement the `Index` trait to allow
/// us to use our new type without retrieving the wrapped index. For now,
/// We assume that our string will be a collection of bytes. That is of course
/// the case for the ascii alphabet
impl std::ops::Index<SuffixIndex> for [u8] {
    type Output = u8;
    fn index(&self, index: SuffixIndex) -> &Self::Output {
        &self[index.0]
    }
}

pub struct SuffixArray<'a> {
    /// The string over which we are building this suffix array
    underlying: &'a str,

    /// The suffix array is simply all the suffixes of the
    /// underlying string in sorted order
    suffix_array: Vec<SuffixIndex>,
}

/// This is an index into the suffix array
#[derive(Debug, PartialEq, Eq, Hash, Clone, Ord, PartialOrd)]
pub struct SuffixArrayIndex(usize);

/// This allows us to easily retrieve the suffix strings by using bracket
/// notation.
impl <'a> std::ops::Index<SuffixArrayIndex> for SuffixArray<'a>  {
    type Output = str;
    fn index(&self, index: SuffixArrayIndex) -> &Self::Output {
        let suffix_idx = &self.suffix_array[index.0];
        &self.underlying[suffix_idx.0..]
    }
}
```
With these abstractions in place, we can go ahead and implement our naive SACA.
```rust
impl<'a> SuffixArray<'a> {
    /// Construct the suffix array by sorting. This has worst case performance
    /// of O(n log n)
    pub fn make_sa_by_sorting(s: &'a str) -> Self {
        let mut suffixes = vec![];
        for i in 0..s.len() {
            suffixes.push(&s[i..]);
        }
        suffixes.sort();
        let mut suffix_array = vec![];
        for suffix in suffixes {
            let cur = SuffixIndex(s.len() - suffix.len());
            suffix_array.push(cur);
        }
        Self {
            underlying: s,
            suffix_array,
        }
    }
}
```
Now that we our suffix array, how can we use it to do substring search? In particular, suppose that we build a suffix array for the string <!-- $\Omega, |\Omega| = m$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/56ySZzOixg.svg">  and we want to finds all places that another query string <!-- $\alpha, |\alpha| = n$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/BZ9vf2pDJC.svg"> occurs in the first string. How fast can we do this? one approach would be to use two binary searches to find the left and right region in the suffix array where our query string occurs. We'll have to make at least <!-- $\lg m$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/0jveyyhdre.svg"> comparisons. To fully characterize the runtime, we need to figure out how long each comparison takes. If each lexicographic comparison is implemented by linear scanning, then thr runtime to find the boundaries will be <!-- $O(n \lg m)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/dGzFvWPAC8.svg">. Do note that it is possible to implement string comparisons in near constant time using word-level parallelism tachniques that are at the heart of [specialized containers for integers](https://github.com/jlikhuva/blog/blob/main/posts/integer.md). Once we know the boundaries of our region of interest, we can simply retrieve the mathching locations by doing a linear scan. If we have <!-- $z$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/HGgKz50xnG.svg"> matches, the total runtime for this scheme becomes <!-- $O(n \lg m + z)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/m66NpQVRE6.svg">. We leave the implementation details as an exercise to the reader.

#### The LCP Array: Kasai's Procedure
Suffix arrays are fairly useful on their own. However, they become even more useful when coupled with information about the lengths of common prefixes. In particular, the time to do substring matching is greatly reduced if we know the length of the longest common prefixes between all pairs of adjacent suffixes in the suffix array. How can we get that information? We explore that here.

**A Naive Solution:** Given a suffix array, the most straightforward way to construct an LCP array is to scan over the elements of the array, calculating the lengths of common prefixes for all adjacent suffixes. We implement that below.
```rust
/// The length of the longest common prefix between the
/// suffixes that start at `left` and `right`. These
/// suffixes are adjacent to each other in the suffix array
#[derive(Debug)]
pub struct LCPHeight {
    left: SuffixIndex,
    right: SuffixIndex,
    height: usize,
}

impl<'a> SuffixArray<'a> {
    /// Retrieve the index of the suffix stored at this location
    /// in the suffix array. Put another way, we retrieve the id
    /// of the (idx + 1) smallest suffix in the string
    pub fn get_suffix_idx_at(&self, idx: usize) -> SuffixIndex {
        // These clones are quite cheap
        self.suffix_array[idx].clone()
    }
}

/// Calculate the length of the longest common prefix
/// between the two string slices in linear time
fn calculate_lcp_len(left: &str, right: &str) -> usize {
    let mut len = 0;
    for (l, r) in left.as_bytes().iter().zip(right.as_bytes()) {
        if l != r {
            break;
        }
        len += 1;
    }
    len
}

/// The naive procedure described in the preceding section
fn make_lcp_by_scanning(sa: &SuffixArray) -> Vec<LCPHeight> {
    let mut lcp_len_array = Vec::with_capacity(sa.len());
    for i in 1..sa.len() {
        let prev_sa_idx = SuffixArrayIndex(i - 1);
        let cur_sa_idx = SuffixArrayIndex(i);
        let lcp_len = calculate_lcp_len(&sa[prev_sa_idx], &sa[cur_sa_idx]);
        lcp_len_array.push(LCPHeight {
            left: sa.get_suffix_idx_at(i - 1),
            right: sa.get_suffix_idx_at(i),
            height: lcp_len,
        });
    }
    lcp_len_array
}
```
How fast is this procedure? Well, clearly it takes at least `O(n)`. To get a tighter bound, we need to investigate the worst case behavior of the inner loop that calculates the `LCP` between two suffixes. Suppose that the two strings are identical except that one is one character shorter than the other. In that case, the inner loop will iterate `n-1` times. This means that the runtime of this procedure is <!-- $O(n^2)$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/I7UitjU2R1.svg">. This is bad. Do note that the example used is not a degenerate case, it is quite likely to occur when dealing with really long strings (for example 3 billion characters) from a really small alphabet (for instance <!-- $\Sigma = 4$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/rRwYDAdqVg.svg">). We need a faster method. 

**[Kasai's Procedure:](http://web.cs.iastate.edu/~cs548/references/linear_lcp.pdf)** The main reason why the naive solution is sub-optimal is the inner loop. If we could somehow reduce the time needed to compute `LCP` values, we could markedly improve the overall runtime. The first thing to observe is that when implementing the naïve procedure, we iterated over the suffix array -- not the underlying string. Because of this,  we are unable to exploit the fact that the only difference between two suffixes $S_i, \text { and } S_{i + 1}$ that are adjacent to each other _in the string_ is that one has one more character at the start. Suppose we have already found the `lcp` length between `S_i` and the `S_k`, a suffix adjacent to it _in the suffix array_, to be `h > 1`. How can we use this information to calculate the lcp length between $S_{i +1}$ and `S_j`, the suffix adjacent to it in the suffix array? The key insight stems from observing that if we delete the first character from $S_i$ we get $S_{i + 1}$, and since `h > 1` deleting that character from `S_j` yields another suffix that is adjacent to $S_{i + 1}$ with overlap in at least `h-1` location. Therefore, to calculate the `lcp` between the shorter suffixes, we do not need to compare the first `h-1` suffixes for we already know that they are the same. This effectively reduced the number of times the inner loop iterates and results in a linear time solution. We implement this scheme below.
```rust
impl From<(SuffixIndex, SuffixIndex, usize)> for LCPHeight {
    fn from((l, r, h): (SuffixIndex, SuffixIndex, usize)) -> Self {
        LCPHeight {
            left: l,
            right: r,
            height: h,
        }
    }
}
/// Computes the lcp array in O(n) using Kasai's algorithm. This procedure assumes that
/// the sentinel character has been appended onto `s`.
fn make_lcp_by_kasai(s: &str, sa: &SuffixArray) -> Vec<LCPHeight> {
    let s_ascii = s.as_bytes();
    let mut lcp_array = Vec::with_capacity(s.len());
    // We need a quick way to move from the index of the suffix in the
    // string (the `SuffixIndex`) to the index of that same string in the
    // suffix array (the `SuffixArrayIndex` aka the rank). This map
    // will allow us to do that once populated.
    let mut suffix_index_to_rank = HashMap::with_capacity(s.len());
    for i in 1..s.len() {
        suffix_index_to_rank.insert(sa.get_suffix_idx_at(i), SuffixArrayIndex(i));
        lcp_array.push((sa.get_suffix_idx_at(i - 1), sa.get_suffix_idx_at(i), 0).into())
    }
    let mut h = 0;
    // We then loop over all the suffixes one by one. One thing to note that we
    // are looping over the suffixes in the order they occur in the underlying
    // string. This order is different from the order in which they occur in
    // the suffix array. This means that we will not fill the `lcp_array`
    // in order
    for i in 0..s.len()-1 {
        let cur_suffix_index = SuffixIndex(i);
        // We are currently processing the suffix that starts at index `i` in
        // the underlying string. We'd like to know where this suffix
        // is located in the lexicographically sorted suffix array
        let location_in_sa = suffix_index_to_rank.get(&cur_suffix_index).unwrap();

        // grab a hold of the id of the suffix that is just before `location_in_sa`
        // in the suffix array
        let left_adjacent_suffix = sa.get_suffix_idx_at(location_in_sa.0 - 1);

        // Here, we compute the length of the longest common prefix between
        // the current suffix and the suffix that is left of it in the suffix
        // array
        while s_ascii[cur_suffix_index.0 + h] == s_ascii[left_adjacent_suffix.0 + h] {
            h += 1
        }
        lcp_array[location_in_sa.0 - 1] = (left_adjacent_suffix, cur_suffix_index, h).into();

        // When we move from i to i+1, we are effectively moving from processing the suffix
        // A[i..] to processing the suffix A[i+1..]. Notice how this is the same as moving
        // to processing a suffix formed by dropping the first character of A[i..]. Therefore,
        // Theorem 1 from Kasai et al. tells us that the lcp between the new shorter suffix
        // and the suffix adjacent to its left in the suffix array is at least `h-1`
        if h > 0 {
            h -= 1
        }
    }
    lcp_array
}
```


#### The Suffix Array: A Linear Time Solution
In the first section, we implemented a suffix array construction algorithm (SACA) that worked by sorting the suffixes. During that discussion, we noted that the runtime of that scheme is lower bounded by the time it takes to sort the suffixes. For long sequences, this time can be quite large. For example, may want to build a suffix array of the human genome approx: 3 bilion characters. Can we do better? [Can we shave off a log factor](https://github.com/jlikhuva/blog/blob/main/posts/rmq.md#the-method-of-four-russians)? Yes. Yes we can. We won't use the method of four russians though (I should note that sometimes whenever I stare at SA-IS, the algorithm we're about to discuss, I'm almost convinced that it can be characterized using the method of four russians). 

**SA-IS: A suffix array via Induced Sorting:**
What is `induced sorting?` and how does it differ from normal sorting? The word `induce` in the title of this procedure refers to inductive reasoning or, more plainly, inference. `induced sorting` is thus sorting by inference. Note that I'm using the term `inference` in its natural language sense, not its statistical sense. As we shall see, in induced sorting, we are able to infer the order of certain suffixes once we know the order of some specific suffixes. This means that we can sort without comparisons and can thus beat the `n lg n` lower bound that hamstrung the naive SACA method. 
```rust
/// WIP: SA-IS
```

