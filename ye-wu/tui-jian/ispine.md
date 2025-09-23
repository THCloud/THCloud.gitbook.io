---
description: 代码：https://github.com/yinqiwen/myctk/tree/master
---

# ispine

### 背景 <a href="#e8-83-8c-e6-99-af" id="e8-83-8c-e6-99-af"></a>

推荐搜索类在线服务由于面向的场景是数据密集+计算密集，通常由C++实现；而且由于算法策略越来越复杂，算法开发同学也深度参与负责对应的在线算法开发，传统的基于RPC框架开发模式已经不适应业务需要； 为了提升开发效率，改善服务质量，我们设计实现了一套针对推荐搜索等算法工程类在线服务的开发框架，用于为所有类似场景服务开发提供一套标准统一的编程模型。

### 设计目标与核心思路

#### 抽象过程：在线算法类服务

忽略具体算法实现细节过程，我们可以认为一个具体的在线算法服务就是一个具备输入和输出的函数实现；进一步细化，可以将这个函数实现拆解为N个独立的计算执行单元（算子）；

纯粹从工程架构角度而言，整个过程可以看做是将给定的数据（输入的RPC请求，配置，词典索引等），输入给指定的算子集合（可以看做一个计算公式），执行后得到结果；到这里，可以看到这里已经非常类似机器学习模型执行过程了（算法同学应该有更深的理解）；

所以，在实际业务场景中，这里面临的架构问题实际是两个：

* 算子集合的执行编排（如何有序执行一组算子的集合, 像不像模型训练？）
* 数据的准备和如何供给给给定的算子（不同的算子需要的数据不一致，像不像特征抽取准备？）

针对第一个问题，其实可以进一步展开成三个小问题：

* 算子如何定义？
* 算子集合如何编排？
* 编排后的算子集合如何执行？

针对第二个问题，也可以进一步展开成两个小问题：

* 数据的定义
* 数据如何在算子间高效传递

以下讨论我们针对这些问题的思路和一些实践；

#### 开发框架的思路

**旧有的框架思路**

首先我们可以看下面对类似的问题业内的解决方案；

对于第一个算子问题，机器学习框架如Tensorflow/Pytorch的解决方案是基于图的执行，将各个算子编排成一个计算图，训练/预测过程执行计算图即可得到结果； 而对于第二个数据问题，所有的数据都是类似tensor这种标准格式数据，算子间数据都以这种格式传递数据；

而针对在线服务场景，常见的做法一般也是基于DAG的思路，只是将算子执行逻辑抽象为function视为一等公民，并未针对算子的数据有设计上考虑；其特点一般如下：

* 一个算子作为一个可调度的单元；
* 算子需要的数据依赖算子的实现方在算子实现中硬编码获取；
* 这里通常有一个全局Context类用于全流程存放以及协调各种数据的存取；
* 通过流程配置定义DAG执行图，并发、串行都在配置中约束

早期在推荐搜索类业务，我们就是基于这种思路构建的DAG框架，其它业内也有类似的做法，例如：

