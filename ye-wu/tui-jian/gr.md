# GR

## 1. 背景 <a href="#id-1.-bian-xie-mu-di" id="id-1.-bian-xie-mu-di"></a>

2024年2月，Meta提出了GR模型\[1]，颠覆了当前主流的推荐模型的设计思路，论文实验中也取得了不错的业务效果。当前业务正在进行尝试，由于其中HSTU结构计算的复杂性，当前遇到了训练和推理性能上的问题，导致

1. 实验迭代慢
2. 线上推理成本高

我们需要设计一个支持业务快速迭代的GR模型训练+推理流水线，实现

* 模型结构自由定制能力，方便业务改造网络
* 训练上，降低训练成本，支持HSTU向B级别的参数规模扩展
* 推理上，解决GR类模型推理复杂度O(序列长度\*item数量)过高的问题，实现O(序列长度+item数量)的计算复杂度
* 由于GR模型效果的不确定性，方案应尽量复用已有系统能力

方案的大致思路：

* API层面：规范GR模型的算法编程框架为：prefixDNN + HSTU + postfixDNN，三个模块顺序执行的算法编写框架。
* 训练框架复用Numerous框架现有能力，对HSTU模块进行专用优化，实现多模块独立导出模型的能力
* 推理框架复用Numerous serving现有能力，实现多模块独立加载和推理的能力，其中HSTU模块采用yinian LLM进行推理，利用LLM推理中的prefix caching技术来实现整体计算量数量级的下降。

本文档通过分析GR模型的特点，训练和推理性能问题的根源，结合当前系统的现状，设计适应GR模型的模型管线。

## 2. 术语定义 <a href="#id-2.-shu-yu-ding-yi" id="id-2.-shu-yu-ding-yi"></a>

定义本文档中用到的术语。包括两个部分：

### 2.1 已有术语 <a href="#id-2.1-yi-you-shu-yu" id="id-2.1-yi-you-shu-yu"></a>

* GR模型：Generative Recommendation，生成式推荐模型。
* HSTU：Hierarchical Sequential Transduction Units，一种类transformer结构的模型单元。
* SavedModel格式（简称：TF格式）：一种tensorflow定义的模型文件格式
* TensorRT格式：Nvidia的GPU推理加速组件TensorRT定义的模型文件格式
* Onnx格式：一种开源的模型文件格式
* Numerous Sparse格式：Numerous 自定义的模型文件格式，主要解决sparse参数分片存储和全量/增量上线的问题

### 2.2 新定义术语 <a href="#id-2.2-xin-ding-yi-shu-yu" id="id-2.2-xin-ding-yi-shu-yu"></a>

无

## 3. 参考资料 <a href="#id-3.-can-kao-zi-liao" id="id-3.-can-kao-zi-liao"></a>

\[1] Actions Speak Louder than Words: Trillion-Parameter Sequential Transducers for Generative Recommendations，[https://arxiv.org/pdf/2402.17152.pdf，2024.2](https://arxiv.org/pdf/2402.17152.pdf%EF%BC%8C2024.2)

## 4. 问题 <a href="#id-4.-wen-ti" id="id-4.-wen-ti"></a>

本章详细描述需要解决的问题。

### 4.1 问题场景描述 <a href="#id-4.1-wen-ti-chang-jing-miao-shu" id="id-4.1-wen-ti-chang-jing-miao-shu"></a>

#### 4.1.1 GR模型的介绍 <a href="#id-4.1.1gr-mo-xing-de-jie-shao" id="id-4.1.1gr-mo-xing-de-jie-shao"></a>

GR模型中，大致有三个特点：\
**特点1**：核心的HSTU结构是一种类transformer结构，定制了attention结构.

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=22155784)

**特点2**：在结构相对固定的HSTU单元之外，还存在两个与业务场景相关的前缀模块和后缀模块，为便于描述，我们称之为prefixDNN和postfixDNN。prefixDNN类似多模态大语言模型中的encoder模块。postfixDNN类似推荐模型中常见的MoE模块。这两个模块的特点是模型结构与业务场景相关，结构差异大。

