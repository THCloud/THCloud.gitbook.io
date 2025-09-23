# Rule规则引擎

## **1. 背景**

互联网技术飞速发展，坏人攻击手段也快速多变。过去安全烟囱式的业务开发模式，使得策略开发同学在与坏人对抗过程中颇显疲惫。为此，经过数月努力，规则引擎WeRule作为平台的核心组件，对接了包括机器学习平台，数据仓库，数据分析系统，实时计算系统，用户反馈系统等灵磐其他组件，驱动整个灵磐平台闭环运行。

目前业界优秀的规则引擎项目，外部开源的有drools，公司内部有src平台。规则引擎的核心目标是为了解决用户复杂多变的业务逻辑，能提供一种便捷的方式给用户调整，因此，规则引擎的架构基本一致，但由于采用的实现语言不同，规则引擎在规则执行效率和资源消耗方面都有差异。

| 对比项           | Rules      | SRC       | drools    |
| ------------- | ---------- | --------- | --------- |
| 开发语言          | c++        | lua       | java      |
| web workbench | 支持         | 支持        | 支持        |
| 规则版本管理        | 支持         | 支持        | 支持        |
| 冷启动耗时         | 短，无需重新加载规则 | 长，需重新加载规则 | 长，需重新加载规则 |
| 规则执行速度        | 极快         | 较快        | 快         |
| 空跑对比          | 支持         | 支持        | 不支持       |
| 内存消耗          | 较低         | 较低        | 高         |

SRC和drools在服务冷启动时重新加载规则，耗时较长，Rule的规则经过第一次更新编译后，规则数据保存在共享内存，重启服务无需再加载；

drools执行规则时会保存中间结果以加速规则执行，当规则结点较多时内存消耗很大，SRC和Rule不会保存中间结果，内存开销低；

SRC受限于lua，规则结点表达式计算效率不如c++和java，我们在benchmark中，对”(a-b)+(c\*d+e)”循环执行10000次, Rule耗时620ms，luajit耗时1288ms。

Rule经过近一年研发迭代，功能已相对完善，具有如下特点：

* 基于c++语言实现，规则执行速度快；
* 双buff无锁更新规则，性能显著；
* web workbench，拖拽式控件化规则工作台，脱离写代码上线；
* 规则版本管理与回退机制，提供类svn的版本记录，出现问题可快速回退到历史任意版本；
* 空跑与用户反馈，当一个规则版本发布后，可空跑对比现网生效的规则，上线后直接打通用户反馈，避免规则错误对现网用户造成影响；
* 规则插件，极大提升规则引擎的使用场景，降低业务接入门槛。

Rule已经在内部逐步使用，目前已接入33个业务场景，60+规则策略，接下来将详细阐述Rule的设计与实现。

&#x20;

## **2. 系统设计** <a href="#id-2.-xi-tong-she-ji" id="id-2.-xi-tong-she-ji"></a>

Rule整体包含六大核心部分，分别是：workbench，编译引擎，执行引擎，存储引擎，规则插件，异步mq。系统架构如下图所示：

