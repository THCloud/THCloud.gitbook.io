# apache arrow

## 简介 <a href="#e7-ae-80-e4-bb-8b" id="e7-ae-80-e4-bb-8b"></a>

Apache Arrow([https://arrow.apache.org/](https://arrow.apache.org/))是Apache基金会近几年最活跃的项目之一，它基于内存列式格式衍生出了完善的内存计算生态，是当前内存列式数据格式事实上的标准。Arrow的内存模型可以帮助编译器自动地实现向量化，且在传输时没有序列化/反序列化成本，实现了CPU和IO的效率提升。众多我们熟悉的数据处理系统如Spark，Impala，Kudu，Clickhouse，Pandas等项目中都有Arrow的身影。Arrow本身不像是“轮子”，更像是用来造轮子的“轮毂”，在底层支撑各个数据处理项目。本文旨在介绍Arrow的基础知识和它的优势。

## 什么是Apache Arrow

引用Apache Arrow官网的第一句话：

> Apache Arrow定义了一个各语言通用（language-agnostic）的列式内存格式（columnar memory format），支持扁平（flat）和多层级的（hierarchical）数据。\
>

简而言之，Arrow定义了各种数据类型（如简单的int、float和复合的struct, union，list, map等）的数组在内存中如何用一段连续内存表示，且这种表示是各语言通用的。

### 各语言通用的协议

与其他大多数的Apache项目不同，Arrow项目本身不是一个组件而是一个标准化协议，类似于Protobuf和Thrift，Arrow本身定义了一个跨语言的表示数据的方法，同时为各种主流语言提供了原生的处理Arrow数据的实现，当前支持的语言包括：C++、Python、Rust、Java、Go、R、Julia、Matlab等等。除了官方实现外，开发者还可以根据协议编写非官方的实现，比如Rust就有非官方的arrow2，比官方的arrow-rs库更安全、性能更好。

### 列式内存格式

列存（columnar，column based）是与行存（row based）相对应的一种存储方式，两者是数据库领域的术语。对于一张二维数据，列存就是将同一列的数据存储在一起，列存与行存对比有哪些优势和劣势呢？

优势：

1. 更少的IO：可以只读取需要的列。
2. 更好的压缩效果：同类型的数据存储连续存储对某些压缩机制更友好，比如Run Length Encoding，Delta Encoding，Dictionary Encoding等。
3. 更高的计算效率：列式存储对cpu缓存和向量化更友好，后面会单独讲。

劣势：

单行读取，插入，更新和删除操作都更慢。因为这些操作都涉及到整行的操作，对于每行数据都要访问m次（m为列的个数）。

由此可见，列存格式的优势在于行数多，列数少的只读（分析）操作，数据库领域称这种场景为OLAP，A代表Analysis，与之相对的，行存的优势场景在OLTP，T代表Transaction。

#### 小插曲

列存的本质就是“相同类型的数据存储在连续内存中”，实际上就是vector数据结构。其实Apache Arrow项目本来是要叫做Apache Vector的，但是创始成员们考虑到vector可能会有潜在的商标问题，以及arrow在字母序上排位更靠前，最终Arrow以58：56两票之差战胜了Vector（[当时的投票文档](https://docs.google.com/spreadsheets/d/1q6UqluW6SLuMKRwW2TBGBzHfYLlXYm37eKJlIxWQGQM/edit#gid=0)）。在我看来Arrow是比Vector要好得多的名字，因为arrow箭头符号本身在数学中就代表向量（比如就代表x向量），而arrow也有箭矢的含义，给人一种很快速的感觉，和Arrow项目的初衷不谋而合。

### 协议内容

我们简单介绍一下Arrow的协议内容。

#### 基础类型

固定宽度的基础类型与vector并无差异，只是多了validity bitmap。以一个Int32 Array为例：

```cpp
[1, null, 2, 4, 8]
* Length: 5, Null count: 1
* Validity bitmap buffer:

  |Byte 0 (validity bitmap) | Bytes 1-63            |
  |-------------------------|-----------------------|
  | 00011101                | 0 (padding)           |

* Value Buffer:

  |Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-19 | Bytes 20-63 |
  |------------|-------------|-------------|-------------|-------------|-------------|
  | 1          | unspecified | 2           | 4           | 8           | unspecified |
```

首先注意到两个buffer都占了64个字节，Arrow并没有规定buffer的对齐规则，但是推荐实现方给予64个字节对齐，这是因为最大的SIMD寄存器（AVX512）长度为64个字节，与寄存器长度对齐可以最大程度地利用SIMD指令。其次注意到Array中是可以有null值的，因此需要一个单独的validity bitmap buffer来记录null value。而对于没有空值的Array，则可以省略掉这个bitmap，比如

```cpp
[1, 2, 3, 4, 8]
* Length 5, Null count: 0
* Validity bitmap buffer: Not required
* Value Buffer:

  |Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | bytes 12-15 | bytes 16-19 | Bytes 20-63 |
  |------------|-------------|-------------|-------------|-------------|-------------|
  | 1          | 2           | 3           | 4           | 8           | unspecified |
```

#### String类型

StringArray通过增加一个offsets buffer来保证连续内存：

```cpp
["hello", null, "world", "apache", "arrow"]
* Length 5, Null count: 1
* Validity bitmap buffer:

  |Byte 0 (validity bitmap) | Bytes 1-63            |
  |-------------------------|-----------------------|
  | 00011101                | 0 (padding)           |

* Offset Buffer:

  |Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | bytes 12-15 | bytes 16-19 | Bytes 20-23 | Bytes 24-63 |
  |------------|-------------|-------------|-------------|-------------|-------------|-------------|
  | 0          | 5           | 5           | 10          | 16          | 21          | unspecified |

* Value Buffer:

  |Bytes 0-4   | Bytes 5-9   | Bytes 10-15 | bytes 16-20 | Bytes 21-63 |
  |------------|-------------|-------------|-------------|-------------|
  | "hello"    | "world"     | "apache"    | "arrow"     | unspecified |
```

Offset Buffer包括n+1个数字，对于第i个string，起始点为offset\[i]，长度为offset\[i+1]-offset\[i]。

#### 复合类型

复合类型是包括一个或多个子数组的类型，以Struct\<name: VarBinary, age: Int32>为例：

```cpp
[{'joe', 1}, {null, 2}, null, {'mark', 4}]
* Length: 4, Null count: 1
* Validity bitmap buffer:

  |Byte 0 (validity bitmap) | Bytes 1-63            |
  |-------------------------|-----------------------|
  | 00001011                | 0 (padding)           |

* Children arrays:
  * field-0 array (`String`):
    * Length: 4, Null count: 2
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00001001                 | 0 (padding)           |

    * Offsets buffer:

      | Bytes 0-19     |
      |----------------|
      | 0, 3, 3, 3, 7  |

     * Values array:
        * Length: 7, Null count: 0
        * Validity bitmap buffer: Not required

        * Value buffer:

          | Bytes 0-6      |
          |----------------|
          | joemark        |

  * field-1 array (int32 array):
    * Length: 4, Null count: 1
    * Validity bitmap buffer:

      | Byte 0 (validity bitmap) | Bytes 1-63            |
      |--------------------------|-----------------------|
      | 00001011                 | 0 (padding)           |

    * Value Buffer:

      |Bytes 0-3   | Bytes 4-7   | Bytes 8-11  | Bytes 12-15 | Bytes 16-63 |
      |------------|-------------|-------------|-------------|-------------|
      | 1          | 2           | unspecified | 4           | unspecified |
```

更多关于Arrow协议的信息可以参考官方文档：[https://arrow.apache.org/docs/format/Columnar.html](https://arrow.apache.org/docs/format/Columnar.html)

## 为什么使用Apache Arrow

### 高效计算

在分析场景中，我们往往要对同一列的每一行做相同的计算，这时使用Arrow的列存模型会带来三点好处：

1. CPU缓存友好：CPU在加载数据时会一次加载一整个Cache Line的数据，在读Arrow Array的一个Element时会把后面的几个Element一起加载进来，避免了Cache Miss带来的额外读取时间。
2. SIMD向量化计算友好：在循环处理连续内存，且数据间没有依赖时，我们可以利用如AVX、SSE等SIMD指令集加速数据的计算。

Arrow可以给计算效率带来巨大的提升，比如Spark在引入Arrow后处理Dataframe的效率提升了53倍（[https://github.com/apache/spark/pull/15821#issuecomment-282175163](https://github.com/apache/spark/pull/15821#issuecomment-282175163)）。

### 零序列化/反序列化

由于Arrow的任何数据结构都是一段连续的内存，在跨进程/跨机器传输Arrow数据时无需序列化和反序列化，直接发送/接收整段内存即可。

### 跨语言支持

由于Apache Arrow是语言无关的，所以同一份数据在比如C++服务和Java服务当中的内存表示是完全一致的，跨语言传输数据也完全无需额外操作。同时对数据的处理逻辑也可以跨语言迁移，比如在Python中对Arrow数据的操作可以直接翻译成Go语言的操作，Arrow保证数据操作的一致性。

### 完善的数据类型和生态

Arrow能成为内存计算的事实标准并不只是因为“连续内存”这么一个简单的特性，毕竟这个特性太过于容易实现，而设计一个标准也过于容易，很容易出现xkcd927（[https://xkcd.com/927/](https://xkcd.com/927/)）所描述的现象：

* 当前有14个互相竞争的标准。
* 甲：“14个？太离谱了！我们需要设计一个包含了所有人需求的黄金标准！” 乙：“好的！”
* 当前有15个互相竞争的标准。

比如谷歌的Flatbuffer就也是以零拷贝、连续内存著称，然而Flatbuffer实在是太难用了，而且是以社区不太喜欢的“谷歌的方式”开源，即一切以谷歌内部需求优先，变更定期批量同步到社区，社区的需求基本不管（偷偷黑一下老东家），所以也一直不温不火。

而Arrow的易用性和完整性是其他内存协议所不具备的，小小一段连续内存被Arrow团队玩出了花来。从数据类型来看，Arrow不止支持SQL常见的INT，FLOAT，VARCHAR，TIMESTAMP等，还有复合数据结构List、Struct、Map、Union、Dictionary，还有机器学习用的Tensor，大数据分析常用的Dataset等格式，Arrow不仅能用连续内存表示这些数据，还为这些数据结构都提供了完善的处理接口。

除了Arrow本身的数据结构外，Arrow项目还围绕这种数据协议搭建了一整套完善的生态，包括：

1. 各大主流语言的原生支持
2. Compute：++原生的向量化计算引擎，支持数值计算、数组计算、聚合计算等各种常见的计算函数
3. Gandiva：基于LLVM JIT的表达式计算库，在数值计算上可以得到明显加速效果
4. Datafusion：Rust原生的SQL查询引擎
5. Ballista：分布式SQL查询组件
6. Flight：用于传输Arrow数据的RPC框架
7. ADBC：Arrow的标准数据库访问协议
8. 第三方文件格式支持：如JSON、CSV、Apache Parquet、Apache ORC等等协议与Arrow之间的转换
9. 大数据表支持：除了Array外，Arrow还提供了RecordBatch二维数据表、Dataset数据集的类型
10. 完整的平台支持：包括Linux（Ubuntu、CentOS、Arch等），MacOS，Windows等等操作系统和amd64、arm64等架构
11. 完整的语言支持：包括C++、Python、Java、Go等主流语言的支持

这些生态才是Apache Arrow能成为标准的主要原因，一旦项目引入了Arrow数据格式，就可以零成本地引入这些IO、计算和网络库。也正是这些组件，让开发者们能更好的“造轮子”。有一些组件我会在之后的其他文章中详细介绍。

### 业界标准的格式

除了Apache基金会自己的大数据组件如Spark对Arrow有完善的支持外，业界的其他主流大数据/AI系统均对Arrow有支持

1. Google的Tensorflow支持Arrow的Dataset直接导入为Tenforflow Data（[https://blog.tensorflow.org/2019/08/tensorflow-with-apache-arrow-datasets.html](https://blog.tensorflow.org/2019/08/tensorflow-with-apache-arrow-datasets.html)）
2. Google的BigQuery支持原生处理Arrow数据（[https://cloud.google.com/bigquery/docs/samples/bigquerystorage-arrow-quickstart](https://cloud.google.com/bigquery/docs/samples/bigquerystorage-arrow-quickstart)）
3. Facebook开源的Velox向量化计算引擎是基于Apache Arrow数据格式实现的，提供的功能也类似于Arrow官方的Compute（[https://vldb.org/pvldb/vol15/p3372-pedreira.pdf](https://vldb.org/pvldb/vol15/p3372-pedreira.pdf)）
4. Clickhouse、DuckDB、InfluxDB、DataBend等OLAP和时序数据库要么基于Arrow构造，要么对Arrow提供原生支持

## 如何使用Apache Arrow

### 编译依赖

C++由于没有统一的包管理因此需要手动编译，可参考[https://arrow.apache.org/docs/developers/cpp/building.html](https://arrow.apache.org/docs/developers/cpp/building.html)，Go、Python、Java、Rust均有现成的包可使用。



### 代码示例

可参考官方的文档([https://arrow.apache.org/docs/index.html](https://arrow.apache.org/docs/index.html))以及提供的示例Cookbook：C++([https://arrow.apache.org/cookbook/cpp/](https://arrow.apache.org/cookbook/cpp/)), Java([https://arrow.apache.org/cookbook/java/](https://arrow.apache.org/cookbook/java/)), Python([https://arrow.apache.org/cookbook/py/](https://arrow.apache.org/cookbook/py/))。
