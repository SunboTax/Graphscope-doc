# 图分析引擎
Graphscope的图分析引擎继承自Grape。
Grape引擎通过自动并行化单机顺序图算法，实现了现有算法的分布式运行，高效处理大规模图。

Grape在论文[Parallelizing Sequential Graph Computations](./reference/Parallelizing%20Sequential%20Graph%20Computations.pdf)中提出。自动并行化过程如下：
1. Partial Evaluation(PEval):在接收到查询请求$Q$时，集群中每一个$worker$分别计算本地$Fragment$结果并记录一组必要的参数，处理完成后，发送一条信息到$host$。
2. Incremental Computation(IncEval):当$host$接收到所有$worker$传回的信息后，且没有没有待定信息后，Grape调用Assemble组合结果并结束查询；否则将最后一个$worker$发来的信息分发给其他$worker$，每个$worker$在接收到信息后开始增量运算（即通过接收到的信息更新更新本地$Fragment$参数，然后进行本地查询运算），之后同步骤1发送信息给$host$。

---
# Grape论文阅读笔记

## 一、定义：
1. $G = (V,E,L)$，$G' = (V',E',L')$满足$V'\in V， E'\in E，\forall v \in V, L'(v) = L(v)$或$\forall e \in E',L'(e)=L(e)$，则$G'$是$G$的子图
2. $Partition$：将$G$划分为一系列片段$F_{i,i\in[1,m]}$，每一个片段是$G$的子图。用户可以采用任意图划分策略。
    - $F.I$记录该片段上所有存在外部入边的节点
    - $F.O$记录该片段上所有存在外部出边的节点
    - $\mathcal{F}.O = \cup_{i\in [1,m]}{F_i.O}$，$\mathcal{F}.I = \cup_{i\in [1,m]}{F_i.I}$。
    - $F_i.O \cup F_i.I$称为$F_i$的边界点

<center id = figure_1>

``` graphviz
digraph figure{
    rankdir = LR;
    compound  =true;
    subgraph cluster_1{
        label = "F1"
        A->B;
        B->C;
    }
    subgraph cluster_2{
        label = "F2"
        D->E;
        E->F;
    }
    F->A;
    C->D;
}

```
<div>Figure 1 图划分</div>
</center>

