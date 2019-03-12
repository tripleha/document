<h1 style="text-align:center"> mongodb 分片技术 </h1>

- 主要研究源码版本:
    **注意**: 这里出了个小事故，本来以为看的是 4.0 分支的版本，不过可能是切分支时操作失误，最后看的是正在开发的最新代码(12月份初之后未 fetch 更新过)。
    > <https://github.com/mongodb/mongo/tree/master/src/mongo> mongodb 最新源码。
    > <https://github.com/mongodb/mongo/tree/v4.0> mongodb 4.0版本源码，可以做参照，因为实验可能还是需要在一个稳定版本上跑的。
- 主要参考文档:
    > <https://github.com/mongodb/mongo/wiki/Sharding-Internals> 基于 mongodb 3.4源码的分片机制介绍，因为版本老旧，所以只能当作参考，很多地方4.0的源码上都进行了改动。
    > <https://docs.mongodb.com/manual/sharding/> mongodb 官方文档，对于 mongodb 的功能做用户向的了解，以便于理解源码的目的。
- 网上相关论坛博客参考:
    > <http://www.mongoing.com/archives/3476> MongoDB sharding迁移那些事（一），该博客是基于3.2版本的，仅提供思路。

---

## **一. mongodb 分片技术主要模块**

### **1. mongodb 集群的三大组成部分**

&emsp;&emsp;每个 mongos 或者 mongod 实例都会绑定一个**端口**运行，来接受访问和进行通信。

- #### router

    > <https://docs.mongodb.com/manual/core/sharded-cluster-query-router/> 关于 router 的官方介绍

    &emsp;&emsp;一个集群可以有多个 router 。
    &emsp;&emsp;router 是 **mongos** 实例，是客户端程序与 mongodb 集群的交互入口。router 并不会实际存储数据，而是将集群路由表，即 config.databases, config.collections 和 config.chunks 等 collection 中储存的元数据)以易查找的数据结构缓存，将读和写请求按照相关 **shard key** 路由到对应分片上，或者在无法根据路由表确定请求执行分片时广播请求。
    两种操作请求:
    
    > <https://docs.mongodb.com/manual/core/sharded-cluster-query-router/#targeted-operations-vs-broadcast-operations> 关于底下两种操作的官方介绍
    
    - **broadcast operation**
        &emsp;&emsp;当 router 无法根据请求所包含的信息确定哪些分片必须接收这个请求时，便会广播这个请求让所有分片都去执行。多文档的更新或者删除操作若没有包含全部的 shard key 信息，则必定会被广播。
    - **targeted operation**
        &emsp;&emsp;当数据操作包含 shard key 信息时，router可以确认哪些分片可能包含需要操作的数据，或者对于插入操作可以确定数据插入的分片。router会将请求路由到单个分片上或者集群分片的某个子集上。

- #### config server

    > <https://docs.mongodb.com/manual/core/sharded-cluster-config-servers/> 关于 config server 的官方介绍

    &emsp;&emsp;一个集群上只能有一个 config server，但是可以有其他复制集。
    &emsp;&emsp;config server 是特殊的 **mongod** 实例，储存元数据的 **config** 数据库的物理存储位置。在 config server 上存储的元数据是权威性数据，集群上其他分片和 router 都以其上的元数据作为最终版本。config 数据库不可分片。
    &emsp;&emsp;**balancer** 模块运行于 config server 下。其主函数运行在 config server 的 primary(复制集) mongod 实例的一个单独的子线程里。
    - **config 数据库**

        > <https://docs.mongodb.com/manual/core/sharded-cluster-config-servers/#sharded-cluster-metadata> 官方文档中对于集群中 config 数据库的介绍

        &emsp;&emsp;config 数据库虽然可以直接修改，但是可能会导致整个集群的不稳定或者错误的状态，所以最好不要直接修改 config 数据库，应该使用 mongodb 提供的集群操作命令来间接修改 config 数据库。
        config 数据库包含以下collection(主要的):
        - config.changelog
            元数据的修改日志。
        - config.databases
            记录集群中的每个数据库以及它们的primary shard等信息。
        - config.collections
            记录集群中的每个已分片的 collection 的信息。
        - **config.chunks**
            记录集群中每个chunk的信息，包括范围，所在分片，所属集合等。
        - config.lockpings
            跟踪记录集群中的各个实例，以及最后ping的时间。
        - config.locks
            保存分布式锁，记录是哪些模块在使用锁，锁所属的 namespace ，为什么使用锁等信息，锁释放后相关锁文档删除。
        - config.mongos
            保存集群中启用的 mongos 实例的信息。
        - **config.settings**
            保存集群设置，仅包含 chunk size ， balancer 设置和状态， 是否启用自动切分 chunk 机制。
            下面是例子:
            { "_id" : "chunksize", "value" : 64 }
            { "_id" : "balancer", "stopped" : false }
            { "_id" : "autosplit", "enabled" : true }
        - **config.shards**
            保存集群中启用的分片以及分片绑定的tag(zone)的信息。可以添加 maxSize 来表示这个分片可以使用的存储空间。
        - **config.tags**
            记录每个 **zone** 的 tag 和它关联的 range 。
        - config.version
            只包含一个文档，记录当前元数据的版本数字。
            例如: 
            { "_id" : 1, "minCompatibleVersion" : 5, "currentVersion" : 6, "clusterId" : ObjectId("5c1288bc4e3fab8d8de59a22") }
        - **config.migrations**
            存储正在进行和将要进行的 chunk 迁移命令。该集合主要是为了防止 balancer 突然中断而用于恢复 balancer 运行的，即储存了 balancer 还未完成的迁移任务。

- #### shard (分片)

    > <https://docs.mongodb.com/manual/core/sharded-cluster-shards/> 关于分片实例的官方介绍

    &emsp;&emsp;一个集群上可以有多个 shard ，每个分片都可以有其他复制集。
    &emsp;&emsp;shard 是 **mongod** 实例，存储用户数据，对于一个被分片的 collection 而言，每个分片上的数据都是不重叠的。用户数据以 **chunk** 为单位来划分管理。自身也会缓存路由表信息用以执行 chunk 迁移，和分片聚合还有分片 mapReduce 操作。此外还会比 router 多缓存其所拥有的各个 collection 的 chunk 元数据。
    对于每个分片，有**集群用户**和**本地用户**的区分:
    - 集群用户
        &emsp;&emsp;通过连接 router 来间接访问数据的用户是集群用户。在他们的视角集群就是一个整体，他们可以访问所有分片下的数据，并且不需要知道具体的数据位于哪个分片上(数据分片对于普通用户是封装的)。
    - 本地用户
        &emsp;&emsp;直接连接分片 mongod 来访问数据的用户是本地用户，他们只能够访问到存储在该分片下的局部数据。

### **2. 一些重要概念**

- #### primary shard

    > <https://docs.mongodb.com/manual/core/sharded-cluster-shards/#primary-shard>

    &emsp;&emsp;该概念区分于复制集的 primary 实例。
    &emsp;&emsp;**collection** (类似table)可以被切分以将数据分布到集群的所有分片上，一个 **database(数据库)** 可以同时包含已分片的 collection 和未分片的 collection ，未分片的 collection 的数据完全储存在 database 的 primary shard 上。
    &emsp;&emsp;当在 router 上执行建立数据库的命令时，会自动挑选一个数据负载较轻(存储使用空间较少)的分片作为这个新建的 database 的 primary shard 。

- #### shard key

    > <https://docs.mongodb.com/manual/core/sharding-shard-key/>

    分为 **range** 和 **hash** :
    - range
        &emsp;&emsp;可以是复合的key，根据key的大小范围来将数据集切分为一个一个 chunk 。复合key的切分顺序为key的声明顺序。
    - hash
        &emsp;&emsp;先将key hash之后，以hash值来作为range切分的依据，实际上是在range的基础上加上了hash。
        - 相较于直接range切分的优点:
            &emsp;&emsp;对于数据是在shard key上单调递增或递减的情况，插入操作的吞吐率将会受到限制，无法发挥集群的优势，因为数据总会先插满一个chunk再分裂插入下一个chunk，这样某一时刻所有操作都会被路由到同一个分片下。
        - 缺点:
            &emsp;&emsp;对于range查询等一些特定的查询则只能广播请求。而且一些操作会受到限制，因为你无法根据hash值直接映射回具体的值。大多数场景下我们还是会选用 range shard key，寻求从 shard key 的选取上入手的优化。

    &emsp;&emsp;一个 collection 的 shard key 一旦选定之后便不可更改，并且用户也不可以修改已经插入的文档中的 shard key 字段。
    - **关于 _id 字段:**
        &emsp;&emsp;如果 shard key 不包含 _id 字段，则 _id 字段仅仅起到使每个分片下的数据文档保持唯一性的作用。该情况下，不同两个分片中可能会存在相同 _id 字段的数据。
    - **关于唯一性:**
        &emsp;&emsp;只有 shard key 以及包含 shard key 作为前缀的索引可以是唯一性的，_id 字段保证在单个分片下的唯一性。shard key 只能在作为整体时具有唯一性，不能只指定 shard key 中的某个字段有唯一性。hash shard key 不支持具有唯一性(一般来说也很难发生碰撞)。
    
    **注意:** 所有单一文档的更新或者删除请求必须包含 shard key 或者 _id 字段。