**特点3**：模型建模方法上是通过长序列来实现模型的下一个用户行为的预测。相比传统的推荐模型结构，计算量大100-1000倍。

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=22155876)

#### 4.1.2 无量系统现状 <a href="#id-4.1.2-wu-liang-xi-tong-xian-zhuang" id="id-4.1.2-wu-liang-xi-tong-xian-zhuang"></a>

当前在无量系统下使用GPU进行模型训练和推理的流程如下：

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=22155933)

模型主要分为两大块：DNN和Sparse。由于两个模块不同的特性，模型导出使用了不同的格式存储。DNN部分导出的Dense模型文件使用原生TF格式，而Sparse部分使用无量自定义的模型格式，以便支持全量/增量/实时上线等功能。

在模型线上推理过程中，默认使用TensorRT作为GPU上的推理引擎。由于推荐模型的算子复杂，可能导致计算图复杂，或者TensorRT不支持某些算子的问题，DNN模型部分会经过模型转换组件，进行计算图切分，得到适合GPU执行的部分，保存为TensorRT格式的文件，和适合CPU执行的部分，保存成TF格式的文件。

#### 4.1.3 当前系统遇到的问题 <a href="#id-4.1.3-dang-qian-xi-tong-yu-dao-de-wen-ti" id="id-4.1.3-dang-qian-xi-tong-yu-dao-de-wen-ti"></a>

**问题1**：HSTU前向计算和反向计算耗时长。

**问题2**：GR的长序列模型推理过程中，精排会对每个用户预测几百上千个item，计算量为O(序列长度\*item数量)。结合问题1，导致线上推理资源量成数量级增长。单纯的降低HSTU模块的计算耗时难以有数量级上的收益。

**问题3**：为了降低LLM模型推理的成本，业界有许多开源框架vLLM/TensorRT-LLM，包括我们自研的yinian LLM，都是自成体系的推理框架，与推荐类模型的推理框架存在融合困难。比如：由于LLM生成的耗时和吞吐容忍度高（每个token生成耗时普遍超过10ms，甚至100ms都是可以接受的；吞吐则普遍在个位数），vLLM和TensorRT-LLM都是以pytorch驱动的框架。虽然yinian LLM采用纯C++开发，但是调度和显存管理主要是为LLM推理场景设计。目前还不能适应推荐场景严苛的吞吐和耗时要求。

### 4.2 相关场景描述 <a href="#id-4.2-xiang-guan-chang-jing-miao-shu" id="id-4.2-xiang-guan-chang-jing-miao-shu"></a>

**问题4**：硬件供应与软件生态问题：当前条件下，英伟达高端GPU卡只存在少量存货，在可预见的一两年内，不会有新的高端卡供应。相对高端的加速卡目前只有华为的NPU，华为NPU的软件生态还处逐步完善过程中。当前Numerous使用的Tensorflow框架在NPU上问题还很多。

### 4.3 问题定义 <a href="#id-4.3-wen-ti-ding-yi" id="id-4.3-wen-ti-ding-yi"></a>

相对规则化地定义问题

#### 4.3.1 目标（Objective） <a href="#id-4.3.1-mu-biao-objective" id="id-4.3.1-mu-biao-objective"></a>

从前面的问题介绍，我们可以总结下面两个首要优化目标：

* 目标1：优化训练性能，达到与原有推荐模型训练相当的追模型速度。
* 目标2：降低计算资源消耗，实现从O(序列长度\*item数量)到O(序列长度+item数量)的下降

另外，还有两个次要目标：

* 目标3: 由于推荐系统的全流程是通过了多年打磨的成熟系统，新的设计不能对现有系统造成太大的冲击。
* 目标4: TensorFlow逐步被Pytorch生态甩开，LLM相关的技术都是在pytorch上首发，方案需要避免对tensorflow的强依赖。