* [优酷的图执行引擎的算法服务框架](https://cloud.tencent.com/developer/article/1645763)
* [美团算法平台在线服务体系](https://tech.meituan.com/2021/05/13/turing-os-online-serving.html)

长期的使用过程中(2017-2020)，我们逐渐发现这类实现有如下的不足：

1. 数据读写安全与性能：全局Context类用于全流程存放以及协调各种数据的存取，在并发场景下需要小心的控制对同一个数据的读写访问；一般都需要加锁，开发门槛过高，性能也有一定的损失；
2. 算子并发度：较多算子组成的图，很难控制最优的并发度；尤其在一个较多各种abtest实验分支的情况下，若需要达到最优并发度，在人工维护的DAG流程执行图中，需要较高的维护理解成本；
3. 数据的管理：定义在全局Context类中的数据，很难知道是哪个算子读或者写，使用时需要全局搜索确认使用情况；开发理解成本很高；
4. 算子的维护：由于数据的使用硬编码在代码中，很难直观的知道该算子使用或者产出了那些数据（除非通读理解整个代码），对后续维护者以及其它非当前算子的开发者不友好；

百度Feed推荐内部有一个图执行引擎实现，据说解决了问题1与2； 他们的思路是：

* 基于函数编程的思路，算子即函数
  * 函数的输入数据都是只读的；
  * 函数的输出也是只写一次，后续都是只读的；
* 框架通过配置组图，需要配置算子的数据依赖（不用显式配置算子间的调用顺序）；
* 图执行则通过数据驱动
  * 如数据A ready，则依赖于该数据的算子都可判断是否可以执行；
  * 算子的所有数据依赖都ready，则该算子开始执行；

从上可以看出，这是一个动态图实现；我们有同学曾尝试引入这种框架，但在实践过程中发现有以下的局限：

* 动态图无法直观可视化展示（相比流程驱动的静态图），不便于debug；
* 无法从流程驱动的代码平滑过渡迁移到数据驱动（需要大量的算子代码适配改造）；
* 数据使用严格要求算子的代码和配置一致，遗漏会导致undefined的行为；
* 仍然没有解决好算子+数据的管理问题；数据的使用和发布强制通过某种框架定义的编程范式约束，等同于算子代码实现绑定在了框架实现上；

**新框架的思路和设计原则**

类似所有的新轮子的诞生故事，老的架构虽能在一些场景工作，但在更大的业务场景中局限颇多；针对需求和老框架的种种问题，我们开始构建了新的推荐搜索服务开发框架iSpine，其基本指导思路是：

* 算子与框架解耦；便于旧有算子代码移植到ispine框架中，也便于后续升级新框架；
* 算子与算子解耦；单独算子只需要定义数据接口；
* 静态图模式，便于测试和可视化呈现，健壮性也更强一些；
* 图定义支持数据驱动和流程驱动两种模式；
  * 数据驱动， 只需定义图节点的数据属性
  * 流程驱动， 只需定义图节点的依赖关系(兼容旧有的算子实现和图流程实现)
* 基于依赖注入原则管理数据依赖
* 基于函数编程思想控制数据的访问和移动

### iSpine框架的设计和构建

系统总体结构如下图所示:

![](https://km.woa.com/asset/c55fc1be479b4d82aca14cd502f9a8bc?height=1070\&width=2026\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

回到最开始的问题，实际上iSpine框架需要回答的是下面的问题： 算子问题：

* 算子如何定义？
* 算子集合如何编排？
* 编排后的算子集合如何执行？

数据问题：

* 数据的定义
* 数据如何在算子间高效传递

下面，我们将逐一给出答案：

#### 算子与数据

我们认为，一个算子是一个定义了输入/输出数据的计算单元； 其中，输入/输出数据可以是任意的数据结构对象；以一种具备语法元数据的语言为例（如java）， 一个算子可以如下定义（java），分别用不同的annotation标注不同的目标；

```java
//算子
@Operator(name="op1")
public class OperatorA{
    @Input
    private int a;     //输入
    @Input
    private String b;  //输入
    @Output
    private List<String> c;  //输出

    // 算子逻辑
    @OperatorFunc
    public void Execute(){
        //DO the logic
    }
}
```

其它常见的开发语言如python有decorator， C#有attribute， 较新的如rust也有有attribute，golang稍弱，不过也有`struct tags`这种辅助元信息机制； 我们希望算子开发者在声明算子定义时，就需要将算子的数据输入/输出显式地声明出来，最好按照一般地编程范式声明，而不需要引入额外地约束（以免算子实现与某种框架耦合过紧）；

那么， 在C++中我们如何达到类似效果呢？ c++中并没有类似的语法元数据，我们在ispine中，基于模板+宏机制，设计了一个算子定义的语法规则，使用体验上类似java的annotation，举例如下：

```cpp
#include "ispine/dag/dag.h"

// 算子声明
GRAPH_OP_BEGIN(test_op, "this is operator example with name:test_op")
// 输入
GRAPH_OP_INPUT(int, v0)
// 输出
GRAPH_OP_OUTPUT(std::string, v1)
// 输出
GRAPH_OP_OUTPUT((std::map<std::string, std::string>), v2)
int OnExecute(const didagle::Params& args) override {
  // Do the logic here
  if(nullptr == v0){
      // input v0 is null
  }
  // assign data to output
  v1 = "hello, world";
  v2["key"] = "value";
  // return 0 means ok, otherwise failed
  return 0;
}
GRAPH_OP_END
```

其中，input类型的使用上一般按照 **const 指针** 使用（也有特例，后续会提到），使用前一般需要判空（为空表明无此input数据）；output类型数据则按声明的类型对象使用；

实质上，以上的算子声明会展开类似以下代码， 也非常类似常规C++ class类型定义

```cpp
#include "ispine/dag/dag.h"

// 算子声明
struct test_op:public Operator{
    // 输入
    const int* v0;
    // 输出
    std::string v1;
    // 输出
    std::map<std::string, std::string> v2;
    int OnExecute(const didagle::Params& args) override {
      // Do the logic here
      if(nullptr == v0){
        // input v0 is null
       }
       // assign data to output
       v1 = "hello, world";
       v2["key"] = "value";
       // return 0 means ok, otherwise failed
       return 0;
     }
};
```

算子中还存在另外一种特殊数据，称之为**算子参数**； 一般指的是在abtest实验较多的服务场景下，算子一般被设计为接受某些运行参数， 同样的算子命中不同的实验流量，实际上选择的是不同的运行参数；通常的做法是在代码中用到参数的地方调用获取参数值的方式，例如：

```cpp
#include "ispine/dag/dag.h"

    int OnExecute(const didagle::Params& args) override {
      // Do the logic here
      if(args["key"].Int() == 101){
        // logic1
      }else if(args["key"].Int() == 102){
        // logic2
      }
       return 0;
     }
```

可以看到，这里本质上也是一种输入数据，只是被约定为固定的格式而已；按上述的用法， 也存在以下问题：

* 参数管理困难，参数名硬编码到code中，实质上不阅读整体代码是不知道当前算子用到了哪些参数；
* 参数类型安全，无法约束使用者使用正确参数类型，若类型错误很可能导致错误的参数数据，从而导致错误的逻辑；
* 无默认值机制

我们参考gflags的风格设计，同样基于宏+模板实现了类似用法的参数机制，例如：

```cpp
#include "ispine/dag/dag.h"

GRAPH_OP_BEGIN(test_op， "A test op")

GRAPH_PARAMS_bool(aarg, false, "bool arg");
GRAPH_PARAMS_int(iarg, 1212112, "int arg");
GRAPH_PARAMS_string(sarg, "abcde", "string arg");
GRAPH_PARAMS_double(darg, 3.124, "double arg");
GRAPH_PARAMS_int_vector(vec1, ({1, 2}), "int vector");
GRAPH_PARAMS_double_vector(vec2, ({1.1, 2.2}), "double vector");
GRAPH_PARAMS_string_vector(vec3, ({"hello", "world"}), "string vector");
GRAPH_PARAMS_bool_vector(vec4, ({true, false}), "bool vector");

int OnExecute(const didagle::Params& args) override {
  DIDAGLE_DEBUG("param aarg={}, iarg={}, sarg={}, darg={}", PARAMS_aarg, PARAMS_iarg, PARAMS_sarg, PARAMS_darg);
  DIDAGLE_DEBUG("vec1 size:{}, vec1[0]={}", PARAMS_vec1.size(), PARAMS_vec1[0]);
  DIDAGLE_DEBUG("vec2 size:{}, vec2[0]={}", PARAMS_vec2.size(), PARAMS_vec2[0]);
  DIDAGLE_DEBUG("vec3 size:{}, vec3[0]={}", PARAMS_vec3.size(), PARAMS_vec3[0]);
  DIDAGLE_DEBUG("vec4 size:{}, vec4[0]={}", PARAMS_vec4.size(), PARAMS_vec4[0]);
  return 0;
}
GRAPH_OP_END
```

#### 算子元数据

基于以上的基于模板+宏机制，我们可以在框架中运行时提取出所有算子的元数据信息（算子名，算子描述，算子输入，算子输出， 算子参数）；算子元数据一般用于以下用途：

* 算子管理（用于管理平台二次开发）
* 自动DAG图构建
* 数据传递

这里提到的算子的元数据信息通常可以导出为json格式的数据， 用于第三方二次开发；我们将该数据用在了laplace 算子管理平台上，用于算子的管理，dag的管理与发布等；以下是提取的一个算子元数据json样例：

```json
    {
        "name": "phase0",
        "desc": "this is test phase",
        "isIOProcessor": false,
        "input": [
            {
                "type": "std::string",
                "name": "model_id",
                "id": 98765,
                "flags": {
                    "is_extern": 0,
                    "is_aggregate": 0,
                    "is_in_out": 0
                }
            }
        ],
        "output": [
            {
                "type": "std::map<std::string, std::string>",
                "name": "result_map",
                "id": 98768,
                "flags": {
                    "is_extern": 0,
                    "is_aggregate": 0,
                    "is_in_out": 0
                }
            }
        ],
        "params": [
            {
                "name": "debug_enable",
                "type": "bool",
                "default_value": "false",
                "desc": "debug enable"
            }
        ]
    }
```

对应的代码为：

```cpp
GRAPH_OP_BEGIN(phase0, "this is test phase")
GRAPH_OP_INPUT(std::string, model_id)
GRAPH_OP_OUTPUT((std::map<std::string, std::string>), result_map)
GRAPH_PARAMS_bool(debug_enable, false, "debug enable");

int OnExecute(const didagle::Params& args) override {
  return 0;
}
GRAPH_OP_END
```

以上，算是回答**算子和数据的定义**问题；

#### 依赖注入

通常的编程范式下， 算子使用到的数据需要显示的在代码中主动获取，算子输出的数据也需要显示的在代码里调用； 由于这些过程都是硬编码在算子实现中，那么这将导致算子代码高度耦合并且难以维护和调试；[控制反转 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/%E6%8E%A7%E5%88%B6%E5%8F%8D%E8%BD%AC)

通常可以通过依赖注入来实现控制反转，从而解决上述问题；例如Java中的著名的Spring框架，其核心就是以一个IOC控制反转容器实现为基础，其中就是以依赖注入来实现控制反转； 一般如java等具备一定的动态反射能力的开发语言，实现依赖注入会相对容易一些；

在iSpine中，我们也尝试借助[依赖注入](https://en.wikipedia.org/wiki/Dependency_injection)思路来定义数据的传递模式，如下为iSpine框架中算子数据的传输路径：

注入依赖输入数据执行算子逻辑收集发布输出数据

我们参考spring的IOC框架，实现了一个简单的C++ DI(依赖注入)容器，用于控制所有算子的数据输入/输出：

* 任何算子在执行前，都会通过统一的DI容器注入输出数据（对算子的member 数据指针对象赋值）
* 任何算子在执行后，都会通过统一的DI容器注册保存输出数据（指针）

如此，所有的数据移动都是通过指针方式进行，例如算子A生成一个输出数据（OutputA），而算子B需要一个输入数据（InputB）正是算子A的输出， 那么只需要将输入数据（InputB）的指针指向输出数据（OutputA）即可；在这个例子中，实质上执行流程类似：

执行算子ADI容器收集A输出DI容器注入B依赖数据执行算子B

由于数据的移动都是指针方式的赋值，因此这里可以很高效地执行数据的移动；

#### 函数式编程

[函数式编程 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/%E5%87%BD%E6%95%B0%E5%BC%8F%E7%BC%96%E7%A8%8B)的特点和优势：

* immutable data 不可变数据，数据天然无竞争\

* 天然的可并发\

  * 并发的难点在于数据的竞争
  * 函数编程的数据都是无竞争的\

* lazy evaluation惰性求值\

* 不依赖于外部的数据，而且也不改变外部数据的值，而是返回一个新的值

借鉴函数式编程的思想， 我们约定以下的规则：

* 算子的输入数据一般是只读的指针（const pointer），指向实际的数据对象
* 算子的输入数据延迟赋值，只在执行前由DI容器赋值
* 算子的输出数据只在当前算子实现中赋值访问，在算子执行结束后，一般只允许只读的访问
* 算子按函数对象方式调度执行

基于以上的规则， iSpine中实现了一个相当高效的数据传输机制：

#### 高性能算子数据传递

基于前述设计原则， 函数式编程数据传递具备以下特点：

* 无锁（lock free）；
* 内存零拷贝，指针方式移动（immutable data 不可变数据）

实际场景中，immutable data 不可变数据只是理想态。例如召回场景中，召回一组数据后（通常为item数组），需要多个算子针对这个item数组执行一系列的写操作（打分，提权，过滤等），如果硬套函数式编程immutable data的规则，需要每个算子将整个item数组数据copy后再执行写操作，执行效率会相当低，因此我们在框架中增加另一条数据移动的规则：

* 数据只有一个使用者时，使用者可以move 原数据所有权到算子内部；（参考rust 的move 所有权概念）

move数据的实现关键技术点：

* move之后，该数据对其它算子不可见；即在运行中，只有一个算子/线程可以访问该数据；
* 算子可以重新发布move的数据到一个新的引用id（也可以隐藏不发布）；
* move只改变数据的引用id，不涉及数据拷贝；

由此，我们也可以实现在无锁的条件下对可写数据的高效移动；

我们在框架中加入了相关统计， 依赖注入耗时，以及数据发布耗时， 将其与算子运行耗时对比，以及trpc-cpp框架中fiber调度切换耗时比较，结果如下：

| 算子执行耗时（微秒） | 依赖注入（微秒） | 数据发布（微秒） | Fiber调度（微秒） |
| ---------- | -------- | -------- | ----------- |
| 10250      | 12.23    | 2.25     | 39.02       |

可以看到，依赖注入/数据发布运行效率极高，即便和已经非常快的fiber调度相比，也快了近一个数量级；

#### DAG定义

剩下的关键问题是如何编排算子集合，以及算子调度运行问题； 我们这里同样采用DAG用于以图形式描述由一组算子对象完成的计算任务，这里分别讨论顶点和边：

**顶点**

一个顶点存在两种可能情况：

* 算子
* 另一个图调用入口（用于支持图嵌套调用）

每个顶点执行结果只存在两种可能，成功或者失败； 每个顶点的出点有三种， 每个顶点通过其中一个出点与其它顶点相连，代表下一个顶点对于当前顶点执行结果依赖情况；

* 成功的情况下
* 失败的情况下
* 所有的情况下(成功/失败)

**边**

边为顶点间流向关系，一条边代表一个顶点执行完毕后将会开始执行另一侧的顶点； 这里存在两种描述边的途径：

**流程驱动**

通过顶点依赖关系直接描述边， 如 V(A) deps V(B) 或者 V(A) succeed V(B) 就可以描述A与B的边了； 目前支持6种描述方式：

* V(A) deps V(B) ， 任何情况下，执行完V(B)都会执行V(A)；
* V(A) deps\_on\_ok V(B) ， 执行完V(B)成功的情况下会执行V(A)；
* V(A) deps\_on\_err V(B)， 执行完V(B)失败的情况下会执行V(A)；
* V(A) succeed V(B)， 执行完V(A)，任何情况下都会执行V(B)；
* V(A) succeed\_on\_ok V(B)， 执行完V(A)，成功情况下都会执行V(B)；
* V(A) succeed\_on\_err V(B)， 执行完V(A)，失败情况下都会执行V(B)；

当算子存在条件执行的情况下，如`if(cond) -> A, else -> B`; 这里会将条件判断当作一种特殊算子，条件算子执行成功等同条件满足，相反就是条件不满足；

* V(cond) succeed\_on\_ok V(A)
* V(cond) succeed\_on\_err V(B)

基于以上的规则，可以比较方便的构建一个完备的流程图DAG；基本上，这里的边描述规则和常规图数据描述SPO三元组类似，比较容易理解；

**数据驱动**

流程驱动的描述方式优势是简单直接，比较直观； 缺点是完全依赖人为定义顶点执行关系，可能造成并发度不够，如某个顶点可能可以提前执行，但被人为定义在后续某个阶段；在一个比较大的有多个顶点的执行图中，这个缺点会比较明显，依赖人工调整优化流程驱动DAG成本较高； 这里引入一种基于数据驱动的图描述方式，原则上只需要定义顶点与顶点的输入、输出数据， 执行引擎会自动的基于数据的依赖推导出顶点依赖，继而构建一个完整的图；理论上构建出来的图的并行度是最高的。 数据依赖的描述规则如下：

* 顶点定义输入input
* 顶点定义输出output
* 输入输出是同一个数据定义，规则：
  * field， 代表在前述算子定义中的变量名
  * id， 默认为空，等同field； 若存在多个算子定义同名的输出变量时，需要配置id以消除同名
  * extern，作用与input输入， 默认false， 为true意味着该数据为外部非图中其它算子设置
  * required，作用与input输入， 默认false， 为true意味着该数据为必须依赖

边的推导关系过程如下：

* 迭代每个顶点(A)的输入数据，找到该数据署于哪个顶点(B)的输出;
  * 若该数据required为true， 则 V(A) deps\_on\_ok V(B)
  * 否则 V(A) deps V(B)

当算子存在条件执行的情况下，如`if(cond) -> A, else -> B`; 数据驱动的描述会将if/else声明成两个命名的bool数据值；如：

* 图中定义with\_cond, without\_cond两个变量， 分别在满足条件下才会赋值；
* V(A)中定义一个required的input为with\_cond
* V(B)中定义一个required的input为without\_cond

#### DAG DSL

基于以上的设计规则，didagle中定义了一种由toml配置描述的DSL，目的在于通过配置，自动推导整个DAG执行图；其格式说明如下：

**GraphCluster**

一个toml文件为一组图集合，图集合的名字为toml文件名， 格式如下：

```javascript
strict_dsl = true                      # 是否严格校验（判断算子processor是否存在）
default_expr_processor = "ispine_didagle_expr"  # 默认表达式算子
[[config_setting]]                     # 全局bool变量设置，由`default_expr_processor`执行
name = "with_exp_1000"                 # 全局bool变量名
cond = 'expid==1000'                   # bool表达式
[[config_setting]]
name = "with_exp_1001"
cond = 'expid==1001'

[[graph]]                              # DAG图定义  
...
```

**Graph**

一个图为一个完整的执行任务，通常由一组顶点配置组成，格式如下：

```javascript
[[graph]]                               # DAG图定义  
name = "graph_name"                     # DAG图名  
[[graph.vertex]]                        # 顶点  
...
[[graph.vertex]]
...
[[graph.vertex]]
...
```

**Vertex**

顶点配置一般配置顶点的各种属性，示例如下：

```javascript
[[graph]]                               # DAG图定义  
name = "graph_name"                     # DAG图名  
[[graph.vertex]]                        # 顶点  
processor = "phase1"
input = [{ field = "f1", id = "d1", required = true }]
output = [{ field = "ff1" }]
[[graph.vertex]]
...
[[graph.vertex]]
...
```

顶点属性配置则支持两种模式：

* 流程驱动
* 数据驱动

这里分别描述流程驱动和数据驱动描述：

**流程驱动**

流程驱动显式地配置顶点间地依赖，支持如下的显式属性关系设置：

* deps 当前的顶点的前驱顶点（任何情况）
* deps\_on\_ok 当前的顶点的前驱顶点（执行成功的情况下）
* deps\_on\_err 当前的顶点的前驱顶点（执行失败的情况下）
* successor 当前的顶点的后继顶点（任何情况）
* successor\_on\_ok 当前的顶点的后继顶点（执行成功的情况下）
* successor\_on\_err 当前的顶点的后继顶点（执行失败的情况下）
* if 当前的条件顶点的后继顶点（true的情况下）
* else 当前的条件顶点的后继顶点（false的情况下）

以下为示例配置：

```javascript
[[graph]]
name = "sub_graph2"                     # DAG图名  
[[graph.vertex]]                        # 顶点  
processor = "phase0"                    # 顶点算子，与子图定义/条件算子三选一
#id = "phase0"                          # 算子id，大多数情况无需设置，存在歧义时需要设置; 这里默认id等于processor名
successor = ["test_34old"]              # 顶点后继顶点
args = {x=1,y="2",z=1.2}                # 顶点算子参数
[[graph.vertex]]
id = "test_34old"                       # 算子id，大多数情况无需设置，存在歧义时需要设置 
cond = 'user_type=="34old"'             # 条件算子表达式
if = ["subgraph_invoke"]                # 条件算子成功后继顶点
else = ["phase2"]                       # 条件算子失败后继顶点
[[graph.vertex]]
id = "subgraph_invoke"                  # 子图调用算子id
cluster = "eample1.toml"                # 子图集合名
graph = "sub_graph2"                    # 子图名
[[graph.vertex]]
processor = "phase2"
select_args = [
    { match = "with_exp_1000", args = { abc = "hello1", xyz = "aaa" } },    # 条件变量， 全局变量with_exp_1000为true时生效，顺序判断选择
    { match = "with_exp_1001", args = { abc = "hello2", xyz = "bbb" } },
    { match = "with_exp_1002", args = { abc = "hello3", xyz = "ccc" } },
]
args = { abc = "default", xyz = "default" }                                 # 默认变量， 条件变量选择失败时选择  
[[graph.vertex]]
processor = "phase3"
deps = ["subgraph_invoke", "phase2"]    # 算子依赖的算子集合             
```

以上流程驱动样例构建的可视图 ： ![](https://km.woa.com/asset/1a8d2bb9ce714109895cf989ee434d77?height=141\&width=1145\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

**数据驱动**

数据驱动 则继续有两种情况：

* 隐式推导
* 显式配置

隐式推导基于算子成员变量的推导，如\[算子与数据]\[###算子与数据]; 框架会基于算子定义的输入/输出，自动在同名的输入、输出之前建立关联；如下例，若图中算子的成员数据名能自动关联无冲突，则配置无需定义input/output；

```javascript
[[graph]]
# 若processor的成员数据名能自动关联无冲突，则配置无需定义input/output, 能自动关联
name = "auto_graph"
[[graph.vertex]]
processor = "phase0"
input = [{ field = "v0", extern = true }]   # 外部设置的变量需要显示标明extern，否则图解析时会判断依赖input不存在
args = { abc = "default", xyz = "zzz" }
[[graph.vertex]]
processor = "phase1"
[[graph.vertex]]
processor = "phase2"
[[graph.vertex]]
processor = "phase3"
```

其对应的生成流程图： ![](https://km.woa.com/asset/c063f05202154694bb39025304b10769?height=303\&width=1163\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

显式的基于数据声明的推导：

```javascript
[[graph]]
name = "sub_graph0"
[[graph.vertex]]
processor = "phase0"
input = [{ field = "v0", extern = true }]    # 算子extern输入
output = [{ field = "v1", id = "v1" }, { field = "v2" }]  # 算子两个输出
select_args = [
    { match = "with_exp_1000", args = { abc = "hello1", xyz = "aaa" } },
]
args = { abc = "default", xyz = "zzz" }
[[graph.vertex]]
processor = "phase1"
input = [{ field = "v2" }]
output = [{ field = "v3" }, { field = "v4" }]
[[graph.vertex]]
processor = "phase2"
input = [{ field = "v1" }]
output = [{ field = "v5" }, { field = "v6" }]
[[graph.vertex]]
processor = "phase3"
id = "phase3_0"
# 显示声明input数据
input = [
    { field = "v1" },
    { field = "v2" },
    { field = "v3" },
    { field = "v4" },
    { field = "v5" },
    { field = "v6" },
]
output = [{ field = "v100", id = "m0" }]
[[graph.vertex]]
processor = "phase3"
id = "phase3_1"
# input = ...  , 无需设置，隐式可以推导出和phase3_0配置一样效果
output = [{ field = "v100", id = "m1" }]     # 将field输出到数据m1
[[graph.vertex]]
#expect = 'user_type=="34old"'               # 当条件'user_type=="34old"'满足时该算子才会运行
expect_config = "with_exp_1002"              # 当全局bool变量with_exp_1002为true时，该算子才会运行
processor = "phase3"
id = "phase3_2"
# input = ...  , 无需设置，隐式可以推导出和phase3_0配置一样效果
output = [{ field = "v100", id = "m2" }]     # 将field输出到数据m2
[[graph.vertex]]
processor = "phase4"
input = [{ field = "v100", aggregate = ["m0", "m1", "m2"] }]   # 输入为m0,m1,m2的聚合， 用途将多个重复类型的不同数据聚合在一起
```

以上样例构建的可视图

![](https://km.woa.com/asset/a8a12cb89a584bbaadbb0574859542ef?height=368\&width=1159\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

DAG执行引擎工作时时自动将隐式、显式两种机制结合在一起工作，显式的配置会覆盖并merge隐式的推导；

#### DAG执行引擎

DAG执行引擎实现包含两部分：

* 图定义脚本的动态解析（热加载）
* 图脚本的执行（算子的调度）

**DAG动态解析**

通常，DAG脚本定义在一个远程的DAG管理发布平台上管理；iSpine框架会自动从平台上同步配置；

由于图解析是一个非常耗时的操作，因此这里的实现是异步parse 解析图实例，并预先创建一个图对象池用于后续运行；整个过程是一个类似热加载实现， 在DAG管理发布平台上的变更会实时体现到运行中；

**DAG执行**

图算子的调度执行规则遵循两组规则， 按图的定义方向流动执行：

* 顶点
  * 每个顶点有初始化依赖计数， 初始化依赖计数为0的为起始顶点，可以多个
  * 当顶点的的依赖计数为0时，执行引擎可以启动执行该顶点；
  * 顶点执行完毕后，需要将顶点的后继依赖计数减去1
  * 顶点执行完毕后，同时将图的join计数减去1
* 图
  * 每个图初始化join计数，数目为整个图的定点数
  * 当图的join计数为0， 整个图执行完毕，通知调用者的done closure

基于以上的规则，可以看到整个图的调度执行是无阻塞全异步的（非算子执行），且能达到最大并发，充分利用了计算资源；

本质上，由于所有的算子的执行都是异步的，我们还需要实现一个类似goroutine的调度器实现一个多线程调度实现（部分框架就是这么实现的，例如开源的[taskflow](https://github.com/taskflow/taskflow) ）;

对此我们在技术上的考虑是：

* 我们需要引入fiber，基于fiber编程有更多的优势（同步编程的开发体验远优于异步编程开发）
* fiber已经有一个fiber的调度器实现；
* 两者功能极其相似，再开发一遍意义不大；即便遇到问题，也可以在fiber调度器上进行优化；

所以我们决定整个DAG的执行引擎约定在trpc-cpp fiber上实现；

同时，在设计实现上我们也采用以下技术优化整个DAG执行引擎的性能：

* 图结构分为immutable部分和 mutable的执行部分
  * mutable的执行部分只用保存顶点/图运行的计数器，占用资源很小，因此可以每次请求是快速创建
  * immutable部分表示整个图顶点的依赖关系等，基本不变，在所有图实例中复用；
  * 同时采用对象池进一步减少图mutable对象创建的开销；
* DAG调度和算子执行线程融合
  * 无独立的调度线程实现
  * 每个算子执行后，就地执行下一步调度实现，而非通知DAG调度（避免线程/fiber通知以及切换的开销）
  * 由于串行的算子较多，实际上这样的实现减少了线程/fiber切换开销，执行效率更高
* 条件分支动态裁剪
  * 如果图结构中存在条件节点，会根据条件节点的动态结果裁剪后续图节点的运行。
  * 如果一个图节点的执行条件为否，后续单独依赖它的节点都不会运行，条件节点具备传递性。如果后续节点不单独依赖不运行的节点，则当前节点可运行。
* 超时控制
  * 整个图可以设置一个超时时间，每个算子执行前判断是否超时；若超时，后续的执行都可以跳过
  * 可以避免在整体超时情况，仍然进行冗余无用的计算
* 基于TRPC-CPP的Fiber优化IO/计算算子
  * 在M：N的Fiber运行模式下，不同的算子代码能够轻量级地分配到多个核上充分利用机器资源，同时遇到阻塞逻辑，也能够通过运行时主动挂起任务，将线程资源让出给队列中的任务
  * 不用单独再实现一个重复能力的调度器\
    \
    针对DAG执行引擎的开销，我们也在实际服务`wezone_mv_irs`中统计了执行引擎的开销，如下表格所示，从中可以看出，目前执行引擎的开销占比算子的执行0.5%；

| 算子执行耗时（微秒） | DAG依赖注入（微秒） | DAG数据发布（微秒） | DAG Fiber调度（微秒） | 占比   |
| ---------- | ----------- | ----------- | --------------- | ---- |
| 10250      | 12.23       | 2.25        | 39.02           | 0.5% |

#### DAG与ABTest实验

通常，算法类在线服务需要做大量的ABTest实验，在一个较大的在线服务系统上，同时在线生效的实验可能有成百上千之多； 例如小世界推荐（尚在起步发展期），目前（2021.12.10）已有101个在线实验； ABTest实验通常由两种行为：

* 调参； 不同的实验执行的算子逻辑一样，只是运行参数不一致，如召回条数等
* 不同的分支逻辑； 不同的实验执行的算子逻辑有区分，比如新上实验增加一路召回算子调用

我们在DAG执行引擎中引入表达式能力，结合DAG DSL的分支运行能力，支持如下的ABTest场景：

**顶点条件运行参数**

引入顶点条件运行参数能力，运行期基于动态条件选择运行参数， 示例如下：

```javascript
[[graph]]
name = "sub_graph2" # DAG图名  
[[graph.vertex]]
processor = "phase2"
select_args = [
    { match = "with_exp_1000", args = { abc = "hello1", xyz = "aaa" } },
    { match = "with_exp_1001", args = { abc = "hello2", xyz = "bbb" } },
    { match = "with_exp_1002", args = { abc = "hello3", xyz = "ccc" } },
] # 条件变量， 全局变量with_exp_1000为true时生效，顺序判断选择
args = { abc = "default", xyz = "default" } # 默认变量， 条件变量选择失败时选择  
```

**顶点运行条件**

引入顶点运行条件， 顶点仅在满足条件情况下运行，示例如下:

```javascript
[[graph]]
name = "sub_graph2" # DAG图名  
[[graph.vertex]]
expect = '$user_type=="34old"'               # 当条件'user_type=="34old"'满足时该算子才会运行
#expect_config = "with_exp_1002"              # 当全局bool变量with_exp_1002为true时，该算子才会运行
processor = "phase3"
id = "phase3_2"
```

**多层实验**

通常，在流量不足的情况下，一个具体的在线服务会划分多层实验；整个推荐系统一般又是多个具备实验能力的子系统组成，这里就会遇到这样的问题：

* 单服务多层实验状态下， DAG图如何设置？
* 全流程下，实验参数如何传递？尤其是需要全流程贯穿的参数，如正排的实验参数等

由于这里通常和具体RPC协议关联（而框架是和协议无关的实现），我们没有将这一部分实现在框架中，而是将依赖的基础能力抽象出来集成在框架中；在QQ小世界推荐，我们基于iSpine的框架能力（DAG子图，参数管理，参数表达式）构建了一整套DAG分层实验实现， 这里限于篇幅，不再展开详述，后续会单独分享；

#### 插件与词典

在线服务中，算子的实现并不是仅仅依赖其它算子的数据/rpc请求，一般还需要有其它组件能力的支持才能实现比较完整的功能； 通常有两种能力：

* 公共组件(插件)的定义：一个独立模块功能， 如redis client， 向量检索， 倒排检索等;
* 词典索引：服务进程加载的自定义格式的内存索引

我们在iSpine中也实现了这两类组件，不过限于篇幅，这里不再展开详述，后续会单独分享；

插件与词典也是通过依赖注入方式应用在算子实现中；

#### 算子Debug调试

**算子单元测试**

由于算子的依赖都可以通过依赖注入设置，实质上可以很方便的对一个算子编写单元测试用例；例如：

```cpp
TEST(ProcessorUT, Porcessor) {
  GraphDataContext ctx;
  int tmp = 101;
  ctx.Set("v0", &tmp);  // 设置依赖
  ProcessorRunOptions opts;
  //执行算子 test_phase
  ProcessorRunResult result = run_processor(ctx, "test_phase", opts);
  EXPECT_EQ(result.rc, 0);

    //校验算子输出
  EXPECT_EQ("val101", *(ctx.Get<std::string>("v1")));
  const auto* map =  ctx.Get<std::map<std::string, std::string>>("v2");
  //校验算子输出
  EXPECT_EQ("val1", map->at("key1")); 
  EXPECT_EQ("val2", map->at("key2"));
}
```

**算子统计与监控**

在DAG执行引擎运行过程中，框架运行时会捕捉算子运行异常， 耗时等信息，通过trpc-cpp内置的监控统计能力上报到统一的监控告警平台，用户可以在监控告警平台上配置对应的告警设置；

#### DAG管理发布平台

配合DAG的管理发布，我们也单独实现了一个简单的DAG管理发布平台， 具备如下功能：

* 算子元数据的管理；
* DAG图脚本的编辑
* DAG图脚本的可视化
* 指定容器/set等多种发布级别

当然，整个管理平台的实现还比较粗糙，还有如下的不足：

* 算子管理
  * 无可视化编辑能力
  * 无搜索能力
* DAG图脚本管理
  * 无可视化编辑能力，仅有可视化呈现编辑脚本能力
  * 多层子图展示上不友好（需要自行跳转）

#### 研发流程优化

在基于iSpine框架的研发流程中，我们解耦了算法、工程，实现了算法与工程迭代的各自闭环，提升了研发效率，算法迭代上线周期大幅缩短；

整个推荐搜索研发迭代流程：

* 模型迭代、特征变更及算法策略迭代\

  * 算法工程师可以自主完成全链路的开发测试，无需工程研发人员和测试工程师的介入；大多数场景只涉及现有算子、DAG的实现调整；
  * DAG脚本独立托管在远程DAG管理发布平台，无需任何服务上线，DAG脚本上线后周知到工程侧及产品方关注相关指标变化即可\

* 新业务场景和新算法策略接入\

  * 需要算法和工程共同开发
  * 工程定义好插件、词典等算子依赖公用组件，通过protobuf或者接口形式暴露给算子开发者
  * 算子开发者基于公用组件以及现有算子开发新的算子实现