- #### chunk

    > <https://docs.mongodb.com/manual/core/sharding-data-partitioning/> chunk 机制的官方介绍

    &emsp;&emsp;chunk 是 balancer 等各种数据分片管理模块管理数据的最小单位，也是 mongodb 管理集群的最终要的数据结构之一，代表的是一个连续范围内的数据，可用一个上开下闭(包括下界，不包括上界)的区间表示。
    &emsp;&emsp;chunk 有大小上限(chunk size)，可以自行设定并可以随时更改，范围是 1-1024MB，默认值是 64 MB。除了大小限定，也有文档数量限定(Maximum Number of Documents Per Chunk to Migrate)。
    &emsp;&emsp;chunk 会自动分裂，真正意义上的自动分裂是发生在数据插入的时候的，balancer 运行时也有可能去分裂 chunk。一般来说当 chunk 在插入或更新操作后超过大小上限或文档数量上限时就会触发自动切分(见下面介绍)。
    - **Maximum Number of Documents Per Chunk to Migrate(以下简称 max doc num)**

        > <https://docs.mongodb.com/manual/reference/limits/#Maximum-Number-of-Documents-Per-Chunk-to-Migrate> 官方文档中关于 max doc num 的描述

        &emsp;&emsp;一个 chunk 允许储存的最大文档数量。计算公式: (1.3 * chunk大小上限 / collection平均文档大小)。如: 平均文档大小为 32KB，则这个限制是 1.3 * 64 * 1024 / 32 = 2662 个。
        &emsp;&emsp;这个限制也是一个可以进行迁移的 chunk 最多包含的文档数量。
    - **jumbo chunk**
        &emsp;&emsp;超过限制且无法切分的 chunk，会被标记为 jumbo。jumbo chunk 是无法进行迁移的。一般来说是因为 shard key 选取不当导致某一 shard key 的值包含了超过文档数量限制的文档数或者其内文档总大小超过 chunk size，毕竟 chunk 切分只能最小切到仅包含某一 shard key 值的程度。
    - **chunk 与存储引擎的关系**
        &emsp;&emsp;因为 mongodb 是可插拔存储引擎的，所以这里指的 chunk 是抽象于底层存储(存储引擎)而存在的，是 mongodb 功能实现层的数据结构，与存储引擎层的chunk是两个概念(chunk 按照 range 来划分数据，给人一种数据是按照 shard key 排序存储的感觉，然而存储引擎并不是按照这个顺序来排序存储数据的)，所以大部分情况下对于 chunk 的操作(如切分，合并)并不改变引擎层的存储，改变的仅仅是集群的元数据(metadata)而已，因此这些操作消耗小而且比较快。
    - **chunk size的改变的影响**
        &emsp;&emsp;我们可以按照需求来实时改变chunk size，当我们改变chunk size的时候，并不会直接产生什么影响，而是等到数据插入或者更新，或者balancer运行的时候才会来检查。
        &emsp;&emsp;当我们把chunk size改大(max doc num 也会变大)，那么一些被标记为 jumbo 的 chunk 会取消标记，而变得可以在分片间迁移。
        &emsp;&emsp;当我们把chunk size改小，那么一些超过大小上限或者数量上限的 chunk 就会被切分成更小的 chunk 。
        &emsp;&emsp;chunk 的切分是不可逆，**因为 mongodb 并没有自动合并 chunk 的机制**，所以一旦我们发现 chunk size 改得太小而调大的时候，已经分裂的 chunk 并不会重新合并到一起去。
    - **手动合并chunk**
        &emsp;&emsp;chunk 虽然不会自动合并，但是可以使用mongodb 命令(mergeChunks)手动合并 chunk ，但是合并仅限于位于同一分片下的范围相接的chunks。mongodb 不提供自动合并的机制是因为没有这个必要，因为数据是不停插入的，一般数据集都会不断变大，而且 mongodb 自身的自动切分 chunk 机制并不会把数据切成碎片，所以自动合并一般并没有太大意义。

- #### routing table(路由表)

    > <https://github.com/mongodb/mongo/wiki/Sharding-Internals#caching-the-routing-table-databases-collections-chunks> 3.4版本的文档，实现有变化，不过可以参考。

    &emsp;&emsp;mongodb 采用了路由表缓存机制来节省每次从config server拉取路由信息的网络传输开销。但是这样一来又引进了新的问题，即如何控制路由表的更新。
    - **version**
        &emsp;&emsp;mongodb 采用 version 信息来控制。version 实际上是一种bson数据结构 **Timestamp** 。
        - **Timestamp**
            &emsp;&emsp;是一个64位值，原始定义中前32位 time_t 表示unix时间(秒数)，后32位是一个 counter 值。而在 version 的使用中，情况有所不同。之所以使用Timestamp是为了持久化存储。
        - **ChunkVersion**
            &emsp;&emsp;对于 **ChunkVersion** 形式的 version ，其前32位用来表示 majorVersion，后32位用来表示 minorVersion。当major加一时，minor置零。ChunkVersion 还包含一个 epoch 参数，是一个 ObjectID。当比较 ChunkVersion 是否相同时，还必须要检查 epoch 是否相等，比较大小时只需比较64位数据。
            &emsp;&emsp;写一致性保证需要确保操作所知的 version 的 epoch 和 major 与当前 shard 的记录相同。
    
    - 路由表会在三种情况下变成需要更新的状态:
        1. 一个分片被从集群中移除
        2. 一个 collection 改变分片状态(变成可分片的或者不可分片的)
        3. 一个 chunk 完成迁移

    - 更新策略:
        &emsp;&emsp;一般对于分片自己发起的操作(如自动分片操作)，或者需要分片配合完成的与整个数据库或者 collection 状态改变相关的操作，在操作结束后分片都会刷新自己的路由表数据缓存。
        &emsp;&emsp;其他时候，router 和分片可以进行一种被动的路由表更新，一般如果 config server 上的权威元数据更新之后，router 的请求操作会带上其所知的当前请求所使用的元数据的版本，若是版本错误，或者说路由失败，则请求会返回错误信息，此时 router 便会向 config server 拉取最新的相关元数据进行路由表增量更新。
        &emsp;&emsp;当 router 的请求带有的路由信息版本比分片当前所知的路由版本高时，分片也会向 config server 拉取最新的元数据进行路由表更新和本地 chunk 元数据更新。

- #### zone

    > <https://docs.mongodb.com/manual/core/zone-sharding/> 关于 zone 的官方文档介绍

    &emsp;&emsp;mongodb 提供给用户自定义数据在集群中分布的数据结构。
    包含 **tag** 和 **range** :
    - **tag**
        &emsp;&emsp;tag 表示一个 zone 的命名。其用于与shard进行绑定。一个 tag 可以绑定到多个 shard 上。绑定关系记录在 config.shards 中。
    - **range**
        &emsp;&emsp;range 用于表示属于这个 zone 的数据的范围，这个 range 的含义和 chunk 是一样的。用户可以绑定多个不连续的 range 到同一个 tag 上。相关信息记录在 config.tags 中。

- #### balancer

    > <https://docs.mongodb.com/manual/core/sharding-balancer-administration/> 关于 balancer 的官方介绍

    &emsp;&emsp;mongodb 自动化集群管理中最重要的模块。数据在集群中的自动化迁移依赖于 balancer 来完成。balancer 功能处于不断完善的过程中。
    下面信息主要来源于官方介绍以及相关博客内容的推导，并未求证于源码:
    > <https://docs.mongodb.com/manual/core/sharding-balancer-administration/> 中有提到 config.locks 中文档关于不同版本的兼容问题。在其他文档中也有相关介绍。
    
    > 在 3.2 版本位于 router 上，每个 router 都会运行一个 balancer ，balancer 每一个 round 开始时都会先抢锁，谁抢到谁就可以运行。锁存在config server上。

    > 在 3.4 版本，balancer 移到了 config server 的 primary(复制集) 上，在 balancer 运行时还是会获取一个锁。
    
    > 在 3.6 版本，移除了 balancer 本身的锁占用，而是在运行到具体步骤需要进行数据修改时才申请锁。

    &emsp;&emsp;用户可以手动设置 balancer 的运行窗口，即运行时段，balancer只会运行窗口内启动运行。用户还可以手动停止 balancer 或者禁用 balancer。禁用 balancer 可以只针对某些 collection 。禁用后可再次启用。
    主要相关类:
    - Balancer
        &emsp;&emsp;所有迁移相关的请求(自动切分所引发的迁移，用户手动命令调用)最终都是调用的 Balancer 类上定义的函数，也就是说所有真正的迁移过程都是由 Balancer 发起的。
    - BalancerChunkSelectionPolicy
        &emsp;&emsp;定义 chunk 的选取方法。Balancer 的成员变量。
    - MigrationManager
        &emsp;&emsp;定义迁移命令函数入口。Balancer 的成员变量。

    &emsp;&emsp;总的来说，balancer 主要包含迁移chunk选取策略和迁移执行两个模块。在迁移chunk选取时也会进行跨 zone 边界的 chunk 的切分。而迁移执行部分主要进行的是迁移命令的发布，具体迁移的执行并不由 balancer 来完成。可以说 zone 的功能是由 balancer 来完成的。

