---
title: "MatrixKV: Reducing Write Stalls and Write Amplification in LSM-tree Based KV Stores with a Matrix Container in NVM"
date: 2020-08-08
categories: PMKV
---

## Introduction

* LSM-tree's target workload
  * OLTP workloads: (bursty) random writes
* Traditional LSM KV storage: (Fast Volatile) DRAM-(Slow Persistent) SSD
  * Limitations(cell size, power consumption...) prevent system performance from being further improved via increasing DRAM size.
  * Promising to use NVM
* LSM-tree  performance problem
  - **Write stalls** mainly stem from the significantly large amount of data involved in each compaction between L0-L1.
    - Involves almost all data in both levels due to the **unsorted L0**
    - The all-to-all compaction takes up CPU cycles and SSD bandwidth.
  - **Write amplification** increases with the depth of LSM-trees.
* Novel techniques
  - We **relocate and manage the L0 level in NVM with our proposed matrix container**
  - the new column compaction is devised to **compact L0 to L1 at fine-grained key ranges**
  - MatrixKV **increases the width of each level to decrease the depth of LSM-trees** thus mitigating write amplification
  - the cross-row hint search is introduced for the matrix container to keep adequate read performance

## Background and Motivation

### Non-volatile Memory

* In particular, systems with DRAM-NVM-SSD storage are recognized as a promising way to utilize NVMs due to the following three reasons.
  1. NVM is expected to **co-exist with large-capacity SSDs** for the next few years. 
  2. Compared to DRAM, **NVM still has 5 times lower bandwidth and 3 times higher read latency**. 
  3. A hybrid system **balances the TCO and system performance**. 

### Log-structured Merge Trees

<img src="img/Figure 1.PNG" width="500" height=""/>

* LSM Design principal: Defer and batch write requests in memory to exploit the high sequential write bandwidth of storage devices.
* To serve write requests, 
  * writes are first batched in DRAM by two skip-lists (**MemTable** and **Immutable MemTable**).
  * Then, the **immutable MemTable is flushed to L0 on SSDs** generating Sorted String Tables (SSTables)
    * **To deliver a fast flush**, **L0 is unsorted** where key ranges overlap among different SSTables.
    * **Compaction makes each level sorted** (except L0) thus bounding the overhead of reads and scans
* To conduct a compaction,
  1. An SSTable in Li (called a victim SSTable) and multiple SSTables in Li+1 who has overlapping key ranges (called overlapped SSTables) are picked as the compaction data. 
  2. Other SSTables in Li that fall in this compaction key ranges are selected reversely. 
  3. Those SSTables identified in steps (1) and (2) are fetched into memory, to be merged and sorted. 
  4. The regenerated SSTables are written back to Li+1.
* To serve read requests,
  * RocksDB searches the **MemTable first**, **immutable MemTable next**, **and then SSTables in L0 through Ln in order.**
    * Since SSTables in L0 contain overlapping keys, a lookup may search multiple files at L0

### LSM-tree based KV stores

* Reducing WA
  * PebblesDB, Lwc-tree, Wisckey, LSM-trie, TRIAD ...
* Reducing write stalls
  * SILK, Blsm, KVell ...
* Improving LSM-trees with NVMs
  * SLM-DB, MyNVM, NVMRocks, NoveLSM ...

### Challenges and Motivations

#### Write Stalls

1. Insert stalls: if **MemTable fills up before the completion of background flushes**, all insert operations to LSM-trees are stalled.

2. Flush stalls: if **L0 has too many SSTables** and reaches a size limit, flushes to storage are blocked.
3. Compactions stalls: **too many pending compaction bytes** block foreground operations

<img src="img/Figure 2.PNG" width="350" height=""/> <img src="img/Figure 3.PNG" width="350" height=""/>

* In Figure2, the period of **L0-L1 compaction approximately matches write stalls observed**
* In Figure 3, write stalls induce **high write latency leading to the long-tail latency problem**.

#### Write Amplification

* Since the sizes of adjacent levels from low to high increase exponentially by an amplification factor (**AF = 10**), compacting an SSTable from Li to Li+1 results in a WA factor of AF on average.
  * The increased WA consumes more storage bandwidth, competes with flush operations, and ultimately slows down application throughput. 

#### NoveLSM

<img src="img/Figure 4.PNG" width="400" height=""/>

1. adopting NVMs as an alternative DRAM to increase the size of MemTable and immutable MemTable
2. making the NVM MemTable mutable to allow direct updates thus reducing compactions.

* However, these design choices **merely postpone the write stalls**.
* When the **dataset size exceeds the capacity of NVM MemTables**, flush stalls still happen, blocking foreground requests. 
  * Furthermore, the enlarged MemTables in NVM are flushed to L0 and dramatically increase the amount of data in L0-L1 compactions, resulting in even more severe write stalls. 
* In Figure 4, NoveLSM reduces the overall loading time by 1.7× compared to RocksDB
  * However, **the period of write stalls is significantly longer**

## MatrixKV Design

### Matrix Container

* <img src="img/Figure 6.PNG" width="400" height=""/>
* Elevate L0 from SSDs to NVMs to **manage the L0 of LSM-trees**
  * Exploit the byte-addressablility and fast random accesses of NVMs
* **Receiver**
  * accepts and retains MemTables flushed from DRAM
  * is serialized as a single row of the receiver and organized as a RowTable *(SSTable 만드는 비용과 같다고 한다, 메타데이터 구성만 다르기 때문)*
  * When the receiver size reaches its size limit (e.g., 60% of the matrix container) and the compactor is empty, the receiver stops receiving flushed MemTables and dynamically turns into the compactor.
  * In the meantime, a new receiver is created for receiving flushed MemTables. There is no data migration for the logical role change of the receiver to the compactor. *(논리적인 상태변화만.)*

