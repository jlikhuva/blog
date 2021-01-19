# Graphs: All Foundational Methods [WIP: Draft]
In this note, we shall take a detailed look at perhaps the most versatile combinatoric structure: the graph. We'll examine key foundational ideas that allow us to reason about graphs. Although this note aims to be as detailed as possible, it does not seek to be comprehensive. To that end, we constrain ourselves to ideas that one would find in an introductory algorithms textbook (with the observation that all other complex methods are derived from these ideas). In particular, we deeply explore ideas and methods presented in `CLRS 22 – 26`, with occassional references to interesting blog posts.

#### Type States In Rust
Before we begin, let's briefly explore a neat idea that we'll make use of later on. When writing programs, we often have a set of invariants that once broken we end up with an illegal program – so to speak. For instance, if you're working with files, you cannot read from a file that was not previously opened. Similary, you cannot read from a file that has already been closed. Furthermore, we need to ensure that we close our files once we're done with them  so as to avoid leaking memory. In languages without a rich type system, such invariants are enforced by doing runtime error checks. Languages with a rich type system, on the other hand, allow us to encode these invariants as types. This then ensures that operations that may result in our program entering an illegal state do not compile. We thus make illegal states unrepresentable. For a better, and much more refined discussion of the type state programming pattern, please refer to these two excelent notes: [Will Crichton's CS 242 Lecture Note](https://stanford-cs242.github.io/f19/lectures/08-2-typestate) and [this blog post from the systems group at ETH Zurich](https://blog.systems.ethz.ch/blog/2018/a-hammer-you-can-only-hold-by-the-handle.html). We'll only make use of this rudimentary understanding when implementing our graph data structure.

#### Representing Graph with Indexes
Graphs are complex because they involve cyclic ownership where multiple variables may need mutable access to a single recource. This can be represented in Rust using reference counting via `std::Rc` coupled with `RefCell`. This, however, introduces complexity that I'm quite frankly not ready to deal with. Furthermore, I think it's quite inelegant. Therefore, in this note, we'll instead represent our graphs using indexes as explained [here](http://smallcultfollowing.com/babysteps/blog/2015/04/06/modeling-graphs-in-rust-using-vector-indices/). Note that this is also the approach we took when discusing [the bottom up splay tree](https://github.com/jlikhuva/blog/blob/main/posts/splay.md). 

We shall keep all the nodes in the graph in a single vector. We'll also store our edges in a separate vector. To index into these containers, we could simply use a raw `usize`. Doing so, however, has a number of limitations, the main one being that given a `usize` we have no idea if it refers to an edge or a node. One has to be super vigilant not to mistakenly use an index meant for an edge to index into the nodes vector. This vigilance imposes a cognitive load that we'd like to offload to the compiler via the type system. We do so by using the [wrapped index pattern](https://matklad.github.io/2018/06/04/newtype-index-pattern.html). Below, we use this pattern to introduce the core abstractions that we'll be using going forward.

```rust
/// We use the new type index pattern to make our code more
/// understandable. Using indexes to simulate pointers
/// can lead to opaqueness. Using a concrete type instead of
/// raw indexes ameliorates this. The 'insane' derives ensures
/// that the new type has all the crucial properties of the
/// underlying index
pub mod handles {
    use super::{Edge, Node};
    use std::marker::PhantomData;

    /// We use the new type index pattern to make our code more
    /// understandable. This is a handle that can be used to access
    /// nodes
    #[derive(Debug, Clone, Ord, PartialOrd, Eq, PartialEq, Hash)]
    pub struct NodeHandle<T: Ord>(pub(super) usize, pub(super) PhantomData<T>);

    impl<K: Ord, V> std::ops::Index<NodeHandle<K>> for Vec<Node<K, V>> {
        type Output = Node<K, V>;

        /// This allows us to use NodeHandle directly as an index
        fn index(&self, index: NodeHandle<K>) -> &Self::Output {
            &self[index.0]
        }
    }

    impl<K: Ord, V> std::ops::IndexMut<NodeHandle<K>> for Vec<Node<K, V>> {
        fn index_mut(&mut self, index: NodeHandle<K>) -> &mut Self::Output {
            &mut self[index.0]
        }
    }

    /// We use the new type index pattern to make our code more
    /// understandable. This is a handle that can be used to access
    /// edges
    #[derive(Debug, Clone, Ord, PartialOrd, Eq, PartialEq, Hash)]
    pub struct EdgeHandle(pub(super) usize);

    impl<K> std::ops::Index<EdgeHandle> for Vec<Box<dyn Edge<K>>> {
        type Output = Box<dyn Edge<K>>;

        /// This allows us to use NodeHandle directly as an index
        fn index(&self, index: EdgeHandle) -> &Self::Output {
            &self[index.0]
        }
    }

    impl<K> std::ops::IndexMut<EdgeHandle> for Vec<Box<dyn Edge<K>>> {
        fn index_mut(&mut self, index: EdgeHandle) -> &mut Self::Output {
            &mut self[index.0]
        }
    }

    /// A single entry in the tree
    #[derive(Debug)]
    pub struct Entry<K, V> {
        pub(super) key: K,
        pub(super) value: V,
    }

    impl<K: Ord, V> From<(K, V)> for Entry<K, V> {
        /// Allows us to quickly construct an entry from
        /// a key value tuple
        ///
        /// # Example
        ///
        /// ```
        /// let entry: Entry<&str, usize> = ("usa", 245).into();
        /// ```
        fn from((key, value): (K, V)) -> Self {
            Entry { key, value }
        }
    }
}
```
Now that we have a rough idea how we plan to store the nodes and edges, let's introduce types for these structures

```rust
#[derive(Debug)]
struct Node<K: Ord, V> {
    entry: Entry<K, V>,