## **二. mongodb 分片技术的重要数据结构实现和算法流程**

### **1. 数据结构实现**

#### 1.1 路由表缓存层次数据结构

> <https://github.com/mongodb/mongo/blob/master/src/mongo/s/catalog_cache.h>
> <https://github.com/mongodb/mongo/blob/master/src/mongo/s/chunk.h>
> <https://github.com/mongodb/mongo/blob/master/src/mongo/s/chunk_version.h>

&emsp;&emsp;这是 mongodb 的缓存结构设计，因为在源码中经常涉及到缓存相关的操作，所以略做介绍，还有分片直接缓存的元数据结构未做介绍。

- CatalogCache
    &emsp;&emsp;CatalogCache 是 mongodb 路由表缓存结构的入口。
    - _databases
        &emsp;&emsp;是一个hash表，
        &emsp;&emsp;\<数据库的namespace: DatabaseInfoEntry\>
    - _collectionsByDb
        &emsp;&emsp;是一个双层hash表，
        &emsp;&emsp;\<数据库的namespace: \<collection的namespace: CollectionRoutingInfoEntry\>\>
    
    &emsp;&emsp;Entry类数据结构定义一般都包含是否需要更新和等待更新完成的标志位，此外才是真正的包含信息的结构体。

- DatabaseInfoEntry
    - bool needsRefresh
        &emsp;&emsp;是否需要更新
    - refreshCompletionNotification
        &emsp;&emsp;若需要更新，更新是否完成
    - bool mustLoadShardedCollections
        &emsp;&emsp;数据库路由更新后，相应的collection路由也需要由其他线程更新，这个标志位防止多个线程竞争更新，只有第一个修改这个标志位的线程可以来执行更新操作。
    - DatabaseType dbt
        &emsp;&emsp;表示 config.databases 里面的文档的数据结构
        - _name
            &emsp;&emsp;即 数据库的namespace
        - _primary
            &emsp;&emsp;记录了这个数据库的 primary shard 的ID
        - bool _sharded
            &emsp;&emsp;记录这个数据库是否是分片的
        - _version
            &emsp;&emsp;记录当前路由表所知道的这个数据库的版本，是 DatabaseVersion（不同于ChunkVersion且不是基于Timestamp）。

- CollectionRoutingInfoEntry
    - needsRefresh
    - refreshCompletionNotification
    - RoutingTableHistory routingInfo
        &emsp;&emsp;存储了相关 collection 的 shard key 信息，包括其具体形式，key的排序和是否具有唯一性。以及下列信息。
        - _chunkMap
            &emsp;&emsp;是一个 std::map ，按照chunk的上界来检索chunk信息。
            &emsp;&emsp;\<chunk的上界: ChunkInfo\>
            - ChunkInfo
                &emsp;&emsp;包含 chunk 的range信息，shardID，和该 chunk 的 ChunkVersion 等信息。
                - _writesTracker
                    &emsp;&emsp;是一个 ChunkWritesTracker 实例。记录该 chunk 写入了多少数据
        - _collectionVersion
            &emsp;&emsp;是一个 ChunkVersion 实例，是当前该 collection 中所有的 chunks 中最高的 ChunkVersion 值。
        - _shardVersions
            &emsp;&emsp;是一个 std::map ，用该 collection 在每个 shard 中最高的 ChunkVersion 来表示 shardVersion 。
            &emsp;&emsp;\<shardID: ChunkVersion\>
        - ... shard key 相关变量

### **2. 算法流程**

**注意**: mongodb 源码一般会把函数功能注释写在 .h 文件里，具体实现在 .cpp 文件中。在浏览器中打开可使用 ctrl + f 查找相应函数名。

#### 2.1 chunk 自动切分算法

#### **2.1.1 算法简述**

&emsp;&emsp;因为文档在分片间的迁移管理是以 chunk 为单位的，所以 mongodb 的集群体系中需要包含 chunk 自动切分的机制来使得每一个 chunk 的大小都尽量均匀，这样其它基于 chunk 的管理机制才有意义。chunk 只是 mongodb 层面的概念，它的切分并不会对底层数据存储造成影响，只是基于元数据的操作，所以即使进行频繁的 chunk 切分也不会有太严重的性能影响。当然，合适的 chunk 切分机制能减少切分次数，还是很有必要的。mongodb 作为一个通用型数据库，它的所采用的方法是面向一般情况的简单处理及优化方法。而且 mongodb 提供了手动的切分和合并 chunk 的命令，使得DBA(数据库管理员)在特殊情况下可以手动控制调优数据切分。chunk 的自动切分发生在 chunk 所在的分片上。
- top chunk 优化:
    &emsp;&emsp;只针对于 range shar key 的优化，hash shard key 不会执行该优化。mongodb 认为range shard key 总是按照 shard key 值上递增或者递减插入的，所以插入操作总是集中在边缘 chunk 。所谓边缘 chunk ，即那种包含了整个集合 shard key 最小值或者最大值，边缘为无穷的 chunk 。mongodb 会尽量让这些 chunk 在切分时拥有更多可插入的空间，并尝试将这些 chunk 迁移走以平衡集中的插入操作所带来的负载(从mongodb 的源码设计上可以看出，mongodb 假设查询和更新负载是均匀分布在数据上的，也就是 chunk 越多，负载就越重，所以这里会尝试找一个 chunk 少的分片安置这个边缘 chunk ，如果分片本身 chunk 最少就不会发生迁移)。
- mongodb 的 chunk 自动切分机制主要有四个步骤:
    1. 调用 splitVector() 获取切分点列表(升序的)。首先利用查询方法获取到当前分片下该 collection 所有的文档个数和总大小计算平均文档大小，然后根据 chunk size 阈值计算一个 chunk 达到阈值所会包含的文档个数的一半(实际上就是按照阈值一半大小切)，将这个值作为切分间隔，遍历 chunk 文档选定切分点。(例如，一个 chunk 包含 1～13 13个文档，阈值10,则切分点列表是 [5, 10])。若是切分点个数不超过 1 个则说明 chunk 大小还没有到阈值，无需切分。
    2. top chunk 优化第一步，若当前切分的 chunk 是边缘 chunk ，则将切分点列表中的第一个/最后一个切分点换成当前最小/最大 shard key 值。(如上面的例子，若1是最小key，最后列表会变为 [1, 10])
    3. 第二步骤是调用 splitChunk() 按照切分点切分 chunk (修改元数据)，并更新到 config server 的权威元数据上。
    4. top chunk 优化第二步，若是切分的是边缘 chunk ，那么肯定会切出另一个很小的边缘 chunk ，此时以命令请求的形式发送向 config server 让其执行 Balancer::rebalanceSingleChunk() 。

#### **2.1.2 基于源码的详细分析**

