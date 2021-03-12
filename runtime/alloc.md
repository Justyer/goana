# 内存分配

## 内存分配器
基于google的tcmalloc，但在其基础上又做了改进。
主分配器基于页(page)来管理，小对象分配器的尺寸不超过32KB，有70种class

分配器数据结构：
- fixalloc：固定尺寸的堆外内存的自由表分配器
- mheap：内存分配的堆，以一页8KB为粒度进行管理
- mspan：mheap用其管理正在使用的页
- mcentral：收集给定尺寸的class的所有span
- mcache：空闲空间的mspan列表的缓存
- mstats：分配器统计相关

小对象分配内存：
1. 四舍五入到一个小尺寸class，在这个p的mcache中寻找对应的mspan，扫描这个mspan的空闲的bitmap，找到一个空槽点，如果存在则分配内存，否则执行第二步。这个操作全过程不需要加锁
2. 如果没有空闲的槽点，就从mcentral的mspan列表中获取一个所需尺寸的空闲的新的mspan。这中操作包含锁住mcentral整个span的开销
3. 如果mcentral中没有这种空闲的span了，就从堆上获得
4. 如果堆上为空或者没有页能运行足够大的空间，就从os中申请至少1MB的内存，分配一组新的页。这种分配在和os沟通时会有巨大的开销

清扫mspan和释放对象提升的层次差不多：
1. 如果mspan在响应分配器时被清扫，它返回到mcache以满足分配
2. 否则，如果mspan仍然要在其上分配对象，则根据mspan的大小把它放在mcentral空闲列表
3. 否则，如果mspan中的所有对象都空闲，mspan的页返回mheap并且mspan当场死亡

分配和释放大对象直接使用mheap，跳过mcache和mcentral
1. 栈帧分配器可以避免完全归零
2. 它表现出更好的时间局部性，因为程序可能集中写入内存
3. 我们不会把用不重用的page归零

虚拟内存布局：
- 堆由arena集合组成，在64位系统中为64MB，32位为4MB，每个arena的起始地址与其尺寸对齐。
- 每个arena有一个关联的heapArena对象用来存储arena的元信息：heap的bitmap所有在arena中的字和arena中所有页的span map。heapArena对象它们自己分配堆外内存。
- 自此arena都是对齐的，地址空间可以作为连续的arena帧来被查看。arena map对应arena帧数字到*heapArena或者nil，nil表示Go堆地址空间不支持的部分。arena map是一个由L1 arena map和许多L2 arena map组成的二级数组结构；然而，因为arena太大了，所以在很多架构中是由简单的巨大的L2 map组成。
- arena map包含整个地址空间，允许Go堆使用地址空间的任何一部分。分配器试图保持arena连续以至于巨大的span（因此大对象）可以穿过arena。

> 以上翻译自runtime/malloc.go

## mheap

```go
type mheap struct {
    // 锁必须只能从系统栈获得，否则当被扩容的栈持有时会发生死锁。
	lock      mutex
	pages     pageAlloc // 页分配器的数据结构

    // allspans是一个所有m曾经创建过的span切片。每个mspan恰好出现一次
    // 它的所有span的内存通常在这里管理，可以像堆增长一样重新分配和移动。
    // 通常情况下，它被mheap_.lock保护，阻止当前访问、释放备用存储。
    // 在STW期间访问可能不会加锁，但是必须保证内存分配在访问期间不发生
    // （因为可能释放备用存储）。
	allspans []*mspan // all spans out there

	// scavengeGoal is the amount of total retained heap memory (measured by
	// heapRetained) that the runtime will try to maintain by returning memory
	// to the OS.
	scavengeGoal uint64

	// Page reclaimer state

	// reclaimIndex is the page index in allArenas of next page to
	// reclaim. Specifically, it refers to page (i %
	// pagesPerArena) of arena allArenas[i / pagesPerArena].
	//
	// If this is >= 1<<63, the page reclaimer is done scanning
	// the page marks.
	//
	// This is accessed atomically.
	reclaimIndex uint64
	// reclaimCredit is spare credit for extra pages swept. Since
	// the page reclaimer works in large chunks, it may reclaim
	// more than requested. Any spare pages released go to this
	// credit pool.
	//
	// This is accessed atomically.
	reclaimCredit uintptr

	// arenas is the heap arena map. It points to the metadata for
	// the heap for every arena frame of the entire usable virtual
	// address space.
	//
	// Use arenaIndex to compute indexes into this array.
	//
	// For regions of the address space that are not backed by the
	// Go heap, the arena map contains nil.
	//
	// Modifications are protected by mheap_.lock. Reads can be
	// performed without locking; however, a given entry can
	// transition from nil to non-nil at any time when the lock
	// isn't held. (Entries never transitions back to nil.)
	//
	// In general, this is a two-level mapping consisting of an L1
	// map and possibly many L2 maps. This saves space when there
	// are a huge number of arena frames. However, on many
	// platforms (even 64-bit), arenaL1Bits is 0, making this
	// effectively a single-level map. In this case, arenas[0]
	// will never be nil.
	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	// arenaHints is a list of addresses at which to attempt to
	// add more heap arenas. This is initially populated with a
	// set of general hint addresses, and grown with the bounds of
	// actual heap arena ranges.
	arenaHints *arenaHint

	// allArenas is the arenaIndex of every mapped arena. This can
	// be used to iterate through the address space.
	//
	// Access is protected by mheap_.lock. However, since this is
	// append-only and old backing arrays are never freed, it is
	// safe to acquire mheap_.lock, copy the slice header, and
	// then release mheap_.lock.
	allArenas []arenaIdx

	// sweepArenas is a snapshot of allArenas taken at the
	// beginning of the sweep cycle. This can be read safely by
	// simply blocking GC (by disabling preemption).
	sweepArenas []arenaIdx

	// markArenas is a snapshot of allArenas taken at the beginning
	// of the mark cycle. Because allArenas is append-only, neither
	// this slice nor its contents will change during the mark, so
	// it can be read safely.
	markArenas []arenaIdx

	// curArena is the arena that the heap is currently growing
	// into. This should always be physPageSize-aligned.
	curArena struct {
		base, end uintptr
	}

	_ uint32 // ensure 64-bit alignment of central

    // central 小尺寸class的空闲列表
	// the padding makes sure that the mcentrals are
	// spaced CacheLinePadSize bytes apart, so that each mcentral.lock
	// gets its own cache line.
	// central is indexed by spanClass.
    // 每一种class对应一个
	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	}

	spanalloc             fixalloc // allocator for span*
	cachealloc            fixalloc // allocator for mcache*
	specialfinalizeralloc fixalloc // allocator for specialfinalizer*
	specialprofilealloc   fixalloc // allocator for specialprofile*
	speciallock           mutex    // lock for special record allocators.
	arenaHintAlloc        fixalloc // allocator for arenaHints

	unused *specialfinalizer // never set, just here to force the specialfinalizer type into DWARF
}
```