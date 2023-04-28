# 图分析引擎
Graphscope的图分析引擎继承自Grape。
Grape引擎通过自动并行化单机顺序图算法，实现了现有算法的分布式运行，高效处理大规模图。

Grape在论文[Parallelizing Sequential Graph Computations](./reference/Parallelizing%20Sequential%20Graph%20Computations.pdf)中提出。自动并行化过程如下：
1. Partial Evaluation(PEval):在接收到查询请求$Q$时，集群中每一个$worker$分别计算本地$Fragment$结果并记录一组必要的参数，处理完成后，发送一条信息到$host$。
2. Incremental Computation(IncEval):当$host$接收到所有$worker$传回的信息后，且没有没有待定信息后，Grape调用Assemble组合结果并结束查询；否则将最后一个$worker$发来的信息分发给其他$worker$，每个$worker$在接收到信息后开始增量运算（即通过接收到的信息更新更新本地$Fragment$参数，然后进行本地查询运算），之后同步骤1发送信息给$host$。

    

---
# 算法阅读笔记
1. [PageRank算法](./reference/pagerank.pdf)：