如上图[Figure 1](#figure_1)，$F_1.I=\{A\}$，$F_1.O = \{C\}$，$F_2.I=\{C\}$，$F_1.O = \{A\}$，

## 二、并行化编程

### (一)、 Grape并行模型
给定图查询问题类$\mathcal{Q}$，用户需要明确划分策略$P$，局部评估函数$PEval$，增量计算函数$IncEval$以及聚合函数$Assemble$。将图$G$划分到$m$个片段之后，为每个片段分配一个虚拟的非共享内存的$worker(P_1,P_2,...,P_m)$。如果物理节点数量$n<m$,那么将会有多个虚拟$worker$映射到同一个节点上，这些虚拟$worker$共享内存。
- 局部评估：在接收到查询$Q \in \mathcal{Q}$后，每一个$worker\ P_i$开始调用$PEval$计算局部结果$Q(F_i)$。同时需要找到边界点并初始化记录边界点状态的更新参数。完成局部计算后，每个$worker$根据更新参数向$host$节点发送一条信息，
- 增量计算：该步分为两个阶段不断迭代，直至结束
    - $host\ P_0$检查是否所有的$worker$都处于非激活状态（即本地计算已经结束且$host$没有需要传递给$worker$的信息）。如果都处于非激活状态，那么调用聚合函数并结束查询；反之，将上一轮迭代中的信息传递给$worker$，并继续下一阶段。
    - $worker\ P_i$在接收到信息$M_i$之后，调用$IncEval$函数增量计算$Q(F_i \oplus M_i)$（其中$M_i$是更新信息用以该边更新参数）。$IncEval$函数将自动发现更新参数的变动并将这些变动打包为信息发送给$host$。
- 聚合：当更新参数不在有变化时$host\ P_0$结束整个查询，此时，$P_0$拉取所有的局部查询结果，调用$Assemble$函数计算$Q(G)$。

### (二)、Partial Evaluation
$PEval$函数在每一个片段上计算查询问题$Q$的局部结果$Q(F_i)$，因此该过程可以所有片段上的计算可以并行化进行，而每个片段上的计算可以采用已有的任何串行算法。需要额外注意的是我们需要为这些串行算法添加保存局部结果的变量，并且需要指定用于和$IncEval$交互的信息。
不同$worker$之间的通讯通过消息传递进行，消息定义为更新参数，包括以下部分
1. 消息前导码：$PEval$定义一个状态变量$\vec{x}$，确定和$F_i.I$或$F_i.O$相关的点和边的集合$C_i$，与$C_i$相关联的状态变量记为$C_i.\vec{x}$，该变量即为$F_i$的更新参数。直观上$M_i$用于更新$C_i.\vec{x}$。更具体的定义$C_i$:给定一个整数$d$，$S$是$F_i.I$或$F_i.O$，那么$C_i$就是$S$中节点的所有$d-hop$邻居节点（或边）。这也就是意味着：当$d=0$，$C_i=S$；否则，$C_i$可能包含其他片段中的节点（或边）。
2. 消息段：$PEval$可能需要指定$AggregateMsg$函数来解决通讯冲突问题（多个$worker$给$host$发送消息后，同一个更新参数的写入冲突）。
3. 消息分组：GRAPE推导每个片段的$C_i.\vec{x}$的更新，并将其作为不同$worker$之间交互的信息。更加详细的步骤如下：
    - 识别$C_i$。$P_0$通过分割图$G_P$的推导，$C_i$在整个过程中保持不变。$P_0$维护了片段的更新参数。
    - 组合$M_i$。对于每个$P_i$的消息，GRAPE会识别更新参数中发生变动的变量，参考$G_P$推导消息的目的$worker$。如果$\mathcal{P}$是一个边割划分，那么标记有$F_i.O$中的节点$v$的$C_i.\vec{x}$的分量将被发送到$v$所属的片段（对$F_i.I$做相同操作）；如果$\mathcal{P}$是一个点割划分，GRAPE将识别所有$F_i$和$F_j$的公共节点。另外GRAPE结合分配给$P_j$的变动的变量值为一条消息$M_j$。

并行过程中不可避免的会出现多个$worker$向$host$发送的消息中存在对同一变量的写入操作，$AggregateMsg$函数将会解决这些冲突，并返回一个结果写入到这个变量中。为减小通信开销和$host$负载，只有更新的变量值会被放入传递消息中，另外会有每个$worker$都可能维护一个$G_P$的副本并并行推导该$worker$发出的消息的目的$worker$。

### (三)、Incremental Evaluation
给定查询问题$Q$，需要更新的片段$F_i$，局部查询结果$Q(F_i)$以及消息$M_i$，$IncEval$函数将会增量式的计算$Q(F_i \oplus M_i)$，根据计算结果对状态变量的改动生成新的消息传递给$host$。
增量函数的界由状态变量的变动量确定，而非片段的大小，$Q(F_i \oplus M_i)=Q(F_i) \oplus \Delta O_i$，即我们始终需要关注的$M_i$引起的$\Delta O_i$。

### (四)、Assemble Partial Results
$Assemble$函数将会通过局部结果$Q(F_i \oplus M_i)$和$G_P$整合出最终查询结果$Q(G)$。
[基于Grape的分布式sssp_path问题查询算法](./note/SSSP/GraphScope%E5%AE%9E%E7%8E%B0.pdf)是一个简单的串行查询算法并行化示例，展示了$PEval$和$IncEval$的定义方式，这里不需要非常明确的定义$Assemble$函数，因为最短路径查询必然是一个非递增的过程，每一轮增量计算的过程中查询答案的自动聚合。

---
# 算法阅读笔记
1. [PageRank算法](./reference/pagerank.pdf)：