![](https://km.woa.com/asset/004ef60d10b74b4897b3f17a212593f3?height=639\&width=969\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

Rule系统架构图

Web WorkBench：规则编写网页工作台，提供规则树查看、新建、编辑、发布与回滚的能力，同时也为规则树提供语法检测及逻辑检测，确保规则树有效；

RulePublish MGR：规则管理server，主要负责与workbench交互，以及与规则信息RuleDB交互；

FilePublishSystem：微信统一数据集文件发布系统，负责将RulePublish MGR生成的规则文件发布到现网的RuleSvr集群机器上；

RuleSvr：规则引擎主server，包含编译引擎，执行引擎，存储引擎及规则插件agent核心组件，后面将详细介绍这几块的实现细节；

RuleMQ：规则引擎异步消息队列，负责规则发布前的空跑对比，发布后的用户反馈预警；同时也负责上报规则执行流水，对接数据分析系统，实时计算系统；

&#x20;

### **2.1 workbench** <a href="#id-2.1-workbench" id="id-2.1-workbench"></a>

对于一些经常需要变更的应用场景，传统的方式是修改代码，发布二进制服务，这为开发人员带来很大的开发运营成本。有了规则引擎workbench，我们就能直接在网页上编辑规则，无需再更改代码，极大的提升业务开发与运营效率。下面是Rule workbench的规则编辑页面图：



如上图所示，用户可以在工作台里编辑规则树，每个判断节点里面的表达式语法和c++基本一致，下图是判断节点的表达式输入框：

![](https://km.woa.com/asset/8eb5b55de00b4081b8104a4110d86c65?height=354\&width=344\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

表达式的变量支持业务直接赋值，同时也支持通过workbench的rpc插件赋值，后面将在规则插件分节详细介绍rpc插件的实现原理。

workbench支持规则编辑后正式上线前的空跑，也支持发布后的版本回退，这里不再赘述。

&#x20;

### **2.2 规则版本控制RVC(Rule Version Control)** <a href="#id-2.2-gui-ze-ban-ben-kong-zhi-rvcrule-version-control" id="id-2.2-gui-ze-ban-ben-kong-zhi-rvcrule-version-control"></a>

每次规则变更发布都对应一次规则版本记录，考虑到业务开发同学在维护规则的时候，需要一个可记录历史每次变更内容的能力，规则异常时可回退到指定版本，我们设计实现了一套规则版本控制RVC机制。

![](https://km.woa.com/asset/f107c9754a0c42519708b855c7fae030?height=702\&width=854\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

规则版本控制示意图

master主干保存历史版本记录，每次发布后的版本作为最新版本合入master版本主干上，每次编辑规则树时，都是一个develop开发分支，编辑完新特性保存后即生成一个最新版本号。版本号是uint32类型，最终会记录于规则存储。规则存储是在规则svr机器上开辟的大块共享内存，规则双buff更新，最近两个版本做切换，在规则编译与加载分节中，将详细介绍规则存储的设计。

&#x20;

#### **2.2.1 版本回退** <a href="#id-2.2.1-ban-ben-hui-tui" id="id-2.2.1-ban-ben-hui-tui"></a>

和二进制服务部署一样，高可用的规则引擎必须支持规则回退，否则一旦上线错误的规则不能及时回退，将带来严重的运营事故。

![](https://km.woa.com/asset/96cd3845554a41dd897d8bcedd8b0d31?height=518\&width=641\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

版本回退示意图

如上图所示，假设当前最新版本是v3，需要回退到v2，dev开发分支在v2的基础上保存发布，即完成回退。回退后的版本id是新生成的v4，和新增规则树版本一致，这么做的好处有以下几点：

* 和svn代码回退一致，符合开发同学代码回滚习惯；
* 最新版本即工作版本，发生问题能更快速定位；
* 开发维护成本更低，回退时，仅需生成新增版本号；

&#x20;

### **2.3 规则编译** <a href="#id-2.3-gui-ze-bian-yi" id="id-2.3-gui-ze-bian-yi"></a>

通过RulePublishMgr生成的规则文件，由文件发布系统发布到现网规则引擎RuleSvr机器，再由管理进程编译加载到本机共享内存上。

![](https://km.woa.com/asset/7a6fbbe984c2428fb98213258cf24054?height=297\&width=725\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

如上图所示，scanner会定时扫描工作目录规则文件是否有更新，若发现有更新，则将更新的规则文件交给lex\&yacc compiler编译。编译完成后，输出的堆栈数据通过加载器保存到共享内存上，同时更新时间戳，作为下次scanner扫描的基准时间戳。

scanner工作于异步管理进程，在扫描规则文件时不影响线上规则执行。scanner用到的基准时间戳存于共享内存，首先扫描出工作目录的规则文件，然后依次比较规则文件和基准时间戳大小，新上线的规则文件，时间戳比基准时间戳大，scanner将新上线的规则文件传给compiler处理，最后再更新基准时间戳。基准时间戳存于本机共享内存的好处是在机器宕机又恢复服务后，scanner能及时重编所有规则文件，避免服务故障。

&#x20;

#### **2.3.1 lex\&yacc compiler** <a href="#id-2.3.1-lex-and-yacc-compiler" id="id-2.3.1-lex-and-yacc-compiler"></a>

规则文件经过scanner过滤后，转入compiler编译流程。Rule处理的规则和c++表达式语法规则一致，在1.0版本时，编译过程由逆波兰表达式解析器完成，编译输出共享内存上的规则树对象。这种方案有不少问题：

* 开发成本高，所有代码都得自己完成，项目整体进度受影响；
* 容易出bug，语法规则很容易覆盖不全和解析出错；
* 解析代码繁琐难维护，且扩展语法容易导致旧语法出轨。

因此，在1.0版本后，我们引入lex\&yacc作为规则编译引擎。lex和yacc是unix下两款强悍的编译工具，lex可以看作词法解析器，yacc看作语法解析器。正如英文由单词和语法规则组成一样，c++表达式也由变量、各种运算符和c++语法组成，所以复杂的语法表达式经常由lex和yacc配合使用。compiler结构如下：

![](https://km.woa.com/asset/77ef5fe9a1a44f169adda6a6c6d6c274?height=465\&width=825\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

lex\&yacc compiler 结构图

首先，我们定义好语法解析规则文件expr.y以及词法解析规则文件expr.lex，lex读入expr.lex词法描述生成词法解析器，即expr.yy.cpp的yylex；yacc读入expr.y语法描述生成语法解析器，即expr.tab.cpp的yyparse。yyparse和yylex共同构成编译引擎。

原始的规则数据是中缀表达式，而后缀表达式（逆波兰）计算高效，因此，我们在expr.y和expr.lex分别定义语法动作和词法动作，编译输出逆波兰表达式堆栈，于是compiler引入了“逆波兰数据索引容器”和“数据容器”。

![](https://km.woa.com/asset/b0d90913912744e0932ae5538821a42f?height=279\&width=690\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

compiler容器数据结构

vector\<ExpNode> vec\_expnode主要负责存储词法解析器的输出，vector\<SuffixData> vec\_suffixdata主要负责存储语法解析器的输出。下面将以一个简单例子说明表达式编译过程。

expr.lex部分词法定义：

```lua
…
…
%%
"false"  {                     
uint64_t x=0 ;
int exp_node=ExpNode::push_back(CONST_VALUE,x);
yylval.val=exp_node;
return BOOL;
         }
"true"   {
uint64_t  x=1;
int exp_node=ExpNode::push_back(CONST_VALUE,x);
yylval.val=exp_node;
return BOOL;
         }
[0-9]+    {
int64_t x=atol(yytext);
int exp_node=ExpNode::push_back(CONST_VALUE,x);
yylval.val=exp_node;
return(INTEGER);
          }
"+"       {
int exp_node=ExpNode::push_back(PLUS);
yylval.val=exp_node;
return(PLUS);
           }
"-"        {
int exp_node=ExpNode::push_back(MINUS);
yylval.val=exp_node;
return(MINUS);
           }
"*"        {
int exp_node=ExpNode::push_back(MUL);
yylval.val=exp_node;
return(MUL);
           }
"/"        {
int exp_node=ExpNode::push_back(DIV);
yylval.val=exp_node;
return(DIV);
           }
"&&" 		{    
	int exp_node=ExpNode::push_back(LOGICAND);
yylval.val=exp_node;
return LOGICAND;
        	}
[a-zA-Z\_]*(::[0-9a-zA-Z\_]*)*[0-9a-zA-Z\_]*[0-9a-zA-Z]* {
std::string  x(yytext);
int exp_node=ExpNode::push_back(VAR_VALUE,x);
yylval.val=exp_node;
return NAME;
}
%%
```

&#x20;

expr.y部分语法定义：

```lua
…
…
%token CONST_VALUE VAR_VALUE
%token NAME STRING INTEGER DOUBLE  BOOL
%token PLUS MINUS MUL DIV MOD DIVLEFT
%token LOGICAND LOGICOR
%token LOGICAND_JC LOGICOR_JC
%left LOGICOR
%left LOGICAND
%left MINUS PLUS
%left MUL DIV MOD
…
%%
exp
 	: INTEGER	{
		   SuffixData::push_back_suffix_data($1);
		   $$=$1;
				 }
	| NAME {
		    SuffixData::push_back_suffix_data($1);
		    $$=$1;
			}
|exp  LOGICAND exp {
       	SuffixData::push_back_suffix_data($2);
	       int64_t d=SuffixData::distance_to_end($1);
			int64_t step=ExpNode::push_back<int64_t>(CONST_VALUE,d);
			int order=ExpNode::push_back(LOGICAND_JC);
			SuffixData::insert_after_suffix_data($1,order);
			SuffixData::insert_after_suffix_data($1,step);
			$$=$2;
                    		}
| exp PLUS exp {
SuffixData::push_back_suffix_data($2);
			$$=$2;
        			  }
| exp MINUS exp {
SuffixData::push_back_suffix_data($2);
			$$=$2;
        			  }
| exp MUL exp {
SuffixData::push_back_suffix_data($2);
			$$=$2;
        			 }
| exp DIV exp {
SuffixData::push_back_suffix_data($2);
			$$=$2;
        			 }
%%
…
```

输入表达式：”a + b\*2 && c \* (d + g)”

编译流程如下所示：

![](https://km.woa.com/asset/7c112492f18346c298debd12186096cc?height=3062\&width=1034\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

上面详细描述了编译”a + b\*2 && c \* (d + g)”的过程，编译结果数据保存在表达式结点容器和索引容器，最终加载到共享内存上的规则数据基本就是容器数据。在规则执行的时候，扫描一遍逆波兰索引容器即可完成计算，关于表达式计算后面在规则执行分节将详细介绍。

需要注意的一点是，我们的编译过程实现了逻辑表达式短路算法：跳转法。如上述编译过程step16\&step17，在语法“exp && exp” 归约&&的过程中，首先将&&索引id=5 压入到逆波兰索引容器，然后将计算出来的跳转长度和跳转符压入到表达式结点容器，最后将跳转长度和跳转符对应的索引id插入到逆波兰索引容器位置”1”后。短路算法能极快加速逻辑运算，提升规则引擎执行效率。常见的短路算法有跳转法和递归树法，因递归树法实现相对复杂，且因递归栈开销导致内存开销也较大，所以一般编译期的短路算法都是用跳转指令实现。如下面这段c代码，表达式”a && b && c”经gcc编译后的汇编代码里也插入了跳转指令实现短路逻辑。

![](https://km.woa.com/asset/03b54947e94a482c89e0ccaab0469c0f?height=184\&width=143\&imageMogr2/thumbnail/1540x%3E/ignore-error/1) ![](https://km.woa.com/asset/8433df743c664a9e8900616a982b456d?height=180\&width=215\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

\


#### **2.3.2 规则存储与加载** <a href="#id-2.3.2-gui-ze-cun-chu-yu-jia-zai" id="id-2.3.2-gui-ze-cun-chu-yu-jia-zai"></a>

规则经过compiler编译后的结果输出到两个容器，有三种方案可以存储。1. 进程cache；2. 共享内存；3. kv类存储。

| 存储类型 | 进程cache                                                                                | 共享内存                                                                                              | kv                                                        |
| ---- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| 优点   | <p>1.    开发成本低，基本只需对编译输出稍作逻辑调整；</p><p>2.    效率高，无rpc调用，规则执行时间开销小</p>                   | <p>1.    可用性高，进程重启后无需重建规则；</p><p>2.    支持进程间共享规则数据，规则只需编译一次；</p><p>3.    效率高，无rpc调用，规则执行时间开销小</p> | <p>1.    开发成本低，调kv接口即可；</p><p>2.    可用性高，kv组件有容灾机制；</p>   |
| 缺点   | <p>1.    初始化时间开销高，worker进程间无法共享数据，每个进程都需要compiler编译；</p><p>2.    可用性低，进程重启后规则数据即丢失</p> | <p>1.    开发成本高，项目整体进度面临考验；</p><p>2.    机器宕机后共享内存会丢失，需要重建内存；</p>                                   | <p>1.    效率低，每次需要rpc拉取kv数据；</p><p>2.    占用kv资源，运营成本高；</p> |
| 选用结果 | 否                                                                                      | 是                                                                                                 | 否                                                         |

经过上述对比，最终选择共享内存方案。为了降低开发成本，我们选择使用c++进程间通信基础库boost interprocess（后面简称bipc）。

* bipc高度封装了进程间通信和同步机制；
* 基于红黑树实现了高效的内存分配算法；
* 提供了可定制化的STL-like分配器allocator，这使得用bipc allocator替换std allocator，可以很方便的在共享内存上使用STL-like容器。

bipc实现于三大基础类：memory algorithm，segment manager和managed memory segment。类结构关系如下图所示：

![](https://km.woa.com/asset/372f53fa424f4582b62232ad8af878f0?height=768\&width=846\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

bipc核心类关系图

memory algorithm是bipc独立的内存分配算法类，负责对象的内存分配；segmen tmanager继承于memory algorithm，提供丰富的接口管理内存段；managed memory segment负责创建共享内存，同时提供接口获取segment manager对象。下面将先介绍内存分配算法memory algorithm，再说明规则在共享内存上的加载与管理。

&#x20;

#### **2.3.2.1 simple\_seq\_fit** <a href="#id-2.3.2.1-simple_seq_fit" id="id-2.3.2.1-simple_seq_fit"></a>

simple\_seq\_fit是bipc基于单链表实现的以地址有序的内存分配算法，属于memoryalgorithm中的一种。它将managed memry segment申请创建的共享内存段按一定大小分成自由内存块。

![](https://km.woa.com/asset/52c3a79e5b5d47299e53093e7851cd70?height=254\&width=690\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

当业务申请N字节内存时，分配器遍历空闲块有序链表寻找足够大的内存块。如果空闲块的内存正好等于申请的内存，分配器将该内存块从空闲块链表中删除给业务使用；如果空闲块的内存大于申请的内存，分配器切分该内存块，一块给业务使用，一块保留。当业务删除数据对象释放内存的时候，分配器遍历链表找到合适的地址，将内存块插入空闲链表。simple\_seq\_fit算法额外内存开销小，但是内存申请和释放，时间复杂度都是o(n)，适用于小数据规模的应用场景。

&#x20;

#### **2.3.2.2 rbtree\_best\_fit** <a href="#id-2.3.2.2-rbtree_best_fit" id="id-2.3.2.2-rbtree_best_fit"></a>

rbtree\_best\_fit是bipc基于红黑树实现的以空闲内存段大小排序的内存分配算法。因为基于红黑树，内存分配和释放的时间复杂度都是log(n)，相比于simple\_seq\_fit，性能有很大提升。rbtree\_best\_fit同时还设计了双链表数据结构，这让相邻内存块merge操作仅需要o(1)时间就能完成。当空闲内存块被申请使用后，用于构建红黑树结点的数据将被覆盖写，这使得维护已分配内存仅仅只有双链表的开销。

![](https://km.woa.com/asset/7c91e958657c4c408a6089ba5bcedcfe?height=197\&width=783\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

如上图所示，空闲块由双链表指针和红黑树控制数据以及数据内存构成，空闲块大小严格按照对齐值评估。对大多数32位8字节对齐系统来说，最小空闲块是24字节，业务申请分配1字节的数据，将至少需要24字节的内存块开销。对于大多数小数据(< 8字节)分配场景，rbtree\_best\_fit比simple\_seq\_fit更浪费内存，而对大数据场景，两者的内存开销基本相同；rbtree\_best\_fit算法分配性能更高，因此bipc用rbtree\_best\_fit算法作为内存分配的默认算法。

&#x20;

#### **2.3.2.3 规则加载与管理** <a href="#id-2.3.2.3-gui-ze-jia-zai-yu-guan-li" id="id-2.3.2.3-gui-ze-jia-zai-yu-guan-li"></a>

bipc不仅实现了高效的内存分配算法，同时也提供了STL-like allocators，使得业务能够在共享内存上方便的使用STL-like 容器。allocators分配器负责数据对象构建和内存申请，和STL分配器一样，bipc 分配器也实现有内存池机制，避免频繁的申请和释放内存。

回顾规则编译小节，我们知道编译器输出两个数据容器。在规则引擎1.0版本的实现中，每个规则以RuleTree对象在共享内存上构建，这种方式在执行规则时效率颇高，因为找到规则地址后，可以直接获取RuleTree对象执行。但是在后续的版本迭代中，需要支持的功能特性越来越多，而这种实现方式扩展性太差，只要涉及到RuleTree结构的调整，都需要重编加载全部规则，相当于规则冷重启，导致现网服务抖动。

![](https://km.woa.com/asset/b561b7b93ac84867850db82afa4a009b?height=347\&width=1002\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

因此，我们需要一种灵活可扩展的数据载体用来存储规则，考虑到执行规则时，只需传入编译引起输出数据，于是我们将RuleTree的成员变量数据抽象成protobuf message结构，利用强大的protobuf扩展规则数据结构；同时将规则执行单独抽象成执行引擎，整体设计层次更清晰。

用protobuf能有效支持规则数据扩展，但取出数据需要反序列化有一定的cpu消耗，我们调研到同样源自google的数据协议flatbuffers可直接读取数据，协议灵活可扩展，后续将考虑用flatbuffers替代protobuf，以提升规则执行性能。

目前业务规则id等于模块名+规则名，模块之间相互独立，可保证规则id的唯一性。bipc支持命名对象，我们以规则id为key，pb序列化buffer为data存入一个规则。bipc提供多种索引数据结构，比如boost::flat\_map，boost::map，boost::unordered\_map，flat\_map是有序vector的封装，下面是各索引的对比：

| 索引类型  | flat\_map                              | map                      | unordered\_map         |
| ----- | -------------------------------------- | ------------------------ | ---------------------- |
| 插入复杂度 | O(logN)                                | O(logN)                  | O(1)                   |
| 查询复杂度 | O(logN)                                | O(logN)                  | O(1)                   |
| 优点    | 额外空间开销低                                | 不存在重分配问题                 | 插入和查询效率极高              |
| 缺点    | 分配的连续地址空间写满数据后，需要重新申请新内存块，同时将旧数据拷贝到新地址 | 插入数据时，可能需要rebalance tree | 内存利用率差，数据写满后，存在重分配内存问题 |

flat\_map适用于初始化一批数据，运营时只提供查询服务，它是bipc默认索引。规则数据需要实时低频更新，考虑到flat\_map和unorded\_map都有重分配问题，当数据规模较大时，会导致严重的业务抖动，因此我们选择boost::map作为索引。

在共享内存上更新已有规则时，有两种方案可以选择：加锁更新和双buffer切换。加锁更新会降低系统性能，双buffer会占用更多的内存，权衡之下，我们选择双buffer方案。假如现有三个业务规则：exp\_one, exp \_two, exp \_three，规则内存布局如下图所示：

![](https://km.woa.com/asset/454e1f8c581444aab025e492652d5b9d?height=319\&width=913\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

规则内存布局图

业务规则管理结点的数据是:

```cpp
struct RuleCtrl {

       int32_t cursor;                 //工作指针

       uint32_t version[2];        //规则版本号

};
```

&#x20;

上面示例图中，example\_one的cursor为0，则当前工作版本version\[0]为13，从而找到当前example\_one的规则数据(example\_one\_13，PbRule buffer)。

以example\_one为例，规则更新时，由管理进程将规则新的版本编译后加载到共享内存，更新流程如下图所示：

![](https://km.woa.com/asset/ee57de8bb6a34d1ca568b43dd3f74950?height=650\&width=814\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

1. 销毁exp\_one\_12规则结点对应的数据，避免内存泄漏；
2. 构建exp\_one\_14规则结点数据，同时将版本号14赋值于RuleCtrl的version\[1]；
3. 切换RuleCtrl工作指针，将cursor切换为1；

这就是规则双buffer更新的完整流程，整个更新过程由worker进程完成，worker进程负载规则执行，完全不受规则更新影响。

&#x20;

#### **2.3.2.4 规则执行** <a href="#id-2.3.2.4-gui-ze-zhi-xing" id="id-2.3.2.4-gui-ze-zhi-xing"></a>

由上一小节规则内存布局图我们可以知道，规则在执行时，通过规则id最终得到PbRule buffer，反序列化就能得到可用的PbRule结构，PbRule部分定义如下：

```c
message PbRule  {
repeated PbNode nodes = 5;	//规则树
repeated PbPlugin plugins = 12;	//插件相关
}

message PbNode {
optional uint32 node_id = 1;		//规则树结点id
optional uint32 node_type = 2;		//规则树结点类型(0：初始结点；1：逻辑结点；2：返回结点)
repeated uint32 abbreviated_suffix_expression = 4;		//逆波兰结点类型栈，由编译器输出转换获得
repeated PbOperand specific_data = 5;		//逆波兰结点数据栈，由编译期输出转换获得
repeated uint32 true_child_nodes_idx = 11;		//结点逻辑为真时子结点
repeated uint32 false_child_nodes_idx = 12;	//结点逻辑为假时子结点
}

message PbOperand	{	//规则结点操作数具体数据和类型
oneof type  {
        bool bool = 1;
        double double = 2;
        sfixed64 int64_t = 3;
        string string = 4;
        fixed64 uint64_t =7;
    }
    optional string comment = 5;
optional string source = 6;
}
```

&#x20;

一个规则对应一棵规则树，一颗规则树对应一个PbRule结构，规则树中的规则结点对应PbNode，如下图：

![](https://km.woa.com/asset/bcbe0a067e854207a4c82b8b107d9176?height=550\&width=972\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

规则树示意图

规则首先从start结点开始，循环遍历决策结点；

决策结点表达式执行结果为true时，走true子结点；

决策结点表达式执行结果为false时，走false子结点；

每一个决策结点都是一个表达式，经过编译引擎最终转化为PbNode:: abbreviated\_suffix\_expression和PbNode:: specific\_data对应的数据（PbNode定义参看前文说明）。仍然以”a+b\*2 && c\*(d+g)”为例说明决策结点的执行过程。abbreviated\_suffix\_expressio从左到右依次是VARIABLE，VARIABLE，CONST,MUL，PLUS，CONST，LOGICAND\_JC，VARIABLE，VARIABLE，VARIABLE，PLUS，MUL，LOGICAND；

specific\_data从左到右依次是a,b,2,6,c,d,g。执行流程示意图如下：

![](https://km.woa.com/asset/9434fc3026604ceda17d97453328a409?height=1390\&width=1226\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

表达式执行流程图

上面省略了操作数入栈出栈的过程，需要注意的是step3处理”&\&jmp”操作符的时候，如果result\_3是true，则会继续执行step4，否则执行短路逻辑，表达式运算结束。因为不确定操作数的类型，实现时运用了c++17的std::variant定义操作数，对应pb里面是one of结构，操作符是variant数据执行体，利用c++17 std::visit实现。

\


### **2.4 规则插件** <a href="#id-2.4-gui-ze-cha-jian" id="id-2.4-gui-ze-cha-jian"></a>

规则的执行需要调用端将表达式数据传过来，然而很多业务数据需要通过rpc获取，调用端希望这些数据规则引擎能帮忙完成；

很多业务在执行规则时，还有校验关键词和频控限制的需求；

为支持这些业务需求，减少业务开发量，扩大规则引擎应用场景，我们实现了一套规则引擎插件框架，并抽象出一些常用的插件，如通用svrkit rpc插件：支持业务自定义rpc拉其他svrkit模块数据；关键词插件：支持业务校验关键词功能；频率插件：支持业务通过规则引擎接入频控限制。接下来2.4.1将详细介绍规则插件框架的实现，2.4.2小节将介绍通用rpc插件的实现。

&#x20;

#### **2.4.1 插件框架** <a href="#id-2.4.1-cha-jian-kuang-jia" id="id-2.4.1-cha-jian-kuang-jia"></a>

插件框架的设计目标：

* 业务接入简单，接入太复杂会增加业务的接入成本；
* 接口设计简单，接口太复杂会增加我们的开发成本，维护性低；
* 模块化设计，避免或减少插件逻辑与规则表达式执行逻辑耦合；
* 性能为王，框架应尽可能高效；

如果我们把编辑规则树理解成写代码的过程，则使用插件就可以理解为使用规则引擎库函数的过程。按照这个思想，现在业务接入规则仅需在workbench里拖入插件。

实现时，以svrkit作为插件底层框架，以agent的形式部署在rulesvr机器上，通过tcp环回端口方式转发数据。

![](https://km.woa.com/asset/db2fbd5f3a4f43539a2c4a28bf4ae751?height=790\&width=1070\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

插件框架图

插件的输入输出和规则树通过数据文件统一发布到rulesvr，再经过规则编译引擎保存在共享内存；执行时，首先从共享内存上解析得到插件数据；

再以tcp环回端口的方式请求插件引擎，通过请求中的插件id，找到对应的插件执行。本机服务间通信方式中，tcp环回端口效率较低， unix socket无需经过网络协议栈，无需计算校验和等，比tcp环回端口效率高，后续将把tcp环回端口通信方式改进为unix socket。

\


#### **2.4.2 通用rpc插件** <a href="#id-2.4.2-tong-yong-rpc-cha-jian" id="id-2.4.2-tong-yong-rpc-cha-jian"></a>

svrkit框架高度封装底层网络通信协议，开发同学很方便就能搭建一个svrkit server，模块间rpc调用通常需要在二进制里依赖调用模块的client lib，因此通用rpc插件首先要解决调用依赖问题。分析svrkit模块sm\_\*\*\*cliproto.pb.cpp文件发现，我们需要定义一个类继承ClientCallObj，传入cmdid(命令字)，magicid(端口)，req(pb请求包体)，rsp(pb回包)；

```cpp
class CommClientCallObj: public ClientCallObj {
public:
    CommClientCallObj(uint32_t uin, 
        int cmdid,
        int magicid,
        const google::protobuf::Message* req,
        google::protobuf::Message* rsp);

    ~CommClientCallObj();
};
```

&#x20;

同时为解决路由问题，我们还需传入模块名。解决了路由与svrkit打包问题后，更大的问题来了，protobuf的请求包message和回包message如何构造？查阅protobuf的官方文档，我们了解到pb文件支持动态编译，**动态编译+反射**，完美解决protobuf message构造问题。业务同学在workbench里使用rpc插件时，首先上传pb文件，然后依次填入请求message名，模块名，请求命令字，目标端口，回包message名。



rpc插件执行时，检测到pb文件有更新，则重编最新pb文件，构造rpc请求获取数据。

&#x20;

### **2.5 规则引擎异步mq与规则生态** <a href="#id-2.5-gui-ze-yin-qing-yi-bu-mq-yu-gui-ze-sheng-tai" id="id-2.5-gui-ze-yin-qing-yi-bu-mq-yu-gui-ze-sheng-tai"></a>

规则引擎WorkBench为业务提供规则增删改查的web工作区，RulePublishMGR为业务提供规则发布，规则版本回退的能力，RuleSvr为业务提供规则编译与执行的能力，丰富的插件也使规则引擎具备扩展功能，扩大了业务使用场景，这些模块组件基本覆盖规则需要的实时接入能力，而离线能力需要一个mq来补充，因此我们搭建了异步mq模块，负责规则引擎的数据流水落地，规则执行结果数据分析，规则发布上线前空跑对比，规则上线后用户反馈，规则请求流量旁路实时计算系统等能力。

![](https://km.woa.com/asset/28872785003e4feebec06ef5e98fd236?height=658\&width=1072\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

异步mq架构图

为了实现规则空跑对比，我们将RuleSvr的编译引擎，执行引擎，存储引擎抽象成lib库的形式由MQ Server依赖，相当于MQ Server有一份RuleSvr的相似镜像。因为空跑不能影响现网规则，所以空跑的规则还没经过文件发布系统更新到现网，文件发布系统指定MQ模块发布。由前文知道规则双buffer更新机制，共享内存保存有新旧两份规则数据，正好适合空跑，将新旧版本规则结果数据对比，不一致的结果入库告警到规则发布流程。

流水系统保存业务请求的规则执行过程和结果，方便有问题时追根溯源。

数据分析系统为规则请求和执行数据提供各个维度的分析能力，业务同学在workbench页面上可以清晰的看到现有规则各维度的数据统计图。

实时计算系统flink提供不同维度数据的特征计算与统计，规则引擎的数据可以流入flink作为数据源，flink计算结果也可以做为新的特征数据反馈接入规则引擎形成闭环。

用户反馈系统为线上的规则提供预警。规则执行异常分支后，用户通过在mq下发的反馈入口提交反馈数据，规则相关负责人收到用户反馈告警，从而判断线上规则是否有异常。

\


### **2.6 性能与容灾** <a href="#id-2.6-xing-neng-yu-rong-zai" id="id-2.6-xing-neng-yu-rong-zai"></a>

**性能:**

| 机型          | 峰值请求量     | 峰值cpu负载 | 接口平均耗时                          |
| ----------- | --------- | ------- | ------------------------------- |
| vc-48(4核8G) | 15.6w/min | 60%     | <p>上海idc：4.5ms<br>深圳idc：6ms</p> |

现网idc单机请求量：

![](https://km.woa.com/asset/2785d819ca584da4af419010227511e9?height=221\&width=435\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

单机cpu负载：

![](https://km.woa.com/asset/7f75d747119e4f5f95c62f59ce3ae32c?height=219\&width=442\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

性能分析：规则编译由RuleSvr异步完成，只有规则更新时才会经过规则编译过程，所以规则执行影响系统性能。回顾规则执行过程，首先需要从共享内存取出PbRule buffer；然后对buffer反序列化；再对规则树遍历执行规则结点表达式；如果规则调用了插件，则还需通过tcp环回端口的形式将数据发送到插件引擎执行插件逻辑。这里有几个优化点：1. PbRule buffer反序列化过程可以引入flatbuffer解决，flatbuffer由google开源，序列化的数据可以不用反序列化直接读取； 2. tcp环回端口改造为unix socket，避免tcp网络协议栈的数据拷贝等的开销；3. 优化log，降低磁盘io，RuleSvr目前打开了很多debug log，可以梳理出来关闭。

&#x20;

**容灾：**

* 自动扩容，RuleSvr是无状态逻辑服务，容量问题可走自动扩容流程；
* 多IDC+多园区部署，充分保障服务稳定运行；
* 分模块部署，敏感重要业务单独部署RuleSvr
* 熔断机制，避免因单个业务规则流量突发暴涨影响其他业务规则；
* 共享内存重建机制，确保RuleSvr宕机后再恢复，所有业务规则能全部编译加载到共享内存；
* 规则多版本管理，确保有bug的业务规则能通过快速回滚机制而恢复服务；
* 空跑对比与用户反馈相结合，对异常规则告警提醒负责人；

&#x20;

### **2.7 现网运营** <a href="#id-2.7-xian-wang-yun-ying" id="id-2.7-xian-wang-yun-ying"></a>

目前RuleSvr部署有33台vc4-8；

日请求量27.5亿，峰值请求量300w/min；

![](https://km.woa.com/asset/ea91619ab9384f4092d4cb8ce38e2f02?height=110\&width=554\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

接入33个业务场景，共有186个业务规则在跑；

覆盖微信安全的注册，加好友，朋友圈，微信支付商户入驻审核，搜索应用部的看一看直播等业务；

业务规则累计变更3651次，解决了业务因规则经常变更而改代码上线的痛点；

99.98%的接口成功率以及通过重试最终99.999999%的成功率，体现了WeRule的高可靠性；

![](https://km.woa.com/asset/5c0c0e7818a643598becc3124652cd85?height=94\&width=554\&imageMogr2/thumbnail/1540x%3E/ignore-error/1)

## &#x20;