    /// The start of the list of other nodes in the graph to which
    /// this node has a direct connection
    neighbors_head: Option<EdgeHandle>,
}

impl<K: Ord, V> Node<K, V> {
    pub fn new(entry: Entry<K, V>) -> Self {
        Node {
            entry,
            neighbors_head: None,
        }
    }

    pub fn set_entry(&mut self, entry: Entry<K, V>) {
        self.entry = entry;
    }
}
```

So now we know what our nodes will look like and how we can create them. I have purposely left out any talk of how edges will be represented because it involves a bit of complexity. Since our aim is to be as detailed as possible, we want ensure that our smal library is aware of the 4 major kinds of graphs: Either `weighted or unweighted` and `directed or undirected`. These are properties of the edges in the graph. Furthermore, some methods only apply to specific kinds of graphs. For instance, `BFS` (when used to find shortest paths) assumes that the graph is unweighted whereas all methods for solving flow problems assume that the graph is weighted and directed. We would like to encode all these ideas and constraints using the type system. To that end, we'll have 4 different edge types, one for each of the 4 possible graph types. We'll then use a trait to unify then so that our graph object doesn't have to worry about the different edge types. The snippets below implement these ideas.
```rust

pub mod edge {
    use super::{EdgeHandle, NodeHandle};
    use std::fmt::Debug;
    pub trait Edge<K: Ord>: Debug {
        /// The first end of this edge. If the edge is directed
        /// then this is the source node. If not, then this is
        /// simply one of the nodes forming this edge
        fn first_end(&self) -> &NodeHandle<K>;

        /// The second end of this edge. If the graph is directed, then
        /// this is the destination node.
        fn second_end(&self) -> &NodeHandle<K>;

        /// Since we represent the neighbor list implicitly
        /// in the edges, this method gives us a handle to
        /// the next edge that either emanates from (if directed)
        /// or is connected to (if undirected) the handle returned
        /// by `first_end`
        fn next_neighbor(&self) -> Option<&EdgeHandle>;
    }

    pub trait WeightedEdge<K: Ord>: Edge<K> {
        /// For weighted edges, retrieve their weight
        fn weight(&self) -> f32;
    }

    /// An undirected, unweighted edge
    /// (left_node_idx, right_node_idx)
    #[derive(Debug)]
    pub struct UnweightedUndirected<K: Ord> {
        pub(super) left: NodeHandle<K>,
        pub(super) right: NodeHandle<K>,
        pub(super) next: Option<EdgeHandle>,
    }

    impl<K: Ord + Debug> Edge<K> for UnweightedUndirected<K> {
        fn first_end(&self) -> &NodeHandle<K> {
            &self.left
        }

        fn second_end(&self) -> &NodeHandle<K> {
            &self.right
        }

        fn next_neighbor(&self) -> Option<&EdgeHandle> {
            self.next.as_ref()
        }
    }

    /// A directed, unweighted edge
    /// (src_node_idx, dest_node_idx)
    #[derive(Debug)]
    pub struct UnweightedDirected<K: Ord> {
        pub(super) src: NodeHandle<K>,
        pub(super) dest: NodeHandle<K>,
        pub(super) next: Option<EdgeHandle>,
    }

    impl<K: Ord + Debug> Edge<K> for UnweightedDirected<K> {
        fn first_end(&self) -> &NodeHandle<K> {
            &self.src
        }

        fn second_end(&self) -> &NodeHandle<K> {
            &self.dest
        }

        fn next_neighbor(&self) -> Option<&EdgeHandle> {
            self.next.as_ref()
        }
    }

    /// An undirected, weighted edge
    /// (left_node_idx, right_node_idx, weight)
    #[derive(Debug)]
    pub struct WeightedUndirected<K: Ord> {
        pub(super) left: NodeHandle<K>,
        pub(super) right: NodeHandle<K>,
        pub(super) weight: f32,
        pub(super) next: Option<EdgeHandle>,
    }

    impl<K: Ord + Debug> Edge<K> for WeightedUndirected<K> {
        fn first_end(&self) -> &NodeHandle<K> {
            &self.left
        }

        fn second_end(&self) -> &NodeHandle<K> {
            &self.right
        }

