#### Introduction
A splay tree is an adaptive, amortized, self balancing tree data structure. What does this mean? Adaptive refers to the fact that the data structure is able to change its conformation in order to best serve the current data access patterns. Amortized refers to the fact that some operations on the tree take much longer than logarithmic time. However, on aggregate, any sequence of `n` operations takes `O(n lg n)` meaning that each operation takes, on average, logarithmic time. Self balancing refers to the fact that we do not store any auxilliary data in the nodes to use when balancing the tree -- the way, for instance, Red-Black-Trees do.

In this note, we will mostly focus on how to implement a bottom up splay tree in Rust. I do have a different note that explains why we'd want to have such a tree in the first place. You can find it on [this notion page](https://www.notion.so/Splay-Trees-3942f6942b7f4b06b5f666912f26a33a).

#### Rotations
A splay tree achieves balance by moving the last node that was `touched` during each operation to the root of the tree. To move the node, it uses a series of carefully chosen rotations. To deeply understand the mechanichs of a splay tree, therefore, we need to first understand what a rotation is and how it works. In this section, we explore explain what rotations are by describing how they reconfigure the tree. We also provide a working implementation.

A rotation about a subtree root `x` is a local structural change whereby we exchange `x` with one of its children. If the child we exchange `x` with is a left child, we call that a right rotation (names so because `x` become a right child -- `x` is rotated to the right, if you will). Similary, if the child we exchange `x` with is a right child, we call that a left rotation. Let's consider each of these cases separately.
1. ###### Right Rotations
   We are given a node `x` whose left child `y` exists (otherwise we can't right rotate).  `x` also has a right child `XR` (which may be `Nil`). `y` too has optional children -- `YL` to theleft and `YR` to the right. Right off the bat, we know a few key facts which can be summarized as follows: `YL < y < YR < x < XR`
2. ###### Left Rotations
#### Zig, ZigZig, & ZigZag
#### The Splay Operation
#### Splitting & Merging Trees
#### The Full API
#### Testing the API
