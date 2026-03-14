# Introduction
This post walks through the performance journey of an MVCC transactional map I've been building in Java, from a few thousand ops/s to a few million. It's mostly numbers and profiling observations, so if that's not your thing, fair warning.

## Benchmark Setup
- JMH version: 1.37
- JVM: Java 25, HotSpot 64-Bit Server VM (25+37-LTS-3491)
- Benchmark mode: Throughput (ops/s)
- Warmup: 10 iterations × 1s each
- Measurement: 5 iterations × 1s each
- Forks: 2
- Thread configurati- on: 1, 2 , 4, 8 (Platform threads)
  CPU Specs: Intel(R) Core(TM) i5-10300H CPU @ 2.50GHz (2.50 GHz), 8 cores

## The Journey
Initially my MVCC txMap had good read numbers for thrpt and decent write numbers, though the error margins for the write numbers were bad, so I decided to investigate. While investigating, I encountered an issue. Also, just a quick note before we continue that `ActiveTransactions` just keeps tab on all active transactions at the moment.


```java
//A map keeping track of all active transactions
ActiveTransactions activeTxns = mvccMap.activeTransactions.copy(); //Copied the entire map on active txns, could be thousands 
long minVisibleEpoch = activeTxns.findMinVisibleEpoch(); 
versionChain.removeUnreachableVersions(minVisibleEpoch);

// ActiveTransactions method call
long findMinActiveEpoch(){
    Set<Long> set = new HashSet<>(map.values()); //Copy to set to prevent duplicates and min traversals, caused an OOME!!
    long min = 0;
    int count = 0;
    for (long l : set){
        if (count == 0 || l < min){
            min = l;
        }
        ++count;
    }

    return min;
}
```

This line of code caused an OOME under contention the active given the fact the active transactions map could be very large, if we’re copying and iterating it anytime we want to prune an old version, issues would occur. To fix this, I used a single writer that automatically tracks the minimum visible epoch(tBegin) from all active transactions in the set. The main tradeoff with this was that under contention the single writer could not keep up, but it was better than copying the map on every write transaction.

However, write throughput dropped to ~20k ops/s after the single writer fix, so I profiled to find out why. The culprit was `SerialTransactionKeeper#remove`. Instead of submitting a remove request to the queue, it was removing a non-existent entry, meaning every transaction commit was doing an O(N) traversal for nothing. And since nothing was actually being removed from the map, the `minVisibleEpoch` never advanced, so the GC epoch reclamation was doing meaningless work the entire time. Fixing this brought numbers back to a reasonable range.

**Before**

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 1,248.762 | ± 921.068 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,205.265 | ± 413.555 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,407.078 | ± 528.927 | ops/s |
| readHeavy_8threads | thrpt | 10 | 2,566.666 | ± 610.823 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 2,021.587 | ± 487.228 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 1,475.348 | ± 388.932 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 1,693.183 | ± 532.851 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 13,806.692 | ± 4,179.536 | ops/s |

**After**

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 454,138.457 | ± 92,558.035 | ops/s |
| readHeavy_2threads | thrpt | 10 | 449,543.650 | ± 226,056.899 | ops/s |
| readHeavy_4threads | thrpt | 10 | 623,248.848 | ± 177,717.798 | ops/s |
| readHeavy_8threads | thrpt | 10 | 464,862.805 | ± 234,641.969 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 369,462.166 | ± 79,723.306 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 445,274.916 | ± 171,877.221 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 728,382.685 | ± 174,790.985 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 902,537.739 | ± 93,291.275 | ops/s |


After another round of profiling, I realized that I was making `findOverlap()` calls frequently on my read heavy transactions, so basically an O(n) traversal for each find overlap call, which just becomes worse as the version queue per key grows, so I decided to use a different approach, I decided to use a navigable map as my version chain to reduce this traversal time per call to O(logN), at the cost of more expensive writes, and the numbers showed significant improvement

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 601,083.402 | ± 165,550.604 | ops/s |
| readHeavy_2threads | thrpt | 10 | 642,442.703 | ± 187,617.695 | ops/s |
| readHeavy_4threads | thrpt | 10 | 688,103.749 | ± 89,658.431 | ops/s |
| readHeavy_8threads | thrpt | 10 | 690,432.423 | ± 101,032.453 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 454,926.218 | ± 184,083.441 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 594,926.342 | ± 92,822.567 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 836,488.562 | ± 80,008.830 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 979,821.239 | ± 89,332.281 | ops/s |