- **ChunkSplitter**

    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/chunk_splitter.h>
    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/chunk_splitter.cpp>

    &emsp;&emsp;处理 chunk 自动切分的类，采用线程池的方式来并发执行 chunk 切分操作。
    &emsp;&emsp;自动切分的运行入口是 ChunkSplitter::trySplitting() 。**但是还没有找到 ChunkSplitter::trySplitting() 的调用处，所以还不能从源码上确认触发机制。**
    - **ChunkSplitter::trySplitting()**
        &emsp;&emsp;调用时会传入一个 ChunkSplitStateDriver 实例。
        &emsp;&emsp;检查当前的 ChunkSplitter 是否是 primary(复制集) 下的实例。
        &emsp;&emsp;将一个 ChunkSplitter::_runAutosplit() 任务放入线程池中执行。ChunkSplitStateDriver 也被作为参数传入。
        - **ChunkSplitter::_runAutosplit()**
            &emsp;&emsp;检查 primary(复制集) 标志。
            &emsp;&emsp;获取相关路由信息和 ChunkManager 以及 balancer 设置。因为用户是否允许集群自动切分 chunk 的设置是定义在 balancer 的设置中的。获取到之后检查是否允许自动切分。
            &emsp;&emsp;调用 ChunkSplitStateDriver::prepareSplit() 进行切分前的准备。
            - **ChunkSplitStateDriver::prepareSplit()** 

                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/chunk_split_state_driver.h>
                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/chunk_split_state_driver.cpp>

                &emsp;&emsp;将 ChunkSplitStateDriver::_splitState 修改为准备就绪状态。
                &emsp;&emsp;将当前 chunk 的 ChunkWritesTracker 记录的当前 chunk 的写入字节数(这只是一个估计值，并不准确)暂存到 ChunkSplitStateDriver::_stashedBytesWritten 中，然后清零 ChunkWritesTracker 记录的写入字节数。

            &emsp;&emsp;
            &emsp;&emsp;调用 splitVector() 方法获取该 chunk 的切分点列表。
            - **splitVector()**

                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/split_vector.h>
                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/split_vector.cpp>

                &emsp;&emsp;如果 maxChunkObjects 未传入，则设置其为 250000 ，表示一个 chunk 默认最多包含 250000 个文档，在 _runAutosplit() 的调用中是没有传入的，所以使用的是默认值。
                &emsp;&emsp;找到操作的 collection 和 shard key 的索引。然后获取到当前分片下整个 collection 的文档数 recCount 和总大小 dataSize 。
                &emsp;&emsp;此时会根据不同的参数传入情况而决定 maxChunkSize 。在 _runAutosplit() 的调用中，是用户定义的 chunk size 所代表的字节数。
                &emsp;&emsp;如果 maxChunkSize 大于 dataSize ，则直接返回空切分点列表。
                &emsp;&emsp;avgRecSize = dataSize / recCount 计算平均单个文档大小。
                &emsp;&emsp;keyCount = maxChunkSize.get() / (2 * avgRecSize) 计算每个子 chunk 应该包含的文档数量。
                &emsp;&emsp;初始化 splitKeys 数组，建立一个 cursor 用以遍历 chunk 范围内的文档(不包括 chunk 上界)。并初始化一个 tooFrequentKeys set 来存那些重复出现的 key 值。
                &emsp;&emsp;先将第一个文档 key 加入到 splitKeys 数组中。这个 key 是为了防止第一个切分点与 chunk 第一个文档 key 相同。
                &emsp;&emsp;然后开始遍历，这个 cursor 是按照升序或者降序遍历文档的，每遍历到第 keyCount 个文档，检查其 shard key 部分是否与 splitKeys 的最后一个 key 相等，若相等则加入到 tooFrequentKeys 中，否则加入到 splitKeys 中。
                &emsp;&emsp;遍历 tooFrequentKeys 打印中的 key 。
                &emsp;&emsp;把 splitKeys 开头的第一个 key 移出该数组。
                &emsp;&emsp;排序 splitKeys ，保证其是升序的，然后返回。

            &emsp;&emsp;
            &emsp;&emsp;获取到切分点列表之后。如果切分点个数不大于1，则暂时不切分。因为按照切分点的获取算法，可以知道正常情况下第二个切分点处正好达到 chunk size 的大小，所以只有一个切分点说明 chunk 大小应该介于一半的 chunk size 与 chunk size 之间。这时调用 ChunkSplitStateDriver::abandonPrepare() 直接把 ChunkSplitStateDriver::_stashedBytesWritten 清零。然后退出。这样调用 ChunkSplitter::trySplitting() 可以检测到切分操作被放弃。
            &emsp;&emsp;若拥有足够的切分点，切分继续进行。
            &emsp;&emsp;源码此处有一个考量(top-chunk 优化)，就是设计者认为对于排序型的 shard key (与 hash 区别)一般的插入操作都是根据 shard key 递增或者递减的，那么处于边缘的 chunk (即那些包含了 shard key 最小值或者最大值的 chunk ，一般它们的下界无穷小或者上界无穷大)，这样的 chunk 可能是插入操作集中发生的地方，也是最容易被插满又很快需要切分的。
            &emsp;&emsp;调用 KeyPattern::isOrderedKeyPattern() 查看 shard key 是否是排序型的。
            &emsp;&emsp;如果 shard key 是排序型的并且此时切分的是上面说的边缘 chunk ，那么就稍微更改一下 splitKeys 。如果是包含 minkey 的 chunk 就将 splitKeys 的第一个切分点改为那个 minkey，包含 maxkey 的就将 splitKeys 的最后一个切分点改为 maxkey ，注意这里说的 minkey 和 maxkey 不是无穷小或者无穷大，而是真是存在 collection 中的文档的最小或者最大 key 。
            &emsp;&emsp;上述步骤结束后，调用 splitChunkAtMultiplePoints() 按照 splitKeys 定义的切分点对 chunk 进行切分。
            - **splitChunkAtMultiplePoints()**
                &emsp;&emsp;检查 splitPoints 数组大小，其大小不能超过 **8192**，因为正常情况下就算 chunk size 从 1024MB 变为 1MB，需要的最多的切分点个数也不会超过 8192 个。
                &emsp;&emsp;检查完毕后调用 splitChunk() 进行 chunk 切分。
                - **splitChunk()**

                    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/split_chunk.h>
                    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/split_chunk.cpp>

                    &emsp;&emsp;获取该collection在该分片上的元数据锁。
                    &emsp;&emsp;然后向 config server 发送切分chunk的命令。实际上是远程调用位于 src/mongo/db/s/config/configsvr_split_chunk_command.cpp 中定义的 ConfigSvrSplitChunkCommand::run() 函数。该函数在解析了请求之后会调用 ShardingCatalogManager::commitChunkSplit() 。
                    - **ShardingCatalogManager::commitChunkSplit()**

                        > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/config/sharding_catalog_manager.h>
                        > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/config/sharding_catalog_manager_chunk_operations.cpp>

                        &emsp;&emsp;获取一个锁来防止元数据的改动，这里的注释上标明了TODO，将来会将这个全局锁变成一个只针对单个 collection 的锁。
                        &emsp;&emsp;接下来会获取config server上的权威元数据的 collection 的 version (即 当前 collection 中最大的 chunkVersion)。如果请求的 chunkVersion 和 collection 的 version 的 epoch 不相同，则说明版本不一致，返回错误。
                        &emsp;&emsp;在验证过 epoch 之后，开始遍历 splitPoints 。遍历时先检查每个切分点是否都在chunk范围内，检查切分点是否是按照升序传入的并且不重复(必须比前一个大)，检查这个切分点的key形式的大小，不能超过阈值512字节。并且检查这个切分点的key形式没有使用不允许使用的bson结构，因为 shard key 只规定了key名，但是没有规定每个key下value必须是什么类型。在完成所有这些检查之后，将当前最大 chunkVersion 的 minor 加一作为当前这个切分 chunk 的 chunkVersion 。然后以 update 的形式(upsert=true，按照 _id 字段索引，chunk的 _id 字段实际上就是nss-minkey的各个字段和其值)修改 config.chunks 元数据。这些 update 请求都收集起来，最后一起如果所有切分点都检查完毕了才会执行。并且把元数据更新操作也记录到 config.changelog 中。

                    &emsp;&emsp;
                    &emsp;&emsp;在远程调用返回后，如果返回的是错误，那么还需要调用 checkMetadataForSuccessfulSplitChunk() 检查一遍切分是否真的发生了，因为有时网络错误会导致操作成功，但是请求出错的情况。
                    &emsp;&emsp;在这些完成之后，这个函数也会考虑 top-chunk 优化，它会检查这次切分是否包含边缘 chunk ，并且如果边缘 chunk 只包含一个文档，就会返回那个边缘 chunk 的 range ，这个返回结果现在暂时没有被使用，但是可能在 mongodb 的后续版本中会用到。
                    &emsp;&emsp;否则返回空结果。
                    &emsp;&emsp;函数退出之后分片上的 collection 元数据锁会自动释放。

            &emsp;&emsp;
            &emsp;&emsp;切分完成后调用 ChunkSplitStateDriver::commitSplit() 将 ChunkSplitStateDriver::_splitState 修改为以提交状态。这样调用 ChunkSplitter::trySplitting() 的线程就可以知道切分操作已经完成了。
            &emsp;&emsp;完成切分之后会调用 forceShardFilteringMetadataRefresh() 刷新分片路由表缓存。
            &emsp;&emsp;在最后，考虑 top-chunk 优化的第二部分。这一部分的内容是尝试将边缘 chunk 移动到其他分片上。需要调用 isAutoBalanceEnabled() 检查是否允许 balancer 运行。如果允许则调用 moveChunk() 来转移 top-chunk 。moveChunk() 函数实际上是以远程请求的方式调用位于 config server 上的 Balancer::rebalanceSingleChunk() 函数。
            &emsp;&emsp;Balancer::rebalanceSingleChunk() 的调用和底下要介绍的 balancer main loop 中的过程类似，不过更加简单。实际步骤是为该 chunk 寻找一个所属 tag 内负载最轻的分片，如果该 chunk 已经位于该分片上就不用迁移了，否则调用 Balancer::_migrationManager.executeManualMigration() 进行迁移。该方法的执行与下面要介绍的_migrationManager.executeMigrationsForAutoBalance()过程类似，不过只需要考虑一个 chunk 。

