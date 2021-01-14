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
#[derive(Debug, PartialEq, Eq, Hash, Clone)]
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
Suffix arrays are fairly useful on their own. However, they become even more useful when coupled with information about the lengths of common prefixes. In particular, the time to do substring matching is greatly reduced if we know the length of the longest common prefixes between all pairs of adjacent suffixes in the suffix array. How can we get that information? Let's explore that next.

#### The Suffix Array: A Liner Time Solution
In the first section, we implemented a suffix array construction algorithm (SACA) that worked by sorting the suffixes. During that discussion, we noted that the runtime of that scheme is lower bounded by the time it takes to sort the suffixes. For long sequences, this time can be quite large. For example, may want to build a suffix array of the human genome approx: 3 bilion characters. Can we do better? [Can we shave off a log factor](https://github.com/jlikhuva/blog/blob/main/posts/rmq.md#the-method-of-four-russians)? Yes. Yes we can. We won't use the method of four russians though (I should note that sometimes whenever I stare at SA-IS, the algorithm we're about to discuss, I'm almost convinced that it can be characterized using the method of four russians). 

**SA-IS: A suffix array via Induced Sorting**

