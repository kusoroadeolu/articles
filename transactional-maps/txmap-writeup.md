# Transactional Maps
- Date Written: Thursday, February 26, 2026. 

## Writeup
The goal of this writeup isn't necessarily to sell the idea of in memory transactional maps to you, but rather document my findings about the different strategies of transactional maps I implemented in [this repo](https://github.com/kusoroadeolu/tx-map)
Before we continue, the main idea of transactional collections/maps, is to allow developers to achieve similar performance in large atomic regions as they would by interleaving multiple operations in mutual exclusion based primitives, while providing atomicity and different isolation levels based on preferences.
With that said, lets get into it


## Introduction
I initially started building these transactional maps with only a single implementation which was an optimistic like transactional map with semantic concurrency control. I got most of the ideas of this from this [paper](https://people.csail.mit.edu/mcarbin/papers/ppopp07.pdf). 
The main idea was pretty clear: **transactional semantics for a map using optimistic concurrency control**. However, the paper assumed the reader/implementor had an underlying STM (Software Transactional Memory) runtime, hence, all the talk about rollbacks, aborts, open-nested, closed-nested transactions and memory level conflicts that the STM runtime handled for them.
Most of the ideas presented by the paper were very smart, and a lot of the ideas were transferred into a good number of the transactional maps I built, however there were a lot of significant limitations which prevented me from making a near 1:1 copy of what the paper suggested and honestly what I initially planned to do. 
These limitations, rather than being roadblocks, ended up pushing me to explore different synchronization strategies and ultimately build 5 distinct transactional map implementations, each with their own tradeoffs.

## Transactional Map Implementations
1. **Optimistic Transactional Map:** At first glance, the name of this map could imply that this map carries out optimistic reads, however, the transactional semantics of the map are quite different. 
Before I explain the map's semantics, this map's commit phase is split into two, validation and commit. During validation, the transaction tries to acquire all semantically meaningful locks for the transaction i.e. if a put operation for key, already has a value for that key in the map, the contains key nor size lock is acquired.
This transactional map's semantics require readers eagerly stating their intent by acquiring read locks for their semantics at scheduled time(before commits). Writes however lazily state their intent and are serialized through one lock per key. This map promises **READ COMMITTED** isolation levels across the entire map but **SERIALIZED** isolation levels per key when writes are involved , meaning no dirty reads can ever happen on any key in the map, and repeatable or phantom reads are impossible per key.

2. **Pessimistic Transactional Map:** This transactional map's semantics require readers lazily stating their intent by incrementing an atomic counter at validation time. Writes however are serialized through one lock per key and lazily state their intent. This map also promises **READ COMMITTED** isolation levels across the entire map but **SERIALIZED** isolation levels per key 
**NOTE:** For both optimistic and pessimistic maps, writers block readers and readers block writers, but readers never block each other.

3. **Read Uncommitted Transactional Map:** This transactional map's semantics do not require readers stating their intent at all. Writes however are still serialized through one lock per key and lazily state their intent. This map promises **READ UNCOMMITTED** isolation levels, meaning transactions can read data by transactions not fully committed yet i.e. Dirty reads. In the [paper](https://people.csail.mit.edu/mcarbin/papers/ppopp07.pdf), this is regarded as an **open-nested** transaction.

4. **Copy On Write Transactional Map:** This transactional map's semantics can be linked closely to that of **CopyOnWriteArrayList**. This copies the last seen map reference onto it's thread stack, if the map on the stack was modified, it tries to replace the shared map reference using CAS, if it fails, it recopies to map, modifies and tries to cas until it succeeds. This map promises **READ COMMITTED** isolation levels across the entire map. Repeatable and phantom reads are possible if you read a value, another thread commits, and you read again, you might see the new value

5. **Flat Combined Transactional Map:** I wanted to see how a fully serialized approach would compare, which led me to flat combining.

### Flat Combining
Flat combining is a synchronization semantic that proposes, rather than using fine-grained locking for critical sections, a **coarse-grained lock** is used for all operations. Now, rather than each thread contesting for the lock on the data structure which would kill performance, each thread enqueues their proposed operation onto a queue as a node and stores their operation in a thread local variable, then spins or waits until a combiner processes their request, if no combiner is active, any thread can become one. The combiner, traverses the queued operations and performs those operations on behalf of those threads, then unlocks the lock. 
My explanation is a pretty simplified version of what flat combining actually entails and if you're interested in going into the actual details for flat combining, you can check it out [here](https://people.csail.mit.edu/shanir/publications/Flat%20Combining%20SPAA%2010.pdf)
There are two possible combiners my transactional map could use:
- **Unbound Combiner:** This combiner implementation is faithful to what the paper describes, threads storing their intent in thread local variables, enqueuing its actions unto a shared publication queue and then becoming the combiner or waiting for the combiner to perform its operation of its behalf.
- **Semaphore Combiner:** This combiner implementation cycles nodes across threads. Here, threads replace their thread local nodes with the current node at the head of the combiner publication queue, then sets that node as their tail. A combining node, then scans the queue, executes all actions on behalf of the waiting threads except the node at the tail of the queue, then informs the thread holding node at the tail of the queue that it is the next combiner. This combiner does not use a lock. 
 

## Benchmarks
To evaluate the properties of all my transactional maps, I ran two benchmarks: a disjoint key benchmark, where each thread operates exclusively on its own key (no key contention by design), and a contention benchmark, where all threads compete over a small fixed pool of 4 keys across three workload profiles — read heavy (90% get operations), balanced (50/50), and write heavy (90% put operations) both primarily focusing on throughput. I will only be focusing in this writeup on the numbers on read/write heavy portions of the contention benchmarks
It is important to point out that combining operations are fully serialized, hence these benchmarks may not actually be measuring what we think they'll be measuring potentially yielding false results. So I won't focus too much on the flat combined numbers until later, and then I'll share how I plan on improving them and more importantly reducing their error margins.
 
**NOTE:** These benchmarks were run using JMH(Java MicroHarness Benchmark)

### Benchmark Setup
- JMH version: 1.37
- JVM: Java 25, HotSpot 64-Bit Server VM (25+37-LTS-3491)
- Benchmark mode: Throughput (ops/s)
- Warmup: 5 iterations × 1s each
- Measurement: 5 iterations × 1s each
- Forks: 2
- Thread configuration: 1, 2 , 4, 8 (Platform threads)
- CPU Specs: Intel(R) Core(TM) i5-10300H CPU @ 2.50GHz (2.50 GHz), 8 cores


### Read Heavy
![Read Heavy - 1 Thread](https://docs.google.com/spreadsheets/d/e/2PACX-1vQCZMJbJvC9LBh-MxqySWYzYu_FaZ5z2hAX2eQyxyzRh_QEvyBJcoQrWem-sMDmwYjZzxN1XQjZcHtD/pubchart?oid=419529405&amp;format=image)
![Read Heavy - 8 Threads](https://docs.google.com/spreadsheets/d/e/2PACX-1vQCZMJbJvC9LBh-MxqySWYzYu_FaZ5z2hAX2eQyxyzRh_QEvyBJcoQrWem-sMDmwYjZzxN1XQjZcHtD/pubchart?oid=1075497262&amp;format=image)

Under contention, the copy on write implementation's performance, throughput grows from ~3.7M ops/s at 1 thread to 6.4M at 8 threads (+72%), since readers don't block each other at all. This is a known strength of copy on write implementations which sacrifice write throughput for high read throughput. 

The pessimistic transactional map's throughput drops from ~1.2M ops/s at 1 thread to ~865k at 8 threads. This is mainly due to the strong isolation guarantees this implementation provides, blocking readers while writes are active and vice versa. Also, context switches from multiple reader threads waking up after a writer has notified them contributes to this throughput drop as well

And for the optimistic transactional map, the story is similar to that of pessimistic, throughput drops from ~1.27M ops/s at 1 thread to ~701k ops/s at 8 threads, though read operations scale worse than pessimistic due to similar problems, though rather than only reader threads performing OS context switches, writers too, now also park/unpark due to lock contention.

Read uncommitted implementation's performance is more balanced with throughput growing from ~2.0M ops/s at 1 thread to ~3.1M at 8 threads compared to the others, though it offers much weaker isolation guarantees compared to the previous three.

### Write Heavy
![Write Heavy - 1 Thread](https://docs.google.com/spreadsheets/d/e/2PACX-1vQCZMJbJvC9LBh-MxqySWYzYu_FaZ5z2hAX2eQyxyzRh_QEvyBJcoQrWem-sMDmwYjZzxN1XQjZcHtD/pubchart?oid=1590911224&amp;format=image)
![Write Heavy - 8 Threads](https://docs.google.com/spreadsheets/d/e/2PACX-1vQCZMJbJvC9LBh-MxqySWYzYu_FaZ5z2hAX2eQyxyzRh_QEvyBJcoQrWem-sMDmwYjZzxN1XQjZcHtD/pubchart?oid=1945883913&amp;format=image)

Copy on Write's throughput drops ~60% from 1 thread 2.4M ops/s to 8 threads 941K ops/s.Unlike reads, write operations cause multiple CAS retries and the over head of recopying the shared map on each retry under high contention becomes a big bottleneck.

The pessimistic transactional map's throughput drops ~60% from 1 thread ~927k ops/s to 8 threads ~388k ops/s. Since all writes per key are serialized under one lock to enforce serial writes and uphold its isolation guarantees, the overhead of multiple OS context switches due to lock contention becomes a primary bottleneck, as threads spend more time waiting to acquire locks than actually doing work

While for the optimistic transactional map, the story is similar to that of pessimistic, throughput drops from ~883k ops/s at 1 thread to ~304k ops/s at 8 threads, though read operations scale worse than pessimistic due to similar problems, much more time is spent waiting to acquire locks than doing actual work.

Read uncommitted implementation offers a much more interesting view as throughput only goes down ~18% from 1 thread ~1.36M ops/s to 8 threads at ~1.11M ops/s, offering the best write performance out of the three implementations, due to weak isolation guarantees.

From these numbers we can see that no single implementation wins across all workloads, the right choice depends entirely on your read/write ratio and how much you're willing to trade isolation guarantees for throughput.


## Conclusions
### Flat Combining
Looking back at the flat combining numbers even though both have terrible variance for both read/write heavy workloads, I do want to offer what I plan to do to improve them.
1. Experimenting with multiple idle strategies. Right now, both implementations while idle, spin for **n** number of times, before rechecking their results and potentially trying to become the combiner, this could lead to wasted work as their results could be ready much earlier or the combiner might have forfeited its status earlier as well. Two other strategies I'd implement and benchmark against are
- **Plain Busy Waiting:** Threads just spin on their results continuously without spinning **n** number of times before checking their results or trying to acquire the lock. Though constant CAS `tryLock()` failures could in the unbound combiner could ideally make throughput and its variance worse than just using a global lock
- **Timed Parks:** This involves threads parking for a fixed amount of time before rechecking their result, though OS context switches might actually become a bottleneck here.

2. Using atomic arrays instead of a publication linked queue: Right now, both combiners use a linked publication queue to share nodes for the combiner to process. Linked queues are susceptible to pointer chasing which could put pressure on the GC and JVM, rather we could use an **AtomicReferenceArray** with a fixed threshold to store nodes, in which the combiner can traverse. However, this could leave nodes susceptible to false sharing.

3. Increasing the threshold of the publication queue of both implementations: One of flat combining's strengths is its ability to batch apply requests for other waiting threads, increasing this amount while increasing wait time, could increase reduce the amount of serial handoffs needed for each combiner and less failed CAS operations to acquire the lock to become the combiner

### Further Experimentation
We've compared multiple implementations of these transactional maps against each other in this writeup, here are some ways I plan to push my implementations further
1. MVCC Transactional Maps: MVCC(Multi Version Concurrency Control) is a very common technique used in relational databases providing **READ-COMMITTED** isolation guarantees while ensuring readers and writers do not block each other
2. Partitioned Flat Combined Transactional Maps: Rather than wrapping the whole transactional map in a combiner, we instead map each key and the **size** value to a combiner, transactions submit their read/write requests per key and wait on a response.

As I implement these, I'll be sure to document the performance/throughput of every experiment I perform here.
**GitHub:** https://github.com/kusoroadeolu/tx-map