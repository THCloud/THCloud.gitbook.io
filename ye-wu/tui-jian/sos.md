# sos

### 单机存储 <a href="#dan-ji-cun-chu" id="dan-ji-cun-chu"></a>

#### **高速裸盘存储dstorage** <a href="#gao-su-luo-pan-cun-chu-dstorage" id="gao-su-luo-pan-cun-chu-dstorage"></a>

对于单机存储，目前很多存储系统都用开源的文件存储系统：leveldb，rocksdb等等。sos绕过了文件系统，选择直接操作磁盘，在裸盘上构建了一个单机db。首先在每一块盘上建立一个db，起个名字叫做dstorage，构建的时候需要对这个盘进行格式化，主要是为了设置内部基本数据大小，分块大小等信息。为了方便数据管理，默认以64M为一块进行内存分配、回写磁盘管理的基本单位（名为chunk），同一个块都用来存同一种大小的数据。在数据写入的时候，会根据块的情况来决定是否进行回写。如果一次写操作之后导致当前的块需要回写了，则会触发一次块的磁盘回写。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=8667232)

因为数据是变长的，也是为了方便管理，sos会给出某些固定大小的step，数据会从小到大找到第一个大于等于自身大小的step，存到这个step当前分配出到内存里的块上。

对于db层来说，一次操作的key是要在磁盘上放的位置，value是用户端看到的key和value和时间戳等信息。具体格式为：\<offset,key+timestamp+value>

&#x20;

#### 数据索引——哈希 <a href="#shu-ju-suo-yin-ha-xi" id="shu-ju-suo-yin-ha-xi"></a>

一般来说数据的索引常见的方式有几种：

1.哈希表：它很分散，检索快，但是无序；

2.Rbtree：检索log(n)，可以根据树的特性进行一些有序查找。但是树最不好的一点就是不好加锁。

3.跳表：效率比树高，但是需要额外维护多个链表，用空间换时间。加锁粒度也是一个问题。

因为sos对上层是不提供range查找的，所以围绕着根本原则：快，最终选择了使用哈希表。加锁方式是使用分段加锁，细化锁的粒度，从而提高并发能力。因为对索引的修改是很快的，所以不用担心锁冲突会耗时很高。

哈希表中存储着一个key对应的数据在磁盘中的位置。

&#x20;

#### **索引表落盘——mmap** <a href="#suo-yin-biao-luo-pan-mmap" id="suo-yin-biao-luo-pan-mmap"></a>

索引表也是需要落盘的，一般的做法我们会mmap进内存，这样我们重启的时候一般就不用把这个整个索引表文件都load一遍，极大地提升了重启的可用性。落盘是根据配置的回写时间，把索引表的脏页回写到磁盘进行持久化。一般来说都是额外有个线程在做这个事情，不会占用读写线程。

mmap 即 memory map，也就是内存映射。mmap 是一种内存映射文件的方法，即将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必再调用 read、write 等系统调用函数。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以实现不同进程间的文件共享。

&#x20;

#### **读cache设计** <a href="#du-cache-she-ji" id="du-cache-she-ji"></a>

使用LRU策略缓存了热点kv，写和删都会先过一遍cache，读热点数据可以在10ns内完成。

LRUCache的结构是rbtree，可以保证log(n)判断数据是否在cache中，查询的时候是通过key的md5来查询rbtree判断是否存在。如果存在直接返回，不存在的话再通过哈希表查询

![](https://iwiki.woa.com/download/attachments/2400464482/image2023-1-30_18-40-58.png?version=1\&modificationDate=1675075259000\&api=v2)

&#x20;

### 多机存储一致性保证——chain replication算法 <a href="#duo-ji-cun-chu-yi-zhi-xing-bao-zheng-chainreplication-suan-fa" id="duo-ji-cun-chu-yi-zhi-xing-bao-zheng-chainreplication-suan-fa"></a>

#### **多机存储方案** <a href="#duo-ji-cun-chu-fang-an" id="duo-ji-cun-chu-fang-an"></a>

多机器存储需要多备份冗余，假设一个数据想要存3份，很多古老的系统是3台机器做机器镜像。这种方式不够高效和优美，因为如果一台机器挂了，就只能从另外2台恢复。

这里采用的是另一种多机存储方案：如果集群有很多很多台机器（假设是m台，m一般来说远大于副本数），可以进行切分操作，把数据均匀打散，我们这里称作段。系统初始化的时候会配好x个段，那么x\*n备份个数据段就可以均匀打散到m台机器上，如果有一台机器需要数据恢复，其余有这个机器拥有的段的机器都可以往他进行数据恢复。所以需要x要远远大于m，就可以尽量让数据分散开了。

使用哈希方式切分，可以快速计算出一个数据属于哪个段，然后找到这个段的n个副本在哪些机器上。

&#x20;

#### chain replication算法介绍 <a href="#chainreplication-suan-fa-jie-shao" id="chainreplication-suan-fa-jie-shao"></a>

对于数据的一致性保证，业界有很多共识算法，比如paxos， raft，multi-raft等。但是这些强一致性的协议性能上不能满足平均单机两三千的写入qps。最终，sos采用的是chain replication算法。

CR算法（chain replication）基于P/B模型：\
  P/B（Primary/Backup）是最出名的分布式一致性解决方案，P/B拥有一个基础服务器和N个备份服务器。如图所示是一个P/B模型的工作流程。其中q1是基础服务器，接收和相应用户的请求。而q2和q3作为备份服务器，会根据q1的更新命令进行数据的更新存储。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=8667228)

