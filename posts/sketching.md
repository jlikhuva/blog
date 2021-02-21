# Sketching Algorithms

All algorithms operate on some data. Traditional algorithms assume that they have full access to the data they need in order to solved the problem at hand. For instance, algorithms for sorting a collection of items naturally assume that they have access to a buffer holding all the items. Similarly, procedures for computing the convex hull of a set of points assume that they have all the points. In some applications, however, the full set of data is not known a priori. In this _online setting_, algorithms have to provide useful answers to queries before having the chance to see all the data.

The way such algorithms provide their answers depends on the kind of query being asked. Queries can be `ad hoc` meaning that we do not know the exact query we'd like to run a priori. In this case, the most common technique used is `random sampling`. We can also have `standing queries`. In this case, we know exactly the queries that we'd like to answer. Because we know the queries beforehand, we can design specialized data structures capable of providing useful answers to our queries.

The two cases discussed above share two key characteristics:

1. The answers that they produce will be approximate — not exact.
2. They do not seek to store and process all the incoming data. Instead, their goal is to quickly process each observation to create a summary that they can use to answer either standing or ad hoc queries. This summary is often referred to as a Sketch of the data.

In this note, we discuss and implement several ideas that are useful when designing online algorithms. We begin by discussing a classic method for selecting a representative sample from a stream of data. We then discuss the idea that is foundational to most sketching algorithms and show how it's employed to improve the accuracy of approximate online methods. Finally, we discuss a few key algorithms for answering some standing queries.

## Reservoir Sampling

Reservoir Sampling is a simple procedure for picking a random uniform sample of size `k` from a datastream whose size is unknown (it could be infinite). This scheme works as follows: We store the first `k` items into a buffer of size `k`. Then for each element that arrives afterwards, with probability <!-- $\dfrac{k}{i}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/w53pBCBLXV.svg"> we decide to keep the <!-- $i_{th}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/XptcTto55U.svg"> sample. If we keep it, we pick an element, uniformly at random, i.e with probability <!-- $\dfrac{1}{k}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/ZAhf8BKElA.svg"> to evict from the buffer and replace it with the new sample. We implement this scheme below.

First, since we may be implementing other samplers later on, we introduce a trait that encapsulates the behavior that we expect from any sampler. Put simply, a sampler is anything that can observe a stream of events and produce a sample from that stream at any time

```rust
use rand::{thread_rng, Rng};

/// A sampler will be anything that can observe a possibly infinite
/// number of items and produce a finite random sample from that
/// stream
pub trait Sampler<T> {
    /// We observe each item as it comes in
    fn observe(&mut self, item: T);

    /// Produce a random uniform sample of the all the items that
    /// have been observed so far.
    fn sample(&self) -> &[T];
}
```

With that out of the way, we can now introduce the reservoir sampling data structures and procedures. The reservoir sampler only needs to keep track of two values: the number of events that have been observed so far and the requested sample size.

```rust
#[derive(Debug)]
pub struct ReservoirSampler<T> {
    /// This is what we produce whenever someone calls sample. It
    /// maintains this invariant: at any time-step `t`, reservoir
    /// contains a uniform random sample of the elements seen thus far
    reservoir: Vec<T>,

    /// This determines the size of the reservoir. The
    /// client sets this value
    sample_size: usize,

    /// The number of items we have seen so far. We use this
    /// to efficiently update the reservoir in constant time
    /// while maintaining its invariant.
    count: usize,
}
impl<T> ReservoirSampler<T> {
    /// Create a new reservoir sampler that will produce a random sample
    /// of the given size.
    pub fn new(sample_size: usize) -> Self {
        ReservoirSampler {
            reservoir: Vec::with_capacity(sample_size),
            sample_size,
            count: 0,
        }
    }
}
```

Finally, we turn our `ReservoirSampler` object into a sampler by implementing the appropriate trait