#### 4.3.2 限制条件 <a href="#id-4.3.2-xian-zhi-tiao-jian" id="id-4.3.2-xian-zhi-tiao-jian"></a>

设计上我们需要考虑一下的约束条件

* 硬件条件限制。
  * 约束1: 在高端卡供应问题不足的现状下，系统需要降低对国外高端卡的依赖。
* 软件系统限制：
  * 约束2: 模型描述能力上需尽量保留推荐系统原有的定义能力。HSTU可以相对固定，使用配置化改变。prefixDNN和postfixDNN需要能够灵活修改

## 5. 方案 <a href="#id-5.-fang-an" id="id-5.-fang-an"></a>

### 5.1 业界方案分析 <a href="#id-5.1-ye-jie-fang-an-fen-xi" id="id-5.1-ye-jie-fang-an-fen-xi"></a>

暂无业界方案可参考。Meta论文中对系统方案也无描述

### 5.2 总体方案 <a href="#id-5.2-zong-ti-fang-an" id="id-5.2-zong-ti-fang-an"></a>

总体方案的思路是

* 模型定义上独立出专门的HSTU模块，进行专项的算子和图融合优化，以提高计算性能。同时在推理端支持类似LLM的kv-cache优化能力。
* 在训练上支持多DNN模块导出的能力，在推理上实现多种推理引擎协同工作的能力，实现传统图计算推理引擎和LLM推理引擎协同工作。

总体流程如下图所示：

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=22163864)

在这个流程下，PrefixDNN，HSTU和PostfixDNN三个模型固定以串行的方式链接起来。然后分别产出模型文件。PostfixDNN和PostfixDNN的模型文件根据需要，选择性调用模型转换模块进行优化。由于yinian LLM会专门优化HSTU单元，模型文件直接交给推理服务加载。

#### 5.2.1 算法编程 <a href="#id-5.2.1-suan-fa-bian-cheng" id="id-5.2.1-suan-fa-bian-cheng"></a>

**5.2.1.1 模型结构说明**

如前所述，为了实现推理性能的优化，需要把HSTU部分单独出来，使用优化的方式进行推理。因此，在定义模型结构的时候，需要明确的结构去支持后续的处理。模型结构入下图所示，具体来说：

1.把模型的定义划分为几个固定的部分：

1） split\_input拆分输入的特征

2）PrefixDNN准备HSTU部分的输入

3）HSTU部分实现HSTU模型

4）PostfixDNN实现后置的处理逻辑

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=22562286)

&#x20;

2\. 结合双塔模型，举例说明如何把各个部分拆分到新的模型结构里面，如果下图所示

![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=22562782\&width=0)

&#x20;

&#x20;

**5.2.1.2 模型代码说明**

HSTU单元以定制模块方式提供，业务算法人员不能自由实现，只能通过类似LLM模型配置的方式来修改，接口如下：

代码解释代码改写

```
# HSTU类不可以被修改，由框架提供具体的事项，业务只能用HSTUConfig来配置
class HSTU(tf.keras.Model):
  def __init__(self, HSTUConfig config):
    ...
  # hstu_input是个dict，需要输入token_embedding, seq_lengths, timestamp,is_training
  def call(self, hstu_input_dict):
    ...
    
    return output_tensor
```

业务算法开发人员编程上，流程为：

代码解释代码改写

