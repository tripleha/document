<h1 style="text-align:center"> Log-Structured Merge Trees </h1>

## 一、LSM-Tree 基础设计总结

- ### LSM-Tree 的出现以及想要解决的问题
    随着硬件技术的进步， CPU 和内存技术都得到了不小的发展，单机情况下计算力总是富裕的，并且RAM也越来越大，允许我们将大部分热数据缓存在内存中。但是对于磁盘等外存而言，随机I/O的效率仍然特别低下，尤其是随机写的效率(与顺序写效率差千倍)。即使是 SSD ，虽然顺序写和随机写的性能差距不像磁盘那么大(差距十倍)，但也有足够的优化空间(随着 SSD 的普及 LSM-Tree 在 SSD 上的优化也是当前的一个热点)。
    1991年加利福尼亚大学伯克利分校的一篇论文《The Design and Implementation of a Log-Structured File System》提出了一种将多次随机写转化为一次顺序写的 append only log 的磁盘写思路。
    在1996年O'Neil等人的论文《The Log-Structured Merge-Tree (LSM-Tree)》正式提出了 LSM-Tree 数据结构。
    2006年Google发表的 BigTable 文章引起了人们对于 LSM-Tree 的广泛关注，各种使用 LSM-Tree 的存储引擎相继出现。
    LSM-Tree 是 tradeoff 思维的一种体现。通过牺牲一些读性能，增加一定的数据冗余，换取写性能的巨大提升。后续的 LSM-Tree 的相关优化也有很多是基于这样的思维。 tradeoff 源自于需求，对数据库的需求实际上就是不同的数据，不同的 workload 。更具体地说， LSM-Tree 的 tradeoff 对于读写均衡甚至读少写多的 workload 是非常值得的。

- ### LSM-Tree 基本设计思路
    在该算法提出时，算法中描述的数据集合是按照树结构组织的。LSM-Tree 实际上是将一颗大树拆分管理，优化写效率。而发展到今天，LSM-Tree 已经不再是严格意义上的树结构了。拆分管理的一个个小数据集合是一个个有序 kv 对集合，我们这里称之为 table ，对于 table 所使用数据结构的基本要求是能够快速有序地输出集合内数据， B+树 、 skiplist 都是不错的选择。
    **LSM-Tree 基本概念:**
    - **run**
        run 是 LSM-Tree 的数据管理的核心概念，表示的是一组不带重复key值的数据。实际上是一组没有重叠部分的 table 。run 和 LSM-Tree 的版本控制息息相关。 LSM-Tree 允许数据冗余，通过数据的版本信息得知数据是否是最新的。一般 runs 都是按顺序写入到外存中的，所以内存中的数据往往是最新的，后持久化的 table 中的数据总是新于新持久化的数据。所以数据查找顺序往往是先查找内存中的新 run 的 memtable 是否包含，然后是还未持久化的旧 run ，然后根据持久化的时间从较新的 run 的 sstable 中找。对于某个 key 值的查找，在一个 run 里面只需要查找一次便可以确定。一般根据版本逆序查找，找到最高版本的一个数据便可以停止了。
    - **memtable**
        memtable 是在内存中可以执行数据写操作的 table ，一般存在 memtable 中的数据为热写数据(刚被插入或者更新)。对于存在于 memtable 中的热写数据的更新操作，可以直接采用快速的 in-place update 。
    - **immutable memtable**
        是只读的 memtable ，不可再对其执行写操作。因为内存无法装下所有数据，所以要在内存使用达到阈值之后将 memtable 持久化到磁盘。immutable memtable 是 memtable 持久化到外存变为 sstable 的一个中间状态。持久化时 memtable 不接受用户的写操作，以免操作丢失。
    - **WAL (write-ahead logging)**
        对于 LSM-Tree 实现的存储引擎而言，数据的写入不会立刻持久化到磁盘，热数据会暂存在 memtable 中直到内存使用达到阈值，而一旦发生宕机，则 memtable 中未持久化的所有数据及更新都将丢失。为了防止上述情况会采用 WAL 的机制。所有写操作都会先写入到 WAL 中， WAL 的写入是直接持久化到磁盘的，所以操作成功写入 WAL 后就不会因为宕机而丢失了。 WAL 是顺序写入顺序读出的，所以读写速度都有保证。后台线程会根据 WAL 对 memtable 进行操作，最终 memtable 就可以体现用户操作的结果了。而一旦宕机，又可以根据 WAL 快速重建起 memtable 的内容。 WAL 大小有阈值，一般持久化到磁盘后的数据对应的操作记录就可以丢弃了。 WAL 的设计原则是未持久化但是已经提交的操作记录必须已经被持久化到日志中。
        - bulk write
            WAL 性能优化的一种方式。WAL 顺序写入，所有数据写操作都需要写入，所以写 WAL 很容易成为并发写入的性能瓶颈，而 bulk write 优化是将一小段时间内的 log 收集起来一起写入，这个时间间隔非常短，因为如果太长会影响用户体验，一般会规定一个时间间隔和缓存阈值，超过其一则写入 WAL 文件。这个优化减少了写 WAL 文件的I/O调用，对于连续插入或者连续更新操作的优化非常明显。当记录还在 WAL 的缓冲区中时是不可以返回的，必须等到确实持久化了以后才能算操作成功(可以选择等待 table 数据确实更改操作成功再返回，在大部分系统中具体的程度是可以由参数控制的)。
    - **sstable (sorted string table)**
        存在于外存中的 table ，会按照一定的方式组织，并在一定条件下自动触发合并(compaction 过程)以优化外存数据的读取效率。sstable 有序存储 kv 对，并且是只读的。
    - **与传统数据库不同的 update 和 delete**
        对于 update 操作，若需要修改的数据在 sstable 中找到，则会先将该数据拷贝到内存中，再进行修改，修改后的数据并不会回写到 sstable 原来的位置，而是插入到 memtable 中。修改后的数据版本信息变高了，这样表明该数据是最新的，比其版本信息低的冗余数据会在后续的 compaction 过程中被消除。
        而对于 delete 操作，则是直接插入一个墓碑值作为最高版本数据，这样后续的 compaction 过程会将该数据的其他版本消除。