#### 2.2 balancer main loop

#### **2.2.1 算法简述**

&emsp;&emsp;mongodb 的数据库逻辑层按照 chunk 来管理数据文档。随着数据的插入和更新导致的数据量的增大而使 chunk 不断分裂，就需要利用集群优势分布式存储数据(一刚开始只有一个 chunk ，数据都在一个分片上)。前文也介绍过 mongodb 的自动切分算法会尽量的让每个 chunk 大小均匀，并且 mongodb 假设负载均匀分布在数据上，那么此时为了平衡负载，mongodb 需要有一个 balancer 来管理 chunk 分布。而 balancer 的最佳位置还是在 config server 上，因为 balancer 本身的运算量不大，进行的多是发送命令的网络操作而且需要实时获取大量的权威元数据。balancer 最主要的工作就是让 chunk 在分片中均匀分布。让分片间的 chunk 数量差尽可能小。但是工作负载并不会如假设那样，所以 mongodb 使用 zone 这个数据结构让DBA能够自定义集群的数据分布，而 balancer 是在完全满足 zone 的基础上进行数据负载平衡的。
&emsp;&emsp;balancer 是不停在休眠和运行间切换的，数据负载不平衡时，balancer 并不能保证一次运行就全部解决，迁移协议也无法支持(同一时刻分片只能参与到一个迁移任务中)，所以 balancer 靠的是不停的休眠后运行来保持集群的动态平衡，当一次运行中发生了迁移之后，balancer 会缩短该次休眠时间快速开启下次运行，尽快解决所有的不平衡分布情况。
- balancer 的单次运行主要包含两大步骤:
    1. 切分阶段。调用 _enforceTagRanges() 来按照 zone 的范围切分现有 chunk 。
        &emsp;&emsp;balancer 必须首先满足DBA要求，而数据迁移又是以 chunk 为单位的，chunk 的范围不一定满足用户的要求，所以就需要切分那些跨越了不同 zoneRange 的 chunk ，把它按照 zoneRange 边界切分开，使得每个子 chunk 只能属于某一个 tag 。
        (1). 调用 Balancer::_chunkSelectionPolicy->selectChunksToSplit() 来获取需要按 zone 切分的 chunk 和切分点，这一步获取的切分点总是 zoneRange 边界，并且这一步并不考虑 chunk 本身大小是否适合切分。
        (2). 调用 shardutil::splitChunkAtMultiplePoints() 进行具体每个 chunk 的切分，该步骤实际上是发送请求到 chunk 所在分片来让其切分 chunk (主要是等待耗时)。
    2. 迁移阶段。调用 Balancer::_chunkSelectionPolicy->selectChunksToMove() 来选取当次运行需要迁移的 chunk 和其迁移目标节点。然后调用 _moveChunks() 来实施选定的迁移任务。
        (1). 选取时，主要按顺序进行三层选取:
        - 将要下线的分片上的 chunk 转移，选取目标是包含该 chunk 的 tag 绑定的分片，若没有 tag 包含则在集群范围内选取。
        - 将不在包含自己的 tag 绑定的分片上的 chunk ，将它们迁移到 tag 绑定的分片上。
        - 之后是 tag 绑定分片内部的平衡，因为一个 tag 会绑定多个分片。最后才是无tag相关的 chunk 在集群范围内的平衡。
        
        &emsp;&emsp;上面三层选取进行的过程中会有一个 usedShards 来记录每个已经选定参与到当次运行中的迁移任务的分片(包括 from 和 to)，如果选取迁移 chunk 时遍历到了 usedShards 中的分片则直接跳过，选取目标分片时也会忽略 usedShards 中的分片。
        (2). 调用 _moveChunks() 实施迁移任务是将 chunk 迁移命令发送到 chunk 所在分片执行。所以 balancer 端主要的操作还是挂起等待。
        (3). 对于因为 chunk 太大而无法完成迁移的 chunk ，会调用 _splitOrMarkJumbo() 来尝试对其进行类似于自动切分的阈值对半切分，对于无法再分的 chunk 会标记上 jumbo 标识(只是在缓存中)，之后的 balancer 运行会自动忽略 jumbo chunk 以节省时间。
        &emsp;&emsp;当然 jumbo chunk 的标记不是永久的，当DBA修改了 chunk size 后，自动切分机制运行时可能会将 jumbo chunk 切分开，那么形成的新的子 chunk 的元数据自然也就没有 jumbo 标识了。

#### **2.2.2 基于源码的详细分析**

- **Balancer._mainThread()**

    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/balancer.h>
    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/balancer.cpp>

- **启动和恢复阶段**
    &emsp;&emsp;在 Balancer._mainThread() 开始时，首先在当前线程建立 Client ，这个 Client 是和当前线程绑定的，用于在当前线程与其他节点沟通，mongodb每个需要网络通信的工作线程都会拥有一个 Client。
    &emsp;&emsp;然后从运行上下文中获取 BalancerConfiguration 实例，这个实例让 Balancer 可以获取到相关的设置信息以及调用接口判断是否能够进行某些动作。
    &emsp;&emsp;随后 Balancer 会开始尝试恢复之前未完成的迁移任务，等待直到之前的迁移任务完成。