```
class PrefixDNN(tf.keras.Model):
  def call(self, token_features_dict, training):
    # 用户定义模型结构
    # 这个模型结构中不能进行任何综合sequence信息的操作，比如attention。
    # 否则会导致PrefixCaching性能优化失效。
    # 这个模型输入hstu需要的相关特征，输出encoding之后的embedding
    hstu_input = encoder(token_features_dict)
    return hstu_input
    
class PostfixDNN(tf.keras.Model):
  def call(self, hstu_output_tensor, common_features_dict, training):
    # 用户定义模型结构
    # hstu_output是hstu模型的输出，用户可以使用这个输出做后续需要的逻辑
    ...
    return outputs #此处可以定义任意输出用于计算loss等
class EmbeddingMlpNet(numerous.Model):
  def __init__(self):
    # 用户自定义HSTU配置，编写PrefixDNN和PostfixDNN的初始化逻辑
    hstu_config = HSTUConfig
    self.prefix_dnn = PrefixDNN    
    self.postfix_dnn = PostfixDNN
    ...
    # 这部分为固定逻辑，算法人员不能修改
    self.hstu = HSTU(hstu_config)
    self._dump_slices["PrefixDNN"] = self.prefix_dnn # 固定slice名称，便于后期流程处理
    self._dump_slices["HSTU"] = self.hstu # 固定slice名称，便于后期流程处理
    self._dump_slices["PostfixDNN"] = self.postfix_dnn # 固定slice名称，便于后期流程处理
  def split_input(self, inputs):
    # 用户自定义：
    # 将样本数据的特征拆分位用于hstu输入(token_features_dict)和postfix_dnn(common_features_dict)输入两个部分,
    return token_features_dict, common_features_dict
  
  # 本函数为固定逻辑，用户不可以修改
  def call(self, inputs, training):
      token_features_dict, common_features_dict = self.split_input(inputs)
      hstu_input_dict = self.prefix_dnn(token_features_dict)
      hstu_output_tensor = self.hstu(hstu_input_dict)
      pred_tensors = self.postfix_dnn(hstu_output_tensor, common_features_dict)
      return pred_tensors43​44
  
  def compute_loss(self, label_tensors, pred_tensors, training):
      # 用户定义loss计算逻辑
      return loss
  
  ...
```

&#x20;

#### 5.2.2 模型存储格式 <a href="#id-5.2.2-mo-xing-cun-chu-ge-shi" id="id-5.2.2-mo-xing-cun-chu-ge-shi"></a>

以下是模型实例在存储时以不同slice分开各个单元

PrefixDNN和PostfixDNN两个slice与原有的Dense模型文件描述相同。不同部分在default和HSTU两个slice。

* default slice中有一个slice\_graph.ini文件，用于描述不同slice之间的关联关系，用于serving加载时，按照规则串接不同的slice
* HSTU slice中有一个config.ini，是一个类似huggingface上大语言模型的config.ini文件。用于yinian按照固定的名字映射关系加载HSTU模块。

#### 5.2.3 训练 <a href="#id-5.2.3-xun-lian" id="id-5.2.3-xun-lian"></a>

基础功能：

* 多模块训练能力。无量框架需要在当前训练能力的基础是上，支持“5.2.1 算法编程”中描述的接口。在GR模型全流程上，模型以“5.2.2 模型存储格式”中描述的方式存储。

性能优化：

* HSTU性能优化。HSTU结构在“5.2.1 算法编程”接口上被固定，只能通过HSTUConfig进行配置，就可以采用深度的算子融合技术进行优化。与LLM模型的训练和推理优化路径相同，业界会逐步出现很多深度优化的融合算子，这个部分的优化将充分吸收借鉴业界的优秀方案，叠加系统自身特性相关进行优化的方式。
* 样本处理优化。在当前无量的样本格式，对于单个item的特征基本采用一对一展开的方式写入样本。在长序列的情况下，可能因为item重复，tag重复等问题导致样本量大，unique处理成本高的问题。如果能够提前在样本生成时完成部分unique操作，则会对样本传输和样本处理两个环节产生巨大的收益。

### 5.2.4 推理 <a href="#id-5.2.4-tui-li" id="id-5.2.4-tui-li"></a>

#### 5.2.4.1 概述 <a href="#id-5.2.4.1-gai-shu" id="id-5.2.4.1-gai-shu"></a>

基础功能：

* 多模块推理能力。无量serving通过加载“5.2.2 模型存储格式”的模型文件，构建起PrefixDNN+HSTU+PostfixDNN三个模块串行执行逻辑。



