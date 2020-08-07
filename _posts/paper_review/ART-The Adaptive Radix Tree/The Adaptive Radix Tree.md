---
title: "Redesigning LSMs for Nonvolatile Memory with NoveLSM Sudarsun"
date: 2020-08-06
categories: PMKV
---

## Background

### Log Structured Merge Tree

<img src="img/Figure 1.PNG" width="350" height=""/>

### LevelDB

* LSMs achieve higher write throughput by first staging data in memory and then across multiple levels on disk to avoid random disk writes.
* During an insert operation, LevelDB buffers updates in a memory-based skip list table (referred to as memtable hereafter) and stores data on multiple levels of on-disk block structures know as sorted string tables (SSTable).
* With an exception to memtables, all lower levels are mutually exclusive and do not maintain redundant data

### RocksDB

* RocksDB achieves this by supporting multi- threaded background compaction which can simultaneously compact data across multiple levels of LSM hierarchy and extracts parallelism with multi-channel SSDs
* RocksDB that can be beneficial for NVM is the use of a Cuckoo hashing-based SST table format optimized for random lookup instead of the traditional I/O block-based format with high random-access overheads.

## Motivation

<img src="img/Figure 2,3,4.PNG" width="" height=""/>

* As shown in Figure 2, the results show that current LSMs do not fully exploit the hardware benefits of NVM and suffer from significant software overheads. 
* Figure 3 shows the cost breakup for insert operations with 4 KB, 8 KB, and 16 KB values.
  * **data compaction dominates the cost**
* Increasing the in-memory buffer (memtable) can reduce compaction frequency, but has drawbacks
  * DRAM usage increases by two times: memory must be increased for both mutable and immutable memtables. 
  * only after the immutable memtable is compacted, log updates can be cleaned, leading to a larger log size
  * LSMs do not enforce commits (sync) when writing to a log => an application crash or power-failure could lead to data loss
* Figure 4, For small values, searching the SSTable dominates the cost, 
  * (e.g., 16 KB) the deserialization cost increases with increasing value size .

## Design

#### Principle 1: Exploit byte-addressability to reduce se- rialization and deserialization cost.

* NoveLSM provides a **persistent NVM memtable** by designing a persistent skip list
* the **DRAM memtable data can be directly moved to the NVM memtable** without requiring serialization or deserialization

#### Principle 2: Enable mutability of persistent state and leverage large capacity of NVM to reduce compaction cost.

* NoveLSM designs a large mutable persistent memtable to which applications can directly add or update new key-value pairs.
* The persistent memtable allows NoveLSM to alternate between small DRAM and large NVM memtable without stalling for background compaction to complete.

#### Principle 3: Reduce logging overheads and recovery cost with in-place durability

* NoveLSM avoids logging by **immediately committing updates to the persistent memtable in-place**.

#### Principle 4: Exploit the low latency and high band- width of NVM to parallelize data read operations.

* NoveLSM exploits NVMs’ low latency and high bandwidth by parallelizing search across the memory and storage levels.

<img src="img/Figure 5.PNG" width="" height=""/>

### Addressing (De)serialization Cost 

* During compaction, each key-value pair from the DRAM memtable is **moved (via memcpy()) to the NVM memtable without serialization**.

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/