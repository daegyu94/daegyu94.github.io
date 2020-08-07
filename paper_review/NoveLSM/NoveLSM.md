---
title: "Redesigning LSMs for Nonvolatile Memory with NoveLSM Sudarsun [ATC '18]"
date: 2020-08-07
categories: PMKV
---

## Background

### Log Structured Merge Tree

<img src="img/Figure 1.PNG" width="350" height=""/>

### LevelDB

* LSMs achieve higher write throughput by **first staging data in memory and then across multiple levels on disk to avoid random disk writes.**
* During an insert operation, LevelDB buffers updates in a memory-based skip list table (referred to as memtable hereafter) and stores data on multiple levels of on-disk block structures know as sorted string tables (SSTable).
* With an exception to memtables, **all lower levels are mutually exclusive and do not maintain redundant data**

### RocksDB

* RocksDB achieves this by **supporting multi- threaded background compaction** which can simultaneously compact data across multiple levels of LSM hierarchy and extracts parallelism with multi-channel SSDs
* RocksDB that can be **beneficial for NVM is the use of a Cuckoo hashing-based SST table format optimized for random lookup** instead of the traditional I/O block-based format with high random-access overheads.

## Motivation

<img src="img/Figure 2,3,4.PNG" width="" height=""/>

* As shown in Figure 2, the results show that **current LSMs do not fully exploit the hardware benefits of NVM and suffer from significant software overheads**. 
* Figure 3 shows the cost breakup for insert operations with 4 KB, 8 KB, and 16 KB values.
  * **data compaction dominates the cost**
* Increasing the in-memory buffer (memtable) can reduce compaction frequency, but has drawbacks
  * DRAM usage increases by two times: memory must be increased for both mutable and immutable memtables. 
  * only after the immutable memtable is compacted, log updates can be cleaned, leading to a larger log size
  * LSMs do not enforce commits (sync) when writing to a log => an application crash or power-failure could lead to data loss
* Figure 4, For small values, searching the SSTable dominates the cost, 
  * **(e.g., 16 KB) the deserialization cost increases with increasing value size .**

## Design

#### Principle 1: Exploit byte-addressability to reduce se- rialization and deserialization cost.

* NoveLSM provides a **persistent NVM memtable** by designing a persistent skip list
* The **DRAM memtable data can be directly moved to the NVM memtable** without requiring serialization or deserialization

#### Principle 2: Enable mutability of persistent state and leverage large capacity of NVM to reduce compaction cost.

* NoveLSM designs **a large mutable persistent memtable** to which applications can directly add or update new key-value pairs.
* To exploit mutability of persistent state, The persistent memtable allows **NoveLSM to alternate between small DRAM and large NVM memtable without stalling for background compaction to complete**.

#### Principle 3: Reduce logging overheads and recovery cost with in-place durability

* NoveLSM avoids logging by **immediately committing updates to the persistent memtable in-place**.

#### Principle 4: Exploit the low latency and high band- width of NVM to parallelize data read operations.

* NoveLSM exploits NVMs’ low latency and high bandwidth **by parallelizing search across the memory and storage levels**.

<img src="img/Figure 5.PNG" width="" height=""/>

### Addressing (De)serialization Cost 

* During compaction, each key-value pair from the DRAM memtable is **moved (via memcpy()) to the NVM memtable without serialization**.
* NVM memtable skip list nodes are **linked by their relative offsets(using physical offset relative to the starting address of the root node) in a  memory-mappred region** instead of virtual address pointers and **commited in-place**.
  * persistent updates by using **hardware memory barriers and cacheline flush in-structions**

### Reducing Compaction Cost

* To address the issue of compaction stalls, NoveLSM makes **the NVM memtable mutable**, thereby **allowing direct updates to the NVM memtable** (Figure 5.b)
  * When the in-memory memtable is full, application threads can alternate to using the NVM memtable without stalling for the in-memory memtable compaction to complete.
* NoveLSM makes **NVM memtable active(mutable)** and **keys are directly added to mutable NVM memtable**. 
  * *Legacy: When the DRAM memtable is full, it is made immutable, and the background compaction thread is notified to move data to the SSTable.*
    * (DRAM immutable memtable으 SSTable로 flush 되고 만약 compaction이 발생해서 write stall이 발생한다면 멈추지 않고 NVM에 쓰기 시작한다.)
* For read operations, the most recent value for a key is fetched by **first searching the current active memtable**, **followed by immutable memtables and SSTables**.

### Reducing Logging Cost

* NoveLSM **eliminates logging** for inserts added to mutable NVM memtable by persisting updates in-place.
  * *Current Logging: Each key-value pair and 32-bit CRC checksum is first appended to a persistent log.*
  * For NoveLSM, **when inserting into the mutable persistent memtable in NVM,** all updates are written and **committed in-place** without requiring a separate log.
  * Log writes are avoided only for updates to NVM memtable, whereas, all inserts to the DRAM memtable are logged.
* The key-value pair is copied persistently by ordering the writes using **memory store barrier and cache line flush of the destination addresses, followed by a memory store barrier**
  * small updates to NVM (8-byte) are committed with **atomic store instruction not requiring a barrier**. 
  * Finally, for overwrites, the **old nodes** are marked for **lazy garbage collection**.

### Supporting Read Parallelism

* NoveLSM uses **only one worker thread for searching across the mutable DRAM and NVM memtable** because of the relatively smaller DRAM memtable size compared to the NVM memtable.
* NoveLSM always considers the value of a key returned by a thread accessing the highest level of LSM as the correct version. 
  * To satisfy this constraint, a worker thread Ti accessing Li is made to wait for other worker threads accessing higher levels L0 to Li−1 to finish searching, and **only if higher levels do not contain the key, the value fetched by Ti is returned.**
  * To overcome stalling higher-level threads to wait for the lower-level 
    * In NoveLSM, once a thread succeeds in locating a key, all lower-level threads are immediately suspended.
* Thread Management
  * NoveLSM always **colocates the master and the worker threads to the same CPU socket** to avoid the lock variable bouncing across processor caches on different sockets. 
  * Further, threads dedicated **to parallelize read operation are bound to separate CPUs from threads performing backing compaction**.
* Optimistic parallelism
  * Bloom filter for NVM and DRAM memtable.
  * *read parallelism is enabled only when a key is predicted to miss the DRAM or NVM memtable;* (잘 모르겠음...)

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/