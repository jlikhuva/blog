# Sketching Algorithms

All algorithms operate on some data. Traditional algorithms assume that they have full access to the data they need in order to solved the problem at hand. For instance, algorithms for sorting a collection of items naturally assume that they have access to a buffer holding all the items. Similarly, procedures for computing the convex hull of a set of points assume that they have all the points. In some applications, however, the full set of data is not known a priori. In this _online setting_, algorithms have to provide useful answers to queries before having the chance to see all the data.

The way such algorithms provide their answers depends on the kind of query being asked. Queries can be `ad hoc` meaning that we do not know the exact query we'd like to run a priori. In this case, the most common technique used is `random sampling`. We can also have `standing queries`. In this case, we know exactly the queries that we'd like to answer. Because we know the queries beforehand, we can design specialized data structures capable of providing useful answers to our queries.

The two cases discussed above share two key characteristics:

1. The answers that they produce will be approximate — not exact
2. They do not seek to store and process all the incoming data. Instead, their goal is to quickly process each observation to create a summary that they can use to answer either standing or ad hoc queries. This summary is often referred to as a Sketch of the data.

In this note, we discuss an implement several ideas that are useful when designing online algorithms. We begin by discussing a classic method for selecting a representative sample from a stream of data. We then discuss the idea that is foundational to most sketching algorithms and show how it's employed to improve the accuracy of approximate online methods. Finally, we discuss a few key algorithms for answering some standing queries.

## Reservoir Sampling

Reservoir Sampling is a simple procedure for picking a random uniform sample of size `k` from a datastream whose size is unknown (it could be infinite). This scheme works as follows: We store the first `k` items into a buffer of size `k`. Then for each element that arrives afterwards, with probability <!-- $\dfrac{k}{i}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/w53pBCBLXV.svg"> we decide to keep the <!-- $i_{th}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/XptcTto55U.svg"> sample. If we keep it, we pick an element, uniformly at random, i.e with probability <!-- $\dfrac{1}{k}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/ZAhf8BKElA.svg"> to evict from the buffer and replace it with the new sample. We implement this scheme below.

```rust
use rand::{thread_rng, Rng};

pub trait Sampler<T> {
    fn observe(&mut self, item: T);
    fn sample(&self) -> &[T];
}

#[derive(Debug, Default)]
pub struct ReservoirSampler<T> {
    reservoir: Vec<T>,
    sample_size: usize,
    count: usize,
}
impl<T: Default> ReservoirSampler<T> {
    pub fn new(sample_size: usize) -> Self {
        ReservoirSampler {
            reservoir: Vec::with_capacity(sample_size),
            sample_size,
            count: 0,
        }
    }
}

impl<T> Sampler<T> for ReservoirSampler<T> {
    fn observe(&mut self, item: T) {
        if self.reservoir.len() == self.sample_size {
            let keep_probability = self.sample_size as f32 / self.count as f32;
            if keep_probability < thread_rng().gen_range(0.0, 1.0) {
                let rand_idx = thread_rng().gen_range(0, self.sample_size);
                self.reservoir[rand_idx] = item;
            }
        } else {
            self.reservoir.push(item)
        }
        self.count += 1;
    }

    fn sample(&self) -> &[T] {
        self.reservoir.as_ref()
    }
}

#[test]
fn test_reservoir_sampler() {
    let mut sampler = ReservoirSampler::new(500);
    for i in 0..1000 {
        sampler.observe(i);
    }
    println!("{:?}", sampler.sample())
}

```

## Foundational Ideas

### Hashing

### Fingerprints

### Probability Amplification

## Bloom Filters

## The Count-Min Sketch

## The Hyper-Log-Log

## The Johnson-Lindenstrauss Transform
