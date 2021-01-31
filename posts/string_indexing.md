# String Indexing: A Rust Implementation
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

**[Kasai's Procedure:](http://web.cs.iastate.edu/~cs548/references/linear_lcp.pdf)** The main reason why the naive solution is sub-optimal is the inner loop. If we could somehow reduce the time needed to compute `LCP` values, we could markedly improve the overall runtime. The first thing to observe is that when implementing the naïve procedure, we iterated over the suffix array -- not the underlying string. Because of this,  we are unable to exploit the fact that the only difference between two suffixes <!-- $S_i, \text { and } S_{i + 1}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/cUBE3Nhoei.svg"> that are adjacent to each other _in the string_ is that one has one more character at the start. Suppose we have already found the `lcp` length between `S_i` and the `S_k`, a suffix adjacent to it _in the suffix array_, to be `h > 1`. How can we use this information to calculate the lcp length between <!-- $S_{i +1}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/dXwLj9E9Lo.svg"> and `S_j`, the suffix adjacent to it in the suffix array? The key insight stems from observing that if we delete the first character from `S_i` we get <!-- $S_{i +1}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/dXwLj9E9Lo.svg">, and since `h > 1` deleting that character from `S_j` yields another suffix that is adjacent to <!-- $S_{i +1}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/dXwLj9E9Lo.svg"> with overlap in at least `h-1` location. Therefore, to calculate the `lcp` between the shorter suffixes, we do not need to compare the first `h-1` suffixes for we already know that they are the same. This effectively reduces the number of times the inner loop iterates and results in a linear time solution. See the linked paper for a proof of correctness and runtime analysis. We implement this scheme below.
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


#### The Suffix Array: A Linear Time Solution[WIP]
In the first section, we implemented a suffix array construction algorithm (SACA) that worked by sorting the suffixes. During that discussion, we noted that the runtime of that scheme is lower bounded by the time it takes to sort the suffixes. For long sequences, this time can be quite large. For example, we may want to build a suffix array of the human genome approx: 3 bilion characters. Can we do better? [Can we shave off a log factor](https://github.com/jlikhuva/blog/blob/main/posts/rmq.md#the-method-of-four-russians)? Yes. Yes we can. We won't use the method of four russians though (I should note that sometimes whenever I stare at SA-IS, the algorithm we're about to discuss, I'm almost convinced that it can be characterized using the method of four russians). 

##### [SA-IS: A suffix array via Induced Sorting](https://ieeexplore.ieee.org/document/4976463)
What is `induced sorting?` and how does it differ from normal sorting? The word `induce` in the title of this procedure refers to inductive reasoning or, more plainly, inference. `induced sorting` is thus sorting by inference. Note that I'm using the term `inference` in its natural language sense, not its statistical sense. As we shall see, in induced sorting, we are able to infer the order of certain suffixes once we know the order of some specific suffixes. This means that we can sort without comparisons and can thus beat the `n lg n` lower bound that hamstrung the naive SACA method. 

###### Foundational Concepts
Below, we briefly discuss some key ideas that we need in order to fully understand the SA-IS procedure.

**L-Type & S-Type Suffixes:** A suffix starting at some position `k` in some text `T` is an `S-type` suffix if: `T[k] < T[k + 1]` OR `T[k] == T[K + 1] AND k + 1 is an S-type suffix` OR `T[k] = $`, the sentinel character. Similarly, A suffix starting at some position `k` in some text `T` is an `L-type` suffix if `T[k] > T[k + 1]` OR `T[k] == T[K + 1] AND k + 1 is an L-type suffix`. 

**LMS Suffixes and Substrings:** An `S` type suffix is said to be a Left Most S-suffix (LMS suffix for short) if it is an `S` type suffix that has an `L` type suffix as its left neighbor. The sentinel `$` is an `LMS` suffix by definition. An `LMS substring` is a contiguous slice of the underlying string that starts at the starting index of some `LMS` suffix and runs up to the start of the next, closest `LMS` suffix.

What's the purpose of all these concepts? Well, SA-IS is based on two key ideas. The first one is that if we know the locations of the `LMS` suffixes in the suffix array, then we can infer the location of all the other suffixes (induced sorting). The second one is divide and conquer. Since SA-IS is a divide an conquer method it needs a way of reducing the problem space. This is called `substring renaming` in the literature. Since `LMS` suffixes are sparsely distributed in a string, SA-IS leverages the substrings that they produce (`LMS Substrings`) to reduce the problem size. We shall discuss how `substring renaming` is done later on. For now, let us introduce abstractions that allow us to work with the concepts we have seen so far.
```rust
#[derive(Debug)]
pub struct Suffix<'a> {
    start: SuffixIndex,
    suffix_type: SuffixType,
    underlying: &'a str,
}

impl<'a> Suffix<'a> {
    /// Is this suffix a left most `S-type` suffix?
    pub fn is_lms(&self) -> bool {
        match self.suffix_type {
            SuffixType::L => false,
            SuffixType::S(lms) => lms,
        }
    }
}

impl<'a> From<(SuffixIndex, SuffixType, &'a str)> for Suffix<'a> {
    /// Turn a 3-tuple into a Suffix object
    fn from((start, suffix_type, underlying): (SuffixIndex, SuffixType, &'a str)) -> Self {
        Suffix {
            start,
            suffix_type,
            underlying,
        }
    }
}

#[derive(Debug, PartialEq, Eq, Clone)]
enum SuffixType {
    /// A variant for S-type suffixes. The associated boolen
    /// indicates whether this suffix is an `LMS` suffix. We
    /// discuss what that means below.
    S(bool),

    /// A variant for L-type suffixes
    L,
}

impl<'a> SuffixArray<'a> {
    /// Create a list of the suffixes in the string. We scan the underlying string left to right and
    /// mark each corresponding suffix as either `L` or `S(false)`. Then we do a second right to left
    /// scan and mark all LMS suffixes as `S(true)`. We also keep track of the locations of the lms
    /// suffixes
    fn create_suffixes(underlying: &'a str) -> (Vec<Suffix>, Vec<SuffixIndex>) {
        let s_len = underlying.len();
        let mut tags = vec![SuffixType::S(false); s_len];
        let mut suffixes = Vec::with_capacity(s_len);
        let mut lms_locations = Vec::with_capacity(s_len / 2);
        let s_ascii = underlying.as_bytes();

        // We tag each chatacter as either `S` type or `L` type. Since
        // we initialized everything as `S` type, we only need to mark
        // the `L` type suffixes
        for i in (0..s_len - 1).rev() {
            let (cur, next) = (s_ascii[i], s_ascii[i + 1]);
            if (cur > next) || (cur == next && tags[i + 1] == SuffixType::L) {
                tags[i] = SuffixType::L;
            }
        }
        // The first character can never be an `lms` suffix, so we skip it
        // Similary, the last character, which is the sentinel `$` is definitely
        // an lms suffix, so we deal with it outside the loop
        for i in 1..s_len - 1 {
            match (tags[i - 1].clone(), &mut tags[i]) {
                // If the left character is the start of an `L-type` suffix
                // and the current character is an `S-type` suffix, then
                // we mark the current suffix as an `lms` suffix. We
                // ignore all other cases
                (SuffixType::L, SuffixType::S(is_lms)) => {
                    *is_lms = true;
                    lms_locations.push(SuffixIndex(i));
                }
                _ => {}
            }
        }
        tags[s_len - 1] = SuffixType::S(true);
        lms_locations.push(SuffixIndex(s_len - 1));

        // Now that we have all the suffix tags in place, we can construct
        // the list of suffixes in the string
        for (i, suffix_type) in tags.into_iter().enumerate() {
            suffixes.push((SuffixIndex(i), suffix_type, underlying).into())
        }
        (suffixes, lms_locations)
    }
}

```
**Buckets:** A bucket is a contiguous region of the suffix array where all suffixes begin with the same character. That starting character serves as the label for that bucket. Buckets are important in SA-IS because, as we'll soon see, we use them for induced sorting. Specifically, by placing `LMS` suffixes in their buckets at the right locations, we are able to infer the appropriate locations of the other suffixes in those buckets. Below we introduce the abstraction for a bucket
```rust
/// The first character of the suffixes in a bucket
/// uniquely identifies that bucket
#[derive(Eq, PartialEq, Hash, Debug)]
pub struct BucketId<T>(T);

#[derive(Debug)]
pub struct Bucket {
    /// Index in suffix array where this bucket begins
    start: SuffixArrayIndex,

    /// Region in suffix array where this bucket ends.
    /// Note that these indexes are 0-based and inclusive
    end: SuffixArrayIndex,

    /// Tells us how many suffixes have been inserted at the start
    /// of this bucket. Doing `start + offset_from_start` gives us
    /// the next empty index within this bucket from the start
    /// The utility of this will become clear once we look at
    /// the mechanics of induced sorting
    offset_from_start: usize,

    /// Used in a similar manner as `offset_from_start` just
    /// from the end index
    offset_from_end: usize,

    /// Used for inserting `lms` suffixes in a bucket
    lms_offset_from_end: usize,
}

impl<'a> Suffix<'a> {
    pub fn get_bucket(&self) -> BucketId<u8> {
        let first_char = self.underlying.chars().nth(self.start.0);
        debug_assert!(first_char.is_some());
        BucketId(first_char.unwrap() as u8)
    }
}
```

**The Alphabet <!-- $\Sigma$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/pqSc9UzkQu.svg">**: An alphabet, simply put, is the ordered unique list of all the characters that can ever appear in our strings. For instance, if our problem domain is genomics, then our alphabet could be `{A, T, U, C, G}`. If it is proteomics, the alphabet could be the 20 amino acids. For our case, our alphabet is the `256` ascii characters. To create a bucket, we need to know where it starts and where it ends. For each unique character in the underlying string, we can obtain this information by leveraging the associated alphabet to implement a modified version of couting sort. In particular, we start by keeping a count of how many times each character appears in the underlying string — just as we do in counting sort. We then scan across this array of counts using the count information to generate our buckett. We implement this functionality below.
```rust
pub struct AlphabetCounter([usize; 1 << 8]);
impl AlphabetCounter {
    /// Initialize the counter from a string slice. This procedure expects
    /// the give slice to only have ascii characters
    pub fn from_ascii_str(s: &str) -> Self {
        let mut alphabet = [0_usize; 1 << 8];
        for byte in s.as_bytes() {
            alphabet[*byte as usize] += 1;
        }
        Self(alphabet)
    }

    pub fn create_buckets<'a>(&self, array: &'a [SuffixIndex]) -> HashMap<BucketId<u8>, Bucket> {
        let mut buckets = HashMap::new();
        let mut start_location = 0;
        let alphabet_counter = self.0;
        for i in 0..alphabet_counter.len() {
            if alphabet_counter[i] > 0 {
                let end_location = start_location + alphabet_counter[i] - 1;
                let bucket = Bucket {
                    start: SuffixArrayIndex(start_location),
                    end: SuffixArrayIndex(end_location),
                    offset_from_end: 0,
                    offset_from_start: 0,
                    lms_offset_from_end: 0,
                };
                buckets.insert(BucketId(i as u8), bucket);
                start_location = end_location + 1;
            }
        }
        buckets
    }
}
```
**Induced Sorting Part One:** Once we know the nature of all our suffixes and the locations of all our buckets, we can begin slotting suffixes in place. First, we place all `lms` suffixes in their buckets. Because they are `S` type suffixes, we now that they will occupy the latter portions of their respective buckets. Why is this so? Well, think about how the suffixes in a given bucket compare if we exclude the first character. Once we have all the `lms` suffixes in position, we proceed to place `L` type suffixes at their appropriate location, after which we do the same for `S` type suffixes. We implement this logic below.
```rust
impl Bucket {
    /// Put the provided s_type suffix into its rightful location.
    /// In a bucket, S-type suffixes appear after all L-Type suffixes
    /// because they are lexicographically larger. Furthermore, in a given bucket,
    /// `lms` suffixes are larger than all other suffixes.
    fn insert_stype_suffix(&mut self, suffix: &Suffix, sa: &mut Vec<SuffixIndex>) {
        sa[self.end.0 - self.offset_from_end] = suffix.start.clone();
        self.offset_from_end += 1;
    }

    fn insert_lms_suffix(&mut self, suffix: &Suffix, sa: &mut Vec<SuffixIndex>) {
        sa[self.end.0 - self.lms_offset_from_end] = suffix.start.clone();
        self.lms_offset_from_end += 1;
    }

    /// Put the provided l_type suffix in its approximate location in this
    /// bucket
    fn insert_ltype_suffix(&mut self, suffix: &Suffix, sa: &mut Vec<SuffixIndex>) {
        sa[self.start.0 + self.offset_from_start] = suffix.start.clone();
        self.offset_from_start += 1;
    }
}

type Buckets<'a> = HashMap<BucketId<u8>, Bucket<'a>>;

impl<'a> SuffixArray<'a> {
    fn induced_lms_sort(s: &'a str, buckets: &mut Buckets, sa: &mut Vec<SuffixIndex>) {
        let (suffixes, lms_locations) = Self::create_suffixes(s);
        // Place LMS suffixes in position.
        for lms_idx in lms_locations {
            let cur_lms_suffix: Suffix = (lms_idx, SuffixType::S(true), s).into();
            let id = cur_lms_suffix.get_bucket();
            buckets.get_mut(&id).unwrap().insert_lms_suffix(&cur_lms_suffix, sa);
        }

        // 2. Place L-type suffixes in position
        for suffix in &suffixes {
            if suffix.suffix_type == SuffixType::L {
                let id = suffix.get_bucket();
                buckets.get_mut(&id).unwrap().insert_ltype_suffix(&suffix, sa);
            }
        }

        // 3. Place S-type suffixes in position. This operation
        //    may change the location of the `lms` suffixes that
        //    (1) ordered.
        for suffix in suffixes.iter().rev() {
            if suffix.suffix_type != SuffixType::L {
                let id = suffix.get_bucket();
                buckets.get_mut(&id).unwrap().insert_stype_suffix(&suffix, sa);
            }
        }
    }
```
**Substring Renaming:** In substring renaming, we'd like to reduce the size of our input. We do so by forming a new string out of our `LMS substrings`. That is, all the characters that belong to a single substring are reduced to a single label. We implement substring renaming below. 
```rust
/// WIP
```
After obtaining the shorter substring, we check to see if it contains any repeated labels. If it does not, then we are done, we can go ahead and create a sufffix array for the reduced string. If it contains duplicates, we recurse on the reduced string and alphabet. In the next section we discuss how to compute the suffix array given the suffix array of the reduced string.

**Inducing the SA from an approximate SA:**  
```rust
/// WIP
```
##### The Whole Enchilada
```rust
/// WIP
```

#### References
1. [CS 166 Lecture 3](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/03/Small03.pdf)
1. [CS 166 Lecture 4](http://web.stanford.edu/class/archive/cs/cs166/cs166.1196/lectures/04/Small04.pdf)
2. [This Exposition](https://zork.net/~st/jottings/sais.html#induced-sorting-l-type-suffixes)

###### Cite As
```latex
@article{jlikhuva2021string_indexing,
  title   = "String Indexing: A Rust Implementation.",
  author  = "Okonda, Joseph",
  journal = "https://github.com/jlikhuva/blog",
  year    = "2021",
  url     = "https://github.com/jlikhuva/blog/blob/main/posts/string_indexing.md"
}
```