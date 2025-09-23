# 基于apache arrow的高性能向量化在线特征计算引擎

### 简介 <a href="#e7-ae-80-e4-bb-8b" id="e7-ae-80-e4-bb-8b"></a>

基于“向量化”和“计算图”技术，相较于传统的在线特征计算框架可实现数倍到数十倍的性能提升。

### 背景

在使用深度学习模型对用户进行个性化推荐时，我们经常会需要在用户线上请求时进行一部分特征计算，如过滤、交叉、聚合、分桶、哈希等等操作。由于线上请求对于时延的敏感性，传统大数据框架如Spark、Flink往往无法胜任（这两个框架经常被应用离线和实时特征**生成**，而非在线**计算**），业界各大公司往往会用C++实现一套自己的特征在线计算框架（如[Facebook的f3](https://www.facebook.com/atscaleevents/videos/ai-scale-2020-f3-next-generation-feature-framework-at-facebook/1073947483052122/)）。

在过去的一年多时间里，团队一直在探索如何更高效地计算在线特征。在个性化推荐场景里，特征往往分为user和item两类，一次在线请求包含一个user和成百上千个item。在过去的实践中，我们会对user和每个item单独存储和计算他们的特征，类似传统的OLTP“行存”。然而我们注意到，由于每个模型的特征计算逻辑都是固定的，我们在一次请求中会对大量的item特征数据执行同样的计算逻辑，我们是不是可以借鉴OLAP数据库的“列存+向量化“技术加速线上的特征计算呢？

我们基于这个理念开发了在线特征计算引擎。使用Apache Arrow作为底层数据结构，Arrow是一种列存内存格式标准，充分地利用了向量化SIMD指令集和CPU缓存（关于Apache Arrow的更多介绍可以参考我的上一篇文章）。同时在特征计算过程中还应用了算子融合、子图共享等技术。我们在业务上的测试数据显示，相较于原有的特征计算框架实现了数倍的性能提升。

### 示例

在引擎中，特征通过将某个算子(Op)应用到一个或多个原始值上来得到特征结果，用户只需要通过JSON配置定义好每个特征的计算逻辑，即可实现零代码上线新特征。

例如特征Final通过对原始值Raw进行哈希后得到，则配置为

```json
{
  "Final": {
    "Op": "CityHash64Op", // Compute内置的哈希算子为Google的CityHash
    "Dependency": ["Raw"]
  }
}
```

每个算子可以类比做一个数学函数，输入一个或多个值输出一个值，如上面的配置的含义为`Final = CityHash64Op(Raw)`。

用户也可以通过定义中间层变量，组合多个算子，表达出复杂的特征计算逻辑。如特征Final通过对原始值Raw进行过滤负值、分桶离散化后哈希得到，则配置为

```json
{
  "Mid_Filtered": {
    "Op": "FilterOp", // 过滤算子
    "Filter": "v1 > 0", // FilterOp需要给出过滤条件，v1即代表第一个依赖：Raw 
    "Dependency": ["Raw"]
  },
  "Mid_Discrete": {
    "Op": "BucketOp",  // 离散化分桶算子
    "Args": {
      "boundaries": [1, 10, 100, 1000] // BucketOp需要给出每个分桶的边界
    },
    "Dependency": ["Mid_Filtered"]
  },
  "Final": {
    "Op": "CityHash64Op",
    "Dependency": ["Mid_Discrete"]
  }
}
```

通过定义中间层特征，用户可以表达出复杂的计算逻辑。如这个配置对应的数学表达式是`Final = CityHash64Op(BucketOp[1, 10, 100, 1000](FilterOp[v1 > 0](Raw)))`。

注：只是计算框架，并没有从存储读取数据的功能，因此用户先从存储读取好数据，将map\<name, raw\_value>作为输入传入。另一个工具DAO可以实现零代码读取KV原始值，只需要提供每张表的key、proto协议和所需要的字段信息，就可以将KV表中的任意字段转换为Compute需要的原始值。

在了解了 Compute的大体概念后，我们来介绍一下实现的技术细节。

### Apache Arrow列式内存表示

Compute使用Apache Arrow格式表达特征，每一个特征用一个Arrow Array表示，包含这个特征的所有item的特征值。我们主要使用Arrow的三种Array：

1. PrimitiveArray：用于表达单值特征，每一个slot都是一个数值或者字符串，长度和item size相同
2. ListArray：用于表达列表特征，将所有item的特征列表值拼接在一起，长度为所有item值长度的和，通过offsets来区分每个item的列表区间。
3. MapArray：用于表达Map类特征，MapArray包含两个子ListArray，分别存储key和value两个array。如“用户对每个类目的喜爱度”，Key Array存储类目id列表，Value Array存储分数列表。

在处理user特征时，为了通用性我们同样用一个长度为1的Array来表示，虽然在处理单值特征时有额外的性能损耗，但user存在大量的列表特征和Map特征，这部分特征的处理仍然能够受益于Arrow的连续内存和向量化。

### 向量化算子

算子是Compute最小的计算单位，所有特征都通过算子计算得出，因此算子的性能对特征计算框架来说至关重要。每个Compute算子输入一个或多个Arrow Array，输出一个结果Array，对于每个特征，所有的item的该特征在一起计算。算子的高性能主要来自于SIMD向量化技术。我们的向量化通过四种途径实现：

#### Gandiva向量化表达式引擎

[Gandiva](https://arrow.apache.org/docs/dev/cpp/gandiva.html)是Apache Arrow的一个子项目，基于LLVM JIT实现表达式向量化计算。Gandiva提供数值变换（Projector）和过滤（Filter）两种运算。Gandiva通过将基础计算函数预编译成LLVM IR代码，在编译表达式时内联基础函数，构成完整的IR代码，然后在优化阶段通过LLVM的各个优化Pass来实现性能提升。其中涉及到向量化的主要是[LoopVectorizePass](https://llvm.org/docs/Vectorizers.html#the-loop-vectorizer)和[SLPVectorizerPass](https://llvm.org/docs/Vectorizers.html#slp-vectorizer)，前者可以将循环自动展开后替换为SIMD指令，后者可以将独立的相似计算逻辑合并为SIMD指令。

研究表明，在数值计算场景中预编译模式是性能最优的策略（而交叉、聚合等操作则是预编译的弱点）。Riemann Compute中绝大部分的数值计算都是通过Gandiva实现。

#### Arrow Compute计算函数

Apache Arrow包括了一个[Compute](https://arrow.apache.org/docs/cpp/compute.html)框架，Arrow Compute与Gandiva类似，提供了诸多的函数供用户调用。不同的是，Gandiva是通过预编译技术实现优化，而Arrow Compute则是通过解释型向量化（Interpreted Vectorization），即在线上直接调用对应函数的指针。这种模式导致函数无法被内联，失去了一些全局优化的机会，但省去了LLVM IR转换为X86机器码的开销。（关于编译型向量化和解释型向量化的探讨，可以参考这两篇paper：[《Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask》](https://db.in.tum.de/~kersten/vectorization_vs_compilation.pdf?lang=de)和《[Vectorization vs. Compilation in Query Execution](https://15721.courses.cs.cmu.edu/spring2016/papers/p5-sompolski.pdf)》）。

Arrow Compute的每个函数可以针对不同的硬件架构提供多个kernel，如X86的kernel会检测当前CPU支持的向量化指令，显式地使用SSE、AVX2、AVX512等SIMD命令进行计算，ARM kernel会使用NEON，GPU kernel会使用CUDA等等。

除数值计算外的大部分操作如聚合、类型转换、Join等操作都由Arrow Compute完成，在这些操作中解释型向量化的性能要优于预编译。

#### 编译器自动向量化

Apache Arrow的同一段数据会存储在连续的内存中，这为编译器自动向量化提供了良好的条件。在算子内部对Array进行一些轻量化的遍历操作时，依赖Gandiva或Arrow Compute都比较重，因此这时我们依赖编译器来实现自动向量化，在GCC的编译指令中加入`-ftree-vectorize -mavx2`即可开启。

#### 通过编译器提供的SIMD Intrinsics手动向量化

再需要进行复杂运算时，编译器的自动向量化就没有那么可靠了。比如我们的哈希算子使用Google的CityHash算法，涉及复杂的位运算和内存读取，我们使用的GCC 7.5并不能自动向量化。因此我们使用了编译器提供的intrinsics函数来手动向量化实现CityHash。相比于手写SIMD汇编指令，Intrinsics为我们提供了一个相对更好用的封装。

举个例子，这段代码是原生的Rotate函数和我们实现的向量化Rotate

```cpp
#include <immintrin.h>

// Bitwise right rotate.  Normally this will compile to a single
// instruction, especially if the shift is a manifest constant.
template <int shift>
__attribute__((always_inline)) inline uint64 Rotate(uint64 val) {
  // Avoid shifting by 64: doing so yields an undefined result.
  if constexpr (shift == 0) {
    return val;
  } else {
    constexpr int kLeftShift = 64 - shift;
    return ((val >> shift) | (val << kLeftShift));
  }
}

// 向量化实现
template <int shift>
__attribute__((always_inline)) inline __m256i AVXRotate(__m256i val) {
  // Avoid shifting by 64: doing so yields an undefined result.
  if constexpr (shift == 0) {
    return val;
  } else {
    constexpr int kLeftShift = 64 - shift;
    auto val_sr_reg = _mm256_srli_epi64(val, shift);
    auto val_sl_reg = _mm256_slli_epi64(val, kLeftShift);
    return _mm256_or_si256(val_sr_reg, val_sl_reg);
  }
}
```

#### 可扩展性

有时业务会出现业务定制化的复杂计算逻辑，无法通过组合现有算子表达。这是业务可以通过实现自定义算子去表达这部分逻辑。因为Riemann Compute使用了C++的[静态注册模式](https://dxuuu.xyz/cpp-static-registration.html)，用户只需要在自己的代码库实现算子后，上线服务链接实现的文件即可。

举个例子，如果用户想实现一个XyzOp，只需要在自己的代码库下创建XyzOp.cc文件：

```json
// XyzOp.cc（在业务自己的代码库里）
#include "mllib/compute/op.h"

class XyzOp: compute::Op {
// 实现对应的计算接口
};

REGISTER_OP(XyzOp);
```

然后在服务编译上线时链接XyzOp.cc文件即可，无需修改Riemann代码库下的代码也可以轻松地新建特征。

#### 正确性

Compute框架对提供的算子有100%的单元测试覆盖率，除了算子外，我们对配置解析、格式适配、计算图等组件也有完整的单元测试。同时Compute也有针对完整特征计算流程的端到端测试，保证特征计算的正确性。Compute还提供了特征Benchmark框架，可以便捷地查看各个特征在不同场景下的性能指标。

### 计算图

在Compute框架中，每个特征的计算过程都可以表示为一个树，每个节点都代表一个特征，每个字节点就是个依赖的特征。所有的“特征树”合并在一起构成了一个有向无环图（Directed Acyclic Graph，简称DAG），Compute称之为计算图。接下来我们介绍下计算图的执行中所用到的技术。

#### 自底向上执行模型

DAG的执行流水线有两种流派，一种是自顶向下执行，或称拉取模型（Pull-Based Model），在这种执行模型中，从根节点开始计算，当前节点向其依赖的上游节点拉取数据，拉到数据后进行当前节点的计算，这种模型常见于传统的数据库执行，如经典的火山模型，优点是实现简单，逻辑清晰。

而Riemann Compute选用的是自底向上模型，或称推送模型（Push-Based Model），这种执行模型从叶节点开始执行，计算出数据后传输给下游节点，触发下游节点的计算。我们选用Push-Based Model出于两点原因：

1. 多线程友好：在Pull Model中，根结点开始运行时，会先触发上游节点的计算任务，在多线程的场景下，当前节点可能需要通过条件变量不断轮询上游任务是否执行完毕。而Push Model则没有这部分损耗，因为只有当上游节点全部计算完毕后当前节点的任务才被创建，被创建后可以立即进行计算，计算完毕后即被析构。
2. 调度更容易：同理，由于Push Model中每个节点任务的生命周期是连续的，不会存在等待中的任务，任务之间都是相互独立的，不需要多线程通信机制，线程池调度起来会更简单，无需复杂的调度算法。
3. 减少重复计算：Push Model天然地可以实现子图共享机制，有效地避免重复计算，在下文中会详细介绍。

#### 子图共享

在特征计算过程中，常常会出现同一个中间层特征被多个特征依赖，比如将一个特征进行不同的聚合函数处理为多个特征。Riemann Compute可以自动地避免这些中间层特征被重复计算：

![](https://km.woa.com/asset/bb2620f8fb2848c3b02ef7172225f89a?height=1293\&width=2217\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

在这个例子中，Feature A和Feature B的计算过程中都需要计算X和Y两个中间层特征。Compute实现了自动的子图共享，相同的特征计算任务只会被创建一次，两个特征树会被合并为右侧的DAG。

#### 算子融合

在每个算子任务中，除了算子本身的计算过程外，还会有些附加的指令，比如校验输入值有效性、准备Gandiva输入值、检查是否可触发下游计算等等，这些overhead是无法避免的。尤其对于一些简单的算子，这些额外开销甚至会比计算本身更耗资源。

在实际的特征生产过程中我们观察到，在复杂算子如聚合前，用户经常会前置一些对输入进行简单的“预处理”的算子如取log、sigmoid、异常值过滤等。因此我们对大部分复杂算子都支持前置一个数值变换和一个过滤操作：过滤、数值变换和算子本身计算会融合在同一个任务中进行，避免了独立算子的额外开销。

举个例子，假如我们想对一个特征X过滤非法值后取log2，然后哈希得到特征结果，应当使用如下的算子组合：

```json
{
  "X_filtered": {
    "Op": "FilterOp", // 过滤算子
    "Filter": "v1 > 0", 
    "Dependency": ["X_raw"]
  },
  "X_filtered_log2": {
    "Op": "ProjectorOp", // 数值变换算子
    "Projector": "log(2, v1)",
    "Dependency": ["X_filtered"]
  },
  "X_final": {
    "Op": "CityHash64Op", // 哈希算子
    "Dependency": ["X_filtered_log2"]
  }
}
```

我们为简单的过滤和取对数操作中创建了额外的两个算子任务，大大增加了我们计算的不必要开销。在应用Riemann Compute的算子融合功能后，配置可以融合为一个算子：

```json
{
  "X_final": {
    "Op": "CityHash64Op", // 哈希算子
    "Filter": "v1 > 0", 
    "Projector": "log(2, v1)",
    "Dependency": ["X"]
  }
}
```

在支持了将过滤和数值变换作为附加在算子前的预处理操作后，我们可以将整个计算逻辑合并为一个算子，避免了算子的额外开销。

### 性能测试

我们在针对不同复杂度的特征，在不同的batch size下做了性能测试：

注：横坐标为单次计算的Item数，纵坐标为计算100次该特征所需要的毫秒耗时。对比的蓝线为我们旧的“行式”计算框架，即每个item的特征单独计算。运算环境都为单线程，由于计算过程没有io，计算耗时就约等于实际的CPU计算量。

这是一个简单的过滤+哈希逻辑计算出的特征，它的逻辑是可以全部被向量化的，因此我们看到Compute框架的性能要显著优于旧框架。且随着Item Size增长，行式计算的耗时是成倍增长的（Size300时的耗时是Size100耗时的3倍），而得益于列式计算，Riemann Compute的增长速度则显著更慢。在Item Size 300时，Compute耗时比旧框架节省了16倍之多。

这个特征是将多个预处理后的单值特征拼接成为一个列表特征，预处理过程包括过滤和数值转换。“将多个特征拼接在一起”是一种单行多列操作，“行式”计算有天然的优势，因此我们观察到在item size较小时Compute要慢于旧框架。但是得益于预处理阶段的列式计算，还有算子融合等其他的优化，在item size 300时Compute仍然取得了4倍的耗时下降。

### 未来规划

#### 完善算子，业务上线

当前Compute支持的算子数量还很少，聚合、交叉等算子还在规划开发中。因此，当前在视频号直播推荐业务上仍是新旧框架同时部署状态，待算子进一步完善后将全部迁移至Compute框架。

#### 预编译整个计算图

除了数值计算和过滤的操作使用了Gandiva预编译以外，Riemann Compute整体上是类似于Arrow Compute的解释型Interpreted vectorization架构。然而我们观察到，对于每个模型来说，其需要计算的特征是固定的，而每个特征的计算逻辑也是固定的。因此对于一个在线请求，其需要计算的整个DAG都是可以预先编译好的。

因此，如果我们能将每个模型的DAG预先内联并编译好，在线上可以省略掉构图和虚函数调用等操作，进一步优化我们的时延。

事实上，我们并不是唯一这样设想的团队。Facebook的paper[《Velox: Meta’s Unified Execution Engine》](https://vldb.org/pvldb/vol15/p3372-pedreira.pdf)中提到，其特征工程框架F3正在将计算的部分迁移到内部的统一计算引擎Velox。为了解决在线计算的时延问题，Velox正在实验预编译整个DAG的可行性。

#### 声明式DSL前端

以当前的设计而言，Compute更像是一个特征计算后端而非完整的特征工程框架。原因是用户当前需要完整地列出整个计算过程，包括每一步的计算逻辑和存储结果的中间变量。因此，上文提到的算子融合、子图共享等技术也需要用户手动去控制。

事实上，用户应当提供的是特征的“计算逻辑”而非当前的“执行计划”，我们可以为Compute封装一个声明式语言前端（比如SQL或自己设计的DSL），将用户提供的计算逻辑编译为后端的执行计划，在这个过程中自动化地实现算子融合、子图共享等优化。

#### 通用计算引擎

在LLVM项目出现之前，每个编译器都是一个“庞然大物”，而LLVM的出现使得编译流程变得流水线化，开发者可以自由的组合语言前端、优化流和后端来构造新的语言和编译器。

在数据处理系统中，类似的“标准化”运动也正在逐渐流行，目前为止的数据系统几乎都是不可拆分的，不论是传统数据库如PostgreSQL、MySQL，还是大数据处理系统Spark、Flink，或是最近的OLAP数据库如Clickhouse等，都完整地包含了自己的语言前端、执行引擎和存储系统。在未来，我们预期开发者可以通过自由组合这些组件来构造自己的数据处理系统。

Apache Arrow团队的成员推出了[Substrait项目](https://substrait.io/)，Substrait类似于一个“执行IR”，前端将自己的计算逻辑（如SQL或其他DSL）翻译为Substrait标准，然后后端可以部署任何一个符合Substrait标准的通用计算引擎。

上文提到的Facebook的Velox和Apache Arrow的Compute框架都是类似的通用计算引擎，而我们也在探索微信内部的数据处理生态是否需要一个通用的计算引擎，以及Riemann Compute是否有成为通用计算引擎的潜力。

### 开源贡献

在引入Apache Arrow和其众多组件的过程中，我们团队也为Arrow社区做出了一定的贡献。比如为Gandiva设计了一套语法和解析器、为Gandiva适配LLVM 15新版的IR语法、以及诸多的性能优化和bugfix，我们总计为Arrow提交了超过20个PR。在后续工作中我们将继续为Apache Arrow社区提交代码。
