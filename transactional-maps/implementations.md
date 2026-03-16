# Transactional Map Implementations 
This writeup contains all my transactional map implementations so far. All my transactional maps are fully lazy until `commit()` is called.


## 1. Optimistic Transactional Map

### Isolation Level
- **READ COMMITTED** globally
- **SERIALIZABLE per-key** when writes are involved
- No dirty reads, no phantom reads per key

### Implementation
**Core Data Structures:**
- `ConcurrentMap<K, V>` — The underlying map
- `KeyToLockers<K>` — Maps each key to operation-specific `GuardedTxSet`s
- Each `GuardedTxSet` contains:
    - A `ReentrantReadWriteLock` (split into read/write locks)
    - A set of transactions waiting on that operation

### Transaction Lifecycle

#### 1. Schedule Phase (when operations are called)

```java
tx.get(key)       // Acquires READ lock immediately
tx.put(key, val)  // Does NOT acquire any lock yet
```

**For reads (GET/CONTAINS/SIZE):**
- Immediately acquire the **read lock** for that operation type on that key
- Add transaction to the `GuardedTxSet` for that key + operation
- Multiple readers can proceed concurrently

**For writes (PUT/REMOVE):**
- No locks acquired yet (lazy intent declaration)
- Just record the operation

#### 2. Validation Phase

**For reads:**
- Already validated (holding read lock since schedule time)

**For writes:**
- Sort all write keys by `identityHashCode()` (prevents deadlock via global ordering)
- For each key being written:
    1. Acquire the **write lock** for that key
    2. For each conflicting read operation type (GET, CONTAINS):
        - Check if this transaction holds the **read lock** for that operation
        - If yes, **release the read lock first** (prevent upgrade deadlock)
        - Acquire the **write lock** for that operation type
    3. Check if SIZE lock is needed:
        - `PUT` on new key → acquire SIZE write lock (size will increase)
        - `PUT` on existing key → release CONTAINS write lock if held (size unchanged, CONTAINS always true)
        - `REMOVE` on existing key → acquire SIZE write lock (size will decrease)
        - `REMOVE` on non-existent key → release CONTAINS write lock if held (size unchanged, CONTAINS always false)

#### 3. Commit Phase
- Apply all operations to the underlying map
- Release all held locks (both read and write)
- Remove transactions from all `GuardedTxSet`

### Key Insights
- **Readers block writers, writers block readers, but readers never block readers**
- The lock upgrade implementation (release read → acquire write) prevents self-deadlock
- Semantic awareness: doesn't acquire **SIZE** lock when not semantically necessary. Acquire **CONTAINS** lock, only holds on to it if in the case of **REMOVE**(the value already existed), and, in the case of **PUT** the values did not exist already 

---

## 2. Pessimistic Transactional Map

### Isolation Level
- **READ COMMITTED** globally
- **SERIALIZABLE per key** when writes are involved

### Implementation

**Core Data Structures:**
- `ConcurrentMap<K, V>` — The underlying map
- `KeyToLockers<K>` — Maps each key to operation-specific `GuardedTxSet`s
- Each `GuardedTxSet` contains:
    - A `ReentrantLock` (write lock)
    - An `AtomicInteger` reader count
    - A `Latch` with status (FREE/HELD), parent transaction, and `CountDownLatch`

### Transaction Lifecycle

#### 1. Schedule Phase

```java
tx.get(key)       // Does NOT acquire lock, just registers
tx.put(key, val)  // Does NOT acquire lock, just records
```

**For reads:**
- Add transaction to the appropriate `GuardedTxSet`
- No locks acquired (lazy intent declaration)

**For writes:**
- Just record the operation
- No locks acquired

#### 2. Validation Phase

**For writes:**
- Sort all write keys by `identityHashCode()` for deadlock prevention
- For each write key:
    1. Acquire write locks for GET and CONTAINS operation types on that key
    2. Check if key exists in underlying map
    3. Determine if SIZE lock needed (same logic as optimistic):
        - `PUT` on new key -> acquire SIZE lock
        - `REMOVE` on existing key -> acquire SIZE lock
    4. Set latch status to HELD with this transaction as parent
    5. **Drain existing readers**: spin-wait while `readerCount > 0`

**For reads:**
- Check the latch for the operation type on that key
- If latch is HELD by another transaction:
    - **Waits** on the `CountDownLatch` (blocks until writer releases)
    - When awakened, recheck if the latch has been reacquired else increment `readerCount`
- If latch is FREE:
    - Just increment `readerCount`

#### 3. Commit Phase
- Apply all operations
- For writers: set latch to FREE, countdown the latch (wake waiting readers)
- Release all locks
- For readers: decrement `readerCount`
- Remove from `GuardedTxSet`s

### Key Insights
- **Readers use atomic counters, not locks** — cheaper than lock acquisition
- Writers must **drain readers** before proceeding (spin on `readerCount`)
- The latch mechanism prevents TOCTOU bugs: check status and await in single atomic read
- More OS context switches due to readers parking/unparking on `CountDownLatch`