如下所示为一个链复制基本模型。当头部节点接收到请求后，会解析请求并执行，然后顺链路传递直至尾结点。当尾节点接收到后，会逆向传递ACK返回至头结点，从而保证了数据的一致性。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=8667227)

链复制拥有这以下特点

* 由一系列服务器组成链表
* 首节点接收用户写请求，尾节点接收并返回用户的读请求
* 要求节点间保证健壮可靠的FIFO通信
* 可以容忍最多n-1节点的失败
* 写性能和P/B类似
* 首部节点的故障易于修复重置，快于P/B，其他节点则速度类似

&#x20;

#### **节点视角的状态转换** <a href="#jie-dian-shi-jiao-de-zhuang-tai-zhuan-huan" id="jie-dian-shi-jiao-de-zhuang-tai-zhuan-huan"></a>

对于每个节点i，我们需要存放三种数据：1.Pending(i)：尾结点尚未处理的收到的请求  2.History(i, key)：key值变化的链表，可存放完整历史或者仅存放当次修改

下图所示为节点视角的伪代码处理逻辑:

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=8667233)

我们通过说明object状态及状态转换，详细描述存储服务功能，如上图所示，objID的状态用两个变量描述：History(objID) 表示Update操作序号，Pending(objID)表示待执行的请求列表。objID所有可能的状态表示：

• T1表示新的请求到达，添加到 Pending(objID) 列表；

• T2表示部分Pending请求被忽略（很少发生）；

• T3表示请求r处理过程，首先从 Pending(objID) 列表中移除，若为Query操作，根据 History(objID)生成reply。若为Update操作，更新 History(objID) 。

&#x20;

#### **fail-stop故障恢复** <a href="#failstop-gu-zhang-hui-fu" id="failstop-gu-zhang-hui-fu"></a>

存储服务fail-stop假设：

• 发生故障时，停止服务，而不是转换为不确定的状态；

• 可以检测到服务停止状态；

• 对于object复制存储&#x5230;_&#x74;_&#x4E2A;servers的情况，至多只&#x6709;_&#x74;-&#x31;_&#x4E2A;server同时故障；

&#x20;

为了支持服务故障检测、恢复，引入master服务，主要负责：

• 检测服务是否故障，并删除故障服务；

• 通知复制链中的每个server，其新的上游和下游server；

• 通知Clients，新的HEAD server和TAIL server；

为了简化描述，假设master服务以单进程形式存在，且永远不会发生故障。实际上，master服务实现为集群形式，采用Paxos算法保证集群服务一致性。

Master将服务故障分为三类：i）HEAD server故障；ii）TAIL server故障；iii）其他server故障，并对每一类故障单独处理。

i)  HEAD server故障

Master直接删除已故障的HEAD server H，并选择其下游server作为新的HEAD server H。

Query操作不受影响。Update操作短暂中断，直到Master完成以下操作后恢复：(1)通知下游server作为新的HEAD  (2)通知所有Clients新的HEAD；

ii）TAIL server故障

Master直接删除已故障的TAIL server T，并选择其上游server T-1 作为新的TAIL server T

服务短暂中断，直到Master完成以下操作后恢复：(1)通知上游server作为新的TAIL (2)通知所有Clients新的TAIL；

_iii）其他server故障_

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=8667226)

Master直接删除故障server S，然后先后通知其下游server S+ 和上游server S− 。该服务状态转变可能会导致Update传播无效，需要采取某种措施，保证server S故障前已接收的Update请求能够继续传播。最理想的是选择server S− 继续传播这些请求。server S− 连接到server S+ 之后，首先需要传递那些 S+ 还未收到的请求，完成之后再按照正常流程传递后续Update请求。

Query操作不受影响。Update操作时延增加，但业务不中断，故障server的上游server仍可以继续接收处理请求。

