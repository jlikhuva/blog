##### Introduction
A running theme when designing algorithms and data structures is that if we know something about the structure of our data or inputs, we can often come up with bespoke procedures whose performance is better than the worst case lower bounds. A classic example is sorting. If we assume that we know nothig about the items that we are sorting, other than that they are pairwise comparable, we cannot sort faster that `O(n lg n)`. However, if we assume that our items are integer-like with an upper bound, we can sort in linear time using counting sort. Can we extend this reasoning to design a dictionary container that is tailor made for integers with worst case lookup better than `O(lg n)`? That is the subject of this note.

**Why Integers?** Well, they are perhaps the most structured inputs that computers deal with. Firstly, we can use them directly as indexes. Secondly, we can treat each as a string of bits. Doing so opens up all string data structures for use on integers. Furthermore, note that as strings, integers have a very small alphabet size (`sigma = 2`). Finally, practically all integers can fit in a single machine word (64 bits). This means that if we know that our integers are small, we can pack multiple of them in a single word and operate on them in parallel. For instance, if we're dealing with 8 bit integers, we can fit 8 in a single word (actually, only 7 as we'll see later on). 

Another reason why we may be interested in designing specialized containers for integers is that even when dealing with no-integer data, we can easily generate integer identifiers for such data (by hashing, for instance). Therefore, methods that speed up procedures for integers can be applied on on-integer data.

**A note on Computer Memory**

**Our Goal:**
##### X-Fast Tries
##### Y-Fast Tries
##### Word-Level Parallelism
##### Fusion Trees
##### Exponential Search Trees