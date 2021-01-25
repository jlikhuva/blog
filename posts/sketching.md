# Sketching Algorithms
All algorithms operate on some data. Traditional algorithms assume that they have full access to the data they need in order to solved the problem at hand. For instance, algorithms for sorting a colletion of items naturally assume that they have access to a buffer holding all the items. Similarly, procedures for computing the covex hull of a set of points assume that they have all the points. In some applications, however, the full set of data is not known apriori. In this _online setting_, algorithms have to provide useful answers to queries before having the chance to see all the data. 

The way such algorithms provide their answers depends on the kind of query being asked. Queries can be `ad hoc` meaning that we do not know the exact query we'd like to run apriori. In this case, the most common technique used is `random sampling`. We can also have `standing queries`. In this case, we know exactly the queries that we'd like to answer. Because we know the queries beforehand, we can design specialized data structures capable of providing useful answers to our queries.

The two cases discussed above share two key characteristics: 
1.  The answers that they produce will be approximate — not exact
2.  They do not seek to store and process all the incoming data. Instead, their goal is to quickly process each observation to create a summary that they can use to answer either standing or ad hoc queries. This summary is often referred to as a Sketch of the data.

In this note, we discuss an implement several ideas that are useful when designing online algorithms. We begin by discussing a classic method for selecting a representative sample from a stream of data. We then discuss the idea that is foundational to most sketiching algothims and show how it's employed to improve the accuracy of approximate online methods. Finally, we discuss a few key algorithms for answering some standing queries.

#### Reservoir Sampling
Reservoir Sampling a simple procedure for picking a random uniform sample of size `k` from a datastream whose size is unknown (it could be infinite). This scheme works as follows: We store the first `k` items into a buffer of size `k`. Then for each element that arrives afterwards, with probability <!-- $\dfrac{k}{i}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/w53pBCBLXV.svg"> we decide to keep the <!-- $i_{th}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/XptcTto55U.svg"> sample. If we keep it, we pick an element, uniformly at random, i.e with probability <!-- $\dfrac{1}{k}$ --> <img style="transform: translateY(0.1em); background: white;" src="../svg/ZAhf8BKElA.svg"> to evict from the buffer and replace it with the new sample. We implement this scheme below.
```rust
/// WIP: Reservoir Sampling
```
#### Foundational Ideas: Hashing, Fingerprints & Probability Amplification

#### Bloom Filters
#### The Count-Min Sketch
#### The Hyper-Log-Log
#### The Johnson-Lindenstrauß Transform