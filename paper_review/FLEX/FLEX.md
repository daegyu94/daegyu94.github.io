---
title: "Finding and Fixing Performance Pathologies in Persistent Memory Software Stacks [ASPLOS '19]"
date: 2020-08-10
categories: PMKV
---

## Introduction

* File access via conventional file system interfaces (e.g. open, read, write, fsync...)
* oft-cited benefit of NVMM is direct access (**DAX) mmap**
  * allows an application to map the pages of an NVMM-backed file into its address space and then **access it via load and store instructions**
  * DAX **removes all of the system software overhead** for common-case accesses enabling the fastest-possible access to persistent data
  * App should adopt mmap-based interface
* Brief findings
  * FiLe Emulation with DAX (FLEX) provides almost as much benefit as building complex crash-consistent data structures in NVMM.
  * Block-based journaling mechanisms are a bottleneck for adapted NVMM file systems. **Adding DAX-aware journaling** improves performance on many operations.
  * The block-based compatibility requirements of adapted NVMM file systems limit their performance on NVMM in some cases, **suggesting that native NVMM file systems are likely to maintain a performance advantage.**
  * Poor performance in **accessing non-local memory (NUMA effects) can significantly impact NVMM file system performance.** Adding NUMA-aware interfaces to NVMM file systems can relieve these problems

## Background

### NVMM File Systems and DAX

* Direct Access (DAX)
  * the file system **avoid using the operating system buffer cache**: There **is no need to copy data from “disk” into memory** since file data is always in memory (i.e., in NVMM).
  * DAX-mmap allows msync() to be implemented in user space by flushing the affected cachelines and issuing a memory barrier.
  * mmap **must include the recently-added MAP_SYNC flag that ensures that the file is fully allocated and its metadata has been flushed to media.**
    * This is necessary because, without MAP_SYNC, in the disk-optimized implementations of msync and mmap that ext4 and xfs provide, msync can sometimes require metadata updates (e.g., to lazily allocate a page).
* Current processors provide **8-byte atomicity for stores to NVMM** instead of the sector atomicity that block devices provide.
  * native NVMM file systems 	
    * BPFS - copy-on-write file system that introduced short-circuit shadow paging
    * PMFS: first NVMM file system released publicly, has scalabiltiy issues with large directories and metadata operations
    * NOVA: gives each inode a separate log to ensure scalability, and combines logging, light-weight journaling and copy-on-write to provide strong atomicity guarantees to both metadata and data.
    * Strata: cross-media file system that runs partly in userspace, provides strong atomicity and high performance, but does not support DAX
  * adapted NVMM file system (ext4-dax, xfs-dax...)
    * add DAX support to the original file systems so that data page accesses bypass the page cache, **but metadata updates still go through the old block-based journaling mechanism.**
    * give up some features in DAX mode: **ext4 does not support data journaling in DAX mode**, so it cannot provide strong consistency guarantees on file data. 

### NVMM programming

* **To ensure persistence and consistency**, these libraries use instructions such as **clflushopt and clwb to flush the dirty cachelines**.
* **non-temporal store** instructions like **movntq** **to bypass the CPU caches and write directly to NVMM**
* **Enforcing ordering between stores** requires memory barriers such as **mfence**

## Adapting applications to NVMM

* Blithely running that code on a new, faster storage system will yield some gains, but fully exploiting NVMM’s potential will require some tuning and modification. The amount, kind, and complexity of tuning required will help determine how quickly and how effectively applications can adapt.

### SQLite

<img src="img/Figure 1.PNG" width="400" height=""/>

* We use Mobibench to **test the SET performance of SQLite in each journaling mode.** 
* Result
  * DELETE and TRUNCATE incur significant file system journaling overhead with ext4-DAX and xfs-DAX. 
  * NOVA performs better because **it does not need to journal operations that affect a single file.** 
  * NOVA keeps allocator state in DRAM and only writes it to NVMM on a clean unmount. 
* Optimization
  * We modified SQLite **to avoid allocation overhead by using fallocate to pre-allocate the WAL file.** 
  * **FiLe Emulation with DAX (FLEX) to avoid the kernel completely for writes to the WAL file**. 
    * To implement FLEX, **SQLite DAX-mmaps the WAL file into its address space and uses non-temporal stores and clwb to ensure the log entries are reliably stored in NVMM**. *(FLEX를 사용하여 App에서 커널 파일 시스템을 사용하지 않고 직접 유저스페이스에서  WAL 파일을 App 주소 공간으로 mmap하고  ntstore랑 clwb 로 load/store로 파일을 쓴다.)*

### Kyoto Cabinet and LMDB

<img src="img/Figure 2.PNG" width="400" height=""/>

* KC와 LMDB는 DAX 없이도 mmap을 사용하는데, DAX의 이점을 극대화 하려면 변경이 필요하다.
* Kyoto Cabinet
  * **memory maps the metadata region**, uses **load/store instructions to access and update it**, and calls **msync to persist the changes**
  * we change KC to use **FLEX writes to update the log**
  * WAL-FLEX clwb avoid msync(operates on pages rather than cache lines)
  * WAL-FLEX clwb + falloc + mremap: fallocate and mremap to resize the file
