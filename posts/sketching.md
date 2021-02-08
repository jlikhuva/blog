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

Although hashing is mostly known as the first key ingredient in the design of hash tables (the other half being techniques for resolving collisions), it is also quite useful when designing probabilistic sketching algorithms.  Takes an arbitrary input and transforms it to an integer. Universal hashing. Perfect hashing. p f(x) = f(y) if x neq y. Hashing as an attack vector (via hashflooding) and the need for using a random hash function.   vs streaming hash functions. Cryprtographic vs non cryptographic hash functions. Tradeoff between speed and security. Examples of common hash functions (xxhash, murmurhash). Demonstrate the hashing interface of rust.

### Fingerprints & Probability Amplification

Once we apply a hash function to an item, we can use the generated hash value to almost uniquely identify that object. The hash value can thus be thought of as a fingerprint of the initial object in that it is a relatively unique and lightweight identifier of the object -- just like human fingerprints. Because of hash collisions, however, it is not fully unique. Our goal is to reduce the likelihood of a collision. That is where probability amplification comes in. Rerun the hashing experiment `k` times, each time with a different random hash function. Now, the fingerprint is composed of the `k` hash values. This dramatically reduces the probability of fingerprint collision. probability that two fingerprints from two different objects are the same is `m^-k`.

## Bloom Filters

Suppose you wish to compactly represent a set of items in some universe. Options are a hash set

## The Count-Min Sketch

A compact structure for estimating the counts for each type of item

## The Hyper-Log-Log

A method for estimating the cardinality of a multi-set. That is, answering the question, how many distinct items do we have?

## The Johnson-Lindenstrauss Transform
