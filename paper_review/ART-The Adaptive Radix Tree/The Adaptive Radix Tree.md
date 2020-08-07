---
title: "The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases [ICDE '13]"
date: 2020-08-07
categories: PMKV
---

## Adaptive Radix Tree

<img src="img/Figure 1.PNG" width="370" height=""/> <img src="img/Figure 2.PNG" width="400" height=""/>

### Radix tree

* Interesting properties
  * The **height (and complexity) of radix trees depends on the length of the keys** but in general not on the number of elements in the tree.
  * Radix trees require **no rebalancing operations** and all insertion orders result in the same tree.
  * **The keys are stored in lexicographic order**
  * The path to a **leaf node represents the key of that leaf**. Therefore, keys are stored implicitly and can be reconstructed from paths.
* Inner node: map parital keys to other nodes, leaf node: store the values corresponding to keys
* s (span): How many bits of the partial key the node has,
  * 32 bit keys, s = 8 => height [32/8] = 4 levels



### Adaptive Nodes

<img src="img/Figure 3.PNG" width="450" height=""/>

<img src="img/Figure 4.PNG" width="" height=""/>

* Figure 3 shows the height and space consumption for different values of the span parameter when storing 1M uniformly distributed 32 bit integers.
  * **Space usage can be excessive when most child pointers are null** 
* Figure 4: The key idea that achieves both space and time efficiency is **to adaptively use different node sizes with the same, relatively large span, but with different fanout**.

### Structure of Inner Nodes

<img src="img/Figure 5.PNG" width="500" height=""/>

```C
union Node {
    Node4* n4;
    Node16* n16;
    Node48* n48;
    Node256* n256;
}
```

* Node 4, 16

  ```C
  struct Node4 {
      char child_keys[4];
      Node* child_pointers[4];
  }
  
  Node* find_child(char c, Node4* node) {
      Node* ret = NULL;
      for (int i = 0; i < 4; ++i) {
          if (child_keys[i] == c) ret = node->child_pointers[i];
      }
  
      return ret;
  }
  ```

  * ART stores **all the keys in a list**, and the **child pointers in a parallel list**
  * Looking up the next character in a string means searching the list of child keys, and then using the index to look up the corresponding pointer.
  * Node 16: Since there are only 16 of them, it’s also possible to search all the keys in parallel using SIMD.

* Node 48

  ```c
  struct Node48 {
  // Indexed by the key value, i.e. the child pointer for 'f'
  // is at child_ptrs[child_ptr_indexes['f']]
  char child_ptr_indexes[256];
  
  Node* child_ptrs[48];
  char num_children;
  }
  
  Node* find_child(char c, Node48* node) {
  int idx = node->child_ptr_indexes[c];
  if (idx == -1) return NULL;
  
  return node->child_ptrs[idx];
  }
  ```
  * Searching for the key can become expensive, so instead the keys are stored implicitly in an array of 256 indexes. **The entries in that array index a separate array of up to 48 pointers.**
  * The idea here is that this is superior to just storing an array of 256 `Node` pointers because you can store 48 children in 640 bytes (where 256 pointers would take 2k).

* Node 256

  ```c
  struct Node256 {
      Node* child_ptrs[256];
  }
  
  Node* find_child(char c, Node256* node) {
      return child_ptrs[c];
  }
  ```

  * Looking up child pointers is obviously very efficient - the most efficient of all the node types

### Structure of Leaf Nodes

* all values are stored in one of three way
  * **Single value** nodes are leaf nodes which store… exactly one value. They’d be represented in our scheme as a different `Node` type.
  * **Multi-value** leaves are just like the regular `Node[4|16|48|256]` types, but the `child_ptrs` now become an array of `Value*`.
  * **Pointer/value slots** store values *directly* in the slots otherwise used for child pointers, and distinguishes between them based on the highest bit - let’s say 0 to interpret the pointer as a child node pointer, and 1 to interpret it as a pointer to a value. Using these high bits doesn’t lose us anything on a modern CPU whose addressable memory is ‘only’ 2^48 bytes - the extra 16 bits in a 64-bit pointer can be used to store extra information.
* but what about keys that are prefixes of some other key? What about `http://google.com` and `http://google.com/chrome`?
  * **Fixed-length datatypes** such as 128-bit integers, or strings of exactly 64-bytes, don’t have any problem because there can, by construction, never be any key that’s a prefix of any other.
  * **Variable-length datatypes** such as general strings, can be transformed into types where no key is the prefix of any other by a simple trick: *append the NULL byte to every key*. The NULL byte, as it does in C-style strings, indicates that this is the end of the key, and no characters can come after it. Therefore no string with a null-byte can be a prefix of any other, because no string can have any characters after the NULL byte!

### Collapsing Inner Nodes

<img src="img/Figure 6.PNG" width="400" height=""/>

* ##### Lazy exapnsion

  * Inner nodes are only created if they are required to distingish at least two leaf nodes.
  * Implementing lazy expansion really just means having a separate `Node` type that has a key pointer and a value pointer. 
  * If this node type is encountered during a query, you can just compare the key to the searched-for key character-by-character. 
  * If this node type is found during an insertion, it has to be swapped out for a ‘real’ node, and each suffix of the existing key and the new one needs to be inserted

* **Path compression**

  * Removes all inner nodes that have only a singel child
  * That is, consider inserting `CAST` and `CASH` into an empty trie; path compression would create a node at the root that has the `CAS` prefix, and two children, one for `T` and one for `H`. That way the trie doesn’t need individual nodes for the `C->A->S` sequence.
  * We use a hybrid approach by storing a vector at each node like in the pessimistic approach, but with a constant size (8 bytes) for all nodes.
  * *Pessimistic vs Optimistic 내용 이해 X..*

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/