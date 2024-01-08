# 近邻检索

如何在大规模向量库中快速的找到距离最近topk向量集合？这个问题在学术界上被称为“最近邻检索”（ANN，Approximate Nearest Neighbor）。主流ANN算法有四类

1. 基于树结构的方法

首先使用树结构（KD-Tree KMeans-tree等）对向量空间进行逐层切分，然后将query向量q从根节点开始走一遍树结构，直到叶子节点。与q同叶子节点的候选向量作为近邻向量被返回。如果单个叶子节点中的候选向量数目不够，则树方法还可以进行回溯，召回更多的临近叶子向量。\
树方法优势是原理简单，实现容易，缺点是在向量维度较大时候退化为线性搜索，几乎等于暴力。

2. 基于hash的方法

hash类的方法首先使用hash函数将向量投射到一个向量桶中，理想的hash函数能够将相近的向量投射到同一个向量桶中，近邻搜索时候，在q所在向量桶和邻近桶进行查找，减小搜索范围提高效率。\
hash方法本质依然是对空间的切分。

3. 基于图的方法

基于图的方法分位两步：第一步是构建近邻图，第二步是在近邻图中进行启发式搜索。\
构建近邻图时，先将所有候选向量作为顶点，然后将每个顶点的近邻点都使用边连接起来，近邻点的求解可以使用暴力计算，也可以使用改进方法比如kd-tree。启发式搜索时候，从近邻图任意一个点出发，将该点的近邻加入候选集合U，然后在U中与q进行比较，找到最近的一个点，再将这个新点的近邻加入集合U，重复上述步骤。

4. 基于量化的方法

基于量化（product quantization，PQ）的方法将向量切割为多段子向量，然后分别在每段子向量空间中分别进行K-Means聚类，再将小向量用簇中心点代表，即为量化。同样的将请求向量q进行切割量化，然后用量化后的中心点之间的距离代替原始子向量之间的距离，这样便大大减少了距离计算。\
PQ有一些改进算法，最常见的就是IVFPQ，将量化过程分为两步，第一步先做粗量化，用少量的中心点代表向量全集，这样肯定会产生比较多的残差。还需对残差向量金鼎分段量化，即“积量化”。\
IVFPQ的查询过程：（1）找到与请求向量x最邻近的粗量化中心点，计算两者的残差。（2）计算残差与积量化中心点之间的距离，将此距离作为x与候选向量y之间的距离。（3）返回最近邻粗量化簇中，距离在topk的向量。\
IVFPQ的好处是进一步缩小了搜索集合，加快了查找效率。Facebook开源了内部项目FAISS，正是标准的PQ索引构建库。

{% embed url="https://blog.csdn.net/u013066730/article/details/106252573?spm=1001.2101.3001.4242&utm_medium=distribute.pc_relevant.none-task-blog-baidujs_baidulandingword-0" %}

{% embed url="https://zhuanlan.zhihu.com/p/80552211" %}

{% embed url="https://zhuanlan.zhihu.com/p/264832755" %}

{% embed url="https://www.zhihu.com/question/19640394/answer/207795500" %}

{% embed url="https://cloud.tencent.com/developer/article/1487432" %}

{% embed url="https://zhuanlan.zhihu.com/p/23966698" %}

{% embed url="https://blog.csdn.net/u010376788/article/details/46957957" %}