&#x20;

删除故障server之后，需要添加新server，以保证期望的副本冗余。理论上，可以从链表任意位置出入新server，但从链表尾部添加最简单。

&#x20;

在故障server恢复并被写入期间重试的数据之前，可以认为数据不是一致的，我们把这种保证最终一致性的做法也叫做弱一致性。

在sos中，如果某个目标机器最终超过了重试时间，那么就不会再继续重试。如果这个时候它又活过来了，那么我们的数据就会不一致，所以，在停止重试之前，如果它还没活过来，就需要分布式系统在集群的服务层面把它摘除。

&#x20;

### 集群管理——configserver <a href="#ji-qun-guan-li-configserver" id="ji-qun-guan-li-configserver"></a>

![](https://iwiki.woa.com/download/attachments/2400464482/image2023-1-30_18-20-58.png?version=1\&modificationDate=1675074059000\&api=v2\&width=0)

#### 路由表 <a href="#lu-you-biao" id="lu-you-biao"></a>

config server存储使用的路由表，结构示意如下（用三副本集群举例）：

![](https://iwiki.woa.com/download/attachments/2400464482/image2023-1-30_18-41-17.png?version=1\&modificationDate=1675075278000\&api=v2)

**完整读数据流程：**

对数据进行分桶操作后，路由表中存储着dataserver映射信息，可以依据key的md5计算出应该属于路由表中的哪一行，然后根据表格中保存的dataserver映射信息，就可以找出对应的数据的三副本存在于哪个dataserver了。

在对应的dataserver中，首先会在LRUCache中查询这条数据是否存在。LRUCache的结构是rbtree，可以保证log(n)判断数据是否在cache中，查询的时候是通过key的md5来查询rbtree判断是否存在。如果存在直接返回，不存在的话再通过哈希表查询，进而拿到一个30字节存储的包含在磁盘中的偏移+时间戳+业务编号等信息的内容，这样就可以去对应的dataserver去查询对应的数据。

&#x20;

**完整写数据流程：**

首先会key的md5计算出应该属于路由表中的哪一行，然后将数据写入对应的dataserver中，这样是为了尽量均匀的将数据写入各个桶。

接着判断哈希表中是否有这个数据，如果没有的话就可以直接写，然后更新哈希表中这个数据在磁盘中的位置，同时判断LRUCache是否存在，不存在就更新。

如果有数据，就会给哈希表加锁，然后在磁盘中的新的位置去写数据，最后更新哈希表指向新地址，同时判断LRUCache是否存在，不存在就更新。

&#x20;

Configserver不需要每次请求都被访问，只需要客户端要拿路由表的时候访问一次，把路由表存到客户端的内存里，就能知道要访问哪个数据段的那个dataserver了。直到路由表版本号过旧，再去重新获取一次。

#### 集群状态管理 <a href="#ji-qun-zhuang-tai-guan-li" id="ji-qun-zhuang-tai-guan-li"></a>

1.正常状态，对dataserver进行心跳检测，心跳设置的长短根据集群具体业务需求来决定。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=8667230)

2.当有dataserver挂掉时，首先保证主段数据，因为一般是直接从主副本copy1进行读取的。dataserver3挂掉后dataserver6顶替了主副本。

此时会认为当前的0数据段和1数据段，都只有两个副本，那么写入的时候就是往两台写。dataserver3挂掉并被踢除之后，依然高可用并保证剩下两个副本的一致性。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=8667229)

3.进行数据修复，先给这个机器分配数据段，然后有它要的段的源机器，都会收到一个命令，就开始往dataserver7这个目标机器进行迁移。期间所有服务都是正常的。扩容的原理也是类似的。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=8667231)

### 服务相关 <a href="#fu-wu-xiang-guan" id="fu-wu-xiang-guan"></a>

Client： nginx模块：兼容http协议； lua代码扩展mget功能； 带宽/qps限制模块，进行用户层级的资源隔离；

sosclient：与后端多台机器的复合交互，使用workflow实现（[https://github.com/sogou/workflow](https://github.com/sogou/workflow)）；

qdb2sos\_proxy：兼容QDB协议；







#### SOS的框架： <a href="#sos-de-kuang-jia" id="sos-de-kuang-jia"></a>

&#x20;

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=12615307)

&#x20;

#### 读数据流程： <a href="#du-shu-ju-liu-cheng" id="du-shu-ju-liu-cheng"></a>

弹幕使用SOS client方式调用，通过SOS侧nginx访问到config server。

config server存储了router table这个表的结构示意如下：

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=12615310)

