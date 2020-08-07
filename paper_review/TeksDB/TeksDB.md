---
title: "TeksDB: Weaving Data Structures for a High-Performance Key-Value Store [SIGMETRIC '19]"
date: 2020-08-07
categories: PMKV
---

## Introduction

* KV Store need to consider highly-concurrent data store that simultaneously processes mixed operations
* The current memtable structure of KVS, however, **generally assumes a limited memory capacity that only serves as a temporary write buffer, and the use of single-threaded writes or unscalable data structures** for the memtable fails to fulfill this external demand.
* RocksDB memtable: list-based (skip list), hash-based (cuckoo hash), hybrid (hashed skip list, hashed linked list)
  * No one size fits all solution
  * Because there are **multi-dimensionally conflicting goals** (such as concurrency, scalability, fast insertion and retrieval, consistent scanning support, and efficient in-order traversals).
  * Furthermore, our experiments reveal **a performance anomaly when increasing the memory size.**

## Background

* Modern KVS consists of three key components: **a memtable, a persistent storage, and a log**
* **SkipList**
  * The SkipList supports a probabilistically binary search over sorted data, **allowing O(logN) insertion and retrieval times**
  * **Append-only manner:** new data with the same key are never overwritten.
    * Maintaining all versions of the data makes it easy to support **snapshot isolation** through MVCC
    * with append-only insertions, **multiple writers can concurrently manipulate the data structure in a thread-safe manner** through atomic operations (CAS).
    * 
* **CukooHash**
  * reduces the chance of collisions by **using multiple hash functions**
  * Collision: **the newly inserted entry pushes the older one to a different location** using a different hash function in the table, and this pushing of older entry may happen recursively
  * If too many items need to be moved (over one hundred in RocksDB), the new item is maintained by another backup list.
  * CuckooHash allows fast O(1) point updates and lookups.
  * It poorly services ranged retrievals because it does not maintain data sorted, and **thus it should sort entire dataset upon a ranged query**, which takes **O(NlogN)**
  * CuckooHash updates **data in-place** with no MVCC support
  * The writes are also made by a single thread, **using a write queue that serializes concurrent accesses**
* **HashSkipList**
  * the bucket is indexed by the hash ofthe key’s prefix
  * Inside each bucket, entries are organized using SkipList 
  * data are only partially sorted ==> O(log(N/B))
* **HashLinkedList**
  * within each bucket, entries are at first maintained in a linked list, but are reorganized into the SkipList once its size becomes too big.

<img src="img/Table 1.PNG" width="700" height=""/>



## Quantitative Analysis

<img src="img/Figure 1.PNG" width="" height=""/>

* Please refer papaer...

## TeksDB

<img src="img/Figure 2.PNG" width="250" height=""/>

* The key-value pairs are maintained in a shared data pool and the data structures in the CuckooHash and the SkipList only maintain pointers to the data pool.
* Challenges
  * **Blocking updates of the CuckooHash**
    * CuckooHash serializes writes with a single-threaded work queue 
      => Design optimistically concurrent version 
  * **Failing to provide point-in-time consistent views.**
    * CuckooHash overwrites data even while it is being read, it causes half-written or timestamp-inconsistent data to be read
      => Support MVCC 
  * **List size inflation due to duplicate keys**
    * selectively allows in-place updates to eliminate the number of duplicate keys
  * **Threats and opportunities for combining two data structures.**
    * We can accelerate the seek operation for locating the starting key in the SkipList by using a shortcut pointer from the CuckooHash.
    * this imposes a data synchronization overhead. 
    * by deferring the update of the SkipList until ranged retrieval is needed, but processing it fast through multi-threaded updates

### Optimistically Concurrent CuckooHash

* The collision handling routine is made up a series of atomic compare-and-swap (CAS) instructions and implements a careful **check-before-update logic**.

* it checks and ensures that a concurrent update in the middle of the path is not lost

* > Study to CukooHash BFS strategy...

### Point-In-Time Consistency Support

* > https://en.wikipedia.org/wiki/Data_consistency

  * RocksDB use a group-based update protocol

* 

### Selective In-Place Updates in SkipList

* <img src="img/Figure 3.PNG" width="" height=""/>

* SkipList: append-only designed for snapshot isolation => inflates the data structures

* by selectively allowing in-place updates. 

  * 1) when the multi-version maintenance is essential for snapshot isolation, i.e., under the concurrent reads and writes, the SkipList performs the native append-only updates.
  * 2) However, when the multi-versions are not necessarily needed, i.e., when only writers exist, the SkipList performs in-place updates, eliminating duplicates

* > Figure 3 explanation: refer the paper.. 

* Data consistency of the read operation:
  * The readers arriving when SkipList runs in an in-place mode wait until all updates in the active group are completed.
  * Once they all are completed, TeksDB switches the SkipList into the append-only mode, allowing readers to access it with the last committed timestamp.

### Bridging Structures for Fast Seek

* CuckooHash serves lookups and retrievals with O(1), while the SkipList supports sorted data scans immediately, without reorganizing data on-demand.
* We reduce this seek latency **by introducing a shortcut pointer in CuckooHash, which points to the node with a given key in SkipList**.
  * The shortcut pointer allows to locate the starting node for the range_query in SkipList with O(1) time complexity.

### Weaving the Two Data Structures Together

* TeksDB resolves this challenge by **deferring the update of the SkipList**, while **synchronously updating the CuckooHash**.
* Upon a put request,
  * inserts a new data into the CuckooHash in O(1) time and puts the request to a work queue that is serviced by a background worker thread.
  * This asynchronous update protocol decouples the backend SkipList and the frontend CuckooHash, hiding the double update cost from the users
* To deal with single wait queue => adopt the concurrent queue implemented in a lock-free manner by internally making use of pre-allocated contiguous blocks instead of linked lists
* For frequent sorted data retrieval operations,
  * TeksDB uses a single background worker to manage the cost of maintaining a large thread pool, but it receives help from user threads in processing concurrent operations.
    * More specifically, foreground threads for the range_query or scan are blocked and queued in a wait queue while the SkipList becomes up-to-date. 
    * If needed, TeksDB wakes these foreground threads and have them execute the update routine on the SkipList, thereby allowing concurrent updates without managing multiple background worker threads

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/