# Genome Assembly

To sequence a genome, scientists begin by collecting tissue samples. The samples are then treated with biochemical agents that break down the DNA in the cells into _very many_ fragments. This is called the _Fragmentation_ step. Sequencing machines produce _reads_ (of the same size) by looking at these fragments. In spite of the rapid advancements in the field, modern sequencing machines are still not able to generate entire genomes in a single read. Therefore, the final step in any sequencing pipeline involves computationally reconstructing the genome from the collection short reads. This final step, called genome assembly,  will be the focus of this note. We'll begin by examining the complexity that arises when trying to reconstruct the genome from _reads_. We'll then explore two ways of characterizing the genome assembly problem using graphs. Eventually, we'll derive and implement an algorithm for genome assembly that depends on finding an Eulerian Path in a De Bruijn Graph.

## The nature of Fragments & Reads

Right off the bat, we can note a few key things. First, we do not know the order in which the reads produced by the sequencing machines occur in the underlying genome. Therefore, during the assembly step, we often begin by ordering the reads lexicographically. Secondly, since DNA is double stranded, we do not know which strand a particular read comes from. This means that we do not know its `5' — 3'` orientation. Finally, the reads produced will contain errors (mostly substitution errors) and will most certainly cover the entire genome.

We can, however, make a few simplifying assumptions. These assumptions, while unrealistic, will allow us to concentrate on the core algorithmic aspects of genome assembly. We'll assume that: (a) all reads are strings of the same length `k`, (b) The reads cover the entire genome, (c) The reads come from the same strand and contain no errors.

At this point, we are ready to introduce our first abstraction — that of a single read. In computational biology s string of a fixed length `k` is often called a `k-mer`.

```rust
/// A K-Mer is simply a string of some given length
#[derive(Debug)]
pub struct KMer(String);

/// Turn a String into a KMer, Consuming the String
/// in the process
impl From<String> for KMer {
    fn from(s: String) -> Self {
        KMer(s)
    }
}

impl KMer {
    /// The length of a k-mer is simply the length
    /// of the underlying string
    pub fn len(&self) -> usize {
        self.0.len()
    }
}
```

## The Overlap Graph

## The De Bruijn Graph

## Euler's Theorem

## Finding Eulerian Cycles