- **main loop** (是一个循环，每次循环称为一个round)
    &emsp;&emsp;初始化一个 BalanceRoundDetails 来记录本次round的计时以及一些相关计数或者错误信息。
    &emsp;&emsp;设置 _inBalancerRound 为 true ，表示正在运行 Balancer 。
    &emsp;&emsp;重新加载和检查 Balancer 相关设置和参数，检查是否允许运行 Balancer ，主要检查是否 disable 了 Balancer 或者是否处于运行窗口时间内。
    &emsp;&emsp;调用 _enforceTagRanges() 按照 zone 裁剪 chunk 。
    - **_enforceTagRanges()**
        &emsp;&emsp;调用 _chunkSelectionPolicy->selectChunksToSplit() 获取根据 zone 切分 chunk 的切分信息(SplitInfo数组)。
        - SplitInfo
            &emsp;&emsp;存储一个要被切分的 chunk 的相关信息和切分点。
            - shardId
            - nss
            - collectionVersion
            - chunkVersion
            - minKey
            - maxKey
            - splitKeys 切分点数组
        - **_chunkSelectionPolicy->selectChunksToSplit()**

            > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/balancer_chunk_selection_policy.h> 基类定义位置，可看虚函数注释，SplitInfo 定义位置
            > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/balancer_chunk_selection_policy_impl.h>
            > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/balancer_chunk_selection_policy_impl.cpp>

            &emsp;&emsp;从 catalog 上获取所有 collections 。
            &emsp;&emsp;先打乱顺序，(这样返回的 SplitInfo 数组就不会总是按 collection 排序固定的)，再遍历每个 collection 。
            &emsp;&emsp;对于每个 collection ，先检查其是否已被删除。
            &emsp;&emsp;然后调用 _getSplitCandidatesForCollection() 获取该 collection 需要切分的 chunk 的SplitInfo 数组。
            - **_getSplitCandidatesForCollection()**
                &emsp;&emsp;获取相应 collection 的 ChunkManager，调用 createCollectionDistributionStatus() 获取该 collection 相关的 DistributionStatus 。
                - **DistributionStatus**
                    - _nss collection的namespace
                    - _shardChunks shardID对应chunk数组的map
                    - _zoneRanges range的上界对应range信息的map
                    - _allTags 所有该 collection 相关的 tag
                
                &emsp;&emsp;
                &emsp;&emsp;遍历 DistributionStatus._zoneRanges 的键值对，即 遍历每个zone的range，根据range的下界和上界，将它们添加到包含它们的 chunk 的 SplitInfo.splitKeys 中，表示要切分这些跨zone的range边界的 chunk 。
                &emsp;&emsp;收集这些 SplitInfo 到一个数组中，并返回。

            &emsp;&emsp;
            &emsp;&emsp;最终收集合并所有这些 SplitInfo 数组为一个 SplitInfo 数组，并返回。

        &emsp;&emsp;
        &emsp;&emsp;遍历数组中的每个 SplitInfo ，调用 shardutil::splitChunkAtMultiplePoints() 来根据切分点数组进行具体的chunk切分。
        - **shardutil::splitChunkAtMultiplePoints()**

            > <https://github.com/mongodb/mongo/blob/master/src/mongo/s/shard_util.h>
            > <https://github.com/mongodb/mongo/blob/master/src/mongo/s/shard_util.cpp>

            &emsp;&emsp;因为涉及到具体的 chunk 的切分，所以需要使用命令的形式，将切分命令发送到拥有这个chunk的分片上执行，执行的步骤类似上面介绍的自动切分的步骤，不过这个调用已经指定了切分点。请求发送后阻塞等待命令执行结果返回。
            &emsp;&emsp;这里规定了一个最大切分点数量**8192**，如果切分点数组大小超过这个数字将不执行切分而返回错误。
            &emsp;&emsp;之后会再次检查切分点数组中的切分点是否是 chunk 的边界，这里的检查主要是为了兼容一些老的代码(新的代码在调用前就已经检查了)。

    &emsp;&emsp;
    &emsp;&emsp;调用 _chunkSelectionPolicy->selectChunksToMove() 来选取需要迁移的 chunk 和确定它们的目标 shard 。
    - **_chunkSelectionPolicy->selectChunksToMove()**
        &emsp;&emsp;检查有多少分片，若只有一个分片就不需要选取了直接返回。
        &emsp;&emsp;获取所有 collections ，若没有 collection 也直接返回。
        &emsp;&emsp;先打乱顺序(这里打乱顺序是因为每个round实际上不会一次性执行完所有可能的迁移操作，所以为了不偏向于某个 collection ，需要乱序遍历)，再遍历每个 collection 。
        - **注意**
            &emsp;&emsp;在遍历开始之前，需要建立一个set **usedShards** ，这个set会用来记录本次round中所涉及到的分片，包括发送方和接收方。
            &emsp;&emsp;如果有分片已经作为了发送方或者接收方了，那么本次round就会忽略之后和其相关的操作。就是不会再给其派发迁移任务。

        &emsp;&emsp;
        &emsp;&emsp;对于每个 collection ，先检查该 collection 是否已经删除，然后检查该 collection 是否允许进行 balancing 。
        &emsp;&emsp;检查通过后，调用 _getMigrateCandidatesForCollection() 来选取迁移 chunk 。
        - **_getMigrateCandidatesForCollection()**
            &emsp;&emsp;获取相应 collection 的 ChunkManager，调用 createCollectionDistributionStatus() 获取该 collection 相关的 DistributionStatus 。
            &emsp;&emsp;遍历 DistributionStatus._zoneRanges 的键值对，检查是否有跨两个tag的chunk，如果有的话就暂不处理这个 collection 的 chunk 而直接返回(也就是说这个 collection 留到下一次round再处理，出现这种情况是用户可能在任何时刻添加range到tag上，可能正好在切分步骤结束之后，所以官方文档建议用户在自定义zone的时候暂时停用 balancer)。
            &emsp;&emsp;完全检查完毕之后，调用 BalancerPolicy::balance() 真正进入该 collection 的迁移chunk选取阶段，获取 MigrateInfo 数组。
            - **MigrateInfo**
                &emsp;&emsp;包含一个 chunk 的迁移相关参数的数据结构。
                - nss 所属 collection 的 namespace
                - to 接收分片 shardID
                - from 发送分片 shardID
                - minKey
                - maxKey
                - version 该 chunk 的 ChunkVersion
            - **BalancerPolicy::balance()**

                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/balancer_policy.h> 同样定义了 DistributionStatus ，MigrateInfo
                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/balancer_policy.cpp>

                1. 检查包含该collection数据的每个分片，如果分片处于 draining (表示分片正在被移出集群)，并且该分片的 shardID 还未被加入 usedShards 中，就按顺序遍历其上所包含的该collection的chunk，调用 _getLeastLoadedReceiverShard() 获取包含在tag范围内的chunk最少的分片作为接收方(不包括发送方自己)，将第一个符合条件的迁移 chunk 的MigrateInfo加入到结果数组中。然后将该次迁移的发送方和接收方的shardID都加入到 usedShards 中，跳出对该分片上chunk的循环。继续查看下一个分片。
                2. 检查所有chunk，按照每个分片来遍历，跳过已经加入 usedShards 的分片。选取不位于所属tag标记分片上的chunk，选取目标分片过程与 1. 相同。当选中第一个可以迁移到其他分片的chunk后，便将接收方和发送方分片的shardID 加入 usedShards ，退出当前对该分片上chunk的循环。继续查看下一个分片。
                3. 该步是针对每个tag内部分片间的平衡。这里有个小trick，在遍历之前向tag数组末尾插入了一个空串，当遍历到这个空串时获取到的chunk是全部的chunk，因为用户定义的tag一般不会覆盖全部的chunk，所以前面先优先操作包含在tag中的chunk，最后的空串保证了其他chunk也能参加 balancing 。遍历按照tag进行，先计算tag分片平分tag内chunk的个数 idealNumberOfChunksPerShardForTag 作为参照(当前tag总chunk个数/tag的分片数 四舍五入取整)。然后重复调用 _singleZoneBalance() 直到返回 false。
                    - **_singleZoneBalance()**
                        &emsp;&emsp;先调用 _getMostOverloadedShard() 获取不包含在 usedShards 中且包含属于该tag的chunk数量最多的分片(**注意**这里并不是统计包含的全部chunk)作为发送分片。
                        &emsp;&emsp;如果该分片拥有的属于该tag的chunk数量 max 不超过 idealNumberOfChunksPerShardForTag 则直接返回 false 。出现这种情况是因为本次round那些分片已经有迁移任务了。
                        &emsp;&emsp;然后调用 _getLeastLoadedReceiverShard() 获取负载最少属于该tag的分片，如果其上属于该tag的chunk数量 min 超过 idealNumberOfChunksPerShardForTag ，就不能作为目标分片了，此时直接返回 false 。
                        &emsp;&emsp;此时计算 max - idealNumberOfChunksPerShardForTag ，将结果与 kDefaultImbalanceThreshold(该值为1) 比较，如果小于则直接返回 false 。
                        &emsp;&emsp;然后从发送分片中遍历选取第一个不是 jumbo 并且属于当前指定tag的 chunk 来迁移到目标分片，并将发送分片和目标分片加入到 usedShards 中。

        &emsp;&emsp;
        &emsp;&emsp;收集所有collection的MigrateInfo数组，并返回。
    - **若有需要迁移的 chunk :**
        &emsp;&emsp;调用 _moveChunks() 执行对选中的 chunk 的迁移。
        - **_moveChunks()**
            &emsp;&emsp;在开始迁移之前会再次检查 balancer 是否被停止了或者是否超出运行窗口时间等。
            &emsp;&emsp;调用 _migrationManager.executeMigrationsForAutoBalance() 进行迁移处理。
            - **_migrationManager.executeMigrationsForAutoBalance()**

                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/migration_manager.h>
                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/balancer/migration_manager.cpp>

                &emsp;&emsp;遍历每个 MigrateInfo 。
                &emsp;&emsp;调用 ScopedMigrationRequest::writeMigration() 尝试将该次迁移插入到 config.migrations ，在函数中会检查 chunk 是否已经不在源节点上了，如果不在则插入失败。若插入失败则放弃该迁移。
                &emsp;&emsp;插入成功后，调用 _schedule() 以远程命令异步调用的方式进行chunk迁移，会返回一个 notification 。
                - **_schedule()**
                    &emsp;&emsp;先对 balancer 状态进行检查，可以正常运行再进行之后的操作。
                    &emsp;&emsp;检查 chunk 源分片的状态，不正常则返回。
                    &emsp;&emsp;调用 MoveChunkRequest::appendAsCommand() 构造 chunk 迁移命令。
                    &emsp;&emsp;再次对 balancer 状态进行检查。
                    &emsp;&emsp;调用重载的 _schedule() 进行真正的命令调用。
                    - **重载的_schedule()**
                        &emsp;&emsp;先查看是否已经有该 collection 的 chunk 迁移操作正在进行。通过查看 collection 的 namespace 是否在 MigrationManager::_activeMigrations 这个map中来确定，如果不存在则先申请一个该 collection 的分布式锁，该锁会存在 config.locks 中，然后插入一个键值对 \<nss: 迁移操作列表\>。
                        &emsp;&emsp;之后便可以向该 collection 的迁移操作列表中加入新的迁移操作。这个迁移操作会以远程命令请求的形式来进行，请求的目标是 chunk 的发送分片(下面会介绍)，并且定义回调函数 _complete() ，然后交给一个工作线程来发送请求并等待执行结果传回。执行结果传回时会执行回调函数。
                        - **_complete()**
                            &emsp;&emsp;从该 collection 的迁移操作列表中删除这个已经完成的迁移操作。如果这个 collection 的迁移操作列表变为了空列表，则释放这个 collection 的分布式锁。
                            &emsp;&emsp;将 notification 设置为请求完成的状态。
                    
                &emsp;&emsp;
                &emsp;&emsp;阻塞等待所有的请求的 notification 被设置，收集迁移请求结果，删除相应的 config.migrations 中的文档并返回。(也就是说当前round的所有 chunk 迁移是一同执行的，正常情况下等它们全部执行完成之后当前 round 才会结束)。

            &emsp;&emsp;
            &emsp;&emsp;对返回的迁移状态进行分析，若有迁移失败的情况则打印错误信息。
            &emsp;&emsp;若迁移失败错误是因为 chunk 过大而导致的，则调用 _splitOrMarkJumbo() 对 chunk进行切分，无法切分则标记为 jumbo 。这样做是因为能够成功进入到该函数的chunk在此之前是不会存在 jumbo 标记的。这种情况也会计入成功迁移 chunk 数量。
            - **_splitOrMarkJumbo()**
                &emsp;&emsp;首先会向 chunk 所在分片发起远程调用 splitVector() 的命令，获取 splitPoints ，如果切分点列表的大小不为0则调用 shardutil::splitChunkAtMultiplePoints() 去切分这个 chunk 。否则将会抛出错误。
                &emsp;&emsp;catch 到错误之后，会将该 chunk 标记为 jumbo 。jumbo 的标记只会存在于路由缓存中。是 balancer 在不断执行的过程中慢慢发现并标记的。标记 jumbo 的目的是为了加快 balancer 的处理，防止 balancer 浪费时间去处理无法迁移的 chunk 。jumbo chunk 对本地存储并没有什么影响，会产生的影响只是无法迁移 jumbo chunk 范围内的数据而已。若调整了 chunk size 之后，自动切分过程可能会将原来标记为 jumbo 的 chunk 切分，这样 jumbo 标记也就消失了，因为产生的新的 chunk 缓存结构是不带 jumbo 标记的。

            &emsp;&emsp;
            &emsp;&emsp;返回总共成功迁移的 chunk 数量。

        &emsp;&emsp;
        &emsp;&emsp;将执行信息记录到 BalanceRoundDetails ，然后输出到日志中。
        &emsp;&emsp;结束本次round，若成功进行了 chunk 迁移或者 chunk 因为太大而没有完成迁移(即_moveChunks()返回结果大于0)，都只休眠1s 。这里休眠时间较短是因为每一次round的并不会一定完成所有迁移，因为 usedShards ，如果一个分片参与了某个 chunk 的迁移，那么另一个 chunk 可能也需要迁移并且是该分片参与的，就只能等到下一个round了，所以当前round有处理迁移时，说明很可能还有其他需要进行的迁移没有做，所需需要快速开始下一个round。
        &emsp;&emsp;否则休眠10s 。
    - **若无需要迁移的 chunk :**
        &emsp;&emsp;结束本次round，休眠10s 。
    
    &emsp;&emsp;
    &emsp;&emsp;若该round中发生了任何错误，则记录错误信息到 BalanceRoundDetails ，然后输出到日志中。结束本次round，休眠10s 。

