# rapidudf

### 简介 <a href="#e7-ae-80-e4-bb-8b" id="e7-ae-80-e4-bb-8b"></a>

RapidUDF是针对在线服务应用场景设计的高性能计算引擎; 基于JIT和SIMD技术, 在相应的在线服务计算场景, 可以极大地提升服务性能和事务处理能力；

### 背景

近段时间，一直在思考如何在现代硬件上优化在线数据密集型系统的性能；尤其是像推荐系统这样的应用场景, 索引和存储一般也属于此类系统。这类系统通常面临着极高的并发请求量和海量的数据处理需求，它们需要在毫秒级时间内完成复杂的数据分析和个性化推荐结果的生成。为了更好地理解并解决这些问题，我们可以将此类系统的典型场景概括如下：

* **实时性**; 在线数据密集型系统特别强调实时性, 即系统必须能够快速响应用户的每一个操作，并即时提供准确的服务或信息。
* **密集的数据**; 例如，在电商网站中，当用户浏览商品页面时，后台需要立即根据用户的浏览历史、购买记录等信息计算出最可能引起兴趣的商品列表; 浏览历史可能以万计, 商品集合则可能在百万以上;
* **复杂计算**; 随着人工智能技术的发展，越来越多的高级算法被应用于此类系统中，以提高推荐的精度和用户体验。然而，这也带来了更高的计算资源消耗问题，因此如何在保证服务质量的同时有效控制成本，成为了另一个重要的研究方向。

比较早的时候,我们提出了imos方案,用于解决针对存储索引场景的统一架构; 而对搜推在线服务, 稍晚一些我们则构建了ispine,提供了一套标准统一的编程模型; 不过这些都是在高层架构/调度编排层面做的工作, 而细化具体到算子/任务计算实现上, 由于一般这些的实现粒度较粗, 目前基本上是不同开发不同业务场景都是百花齐放的不同风格实现;

针对算子/任务计算实现统一和优化, 对于某些相对通用的场景, 目前也有不同团队做了相应的应用和探索, 例如规则引擎, 特征抽取计算等; 我们希望在此类工作基础的思路上进一步扩展, 外推应用到更广泛的领域;

### 现有挑战