在原有的Numerous serving服务结构中，已有tensorflow，TensorRT和Sparse三种引擎的执行和模型加载能力，通过增加yinian LLM，可以实现GR模型所需的多引擎执行的能力。同“5.2.2 模型存储格式”中描述的slice关联关系，可以确定不同引擎的执行顺序和数据交换规则。

性能优化：

* HSTU性能优化。与训练相似的优化逻辑，在yinian LLM引入HSTU的融合算子，加速HSTU模块的执行。与训练不同是GR模型执行过程中会因为prefix caching加速技术，需要支持paged attention的加速技术。
*   PrefixCacheing加速。在GR模型的推理中，存在巨大的prefix caching优化的空间。如下图所示：\


    ![](https://iwiki.woa.com/tencent/api/attachments/s3/url?attachmentid=22156053)\
    推荐类模型在线上推理过程中，需要对一个用户进行大量的item进行预测。单个item在执行GR的推理时，会输入很长的历史序列和一个候选item，来预测当前item的点击等信息。用户的一次交互推荐中，历史序列是相同的，只有item信息是变化的，所以可以充分利用decoder类模型推理的prefix caching机制来减少计算量。将计算量从O(prefix\_token\_num\*item\_num)变为O(prefix\_token\_num+item\_num)。由于历史序列非常长，这个收益将是巨大的。

#### &#x20;<a href="#id-5.2.4.2-ye-wu-he-serving-de-jiao-hu-liu-cheng" id="id-5.2.4.2-ye-wu-he-serving-de-jiao-hu-liu-cheng"></a>

#### &#x20;<a href="#id-5.2.4.4-fei-qi-bu-fen" id="id-5.2.4.4-fei-qi-bu-fen"></a>

### 5.2.5 针对NPU的设计 <a href="#id-5.2.5-zhen-dui-npu-de-she-ji" id="id-5.2.5-zhen-dui-npu-de-she-ji"></a>

目前华为对于TensorFllow的支持很弱，而且目前看不到改善迹象。考虑NPU的支持方案考虑替换掉tensorflow

#### 训练 <a href="#xun-lian" id="xun-lian"></a>

华为NPU与业界生态对接最完备的是Pytorch，在CV/NLP场景下，已经充分验证了包括基于Pytorch的分布式训练框架DeepSpeed。

Numerous目前依赖TensorFlow作为Dense部分的训练引擎。随着Pytorch在业界的兴起，Numerous有必要开始Pytorch训练的支持工作。\
在Numerous训练框架GPU多机多卡版本的开发中，已经具备了一部分替代能力。当前的训练任务各个进程的关系如图所示：

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

目前TF进程作为dense部分的训练进程，是以独立进程运行的。

在NPU场景使用pytorch训练，通讯关系调整为：

<figure><img src="../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

原有的TF进程替换为Pytorch进程，由于NPU的算子开发难度大，sparse计算部分需要支持放在CPU上进行。对应的，sparse之间的通讯机制修改为rpc，worker进程与pytorch进程之间采用共享内存或者IPC通讯。

风险：当前算法人员在无量上仍然使用tf的接口在开发，算法人员在使用pytorch版本进行开发时，需要重新开发，前期只在类似GR的新模型架构上推荐使用Pytorch的版本。

#### 推理 <a href="#tui-li" id="tui-li"></a>

yinian负责支持HSTU在NPU上的执行，DenseGraphEngine如果要在NPU上运行，接入OnnxRuntime作为引擎来支持。



* HSTU前向/反向计算时间长。通过在训练和推理上使用高度定制化的融合算子来解决
* 线上推理资源需求成数量级增长的问题。通过Prefix Caching的技术来实现将计算复杂度从O(prefix\_token\_num\*item\_num)下降为O(prefix\_token\_num+item\_num)
* LLM相关推理优化技术难以应用在GR场景的问题。通过将yinian LLM封装成为独立的计算引擎，并定制Cache管理接口，实现PrefixCaching技术的接入。
* 问题4: 硬件供应商的问题。通过“5.2.5 针对NPU的设计”, 可以实现训练和推理都采用NPU。