* LMDB
  * Btree-based lightweight DB library
    * performs **copy-on-write on data pages to provide atomicity**, a technique that requires frequent **msync** calls.
    * msync => clwb: improves performance

### RocksDB and Redis

<img src="img/Figure 3.PNG" width="350" height=""/><img src="img/Figure 4.PNG" width="350" height=""/>

* Redis

  * in-memory KV store as a caching layer and for message queue applications

  * **append only file (AOF) to log all the write operations**

  * xfs-DAX’s journaling overheads hurt append performance

  * Hash table => NVMM conversion, making it persistent would eliminate the need for the AOF

    * P-redis: Create fully-functional persistent version of hash table in NVMM using PMDK

      * CoW for atomic updates

      * To insert/update a key-value pair, we **allocate a new pair to store the data**, and **replace the old data by atomically updating the pointer in the hashtable**. 

      * Porting challenges *(Redis에 대해서 잘 모르는 내용...)*

        1. Redis represents and stores different objects with different encodings and formats, and P-Redis **has to be able to interpret and handle the various types ofobjects properly**. 

        2. Redis stores virtual addresses in the hashtable, and **P-Redis needs to either adjust the addresses upon restart if the virtual address of the mmap’d hashtable file has changed,** or change the internal hashtable implementation to use offset instead of absolute addresses. Neither option is satisfying, and we choose the former solution for simplicity. 

        3. whenever Redis starts, **it uses a random seed for its hashing functions**, and P-Redis **must make the seeds constant**.
        4.  Redis updates the hashtable entry before updating the value, and P-Redis **must persist the key-value pair before updating the hashtable entry for consistency.**
        5. P-redis hashtable **does not support resizing** as it requires journaling mechanism to guarantee consistency.

* RocksDB

  * RocksDB inserts **the data to a skip-list in DRAM**, and **appends the data to a write-ahead log (WAL) file**. **When the skip-list is full, RocksDB writes it to disk and discards the log file**.
  * Pmemtable: Since the skip-list contains the same information as the WAL file, **we eliminate the WAL file by making the skip-list a persistent data structure**



### Evaluating FLEX

<img src="img/Figure 5.PNG" width="400" height=""/>

* FLEX: 

  1. open() + DAX-mmap() to map the file into the application's address space.

  2. Replace read()/write() to userspace operations *(maybe load, store semantics?)*

  * Write: 
    * A FLEX write first checks if the file will grow as a result of the write. If so, the application can expand the file using fallocate() and mmap() or mremap() to expand the mapping. 
    * To amortize the cost of fallocate(), the application can extend the file by more than the write requires.
    * Once space is available, **a FLEX write uses non-temporal stores** to copy data into the file. 
    * FLEX also **uses an sfence instruction** to replace fsync().
  * Read: 
    * FLEX reads are simpler: They simply translate to **memcpy().**
  * FLEX requires the **application to track a small amount of extra state about the file**, including its location in memory, its current write point, and its current allocated size.
  * Different to POSIX
    * operations are not atomic. 
    * POSIX semantics for shared file descriptors are lost

* Evaluation

  * RocksDB uses “append and extend” , SQLite and Kyoto Cabinet use “circular append.”
  * The larger speedup for append and extend is due to the **NVMM allocation overhead**. Performance gains are especially large for small writes

### Best Practices

* Emulating file operations in user space pro- vides large performance gains for very little programmer effort.
* Use fine-grained cache flushing instead of msync
* Use complex persistent data structure judiciously
* Preallocate files on adapted NVMM file systems
* Avoid meta-data operations

### Reducing journaling overhead

<img src="img/Figure 6.PNG" width="300" height=""/> <img src="img/Table 1.PNG" width="400" height=""/>

* Ext4 uses the journaling block device (JBD2) to perform consistent metadata updates.
  * **it always writes entire 4 KB pages**
    * **Transactions often involve multiple metadata pages**: appending 4 KB data to a file and then calling fsync writes one data page and eight journal pages: a header, a commit block, and up to six pages for inode, inode bitmap, and allocator.
  * JDB2 also allows **no concurrency** between journaled operations
* **journaling DAX device (JDD) for ext4** which performs **DAX-style journaling on NVMM and provides improved scalability**.
  1. it **journals individual metadata fields** rather than entire pages
  2. it provides pre-allocated, per-CPU journaling areas so CPUs can perform journaled operations in parallel
  3. it uses **undo logging in the journals:** It copies the old values into the journal and performs updates directly to the metadata structures in NVMM.  *(redo가 아니라 undo..?)*

<img src="img/Figure 7.PNG" width="600" height=""/>

### NUMA Scalability

<img src="img/Figure 10.PNG" width="400" height=""/>

* Intelligently allocating memory in NUMA systems is critical to maximizing performance. 
  * Without the new mechanism, threads are spread across all the NUMA nodes, and some of them are accessing NVMM remotely.
* We created **a new ioctl that can set and query the preferred NUMA node for the file**
* A thread can use this ioctl along with Linux’s CPU affinity mecha- nism to bind itself to the NUMA node where the file’s data and metadata reside.
* *왼쪽 filebench는 강제로 numa-aware(ideal scenario), 오른쪽은 sstable이나 db 파일이 있는 노드로 스레드를 스케줄링하는 ioctl를 추가했을 경우.*

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/