After looking through my code again, I realized I never actually started the worker thread for my `SerialTransactionKeeper` class. After starting the thread and benchmarking again I was basically back to square one

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 37,911.129 | ± 13,675.765 | ops/s |
| readHeavy_2threads | thrpt | 10 | 89,182.952 | ± 16,428.530 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,197,666.295 | ± 370,356.847 | ops/s |
| readHeavy_8threads | thrpt | 10 | 868,822.634 | ± 334,063.540 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 2,849.386 | ± 1,326.803 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 9,548.036 | ± 6,833.268 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 25,872.105 | ± 13,197.781 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 1,132,311.163 | ± 285,098.340 | ops/s |

Reading through my profile data, I realized that garbage collecting on a writer transaction thread was causing a lot of inconsistencies across all benchmarks and thread counts, so I decided to move this to a separate worker thread. Now, a writer txn submits a cleanup request to the GC thread when the version chain depth reaches a certain threshold

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 864,770.978 | ± 123,348.212 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,187,432.216 | ± 171,323.762 | ops/s |
| readHeavy_4threads | thrpt | 10 | 827,029.790 | ± 525,292.066 | ops/s |
| readHeavy_8threads | thrpt | 10 | 661,419.334 | ± 215,284.213 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 476,757.401 | ± 91,269.885 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 622,144.154 | ± 155,951.737 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 767,387.142 | ± 171,878.778 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 819,022.141 | ± 337,489.007 | ops/s |

As a minor optimization, I decided to cache the `minVisibleEpoch` timestamp in my version chain, to prevent redundant iterations through version chains

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 1,384,520.967 | ± 151,058.220 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,559,840.856 | ± 361,048.003 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,328,777.375 | ± 619,468.965 | ops/s |
| readHeavy_8threads | thrpt | 10 | 843,850.257 | ± 241,581.930 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 527,538.045 | ± 160,021.700 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 836,844.450 | ± 95,255.683 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 944,841.844 | ± 321,298.094 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 764,114.028 | ± 472,135.579 | ops/s |

Looking at my profiled data again, I realized submitting requests to the GC thread was still a hotspot, so I decided to spread out the frequency in which requests are submitted by only submitting cleanup requests when `depth % threshold == 0` rather than every time `depth > threshold`, spacing out GC cleanup submissions across writes

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 1,499,641.094 | ± 186,580.267 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,800,303.740 | ± 160,148.052 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,420,702.483 | ± 566,203.194 | ops/s |
| readHeavy_8threads | thrpt | 10 | 828,145.076 | ± 393,441.472 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 654,820.291 | ± 132,849.682 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 895,029.640 | ± 125,955.480 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 987,045.255 | ± 437,727.470 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 853,466.979 | ± 556,489.979 | ops/s |

Write numbers improved slightly and error margins tightened up. Next I tried segmenting my active transactions class just to measure the difference, and the results were a bit surprising

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 601,762.093 | ± 137,771.491 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,005,846.389 | ± 96,703.066 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,700,100.197 | ± 144,301.351 | ops/s |
| readHeavy_8threads | thrpt | 10 | 1,321,714.189 | ± 346,860.821 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 321,688.060 | ± 95,079.876 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 549,510.063 | ± 160,634.881 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 826,926.873 | ± 149,437.015 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 1,026,862.457 | ± 520,060.221 | ops/s |

My scaling basically inverted with my lowest thrpt for both operations being at one thread and the highest being at > 2 threads

I then realized that I'd been using my virtual threads as my background worker threads which is a terrible idea and was probably contributing the high variance, so I decided to run with my background worker threads as platform threads instead

**Segmented Transaction Keeper**

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 543,983.955 | ± 133,134.608 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,020,562.096 | ± 131,954.153 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,632,501.353 | ± 197,194.523 | ops/s |
| readHeavy_8threads | thrpt | 10 | 1,444,733.689 | ± 652,337.205 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 335,078.800 | ± 128,993.334 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 580,449.127 | ± 110,305.596 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 864,819.719 | ± 159,350.383 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 1,120,983.469 | ± 783,465.405 | ops/s |

**Serial Transaction Keeper**

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 1,251,410.402 | ± 170,242.439 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,619,555.645 | ± 107,851.864 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,345,362.000 | ± 445,030.583 | ops/s |
| readHeavy_8threads | thrpt | 10 | 784,240.432 | ± 407,925.752 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 640,249.753 | ± 110,217.595 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 813,339.356 | ± 155,722.320 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 1,011,854.484 | ± 452,289.869 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 927,350.663 | ± 194,550.049 | ops/s |


