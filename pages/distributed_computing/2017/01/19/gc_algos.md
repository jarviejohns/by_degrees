---
layout: page
title: Garbage Collection (Algorithms)
---

[Basics](https://jarviejohns.github.io/by_degrees/pages/distributed_computing/2017/01/19/garbage_collection.html) 
[Algorithms](https://jarviejohns.github.io/by_degrees/pages/distributed_computing/2017/01/19/gc_algos.html) 

In general, a Garbage collector performs 2 main things - 
1. Takes a census of live objects (Marking)
2. Get rid of rest i.e garbage (dead and unused objects)


## Marking
* GC traverses the whole object graph in memory and marks objects that are alive starting with GC roots and following references from roots to other objects. Every object GC visits is *marked* alive.
* The applicaton is suspended/stopped for marking to happen i.e the app reaches a safe point
* Duration of the pause does NOT depend on the "total number of objects in heap" or "size of heap" but on the "number of alive objects". So increasing the size of heap does not directly affect the duration of marking phase.
* When Mark phase is completed, GC starts removing the unreachable objects

## Removing Unused objects
Basically consists of 3 steps 
1. Sweeping
2. Compacting
3. Copying

### Sweep
* Mark-Sweep algo naive GCs, where after mark step, all unvisited objects are considered free and available for reallocation for new objects. 
* However, requires a free-list registry recording every free region and size
Dis 
1. Overhead in maintaining this free-list registry
2. Plenty of free regions but not big enough to accomodate certain allocation leading to allocation errors

### Compact 
* Mark-Sweep-Compact algo designed to overcome deficiencies of Mark-Sweep algo.
* After sweep, there is compaction of free spaces, with alive objects to the beginning of the memory region
* Adv : Cheap allocation via pointer bumping; no fragmentation issue
* Dis : Extra GC pause duration due to reassigning (copying) of live objects to new memory regions and updating their references.

### Copy
* Similar to Mark and Compact
* Difference is relocation happens to different memory region
* Adv : Mark and Copy can occur simultaneously
* Dis : Need one more memory region and large to hold survived objects


## Implementations
Any implementation will need two different GC algos - one for cleaning YoungGen and the other for cleaning OldGen/Tenured

*Enable GC logging via*
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps


Young | Tenured | JVM options | Comments 
------|---------|-------------|----------
Incremental| Incremental | -Xincgc | 
**Serial**| **Serial** | **-XX:+UseSerialGC** | **Serial GC for both Gens**
Parallel Scavenge | Serial | -XX:+UseParallelGC -XX:-UseParallelOldGC | 
**Parallel Scavenge** | **Parallel Old** | **-XX:+UseParallelGC -XX:+UserParallelOldGC** | **Parallel GC for both Gens**
Serial | CMS | -XX:-UseParNewGC -XX:+UseConcMarkSweepGC | 
**Parallel New** | **CMS** | **-XX:+UseParNewGC -XX:+UserConcMarkSweepGC** | **Parallel new for Young and CMS for Old**
**G1** | | **-XX:+UseG1GC** | **G1 collection for both Young and Old Gens**

## Serial GC
* **Young Gen** : Mark-Copy (STW)
* **Old Gen**   : Mark-Sweep-Compact (STW)
* Single thread collectors; cannot parallelize
* Cannot take advantage of multiple cores on a machine i.e only one core is used by JVM for Garbage collection

2015-05-26T14:45:37.987-0200: 151.126: [GC (Allocation Failure) 151.126: [DefNew: 629119K->69888K(629120K), 0.0584157 secs] 1619346K->1273247K(2027264K), 0.0585007 secs] [Times: user=0.06 sys=0.00, real=0.06 secs]
2015-05-26T14:45:59.690-0200: 172.829: [GC (Allocation Failure) 172.829: [DefNew: 629120K->629120K(629120K), 0.0000372 secs]172.829: [Tenured: 1203359K->755802K(1398144K), 0.1855567 secs] 1832479K->755802K(2027264K), [Metaspace: 6741K->6741K(1056768K)], 0.1856954 secs] [Times: user=0.18 sys=0.00, real=0.18 secs]

The first one is a Minor GC, cleaning the YoungGen
The last one is a Full GC, cleaning the entire heap including metaspace

### Minor GC
**2015-05-26T14:45:37.987-0200** - Time when GC event started
**151.126** - Time when GC event started relative to JVM start up time; measured in seconds
**GC** - Flag to distinguish minor vs Full. Here it is minor
**Allocation Failure** - Cause of collection; GC is triggered due to object unable to fit in any region in Young Gen.
**DefNew** - Name of the single-threaded Mark-Copy STW GC to clean YoungGen
**629119K->69888K** - Before and After usage of YoungGen
**(629120K)** - Total size of YoungGen
**1619346K->1273247K** - Before and After usage of Entire Heap
**(2027264K)** - Total available Heap
**0.0585007 secs** - Duration of GC event
**user=0.06 sys=0.00, real=0.06 secs** - User time is CPU time consumed; sys time is time spent in OS calls or waiting; real time is clock time for which application was stopped

So, what does this mean?
a. Out of 2G of total heap, 1.6G of the heap size was used prior to collection.
b. Out of this 1.6G, 629MB was used by Young Gen. This means Old Gen (long term survivors) took about 990MB of the heap.
c. After the GC, Young Gen decreased by 559MB; but the total available heap i.e heap usage decreased only by 346MB. 
This means - 
* (Reduction in YoungGen) - (Reduction in Heap) = (Size of objects moved from YoungGen to OldGen)
i.e 559M - 346M = 213M of objected were promoted.

### Full GC
**172.829** - Time when GC started; in this case 20s after minor GC.
**629120K->629120K(629120K)** - Minor GC reduced Young 
**Tenured** - Name of GC to clean OldGen; single Threaded Mark-Sweep-Compact STW GC
**1203359K->755802K** - Before and after OldGen
**1398144K** - Total capacity of OldGen
**0.1855567 secs** - Time to clean OldGen
**832479K->755802K** -  Heap before and after
**2027264K** - Total heap
**[Metaspace: 6741K->6741K(1056768K)], 0.1856954 secs]** - No GC collected

## Parallel GC
* **Young Gen** : Mark-Copy (STW)
* **Old Gen**   : Mark-Sweep-Compact (STW)
* Multi thread collectors; parallelize and reduction in collection times; increases throughput since all cores are involved in collecting garbage during GC and thus shorter pauses. Between collecting, no collectors consume any resources.
* Number of threads defined 
-XX:+ParallelGCThreads=NNN; default value is equal to number of cores in the machine

2015-05-26T14:27:40.915-0200: 116.115: [GC (Allocation Failure) [PSYoungGen: 2694440K->1305132K(2796544K)] 9556775K->8438926K(11185152K), 0.2406675 secs] [Times: user=1.77 sys=0.01, real=0.24 secs]
2015-05-26T14:27:41.155-0200: 116.356: [Full GC (Ergonomics) [PSYoungGen: 1305132K->0K(2796544K)] [ParOldGen: 7133794K->6597672K(8388608K)] 8438926K->6597672K(11185152K), [Metaspace: 6745K->6745K(1056768K)], 0.9158801 secs] [Times: user=4.49 sys=0.64, real=0.92 secs]

### Minor GC
**PSYoungGen** - parallel, multithreaded Mark-Copy STW GC to clean YoungGen

So, what happened here?
a. Before GC, 2.6G of 2.8G was used by YoungGen. 9.5 G of 11.1 G of Total Heap utilized. This means, out of 8.38 G for OldGen, the latter has objects that occupies 6.82 G.
Existing OldGen Used = Existing Heap Used - Existing YoungGen Used
Total OldGen Available = Total Heap - Total YoungGen
b. After GC, 1.38G of YoungGen released. Heap reduced by 1.1G. Therefore, size of objects moved from YoungGen to OldGen :
1.1 - 1.38 = 271 MB

### Full GC
**Full GC** - Full GC commencing
**1305132K->0K(2796544K)** - Typical of full GC, whole YoungGen was flushed
**ParOldGen** - Parallel Mark-Sweep-Compact STW GC to clean OldGen

1.8G of OldGen collected

## Concurrent Mark and Sweep
* **Young Gen** : Parallel Mark-Copy (STW)
* **Old Gen**   : Concurrent Mark-Sweep 
* This collector was designed to avoid long pauses while collecting in the Old Generation. It achieves this by two means:
** No compaction; uses free list registry to reclaim space
** Jobs tied to Mark-Sweep occurs concurrently with application i.e no explicit STW. By Default the number of threads used by this GC = 1/4 of total number of physical cores on machine.
** Used where latency is critical i.e, more responsive app
** Since some CPU resources are doing concurrent mark and sweep, the throughput is less than Parallel GC

2015-05-26T16:23:07.219-0200: 64.322: [GC (Allocation Failure) 64.322: [ParNew: 613404K->68068K(613440K), 0.1020465 secs] 10885349K->10880154K(12514816K), 0.1021309 secs] [Times: user=0.78 sys=0.01, real=0.11 secs]
2015-05-26T16:23:07.321-0200: 64.425: [GC (CMS Initial Mark) [1 CMS-initial-mark: 10812086K(11901376K)] 10887844K(12514816K), 0.0001997 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2015-05-26T16:23:07.321-0200: 64.425: [CMS-concurrent-mark-start]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-mark: 0.035/0.035 secs] [Times: user=0.07 sys=0.00, real=0.03 secs]
2015-05-26T16:23:07.357-0200: 64.460: [CMS-concurrent-preclean-start]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-preclean: 0.016/0.016 secs] [Times: user=0.02 sys=0.00, real=0.02 secs]
2015-05-26T16:23:07.373-0200: 64.476: [CMS-concurrent-abortable-preclean-start]
2015-05-26T16:23:08.446-0200: 65.550: [CMS-concurrent-abortable-preclean: 0.167/1.074 secs] [Times: user=0.20 sys=0.00, real=1.07 secs]
2015-05-26T16:23:08.447-0200: 65.550: [GC (CMS Final Remark) [YG occupancy: 387920 K (613440 K)]65.550: [Rescan (parallel) , 0.0085125 secs]65.559: [weak refs processing, 0.0000243 secs]65.559: [class unloading, 0.0013120 secs]65.560: [scrub symbol table, 0.0008345 secs]65.561: [scrub string table, 0.0001759 secs][1 CMS-remark: 10812086K(11901376K)] 11200006K(12514816K), 0.0110730 secs] [Times: user=0.06 sys=0.00, real=0.01 secs]
2015-05-26T16:23:08.458-0200: 65.561: [CMS-concurrent-sweep-start]
2015-05-26T16:23:08.485-0200: 65.588: [CMS-concurrent-sweep: 0.027/0.027 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
2015-05-26T16:23:08.485-0200: 65.589: [CMS-concurrent-reset-start]
2015-05-26T16:23:08.497-0200: 65.601: [CMS-concurrent-reset: 0.012/0.012 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]

### Minor GC
**ParNew** - Parallel Mark-Sweep STW GC for YoungGen; designed to work in conjunction with CMS collector for OldGen
**613404K->68068K(613440K)** - Before and After usage of YoungGen
**0.1020465 secs** - Duration for marking
** 10885349K->10880154K(12514816K)** - Before and After usage of Heap
**0.1021309 secs** - The time it took for the garbage collector to mark and copy live objects in the Young Generation. This includes communication overhead with ConcurrentMarkSweep collector, promotion of objects that are old enough to the Old Generation and some final cleanup at the end of the garbage collection cycle.

What does this mean?
a. Before GC, YoungGen share was 613M. Heap was at 10.8G of 12.5G. This means, OldGen share was 10.2G
b. After GC, YoungGen decreased by 545M. Heap decreased by 5.1M. So, size of YoungGen promoted to OldGen :
(5.1M)-(545M) = 540M


### Full GC
In real world situation Minor Garbage Collections of the Young Generation can occur anytime during concurrent collecting the Old Generation. The major collection events will be interleaved with the minor GC events.

#### Phase 1 : Initial Mark 
* one of the two STW events during CMS
* marks all objects in OldGen that are either referenced directly from GC roots or from live objects in YoungGen

#### Phase 2 : Concurrent Mark
* concurrently marks all live objects starting from roots found in Phase 1

#### Phase 3 : Concurrent Preclean
* While Phase 2 was running, some of the objects were mutated. When that happens, JVM marks that area of heap that contains the dirty object -> Dirty Card
* Objects reachable from them are also marked

#### Phase 4 : Concurrent Abortable Preclean
* Step to prepare for next STW collection

#### Phase 5 : Final Remark
* STW event
* Finalize marking of all live objects since the previous concurrent phases may not have kept up with marking of all live objects
**387920 K (613440 K)** - Current occupancy and capacity of YoungGen
**10812086K(11901376K)** - Current occupancy and capacity of OldGen after this phase
**11200006K(12514816K)** - Current occupancy and capacity of Heap after this phase

#### Phase 6 : Concurrent Sweep
* Removes the unused objects to reclaim space

#### Phase 7 : Concurrent Reset
* resets the inner data structures of CMS algos and prepares for next cycle

## G1 - Garbage First