#### 2.3 chunk 迁移算法

#### **2.3.1 算法简述**

&emsp;&emsp;mongodb 所采取的数据迁移协议是拉式的，所谓的拉式迁移，是接收方主动以查询请求的方式从发送方获取数据。，但是考虑到数据迁移过程中各种可能导致失败和取消的情况，这种拉式的迁移协议并不是在一开始就将请求重新路由到接收方，而是先保留发送方的数据，当数据完全迁移到接收方后再更新路由表。因为如果一旦数据迁移取消或者失败，那么接收方还得把已经接收的数据更改回写到发送方，控制上会比较麻烦。
&emsp;&emsp;mongodb 的数据迁移协议还有一个考量是尽量减小 chunk 迁移对于正常数据操作的影响，所以 mongodb 选择分阶段来进行发送方和接收方的数据同步，先让接收方追赶发送方的数据更新，在达到一个比较一致的状态的时候发送方再进入不接受写入的状态，以此来减少影响。
&emsp;&emsp;因为代码实现的问题，分片上很多模块类实例是独一份的，所以当前 mongodb 的分片同一时刻只能参与到一个迁移任务中。不过相信 mongodb 在以后会实现同一分片同时参与多个迁移任务的迁移模式。
- mongodb 的 chunk 迁移协议大体分为四个阶段:
    1. 启动阶段。source(chunk 所在分片) 向 destination(chunk 目标分片) 发送 _recvChunkStart 命令，通知 destination 要开始迁移某个 chunk 。
    2. 复制阶段。destination 不停向 source 发送 _migrateClone 命令，source 根据该命令会返回相应数据。然后 destination 将数据插入到自己的 collection 中。一旦有了这些基础数据之后，destination 开始不停向 source 发送 _transferMods 命令来拉取 chunk 范围内 oplog ，将数据更新写入到自己的 collection 中。直到某一时刻拉取到的 oplog 为空，此时 destination 进入 steady 状态，等待 source 的 _recvChunkCommit 命令。在此阶段中 source 也不停向 destination 发送 _recvChunkStatus 命令来获取 destination 的状态。
    3. 最终同步阶段。一旦 source 得知 destination 进入了 steady 状态，就会进入 critical section 状态，在该状态下 source 将不再接受该 collection 的更新操作。进入到 critical section 后，source 向 destination 发送 _recvChunkCommit 命令。destination 会最后再次进行 _transferMods 命令发送以拉取 source 从得知 destination 进入 steady 状态到进入 critical section 间接收的 chunk 范围内的 oplog 。最终 source 和 destination 在 chunk 内数据达到一致状态。
    4. 更新元数据(路由)阶段。source 在最终一致后会向 config server 发送 _configsvrCommitChunkMigration 命令以更新 config server 上的权威元数据。source 自己也会更新路由数据缓存。并在更新完元数据后退出 critical section 状态，可以看到其实 critical section 状态的时间是很短的，所以用户基本感知不到。而 destination 上的路由缓存并不会立刻更新，而是惰性更新，因为接受到 router 的请求之后(带有 version 信息)，destination 就知道自己的路由缓存需要更新并去更新了。

&emsp;&emsp;source 接收到的所有迁移命令都是由 balancer 发起的，在向 balancer 返回结果前还会根据 balancer 设置来进行已迁移 chunk 数据的删除(默认会删除，但是DBA可以修改设置使其不删除)。

#### **2.3.2 基于源码的详细分析**

- **MoveChunkCommand**

    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/move_chunk_command.cpp>

    &emsp;&emsp;MoveChunkCommand 是 mongodb 内部真正执行 chunk 迁移命令的执行入口定义类。接收到该命令的是 chunk 的发送分片的primary(复制集)。该命令将被发送分片执行，并且需要目标分片和config server参与配合。