Moving on from that minor hiccup, I decided to change my strategy for version tracking to use map based epoch tracking rather than a single writer background thread since I noticed it became a bottleneck, and the results improved significantly, but oddly, at 2 threads, the thrpt is very bad. No idea why this is happening.

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 2,191,449.356 | ± 405,088.856 | ops/s |
| readHeavy_2threads | thrpt | 10 | 195,198.411 | ± 37,480.037 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,733,823.527 | ± 526,463.941 | ops/s |
| readHeavy_8threads | thrpt | 10 | 2,832,819.684 | ± 791,310.719 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 898,844.327 | ± 184,090.618 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 58,322.618 | ± 6,505.121 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 118,490.002 | ± 21,500.755 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 5,211,745.483 | ± 672,226.456 | ops/s |

To find the issue, I looked at my profile data, and found `minVisibleEpoch()` as a hotspot at 2 threads. The current issue is we scan the entire map for the min active epoch, ideally this shouldn't take too long but as the map grows, could become a bottleneck

I then decided to use a navigable map for cheaper reads at the cost of writes being more expensive, and my data actually improved a lot, with stable variance across all thread counts

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 2,189,669.131 | ± 219,454.082 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,209,368.581 | ± 118,708.165 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,390,759.022 | ± 138,734.722 | ops/s |
| readHeavy_8threads | thrpt | 10 | 1,831,714.755 | ± 209,719.251 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 863,360.164 | ± 174,821.023 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 695,208.420 | ± 182,427.969 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 905,598.677 | ± 107,143.027 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 1,438,306.517 | ± 376,373.683 | ops/s |

I moved `minVisibleEpoch()` reads off the writer transaction path by caching and scheduling them in the GC thread at 100ms intervals. The tradeoff being precision as the GC is working off a slightly stale epoch, but it keeps the writer path clean.

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 2,426,304.887 | ± 259,332.240 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,138,292.934 | ± 71,983.527 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,297,350.055 | ± 173,150.260 | ops/s |
| readHeavy_8threads | thrpt | 10 | 1,581,116.656 | ± 142,358.242 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 831,511.799 | ± 272,854.879 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 651,383.441 | ± 125,000.038 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 864,550.813 | ± 102,841.781 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 1,133,963.803 | ± 371,369.242 | ops/s |


Now that reads for minimum active epochs were scheduled, cached and off the hotpath, I decided to move back to a normal concurrent hashmap and compare results

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 2,033,144.625 | ± 384,180.964 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,588,018.927 | ± 270,740.481 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,729,477.638 | ± 222,670.776 | ops/s |
| readHeavy_8threads | thrpt | 10 | 2,155,201.736 | ± 367,418.564 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 806,951.062 | ± 230,957.839 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 736,237.004 | ± 198,644.974 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 1,163,394.531 | ± 183,420.198 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 1,664,437.392 | ± 348,898.556 | ops/s |