```rust
impl<T> Sampler<T> for ReservoirSampler<T> {
    fn observe(&mut self, item: T) {
        // To make sure we that maintain the reservoir invariant,
        // we have to ensure that each incoming item has an equal
        // probability of being included in the sample. We do so by
        // generating a random index `k`, and if `k` falls within
        // our reservoir, we replace the item at `k` with the
        // new item
        if self.reservoir.len() == self.sample_size {
            let rand_idx = thread_rng().gen_range(0..self.count);
            if rand_idx < self.sample_size { 
                self.reservoir[rand_idx] = item;
            }
        } else {
            // If the reservoir is not full, no need
            // to evict items
            self.reservoir.push(item)
        }
        self.count += 1;
    }

    fn sample(&self) -> &[T] {
        // Since we always have a random sample ready to go, we
        // simply return it
        self.reservoir.as_ref()
    }
}
```

You can find runnable code for the reservoir sampling procedure [in the playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=70732619db2d901ea9bdf832793a9563)

## Foundational Ideas

Before we begin discussing strategies for answering standing queries on streaming data, let's first lay the foundation by exploring two key ideas. These ideas are the major motifs that you'll encounter in almost all sketching algorithms.

### Hashing

Although hashing is mostly known as the first key ingredient in the design of hash tables ([the other half being techniques for resolving collisions](https://www.notion.so/Hashing-The-Cutting-Room-Floor-0dd4f5280f3c4ac88d3823a35336f5fa)), it is also quite useful when designing probabilistic sketching algorithms. In this setting, we use hashing to introduce randomness. To see why randomness is important, consider an alternate method for sampling `k` items from a possibly infinite data stream. We could accomplish this by tagging each incoming item with a random number then storing the items and their tags in a bounded heap. Once the heap reaches capacity, we evict the item with the smallest tag before adding in a new `<item, tag>` pair.

At this point, we have a vague idea of why hashing is a useful idea. But, what is hashing — really? [Put simply](https://gankra.github.io/blah/robinhood-part-1/#hashing), hashing takes an arbitrary input (often as a collection of bytes) and transforms it, using a hash function, into a random integer. How is this different from tagging each input with a random number as we did above? Well, if the hash function is _well constructed_, and the functions co-domain is of size `m`, then the probability that two distinct items receive the same tag is <!-- $\frac{1}{m}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/BLfyb0DTU5.svg">. A hash function that provides thus guarantee is called a universal hash function. In this note, we assume that all our hash functions are universal.

In addition to universality, there are other details that are cool to know about hash functions. First, there exists a tradeoff between the speed of a hash function and its security guarantees. Second, when hashing is used as a component of systems that  have exposure to potential adversaries, it as an attack vector — via hash-flooding attacks. This is why we almost always use a randomly initialized hash function. We won't make use of these last two ideas in this note, so we don't discuss them further. For a much more detailed examination fee free to take a look at [this write-up](https://gankra.github.io/blah/robinhood-part-1/) and this other [rough write up](https://paper.dropbox.com/doc/CS-Knowledge-Base--kehJSKrL5YSH8Ws59M1wb#:h2=Hash-Tables-+-Linear-Probing-+)

Below, we explore the hashing interface provided by the rust programming language.

```rust
//! Rust exposes hashing utilities via a Streaming Hasher interface. That is, you
//! feed arbitrary bytes into the hash function, then, when done, you ask
//! the hasher to produce a hash value.
//!
//! use std::hash::{Hash, Hasher};

/// The Hasher trait from the standard library. This is implemented
/// by hash functions
pub trait Hasher {
    /// Writes the given list of bytes into this hasher. This usually
    /// updates some internal state. Because the hasher is stateful,
    /// we can keep on calling `write` until we've exhausted all our
    /// bytes
    fn write(&mut self, bytes: &[u8]);

    /// Produces the hash value of the bytes fed into this hasher so far
    /// without resetting the hasher's internal state. This means that
    /// if we want a hash value for a different stream of bytes, we have to
    /// create a new hasher instance.
    fn finish(&self) -> u64;

    // ... snip
}

/// The Hash trait from the standard library. This is implemented by hashable
/// types
pub trait Hash {
    /// Feed the value into a given Hasher
    fn hash<H: Hasher>(&self, state: &mut H);

    // ... snip
}
```

The long and short of it is, we create a `hasher` from some random state — `random_state.build_hasher()`. Then, given an item that implements the `Hash` trait, we call its `hash` function passing in the hasher — `item.hash(hasher)`. Then to extract the hash value, we call the `finish` method of the hasher. The tricky bit to remember is that calling finish does not reset the hasher's internal state. As we'll see later on, this means that we have to create a new `hasher` instance to hash a fresh item.

### Fingerprints & Probability Amplification

Once we apply a hash function to an item, we can use the generated hash value to _almost_ uniquely identify that object. The hash value can thus be thought of as a fingerprint of the initial object in that it is a relatively unique and lightweight identifier of the object -- just like human fingerprints. Because of hash collisions, however, it is not fully unique. When designing probabilistic data structures, we strive to reduce the likelihood of two distinct items colliding. That is where probability amplification comes in. Instead of simply hashing an item once, we rerun the hashing experiment `k` times, **each time with a different, independent, random hash function**.

Now, the fingerprint is composed of the `k` hash values. This scheme dramatically reduces the probability of fingerprint collision. In particular, the probability that two fingerprints from two different objects are the same is <!-- $\frac{1}{m^k}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/hmoWA7fcKW.svg">. where `m` is the size of the co-domain of our hash functions (this is often the number of buckets we're hashing our items into).

<!-- To really appreciate how powerful these two ideas are, let's look at a quick example borrowed from [CS 168](https://web.stanford.edu/class/cs168/l/l4.pdf). Suppose we have `n` objects from a universe `U`. Each object requires $\lg_2 |U|$ bits to uniquely identify. For instance, if our universe is the address space on a 64 bit machine, each address would require $\lg_ 2 2^{64} = 64$ bits. Now, suppose that we feel that `64` bits are too many, and thus want to represent each address using fewer bits. How could we go about this? -->

To recapitulate, we can represent arbitrary objects using their hash values. These values are often smaller (e.g 8 bytes) than the underlying objects. Furthermore, instead of simply using a single hash value, we can use a collection of `k` hash values each produced by `k` independent hash functions.

## Bloom Filters

A Bloom Filter 🥦  is a compact data structure that summarizes a set of items, allowing us to answer membership questions — has a certain item been seen before? Unlike other set structures — such as hash sets and search trees — a bloom-filter’s answer to a membership query is either a `definite NO` or a `Probable Yes`. That is, it can have false positives — telling us that an item has been seen when in fact it has never been observed. Therefore, Bloom filters are best suited for cases where false positives can be tolerated and mitigated. Cases where the effect of a false positive is not a wrong program state but extra work.  

Below, we introduce the interface shared across all filters capable of answering approximate membership queries.

```rust
/// Indicates that the filter has probably seen a given
/// item before
pub struct ProbablyYes;

/// Indicates that a filter hasn't seen a given item before.
pub struct DefinitelyNot;

/// A filter will be any object that is able to observe a possibly
/// infinite stream of items and, at any point, answer if a given
/// item has been seen before
pub trait Filter<T> {
    /// We observe each item as it comes in. We do not use terminology such as
    /// `insert` because we do not store any of the items.
    fn observe(&mut self, item: T);

    /// Tells us whether we've seen the given item before. This method
    /// can produce false positives. That is why instead of returning a
    /// boolean, it returns `Result<ProbablyYes, DefinitelyNot>`
    fn has_been_observed_before(&self, item: &T) -> Result<ProbablyYes, DefinitelyNot>;
}
```

A Bloom Filter is parameterized by two values: a bit array `buckets` with `m` buckets in it, and a set of `k` hash functions <!-- $\{h_1, h_2, \ldots h_k\}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/1kMHSWazJW.svg">. Given an input `item`, we apply each of our `k` hash functions to it and then set all the indexes that `item`  hashed to to `true`.  To check if an item has already been seen, we again first apply each of our `k` hash functions and check if **all** corresponding locations to are `true`.

Notice how a bloom filter is simply a direct application of object fingerprinting and probability amplification. We hash each item using multiple hash functions to reduce the chances of getting a false positive.

Below, we provide an implementation of a bloom filter

```rust
#[derive(Debug)]
pub struct BloomFilter<T: Hash> {
    /// The bit vector. The number of buckets is determined
    /// by the client
    buckets: Vec<bool>,

    /// The list of hash functions. Again, the number of
    /// hash functions is determined by the client
    hash_functions: Vec<RandomState>,

    /// This is here to make the compiler happy. We'd
    /// like for our filter to be parameterized by
    /// the items it's monitoring. However, we do not
    /// store those items in the filter.
    _marker: PhantomData<T>,
}

impl<T: Hash> BloomFilter<T> {
    /// Creates a new bloom filter with `m` buckets and `k` hash functions.
    /// Each hash function is randomly initialized and is independent
    /// of the other hash functions
    pub fn new(m: usize, k: usize) -> Self {
        // initialize all the bucket locations to false
        let mut buckets = Vec::with_capacity(m);
        for _ in 0..m {
            buckets.push(false);
        }

        // initialize the hash functions randomly.
        let mut hash_functions = Vec::with_capacity(k);
        for _ in 0..k {
            hash_functions.push(RandomState::new());
        }

        BloomFilter {
            buckets,
            hash_functions,
            _marker: PhantomData,
        }
    }

    /// This performs the actual hashing.
    fn get_index(&self, state: &RandomState, item: &T) -> usize {
        let mut hasher = state.build_hasher();
        item.hash(&mut hasher);
        let idx = hasher.finish() % self.buckets.len() as u64;
        idx as usize
    }
}
```

With the above abstractions in place, we can go ahead and implement the core filter procedures.

```rust
impl<T: Hash> Filter<T> for BloomFilter<T> {
    /// As already explained, we apply each of our `k` hash functions to the
    /// given item and set the locations it hashes to to true
    fn observe(&mut self, item: T) {
        for state in &self.hash_functions {
            let index = self.get_index(state, &item);
            self.buckets[index] = true;
        }
    }

    /// Again, we start by applying each of our `k` hash functions 
    /// to the given item then. If all resultant `k` locations are `true`,
    /// we conclude that we MIGHT have observed this item. If any location
    /// is false, we immediately conclude that we NEVER saw this item
    fn has_been_observed_before(&self, item: &T) -> Result<ProbablyYes, DefinitelyNot> {
        for state in &self.hash_functions {
            let index = self.get_index(state, &item);
            if !self.buckets[index] {
                return Err(DefinitelyNot);
            }
        }
        Ok(ProbablyYes)
    }
}
```

## The Count-Min Sketch

Count-Min Sketch 🍅 is a compact structure for estimating the counts of each type of items in our set. A naive way of solving the count each problem would be to allocate an integer counter for each class of items. However, this may not be practical when the number of item types grows huge. The CMSketch provides a tradeoff between count accuracy and low memory usage — it encodes a potentially massive number of item types in a small array, guaranteeing that large counts will be preserved fairly accurately while small counts may incur greater relative error. Count-Min sketches are thus best suited to when we can handle a slight inflation in frequency. As with the bloom filter, we use multiple hash functions. However, unlike with the bloom filter where all the functions hashed to the same array, now each functions has its own dedicated set of buckets. A CMSketch is thus a matrix with as many rows as the number of hash functions and as many columns as the number or buckets. When we see an item, we apply our `k` hash functions to it and increment the slots that it maps to in each row of the sketch. To estimate the count of an item, we apply our hash functions and read the values at the slots that it maps to and then select the minimum out of these. One cool thing to note is that the accuracy guarantees of the CMSketch are not in any way related to the size of the dataset, this is in contrast with the guarantees of the bloom filter. They depend entirely on the number of hash functions `k` and the number of buckets `m`

## The Hyper-Log-Log

A method for estimating the cardinality of a multi-set. That is, answering the question, how many distinct items do we have?

## The Johnson-Lindenstrauss Transform

Euclidean Distance , also known as the $\ell_2$ distance, is the most common distance metric. it is defined over vectors of real values. It is defined as $D_\epsilon (x, y) = ||x - y||_2 = \left( \sum_{i = 1}^{d} [x_i - y_i]^2 \right)^{0.5}$ for points $x \in \mathbb{R}^d, y \in \mathbb{R}^d$. This can be generalized by replacing 2 with arbitrary values $p$. 


The Curse of Dimensionality is the observation that, the number of neighbors of any point $p$ is related exponentially to the number of dimensions of our space. In $\mathbb{R}^1, \mathbb{R}^2, \mathbb{R}^3$ we have $p = 2^1, 2^2, 2^3$ respectively. In general, $p = 2^d, \text{ for } \mathbb{R}^d$. This grows so large for moderate values of $d$. Since there is no known way to overcome this curse, we have to resort to using approximate methods. 

At the heart of approximate nearest neighbor methods  lies Dimensionality Reduction . The goal is to project our high dimensional data into a low dimensional space while preserving inter-point distance as much as possible. This would allows have our cake and it it too — we’d be able to richly represent our data in high dimensions and leverage dimensionality reduction to map our data and queries down to a small number of dimensions so as to overcome — approximately, the curse of dimensionality. 

To motivate dimensionality reduction, it is instructive to look at two key foundational techniques — fingerprinting and probability amplification: Suppose we have a set of $n$ objects from a universe $\mathcal{U}$. Note that we need $\log_2|\mathcal{U}|$ bits to distinctly represent any object from this universe. For large $\mathcal{U}$ this may be too many bits. Can we distinctly represent each object with fewer bits?  One simple scheme is to use a single bit using the mapping  $f(x) = h(x) \mod 2$If the hash function is good, we expect $x = y \rightarrow f(x) = f(y)$, i.e equality to be preserved and when  $\text x \neq y \text{ then }Pr[f(x) = f(y)] \leq 0.5$, i.e distinctness is preserved 50% of the time. 
We can improve this last error rate by repeating the hashing experiment $k$ times using $k$ independent hash functions and label each object using $k$ bits — this bitstring is the fingerprint of the original object. This is probability amplification. It dramatically reduces the error rate. With 2 hash functions, the error rate drops down to $0.25$. In general, the error rate is $2^{-k}$.

Random Projections is an extension of  the idea of fingerprinting to approximately preserve the $L_2$ distance between object pairs. Suppose we have $n$ objects of interest, $x_1 \ldots x_n$ in $k$-dimensional space. We can choose a random vector $r = (r_1 \ldots r_k)$ from the same space. If we take the inner product between $r$ and any of our objects, we’ll get a single number $f_r(x_i) = \langle x_i, r\rangle = x^Tr$. We have thus mapped a k-vector to a single real value that is a random linear combination of the elements in the original vector. This is the random projection of the vector.

The value in our random vector can be sampled iid from a standard gaussian. If we do that, the projections of any two vectors will be an unbiased estimate of the euclidean distance between the two vectors. As usual, we can use the magic of independent trials to increase the “accuracy” of our projections. To do so, instead of picking a single random vector, we pick $d$ random vectors and average the random projections we get from them. Averaging $d$ independent unbiased estimates yields an unbiased estimate and drops the variance of our estimate by a factor of $d$.

The Johnson-Lindenstrauss transform (JL) generalizes this idea. It is defined by a $d \times k$ matrix $A$ composed of our $d$ random vectors. The matrix maps vectors in $\mathbb{R}^k$ to vectors in $\mathbb{R}^d$ using the mapping $x_d = \dfrac{1}{\sqrt d}Ax$. We can apply the JL-transform whenever we have high dimensional data and are working an a computation that only cares about euclidean distance. in that case, there’s little loss in doing the computation in $d$-dimensional space.

## Analysis Tools

### Markov's Inequality

### Chebyshev’s  Inequality

## References

1. [CS 168 Lecture 2](https://web.stanford.edu/class/cs168/l/l2.pdf)
2. [CS 168 Lecture 4](https://web.stanford.edu/class/cs168/l/l4.pdf)
3. [This Survey Paper](http://dimacs.rutgers.edu/~graham/pubs/papers/cacm-sketch.pdf)
4. [This Book by Jelani Nelson](https://www.sketchingbigdata.org/fall20/lec/notes.pdf)