---

## 3. Read Uncommitted Transactional Map

### Isolation Level
- **READ UNCOMMITTED** — dirty reads allowed (can see uncommitted writes)

### Implementation

**Core Data Structures:**
- `ConcurrentMap<K, V>` — The underlying map
- `LockHolder<K, V>` — Per-key `ReentrantLock`s for writes
- Per-transaction **store buffer** (`Map<K, V>`) — local cache of reads, probably might not need this looking back at this

### Transaction Lifecycle

#### 1. Schedule Phase
- All operations just record intent
- No locks acquired for reads OR writes

#### 2. Validation Phase

**For writes:**
- Sort write keys by `identityHashCode()`
- Acquire per-key locks in sorted order

**For reads:**
- No validation needed

#### 3. Commit Phase

Before committing any operations:

```java
// Snapshot current map size
tx.size = txMap.map.size();

// Pre-populate store buffer with current values for each operation with a key
storeBuf.put(key, txMap.map.get(key));
```

Then for each operation:

**For writes (PUT/REMOVE):**
- Apply to underlying map immediately
- Update store buffer with new value
- Track size delta:
    - `PUT` on new key: `delta++`
    - `REMOVE` on existing key: `delta--`

**For reads (GET):**
- First check **store buffer**
- If not found, perform **dirty read** from underlying map
- Return result

**For CONTAINS:**
- Check store buffer only (already populated)

**For SIZE:**
- Return `snapshotSize + delta`

### Unique Properties
- **No reader blocking at all** — highest read throughput
- **Dirty reads** mean you might see uncommitted data
- Store buffer provides **read-your-own-writes** consistency within a transaction
- Only ~18% throughput drop under write-heavy contention (best of all implementations)

---

## 4. Copy-on-Write Transactional Map

### Isolation Level
- **READ COMMITTED** globally (but dirty reads possible between snapshot and commit)
- Repeatable reads not guaranteed

### How It Works

**Core Data Structures:**
- `AtomicReference<ConcurrentMap<K, V>>` — Pointer to current map snapshot

### Transaction Lifecycle

#### 1. Schedule Phase
- Just record operations in local list
- No locks, no map access

#### 2. Validation Phase

**Retry loop:**

```java
do {
    prev = txMap.map.get();                      // Get current snapshot
    underlyingMap = new HashMap<>(prev);          // COPY entire map
    // Apply all operations to local copy
    txs.forEach(child -> child.tryValidate());
} while (hasWrite && !txMap.map.compareAndSet(prev, new ConcurrentHashMap<>(underlyingMap)));
```

**tryValidate():**
- Apply operation to the local copy (`underlyingMap`)
- Store result in child transaction

**If CAS fails:**
- Another transaction committed between snapshot and CAS
- Retry: re-copy map, re-apply operations, re-attempt CAS

#### 3. Commit Phase
- CAS succeeded, so operations already applied to new map
- Just complete futures with stored results

### Unique Properties
- **Entire map copied on every write transaction** — doesn't scale with map size
- Under high contention: lots of CAS retries -> lots of copies -> terrible write heavy performance
- **Best read performance** when map is mostly read (no blocking, no locking)
- Simple implementation but hard on memory and GC under high write contention

---

## 5. Flat Combined Transactional Maps

### Isolation Level
- **SERIALIZABLE** (all operations fully serialized through combiner)

### Core Idea

Instead of threads fighting for locks, they:
1. Enqueue their operation
2. Spin waiting for a "combiner" thread to execute it
3. If no combiner exists, try to become one
4. Combiner executes operations for **all** waiting threads in one critical section

### Implementation Variants

#### A. FlatCombinedTxMap (using combiners)

Uses one of four combiner types:

---

##### UnboundCombiner (Faithful to paper)

**Data Structures:**
- Thread-local `Node<E, R>` for each thread
- Shared publication list (linked via `AtomicReference<Node>` head)
- Per-node `StatefulAction` with:
    - `volatile Action<E, R> action`
    - `R result`
    - `AtomicInteger status` (ACTIVE/INACTIVE)
- Dummy sentinel node marks end of queue

**Implementation:**

_Enqueue:_
```java
if (node.isInactive()) {
    node.setActive();
    prevHead = head.getAndSet(node);  // Atomic swap
    node.setNext(prevHead);
}
```

_Combine:_
```java
while (action != null) {             // Spin until result ready
    if (lock.tryLock()) {
        scanCombineApply();           // Process all nodes
        return result;
    }
    idle();
    enqueueIfInactive(node);          // Re-enqueue if removed
}
```

_scanCombineApply:_
- Traverse from head to DUMMY
- For each node with non-null action:
    - `node.statefulAction.apply(e)`
    - `node.action = null` (signals completion)
    - Set age to current count
- Every `threshold` passes, scan and **dequeue aged nodes**:
    - If `(currentCount - node.age) >= threshold`: unlink, set INACTIVE

**A Livelock Bug I Fixed:**
Threads could get stuck when the combiner removed their node before they noticed their action was applied. Fixed by forcing combiners to always re-enqueue and apply their own action rather than trusting others.