- ### LSM-Tree 的基础优化组件
    LSM-Tree 将存储空间作为 append only log 来操作，本身写性能已经非常不错了，LSM-Tree 的优化方法多数是为了优化磁盘文件查询效率。
    - **fence pointer**
        为了加快 run 内 sstable 的检索而建立的一种索引。查询操作可以根据 fence pointer 通过二分查找快速确定 run 内包含 key 值的 sstable 或者文件块段。
    - **bloom filter**
        关于 bloom filter 这里主要叙述其在 LSM-Tree 中的所扮演的角色。 bloom filter 对 LSM-Tree 的查询性能优化极大，优化程度与 bloom filter 的假阳率(false positive)相关。
        - one bloom filter per sstable
            不得不说 bloom filter 和 sstable 的相性极好。 bloom filter 的弱项是删除，而 sstable 正好是只读的，不存在删除问题，只需要在新的 sstable 的创建的过程中，顺便创建该 sstable 的 bloom filter 即可。 bloom filter 可以快速判断一个 key 是否存在于 sstable 中，如果判定存在则代表可能存在，有读取 sstable 文件的价值，如果判定不存在则一定不存在于 sstable 中。这样牺牲一点空间就可以节约大量不必要的I/O，加快查询效率。
        - one bloom filter per run
            对于采用 size-tiered compaction 策略的 LSM-Tree 可以这样理解。因为大部分时候是 one sstable per run 。这样做的好处是省去了 key 未包含时 run 内部检索的开销，但是如果 run 内包含多个 sstable 而且存在局部更新的情况(如 leveled compaction 策略中就会发生 run 内部分更新)则需要为了这部分更新而新建整个 run 的 bloom filter ，因为 bloom filter 无法处理删除。
        - 关于 bloom filter 的合并与新建的选择
            在 WiredTiger 的 wiki 中有过相关讨论。在合并两个 sstable 时 bloom filter 可以通过或操作来快速合并(如果它们的大小相同)。但是我们知道 LSM-Tree 的删除是通过插入墓碑值来进行的，所以合并两个 sstable 是可能会使某个 key 值被丢弃，如果直接快速合并 bloom filter ，这会因丢弃 key 还存在而造成更大的假阳率。而且这个假阳率的影响是无法简单通过对被丢弃的 key 再建一层索引来解决的(这和 bloom filter 的结构有关，因为被丢弃 key 的 hash 计算值所污染的区域配合上真实存在的 key 的填补区域会造成更多 key 存在的假象 -- emmm 比较难以用语言来形容)。而且在有了 《Less Hashing, Same Performance:Building a Better Bloom Filter》 中所介绍的方法，可以用更少的 hash 计算开销建立 bloom filter 。而且这样不需要保证所有 bloom filter 都保持相同大小。
        - Monkey

- ### 更多的优化方式
    - **key value 分开存储**
    - **TRIAD**
    - **LRUCache**
        leveldb 的读优化策略，使用更多内存来存储热数据。
        在了解此之前我一直有个疑问，就是如果承载热数据的工作交给了 memtable ，那么查询操作所标定的热数据即使不被更新也需要插入到 memtable 中，如果不在刷盘时剔除这些未被修改过的数据会造成严重的写浪费。既然如此，还不如用更加快速更加专业的数据结构和算法来管理这部分读缓存。
        所以 leveldb 用独立的 LRUCache 来管理读缓存。数据结构是 hash 加上热链和冷链。为了支持更高的并发访问又做了专门的一层 ShardedLRUCache ，这层是为了提高多线程访问的效率，减少竞争。 ShardedLRUCache 内部有 16 个独立的 LRUCache ，查找时会先确定 key 值所在 LRUCache ，然后再上锁查找。
        关于 leveldb 的代码实现细节网上的优质文章实在是太多，这里就不再赘述了。

## 二、LSM-Tree 设计核心

- sstable 组织和 compaction 策略
    因为 LSM-Tree 是由树结构衍生而来的，原始算法描述的磁盘内数据也是抽象为一颗树，所以 sstable 天然具有一定的层级结构。一般 sstable 层级结构是按照数据版本(新旧)进行分层的，较新版本的数据位于上层，这与内存中数据刷入磁盘的顺序相符。
    - size-tiered compaction
        按照 sstable 的大小来进行分层，越下层的 sstable 越大。
    - leveled compaction
        one run per level 的 sstable 组织方式，这样的分层方式有着不错的读性能。但是在 compaction 过程中，为了维护 one run per level 的状态，需要将向下层合并的 sstable 和下层的有 key range 重叠部分的 sstable 都读取并将数据归并写入到新的同层 sstable 中。这实际上造成了写放大，因为原本属于该层的数据被读出并再次写入到同一层中。
