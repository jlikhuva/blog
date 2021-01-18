# Graphs: All Foundational Methods
In this note, we shall take a detailed look at perhaps the most versatile combinatoric structure: the graph. We'll examine key foundational ideas that allow us to reason about graphs. Although this note aims to be as detailed as possible, it does not seek to be comprehensive. To that end, we constrain ourselves to ideas that one would find in an introductory algorithms textbook (with the observation that all other complex methods are derived from these ideas). In particular, we deeply explore ideas and methods presented in `CLRS 22 – 26`, with occassional references to interesting blog posts.

#### Type States In Rust
Before we begin, let's briefly explore a neat idea that we'll make use of later on. When writing programs, we often have a set of invariants that once broken we end up with an illegal program – so to speak. For instance, if you're working with files, you cannot read from a file that was not previously opened. Similary, you cannot read from a file that has already been closed. Furthermore, we need to ensure that we close our files once we're done with them  so as to avoid leaking memory. In languages without a rich type system, such invariants are enforced by doing runtime error checks. Languages with a rich type system, on the other hand, allow us to encode these invariants as types. This then ensures that operations that may result in our program entering an illegal state do not compile. We thus make illegal states unrepresentable. For a better, and much more refined discussion of the type state programming pattern, please refer to these two excelent notes: [Will Crichton's CS 242 Lecture Note](https://stanford-cs242.github.io/f19/lectures/08-2-typestate) and [this blog post from the systems group at ETH Zurich](https://blog.systems.ethz.ch/blog/2018/a-hammer-you-can-only-hold-by-the-handle.html). We'll only make use of this rudimentary understanding when implementing our graph data structure.

#### Representing Graph with Indexes
Graphs are complex because they involve cyclic ownership where multiple variables may need mutable access to a single recource. This can be represented in Rust using reference counting via `std::Rc` coupled with `RefCell`. This, however, introduces complexity that I'm quite frankly not ready to deal with. Furthermore, I think it's quite inelegant. Therefore, in this note, we'll instead represent our graphs using indexes as explained [here](http://smallcultfollowing.com/babysteps/blog/2015/04/06/modeling-graphs-in-rust-using-vector-indices/). Note that this is also the approach we took when discusing [the bottom up splay tree](https://github.com/jlikhuva/blog/blob/main/posts/splay.md). 

We shall keep all the nodes in the graph in a single vector. We'll also store our edges in a separate vector. To index into these containers, we could simply use a raw `usize`. Doing so, however, has a number of limitations, the main one being that given a `usize` we have no idea if it refers to an edge or a node. One has to be super vigilant not to mistakenly use an index meant for an edge to index into the nodes vector. This vigilance imposes a cognitive load that we'd like to offload to the compiler via the type system. We do so by using the [wrapped index pattern](https://matklad.github.io/2018/06/04/newtype-index-pattern.html). Below, we use this pattern to introduce the core abstractions that we'll be using going forward.

```rust
/// WIP: Graph
```
#### Adding and Removing Nodes
```rust
/// WIP
```
#### Adding and Removing Edges 
```rust
/// WIP
```

### Methods on Graphs
#### SSSP
1. ###### BFS
2. ###### Relaxation
3. ###### Dijkstra's Algorithm
4. ###### Bellman & Ford's Algorithm 
   
#### Topological Sorting & SCC
1. ###### DFS
2. ###### DFS Applied: Topological Sorting
3. ###### Graphs Cuts
4. ###### Graph Cuts + DFS: SCC
   
#### Minimum Weight Spanning Trees
1. ###### Prim's Algorithms
2. ###### Kruskal's Algorithm

#### APSP
1. ###### Matrix Multiplication & Repeated Squaring
2. ###### Floyd & Warshall's Algorithm
3. ###### Johnson's Algorithm

#### Maximum Flow in Graphs
1. ###### A Naïve Greedy Procedure
2. ###### Ford & Fulkerson's Algorithm
3. ###### The Push-Relabel Method