- **MoveChunkCommand::run()**
    &emsp;&emsp;该函数是整个迁移任务的运行入口。
    &emsp;&emsp;函数刚开始会先检查分片运行状态和更新操作上下文信息，解析迁移请求的参数。
    &emsp;&emsp;调用 ActiveMigrationsRegistry::registerDonateChunk() 函数进行当前分片是否有参与到迁移任务中的检查。
    - **ActiveMigrationsRegistry::registerDonateChunk()**

        > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/active_migrations_registry.h> 同时也定义了 ScopedDonateChunk
        > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/active_migrations_registry.cpp>

        &emsp;&emsp;如果当前分片正在作为某个迁移 chunk 的接收分片，则返回错误。
        &emsp;&emsp;如果当前分片正在作为发送分片，并且执行的迁移请求的参数与当前传入的迁移请求参数相同，说明这个迁移任务已经在执行了，那么就返回一个 ScopedDonateChunk(nullptr, false, 用于设置迁移任务完成的 notification ) 实例。若是不同的迁移请求，则返回错误。
        &emsp;&emsp;如果当前分片没有正在执行的迁移任务，那么就让 ActiveMigrationsRegistry 记录下当前请求(用于给其他迁移任务引发的run的检查)，然后返回一个 ScopedDonateChunk(this, true, 用于设置迁移任务完成的 notification )。
    - **ScopedDonateChunk**
        _registry
        &emsp;&emsp;记录产生它的 ActiveMigrationsRegistry 指针，如果已经有相同请求的迁移任务在执行，则记录的是空指针。
        _shouldExecute
        &emsp;&emsp;是否该发起迁移执行的标志位。
        _completionNotification
        &emsp;&emsp;这个参数可以用于设置完成信号。
    
    &emsp;&emsp;
    &emsp;&emsp;根据返回的 ScopedDonateChunk ，检查其 _shouldExecute 标志位。
    &emsp;&emsp;若为 true 则运行 _runImpl() 函数执行迁移，若正确执行完迁移则调用 ScopedDonateChunk::signalComplete() 来设置 _completionNotification 用以通知其他正在等待其完成的run。
    若为 false 则运行 ScopedDonateChunk::waitForCompletion() 等待正在执行的相同迁移任务完成。
    - **_runImpl()**
        &emsp;&emsp;首先需要获取到 chunk 接收分片的 host 。
        &emsp;&emsp;迁移一个 chunk 一共分为5个步骤(忽略第一步创建记录器的步骤)。
        1. 创建一个 MigrationSourceManager 实例，在构造函数中会对迁移前的各种数据和分片状态进行检查。
            - **MigrationSourceManager()**

                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/migration_source_manager.h>
                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/migration_source_manager.cpp>

                _args 即此次迁移请求的参数
                _donorConnStr 当前分片的连接字串
                _recipientHost 接收分片的host
                _stats 分片状态及设置信息
                _collectionEpoch 记录迁移前的元数据快照的 collectionVersion 的 epoch
                _collectionUuid 记录元数据快照的 uuid
                _state 记录当前的运行阶段，初始为 kCreated
                _cloneAndCommitTimer 
                _cloneDriver 
                &emsp;&emsp;以上是 MigrationSourceManager 的成员变量。
                &emsp;&emsp;构造函数在初始化参数列表赋值前四项成员变量之后，先检查 _args 参数中不会出现 from == to 的情况。
                &emsp;&emsp;然后调用 forceShardFilteringMetadataRefresh() 刷新分片的元数据缓存。
                &emsp;&emsp;然后尝试创建一个迁移 chunk 所在 collection 的当前分片元数据快照，创建快照时一边检查 collection 的分片状态是否正常(还存在，元数据缓存正常，可分片状态)。
                &emsp;&emsp;从快照中获取当前分片的 shardVersion 和 collectionVersion ，检查 shardVersion 的 major 是否为 0 ，为 0 说明当前分片并没有该 collection 的 chunk 存在，所以需要返回错误。检查 collectionVersion 的 epoch 是否与 _args 中的一致，不一致说明当前collection可能已经被删除，也需要返回错误。
                &emsp;&emsp;还需要检查是否能够通过 _args 的 chunk 边界确定 chunk ，如果边界不匹配也需要返回错误。
                &emsp;&emsp;检查完成后赋值 _collectionEpoch 和 _collectionUuid 。

        2. 调用实例的 MigrationSourceManager::startClone() 函数，该函数主要进行了 _cloneDriver 成员变量的创建并将 _state 变为 kCloning，(_cloneDriver 是一个 MigrationChunkClonerSourceLegacy 实例）。然后调用 _cloneDriver->startClone() 向接收分片发送一个 StartChunkCloneRequest 命令，通知接收分片要开始向其复制数据了。StartChunkCloneRequest 命令处理类为 RecvChunkStartCommand (定义于 src/mongo/db/s/migration_destination_manager_legacy_commands.cpp) ，当接收分片收到该命令时会执行 RecvChunkStartCommand::errmsgRun() 。

            > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/migration_chunk_cloner_source_legacy.h> _cloneDriver 定义处
            > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/migration_chunk_cloner_source_legacy.cpp>

            - **RecvChunkStartCommand::errmsgRun()**

                > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/migration_destination_manager_legacy_commands.cpp>

                &emsp;&emsp;接收方先解析命令的相关参数。然后刷新自己的元数据缓存。
                &emsp;&emsp;调用 ActiveMigrationsRegistry::registerReceiveChunk() 来检查自身是否已经在接收 chunk 或者作为发送方发送 chunk 。如果是就返回错误。
                &emsp;&emsp;确保当前分片是空闲的之后，便调用当前分片的 MigrationDestinationManager::start() 函数重置 **MigrationDestinationManager** 实例的成员变量，在函数最后会在一个子线程中开始执行 MigrationDestinationManager::_migrateThread() 。然后 start() 返回，并回复命令调用成功。
                &emsp;&emsp;在 _migrateThread() 中，调用 _migrateDriver() ，如果在迁移的中间阶段失败了，那么需要调用 _forgetPending() 清除已经接收的数据。
                - **MigrationDestinationManager::_migrateDriver()**

                    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/migration_destination_manager.h>
                    > <https://github.com/mongodb/mongo/blob/master/src/mongo/db/s/migration_destination_manager.cpp>

                    1. 调用 cloneCollectionIndexesAndOptions() 进行索引和设置的复制如果这个 collection 在目标分片不存在的话。这一步骤中会获取 X 锁来操作，所以此时无法进行删除这个 collection 的操作。
                    2. 调用 _notePending() 删除本地残留的在将要迁移进入的 chunk 内的数据，如果数据还在被使用则返回错误(这种情况是可能发生的，比如一个 chunk 刚从这个分片迁移走，然后马上又迁移回来了，而这个分片剩余的还在使用相关数据的操作就会引发这个错误)。
                    3. 开启一个子进程来启动迁移会话来传递状态，调用 cloneDocumentsFromDonor() 向 chunk 发送方不停发起 _migrateClone 向 chunk 发送方拉取 chunk 范围内的数据，然后插入本地的 collection 中。
                    4. 在完成一圈基本的复制之后，接收方会开始进行迁移过程中发生的数据更新的复制，它会在一个循环中不停向 chunk 发送分片发送 _transferMods 命令，拉取修改相应的操作列表(oplog)，然后调用 _applyMigrateOp() 本地实施这些修改。直到某一时刻这个列表返回为空或者超过了循环上限(超过循环上限会返回错误)。之后在进入第五步之前还需要等待复制集更新。
                    5. 将当前状态设置为 steady 。并在接受到 _recvChunkCommit 进入到 commit_start 的状态。在循环中向发送方发送 _transferMods 命令，拉取最后剩余的范围内 oplog 。若多次拉取之后更改列表为 0 ，便调用 _flushPendingWrites() 检查当前是否已经将操作都写入 collection 中，成功完成后跳出循环。

        3. 调用实例的 MigrationSourceManager::awaitToCatchUp() 函数，该函数调用了 _cloneDriver->awaitUntilCriticalSectionIsAppropriate() 函数。
            - **_cloneDriver->awaitUntilCriticalSectionIsAppropriate()**
                &emsp;&emsp;在该函数中，发送分片会不断向接收分片发送 kRecvChunkStatus 命令来确认当前 chunk 数据的接收状态。最多等待 maxTimeToWait(6s) 的时间。
                &emsp;&emsp;每次循环中若是收到的回复 waited 字段是 true 的话就会休眠 min(当前循环次数, 10) 毫秒。每次循环还会去获取当前 chunk 剩余的文档数量，如果接收方认为已经完成接收 (进入steady状态) 而 chunk 剩余文档数量不为 0 ，则返回错误。接收到错误状态的响应也会返回错误。
                &emsp;&emsp;对于每一个回复还要检查获取迁移的会话ID，并且检查回复的 chunk 相关描述与当前正在迁移的 chunk 是否相同(有时接收方已经放弃了接收这个分片的 chunk 而开始接收其他分片的 chunk 时，当前分片还处于这个阶段，并未意识到，所以需要检查)。
                &emsp;&emsp;这里还有一个内存占用的检查，如果迁移导致的内存占用超过了 500 MB 时，会放弃该迁移。
                &emsp;&emsp;最终在一切正常并且接收方成功进入 steady 状态时会跳出循环。

        4. 调用 MigrationSourceManager::enterCriticalSection() 函数进入到 critical section 状态，设置 _state = kCriticalSection ，该状态下会使得当前分片不再继续接收该 collection 的写操作。
        &emsp;&emsp;完成后调用 MigrationSourceManager::commitChunkOnRecipient() 函数。该函数会调用 _cloneDriver->commitClone() 向接收方发送 _recvChunkCommit 命令，这个命令会让接收方最后拉取从其进入 steady 到发送方进入 critical section 状态间发生的相关 chunk 内的写操作。成功发送后设置 _state = kCloneCompleted 。

        5. 调用 MigrationSourceManager::commitChunkMetadataOnConfig() 函数来最终通知 config server 更新权威元数据。向 config server 发送 _configsvrCommitChunkMigration 还会先阻塞相关 collection 的读操作，发送的更新请求会增加 config server 关于发送方本地所包含相关的 collectionVersion 的 major ，使得其他操作发送者会去更新路由表。
        &emsp;&emsp;然后本地也调用 forceShardFilteringMetadataRefresh() 更新路由信息和元数据。在更新完元数据并检查之后会退出 critical section 。

## **三. 实验设想**

&emsp;&emsp;可以使用 zone 来完全控制数据分布。在实验数据的选取上，我们让 shard key 值具有唯一性，这样我们就能根据 chunk 的 range 直接确定 chunk 有多少文档了，操作起来也比较方便。一般应用场景中 shard key 也会尽量选取唯一值，所以也比较贴近实际。这样我们能够估算 chunk 文档个数之后，就可以通过查看 config.chunks 之类的元数据 collection 知道数据分布信息，并在需要时发起 mergeChunks 命令合并在同一分片下的邻近并且文档数量少的 chunk ，这样可以减少元数据，方便管理和操作。
&emsp;&emsp;而且可以在源码上修改 balancer 每次的休眠间隔来加快或者减慢 balancer 的运行。

> <https://docs.mongodb.com/manual/tutorial/merge-chunks-in-sharded-cluster/> mergeChunks 命令的官方介绍
