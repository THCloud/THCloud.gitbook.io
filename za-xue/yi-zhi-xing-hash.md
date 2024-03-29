# 一致性hash

### 一致性hash <a href="#yi-zhi-xing-hash" id="yi-zhi-xing-hash"></a>

好的一致性算法应该有什么特性，有一段讲述的非常准确，基本完全match线上生产环境了：

> **平衡性(Balance)**\
> 平衡性是指哈希的结果能够尽可能分布到所有的缓冲中去，这样可以使得所有的缓冲空间都得到利用。很多哈希算法都能够满足这一条件。
>
> **单调性(Monotonicity)**\
> 单调性是指如果已经有一些内容通过哈希分派到了相应的缓冲中，又有新的缓冲区加入到系统中，那么哈希的结果应能够保证原有已分配的内容可以被映射到新的缓冲区中去，而不会被映射到旧的缓冲集合中的其他缓冲区。简单的哈希算法往往不能满足单调性的要求，如最简单的线性哈希：x = (ax + b) mod (P)，在上式中，P表示全部缓冲的大小。不难看出，当缓冲大小发生变化时(从P1到P2)，原来所有的哈希结果均会发生变化，从而不满足单调性的要求。哈希结果的变化意味着当缓冲空间发生变化时，所有的映射关系需要在系统内全部更新。而在P2P系统内，缓冲的变化等价于Peer加入或退出系统，这一情况在P2P系统中会频繁发生，因此会带来极大计算和传输负荷。单调性就是要求哈希算法能够应对这种情况。
>
> **分散性(Spread)**\
> 在分布式环境中，终端有可能看不到所有的缓冲，而是只能看到其中的一部分。当终端希望通过哈希过程将内容映射到缓冲上时，由于不同终端所见的缓冲范围有可能不同，从而导致哈希的结果不一致，最终的结果是相同的内容被不同的终端映射到不同的缓冲区中。这种情况显然是应该避免的，因为它导致相同内容被存储到不同缓冲中去，降低了系统存储的效率。分散性的定义就是上述情况发生的严重程度。好的哈希算法应能够尽量避免不一致的情况发生，也就是尽量降低分散性。
>
> **负载(Load)**\
> 负载问题实际上是从另一个角度看待分散性问题。既然不同的终端可能将相同的内容映射到不同的缓冲区中，那么对于一个特定的缓冲区而言，也可能被不同的用户映射为不同的内容。与分散性一样，这种情况也是应当避免的，因此好的哈希算法应能够尽量降低缓冲的负荷。
>
> **平滑性(Smoothness)**\
> 平滑性是指缓存服务器的数目平滑改变和缓存对象的平滑改变是一致的。

#### consistent hashing

