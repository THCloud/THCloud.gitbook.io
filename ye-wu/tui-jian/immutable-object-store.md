# IMmutable Object Store

### 背景与定位 <a href="#e8-83-8c-e6-99-af-e4-b8-8e-e5-ae-9a-e4-bd-8d" id="e8-83-8c-e6-99-af-e4-b8-8e-e5-ae-9a-e4-bd-8d"></a>

在推荐中应用的一套针对异构数据集合服务化的通用解决方案，定位于解决以下数据应用场景问题：

* 离线数据应用于在线服务
* 数据规模单服务实例无法承载，一般10GB+乃至更高的规模
* 较高的读性能需求，存在100w+ qps 访问压力情况
* 数据格式异构非标准，类型存在持续增长的态势
* 数据份数随abtest实验增加，存在持续增长/调整的态势
* 复杂的耦合于特定数据的计算逻辑与业务代码强耦合

与常规的kv存储(Redis/Memcache)等服务相比，imos具备以下特点： Pros：

* 服务容量扩展性粒度更细，可根据不同数据集合设置不同的副本数目
* Build与Serving分离
  * Serving无锁读，提供更高的读性能
  * 批量更新提供更高的写入吞吐
* 数据格式支持多种异构形式
  * 支持比kv更复杂的数据格式，如faiss/fasttext等
  * 比较方便扩展支持新的数据格式
* 数据计算逻辑扩展性支持自定义逻辑，内置基于数据的Native Query Script机制
  * 支持Cpp/Lua 开发script
* 部署上更为灵活，支持分set隔离不同数据，不同script逻辑

Cons：

* 不支持实时写入数据， 数据生效最快只能到分钟级别
* 数据一致性不严格，数据更新期间存在不同版本数据
* Native Query Script的引入使得服务稳定性受限于Script实现；不适合单一服务应用大量数据与script场景，需要在部署上隔离
* 接入较复杂，需要开发script以及较复杂的NativeQuery接口编写

具体到推荐、搜索业务场景中，适用于字典、倒排索引、正排索引、模型等离线挖掘数据到线上应用场景。

### 依赖与限制

开发依赖：

* C++11
* Go 1.11+
* Boost 1.60+

部署运行依赖：

* CL5, 一般用于对外开放的name serving
* HDFS/COS/S3/Ceph， 用于备份数据
  * 选择其一即可
* &#x20;配置中心
* 用于统计与监控

以上运行依赖均非强绑定耦合，可以比较方便的适配接入其它服务替换。

### 基础功能

imos 具备以下核心基础能力：

* 多种数据格式支持以及更多数据格式支持的扩展能力
  * CMOD, 同时集成FlatBuffers
  * RocksDB
  * ANN索引，Faiss/hnswlib/Annoy
  * FastText模型
* Native Query
  * 用户自定义的script实现，可用C++/Lua实现；支持热更新
  * 基于多级script function组成计算DAG
* 统一的与数据类型无关的分布式服务实现
  * 数据规模， 通过通用的hash sharding机制解决
  * 服务容量， 通过可变副本数机制解决
  * 统一的Build-Ship-Serving pipeline
* 统一的基于文件复制/备份机制，与数据类型无关
  * 基于HTTP的简化P2P file replication
  * 与可靠存储（HDFS/COS/S3/Ceph）无强绑定的备份机制

### imos基本设计

imos中按角色分为以下5种：

* Master
* Data
* Builder
* Access(Proxy)
* CLI