        fn next_neighbor(&self) -> Option<&EdgeHandle> {
            self.next.as_ref()
        }
    }
    impl<K: Ord + Debug> WeightedEdge<K> for WeightedUndirected<K> {
        fn weight(&self) -> f32 {
            self.weight
        }
    }

    /// A weighted, directed edge
    /// (src_node_idx, dest_node_idx, weight)
    #[derive(Debug)]
    pub struct WeightedDirected<K: Ord> {
        pub(super) src: NodeHandle<K>,
        pub(super) dest: NodeHandle<K>,
        pub(super) weight: f32,
        pub(super) next: Option<EdgeHandle>,
    }
    impl<K: Ord + Debug> Edge<K> for WeightedDirected<K> {
        fn first_end(&self) -> &NodeHandle<K> {
            &self.src
        }

        fn second_end(&self) -> &NodeHandle<K> {
            &self.dest
        }

        fn next_neighbor(&self) -> Option<&EdgeHandle> {
            self.next.as_ref()
        }
    }
    impl<K: Ord + Debug> WeightedEdge<K> for WeightedDirected<K> {
        fn weight(&self) -> f32 {
            self.weight
        }
    }
}
```
Talks about graph states. talk about how they aren't really states

```rust
/// Types representing the different types
/// of graphs that we can build
pub mod graph_states {
    pub trait GraphWeight {}
    pub trait GraphDirection {}

    #[derive(Debug)]
    pub struct Directed;
    impl GraphDirection for Directed {}

    #[derive(Debug)]
    pub struct Weighted;
    impl GraphWeight for Weighted {}

    #[derive(Debug)]
    pub struct Undirected;
    impl GraphDirection for Undirected {}

    #[derive(Debug)]
    pub struct Unweighted;
    impl GraphWeight for Unweighted {}
}
```
Say that now we can finally introduce the graph abstraction. Explain what the roles of the mapper are

```rust
#[derive(Debug, Default)]
pub struct Graph<K: Ord + Hash, V, D: GraphDirection, W: GraphWeight> {
    /// We keep all the nodes in the graph in a single list
    nodes: Option<Vec<Node<K, V>>>,

    /// A map from the id of a node to the index at which
    /// it is stored in the list of nodes
    nodes_mapper: Option<HashMap<K, NodeHandle<K>>>,

    /// Similarly, we need all our edges in a single list
    edges: Option<Vec<Box<dyn Edge>>>,

    /// The set of all edges in the graph
    edges_set: Option<HashMap<(NodeHandle<K>, NodeHandle<K>), EdgeHandle>>,

    /// type states
    _kind: (PhantomData<D>, PhantomData<W>),
}
```
### Methods on Graphs
#### Adding and Removing Nodes
Introduce methods that apply to all graph types and are the same regardless. Creating new graph and adding nodes. Say that one could think of introducing a type state like `EmptyGraph` that exposes no methods and transitions to `NonEmptyGraph` once a node is added. But the design is fairly complex already, so we don't do that here. Below we have the logic to creating graph and adding nodes to it

```rust

/// Implementations of procedures that apply to all graph
/// types
impl<K: Ord + Hash + Clone, V, D: GraphDirection, W: GraphWeight> Graph<K, V, D, W> {
    /// Create a new graph instance
    pub fn new() -> Self {
        Graph {
            nodes: None,
            edges: None,
            nodes_mapper: None,
            edges_set: None,
            _kind: (PhantomData, PhantomData),
        }
    }

    /// Adds a new node with the given id into the graph. If the graph
    /// already contains a node with that id, it replaces the old entry.
    /// The procedure returns a handle to the added node.
    pub fn add_node(&mut self, entry: (K, V)) -> NodeHandle<K> {
        let entry = entry.into();
        match &mut self.nodes_mapper {
            None => {
                // The graph is empty. Let's create a new node and add it to the graph.
                let nodes = vec![Node::new(entry)];
                let mut mapper = HashMap::new();
                let new_handle = NodeHandle(mapper.len(), PhantomData);
                mapper.insert(nodes[0].entry.key.clone(), new_handle.clone());
                self.nodes_mapper = Some(mapper);
                self.nodes = Some(nodes);
                new_handle.clone()
            }
            Some(mapper) => {
                match mapper.get(&entry.key) {
                    Some(handle) => {
                        // The graph already contains this node. Lets replace its entry with the new entry
                        self.nodes
                            .as_mut()
                            .and_then(|nodes| Some(nodes[handle.clone()].set_entry(entry)));
                        handle.clone()
                    }
                    None => {
                        // This node is not in the graph, lets add it to both the list of nodes
                        // and the nodes mapper
                        let new_handle = NodeHandle(mapper.len(), PhantomData);
                        mapper.insert(entry.key.clone(), new_handle.clone());
                        let new_node = Node::new(entry);
                        self.nodes
                            .as_mut()
                            .and_then(|nodes| Some(nodes.push(new_node)));
                        new_handle.clone()
                    }
                }
            }
        }
    }
}
```
#### Adding and Removing Edges 
```rust
/// WIP
```
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