After looking at my queue version chain, I realized calling `size()` to check version chain depth on each write txn was killing perf, since we had to scan the whole queue to find the size(even with the queue's dead nodes) and restart from the head if we reached a dead node, so I decided to use a long adder to track the approx size for cheaper `size()` calls

**Queue version chain**

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 2,290,892.594 | ± 491,972.988 | ops/s |
| readHeavy_2threads | thrpt | 10 | 1,604,327.428 | ± 208,550.678 | ops/s |
| readHeavy_4threads | thrpt | 10 | 1,891,341.883 | ± 250,730.085 | ops/s |
| readHeavy_8threads | thrpt | 10 | 1,937,383.908 | ± 501,990.327 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 932,853.659 | ± 165,703.963 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 966,232.277 | ± 279,759.248 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 1,135,401.769 | ± 380,147.279 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 1,620,440.108 | ± 446,644.708 | ops/s |

I decided to use my queue version chain as my default version in the next benchmark.

The hotspot across all benchmarks was now the `computeIfAbsent()` call whenever a transaction requested its current epoch (tBegin) at creation time. To reduce contention on this path, I built a thread-local epoch tracker, not a literal ThreadLocal, but a map keyed by thread ID instead of epoch. Each thread maps to the epoch of the transaction it's currently hosting, and resets to a sentinel value once the transaction ends, signaling it's no longer active.

This was a meaningful shift from the previous DefaultEpochTracker, which tracked all active transactions regardless of thread, making it more suitable for virtual threads but higher contention by nature. Since keys here can't actually be contested (each key belongs to exactly one thread), lock wait time dropped significantly and performance improves as a result. The tradeoff though is that it works best with a small, fixed pool of platform threads.

| Benchmark | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|
| readHeavy_1thread | thrpt | 10 | 2,887,534.127 | ± 554,342.591 | ops/s |
| readHeavy_2threads | thrpt | 10 | 2,366,272.494 | ± 324,415.185 | ops/s |
| readHeavy_4threads | thrpt | 10 | 2,162,652.894 | ± 276,947.370 | ops/s |
| readHeavy_8threads | thrpt | 10 | 2,449,227.477 | ± 443,072.582 | ops/s |
| writeHeavy_1thread | thrpt | 10 | 1,271,287.026 | ± 498,230.994 | ops/s |
| writeHeavy_2threads | thrpt | 10 | 1,578,855.855 | ± 213,555.495 | ops/s |
| writeHeavy_4threads | thrpt | 10 | 1,340,979.654 | ± 316,184.797 | ops/s |
| writeHeavy_8threads | thrpt | 10 | 1,519,457.062 | ± 700,083.591 | ops/s |

The variance was actually worse in some scenarios, sometimes >30% of the actual score. Rerunning these benchmarks, I noticed something odd looking at my profile data for memory allocation, a lot of memory was getting allocated but barely cleaned up by the GC while running my benchmarks, leading to high variance and my numbers tanking in unusual ways during benchmarking. The profile data showed most of this occurred when I started a transaction which wasn't too helpful. So I decided to look at my actual memory usage when running these benchmarks and I noticed around 95% of my memory was being used while running these benchmarks.

My first suspect were my version chains, since stale versions might not be cleaned up by the GC. I decided to write a simple unit test(no idea why I didn't test my version chains earlier) to see if versions were being cleaned up, and they actually were not. The issue lied in a simple check I added earlier to prevent redundant O(N) lookups

```java
    @Override
public void removeUnreachableVersions(long tBegin) {
    if (tBegin <= minVisibleEpoch.epoch) return; //This simple line here, the issue was that minVisibleEpoch was always initialized as Long.MAX_VALUE, even if the epoch could be updated while non-visible version were getting pruned, the GC would never actually get the chance to prune those versions, because beginTs(seen epoch) would always be less than Long.MAX_VALUE

    minVisibleEpoch.reset(); //Reset the holder to Long.MAX_VALUE, to prevent a situation where we are sitting on an older end ts, from a version pruned a while back
    var ls = this.latest;
    Set<Map.Entry<Long, Version<E>>> set = versionMap.entrySet();

    set.removeIf(entry -> {
        var val = entry.getValue();
        boolean shouldRemove = val.endTs < tBegin  && val != ls;

        if (!shouldRemove && val.endTs < minVisibleEpoch.epoch) minVisibleEpoch.epoch = val.endTs;
        return shouldRemove;
    });
}
```

This was changed to
```java
if (minVisibleEpoch.epoch != Long.MAX_VALUE && tBegin <= minVisibleEpoch.epoch) return; //Actually gave the gc a chance to prune the older versions
```

After this, my numbers improved significantly and variance reduced a bit

| Benchmark | Version Chain | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|---|
| readHeavy_1thread | queue | thrpt | 10 | 3,022,274.294 | ± 397,643.609 | ops/s |
| readHeavy_1thread | nav | thrpt | 10 | 2,250,631.595 | ± 201,546.981 | ops/s |
| readHeavy_2threads | queue | thrpt | 10 | 2,583,840.733 | ± 444,871.511 | ops/s |
| readHeavy_2threads | nav | thrpt | 10 | 2,291,496.019 | ± 304,662.994 | ops/s |
| readHeavy_4threads | queue | thrpt | 10 | 3,152,546.075 | ± 350,295.696 | ops/s |
| readHeavy_4threads | nav | thrpt | 10 | 2,933,701.271 | ± 210,834.860 | ops/s |
| readHeavy_8threads | queue | thrpt | 10 | 4,209,719.728 | ± 611,222.951 | ops/s |
| readHeavy_8threads | nav | thrpt | 10 | 4,058,722.339 | ± 265,986.981 | ops/s |
| writeHeavy_1thread | queue | thrpt | 10 | 2,467,899.289 | ± 343,129.882 | ops/s |
| writeHeavy_1thread | nav | thrpt | 10 | 1,224,346.814 | ± 276,528.783 | ops/s |
| writeHeavy_2threads | queue | thrpt | 10 | 1,987,672.075 | ± 142,924.001 | ops/s |
| writeHeavy_2threads | nav | thrpt | 10 | 1,474,317.537 | ± 332,782.403 | ops/s |
| writeHeavy_4threads | queue | thrpt | 10 | 2,540,089.406 | ± 107,245.569 | ops/s |
| writeHeavy_4threads | nav | thrpt | 10 | 1,941,961.678 | ± 335,538.070 | ops/s |
| writeHeavy_8threads | queue | thrpt | 10 | 3,310,156.822 | ± 319,971.721 | ops/s |
| writeHeavy_8threads | nav | thrpt | 10 | 3,599,563.143 | ± 712,225.130 | ops/s |

I was still a bit skeptical about the variance, even though it was pretty reasonable, I realized a lot of memory was getting allocated in my thread local epoch tracker under contention due to `long` boxing when updating epoch values, so I decided to try using **fast-utils** synchronized `Long2LongHashMap`, to prevent boxing and allocations under high contention. Rerunning the benchmarks again, memory allocation on that hotpath dropped to basically zero, and writes under contention did suffer a bit, though variance for both write and read heavy workloads were pretty good

| Benchmark | Version Chain | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|---|
| readHeavy_1thread | queue | thrpt | 10 | 3,917,121.556 | ± 191,182.041 | ops/s |
| readHeavy_1thread | nav | thrpt | 10 | 2,592,747.908 | ± 192,348.108 | ops/s |
| readHeavy_2threads | queue | thrpt | 10 | 2,935,324.852 | ± 380,159.702 | ops/s |
| readHeavy_2threads | nav | thrpt | 10 | 2,587,901.962 | ± 253,365.744 | ops/s |
| readHeavy_4threads | queue | thrpt | 10 | 2,410,564.690 | ± 394,036.694 | ops/s |
| readHeavy_4threads | nav | thrpt | 10 | 2,425,199.868 | ± 109,061.792 | ops/s |
| readHeavy_8threads | queue | thrpt | 10 | 2,138,869.830 | ± 106,605.873 | ops/s |
| readHeavy_8threads | nav | thrpt | 10 | 2,006,265.789 | ± 154,944.074 | ops/s |
| writeHeavy_1thread | queue | thrpt | 10 | 2,789,016.664 | ± 258,145.402 | ops/s |
| writeHeavy_1thread | nav | thrpt | 10 | 1,247,223.769 | ± 212,243.516 | ops/s |
| writeHeavy_2threads | queue | thrpt | 10 | 2,308,032.198 | ± 234,208.698 | ops/s |
| writeHeavy_2threads | nav | thrpt | 10 | 1,395,297.149 | ± 187,661.268 | ops/s |
| writeHeavy_4threads | queue | thrpt | 10 | 2,285,734.168 | ± 222,636.258 | ops/s |
| writeHeavy_4threads | nav | thrpt | 10 | 1,738,846.900 | ± 503,307.766 | ops/s |
| writeHeavy_8threads | queue | thrpt | 10 | 2,274,919.824 | ± 46,946.253 | ops/s |
| writeHeavy_8threads | nav | thrpt | 10 | 1,893,349.970 | ± 213,244.636 | ops/s |

The thrpt was great, though after some research I found out about a generic trick, using primitive arrays as generic types, so instead of boxed long values. I decided to try this out with CHM and compare it to the serialized long2long version
```java
ConcurrentMap<Long, Long> map //Instead of this, we could do
ConcurrentMap<Long, long[]> map //No boxing for values
```

| Benchmark | Version Chain | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|---|
| readHeavy_1thread | queue | thrpt | 10 | 3,921,045.464 | ± 379,095.211 | ops/s |
| readHeavy_1thread | nav | thrpt | 10 | 2,490,400.560 | ± 235,824.152 | ops/s |
| readHeavy_2threads | queue | thrpt | 10 | 3,351,486.812 | ± 330,871.332 | ops/s |
| readHeavy_2threads | nav | thrpt | 10 | 2,961,814.543 | ± 271,271.445 | ops/s |
| readHeavy_4threads | queue | thrpt | 10 | 4,130,101.194 | ± 365,901.882 | ops/s |
| readHeavy_4threads | nav | thrpt | 10 | 3,967,520.781 | ± 267,639.783 | ops/s |
| readHeavy_8threads | queue | thrpt | 10 | 6,100,773.757 | ± 555,229.766 | ops/s |
| readHeavy_8threads | nav | thrpt | 10 | 5,384,214.753 | ± 396,929.168 | ops/s |
| writeHeavy_1thread | queue | thrpt | 10 | 2,948,891.898 | ± 461,806.810 | ops/s |
| writeHeavy_1thread | nav | thrpt | 10 | 1,256,910.691 | ± 203,279.439 | ops/s |
| writeHeavy_2threads | queue | thrpt | 10 | 2,523,596.142 | ± 196,569.443 | ops/s |
| writeHeavy_2threads | nav | thrpt | 10 | 1,408,471.703 | ± 192,463.826 | ops/s |
| writeHeavy_4threads | queue | thrpt | 10 | 2,943,223.429 | ± 276,821.883 | ops/s |
| writeHeavy_4threads | nav | thrpt | 10 | 2,636,191.583 | ± 311,031.690 | ops/s |
| writeHeavy_8threads | queue | thrpt | 10 | 4,064,740.729 | ± 416,730.295 | ops/s |
| writeHeavy_8threads | nav | thrpt | 10 | 4,574,077.728 | ± 549,287.881 | ops/s |


Since, I've been testing thrpt for my mvcc map for "best case scenarios" i.e. no retries on aborts. I decided to test with retries on abort. Note that this is base-lined against my map with a `Long2ArrayEpochTracker`

| Benchmark | Version Chain | Mode | Cnt | Score | Error | Units |
|---|---|---|---|---|---|---|
| readHeavy_1thread | queue | thrpt | 10 | 4,255,294.022 | ± 417,518.780 | ops/s |
| readHeavy_1thread | nav | thrpt | 10 | 3,026,195.253 | ± 382,190.857 | ops/s |
| readHeavy_2threads | queue | thrpt | 10 | 3,727,360.460 | ± 469,491.078 | ops/s |
| readHeavy_2threads | nav | thrpt | 10 | 3,310,257.104 | ± 195,179.750 | ops/s |
| readHeavy_4threads | queue | thrpt | 10 | 4,222,921.041 | ± 334,896.201 | ops/s |
| readHeavy_4threads | nav | thrpt | 10 | 3,352,625.744 | ± 481,297.585 | ops/s |
| readHeavy_8threads | queue | thrpt | 10 | 4,512,409.390 | ± 385,141.320 | ops/s |
| readHeavy_8threads | nav | thrpt | 10 | 3,414,450.281 | ± 391,240.406 | ops/s |
| writeHeavy_1thread | queue | thrpt | 10 | 2,887,463.431 | ± 507,767.083 | ops/s |
| writeHeavy_1thread | nav | thrpt | 10 | 1,330,658.185 | ± 214,850.891 | ops/s |
| writeHeavy_2threads | queue | thrpt | 10 | 2,083,535.454 | ± 214,987.876 | ops/s |
| writeHeavy_2threads | nav | thrpt | 10 | 1,192,807.369 | ± 222,511.666 | ops/s |
| writeHeavy_4threads | queue | thrpt | 10 | 2,093,721.029 | ± 118,319.047 | ops/s |
| writeHeavy_4threads | nav | thrpt | 10 | 1,337,540.905 | ± 198,100.953 | ops/s |
| writeHeavy_8threads | queue | thrpt | 10 | 1,800,086.692 | ± 403,405.040 | ops/s |
| writeHeavy_8threads | nav | thrpt | 10 | 1,028,280.423 | ± 285,679.830 | ops/s |

To fully understand this drop on the write heavy bench I compared profile data from this benchmark to those without retries. While everything looked 'similarish' on the CPU side, memory was a different story with memory usage spiking up from ~16GB to almost ~30GB at every iteration. Due to the frequency of aborts, for each retry, a new transaction object had to be created, meaning more memory allocated for the transaction object, its read/write operation objects and its completable values, hence more pressure on the GC (Java's GC), more GC pauses, lower thrpt and higher variance.


## Conclusions
The `QueueVersionChain` paired with `Long2ArrayEpochTracker` ended up being the best all round configuration, reads comfortably in the multi-million ops/s range across thread counts, writes improved significantly from the initial ~2k. A few things are still unresolved though, the 2-thread throughput anomaly with map-based epoch tracking being the main one, and the retry memory pressure under high abort rates is worth revisiting. I'd likely make transaction objects reusable to reduce GC pressure on retries.

However, if you want to check out the mvcc implementation, benchmark code, or my mvcc map benchmarks under more realistic workloads i.e. Zipfian benchmarks. You can visit the **GitHub** repository: https://github.com/kusoroadeolu/tx-map/tree/mvcc-txmap