<img src="img/Figure 7.PNG" width="500" height=""/>

* **RowTable**
  * serialize KV items from the immutable MemTable in the order of keys and store them to the data region
  * build metadata for all KV items with a sorted array *(array인데 다른 종류의 value가 들어가 있다..? bit 값으로 value 경계를 구분하는건지?)*.
  * To locate KV in a RowTable, **binary search the sorted array to get the target key** and find its value with the page number and the offset.
  * The forward pointer(e.g. P0) in each array element is used for **cross-row hint** searches that contribute to **improving the read efficiency** within the matrix container.
  * SSTable: block unit, RowTable: NVM page unit
* **Compactor**
  * The compactor is used for **selecting and merging data from L0 to L1 in SSDs** at a fine granularity
  * merge a specific key range from L0 with a subset of SSTables at L1 without needing to merge all of L0 and all of L1
  * KV items from different RowTables that fall in the key range of a column compaction logically constitute a column. 
  * *(compactor 약간 이해 부족)*
* **Space management**
  * After compacting a column, the NVM space occupied by the column is freed.
    * To manage those freed spaces, we simply apply the paging algorithm.
  * The NVM pages fully freed after column compactions are added to the free list as a group of page-sized units.
    * To store incoming RowTables in the receiver, we apply free pages from the free list.
* while columns are being compacted in the compactor, the receiver can continue accepting flushed MemTables from DRAM simultaneously. 
* By freeing the NVM space one column at a time, MatrixKV ends the write stalls forced by merging the entire L0 with all of L1.

### Column Compaction

<img src="img/Figure 8.PNG" width="500" height=""/>

1. MatrixKV separates the key space of L1 into multi- ple contiguous key ranges
2. Column compaction starts from the first key range in L1.
3. In the compactor, victim KV items within the compaction key range are picked concurrently in multiple rows.
4. If the amount of data within this key range is under the lower bound of compaction, the next key range in L1 joins.
5. Then a column in the compactor is logically formed.
6. Data in the column are merged and sorted with the overlapped SSTables of L1 in memory
7. Finally, the regenerated SSTables are written back to L1 on SSDs.

* For example explanation, please refer the paper...

### Reducing LSM-tree Depth

* LSM-tree: the overall WA increases with the number of levels (n), i.e., WA=n*AF 
* Flattening conventional LSM-trees with wider levels brings two negative effects
  * Enlarget L0 has more SSTables => Impact on L0-L1 compaction overhead
    * **MatrixKV: the fine-grained column compaction**
  * Travesing the larger unsorted L0 decreases the search efficiency.
    * **MatrixKV proposes the cross-row hint search**

### Cross-row Hint Search

<img src="img/Figure 9.PNG" width="500" height=""/>

* **Constructing cross-row hints:**
  * When we build a RowTable for the receiver of the matrix container, **we add a forward pointer for each element in the sorted array of metadata**.
    * RowTable i의 키 x에 대해 정방향 포인터는 앞의 RowTable i-1에서 키 y를 인덱싱합니다. 여기서 키 y는 x보다 작지 않은 첫 번째 키입니다 (즉, y ≥x).
  * These forward pointers provide **hints to logically sort all keys in different rows**, similar to the fractional cascading
  * Since each forward pointer **only records the array index of the preceding RowTable**, **the size of a forward pointer is only 4 bytes.** Thus, the storage overhead is very small.
* **Search process in the matrix container:**
  * A search process starts from the latest arrived RowTable i. 
    * If the key range of RowTable i does not overlap the target key, we skip to its preceding RowTable i−1. 
    * Else, we binary search RowTable i to find the key range (i.e., bounded by two adjacent keys) where the target key resides. (인접한 min,max narrowed key 사이에서 바이너리서치)
    * With the forward pointers, we can narrow the search region in prior RowTables, i−1, i−2, . . . continually until the key is found. 
    * As a result, there is no need to traverse all tables entirely to get a key or scan a key range.

### Implementation

* MatrixKV is implemented based on RocksDB (The LOC on top of RocksDB is 4117 lines)
* MatrixKV accesses NVMs via the PMDK library and accesses SSDs via the POSIX API
* Write:
  1. Write requests from users are inserted into a write-ahead log on NVMs to prevent data loss from system failures.
  2. Data are batched in DRAM, forming MemTable and immutable MemTable.
  3. The immutable MemTable is flushed to NVM and stored as a RowTable in the receiver of the matrix container.
  4. The receiver turns into the compactor logically if the number of RowTables reaches a size limit (e.g., 60% of the matrix container) and the compactor is empty. This role change has no real data migrations.
  5. Data in the compactor is column compacted with SSTables in L1 column by column. In the meantime, a new receiver receives flushed MemTables.
  6. In SSDs, SSTables are merged to higher levels via conventional compactions as RocksDB does.
* Read:
  * The read thread searches with the priority of DRAM>NVMs>SSDs.
  * In NVMs, the cross-row hint search contributes to faster searches among different RowTables of L0. 
* Consistency:
  * writes/updates for NVM only happen in two processes, flush and column compaction
  * Flush immutable MemTables
    * If a failure occurs in the middle of writing a RowTable, MatrixKV can re-process all the transactions that were recorded in the write-ahead log
  *  Column compaction
    * MatrixKV adopts the versioning mechanism of RocksDB
      * If the system crashes during compaction, the database goes back to its last consistent state with versioning. 



[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/