会依据md5(key)计算出应该属于router table表中的哪一行，router table存储了ds映射信息，例如三副本集群，router table是一个16384\*3副本记录的表格，这里是提前对存储做了分桶操作，分成了16384个桶。表格中保存了对应的弹幕数据的三副本存在在哪个桶里边。，然后就可以判断出要找的ds是哪个。

&#x20;

然后到了对应的ds，cache中如果不存在，就会通过md5的其他位来判断出hash table，进而拿到一个30字节存储的包含在磁盘中的偏移+时间戳+业务编号等信息的内容，依据便宜就可以只读一次磁盘获取到对应的数据，读磁盘之前会读一下page cache看看是否在page cache中，如果不在就会去磁盘读。

同理，如果这个key不存在，那么hash table处就可以判断出来，然后返回，不需要再去读磁盘。

&#x20;

这样就可以获取到一个30字节的指向具体存储的字段，字段包含了地址+时间戳（包含过期时间ttl信息）+服务id（例如弹幕是2这样）等信息，这样就可以去对应的ds去查询对应的弹幕。

&#x20;

ds的结构如下图：

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=12615309)

定位到具体的ds之后，首先会在LRUCache中查询这个弹幕是否存在，如果存在直接返回，不存在的话再通过hash table查询。LRUCache的结构是rbtree，可以保证log(n)判断弹幕是否在cache中。查询的时候是通过key的md5来查询rbtree判断是否存在，对于弹幕的几十亿量级来说，可以认为md5是不会重复的。

不在cache中的通过hash table来找到他对应的磁盘以及offset，在访问磁盘前，还有一层page cache。

page cache会按照固定大小来写回磁盘，比如：4k。所以可能磁盘的第0-4k、12-16k这两块大小，被page cache住，那么从这里读就行。那么假设我的offset是3k，而0-4k在page cache里，就从page cache里读。

page cache中不存在，再去磁盘中取数据。这里要注意的是，写数据的时候会对hash table分段加锁，读数据的时候如果发现获取不到锁就要等待，拿到锁之后再去读数据。

&#x20;

这样的好处是，至多一次磁盘IO就可以取到数据。

取数据一致性的问题：读数据一定是从主副本先读，读不到再读其他副本。

&#x20;

&#x20;

#### 写数据流程： <a href="#xie-shu-ju-liu-cheng" id="xie-shu-ju-liu-cheng"></a>

首先会md5(key)计算出应该属于router table表中的哪一行，然后就可以判断出要找的ds是哪个，这样是为了尽量均匀的将数据写入各个桶。

&#x20;

会先判断hash表中是否有这个数据，如果没有的话就可以直接写，然后更新hash table这个数据在磁盘中的位置，同时判断lrucache是否存在，不存在就更新。

&#x20;

如果有数据了，就会给hash table加锁，然后在磁盘中的新的位置去写数据，然后更新hash table指向新地址，同时判断lrucache是否存在，不存在就更新。

&#x20;

写数据也是先写主副本，然后链式写入其他副本，最后commit回主副本。这里最多允许一个副本出现问题，算写会成功。

&#x20;

&#x20;

#### 节点异常处理流程： <a href="#jie-dian-yi-chang-chu-li-liu-cheng" id="jie-dian-yi-chang-chu-li-liu-cheng"></a>

&#x20;

当一个节点出现问题的时候，比如config server和节点的心跳检测发现一段时间访问不到这个节点了。就会摘掉这个节点，这个时候会有一些数据变成了只有2副本的情况。

如图：

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=12615312)![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=12615311)

此时，对于表格中的第一条数据，副本2就会变空，此时还有2副本，并不影响数据读取。对于第二行的数据，ds6就会变成主副本。

&#x20;

一段时间后，加入了好的集群之后，服务会计算出哪些数据收到了影响，需要从哪些ds节点同步数据到新的ds7，同步完成后，再更新router table，恢复3副本的状态。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=12615313)

&#x20;

这样虽然是三副本的集群，比如弹幕是15台机器，单台机器压测出550wQPS的读上限，实际集群的读上限就是15\*550wQPS，而不是15/3\*550wQPS。

&#x20;

对于磁盘，会分成若干个64兆的chunk，每个chunk再等分成若个个1k或者512B或者其他大小的

更新的时候，比如弹幕大小是900B，首先看page cache中是否有符合1KB大小条件的chunk在，存在的话就写入，等待后续批量异步刷回磁盘，由于chunk是一个一个64兆大小的，所以写入chunk就可以确定磁盘中的offset，就可以更新hash table了。

如果page cache中没有符合条件大小的没有满的chunk，然后取出，不管在chunk最后写入还是插空，然后再更新会磁盘。

磁盘的header会保存chunk的相关信息，包括chunk分成了多大的，offset是多少等。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=12615314)

&#x20;