基本架构图如下： ![](https://git.woa.com/rimos/docs/raw/master/img/rimos_full_arch.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

目前实现中，`Data`与`Access`部署于一起。

以下按具体角色介绍各自承担的功能：

#### Master

* 数据元信息管理
* 强一致性状态，Raft集群
* 主要负责以下元信息管理
  * 节点状态
  * 数据元信息，基本配置/构建文件md5，size等
  * CMOD数据Schema
  * Protobuf
  * Flatbuffers
  * 数据分配状态，数据分配的节点信息
  * Native Query Script
  * Script Source
  * 预编译的so/lua pack文件md5，size等

#### Data

* 承载具体数据存储
* 相对弱状态，数据分配状态由master决定
* 与Master心跳交互，根据同步信息执行对应操作
  * 数据的CRUD
  * Native Query Script的CRUD
* 执行相应的数据计算Native Query RPC请求
  * 根据Native Query请求的参数选择合适的so中的对应函数执行请求

#### Builder

* 执行相应数据构建任务
* 服务无状态
* 一次构建过程时间较长，存在短期构建状态；若期间宕机，则涉及的构建任务快速失败
* 一般按需扩容即可；每次构建任务会自动随机选择实例执行构建任务
* 数据构建过程
  * 随机选择N个节点执行对应Partition的构建任务， N=数据的partition设置
  * 创建索引文件
  * 数据写入索引文件
  * 索引文件写入可靠存储HDFS(或者COS/Ceph/S3)
  * 索引文件预发布Push到Data节点，基于内置的HTTP P2P文件复制机制
  * 向Master写入构建完成的元信息(版本，MD5，Size等)
  * 若构建过程失败，需要cli重试

#### Access

* 对外RPC API入口
* 无状态，按需扩容即可
* 与Master心跳交互，根据同步信息执行对应操作
  * 数据路由信息
  * Native Query Script的CRUD
* Native Query Engine， 参考后文`Native Query`一节

#### CLI

* golang实现, cli工具
* 基于开放的rpc api接口
* 通常用于驱动数据创建/构建/测试验证等管理类场景

### 存储结构

存储结构中设计了一个keyspace概念，用于聚合相似数据到一个名字空间下; 当数据计算逻辑涉及多份相关数据时，由于存储设置相同，可以就地获取相关数据无需增加RPC路由解决。 以下为存储结构基本概念设计：

* (KeySpace，Key) 唯一表示一份数据
* KeySpace 可以设置Partition数，用于存储容量纵向扩展
* KeySpace 可以设置replica副本数，用于服务容量水平扩展
* (KeySpace, Partition) 代表一个数据单元Region， 可以存在多个key
* (KeySpace, Partition, Key) 代表一个最小数据对象

下图示例显示如下keyspace设置的存储示意：

* KeySpace A 设置partition 2， replicas 为2
* KeySpace B 设置partition 2， replicas 为1
* KeySpace C 设置partition 1， replicas 为3
* KeySpace D 设置partition 2， replicas 为1

![](https://git.woa.com/rimos/docs/raw/master/img/rimos_storage.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

### Native Query

rimos中的Native Query被设计为类似spark的DAG执行模式，遵循以下设计原则：

* 计算向数据存储移动
* 用户/service 驱动计算逻辑

rimos中的Native Query具备以下能力：

* 支持C++/Lua开发
* 三种执行模式：
  * 定点Native Query： 选择数据Id所在的节点执行，与Query参数中数据Id相关
  * 广播Native Query： 选择数据所有的partition执行
  * Local Native Query： 与数据无关，一般在当前Access节点执行
* 多stage Native Query任意组合一个计算DAG

大致架构实现如下图所示： ![](https://git.woa.com/rimos/docs/raw/master/img/native_qeury.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

下图则显示一个向量搜索过程的Native Query执行过程， 其执行可通过cli驱动：

```bash
./imos_cli query  
--step "hnsw_test|hnsw_test.cpp|get_vector|6YilqYyFx1IKq9gTm|" 
--step "hnsw_test|hnsw_test.cpp|search_by_vector||" 
--step "|hnsw_test.cpp|sort_by_score_asc||"
```

![](https://git.woa.com/rimos/docs/raw/master/img/rimos_search.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

### 数据构建发布Pipeline

rimos 数据构建发布流程是一个通用的pipeline流程，类似于Docker思路，只不过这里只是针对数据文件。 下图是一个简单的3个partition， 1个副本设置的数据全流程pipline。 ![](https://git.woa.com/rimos/docs/raw/master/img/rimos_pipeline.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

### 系统扩展性

rimos的系统扩展性表现在以下四方面：

* 数据规模
* 服务容量
* 数据查询计算逻辑
* 异构数据支持扩展

其中，关于`数据查询计算逻辑`扩展性已在`Native Query`中讨论。以下主要从数据规模/服务容量/异构数据支持扩展展开讨论：

#### 数据规模扩展

imos支持数据设置自己的分片数目，这里与一致性hash类的设计不同在于，这里的分片数目在整个数据的生命周期内是不可变的； 当单个分片数据增长到单机无法承载的情况下，可以通过如下操作执行：

* 新建一个keyspace，设置partition分片数目到足够大
* 重新执行一次数据全量build
* 数据访问路由指向新的keyspace/key

实际上上述步骤是一个全新的重新build+serving过程，时间较长，类似于elasticsearch重新rebuild过程。

imos也支持实现设置足够大的分片数目如1024、4096实现类似一致性hash实现。 相比前面实现，当数据规模增加后，只需要迁移部分partition到新的节点上即可，扩容过程只相当于新增服务实例+ 迁移复制partition数据到新的服务实例上，过程会比较快，也无需修改数据访问路由。 但这种模式存在几个问题：

* build过程产生的文件过多，管理与复制分发效率都有下降；
* 由于build过程是一个全partition必须全部成功的原子操作，更多的partition可能造成更高的失败率
* 广播模式下会造成更多的rpc放大问题(可以通过某种技术手段合并一些rpc)

考虑到实际数据应用场景中，定期全量build是一个无法避免的操作，实际应用场景以前者为主

#### 服务容量扩展

imos支持动态调整数据的副本数设置，服务容量扩容只需要修改副本数设置到一个期望值即可。新增的副本数据会分配到新的服务实例上，执行相应的复制操作并提供读服务。

#### 异构数据支持扩展

imos中设计了一个与具体数据实现无关的中间层，用于统一的数据同步/CRUD等操作。 若需要增加一种新的数据类型，可以比较方便的通过标准数据接口实现扩展。 大致实现架构如下：

![](https://git.woa.com/rimos/docs/raw/master/img/data_extend.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

### 容灾与故障恢复

分角色讨论：

* Master
  * Raft集群，依赖raft机制自行恢复
* Access
  * 无状态，无需任何处理
* Builder
  * 无状态，无需任何处理
  * 宕机只影响相关的正在执行的build任务，快速失败返回错误；由ETL后续重试
* Data
  * 弱状态，数据状态由master决定
  * Data宕机，Master控制宕机数据副本迁移至资源占用最小节点
  * 由于数据理论上存在多副本replica，宕机期间只影响宕机所在实例当时处理的部分请求
  * 数据都会存放到可靠存储上，当前选择HDFS；在所有节点挂的情况下也不会丢失数据

![](https://git.woa.com/rimos/docs/raw/master/img/rimos_crash.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)
