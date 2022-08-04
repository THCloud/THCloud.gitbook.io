# 粗排

粗排可能是整个推荐流程中最尴尬的存在。同作为排序阶段，从未有人质疑过精排的作用，地位牢固；然而粗排却是常常被人怀疑，让人感觉这部分完全可以被召回阶段替代，不必费时费力。 人力投入上，每个推荐业务必有一整个大组去负责精排；粗排只会占据部分人parttime的精力，甚至于没时间就可以不做了。 从实际现状来看，为了弥补单一召回链路的不足，多路召回是必然选择；而为了争取更好的推荐效果，往往需要增加召回深度，提供更丰富的候选集。 然而精排算力有限，又不允许其上游输入的无节制增加。怎么办呢，只能再做召回截断，依据某些条件对召回结果再过滤。这个某些条件往往就是召回score（比如ann的distance）等。

再谈一些其他现状。应该几乎所有算法都碰到过一个灵魂问题：模型迭代，离线评估auc提升明显，上线以后却收益甚微；有时离线评估效果很差，上线以后收益却意外的好。在推荐中， 召回 or 粗排更容易出这种问题。

* 某新闻app，召回模型cf有突破进展，离线评估提升xxx；上线以后发现效果很差不及预期，回头总结，因为精排使用nn模型，召回使用cf模型，nn模型召回的结果对精排更为友好。

上升到全局的视角，可能对这类问题有更深理解：我们提升了模型的效果，其目标到底是什么？往往你需要深入思考下其在全流程中处于什么位置。

我们举个现实例子来深入理解下：现在业务需要搭建一套bp系统。你收到了两份简历：一份简历是hr找到的，国内互联网大厂做java的不知名人士，多年经验；另一份简历是猎头找到的，是jeff dean。 已知面试官的精力只够面试一个人，两份简历摆在你面前你要做简历评估，你是会留java人士的简历，还是留jeff dean的简历呢。

用这个例子再套回推荐系统中。以粗排为例，粗排的流程在召回之后，在精排之前。那粗排效果的提升目标，是为了在召回结果中找到更好的推荐结果，还是为了在召回结果中找到精排更希望要的结果？结合上面的例子， 就会有更深的理解。

再返回上一层，讨论粗排要不要做。**先说结论：资源有限的前提下，放弃粗排确实是最优选择之一**。面试官面试，考量候选人与岗位的匹配程度，是不可缺少的部分；比起留下jeff dean还是留下java人士，能找出这两份简历是更重要的事情。 精排的输出就是最终推荐结果，这部分建设无比重要；召回的输入是海量候选集，输出是粗筛结果，粗筛结果是后续流程的起点，比起中间阶段，越往前的位置越重要。 在推荐系统起步的阶段，大力建设召回和精排会更快速获取业务收益；然而对于一个成熟的推荐系统来说，当召回和精排都已经遇到瓶颈（虽然没人会承认到瓶颈了），或者精排资源明显不足的时候，建设粗排会变成一个更好的选择。

当然这些是理想态，实际上建设粗排往往是个很让人纠结的事情。作为中间阶段，粗排设计本身不适合太重；那一个不应太重的阶段，它的优化目标是什么，是否有必要作为单独阶段处理。而如果深度设计，当越来越多的特征被引入到粗排，更多的工作被前置，就让人很迷惑：粗排和精排的界限到底在哪里。由此也会让人深思，粗排扮演的地位到底是什么，想不清楚定位，就减少了人力投入。对于粗排的理解，也是后面的日子里值得更深入思考的事情。