[一致性hash原理](https://www.cnblogs.com/lpfuture/p/5796398.html)\
[consistent\_hashing](https://github.com/apache/incubator-brpc/blob/master/docs/cn/consistent\_hashing.md)

* 把所有server的处理区间视为一个环。比如\[0, 1000)这样一个range。1000和0是相连的
* 每个server会扩充m倍形成虚拟节点，比如一个server A，形成虚拟的(A1, A2, … Am)这些虚拟节点，这些节点无论访问哪一个，实际上都是A。所以叫虚拟节点。如果有n个server，那就是nm个虚拟节点。比如三个server(A, B, C), 扩充虚拟节点就是(A1, … Am, B1 … Bm, C1 … Cm)这些节点。
* 虚拟节点 **随机** 插入到环中。这样每一个虚拟节点都会负责一段小range。举个例子：比如3个server(A, B, C)扩充2倍(A1, A2, B1, B2, C1, C2)，随机插入到range，比如 (A1->101, B2->203, C2->321, B1->432, A2->567, C1->800)。这样每个节点就会负责一小段range。假设是按照upper bound寻找节点，那么B2负责的range就是(101, 203], B1负责的range就是(321, 432]，A1负责的range就是(800, 101]。(因为range是个环，1000和0是首尾相连的)。
* 基于上面，很容易理解，真实server会负责随机的区间，比如B负责的区间是(101, 203]和(321, 432]区间。同样可以得知，**由于consistent hash是随机插入虚拟节点的，并不保证所有server最后会均分区间。**但是扩充的虚拟节点越多，区间就越可能趋近于均匀。比如\[0, 1000)，如果你扩充出来1000个虚拟节点，那就一定均匀了。所以虚拟节点的存在才是一致性hash能解决热点问题的核心点。但是扩充太多同样影响查找的性能。所以扩充多少也要结合实际。在brpc里这个数字是100，也就是每个server扩充100个虚拟节点。
* 查找比较简单，就是计算数据的hash值，这个hash值映射到range，然后寻找这个hash值对应range的虚拟节点。比如一个数据计算hash值是140，就应该由B2这个虚拟节点处理。也就是由真实server B处理。
* 宕机掉节点问题：如果一个机器宕机，当hash到与这个机器对应的range的时候，由于这个虚拟节点（对应的真实server）已经宕机，所以他会继续寻找下一个虚拟节点。也就是说，一个server宕机了，他的所有虚拟节点负责的range，被merge到与它相邻的虚拟节点。举例子，我们假设B这个server宕机了，一条数据计算出hash值是140。根据原来的分配，应该由虚拟节点B2处理。但B2已经失效，就会继续向下找节点，找到了C2。也就是说，C2负责的range从原来的(203, 321]变成了(101, 321]。而由于虚拟节点是随机插入的，所以也能保证server宕机的时候，他负责的range会被随机的其他若干server承担。

总结：consistent hash利用虚拟节点解决热点和宕机的问题。 如果server同构，无状态，可以考虑用consistent hashing来实现比较理想的负载均衡。**但如果server异构（比如sharding特征），将不适用一致性hash来解决负载均衡**。 [issue](https://github.com/apache/incubator-brpc/issues/649)

#### Rendezvous hashing

**工作原理**

相关资料奇缺，代码实现貌似也只见过go有个相关package。基于维基百科的[Rendezvous hashing](https://en.wikipedia.org/wiki/Rendezvous\_hashing)尝试自己翻译。去解释rendezous hash的工作原理。从The HRW algorithm for rendezvous hashing这段开始，翻译了三段内容，扩号内容是我自己加的注解，更容易理解：

**The HRW algorithm for rendezvous hashing**

给定一个object O，如何让所有的client去达成一致，选出一个server集合去放置O？每个client都可能自己独立的选一个server，但是最后必须要让所有client选择相同的server。有一个非常重要的点是我们要增加一个最小中断约束，即只有mapping到了被移出server的object才可以重新分配server。

基础的想法是，基于每个object O，为每个server j去计算一个得分，然后将object o分配给最高得分的server。首先，所有的client要有一致的hash算法h()。对于每一个object O，server j会计算出一个权重 `w(i, j) = h(Oi, Sj)`。HRW算法会将object O分配给权重最高的server。因为hash算法是一致的，所有的client都可以各自计算权重并挑选最高权重的server。如果是想做k-agreement，那就每个client都挑选权重最大的k个value就好了。如果server S被添加或删除，只有被mapping到S的object会被remapping，这样就满足了我们上面提到的最小中断约束。任意client都可以基于HRW计算object最终的分配（server），因为影响分配的只有server和object自身。

HRW算法可以轻易适应不同server之间的不同capacity。假如server k拥有别的server的两倍容量，那我们只要将server k push到server list两次(比如k1，k2这样)，很显然这样server k就会被分配到两倍的object。

**Properties**

(最开始出现的基于直接hash的distribute hash table)每一个server都视为一个bucket，形成一个hash table，然后将object O hash到这个table中。然而如果有server出现故障，会引起整个hash table的size变化，所有的object都要remapping，(会带来很多的问题，比如cache颠簸等)这样巨大的破坏，使得直接hash不可行。而rendezvous hash下，client在遇到server失败的时候，会选择下一个weight最大的server。remapping只会在server故障的情况出现，满足最小中断请求。rendezvous hashing有以下几个特点：

1. low overhead：hash算法很高效，所以给client带来的额外压力很低。
2. load balancing：hash算法是随机的，所以每个server在object面前都是等价的。从而负载也是均衡的。在server的capacity不同的时候，可以根据capacity容量比例来进行多重复制。比如一个两倍capacity的server可以在server list出现两次。
3. high hit rate：所有的client都会将object O分配给相同的server O，所以fetch或者place O会达到最大的hit rate。除非server O内部有驱逐算法淘汰，否则你总会在server O中找到object O。
4. minimal disruption：当一个server故障时候，只有被mapping到这个server的objects会被remap。中断在最低可能。
5. distributed k-agreement: 所有的client都可以找到相同的k个server，只要按weight排序选最大的k个server就可以了。

**comparison with consistent hashing**

consistent hashing将server随机分配很多token到环上。object也会map到环上，然后他会归属于顺着环往下寻找的第一个出现的server的token。如果server被移出了，相应的object会转移到第二个出现的server。每个server都有很大的次数(比如100\~200个token)map到环上，(这样server被移出掉的时候，本来map给这个server的)object就会相对均匀的分配到其他server上。

如果(在consistent hash的时候)我们给每个server hash了200个值到环上，那任何object的分配，都需要server存储或者重新计算200个hash值。然而server在token上的(200个)hash值可以提前计算，存储到一个list中，这样我们只需要为object计算hash，然后(在list)进行一次二分查找，找到对应的server就可以了。然而即使每个server都有很多的token值，基础的consistent hash也可能没法做到object在server之间的均衡。consistent hashing的变种(比如amazon的dynamo)会使用更复杂的逻辑去分布token，达到一个更好的复杂均衡效果，减少新增server带来的开销，减少metadata带来的开销，还有一些其他好处。然而这些方法比consistent hashing更复杂。

相比下，rendezvous hashing在概念上和实践上都更简单。给定一个统一的hash函数，它也做到了在所有server间均匀分布object。不像consistent hashing，HRW不需要token的提前计算和存储。一个object会在与所有server计算hash值后，选取hash值最高的server并处理。如果新server加入，新object的placement或者request会计算n+1个hash值，选取最大的那个。如果一个object已经被map到server k上了，然后它需要被更新到server n+1上，那它会在server n+1上更新fetch和cache。然后所有的client都会在这个server n+1上获取(object)，server k上的old cache会被server k的local cache管理算法自动淘汰掉。如果server k下线，那么它的object会被均匀remap到剩下的n-1个server上。

HRW算法的变种，比如skeleton，可以把每个object的(分配server的)定位时间从O(n)减少到O(logn)，以较小的全局均匀placement为代价。然而如果n不是特别大的话，HRW的O(n)计算代价可能并不会是问题。HRW完全避免了正确处理每个server tokens带来的开销和复杂性。

rendezvous hashing也在其他重要问题处理上提供了很简单的解决方案，比如分布到k个server上。

**总结**

结合这个[stackoverflow](https://stackoverflow.com/questions/20790898/consistent-hashing-vs-rendezvous-hrw-hashing-what-are-the-tradeoffs)：rendezvous hash是在server中pick出来k个server，作为object应该对应的对象。当k为1的时候，其实这个算法等价于consistent hash，没区别。

他们的主要区别在于

* server故障时候的的异常处理：consistent hash由于虚拟节点是随机插入的，故障server对应的range被哪些server分摊了，你无法得知；用rendezvous hashing的话你可以用hash算法计算出来这个server在哪。
* 多server pick的支持：计算hash值后，HRW算法可以选多个server。

没有实际应用过这个算法，如何O(logn)的计算没有细看，感觉每一个object都要算多次hash，这个算法计算量太大；并没有感觉这个算法与consistent hash相比有多么大的优势。

### 负载均衡

[locality-aware load balancing](https://github.com/apache/incubator-brpc/blob/master/docs/cn/lalb.md)\
和前面consistent hashing一样。这种负载均衡策略也是适合于server无状态。
