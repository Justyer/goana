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