---

##### NodeCyclingCombiner (Reuses nodes)

**Key Difference:** Reuses nodes instead of cleanup.

**Data Structures:**
- Thread-local `Node<E, R>`
- Each node has `AtomicInteger status` (NOT_COMBINER/IS_COMBINER)
- Shared `AtomicReference<Node>` tail

**How It Works:**

```java
// Reset current node
Node newTail = local.get();
newTail.status.set(NOT_COMBINER);

// Swap, my node becomes tail, get previous tail
curNode = tail.getAndSet(newTail);
local.set(curNode);              // Previous tail becomes my node

curNode.action = action;
curNode.setNext(newTail);

// Wait to become combiner
while (curNode.status.get() == NOT_COMBINER) {
    idle();
}

//Node chain should look something like this given 3 threads(T1, T2, T3 with the inital node T0),  waiting for their result to be applied, assuming natural order
//TO -> T1 -> T2 -> T3 
// T3 node will be marked as the combiner when Thread 1, finishes applying 
        
// Now I'm the combiner, traverse and apply
for (node = curNode; i < threshold && node.next != null; node = node.next) {
    node.apply(e);
    node.action = null;
    node.next = null;
    node.status.lazySet(IS_COMBINER);  // Make next last node in the queue combiner
}
```

**Unique Property:** Nodes cycle between threads, no cleanup needed.

---

##### AtomicArrayCombiner (Bounded combiner)

**Data Structures:**
- `AtomicReferenceArray<Node>` of size `capacity`
- `AtomicLong cellNum` — monotonically increasing cell assignment
- Per-thread `ThreadLocal<Node>`

**Implementation:**

_Enqueue:_
```java
cell = (cellNum.getAndIncrement() % capacity);
// CAS into array slot
while (!cells.compareAndSet(cell, null, node)) {
    Thread.yield();  // Slot occupied, retry
}
```

_Combine:_
```java
while (!node.statefulAction.isApplied) {
    if (lock.tryLock()) {
        scanCombineApply();  // Scan entire array
        return result;
    }
    idle();
}
```

_scanCombineApply:_
```java
for (i = 0; i < capacity; i++) {
    Node curr = cells.get(i);
    if (curr != null) {
        curr.statefulAction.apply(e);
        cells.setOpaque(i, null);  // Clear and apply every slot to prevent a situation where waiters spin on an unapplied node forever
        //Best when working with a fixed capacity of threads
    }
}
```

**Unique Property:** Fixed memory, no pointer chasing, but potential false sharing.

---

##### SynchronizedCombiner (Baseline)

```java
lock.lock();
try {
    return action.apply(e);
} finally {
    lock.unlock();
}
```

Plain lock — used as baseline to measure if flat combining actually helps.

---

#### B. SegmentedCombinedTxMap (Partitioned)

**Key Difference:** Instead of one combiner for the entire map, **one combiner per key + one for SIZE**.

**Data Structures:**
- `ConcurrentMap<K, Combiner<Map<K, V>>>` — Maps each key to its own combiner
- Separate `sizeCombiner` for SIZE operations

**How It Works:**

```java
// Group operations by key
for (key, operations) in keyToFuture:
    combiner = getCombiner(key);  // Get or create combiner for this key
    combiner.combine(_ -> {
        for (operation in operations) {
            result = operation.apply(map);
            operation.complete(result);
        }
    });

// Separately handle SIZE operations
sizeCombiner.combine(_ -> { /* apply size ops */ });
```

**Unique Property:**
- Better parallelism — different keys can be combined concurrently
- More overhead — one combiner per key
- Writes per key still serialized, but across different keys they're parallel
Oddly, from the benchmarks, a single combiner scales better than a segmented combiner, mainly due to the fact that combiners benefit the fact that batching operations and spinning, amortizes the cost of lock contention, unlike a segmented combiner where batching is less common due to each operation of a transaction mainly working on different keys
---

## Summary

| Implementation | Isolation Level | When Locks Acquired | Readers Block Writers? | Writers Block Readers? | Probably Best For                                |
|---|---|---|---|---|--------------------------------------------------|
| **Optimistic** | READ COMMITTED (per-key SERIALIZABLE) | Reads: eager, Writes: lazy | Yes | Yes | Balanced workloads with strong consistency needs |
| **Pessimistic** | READ COMMITTED (per-key SERIALIZABLE) | Both lazy | Yes | Yes | Similar to optimistic                            |
| **Read Uncommitted** | READ UNCOMMITTED | Writes only at validation | No | Yes | Write-heavy and weak consistency is OK           |
| **Copy-on-Write** | READ COMMITTED | Never (uses CAS) | No | No | Read heavy or small map size                     |
| **Flat Combined** | SERIALIZABLE | N/A (combiners) | N/A | N/A | High contention, batch benefits                  |
| **Segmented Flat** | SERIALIZABLE | N/A (per-key combiners) | N/A | N/A | Outperformed by a single flat combined map       |