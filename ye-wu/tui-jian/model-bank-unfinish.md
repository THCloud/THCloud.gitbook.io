# model bank（unfinish）

{% embed url="https://github.com/alibaba/x-deeplearning/wiki/%E6%B7%B1%E5%BA%A6%E6%A0%91%E5%8C%B9%E9%85%8D%E6%A8%A1%E5%9E%8B(TDM)?spm=ata.21736010.0.0.596e2610b5VaRR" %}

无论是DNN、DIN、还是DIEN，以高维稀疏id作为原始输入的深度学习模型都可以被看做是服从Embedding+Dense网络的范式，在这个范式下，绝大多数的参数集中在Embedding部分（参数量10billion级别），而模型结构的迭代和研究多集中在Dense网络部分（million级别）。在过去的经验中，深度学习模型采用端到端的训练方式是一种公认的简洁、高效、可靠的训练模式，然而就是在End-to-End的学习模式下，每当我们想迭代一个新的网络结构或者想增加一些特征的时候，我们都不得不重新训练一个全新的模型。
