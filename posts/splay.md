# The Bottom-up Splay Tree

## Introduction

A splay tree is an adaptive, amortized, self balancing tree data structure. What does this mean? Adaptive refers to the fact that the data structure is able to change its conformation in order to best serve the current data access patterns. Amortized refers to the fact that some *individual* operations on the tree may take much longer than logarithmic time. However, on aggregate, any sequence of `n` operations takes `O(n lg n)` meaning that each operation takes, on average, logarithmic time. Self balancing refers to the fact that we do not store any auxillary data in the nodes to use when balancing the tree — the way, for instance, red-black-trees do.

In this note, we will mostly focus on how to implement a bottom up splay tree in Rust. I do have a different note that explains why we'd want to have such a tree in the first place. You can find it on [this notion page](https://www.notion.so/Splay-Trees-3942f6942b7f4b06b5f666912f26a33a).

## The Splay Tree Abstraction

We will be using indexes to implement the tree structure. That is, we will store the nodes of our tree in a contiguous storage container and use indexes to that container to move around the tree? Why this design choice? Well, as explained [here](http://smallcultfollowing.com/babysteps/blog/2015/04/06/modeling-graphs-in-rust-using-vector-indices/), "... indices are often a compact and convenient way to represent complex data structures". The downside of using raw indexes (`usize`) is that doing so introduces opacity because an index tells us nothing about the data it is referring to. Furthermore, a raw index can be easily misused especially when we have multiple indexes to different containers flying around. The good news is that we can easily mitigate these two problems by using the `Newtype Index Pattern` or as I call it, the `Wrapped Index Pattern`. We wrap our raw index in a concrete type and implement the `Index` trait for this new type. Below we implement the core abstractions that we will be using.

```rust
/// We use the new type index pattern to make our code more
/// understandable. Using indexes to simulate pointers
/// can lead to opaqueness. Using a concrete type instead of
/// raw indexes ameliorates this. The 'insane' derive ensures
/// that the new type has all the crucial properties of the
/// underlying index
#[derive(Debug, Copy, Clone, Ord, PartialOrd, Eq, PartialEq, Hash)]
struct SplayNodeIdx(usize);

impl<K: Ord, V> std::ops::Index<SplayNodeIdx> for Vec<SplayNode<K, V>> {
    type Output = SplayNode<K, V>;

    /// This allows us to use SplayNodeIdx directly as an index
    fn index(&self, index: SplayNodeIdx) -> &Self::Output {
        &self[index.0]
    }
}

impl<K: Ord, V> std::ops::IndexMut<SplayNodeIdx> for Vec<SplayNode<K, V>> {
    fn index_mut(&mut self, index: SplayNodeIdx) -> &mut Self::Output {
        &mut self[index.0]
    }
}

/// A single entry in the tree
#[derive(Debug)]
pub struct Entry<K, V> {
    key: K,
    value: V,
}

impl<K: Ord, V> From<(K, V)> for Entry<K, V> {
    /// Allows us to quickly construct an entry from a key-value pair
    fn from(e: (K, V)) -> Self {
        Entry {
            key: e.0,
            value: e.1,
        }
    }
}

/// A single node in the tree. This is the main unit of
/// computation in the tree. That is, all operations
/// operate on nodes. It is parameterized by a key which
/// should be orderable and an arbitrary value
#[derive(Debug)]
struct SplayNode<K: Ord, V> {
    entry: Entry<K, V>,
    left: Option<SplayNodeIdx>,
    right: Option<SplayNodeIdx>,
    parent: Option<SplayNodeIdx>,
}

impl<K: Ord, V> SplayNode<K, V> {
    /// Create a new splay tree node with the given entry
    pub fn new(entry: Entry<K, V>) -> Self {
        SplayNode {
            entry,
            left: None,
            right: None,
            parent: None,
        }
    }

    /// Retrieve the key in this node
    pub fn key(&self) -> &K {
        &self.entry.key
    }
}
/// A type alias for ergonomic reasons
type Nodes<K, V> = Vec<SplayNode<K, V>>;

/// A splay tree implemented using indexes
#[derive(Debug, Default)]
pub struct SplayTree<K: Ord, V> {
    /// A growable container of all the nodes in the tree
    elements: Option<Nodes<K, V>>,

    // The location of the root. Optional because we first create
    // an empty tree. We keep track of it because its location
    // can change as we make structural changes to the tree
    root: Option<SplayNodeIdx>,
}
```

## Rotations

A splay tree achieves balance by moving the last node that was `touched` during each operation to the root of the tree. To move the node, it uses a series of carefully chosen rotations. To deeply understand the mechanichs of a splay tree, therefore, we need to first understand what a rotation is and how it works. In this section, we explore explain what rotations are by describing how they reconfigure the tree. We also provide a working implementation.

A rotation about a subtree root `x` is a local structural change whereby we exchange `x` with one of its children. If the child we exchange `x` with is a left child, we call that a right rotation (names so because `x` become a right child -- `x` is rotated to the right, if you will). Similary, if the child we exchange `x` with is a right child, we call that a left rotation. Let's consider each of these cases separately.

## Left & Right Rotations

  In a `Left Rotation`, we are given the root `x` of some subtree, we would like to switch this root with its right child `y`. Let `α` be the left child of `x`. Further, let `Ω` be the right child of `y` and `χ` be the left child of `y`. Right off the bat, we know that:

- Both `χ` and `Ω` -- the children of `y` are larger than `x`
- `α` is the only value smaller than `x`.
- `χ` is the only value smaller than `y`
- `Ω` is larger than both `x` and `y`
  
Since the task is to swap `x` and `y`, we can use the observations above to maintain
the search tree invariant using the following steps:

- Swap `x` and `y`.
- This breaks the bst invariant because `x` is smaller than `y` yet its to the right of `y`. To remedy this, make `x` the left child of `y`.
- Make `Ω` the right child of `y`. It's the only value larger than `y`. No modifications are needed for this.
- Now all that remains is to handle `α` and `χ`. This is simple because only `α` is less than `x`, so it stays as `x`'s left child. Since `x` < `χ` < `Ω` it goes to the right of `x`.
- We also have to take care when `x` is the root. In that case, `y` becomes the new root
  
`Right rotations` are similar to left rotations, except that we wish to exchange a subtree root wth its left child. The rest of the steps proceed analogously. We implement these two core procedures below.

```rust
impl<K: Ord, V> SplayTree<K, V> {
    /// Indicates whether a given node is a left or right child of its
    /// parent
    #[derive(Debug)]
    pub enum ChildType {
        Left,
        Right,
    }

    /// Exchange `target` with its left child while preserving the search
    /// tree invariant
    fn rotate_right(nodes: &mut Nodes<K, V>, target: SplayNodeIdx) {
        todo!()
    }

    /// Exchange `target` with its right child while preserving the search
    /// tree invariant
    fn rotate_left(nodes: &mut Nodes<K, V>, target: SplayNodeIdx) {
        todo!()
    }

    /// Is this node a left child or a right child of its parent
    fn child_type(nodes: &Nodes<K, V>, cur: Option<SplayNodeIdx>) -> Option<ChildType> {
        cur.and_then(|cur_idx| {
            let parent = nodes[cur_idx].parent;
            parent.and_then(|parent_idx| match nodes[parent_idx].left.cmp(&cur) {
                Ordering::Equal => Some(ChildType::Left),
                _ => Some(ChildType::Right),
            })
        })
    }
}
```

How do we combine the two primitive operations discussed above  to create a splay tree? We discuss that in the next section.

## Zig, ZigZig, & ZigZag

As mentioned earlier, the key property of a splay tree is that it moves the last accessed node to the root of the tree after each operation. Doing so gives the tree its key properties — [amortized balance, the working set property, the dynamic finger property, and the entropy property](https://www.notion.so/Splay-Trees-3942f6942b7f4b06b5f666912f26a33a). Instead of simply exchanging the last accessed node with the root node in one single operation, the spay tree does so using a series of local rotations. These rotations can be grouped into three classes based on the configuration of the nodes that we apply them to. When a node is a child of the root, we apply a left or right rotation to move it to the root. We call this the `Zig` operation. When a node is a left child of a left child, we apply two right rotations. When a node and its parent are both right children, we apply two left rotations to move the node up the tree. We group both of these operations under `ZigZig`. Finally, when a node is a right child of a left child or a left child of a right child, we call this a `ZigZag` configuration. To move the node up the tree, we either apply a left rotation followed by a right rotation, or a right rotation followed by a left rotation. For a much more involved discussion, please refer to [this note here and the literature it links to](https://www.notion.so/Splay-Trees-3942f6942b7f4b06b5f666912f26a33a).

Below, we implement these three operations.

```rust
/// The three configurations that dictate the number, order,
/// and nature of the rotations we perform during the splay operation
#[derive(Debug)]
pub enum NodeConfig {
    /// A node is in a Zig configuration if it is a
    /// child of the root.
    Zig(ChildType),

    /// A node is in a ZIgZig configuration if it is
    /// a left child of a left child or a right child of
    /// a right child
    ZigZig(ChildType),

    /// A node is in a ZigZag configuration if
    /// it is left child of a right child or
    /// a right child of a left child
    ZigZag(ChildType),
}

    /// The final splay operation to move the target node to the root of the tree
    fn zig(nodes: &mut Nodes<K, V>, target: SplayNodeIdx, child_type: ChildType) {
        match child_type {
            ChildType::Left => Self::rotate_right(nodes, target),
            ChildType::Right => Self::rotate_left(nodes, target),
        }
    }

    /// Applied when the target node is in a cis-configuration with its parent
    fn zig_zig(nodes: &mut Nodes<K, V>, target: SplayNodeIdx, child_type: ChildType) {
        let grand_parent = Self::get_grand_parent(nodes, target);
        match child_type {
            ChildType::Left => {
                Self::rotate_right(nodes, grand_parent);
                Self::rotate_right(nodes, grand_parent);
            }
            ChildType::Right => {
                Self::rotate_left(nodes, grand_parent);
                Self::rotate_left(nodes, grand_parent);
            }
        }
    }

    /// Applied when the target node is in a trans-configuration with  its parent
    fn zig_zag(nodes: &mut Nodes<K, V>, target: SplayNodeIdx, child_type: ChildType) {
        let parent = nodes[target].parent.unwrap();
        let grand_parent = Self::get_grand_parent(nodes, target);
        match child_type {
            ChildType::Right => {
                Self::rotate_left(nodes, parent);
                Self::rotate_right(nodes, grand_parent);
            }
            ChildType::Left => {
                Self::rotate_right(nodes, parent);
                Self::rotate_left(nodes, grand_parent);
            }
        }
    }

    fn get_grand_parent(nodes: &Nodes<K, V>, target: SplayNodeIdx) -> SplayNodeIdx {
        let parent = &nodes[target];
        debug_assert!(parent.parent.is_some());
        parent.parent.unwrap()
    }

    /// What is the local configuration of the nodes around the `target` node?
    /// In particular, what kind of children are `target` and its parent?
    fn get_cur_config(nodes: &Nodes<K, V>, target: SplayNodeIdx) -> NodeConfig {
        match Self::child_type(nodes, Some(target)) {
            Some(ChildType::Left) => match Self::child_type(nodes, nodes[target].parent) {
                None => NodeConfig::Zig(ChildType::Left),
                Some(ChildType::Left) => NodeConfig::ZigZig(ChildType::Left),
                Some(ChildType::Right) => NodeConfig::ZigZag(ChildType::Left),
            },
            Some(ChildType::Right) => match Self::child_type(nodes, nodes[target].parent) {
                None => NodeConfig::Zig(ChildType::Right),
                Some(ChildType::Left) => NodeConfig::ZigZag(ChildType::Right),
                Some(ChildType::Right) => NodeConfig::ZigZig(ChildType::Right),
            },
            None => panic!("This procedure should not be called on the root node"),
        }
    }
}
```

## The Splay Operation

With the three core operation in place, we can noe go ahead and discuss how the bottom up splay operation works. Put simply, whenever a `query` or `update` operation is made to the tree, we keep track of the last node that we access. We move it to the root by calling `splay(last_accessed_node)`. The splay operation simply calls `zigzag` and `zigzig` as needed until the target node is a child of the root. At that point, it calls `zig`. We implement this procedure below.

```rust
/// Implementation of the bottom up splay operation
impl<K: Ord, V> SplayTree<K, V> {
    /// Moves the target node to the root of the tree using a series of rotations.
    fn splay(&mut self, target: SplayNodeIdx) {
        match &mut self.elements {
            None => (),
            Some(nodes) => {
                while nodes[target].parent.is_some() {
                    match Self::get_cur_config(nodes, target) {
                        NodeConfig::Zig(typ) => Self::zig(nodes, target, typ),
                        NodeConfig::ZigZig(typ) => Self::zig_zig(nodes, target, typ),
                        NodeConfig::ZigZag(typ) => Self::zig_zag(nodes, target, typ),
                    }
                }
                self.root = Some(target);
            }
        }
    }
```

## Splitting & Merging Trees

Personally, the biggest effect of the splay operation is that it allows us to split and merge trees very easily. In particular, suppose we with to split a tree at a value `x`. To so so, we begin by finding the smallest value larger than `x`. That is, we find `s = succ(x)`. We then move `s` to the root of the tree by calling `splay(s)`. To split the tree, we simply remove the link that `s` has to its left child. This give us two trees. A left tree in which `x` is the largest value and a right tree in which `s` is the smallest value. We implement this scheme below.

```rust
/// WIP: Tree Splitting
```

To merge two trees, `LT` and `RT` we begin by splaying the largest value in `LT` to the root of its tree. Note that this value does not have a right child. We then make the root of `RT` to be the right child of the root of `LT`. We implement this procedure below.

```rust
/// WIP: Tree Merging
```

## The Full API

Thus far, we have discussed the key components of the splay tree in isolation. In particular, we've discussed and implemented the core splay operation. But, we have yet to discuss how the operations fit into the dictionary API that our search tree needs to ultimately expose. We do so in this section.  We begin by implementing the read operations. Note that unlike in other binary search APIs where read operations do no lead to structural changes, such operations in a splay tree, obviously, do lead to changes in the tree structure. We then provide the implementation of the two main write operations -- `insert` and `delete`.
**Read Operations**

```rust
/// Implementation of Read operations.
impl<K: Ord + Default, V: Default> SplayTree<K, V> {
    /// Create a new, empty red black tree
    ///
    /// # Example
    /// ```
    /// let tree: SplayTree<&str, usize> = SplayTree::new();
    /// ```
    pub fn new() -> Self {
        SplayTree::default()
    }

    /// Retrieves the Key-Value pair associated with
    /// the provided key id it exists in the tree.
    ///
    /// # Examples
    /// ```
    /// use crate::splay_tree::SplayTree;
    /// let mut tree = SplayTree::new();
    /// assert!(tree.get("cat").is_none())
    ///
    /// tree.insert(("cat", "tabby").into());
    /// assert!(tree.get("cat").is_some());
    /// assert_eq!(tree.get("cat").unwrap(), ("cat", "tabby").into());
    /// ```
    pub fn get(&mut self, k: K) -> Option<&Entry<K, V>> {
        match &mut self.elements {
            None => None,
            Some(nodes) => Self::get_helper(nodes, self.root, k)
                .and_then(move |key_index| Some(&nodes[key_index].entry)),
        }
    }

    /// Retrieves the Key-Value pair associated with
    /// the largest key smaller than the provides key
    /// if it exists. Such a value does not exist if the given
    /// key is the smallest element in the tree
    ///
    /// # Examples
    /// ```
    ///
    /// ```
    pub fn pred(&mut self, k: K) -> Option<&Entry<K, V>> {
        match &mut self.elements {
            None => None,
            Some(nodes) => Self::get_helper(nodes, self.root, k).and_then(move |key_idx| {
                let right = nodes[key_idx].right;
                let pred = match right {
                    None => Self::lra(nodes, Some(key_idx)),
                    Some(right_idx) => Self::max_helper(nodes, Some(right_idx)),
                };
                pred.and_then(move |pred_idx| Some(&nodes[pred_idx].entry))
            }),
        }
    }

    /// Retrieves the Key-Value pair associated with
    /// the smallest key larger than the provides key
    /// if it exists. Such a value does not exist if the given
    /// key is the largest element in the tree
    ///
    /// # Examples
    /// ```
    ///
    /// ```
    pub fn succ(&mut self, k: K) -> Option<&Entry<K, V>> {
        match &mut self.elements {
            None => None,
            Some(nodes) => Self::get_helper(nodes, self.root, k).and_then(move |key_idx| {
                let right = nodes[key_idx].right;
                let succ = match right {
                    None => Self::lla(nodes, Some(key_idx)),
                    Some(right_idx) => Self::min_helper(nodes, Some(right_idx)),
                };
                succ.and_then(move |succ_idx| Some(&nodes[succ_idx].entry))
            }),
        }
    }

    /// Retrieves the Key-Value pair associated with
    /// the largest key in the tree
    ///
    /// # Examples
    /// ```
    ///
    /// ```
    pub fn max(&mut self) -> Option<&Entry<K, V>> {
        match &mut self.elements {
            None => None,
            Some(nodes) => Self::max_helper(nodes, self.root)
                .and_then(move |max_idx| Some(&nodes[max_idx].entry)),
        }
    }

    /// Retrieves the Key-Value pair associated with
    /// the smallest key in the tree
    ///
    /// # Examples
    /// ```
    ///
    /// ```
    pub fn min(&mut self) -> Option<&Entry<K, V>> {
        match &mut self.elements {
            None => None,
            Some(nodes) => Self::min_helper(nodes, self.root)
                .and_then(move |min_idx| Some(&nodes[min_idx].entry)),
        }
    }
}
```

## Write Operations

```rust
/// Implementation of Write operations. These procedures
/// do lead to structural changes in the tree
impl<K: Ord, V> SplayTree<K, V> {
    /// Adds a new entry into the red black tree in amortized O(lg n) time.
    ///
    /// # Examples
    /// ```
    ///
    /// ```
    pub fn insert(&mut self, e: Entry<K, V>) -> Option<&Entry<K, V>> {
        todo!()
    }

    /// Removes the entry associated with the provided key
    /// from the splay tree in amortized O(lg n) time.
    ///
    /// # Examples
    /// ```
    ///
    /// ```
    pub fn delete(&mut self, k: K) -> Option<Entry<K, V>> {
        todo!()
    }
}
```

<!-- #### Testing the API
```rust
/// WIP
``` -->