深入到相关在线数据密集型系统内部实现上,我们可以看到, 绝大部分处理是在内存中一组类似的数据上进行; 例如搜索推荐系统一般是在用户特征+一组候选Item特征之上的计算筛选; 索引服务的前后过滤聚合等都是在一个候选list上计算操作; 另外从编程范式上一般可以概括为**面向对象的命令式过程式编程**。 大致上流程如下图 ![](https://i.imgur.com/e9II0yp.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

另外这是推荐系统里一段实际代码, 典型地表明了相关特点：

```cpp
// 设置加权分数
void FillBoostScore(const std::unordered_map<uint64_t, BoostItem> &boost_items_map, int64_t algm_score_key,
                    std::vector<trpc::one_piece::idl_common::ItemInfo> &item_list) {

  std::unordered_map<int, double> origin_score_map;
  for (int index = 0; index < item_list.size(); ++index) {
    auto &item_info = item_list[index];
    if (item_info.attr().empty()) {
      continue;
    }
    auto iter_score = item_info.mutable_attr()->find(algm_score_key);
    if (iter_score != item_info.mutable_attr()->end()) {
      origin_score_map[index] = iter_score->second.val_d();
      item_info.mutable_attr()->erase(iter_score);
    }
  }

  for (auto &[item_id, boost_item] : boost_items_map) {
    // 总分数赋值
    trpc::one_piece::idl_common::ItemAttr attr_score;
    attr_score.set_val_d(boost_item.rank_score);
    auto &map_attr = *(item_list[boost_item.item_index].mutable_attr());
    map_attr[algm_score_key] = attr_score;
    ISPINE_DEBUG("FillBoostScore id: {}, score: {}", item_id, boost_item.rank_score);
  }
}
```

随着数据量增加以及算法逻辑的增加， 上述模式的缺点逐渐面临以下挑战：

* **计算性能不足**：传统计算引擎在处理大规模数据时性能不足，无法满足实时计算的需求
* **过程式编程导致算子实现分散**，只能逐个优化，但这个在现实工程实现中几乎不可能，非常容易导致整体系统性能退化
* **扩展性差**: 多数实现与业务场景协议字段绑定，很难复用；

为了应对上述挑战，我们基于现有的领先技术探索新的方法来提高在线线数据密集型系统服务的性能。其中，**JIT**（Just-In-Time）编译技术、**SIMD**（Single Instruction Multiple Data）技术和**函数式编程范式**在高性能计算领域展现出巨大的潜力。**JIT编译**可以在运行时生成优化的机器码，**SIMD**技术可以通过并行计算提高数据处理速度，而融合**函数式编程**则通过低阶函数和高阶实现提高代码的可读性和可维护性;

### 设计与实现

首先从常见的规则引擎出发，我们期望其具备以下特性能力：

* **完整的数学计算/布尔表达式支持**
* **针对复杂逻辑类c++的script DSL支持**
  * while循环
  * if-elif\*-else分支
  * auto临时变量
* **与C/C++零抽象开销交互**
* **针对列式存储数据(vector)的SIMD加速实现**

针对表达式引擎，业内目前已经有不少类似实现，如java中著名的[drools](https://www.drools.org/); c++中目前[exprtk](https://github.com/ArashPartow/exprtk)应用较多，也有部分[muparser](https://beltoforion.de/en/muparser/)的使用；不过都不具备以上能力，也有各自的缺陷； 这里以exprtk为例，列举下其不足导致应用场景有限（大家可看下是否如此）：

* 线程不安全
* 只支持有限的数据类型（double，vector\<double>,..）;不支持结构化类型
* 不支持SIMD
* 解释型实现性能尚可，但没达到native cpp性能（约80%，复杂情况下更慢）

为了实现上述目标，**RapidUDF** 选择重新思考实现表达式/DSL能力；

#### DSL与JIT

**RapidUDF**选择了基于LLVM开发表达式/DSL支持；这一选择非常自然：

* **JIT编译**比解释在性能上优势明显；也可能比预编译更能利用运行环境的硬件能力；
* **LLVM**是当前领先的开发类似JIT的选择;

![](https://i.imgur.com/mQPSLz5.jpeg?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

当然现实中这里也走了弯路， 最开始采用[xbyak](https://github.com/herumi/xbyak)用于具体实现，结果发现在手写的优化阶段很难达到GCC/Clang的生成代码性能；目前默认开启的LLVM的优化pass如下，基本达到或者超过GCC加上`O2`的优化效果：

* **InstCombinePass** 通过识别和替换冗余的指令序列，减少指令的数量，从而提高代码的效率
* **ReassociatePass** 通过重新排列加法和乘法操作，使得编译器可以更好地利用处理器的并行性和优化机会
* **GVNPass** 通过给每个表达式分配一个唯一的编号，识别出相同的表达式并将其结果复用
* **SimplifyCFGPass** 用于简化控制流图（CFG），通过合并基本块、删除无用的跳转和条件分支，使控制流更加清晰和高效
* **PartiallyInlineLibCallsPass** 通过将库函数的部分代码直接插入到调用点，减少函数调用的开销
* **MergedLoadStoreMotionPass** 用于优化内存访问顺序，通过重新排列加载和存储指令，减少内存访问的延迟和冲突
* **TailCallElimPass** 用于尾递归优化，通过将尾递归调用转换为循环，减少函数调用的开销，避免栈溢出
* **LoadStoreVectorizerPass** 用于向量化加载和存储指令，通过将多个加载和存储操作合并为一个向量指令，提高内存访问的效率
* **VectorCombinePass** 用于合并和优化向量指令。它通过识别和合并相似的向量指令，减少不必要的中间结果和临时变量，提高向量代码的效率
* **LoopVectorizePass** 用于循环向量化。它通过识别循环中的并行性，将循环体中的标量指令转换为向量指令，从而提高循环的执行效率。
* **SLPVectorizerPass** 用于超级字级并行（Superword-Level Parallelism, SLP）向量化。它通过识别和合并相似的标量指令，生成高效的向量指令，从而提高并行性和计算效率。

**RapidUDF** 中定义的DSL接近cpp语法，类似如下代码就是一个合法的DSL文件，有三个function，可以看到和普通cpp语法非常接近:

```javascript
    int ifelse_func(int x){ 
      if(x>10){
         return 20;
      }elif(x > 5){
       return 10;
      }else{
        return 0;
      }
    }
    int while_func(int x, int y){ 
      while(x > 0){
        y = y + 10;
        x = x-1;
      }
      return y;
    }
    simd_vector<f32> get_duration_score(Context ctx, simd_vector<f32> duration, f32 alpha, f32 beta)
    {
        auto x = (duration-alpha)/beta;  // auto声明变量
        return 1.0_f32/(1_f32 + exp(-x));
    }
```

DSL会在运行期编译成机器码，通过函数指针开放给宿主C/C++代码调用；以下是一个简单例子：

```javascript
  std::string source = R"(
    int fib(int n) 
    { 
       if (n <= 1){
         return n; 
       }
       return fib(n - 1) + fib(n - 2); //递归调用
    } 
  )";

  // 3. 编译生成Function,这里生成的Function对象可以保存以供后续重复执行
  rapidudf::JitCompiler compiler;
  // CompileExpression的模板参数支持多个，第一个模板参数为返回值类型，其余为function参数类型
  auto result = compiler.CompileFunction<int, int>(source);
  if (!result.ok()) {
    RUDF_ERROR("{}", result.status().ToString());
    return -1;
  }

  // 4. 执行function
  rapidudf::JitFunction<int, int> f = std::move(result.value());
  int n = 9;
  int x = f(n);  // 34
```

#### FFI: 与C/C++之间零抽象开销交互

由于表达式/DSL并非完整的语言，其中很重要的能力是与宿主语言的**FFI**（Foreign Function Interface）交互； ![](https://i.imgur.com/ZlYqXXR.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

FFI实现最理想的状态是零抽象开销的访问，但现实中大多数类似实现是有很大开销的，在数据密集型系统中并不适用；典型的几个例子都有建议在非频繁调用场景下使用：

* CGO
* JNI
* WASM
* LuaJIT FFI

**RapidUDF**中采用了基于半自动反射实现和LLVM IR结合的能力实现了一个零抽象开销**FFI**交互实现；

**DSL访问POD对象**

对于POD类型的struct，基于宏和模板获取各个struct/field的类型meta信息，大致代码如下：

```cpp
struct TestInternal {
  int a = 0;
};
RUDF_STRUCT_FIELDS(TestInternal, a)
struct TestStruct {
  int a = 0;
  TestInternal internal;
  TestInternal* internal_ptr = nullptr;
  std::vector<int> vec;
};
RUDF_STRUCT_FIELDS(TestStruct, internal, internal_ptr, a, vec)
```

在构建LLVM IR过程中，对于形如`a.b.c`的表达式/DSL可以基于上述的meta信息则可结合LLVM [**GEP**](https://llvm.org/docs/GetElementPtr.html)指令生成相当于直接指针偏移访问的机器码； 例如针对上述结构的`x.internal.a`表达式最终生成的IR为：

```javascript
define i32 @rapidudf_expresion(ptr %x) {
exit:
  %0 = getelementptr inbounds i8, ptr %x, i64 4
  %1 = load i32, ptr %0, align 4
  ret i32 %1
}
```

**DSL调用C/C++方法**

这里相对简单，只需要将函数指针注册到一个索引中，LLVM IR中基于name生成对应的方法调用即可（注意，实现是JIT编译期查找，运行期直接函数指针调用）； cpp中定义注册方法：

```cpp
static int test_user_func() { throw std::logic_error("aaa"); }
RUDF_FUNC_REGISTER(test_user_func) 
```

在构建LLVM IR过程中，对于形如`test_user_func()`的表达式生成相当于直接函数指针调用的机器码：

```javascript
declare i32 @test_user_func()

define i32 @rapidudf_expresion() {
entry:
  %0 = alloca i32, align 4
  ; 调用函数指针
  %1 = call i32 @test_user_func()
  store i32 %1, ptr %0, align 4
  br label %exit

exit:                                             ; preds = %entry
  %2 = load i32, ptr %0, align 4
  ret i32 %2
}
```

**DSL调用C++类成员方法**

对cpp类成员方法的调用实现原理上和调用普通C/C++方法一致，不过由于需要支持形如`a.func()`形式，需要将成员方法转成普通函数指针；GCC编译器下是可以类似如下方式直接强制转换的，不过clang下则无法编译；

```cpp
void MyClass::Call(int a);
//GCC下等价
void MyClassCall(MyClass* c, int a);
```

目前实现是借助宏和模板编译期生成一个中间C风格函数：

```cpp
struct TestStruct {
  void test_funcx() { throw std::logic_error("aaa"); }
};
RUDF_STRUCT_MEMBER_METHODS(TestStruct, test_funcx) //注册类方法
```

构建LLVM IR过程中，对于形如`x.test_funcx()`的表达式生成相当于直接函数指针调用的机器码：

```javascript
declare void @TestStruct_test_funcx(ptr)
define void @test_func(ptr %x) {
exit:
  tail call void @TestStruct_test_funcx(ptr %x)
  ret void
}
```

**DSL访问Protbuf/Flatbuffers对象**

对于protobuf/flatbuffers，原理与上述**DSL调用C++类成员方法**实现类似，也是借助宏和模板生成一个中间C风格函数； **flatbuffers**由于生成的get方法名其实就是字段名，可以直接复用宏**RUDF\_STRUCT\_MEMBER\_METHODS** 对于fbs定义：

```protobuf
namespace test_fbs;

table Item {
  id:uint;
}
table FBSStruct {
  id:uint;
  str:string;
  item:Item;
  strs:[string];
  items:[Item];
  ints:[uint];
}
root_type FBSStruct;
```

对应注册宏

```cpp
RUDF_STRUCT_MEMBER_METHODS(::test_fbs::Item, id)
RUDF_STRUCT_MEMBER_METHODS(::test_fbs::FBSStruct, id, str, item, strs, items, ints)
```

构建LLVM IR过程中，对于以下的DSL：

```cpp
    int test_func(test_fbs::FBSStruct x){
      return x.id();
    }
```

生成相当于直接函数指针调用的机器码：

```javascript
declare i32 @"test_fbs::FBSStruct_id"(ptr)

define i32 @test_func(ptr %x) {
exit:
  %0 = tail call i32 @"test_fbs::FBSStruct_id"(ptr %x)
  ret i32 %0
}
```

**protobuf**由于生成的get方法名相当于`get_<field>`, 因此这里定义了另外的宏**RUDF\_PB\_FIELDS**：

```cpp
RUDF_PB_FIELDS(::test::Item, id)
RUDF_PB_FIELDS(::test::PBStruct, id, str, ids_array, strs_array, item_array, item_map, item_map, str_int_map)
```

其它实现与flatbuffers类似；

上述实现在最终生成的机器码和cpp中硬编码访问区别在于native cpp中的代码会在`O1`优化下内联相关方法访问，而JIT实现由于是运行期编译并不能实现相应能力(无相关代码)； 不过增加的一次C方法调用绝大多数情况并不会造成多少性能折损，如果这里存在瓶颈，就需要采用前述**POD**来作为复杂数据参数传递了；

**DSL访问STL以及其它类型对象**

通常这里实现都是基于**DSL调用C++类成员方法**原理扩展的实现，只是内置实现了以下STL容器访问，这里不再赘述：

* std::vector
* std::map
* std::set
* std::unordered\_map
* std::unordered\_set

#### 向量化计算

所谓向量化计算，一般指单指令多数据对应一类并行架构（现代CPU一般都支持SIMD执行），单条指令同时作用于多条数据流，可成倍的提升单核计算能力。SIMD非常适合计算密集型任务，它能加速的根本原因是“从一次一个跨越到一次一组，从而实现用更少的指令数完成同样的计算任务。” ![](https://i.imgur.com/qZbX5lP.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

**RapidUDF**中的向量化计算能力基于以下两者共同完成：

* LLVM JIT
* [highway](https://github.com/google/highway) & [sleef](https://sleef.org/)

业内有类似的基于LLVM JIT的向量化计算实现工作，如[Arrow Gandiva](https://arrow.apache.org/docs/cpp/gandiva.html)；facebook也有对于存储的向量化计算加速库[velox](https://github.com/facebookincubator/velox); **RapidUDF**的向量化计算实现与这两者都不一样，这里简单分析对比下，如有错误，还请纠正：

**Arrow Gandiva**

![](https://i.imgur.com/B8tBFk2.jpeg?imageMogr2/thumbnail/1540x%3E/ignore-error/1) 基于此文[gandiva-using-llvm-and-arrow-to-jit-and-evaluate-pandas-expressions](https://blog.christianperone.com/2020/01/gandiva-using-llvm-and-arrow-to-jit-and-evaluate-pandas-expressions/)的分析, gandiva实际上生成的是标量计算代码，依赖LLVM的向量化优化Pass来转成向量化计算代码：

```javascript
; Function Attrs: alwaysinline norecurse nounwind readnone ssp uwtable
define internal zeroext i1 @less_than_float32_float32(float, float) local_unnamed_addr #0 {
  %3 = fcmp olt float %0, %1
  ret i1 %3
}
; Function Attrs: alwaysinline norecurse nounwind readnone ssp uwtable
define internal zeroext i1 @greater_than_float32_float32(float, float) local_unnamed_addr #0 {
  %3 = fcmp ogt float %0, %1
  ret i1 %3
}
(...)
%x = load float, float* %11
%greater_than_float32_float32 = call i1 @greater_than_float32_float32(float %x, float 2.000000e+00)
(...)
%x11 = load float, float* %15
%less_than_float32_float32 = call i1 @less_than_float32_float32(float %x11, float 6.000000e+00)
```

Gandiva中主要依赖以下两个向量化优化pass:

* **SLPVectorizerPass**
* **LoopVectorizePass**

这里的局限有几点：

* 自动向量化限制太多，实际情况下稍微复杂的情况自动向量化都会失败
* 内置的计算函数有限，大多数复杂些的math函数都不支持自动向量化；
* 有限的LLVM内置计算函数实现很多与SOTA实现也有比较大的差距（如sin/cos等）

由此Gandiva的生成代码性能很难达到理论上SIMD加速比；

**Velox**

velox的向量化计算实现则与LLVM无关，本质上是针对构建的语法树的一个解释型实现；具体分析可参考[Velox表达式计算原理调研](https://developer.aliyun.com/article/1322080)

![](https://i.imgur.com/YcNGV1m.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

由于velox主要针对离线大数据系统设计，如果在线系统场景，存在以下关键不足：

* 解释型在简单表达式下性能折损偏大
  * velox也有实验性的codegen实现（依赖完整编译器工具运行期生成c++代码）
* 复杂的向量表达式无法自动融合operator（ `x + (cos(y - sin(2 / x * pi)) - sin(x - cos(2 * y / pi))) - y`）

所谓**无法自动融合operator**指的是形如`a*b+c`这样的计算，velox的实现是先对完全列数据计算`tmp=a+b`，然后再计算`tmp+c`; 越复杂的表达式引入的临时变量越多，从而带来的对于临时向量的`load/store`指令越多；此现象对于离线大数据系统可能并非问题，不过对于在线系统, 这会导致实际加速比逐步降低，更复杂的表达式场景甚至会低于标量计算+ `GCC -O2`的实现；

**无法自动融合向量化operator**问题也出现在多数类似实现中；

**RapidUDF**

**RapidUDF**的向量化计算类似`Gandiva`实现，不过不同的是在IR构建阶段就生成向量化计算代码而非标量； 另外对于复合运算，只要是一个完整的表达式，会自动融合到一个复杂向量计算步骤中；

整体来说，**RapidUDF**的向量化计算主要采用以下技术：

* [LLVM Vector JIT](https://llvm.org/docs/LangRef.html#t-vector)
  * 这里的vector指的是IR语言中支持的类型，和std::vector是两种不同概念
  * 构建IR时会直接基于向量类型作为运算数据基本类型
  * 支持标量和向量的混合计算
* [highway](https://github.com/google/highway)
  * 大多数复杂数学计算向量化实现
  * 运行期基于运行环境硬件规格动态dispatch
* [sleef](https://sleef.org/)
  * 部分highway不支持的数学计算向量化实现
* 复杂向量计算自动融合
  * 形如`a*b+c`这样的计算，生成的机器码是直接在一个循环中向量化展开所有向量操作

**RapidUDF**支持自动融合任意复杂的向量表达式： ![](https://i.imgur.com/IcCoV2e.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

以上设计会保证常规复杂计算在SIMD向量化加持下的加速比相当于正常理论的加速比，实际验证情况还有略超过理想态的加速比情况；这一部分会在**性能评估**一节里进一步讨论；

#### 新的编程模型探索

以上主要讨论具体执行效率的优化，并未涉及编程模型的讨论；回到初始问题，针对数据密集型系统， 绝大部分处理是在内存中一组类似的数据上进行，而以搜推广在线系统为例，在搜索推广算法常见的特征提取、召回、打分、排序流程中，计算类似与二维表结构的数据操作。常见的计算逻辑：约2000个商品信息，每个商品信息包含几十个字段，对这样的列表进行计算分数、排序、去重、与其他数据合并等操作。

而对目前的多数实现代码分析，可以观察到，当前各种算子实现基本特点：

* 面向对象的命令式过程式模式
  * 混合数据操作和业务逻辑，逻辑复杂且含义不清，修改时容易出错，运行时容易产生大量内存碎片而拖累整体性能
* 绝大部分是行式逐个物品处理

对这些数据操作分析后可以发现，大部分操作是SQL类操作，例如过滤、排序、合并、分组计算等，少部分是难以抽象的复杂业务逻辑，典型如商品混排，不过也可以利用前述的规则引擎类的高层DSL抽象实现解决；

同时参考Spark的DataFrame我们设计了**SIMD Table**用于表示任意数据集，由于需要支持向量化计算，实际内存表示是列式存储方式：

![](https://i.imgur.com/uYJwYuf.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

业内如阿里也有类似的工作([TPP DataFrame](https://developer.aliyun.com/article/933235))，不过是基于java且暂无无向量化加速能力；

**SIMD Table**是一个运行期创建类DB中的Table, 不过在JIT中则是编译期确定；**SIMD Table**结合动态声明与静态编译访问能力能力，理论上可以基于此实现更通用的算子，避免当前一些与业务协议绑定的算子复用问题；

**SIMD Table**支持类似Spark DataFrame的类似操作，目前支持：

* **.filter(simd::Vector\<Bit>)** 返回过滤后新table
* **.order\_by(simd::Vector\<T> column, bool descending)** 返回重排后的新table
* **.topk(simd::Vector\<T> column, uint32\_t k, bool descending)** 返回topk后的新table
* **.group\_by(simd::Vector\<T> column)** 返回group\_by后的tables

**SIMD Table**支持以protobuf/flatbuffers的schema描述创建和构造实例；

我们期望基于上述数据结构在在线数据密集系统中探索以下新的编程模式：

* 类SQL的基于数据集Table的DSL声明式计算操作
* 类spark的函数式编程模式

**注意**：此部分仍然处于早期探索阶段，较多相关技术点尚未设计稳定和实现；

### 性能评估

这里选择了几个不同应用场景分别验证与传统实现的性能对比：

#### 简单规则引擎

这里找了个java规则引擎[例子](https://segmentfault.com/a/1190000042635731), 基于以下规则例子：

根据订单的金额来动态判断该加多少积分： 小于100元，不加积分。 100到500元，加100积分。 500到1000元，加500积分。 1000元以上，加1000积分。

首先定义一个只有1个字段的Order POD：

```cpp
struct Order {
  int amount = 0;
};
RUDF_STRUCT_FIELDS(Order, amount)
```

对应硬编码的测试代码:

```cpp
static int __attribute__((noinline)) native_order_rule(int amount) {
  if (amount < 100) {
    return 0;
  } else if (amount >= 100 && amount < 500) {
    return 100;
  } else if (amount >= 500 && amount < 1000) {
    return 500;
  } else {
    return 1000;
  }
}

static void BM_native_order_rule(benchmark::State& state) {
  int i = 100;
  size_t total = 0;
  for (auto _ : state) {
    Order order;
    order.amount = i + 100;
    auto result = native_order_rule(order.amount);
    total += result;
    i++;
  }
  RUDF_DEBUG("{}", total);
}
```

而**RapidUDF**的实现：

```cpp
static void rapidudf_order_rule_setup(const benchmark::State& state) {
  std::string source = R"(
    int rule_func(Order order)
    {
      if (order.amount < 100) {
        return 0;
      } elif (order.amount >= 100 && order.amount < 500) {
        return 100;
      } elif (order.amount >= 500 && order.amount < 1000) {
        return 500;
      } else {
        return 1000;
      }
    }
  )";
  rapidudf::JitCompiler compiler;
  auto result = compiler.CompileFunction<int, Order&>(source);
  g_order_rule_func = std::move(result.value());
}

static void BM_rapidudf_order_rule(benchmark::State& state) {
  int i = 100;
  size_t total = 0;
  for (auto _ : state) {
    Order order;
    order.amount = i + 100;
    auto result = g_order_rule_func(order);
    total += result;
    i++;
  }
  RUDF_DEBUG("{}", total);
}
```

实际的测试结果：

```javascript
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
BM_rapidudf_order_rule              2.79 ns         2.79 ns    250735873
BM_native_order_rule                2.17 ns         2.17 ns    322400204
```

可以看到在简单规则下，**RapidUDF**的实现性能相当于`O2`优化编译的实现；

#### 纯计算表达式场景与Exprtk的比较

![](https://i.imgur.com/erq1NQc.png?imageMogr2/thumbnail/1540x%3E/ignore-error/1)

这里选择的几个用例从[exprtk的benchmark代码](https://github.com/ArashPartow/exprtk/blob/master/exprtk_benchmark.cpp)中选择; 从中可以看出越复杂的表达式，exprtk相对于native实现越慢；而RapidUDF基本和native实现一个水准甚至可能略快；

#### 特征计算向量化Wilson Ctr

这里选择的是特征抽取库里一个特征的计算逻辑，测试在支持avx2的cpu上运行，**GCC8.5** 编译优化开关`O2`，**float数组**长度为`4099`; 原始cpp函数原型以及benchmark代码为：

```cpp
float wilson_ctr(float exp_cnt, float clk_cnt) {
  return std::log10(exp_cnt) *
         (clk_cnt / exp_cnt + 1.96 * 1.96 / (2 * exp_cnt) -
          1.96 / (2 * exp_cnt) * std::sqrt(4 * exp_cnt * (1 - clk_cnt / exp_cnt) * clk_cnt / exp_cnt + 1.96 * 1.96)) /
         (1 + 1.96 * 1.96 / exp_cnt);
}
static void BM_native_wilson_ctr(benchmark::State& state) {
  double results = 0;
  for (auto _ : state) {
    for (size_t i = 0; i < test_n; i++) {
      results += wilson_ctr(exp_cnt[i], clk_cnt[i]);
    }
  }
  RUDF_DEBUG("Native _wilson_ctr result:{}", results);
}
```

对应的vector udf脚本实现以及benchmark代码为:

```cpp
    simd_vector<f32> wilson_ctr(Context ctx, simd_vector<f32> exp_cnt, simd_vector<f32> clk_cnt)
    {
       return log10(exp_cnt) *
         (clk_cnt / exp_cnt +  1.96 * 1.96 / (2 * exp_cnt) -
          1.96 / (2 * exp_cnt) * sqrt(4 * exp_cnt * (1 - clk_cnt / exp_cnt) * clk_cnt / exp_cnt + 1.96 * 1.96)) /
         (1 + 1.96 * 1.96 / exp_cnt);
    }
    
    static void BM_rapidudf_vector_wilson_ctr(benchmark::State& state) {
        double results = 0;
        rapidudf::Context ctx;
        for (auto _ : state) {
            ctx.Reset();
            auto result = g_vector_wilson_ctr_func(ctx, exp_cnt, clk_cnt);
            RUDF_DEBUG("{}", result.Size());
        }
    }
```

由于这里涉及复杂函数调用`log10/sqrt`, 编译器优化是不会开启自动向量化的； 理论上理想加速比应该为avx2寄存器位宽对于float位宽的倍数8; 不过实际运行结果如下，可以看到加速甚至超过了8倍，达到了**10.5**

```javascript
Benchmark                               Time             CPU   Iterations
-------------------------------------------------------------------------
BM_native_wilson_ctr                69961 ns        69957 ns         9960
BM_rapidudf_vector_wilson_ctr       6661 ns         6659 ns        105270
```

额外的差距可能在于：

* LLVM部分计算优化优于GCC
* 复杂数学计算log10/sqrt实现优于math lib

### 代码地址

[RapidUDF](https://github.com/yinqiwen/rapidudf)

### 未来工作

**RapidUDF**仍然处于初步完成阶段，仍然有许多待完成的能力；未来计划的进一步工作：

#### 规则引擎/表达式引擎升级替换

目前比较直接的是可以平替升级目前散落在各个模块系统中使用的类规则引擎/表达式引擎实现， 以exprtk/muparser为主； 部分场景还可以继续改造成列式内存SIMD加速计算，事实上我们选择了一个在推荐混排中针对2000个物品的muparser公式打分场景改造成**RapidUDF向量化Expr**，最终加速效果从**65ms**降至**100us**

#### 特征计算

明显提升较大的场景，不过对于现有实现改动过大，也涉及部门合作，需要进一步考虑计划；

#### Table与类spark/sql的新编程模式

进一步探索类spark/sql的新编程模式，完善Table能力;
